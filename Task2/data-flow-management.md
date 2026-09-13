# Управление потоками данных  

| № | Источник | Приёмник | Сущность | Действие | Формат | Частота | Тип обмена | SLA (доступность) | Владелец потока | Контроль качества |
|---|----------|----------|----------|----------|--------|---------|------------|-------------------|-----------------|-------------------|
| 1 | FU CRM / RB CRM | MDM Customer Hub | Клиент | Create/Update/Deduplicate | JSON (CDC) | Near real‑time (≈ 5 сек) | Async (Kafka) | 99,9 % доступность | Data Office – MDM Lead | Проверка дублей (по ИНН/ОГРН), валидация схемы |
| 2 | MDM Customer Hub | Core Banking (FinUnion) | Клиент‑ID | Read | REST/JSON | При запросе (sync) | Synchronous API | 99,5 % | Core Banking Owner | Сопоставление ID, отказ в случае несоответствия |
| 3 | MDM Product Hub | Core, Cards, Loans | Продукт / Тариф | Read | JSON (REST) | Nightly batch (02:00 UTC) | Batch | 99,0 % | Product Owner | Целостность справочника (уникальные коды) |
| 4 | Core Banking (FinUnion) | DWH | Счёт, Договор, Транзакция | Extract/Load | Avro (CDC) | Near real‑time (≈ 1 мин) | Streaming (Kafka) | 99,7 % | Data Engineering | Проверка баланса ≧ 0, отсутствие отрицательных сумм |
| 5 | Card Processing (FinUnion) | DWH | Карта, Карточные транзакции | Extract/Load | Avro (CDC) | Near real‑time | Streaming | 99,7 % | Card Ops | PAN‑токен, валидность дат |
| 6 | Loan System (FinUnion) | DWH | Кредит, Платёж по кредиту | Extract/Load | Avro (CDC) | Near real‑time | Streaming | 99,7 % | Loan Ops | Проверка лимита, процентной ставки |
| 7 | Contact‑Center (FU & RB) | DWH | Обращения | Batch load | CSV (gzip) | Hourly | Batch | 99,5 % | CX Lead | Обязательные поля, время отклика < 24 ч |
| 8 | DWH | BI | Все аналитические данные | ELT | Parquet | Daily (02:00 UTC) | Batch | 99,9 % | BI Lead | Схема BI‑модели, проверка null‑value |
| 9 | DWH | Regulator | Регуляторные отчёты | Export | CSV/JSON | Daily (09:00 UTC) | File Transfer (SFTP) | 99,9 % | Compliance Officer | Соответствие регламенту (формат, подписи) |
|10| Integration Layer | MDM Product Hub & MDM Channel Hub | Справочники | Enrich/Sync | REST/JSON | On‑change (event‑driven) | Async (Kafka) | 99,8 % | Integration Team | Версионность, проверка согласованности |
