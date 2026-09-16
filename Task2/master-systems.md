# Мастер‑системы

| № | Сущность | Источники As‑Is | Мастер‑система (через 3 мес.) | Целевая мастер‑система | BIAN‑домены (помощь в границах) | Обоснование |
|---|----------|------------------|-------------------------------|------------------------|---------------------------------|--------------|
| 1 | **Клиент (Party)** | FU CRM, RB CRM | **MDM Customer Hub** (центр золотой записи) | MDM Customer Hub | *Party Data Management* | Требуется единый клиентский идентификатор, дедупликация и согласованность профилей. |
| 2 | **Счёт (Account)** | FU Core, RB Core | **Core Banking FinUnion** (главный OLTP‑хост) | Core Banking (унифицированный, мульти‑тенант) | *Current Account* | **RetailBank остаётся владельцем** всех **не‑мигрированных** счётов. Эти записи продолжают обслуживаться RB‑ядром, а только новые/мигрированные объекты управляются единым **Core Banking.**|
| 3 | **Карта (Card)** | FU Cards, RB Cards | **Card Processing FinUnion** | Card Processing (унифицированный) | *Card Management* |**RetailBank остаётся владельцем** всех **не‑мигрированных** карт. Эти записи продолжают обслуживаться RB‑ядром, а только новые/мигрированные объекты управляются единым **Card Processing.**  |
| 4 | **Кредит (Loan)** | FU Loans, RB Loans | **Loan System FinUnion** | Loan System (унифицированный) | *Loan Management* | **RetailBank остаётся владельцем** всех **не‑мигрированных** кредитов. Эти записи продолжают обслуживаться RB‑ядром, а только новые/мигрированные объекты управляются единым **Loan System.** |
| 5 | **Договор (Agreement)** | FU Core, RB Core | **Core Banking FinUnion** (внутри продукта) | Core Banking (мульти‑продукт) | *Product Fulfilment* | Договоры связывают клиент‑продукт; хранится в ядре. |
| 6 | **Транзакция (Transaction)** | FU Core, RB Core, FU Cards, RB Cards | **Core Banking FinUnion** (операционный журнал) | Core Banking (операции) | *Payments Execution* | Транзакции – фундаментальная операционная сущность. |
| 7 | **Продукт (Product Catalog)** | FU CRM, RB CRM | **MDM Product Hub** (справочники) | MDM Product Hub | *Product Management* | Справочники продуктов и тарифов должны быть согласованы в MDM. |
| 8 | **Обращение (Customer Interaction)** | FU Contact‑Center, RB Contact‑Center | **Integration Layer → DWH** (синхронный push) | DWH (исторические обращения) | *Customer Contact* | Обращения в аналитике, но не требуют отдельного OLTP‑хранилища. |
| 9 | **Контрагент (Counterparty)** | FU Core, RB Core | **MDM Counterparty Hub** | MDM Counterparty Hub | *Counterparty Management* | Требуется единый реестр контрагентов для AML/Fraud. |
|10 | **Документ (Document)** | FU DocumentMgmt, RB DocumentMgmt | **Object Store (S3‑like)** | Object Store (каталог) | *Document Management* | Хранилище файлов, к нему привязываются ссылки из MDM и Core. |
|11| **Тариф/Комиссия** | FU CRM, RB CRM | **MDM Product Hub** (часть справочников) | MDM Product Hub | *Pricing Management* | Тарифы меняются редко, но требуют контроля версий. |
|12| **Отделение / Канал продаж** | FU CRM, RB CRM | **MDM Channel Hub** | MDM Channel Hub | *Channel Management* | Упрощает маршрутизацию клиентских заявок. |
