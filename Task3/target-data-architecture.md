# Целевая архитектура данных (12 мес.)

### Компоненты

| Категория | Компонент | Краткое назначение | Технология (пример) |
|-----------|-----------|----------------------|----------------------|
| **Operational Systems** | Core Banking (FinUnion) | OLTP‑ядро, счета, договоры, операции | Oracle / PostgreSQL‑core |
| | Core Banking (Retail) – постепенно выводится | Переход к унифицированному ядру | — |
| | CRM (FinUnion + Retail) | Управление клиентской информацией | Salesforce‑like / собственный |
| | Card Processing (FinUnion) | Выпуск и обработка карт, события | Visa/MC‑compatible |
| | Loan System (FinUnion) | Управление кредитным портфелем | Custom loan‑engine |
| | Document Store | Хранение сканов договоров, паспортов | S3‑compatible Object Store |
| **MDM** | Customer Hub | “Золотая запись” клиента, дедупликация | Informatica MDM / Profisee |
| | Product & Counterparty Hub | Справочники продуктов, тарифов, контрагентов | Same MDM platform |
| **Integration Layer** | Enterprise Service Bus (ESB) | Синхронный API‑шлюз (REST/SOAP) | MuleSoft / WSO2 |
| | Event Bus (Kafka) | Асинхронные события (CDC, бизнес‑события) | Apache Kafka (kRaft) |
| | Mapping & Validation Service | Правила трансформации, очистка | KSQL / Kafka Streams |
| **Data Lakehouse** | Data Lake (Iceberg) | Хранилище сырых/партиционированных файлов | Apache Iceberg on S3 |
| | Query Engine | Универсальный SQL‑доступ к lakehouse | Trino (Presto) |
| **DWH** | Регламентированное хранилище | Финальная отчётность, агрегаты | PostgreSQL‑based DWH (Amazon Aurora) |
| **Orchestration** | Airflow | Планирование и мониторинг batch‑/ELT‑задач | Apache Airflow |
| **Catalog** | DataHub | Метаданные, lineage, владельцы, семантика | DataHub (LinkedIn) |
| **BI** | Витрины и дашборды | Управленческая, регуляторная аналитика | Looker / Power BI |
| **Security / Governance** | Centralized IAM, Data Masking, Encryption | Защита ПДн, контроль доступа | HashiCorp Vault, Apache Ranger |

### Поток данных

1. **CDC** из всех operational‑систем → **Kafka** → **MDM** (для клиентских данных) и **Lakehouse** (сырые события).  
2. **ELT** из **Lakehouse** → **DWH** (агрегаты, финансовые отчёты) через **Airflow** + **dbt**.  
3. **API‑Gateway** из **Integration Layer** → **CRM / Core** для синхронных запросов (баланс, статус карты).  
4. **DataHub** хранит метаданные обоих хранилищ, делает lineage доступным для BI и аудиторов.  

### Принципы

* **Непрерывность** – никакой downtime для клиентских сервисов. Все потоки работают в **near‑real‑time** через Kafka, а batch‑потоки – в‑ночные окна.  
* **Разделение OLTP / OLAP** – operational‑systems обслуживают транзакции, аналитика – только из DWH/Lakehouse.  
* **Data‑Governance** – единые политики шифрования, маскирования и RBAC реализованы в DataHub и в хранилищах.  

```mermaid

flowchart LR
    %% Операционные системы
    subgraph Ops["Operational Systems"]
        FU_Core[FinUnion Core Banking]
        RB_Core[RetailBank Core Banking]
        FU_CRM[FinUnion CRM]
        RB_CRM[RetailBank CRM]
        FU_Cards[FinUnion Card Processing]
        RB_Cards[RetailBank Card Processing]
        FU_Loans[FinUnion Loan System]
        RB_Loans[RetailBank Loan System]
        Docs[Document Store & Object Storage]
    end

    %% Интеграционный слой
    subgraph Integration["Integration Layer"]
        ESB[ESB / API Gateway]
        Kafka[Kafka]
    end

    %% Управление мастер‑данными и каталогом
    subgraph Governance["Governance"]
        MDM_Cust[MDM Customer Hub]
        MDM_Prod[MDM Product Hub]
        MDM_Counter[MDM Counterparty Hub]
        DataHub[DataHub]
    end

    %% Аналитическая платформа
    subgraph Analytics["Analytical Platform"]
        Lake["Data Lakehouse<br/>(Iceberg + Trino)"]
        DWH["Data Warehouse<br/>(PostgreSQL/Aurora)"]
        Airflow["Airflow (Orchestration)"]
        BI[BI / Dashboards]
        Reg["Regulator (External)"]
    end

    %% Потоки данных
    %% CRM → MDM
    FU_CRM -->|"CDC (JSON)" | MDM_Cust
    RB_CRM -->|"CDC (JSON)" | MDM_Cust

    %% MDM → Операционные ядра
    MDM_Cust -->|"API"| FU_Core
    MDM_Cust -->|"API"| RB_Core
    MDM_Cust -->|"API"| FU_Cards
    MDM_Cust -->|"API"| RB_Cards
    MDM_Cust -->|"API"| FU_Loans
    MDM_Cust -->|"API"| RB_Loans

    %% Справочники (продукты, контрагенты) → ядра
    MDM_Prod -->|API| FU_Core
    MDM_Prod -->|API| RB_Core
    MDM_Prod -->|API| FU_Cards
    MDM_Prod -->|API| RB_Cards
    MDM_Prod -->|API| FU_Loans
    MDM_Prod -->|API| RB_Loans
    MDM_Counter -->|CDC| FU_Core
    MDM_Counter -->|CDC| RB_Core

    %% Операционные системы → Kafka (CDC)
    FU_Core -->|CDC| Kafka
    RB_Core -->|CDC| Kafka
    FU_Cards -->|CDC| Kafka
    RB_Cards -->|CDC| Kafka
    FU_Loans -->|CDC| Kafka
    RB_Loans -->|CDC| Kafka

    %% Документы → DataHub (metadata events)
    Docs -->|metadata events| DataHub

    %% Kafka → Lakehouse (raw) и DWH (enriched)
    Kafka -->|raw events| Lake
    Kafka -->|enriched streams| DWH

    %% ESB (синхронные запросы) → ядра
    ESB -->|REST API| FU_Core
    ESB -->|REST API| RB_Core
    ESB -->|REST API| FU_Cards
    ESB -->|REST API| RB_Cards
    ESB -->|REST API| FU_Loans
    ESB -->|REST API| RB_Loans

    %% Airflow оркестрирует batch‑ETL
    Airflow -->|Batch / ELT| DWH
    Airflow -->|Batch / ELT| Lake

    %% DWH → BI и Regulator
    DWH -->|BI feeds| BI
    DWH -->|Export CSV| Reg

    %% DataHub → метаданные для аналитики
    DataHub -->|metadata feed| Airflow
    DataHub -->|metadata feed| BI
    DataHub -->|catalog updates| Lake
    DataHub -->|catalog updates| DWH

    %% Стиль
    classDef system fill:#c2e0ff,stroke:#555;
    classDef governance fill:#ffdd99,stroke:#555;
    classDef analytics fill:#dffecc,stroke:#555;
    class Ops,Integration system;
    class Governance governance;
    class Analytics analytics;
```
