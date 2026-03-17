# Confluent Dev Day 2026 — Dynamic Food Waste Reduction

*“Dreaming a better future is free, and designing data pipelines with Confluent is easy. Why not combine both?”*
---

## 🧭 Introduction

This application aims to **minimize food waste** in modern grocery stores using a **dynamic pricing model** powered by a **real-time data pipeline**. The system directly integrates with physical store operations — such as price-reduction sticker applications — to ensure synchronized digital and real-world decisions.

---
## 📈 The Problem

> In Switzerland alone, **260 kg of food is thrown away every second**.  https://uglyfruits.ch/en/food-waste-facts
Picture this: every 4 seconds, that’s the mass equivalent of a **Fiat New 500** going to waste.

This is not just a local issue. Switzerland is not even a "blip" in the radar of big countries. It’s a global symptom of inefficient stock management, rigid pricing strategies, and fluctuating consumer demand. In 2026, no more.

### Key Challenges for Grocery Retailers

- **Forecasting inaccuracies** — due to weather dependencies, trends and events.
- **High perishability** — especially for bakery, dairy, and meat products.
- **Rigid pricing models** — discounts are often too late or static.
- **Operational complexity** — staff rarely optimize continuously, leading to food waste.

### Consumer and Business Perspectives

- **Consumers (B2C):** Want cheaper groceries without compromising quality, and value sustainability. On the other hand, they also expect the widest possible choice available in stores.
- **Businesses (B2B):** Restaurants and small shops seek cost-effective supply, especially for short-term usage or bulk purchases.

---

## 🛠️ Technical Challenges

Integrating **heterogeneous data sources** is never easy, expecially for grocery stores.
Traditional batch-processing systems fail to respond quickly enough, making it impossible to react in real time.

This project leverages **Confluent's out-of-the-box components** and a **shift-left architecture** to enable:

- **Real-time decision-making**  
- **AI-driven analytics** for optimal pricing and waste reduction  
- **Event streaming pipelines** that connect physical operations with intelligent pricing logic and notification layers

---

## 🚀 Vision

By combining **Confluent’s streaming capabilities** with **AI-powered insights**, this solution helps grocery stores dynamically adjust prices, reduce waste, and align profit goals with sustainability values.

---

## 🧨 High Level Architecture

<img width="2180" height="1467" alt="image" src="https://github.com/user-attachments/assets/89338acd-0a0e-473c-b8f3-16fc6226a084" />

Components are divided in main categories:

- Main Data producers
- ML Models - AI
- Data Pipeline (Confluent Kafka -including topics and schema Registry- + Flink processing layer)
- Analytics / BI Pillar
- Digital App ecosystem

---

## ❇️ Use Cases

Time wise, I had the chance to focus only on one specific scenario, but in the high level architecture more and more are feasible. Starting from the vision API one for realtime monitoring the ripeness of certain fruits, forecasting sales based on interests for discounts on certain products, and so on.
Let's focus on details on one specific

### Use case 1 B2C - Apply 50% sticker on specific products when certain conditions applies

> Easter is near, and Zürich stores are filled with lamb meat, following forecasting ERP model adjusted on weather forecast.
> Weather was horrible, no grill events outside, leading to many packs on shelves.

Flink detects this pattern in certain stores in Zurich Area, surge of items close to expiration, and mark the product to be discounted.
Clerks of stores in Zurich area receive the task of putting 50% stickers on meat.
Customers got informed with a PUSH notification their favourite store is having a time limited promotion on meat (Store/Product are preference in app, leading subscription to specific topics).
**Bonus: B2B adoption** - offer for bundle purchase is offered to B2B customers in advance for a reduced discount before proceeding to whole base population.

#### Creation of Flink Table
```sql
CREATE TABLE sales_stream (
  transaction_id STRING,
  store_id STRING,
  product_id STRING,
  quantity_sold INT,
  unit_price FLOAT,
  total_price FLOAT,
  timestamp BIGINT,
  WATERMARK FOR timestamp AS timestamp - INTERVAL '15' SECOND -- not sure 15 sec is enough, tests?
) WITH (
  'connector' = 'kafka',
  'topic' = 'store.sales.transaction',
  'properties.bootstrap.servers' = '<server>',
  'format' = 'avro'
);
```
#### Pattern Detection with simple CEP Logic - Query for detect Overstock Pattern
```sql
CREATE VIEW overstock_detected AS
SELECT
  i.store_id,
  i.product_id,
  COUNT(*) AS inventory_events,
  SUM(i.quantity) AS total_stock,
  COALESCE(SUM(s.quantity_sold), 0) AS total_sales,
  AVG(e.days_to_expiry) AS avg_days_to_expiry,
  MAX(e.timestamp) AS event_time

FROM inventory_stream i

LEFT JOIN expiry_stream e
  ON i.product_id = e.product_idd
  AND i.store_id = e.store_id
LEFT JOIN sales_stream s
  ON i.product_id = s.product_id
  AND i.store_id = s.store_id
WHERE i.product_name LIKE '%lamb%' or LIKE '%pork%' or LIKE '%beef%' -- modify syntax TODO
GROUP BY i.store_id, i.product_id
HAVING
  total_stock > 50   -- assuming 50 as high inventory
  AND total_sales < 10      -- assumyning 10 as product in low demand
  AND avg_days_to_expiry < 3;    -- Assuming 3 as near expiry for meat - to be parametrized
```
#### Generate 50% Discount for matched items
```sql
INSERT INTO dynamic_pricing
SELECT
  UUID(),
  store_id,
  product_id,
  x AS original_price,
  50.0 AS discount_percentage,
  y AS new_price,
  'EASTER_LAMB_OVERSTOCK' AS reason, -- to be parametrized (GEN AI dynamic TODO ?)
  event_time

FROM overstock_detected;
```
####  Trigger now the Store Action (Apply the sticker)
```sql
INSERT INTO sticker_notifications
SELECT
  UUID(),
  store_id,
  product_id,
  50.0,
  'HIGH',
  'URGENT: Apply 50% sticker on lamb meat',
  event_time
FROM overstock_detected;
```
#### B2C Push Notification (Pub/Sub 🔥)
```sql
INSERT INTO recommendation_events
SELECT
  UUID(),
  sub.subscriber_id AS user_id,
  NULL,
  dp.store_id,
  dp.product_id,
  0.95 AS score, --tweak this for B2B to adjust priority
  dp.timestamp

FROM dynamic_pricing dp
JOIN store_subscriptions sub
ON dp.store_id = sub.store_id

WHERE dp.discount_percentage = 50.0;
```

Flink has been used to detect patterns like overstocked lamb meat before Easter, combining inventory, sales, and expiry data in real time.
The system automatically triggers dynamic pricing, store staff actions and personalized notifications to consumers

---
*Let’s make real-time data work for both people and the planet.*
