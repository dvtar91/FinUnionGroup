# Cut‑over‑план для домена **Клиентские данные**

> **Цель:** переключить всех потребителей (Core, Card, Loan, BI, Contact‑Center) с локальных CRM‑шар на **MDM Customer Hub** без простоя клиентских сервисов и с гарантией восстановления новых изменений при откате.

## Подготовка

| Шаг | Действие | Ответственный | Критерий готовности |
|-----|----------|----------------|---------------------|
| P‑1 | Обновить версии **MDM API** до 1.3 (поддержка bulk‑load). | MDM‑Team | Тестовый стенд готов, API‑документация опубликована. |
| P‑2 | Сгенерировать **full‑backup** клиентских таблиц в FU CRM и RB CRM (encrypted, off‑site). | DB‑Admin | Проверка контрольных сумм (SHA‑256). |
| P‑3 | Настроить **CDC‑коннектор** (Debezium) → Kafka‑topic `client_changes`. | Integration‑Team | Тестовое событие проходит сквозную валидацию и публикуется в MDM. |
| P‑4 | Провести **dry‑run** миграции 1 % записей в тестовом MDM‑окружении. | Data‑Quality | Ошибки > 0,5 % → откат и корректировка правил дедупликации. |

## Data Freeze (Final Freeze)

* **Время:** 02:00–03:00 UTC (по меньшему нагрузочному окну).  
* **Что замораживается:** любые операции **создания/изменения** клиентских записей в **FU CRM** и **RB CRM** (включая ввод новых заявок в Contact‑Center).  
* **Технические меры:** включить **read‑only** режим на уровне базы (PostgreSQL `ALTER TABLE … SET (FORCE ROW LEVEL SECURITY)`) и включить **maintenance window** в ESB.

## Финальная выгрузка

| Фаза | Описание | Инструмент | Объём |
|------|----------|------------|-------|
| F‑1 | Export всех клиентских записей (incl. исторических изменений) из обеих CRM в **Avro**‑файлы. | `pg_dump` + `avro-tools` | ~ 12 GB |
| F‑2 | Загрузка файлов в **Object Store** (`s3://migration/client‑dump/`) | AWS CLI | – |
| F‑3 | Подтверждение целостности: сравнение **record‑count**, **SHA‑256** контрольных сумм. | Скрипт `checksum_check.py` | – |

## Загрузка

1. **Bulk‑load** в **MDM Customer Hub** через API `POST /customers/batch`.  
2. Параллельно включить **CDC‑репликацию** из обеих CRM → Kafka → MDM (для последующего **транзиентного** обновления).  
3. После загрузки запустить **deduplication job** (`mdm-dedup-job`) и **enrichment** (добавление `risk_score`, `segment`).  

## Сверка

| Метрика | Метод | Порог принятия |
|---------|-------|----------------|
| **Record count** | `SELECT COUNT(*) FROM mdm_customer` vs. сумма записей из обеих CRM | ± 0,1 % |
| **Checksum** | SHA‑256 хеши всех полей (excluding system timestamps) | 100 % совпадение |
| **Business rule** | Все активные клиенты (`status='ACTIVE'`) имеют уникальный `customer_id` в MDM | 0 дублей |
| **Sample audit** | 500 случайных записей → ручная сверка в исходных CRM | < 1 % расхождений |

## Точка невозврата (Cut‑over point)

* **Момент**: после **обновления маршрутизации** в ESB и переключения DNS/Load‑Balancer на **MDM Customer Hub**.  
* **Идентификатор**: фиксируется в журнале как `cutover‑timestamp = <YYYY‑MM‑DDTHH:MM:SSZ>`.  
* **Действие**: старые CRM‑инстансы переводятся в режим **read‑only** и аварийно‑отключаются от внешних запросов.  
* **Сохранение новых изменений**: сразу после переключения все новые/изменённые записи, пришедшие в MDM, реплицируются в **S3‑bucket `crm‑rollback/changes/`** в виде **Avro‑логов** (Kafka‑Connector с `format=avro`). Для каждой записи генерируется SHA‑256‑контрольная сумма и сохраняется в `crm‑rollback/checksums.txt`.

## Переключение (Switch‑over)

1. **Обновление маршрутизации** в **ESB**: все запросы к `/customers/*` перенаправляются к **MDM API** (blue‑green деплой, 5‑минутный откат).  
2. **Обновление клиентских приложений** (Web, Mobile) – изменить endpoint в конфигурации (CI‑pipeline).  
3. **Запуск обработки новых событий** через **Kafka → MDM → Core**.  
4. **Включение CDC‑lag‑мониторинга** (порог ≤ 2 сек) сразу после переключения.

## Фолбэк (Rollback) – сохранение новых изменений

1. **Триггер отката** – любые несоответствия в `Reconciliation` > 0,5 % **или** падение API > 5 минут подряд.  
2. **Процедура**:  
   - **8‑1** Отключить DNS/Load‑Balancer, вернуть маршрутизацию к старым CRM‑эндпоинтам.  
   - **8‑2** Снять **read‑only** режим в CRM, вернуть их в `read‑write`.  
   - **8‑3** **Импортировать** изменения, сохранённые в `crm‑rollback/changes/` назад в **RetailBank CRM** и **FinUnion CRM** при помощи скрипта `crm-replay-import.sh`:  
     ```bash
     #!/usr/bin/env bash
     set -e
     BUCKET=s3://crm-rollback/changes/
     for file in $(aws s3 ls $BUCKET --recursive | awk '{print $4}'); do
       aws s3 cp s3://$file - | avrocat | curl -X POST -H "Content-Type: application/json" \
         -d @- https://old-crm.example.com/api/v1/customers/batch
     done
     ```  
   - **8‑4** После импорта проверить контрольные суммы из `crm‑rollback/checksums.txt` и сравнить их с итоговыми записями в CRM.  
   - **8‑5** Запустить **post‑rollback validation** (record‑count, бизнес‑правила).  

## Мониторинг

| KPI | Порог | Инструмент |
|-----|-------|------------|
| **API latency** (MDM) | ≤ 150 ms (95‑th perc) | Prometheus + Grafana |
| **Error rate** (HTTP 5xx) | ≤ 0,1 % | Loki / Alertmanager |
| **CDC lag** (Kafka → MDM) | ≤ 2 сек | Confluent Control Center |
| **Data freshness** (Core → MDM) | ≤ 5 мин | Airflow DAG `customer_sync_freshness` |
| **Business KPI drift** (Active accounts) | ≤ 0,5 % | Looker dashboard “Customer Health” |
| **Rollback‑log size** | ≤ 5 GB (по дням) | S3 bucket `crm‑rollback/` monitoring |

> **SLA post‑cutover:** 99,9 % доступности клиентских данных, отсутствие критических ошибок более 5 минут.
