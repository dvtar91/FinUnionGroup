# Пайплайны данных

## Таблица основных пайплайнов

| № | Пайплайн | Источник | Приёмник | Тип | Инструменты | Краткое описание |
|---|----------|----------|----------|----|--------------|-------------------|
| 1 | **Client‑MDM Sync** | FU CRM, RB CRM | MDM Customer Hub | CDC (Change Data Capture) | Kafka Connect → Debezium → dbt (transform) → MDM API | Дедупликация, обогащение, создание единого `client_id`. |
| 2 | **Core → DWH (Reg‑Reporting)** | FU Core, RB Core | PostgreSQL DWH | Batch (nightly) | Airflow → Sqoop/Copy → dbt | Выгрузка счетов, договоров, финансовых агрегатов для регуляторов. |
| 3 | **Card Events → Lakehouse** | FU Cards, RB Cards | Iceberg Lake (S3) | Streaming (CDC) | Kafka → Kafka Streams (enrichment) → Iceberg writer (Spark) | Сохранение всех карточных операций в партиционированных файлах. |
| 4 | **Loan Portfolio → Lakehouse** | FU Loans, RB Loans | Iceberg Lake (S3) | Streaming (CDC) | Kafka → Flink → Iceberg | Исторические изменения кредитного портфеля, версии договоров. |
| 5 | **CRM → BI‑Cache** | MDM Customer Hub, Product Hub | Redis cache (fast lookup) | Near‑real‑time | Kafka → ksqlDB → Redis | Быстрый доступ к клиентским профилям в онлайн‑каналах. |
| 6 | **Document Ingestion** | Document Management System | Object Store (S3) | Batch (hourly) | Airflow → S3 sync | Перенос сканов договоров, подписи, метаданные в объектное хранилище. |
| 7 | **DataHub Metadata Sync** | All services (MDM, Lake, DWH) | DataHub catalog | Event‑driven | Kafka → DataHub ingest plugin | Автоматическое обновление lineage и владельцев. |
| 8 | **Risk‑Scoring (AML)** | Transaction stream (Lake) | AML Engine (real‑time) | Streaming | Kafka Streams → Python ML model (SageMaker) | Оценка риска в режиме реального времени, обратная связь в MDM. |

## Сценарии обработки больших исторических / событийных данных

### Сценарий A – История карточных операций (10 млн транзакций/день)

| Параметр | Значение |
|----------|----------|
| **Источник** | Карточный процессинг (FinUnion Cards + RetailBank Cards) |
| **Характер данных** | Событийные, полуструктурированные (JSON), ~1 TB/мес. |
| **Целевое хранилище** | Data Lakehouse (Iceberg, Parquet) |
| **Способ обработки** | **Streaming**: Kafka → Flink (чистка, обогащение) → Iceberg writer (partition‑by `event_date`, `card_type`) |
| **Потребители** | BI‑аналитика (fraud detection), Risk‑Management (AML), Marketing (поведенческий анализ) |
| **Дополнительные шаги** | Nightly `dbt`‑модель для агрегаций (daily spend, top‑merchant) → DWH (PostgreSQL) для регулятора. |

### Сценарий B – Архивные выгрузки после слияния (historical migration)

| Параметр | Значение |
|----------|----------|
| **Источник** | Экспортированные CSV/AVRO из legacy‑DWH RetailBank (данные 2015‑2022) |
| **Характер данных** | Архивные, структурированные, 5 TB |
| **Целевое хранилище** | Data Lakehouse (Iceberg) + отдельный **archive**‑каталог |
| **Способ обработки** | **Batch**: Airflow DAG → Spark job (schema‑validation, dedup, partition‑by `year`) → Iceberg |
| **Потребители** | Data Science (historical models), Compliance (audit), Business Intelligence (historical trend analysis) |
| **Контроль качества** | Сравнение контрольных сумм, проверка уникальности `account_id`, запись в DataHub. |

## Пример DAG в Airflow

```python
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.providers.amazon.aws.transfers.s3_to_s3 import S3ToS3Operator
from datetime import datetime, timedelta

default_args = {
    "owner": "data-engineering",
    "depends_on_past": False,
    "retries": 1,
    "retry_delay": timedelta(minutes=5),
    "start_date": datetime(2024, 10, 1),
}

with DAG(
    dag_id="loan_portfolio_cdc_to_lakehouse",
    schedule_interval="@hourly",
    default_args=default_args,
    catchup=False,
    tags=["cdc", "lakehouse"],
) as dag:

    # Pull CDC records from Kafka (using Spark Structured Streaming)
    spark_stream = SparkSubmitOperator(
        task_id="spark_cdc_loan_stream",
        application="/opt/spark/jobs/loan_cdc_to_iceberg.py",
        conf={"spark.master": "yarn"},
        jars="/opt/spark/jars/iceberg-spark3-runtime.jar",
        driver_memory="4g",
        executor_memory="8g",
        name="loan_cdc_stream",
    )

    # Optional: validate with dbt after each batch (run-once per hour)
    dbt_test = BashOperator(
        task_id="dbt_test_loan",
        bash_command="dbt test -m loans --profiles-dir /opt/dbt/profiles",
    )

    spark_stream >> dbt_test
```
