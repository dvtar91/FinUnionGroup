# Логическая модель данных (ER‑диаграмма)

```mermaid
erDiagram
    CLIENT ||--o{ ACCOUNT : "владеет"
    CLIENT ||--o{ CARD : "владеет"
    CLIENT ||--o{ LOAN : "владеет"
    CLIENT ||--o{ AGREEMENT : "заключает"
    CLIENT ||--o{ INTERACTION : "инициирует"
    CLIENT ||--o{ DOCUMENT : "имеет"
    CLIENT }|..|{ COUNTERPARTY : "взаимодействует с"

    ACCOUNT ||--|{ TRANSACTION : "генерирует"
    CARD ||--|{ TRANSACTION : "генерирует"
    LOAN ||--|{ TRANSACTION : "погашение"
    AGREEMENT ||--|{ PRODUCT : "определяет тип"
    PRODUCT ||--o{ TARIFF : "имеет"
    PRODUCT ||--o{ CHANNEL : "продаётся через"
    INTERACTION ||--|{ CHANNEL : "через"
    DOCUMENT }|..|{ AGREEMENT : "привязан к"
    COUNTERPARTY ||--|{ TRANSACTION : "участвует в"

    %% Атрибуты (не отображаются в Mermaid, но указаны в описании сущностей)
