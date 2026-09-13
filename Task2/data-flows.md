# Диаграмма потоков данных (уровень 0) – промежуточное состояние (3 мес.)

## Описание потоков

| № | Источник | Приёмник | Сущность | Действие | Тип обмена | Обоснование |
|---|----------|----------|----------|----------|------------|-------------|
| 1 | **FU CRM** / **RB CRM** | **MDM Customer Hub** | Клиент | Create / Update / Deduplicate | CDC (Change Data Capture) → Near‑real‑time | Консолидировать профиль клиента |
| 2 | **MDM Customer Hub** | **Core Banking FinUnion** | Клиент‑ID в договорах, счетах | Read | Sync (API) | Обеспечить единый клиент‑идентификатор в ядре |
| 3 | **MDM Product Hub** | **Core Banking**, **Card Processing**, **Loan System** | Продукт, Тариф | Read | Batch (nightly) | Синхронные справочники продуктов |
| 4 | **Core Banking FinUnion** | **MDM Counterparty Hub** | Контрагент | Create / Update | CDC | Актуальный реестр контрагентов для AML |
| 5 | **Core Banking FinUnion** → **DWH** | — | Счёт, Договор, Транзакция | Extract / Load | CDC + nightly batch | Формировать аналитические витрины |
| 6 | **Card Processing FinUnion** → **DWH** | — | Карта, Карточные транзакции | CDC | Near‑real‑time | Требуется для фрод‑аналитики |
| 7 | **Loan System FinUnion** → **DWH** | — | Кредит, Платёж по кредиту | CDC | Near‑real‑time | Отчётность по кредитному портфелю |
| 8 | **Contact‑Center (FU & RB)** → **DWH** | — | Обращения | Batch (hourly) | Сохранять историю клиентского опыта |
| 9 | **DWH** → **BI** | — | Все аналитические сущности | ELT (batch) | Для построения отчётных витрин |
|10| **DWH** → **Regulator** (внешний) | — | Регуляторные отчёты (по счетам, кредитам, AML) | Export (CSV/JSON) | Периодическая (ежедневно) передача |
|11| **Integration Layer** (ESB) | **MDM Product Hub** & **MDM Channel Hub** | Справочники (продукты, каналы) | Enrich / Sync | API (REST) | Поддерживать актуальные справочники |
|12| **MDM Customer Hub** → **Integration Layer** | **FI‑Reporting** | Профиль клиента (универсальный ID) | Publish/Subscribe (Kafka) | Доступ для новых сервисов (мобайл, онлайн) |

## Mermaid‑диаграмма (DFD уровень 0)

```mermaid
flowchart LR
    %% Системы
    subgraph "FinUnion"
        FU_CRM[FinUnion CRM]
        FU_Core[FinUnion Core Banking]
        FU_Cards[FinUnion Card Processing]
        FU_Loans[FinUnion Loan System]
        FU_Contact[FinUnion Contact‑Center]
    end

    subgraph "RetailBank"
        RB_CRM[RetailBank CRM]
        RB_Core[RetailBank Core Banking]
        RB_Cards[RetailBank Card Processing]
        RB_Loans[RetailBank Loan System]
        RB_Contact[RetailBank Contact‑Center]
    end

    MDM_Cust[MDM Customer Hub]
    MDM_Prod[MDM Product Hub]
    MDM_Counter[MDM Counterparty Hub]
    Integration["Integration Layer (ESB / Kafka)"]
    DWH[Data Warehouse]
    BI[BI‑Витрины]
    Reg["Regulator (External)"]

    %% Потоки
    FU_CRM -->|"CDC (JSON)"| MDM_Cust
    RB_CRM -->|"CDC (JSON)"| MDM_Cust
    MDM_Cust -->|API| FU_Core
    MDM_Cust -->|API| FU_Cards
    MDM_Cust -->|API| FU_Loans

    FU_Core -->|CDC| DWH
    RB_Core -->|CDC| DWH
    FU_Cards -->|CDC| DWH
    RB_Cards -->|CDC| DWH
    FU_Loans -->|CDC| DWH
    RB_Loans -->|CDC| DWH
    FU_Contact -->|Batch| DWH
    RB_Contact -->|Batch| DWH

    DWH -->|ELT| BI
    DWH -->|Export CSV| Reg

    MDM_Prod -->|API| FU_Core
    MDM_Prod -->|API| FU_Cards
    MDM_Prod -->|API| FU_Loans
    MDM_Prod -->|API| RB_Core
    MDM_Prod -->|API| RB_Cards
    MDM_Prod -->|API| RB_Loans

    FU_Core -->|CDC| MDM_Counter
    RB_Core -->|CDC| MDM_Counter

    Integration -->|Publish| MDM_Cust
    Integration -->|Publish| MDM_Prod
    Integration -->|Publish| MDM_Counter

    style FU_CRM fill:#b7c78f,stroke:#555
    style RB_CRM fill:#b7c78f,stroke:#555
    style MDM_Cust fill:#ffdd99,stroke:#555
    style MDM_Prod fill:#ffdd99,stroke:#555
    style DWH fill:#c2e0ff,stroke:#555
    style BI fill:#c2e0ff,stroke:#555
    style Reg fill:#f2c2c2,stroke:#555
```
