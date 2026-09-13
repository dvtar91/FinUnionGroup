# Cut‑over‑план для домена **Клиентские данные**

> **Цель:** переключить всех потребителей (Core, Card, Loan, BI, Contact‑Center) с локальных CRM‑шар на **MDM Customer Hub** без простоя клиентских сервисов.

## Подготовка

| Шаг | Действие | Ответственный | Критерий готовности |
|-----|----------|----------------|---------------------|
| P‑1 | Обновить версии **MDM API** до 1.3 (поддержка bulk‑load). | MDM‑Team | Тестовый стенд готов, API‑документация опубликована. |
| P‑2 | Сгенерировать **full‑backup** клиентских таблиц в FU CRM и RB CRM (encrypted, off‑site). | DB‑Admin | Проверка контрольных сумм (SHA‑256). |
| P‑3 | Настроить **CDC‑коннектор** (Debezium) → Kafka‑topic `client_changes`. | Integration‑Team | Тестовое событие проходит сквозную валидацию и публикуется в MDM. |
| P‑4 | Провести **dry‑run** миграции 1 % записей в тестовом MDM‑окружении. | Data‑Quality | Ошибки > 0,5 % → откат и корректировка правил дедупликации. |

## Data Freeze

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

## Переключение

1. **Обновление маршрутизации** в **ESB**: все запросы к `/customers/*` перенаправляются к **MDM API** (blue‑green деплой, 5‑минутный откат).  
2. **Обновление клиентских приложений** (Web, Mobile) – изменить endpoint в конфигурации (CI‑pipeline).  
3. **Начало обработки новых событий** через **Kafka** → MDM → Core.  

## Фолбэк

* **Trigger:** любые несоответствия в проверке `Reconciliation` > 0,5 % или падение API > 5 минут подряд.  
* **Действия:**
  1. Отключить новые маршруты в ESB, вернуть старый `CRM‑endpoint`.
  2. Переключить `read‑only` режим обратно в `read‑write` для обеих CRM.
  3. Восстановить **backup** клиентских данных в обеих CRM (по контрольным суммам).
  4. Запустить **incident‑response** процесс (Data‑Office).  

## Мониторинг

| KPI | Порог | Инструмент |
|-----|-------|------------|
| **API latency** (MDM) | ≤ 150 ms (95‑th perc) | Prometheus + Grafana |
| **Error rate** (HTTP 5xx) | ≤ 0,1 % | Loki/Alertmanager |
| **CDC lag** (Kafka → MDM) | ≤ 2 сек | Confluent Control Center |
| **Data freshness** (Core → MDM) | ≤ 5 мин | Airflow DAG `customer_sync_freshness` |
| **Business KPI drift** (Active accounts) | ≤ 0,5 % | Looker dashboard “Customer Health” |

> **SLA post‑cutover:** 99,9 % доступности клиентских данных, отсутствие критических ошибок более 5 минут.
