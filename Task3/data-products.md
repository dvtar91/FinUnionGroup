# Доменные data‑продукты (Data Mesh‑подход)

| Data product | Домен (BIAN) | Основные данные | Владелец | Потребители | Как публикуется |
|--------------|--------------|-----------------|----------|-------------|-----------------|
| **Customer 360** | Party Data Management | `client_id`, персональные данные, контакты, сегменты, согласия | Data Office / MDM‑Team | CRM, BI, Marketing, Risk‑Management | Таблица `dim_customer` в Lakehouse + метаданные в DataHub |
| **Credit‑Risk Profile** | Credit Management | История кредитов, платежные графики, скоринг, просрочки | Risk‑Management Office | Risk‑Analytics, AML‑Engine, BI | View `vw_credit_risk` в PostgreSQL DWH (агрегаты) + DataHub |
| **Product Catalog** | Product Management | Продукты, тарифы, каналы продаж, версии | Product‑Owner (FinUnion) | CRM, Core Banking, Card Processing, Marketing | Таблица `dim_product` в Lakehouse, версии в Iceberg |
| **AML Event Stream** | Counterparty Management / Anti‑Fraud | Транзакции с высоким риском, сигналы, черные списки | AML‑Team | Fraud‑Engine, Risk‑Analytics, Compliance | Kafka topic `aml_events` + Trino‑view `aml_events_view` |
| **Channel Performance** | Channel Management | Метрики по каналам (кол‑во заявок, SLA, конверсия) | CX‑Office | BI, Marketing, Executive Board | DWH‑таблица `fact_channel_metrics`, DataHub lineage |

### Механизм публикации

1. **Создание** – каждая доменная команда владеет набором ETL/ELT‑процессов, публикует таблицу/вью в **Lakehouse**.  
2. **Метаданные** – автоматически импортируются в **DataHub** через Kafka‑инжекторы (schema‑registry, lineage).  
3. **Контракт** – в DataHub задаётся JSON‑Schema и SLA (refresh‑rate, latency). Потребители подключаются к **Trino** либо к **PostgreSQL DWH** (для агрегированных представлений).  
