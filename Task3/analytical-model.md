# Аналитическая модель (Star‑schema) – Область «Транзакции»

## Выбор схемы

Для аналитических запросов, связанных с финансовыми операциями, схема **звёза**  обеспечивает:

* Быстрый доступ к фактам через небольшое количество JOIN‑ов.  
* Хорошую поддержку агрегатов (SUM, COUNT, AVG).  
* Простоту построения BI‑витрин (группировки по датам, клиентам, продуктам).  

Для справочников (продукты, контрагенты) используется схема **снезжинка** – нормализованные измерения, позволяющие уменьшить дублирование и упростить управление справочниками.

## Точки фактов и измерений

| Тип | Таблица | Описание | Ключевые поля |
|-----|---------|----------|---------------|
| **Fact** | `fact_transactions` | Все финансовые операции (платежи, снятие, перевод). | `transaction_id`, `account_id`, `card_id`, `product_id`, `date_key`, `amount`, `currency_code`, `status` |
| **Dimension** | `dim_date` | Дата/время, календарные атрибуты. | `date_key`, `date`, `day_of_week`, `month`, `quarter`, `year`, `holiday_flag` |
| **Dimension** | `dim_customer` | Информация о клиенте (из MDM). | `customer_id`, `party_id`, `segment`, `region`, `risk_score`, `birth_date`, `gender` |
| **Dimension** | `dim_account` | Счёт клиента. | `account_id`, `account_number`, `account_type`, `currency`, `open_date`, `status` |
| **Dimension** | `dim_product` | Продукт (карта, кредит, вклад). | `product_id`, `product_type`, `product_name`, `tariff_id`, `category` |
| **Dimension** | `dim_channel` | Канал взаимодействия (offline, mobile, web). | `channel_id`, `channel_name`, `region` |
| **Dimension** | `dim_merchant` | Контрагент‑торговец (для карт). | `merchant_id`, `merchant_name`, `category_code`, `country` |
