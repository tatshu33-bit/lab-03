# Лабораторна робота 03. Модифікація даних та транзакції

**Студент:** Кондатюк Дмитро студент групи ІПЗ-33
**База даних:** «ТехноМарт» (PostgreSQL)
**Файл зі скриптом:** `lab03_technomart.sql`

## Мета

Опанувати безпечну модифікацію даних, транзакції, обмеження цілісності, каскадні операції, тригери та блокування на прикладі БД інтернет-магазину «ТехноМарт».

> Усі результати нижче отримано реальним виконанням `lab03_technomart.sql` на БД «ТехноМарт» (PostgreSQL 16).
>
> Лабораторну виконано на наданій БД «ТехноМарт», а не на бібліотечній. Відповідність сутностей: авторі/книги → категорії/товари, читачі → клієнти, видачі → замовлення та їх позиції, бронювання книг → бронювання товарів.

---

## Рівень 1

### 1.1 Структура та обмеження
Додано обмеження: `chk_orders_status`, `chk_orders_shipped`, `chk_customers_type`, `chk_customers_email`. Разом з наявними PK, FK, UNIQUE, CHECK це дає повну посилальну та доменну цілісність.

**Запит:**

```sql
-- Лабораторна робота 03. Модифікація даних та транзакції
-- База даних: "ТехноМарт" (PostgreSQL 14+ / Supabase)
-- Передумова: виконано скрипт створення БД "ТехноМарт".
--
-- ВАЖЛИВО: у psql / pgAdmin / DBeaver скрипт можна запускати блоками.
-- Рядки з позначкою  -- [ПОМИЛКА]  ПЕРЕДБАЧЕНО: вони навмисно викликають
-- помилку (для демонстрації обмежень). У Supabase SQL Editor кожен такий
-- блок запускайте окремо, бо помилка зупиняє весь запуск.
-- Основні таблиці вже існують (PK, FK, UNIQUE, CHECK). Додаємо ще правила.
ALTER TABLE orders    ADD CONSTRAINT chk_orders_status
    CHECK (order_status IN ('pending','processing','shipped','delivered','cancelled'));
ALTER TABLE orders    ADD CONSTRAINT chk_orders_shipped
    CHECK (shipped_date IS NULL OR shipped_date >= order_date);
ALTER TABLE customers ADD CONSTRAINT chk_customers_type
    CHECK (customer_type IN ('individual','company'));
ALTER TABLE customers ADD CONSTRAINT chk_customers_email
    CHECK (email ~* '^[^@\s]+@[^@\s]+\.[^@\s]+$');
-- Перелік усіх обмежень для звіту
SELECT conrelid::regclass AS table_name, conname, contype, pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE connamespace = 'public'::regnamespace AND contype IN ('f','c','u')
ORDER BY 1, 3, 2;
```

**Результат виконання:**

```text
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
 table_name  |               conname               | contype |                                                                                                     definition
-------------+-------------------------------------+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 regions     | regions_region_code_key             | u       | UNIQUE (region_code)
 suppliers   | suppliers_region_id_fkey            | f       | FOREIGN KEY (region_id) REFERENCES regions(region_id)
 products    | chk_products_price_positive         | c       | CHECK ((unit_price >= (0)::numeric))
 products    | chk_products_stock_positive         | c       | CHECK ((units_in_stock >= 0))
 products    | products_category_id_fkey           | f       | FOREIGN KEY (category_id) REFERENCES categories(category_id)
 products    | products_supplier_id_fkey           | f       | FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id)
 employees   | employees_region_id_fkey            | f       | FOREIGN KEY (region_id) REFERENCES regions(region_id)
 employees   | employees_reports_to_fkey           | f       | FOREIGN KEY (reports_to) REFERENCES employees(employee_id)
 customers   | chk_customers_email                 | c       | CHECK (((email)::text ~* '^[^@\s]+@[^@\s]+\.[^@\s]+$'::text))
 customers   | chk_customers_type                  | c       | CHECK (((customer_type)::text = ANY ((ARRAY['individual'::character varying, 'company'::character varying])::text[])))
 customers   | customers_region_id_fkey            | f       | FOREIGN KEY (region_id) REFERENCES regions(region_id)
 customers   | customers_email_key                 | u       | UNIQUE (email)
 orders      | chk_orders_dates                    | c       | CHECK ((required_date >= order_date))
 orders      | chk_orders_shipped                  | c       | CHECK (((shipped_date IS NULL) OR (shipped_date >= order_date)))
 orders      | chk_orders_status                   | c       | CHECK (((order_status)::text = ANY ((ARRAY['pending'::character varying, 'processing'::character varying, 'shipped'::character varying, 'delivered'::character varying, 'cancelled'::character varying])::text[])))
 orders      | orders_customer_id_fkey             | f       | FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
 orders      | orders_employee_id_fkey             | f       | FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
 orders      | orders_ship_region_id_fkey          | f       | FOREIGN KEY (ship_region_id) REFERENCES regions(region_id)
 order_items | chk_order_items_discount_valid      | c       | CHECK (((discount >= (0)::numeric) AND (discount <= (1)::numeric)))
 order_items | chk_order_items_price_positive      | c       | CHECK ((unit_price >= (0)::numeric))
 order_items | chk_order_items_quantity_positive   | c       | CHECK ((quantity > 0))
 order_items | order_items_order_id_fkey           | f       | FOREIGN KEY (order_id) REFERENCES orders(order_id)
 order_items | order_items_product_id_fkey         | f       | FOREIGN KEY (product_id) REFERENCES products(product_id)
 order_items | order_items_order_id_product_id_key | u       | UNIQUE (order_id, product_id)
(24 rows)
```

### 1.2 INSERT
Виконано одиночну та множинну вставку з `RETURNING`, вставку `INSERT ... SELECT` (таблиця `products_archive`). Перевірено відхилення некоректних даних: дубль email (UNIQUE), від'ємна ціна (CHECK), неіснуюча категорія (FK).

**Запит:**

```sql
-- Одиночний INSERT з RETURNING
INSERT INTO categories (category_name, description, picture_url)
VALUES ('Електротранспорт', 'Електросамокати, гіроскутери, електровелосипеди', 'https://example.com/e-transport.jpg')
RETURNING category_id, category_name;
```

**Результат виконання:**

```text
 category_id |  category_name
-------------+------------------
          11 | Електротранспорт
(1 row)

INSERT 0 1
```

**Запит:**

```sql
-- Множинний INSERT
INSERT INTO products (product_name, supplier_id, category_id, quantity_per_unit,
                      unit_price, units_in_stock, units_on_order, reorder_level, description)
VALUES
  ('Електросамокат Xiaomi Mi Scooter 4', 3,
   (SELECT category_id FROM categories WHERE category_name = 'Електротранспорт'),
   '1 шт в коробці', 21999.00, 14, 6, 4, 'Складний електросамокат, запас ходу 35 км'),
  ('ТЕСТ: Гіроскутер Hoverboard Pro', 1,
   (SELECT category_id FROM categories WHERE category_name = 'Електротранспорт'),
   '1 шт в коробці', 6999.00, 9, 0, 3, 'Тестовий товар для демонстрації видалення')
RETURNING product_id, product_name, unit_price;
```

**Результат виконання:**

```text
 product_id |            product_name            | unit_price
------------+------------------------------------+------------
         26 | Електросамокат Xiaomi Mi Scooter 4 |   21999.00
         27 | ТЕСТ: Гіроскутер Hoverboard Pro    |    6999.00
(2 rows)

INSERT 0 2
```

**Запит:**

```sql
-- Новий клієнт
INSERT INTO customers (company_name, contact_name, contact_title, address, city, region_id,
                       postal_code, phone, email, customer_type)
VALUES ('ТОВ "Нова Хвиля"', 'Захарченко Богдан Ігорович', 'Директор', 'вул. Шота Руставелі, 10', 'Київ',
        25, '01001', '+380441112233', 'zakharchenko@novakhvylia.ua', 'company')
RETURNING customer_id, contact_name;
```

**Результат виконання:**

```text
 customer_id |        contact_name
-------------+----------------------------
          16 | Захарченко Богдан Ігорович
(1 row)

INSERT 0 1
```

**Запит:**

```sql
-- INSERT ... SELECT: архів дешевих товарів
CREATE TABLE products_archive (LIKE products INCLUDING DEFAULTS, archived_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
INSERT INTO products_archive
SELECT p.*, CURRENT_TIMESTAMP FROM products p WHERE p.unit_price < 1000;
SELECT product_id, product_name, unit_price, archived_at FROM products_archive;
```

**Результат виконання:**

```text
CREATE TABLE
INSERT 0 1
 product_id |       product_name       | unit_price |        archived_at
------------+--------------------------+------------+----------------------------
         21 | Зарядний кабель USB-C 2м |     699.00 | 2026-11-08 14:10:41.890033
(1 row)
```

**Запит:**

```sql
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: порушення UNIQUE (дубль email)
BEGIN;
INSERT INTO customers (contact_name, email) VALUES ('Дублікат', 'petrov.ivan@gmail.com');
ROLLBACK;
```

**Результат виконання:**

```text
BEGIN
ERROR:  duplicate key value violates unique constraint "customers_email_key"
DETAIL:  Key (email)=(petrov.ivan@gmail.com) already exists.
ROLLBACK
```

**Запит:**

```sql
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: порушення CHECK (від'ємна ціна)
BEGIN;
INSERT INTO products (product_name, unit_price) VALUES ('Помилковий товар', -10);
ROLLBACK;
```

**Результат виконання:**

```text
BEGIN
ERROR:  new row for relation "products" violates check constraint "chk_products_price_positive"
DETAIL:  Failing row contains (28, Помилковий товар, null, null, null, -10.00, 0, 0, 0, f, null, null).
ROLLBACK
```

**Запит:**

```sql
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: порушення FOREIGN KEY (неіснуюча категорія)
BEGIN;
INSERT INTO products (product_name, category_id, unit_price) VALUES ('Сирота', 9999, 100);
ROLLBACK;
```

**Результат виконання:**

```text
BEGIN
ERROR:  insert or update on table "products" violates foreign key constraint "products_category_id_fkey"
DETAIL:  Key (category_id)=(9999) is not present in table "categories".
ROLLBACK
```

### 1.3 UPDATE
Кожне оновлення виконується за схемою SELECT → UPDATE із `WHERE` → перевірка (`RETURNING`). Використано оновлення за умовою, кількох полів, `UPDATE ... FROM` та захисну умову `units_in_stock > 0`. Спроба зробити залишок від'ємним відхилена CHECK.

**Запит:**

```sql
-- Перевіряємо, які рядки буде змінено
SELECT product_id, product_name, unit_price FROM products WHERE category_id = 8;
```

**Результат виконання:**

```text
 product_id |              product_name              | unit_price
------------+----------------------------------------+------------
         20 | Навушники Apple AirPods Pro 2          |    8999.00
         21 | Зарядний кабель USB-C 2м               |     699.00
         22 | Бездротова клавіатура Logitech MX Keys |    3999.00
(3 rows)
```

**Запит:**

```sql
-- Підвищення цін на аксесуари на 5 %
UPDATE products
SET unit_price = ROUND(unit_price * 1.05, 2)
WHERE category_id = 8
RETURNING product_id, product_name, unit_price;
```

**Результат виконання:**

```text
 product_id |              product_name              | unit_price
------------+----------------------------------------+------------
         20 | Навушники Apple AirPods Pro 2          |    9448.95
         21 | Зарядний кабель USB-C 2м               |     733.95
         22 | Бездротова клавіатура Logitech MX Keys |    4198.95
(3 rows)

UPDATE 3
```

**Запит:**

```sql
-- Оновлення кількох полів одночасно
UPDATE customers
SET phone = '+380501112233', city = 'Львів', region_id = 2
WHERE email = 'zakharchenko@novakhvylia.ua'
RETURNING customer_id, phone, city;
```

**Результат виконання:**

```text
 customer_id |     phone     | city
-------------+---------------+-------
          16 | +380501112233 | Львів
(1 row)

UPDATE 1
```

**Запит:**

```sql
-- UPDATE ... FROM: поповнення замовлення для товарів із малим залишком
UPDATE products p
SET units_on_order = p.reorder_level * 2
FROM suppliers s
WHERE p.supplier_id = s.supplier_id
  AND p.units_in_stock <= p.reorder_level + 1      -- майже вичерпані товари
RETURNING p.product_id, p.product_name, p.units_in_stock, p.units_on_order;
```

**Результат виконання:**

```text
 product_id |    product_name     | units_in_stock | units_on_order
------------+---------------------+----------------+----------------
         23 | PlayStation 5 825GB |              3 |              4
(1 row)

UPDATE 1
```

**Запит:**

```sql
-- Захист умовою: залишок не піде в мінус
UPDATE products
SET units_in_stock = units_in_stock - 1
WHERE product_id = 1 AND units_in_stock > 0
RETURNING product_id, units_in_stock;
```

**Результат виконання:**

```text
 product_id | units_in_stock
------------+----------------
          1 |             24
(1 row)

UPDATE 1
```

**Запит:**

```sql
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: CHECK (units_in_stock >= 0)
BEGIN;
UPDATE products SET units_in_stock = units_in_stock - 1000 WHERE product_id = 1;
ROLLBACK;
```

**Результат виконання:**

```text
BEGIN
ERROR:  new row for relation "products" violates check constraint "chk_products_stock_positive"
DETAIL:  Failing row contains (1, iPhone 15 128GB Чорний, 1, 1, 1 шт в коробці, 29999.00, -976, 10, 5, f, Новий iPhone 15 з потужним процесором A1..., https://example.com/iphone15.jpg).
ROLLBACK
```

### 1.4 DELETE
Видалено лише товар без замовлень (`NOT EXISTS`). Видалення проданого товару відхилено зовнішнім ключем.

**Запит:**

```sql
-- Спочатку SELECT: товари, що ніколи не продавались
SELECT product_id, product_name
FROM products p
WHERE product_name LIKE 'ТЕСТ:%'
  AND NOT EXISTS (SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id);
```

**Результат виконання:**

```text
 product_id |          product_name
------------+---------------------------------
         27 | ТЕСТ: Гіроскутер Hoverboard Pro
(1 row)
```

**Запит:**

```sql
DELETE FROM products p
WHERE product_name LIKE 'ТЕСТ:%'
  AND NOT EXISTS (SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id)
RETURNING product_id, product_name;
```

**Результат виконання:**

```text
 product_id |          product_name
------------+---------------------------------
         27 | ТЕСТ: Гіроскутер Hoverboard Pro
(1 row)

DELETE 1
```

**Запит:**

```sql
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: товар №1 є в замовленнях -> порушення FK
BEGIN;
DELETE FROM products WHERE product_id = 1;
ROLLBACK;
```

**Результат виконання:**

```text
BEGIN
ERROR:  update or delete on table "products" violates foreign key constraint "order_items_product_id_fkey" on table "order_items"
DETAIL:  Key (product_id)=(1) is still referenced from table "order_items".
ROLLBACK
```

**Запит:**

```sql
-- Скільки замовлень блокують видалення клієнта
SELECT c.customer_id, c.contact_name, COUNT(o.order_id) AS orders_cnt
FROM customers c LEFT JOIN orders o USING (customer_id)
GROUP BY c.customer_id, c.contact_name
ORDER BY orders_cnt DESC
LIMIT 5;
```

**Результат виконання:**

```text
 customer_id |          contact_name          | orders_cnt
-------------+--------------------------------+------------
           1 | Петров Іван Миколайович        |          3
           6 | Гриценко Наталія Володимирівна |          2
          14 | Бойко Світлана Петрівна        |          2
          13 | Лисенко Володимир Іванович     |          2
          10 | Федоренко Ірина Петрівна       |          2
(5 rows)
```

### 1.5 Транзакція з COMMIT
Оформлення замовлення: `INSERT orders` → `INSERT order_items` → `UPDATE products` (зменшення залишку) → перевірка → `COMMIT`.

**Запит:**

```sql
BEGIN;
INSERT INTO orders (customer_id, employee_id, required_date, ship_via, freight,
                    ship_name, ship_address, ship_city, ship_region_id, ship_postal_code, order_status)
SELECT customer_id, 2, CURRENT_DATE + 5, 'Нова Пошта', 120.00,
       contact_name, address, city, region_id, postal_code, 'processing'
FROM customers WHERE customer_id = 1;
INSERT INTO order_items (order_id, product_id, unit_price, quantity, discount)
SELECT currval(pg_get_serial_sequence('orders', 'order_id')), product_id, unit_price, 2, 0.05
FROM products WHERE product_id = 3;
UPDATE products SET units_in_stock = units_in_stock - 2
WHERE product_id = 3 AND units_in_stock >= 2;
-- Перевірка до фіксації
SELECT o.order_id, o.order_status, oi.product_id, oi.quantity, p.units_in_stock
FROM orders o
JOIN order_items oi USING (order_id)
JOIN products p USING (product_id)
WHERE o.order_id = currval(pg_get_serial_sequence('orders', 'order_id'));
```

**Результат виконання:**

```text
BEGIN
INSERT 0 1
INSERT 0 1
UPDATE 1
 order_id | order_status | product_id | quantity | units_in_stock
----------+--------------+------------+----------+----------------
       32 | processing   |          3 |        2 |             43
(1 row)
```

**Запит:**

```sql
COMMIT;
```

**Результат виконання:**

```text
COMMIT
```

### 1.6 ROLLBACK
Показано відкат «випадкових» `UPDATE`/`DELETE` без `WHERE`: кількість рядків та ціни після `ROLLBACK` не змінились. Також показано, що після помилки `COMMIT` виконує відкат.

**Запит:**

```sql
-- (а) Свідомий відкат небезпечних змін
SELECT COUNT(*) AS items_before FROM order_items;
```

**Результат виконання:**

```text
 items_before
--------------
           48
(1 row)
```

**Запит:**

```sql
BEGIN;
UPDATE products SET unit_price = 0;          -- "випадково" без WHERE
DELETE FROM order_items;                     -- "випадково" без WHERE
SELECT COUNT(*) AS items_inside_tx FROM order_items;   -- 0
```

**Результат виконання:**

```text
BEGIN
UPDATE 26
DELETE 48
 items_inside_tx
-----------------
               0
(1 row)
```

**Запит:**

```sql
ROLLBACK;
SELECT COUNT(*) AS items_after FROM order_items;       -- як було
```

**Результат виконання:**

```text
ROLLBACK
 items_after
-------------
          48
(1 row)
```

**Запит:**

```sql
SELECT MIN(unit_price) AS min_price FROM products;     -- ціни не змінились
```

**Результат виконання:**

```text
 min_price
-----------
    733.95
(1 row)
```

**Запит:**

```sql
-- (б) Відкат через помилку: після помилки COMMIT фактично стає ROLLBACK
BEGIN;
INSERT INTO categories (category_name) VALUES ('Тимчасова категорія');
UPDATE products SET units_in_stock = -1 WHERE product_id = 1;   -- [ПОМИЛКА] ПЕРЕДБАЧЕНО
```

**Результат виконання:**

```text
BEGIN
INSERT 0 1
ERROR:  new row for relation "products" violates check constraint "chk_products_stock_positive"
DETAIL:  Failing row contains (1, iPhone 15 128GB Чорний, 1, 1, 1 шт в коробці, 29999.00, -1, 10, 5, f, Новий iPhone 15 з потужним процесором A1..., https://example.com/iphone15.jpg).
```

**Запит:**

```sql
COMMIT;   -- повідомить ROLLBACK
SELECT * FROM categories WHERE category_name = 'Тимчасова категорія';   -- 0 рядків
```

**Результат виконання:**

```text
ROLLBACK
 category_id | category_name | description | picture_url
-------------+---------------+-------------+-------------
(0 rows)
```

---

## Рівень 2

### 2.1 Нові таблиці
`product_reservations` (бронювання товарів) та `audit_log` (історія змін, JSONB зі старими й новими значеннями).

**Запит:**

```sql
CREATE TABLE product_reservations (
    reservation_id SERIAL PRIMARY KEY,
    product_id     INTEGER NOT NULL REFERENCES products(product_id)   ON DELETE CASCADE ON UPDATE CASCADE,
    customer_id    INTEGER NOT NULL REFERENCES customers(customer_id)  ON DELETE CASCADE ON UPDATE CASCADE,
    quantity       INTEGER NOT NULL DEFAULT 1 CHECK (quantity > 0),
    reserved_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at     TIMESTAMP NOT NULL,
    status         VARCHAR(20) NOT NULL DEFAULT 'Active'
                   CHECK (status IN ('Active','Fulfilled','Cancelled','Expired')),
    CHECK (expires_at > reserved_at)
);
CREATE TABLE audit_log (
    log_id     SERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    operation  VARCHAR(10) NOT NULL,
    record_id  INTEGER     NOT NULL,
    old_data   JSONB,
    new_data   JSONB,
    changed_at TIMESTAMP   NOT NULL DEFAULT clock_timestamp(),
    changed_by VARCHAR(100) NOT NULL DEFAULT session_user
);
CREATE INDEX idx_audit_table_rec ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_changed_at ON audit_log (changed_at);
-- Тестові бронювання (дефіцитні товари)
INSERT INTO product_reservations (product_id, customer_id, quantity, expires_at) VALUES
  (23, 1, 1, CURRENT_TIMESTAMP + INTERVAL '3 days'),   -- PlayStation 5
  (16, 2, 1, CURRENT_TIMESTAMP + INTERVAL '5 days'),   -- Пральна машина LG
  (16, 4, 1, CURRENT_TIMESTAMP + INTERVAL '5 days')
RETURNING *;
```

**Результат виконання:**

```text
CREATE TABLE
CREATE TABLE
CREATE INDEX
CREATE INDEX
 reservation_id | product_id | customer_id | quantity |        reserved_at         |         expires_at         | status
----------------+------------+-------------+----------+----------------------------+----------------------------+--------
              1 |         23 |           1 |        1 | 2026-11-08 14:10:41.899875 | 2026-11-11 14:10:41.899875 | Active
              2 |         16 |           2 |        1 | 2026-11-08 14:10:41.899875 | 2026-11-13 14:10:41.899875 | Active
              3 |         16 |           4 |        1 | 2026-11-08 14:10:41.899875 | 2026-11-13 14:10:41.899875 | Active
(3 rows)

INSERT 0 3
```

### 2.2 Складна транзакція
Скасування замовлення №28: блокування `FOR UPDATE` → статус `cancelled` → повернення товарів на склад → виконання найстаршого активного бронювання. Усе в одній транзакції.

**Запит:**

```sql
--   1) блокуємо замовлення  2) міняємо статус  3) повертаємо товар на склад
--   4) виконуємо найстарше активне бронювання по повернутих товарах
BEGIN;
SELECT order_id, order_status FROM orders WHERE order_id = 28 FOR UPDATE;
```

**Результат виконання:**

```text
BEGIN
 order_id | order_status
----------+--------------
       28 | pending
(1 row)
```

**Запит:**

```sql
UPDATE orders SET order_status = 'cancelled'
WHERE order_id = 28 AND order_status IN ('pending','processing');
UPDATE products p
SET units_in_stock = p.units_in_stock + oi.quantity
FROM order_items oi
WHERE oi.order_id = 28 AND oi.product_id = p.product_id;
UPDATE product_reservations
SET status = 'Fulfilled'
WHERE reservation_id IN (
    SELECT DISTINCT ON (r.product_id) r.reservation_id
    FROM product_reservations r
    JOIN products p ON p.product_id = r.product_id
    WHERE r.status = 'Active'
      AND p.units_in_stock >= r.quantity
      AND r.product_id IN (SELECT product_id FROM order_items WHERE order_id = 28)
    ORDER BY r.product_id, r.reserved_at
)
RETURNING reservation_id, product_id, customer_id, status;
```

**Результат виконання:**

```text
UPDATE 1
UPDATE 1
 reservation_id | product_id | customer_id |  status
----------------+------------+-------------+-----------
              3 |         16 |           4 | Fulfilled
(1 row)

UPDATE 1
```

**Запит:**

```sql
COMMIT;
SELECT order_id, order_status FROM orders WHERE order_id = 28;
```

**Результат виконання:**

```text
COMMIT
 order_id | order_status
----------+--------------
       28 | cancelled
(1 row)
```

### 2.3 Каскадні операції

| Зв'язок | Поведінка | Результат демонстрації |
|---|---|---|
| `order_items.order_id → orders` | `ON DELETE CASCADE` | видалення замовлення прибирає позиції |
| `products.supplier_id → suppliers` | `ON DELETE SET NULL` | у товарів постачальника `supplier_id = NULL` |
| `employees.reports_to → employees` | `ON DELETE SET NULL` | підлеглі втрачають керівника |
| `orders.customer_id → customers` | `ON UPDATE CASCADE` | зміна PK клієнта оновлює замовлення |
| `order_items.product_id → products` | `ON DELETE RESTRICT` | проданий товар видалити неможливо |

Усі демонстрації виконано в транзакціях із `ROLLBACK`, щоб не змінювати дані.

**Запит:**

```sql
--   CASCADE   : order_items.order_id -> orders
--   SET NULL  : products.supplier_id, employees.reports_to
--   RESTRICT  : order_items.product_id (не можна видалити проданий товар)
--   ON UPDATE CASCADE : orders.customer_id
ALTER TABLE order_items DROP CONSTRAINT order_items_order_id_fkey;
ALTER TABLE order_items ADD  CONSTRAINT order_items_order_id_fkey
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE;
ALTER TABLE order_items DROP CONSTRAINT order_items_product_id_fkey;
ALTER TABLE order_items ADD  CONSTRAINT order_items_product_id_fkey
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE RESTRICT;
ALTER TABLE products DROP CONSTRAINT products_supplier_id_fkey;
ALTER TABLE products ADD  CONSTRAINT products_supplier_id_fkey
    FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id) ON DELETE SET NULL;
ALTER TABLE employees DROP CONSTRAINT employees_reports_to_fkey;
ALTER TABLE employees ADD  CONSTRAINT employees_reports_to_fkey
    FOREIGN KEY (reports_to) REFERENCES employees(employee_id) ON DELETE SET NULL;
ALTER TABLE orders DROP CONSTRAINT orders_customer_id_fkey;
ALTER TABLE orders ADD  CONSTRAINT orders_customer_id_fkey
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON UPDATE CASCADE;
-- (а) ON DELETE CASCADE: видалення замовлення прибирає його позиції
BEGIN;
SELECT COUNT(*) AS items_before FROM order_items WHERE order_id = 26;
```

**Результат виконання:**

```text
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
BEGIN
 items_before
--------------
            1
(1 row)
```

**Запит:**

```sql
DELETE FROM orders WHERE order_id = 26;
SELECT COUNT(*) AS items_after  FROM order_items WHERE order_id = 26;   -- 0
```

**Результат виконання:**

```text
DELETE 1
 items_after
-------------
           0
(1 row)
```

**Запит:**

```sql
ROLLBACK;
-- (б) ON DELETE SET NULL: видалення постачальника
BEGIN;
SELECT COUNT(*) AS products_of_supplier8 FROM products WHERE supplier_id = 8;
```

**Результат виконання:**

```text
ROLLBACK
BEGIN
 products_of_supplier8
-----------------------
                     2
(1 row)
```

**Запит:**

```sql
DELETE FROM suppliers WHERE supplier_id = 8;
SELECT product_id, supplier_id FROM products WHERE product_id IN (14, 21);   -- supplier_id = NULL
```

**Результат виконання:**

```text
DELETE 1
 product_id | supplier_id
------------+-------------
         14 |
         21 |
(2 rows)
```

**Запит:**

```sql
ROLLBACK;
-- (в) ON DELETE SET NULL (самопосилання): видалення керівника
BEGIN;
DELETE FROM employees WHERE employee_id = 1;
SELECT employee_id, reports_to FROM employees WHERE employee_id IN (2,3,4);  -- reports_to = NULL
```

**Результат виконання:**

```text
ROLLBACK
BEGIN
DELETE 1
 employee_id | reports_to
-------------+------------
           2 |
           3 |
           4 |
(3 rows)
```

**Запит:**

```sql
ROLLBACK;
-- (г) ON UPDATE CASCADE: зміна PK клієнта оновлює замовлення
BEGIN;
UPDATE customers SET customer_id = 1001 WHERE customer_id = 15;
SELECT order_id, customer_id FROM orders WHERE customer_id = 1001;
```

**Результат виконання:**

```text
ROLLBACK
BEGIN
UPDATE 1
 order_id | customer_id
----------+-------------
       15 |        1001
       31 |        1001
(2 rows)
```

**Запит:**

```sql
ROLLBACK;
-- (д) RESTRICT: проданий товар видалити не можна
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО
BEGIN;
DELETE FROM products WHERE product_id = 1;
ROLLBACK;
```

**Результат виконання:**

```text
ROLLBACK
BEGIN
ERROR:  update or delete on table "products" violates foreign key constraint "order_items_product_id_fkey" on table "order_items"
DETAIL:  Key (product_id)=(1) is still referenced from table "order_items".
ROLLBACK
```

**Запит:**

```sql
-- (е) CASCADE на бронюваннях: видалення клієнта прибирає його бронювання
BEGIN;
DELETE FROM product_reservations WHERE customer_id = 1;   -- вручну прибираємо
ROLLBACK;
```

**Результат виконання:**

```text
BEGIN
DELETE 1
ROLLBACK
```

### 2.4 SAVEPOINT
Створення клієнта → `SAVEPOINT after_customer` → замовлення → `SAVEPOINT after_order` → позиція з `quantity = 0` (помилка) → `ROLLBACK TO SAVEPOINT after_order` → коректна позиція → `COMMIT`. Клієнт і замовлення збережені, помилкова позиція ні.

**Запит:**

```sql
BEGIN;
INSERT INTO customers (contact_name, address, city, region_id, postal_code, email)
VALUES ('Савчук Олег Павлович', 'вул. Тестова, 1', 'Київ', 25, '01001', 'savchuk.test@example.com');
SAVEPOINT after_customer;
INSERT INTO orders (customer_id, employee_id, required_date, ship_via, ship_name, ship_city, ship_region_id, order_status)
SELECT customer_id, 2, CURRENT_DATE + 5, 'Нова Пошта', contact_name, city, region_id, 'pending'
FROM customers WHERE email = 'savchuk.test@example.com';
SAVEPOINT after_order;
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: quantity = 0 порушує CHECK (quantity > 0)
INSERT INTO order_items (order_id, product_id, unit_price, quantity)
VALUES ((SELECT MAX(order_id) FROM orders), 3, 12999.00, 0);
```

**Результат виконання:**

```text
BEGIN
INSERT 0 1
SAVEPOINT
INSERT 0 1
SAVEPOINT
ERROR:  new row for relation "order_items" violates check constraint "chk_order_items_quantity_positive"
DETAIL:  Failing row contains (49, 33, 3, 12999.00, 0, 0.00).
```

**Запит:**

```sql
-- Транзакція в стані "aborted": відкочуємось тільки до after_order
ROLLBACK TO SAVEPOINT after_order;
-- Коректна позиція
INSERT INTO order_items (order_id, product_id, unit_price, quantity)
VALUES ((SELECT MAX(order_id) FROM orders), 3, 12999.00, 1);
RELEASE SAVEPOINT after_order;
COMMIT;
SELECT o.order_id, c.contact_name, oi.product_id, oi.quantity
FROM orders o JOIN customers c USING (customer_id) JOIN order_items oi USING (order_id)
WHERE c.email = 'savchuk.test@example.com';
```

**Результат виконання:**

```text
ROLLBACK
INSERT 0 1
RELEASE
COMMIT
 order_id |     contact_name     | product_id | quantity
----------+----------------------+------------+----------
       33 | Савчук Олег Павлович |          3 |        1
(1 row)
```

### 2.5 Тригери
- `fn_audit()` + тригери на `products`, `orders`, `customers`: пишуть INSERT/UPDATE/DELETE в `audit_log` (старі та нові значення, користувач, час).
- `fn_orders_guard()` (BEFORE UPDATE): забороняє змінювати статус доставленого замовлення; при переході в `shipped` проставляє `shipped_date`.

**Запит:**

```sql
CREATE OR REPLACE FUNCTION fn_audit() RETURNS trigger
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
DECLARE
    v_pk TEXT := TG_ARGV[0];
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, record_id, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, (to_jsonb(NEW)->>v_pk)::INT, to_jsonb(NEW));
    ELSIF TG_OP = 'UPDATE' THEN
        IF to_jsonb(OLD) IS DISTINCT FROM to_jsonb(NEW) THEN
            INSERT INTO audit_log (table_name, operation, record_id, old_data, new_data)
            VALUES (TG_TABLE_NAME, TG_OP, (to_jsonb(NEW)->>v_pk)::INT, to_jsonb(OLD), to_jsonb(NEW));
        END IF;
    ELSE
        INSERT INTO audit_log (table_name, operation, record_id, old_data)
        VALUES (TG_TABLE_NAME, TG_OP, (to_jsonb(OLD)->>v_pk)::INT, to_jsonb(OLD));
    END IF;
    RETURN NULL;
END;
$$;
CREATE TRIGGER trg_audit_products  AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW EXECUTE FUNCTION fn_audit('product_id');
CREATE TRIGGER trg_audit_orders    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION fn_audit('order_id');
CREATE TRIGGER trg_audit_customers AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW EXECUTE FUNCTION fn_audit('customer_id');
-- Додатковий BEFORE-тригер: контроль життєвого циклу замовлення
CREATE OR REPLACE FUNCTION fn_orders_guard() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    IF OLD.order_status = 'delivered' AND NEW.order_status <> 'delivered' THEN
        RAISE EXCEPTION 'Замовлення % вже доставлене, зміна статусу заборонена', OLD.order_id;
    END IF;
    IF NEW.order_status = 'shipped' AND NEW.shipped_date IS NULL THEN
        NEW.shipped_date := CURRENT_DATE;
    END IF;
    RETURN NEW;
END;
$$;
CREATE TRIGGER trg_orders_guard BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION fn_orders_guard();
-- Тестування
UPDATE products SET unit_price = 13499.00 WHERE product_id = 3;
UPDATE products SET units_in_stock = units_in_stock + 5 WHERE product_id = 3;
UPDATE orders SET order_status = 'shipped' WHERE order_id = 27;   -- shipped_date підставиться автоматично
SELECT order_id, order_status, shipped_date FROM orders WHERE order_id = 27;
```

**Результат виконання:**

```text
CREATE FUNCTION
CREATE TRIGGER
CREATE TRIGGER
CREATE TRIGGER
CREATE FUNCTION
CREATE TRIGGER
UPDATE 1
UPDATE 1
UPDATE 1
 order_id | order_status | shipped_date
----------+--------------+--------------
       27 | shipped      | 2026-11-08
(1 row)
```

**Запит:**

```sql
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: тригер забороняє змінювати доставлене замовлення
UPDATE orders SET order_status = 'pending' WHERE order_id = 1;
```

**Результат виконання:**

```text
ERROR:  Замовлення 1 вже доставлене, зміна статусу заборонена
CONTEXT:  PL/pgSQL function fn_orders_guard() line 4 at RAISE
```

**Запит:**

```sql
SELECT log_id, table_name, operation, record_id,
       old_data->>'unit_price' AS old_price, new_data->>'unit_price' AS new_price, changed_at, changed_by
FROM audit_log
ORDER BY log_id DESC;
```

**Результат виконання:**

```text
 log_id | table_name | operation | record_id | old_price | new_price |         changed_at         | changed_by
--------+------------+-----------+-----------+-----------+-----------+----------------------------+------------
      3 | orders     | UPDATE    |        27 |           |           | 2026-11-08 14:10:41.911063 | postgres
      2 | products   | UPDATE    |         3 | 13499.00  | 13499.00  | 2026-11-08 14:10:41.910449 | postgres
      1 | products   | UPDATE    |         3 | 12999.00  | 13499.00  | 2026-11-08 14:10:41.909928 | postgres
(3 rows)
```

### 2.6 Блокування
Продемонстровано `SELECT ... FOR UPDATE` у двох сесіях (друга чекає завершення першої), режими `NOWAIT` та `SKIP LOCKED`, перегляд блокувань у `pg_locks`, втрачене оновлення та помилку серіалізації на `REPEATABLE READ`.

Для демонстрації відкрито три з'єднання до БД. Сесія A блокує рядок і тримає його 6 секунд, сесія B через 1,5 с намагається заблокувати той самий рядок і **чекає** (час у виводі: B запитує о 14:09:35, а отримує блокування лише після COMMIT сесії A). Сесія C під час очікування бачить блокування через `pg_blocking_pids` та отримує помилку `NOWAIT`.

**Сесія A (тримає блокування):**

```text
BEGIN;
BEGIN
SELECT to_char(clock_timestamp(),'HH24:MI:SS') AS t, 'A: початок' AS подія;
    t     |   подія    
----------+------------
 14:10:41 | A: початок
(1 row)

SELECT product_id, units_in_stock FROM products WHERE product_id = 1 FOR UPDATE;
 product_id | units_in_stock 
------------+----------------
          1 |             24
(1 row)

SELECT pg_sleep(6);
 pg_sleep 
----------
 
(1 row)

UPDATE products SET units_in_stock = units_in_stock - 1 WHERE product_id = 1;
UPDATE 1
COMMIT;
COMMIT
SELECT to_char(clock_timestamp(),'HH24:MI:SS') AS t, 'A: COMMIT виконано' AS подія;
    t     |       подія        
----------+--------------------
 14:10:47 | A: COMMIT виконано
(1 row)
```

**Сесія B (чекає на блокування):**

```text
BEGIN;
BEGIN
SELECT to_char(clock_timestamp(),'HH24:MI:SS') AS t, 'B: запит FOR UPDATE' AS подія;
    t     |        подія        
----------+---------------------
 14:10:43 | B: запит FOR UPDATE
(1 row)

SELECT product_id, units_in_stock FROM products WHERE product_id = 1 FOR UPDATE;
 product_id | units_in_stock 
------------+----------------
          1 |             23
(1 row)

SELECT to_char(clock_timestamp(),'HH24:MI:SS') AS t, 'B: дочекався, блокування отримано' AS подія;
    t     |               подія               
----------+-----------------------------------
 14:10:47 | B: дочекався, блокування отримано
(1 row)

UPDATE products SET units_in_stock = units_in_stock - 1 WHERE product_id = 1;
UPDATE 1
COMMIT;
COMMIT
```

**Сесія C (NOWAIT та перегляд очікування):**

```text
BEGIN;
BEGIN
SELECT product_id FROM products WHERE product_id = 1 FOR UPDATE NOWAIT;
ERROR:  could not obtain lock on row in relation "products"
ROLLBACK;
ROLLBACK
SELECT pid, state, wait_event_type, pg_blocking_pids(pid) AS blocked_by, left(query, 55) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
 pid | state  | wait_event_type | blocked_by |                          query                          
-----+--------+-----------------+------------+---------------------------------------------------------
 498 | active | Lock            | {495}      | SELECT product_id, units_in_stock FROM products WHERE p
(1 row)
```

---

## Рівень 3

### 3.1 Версіонування
Таблиця `product_versions` із тригером: кожна зміна товару закриває попередню версію (`valid_to`) і створює нову. Функції `get_product_version(id, момент)` та `restore_product_version(id, версія)`.

**Запит:**

```sql
--
-- Втрачене оновлення (lost update) без блокування, READ COMMITTED:
--   A: SELECT units_in_stock FROM products WHERE product_id=1;   -> 10
--   B: SELECT units_in_stock FROM products WHERE product_id=1;   -> 10
--   A: UPDATE ... SET units_in_stock = 9 ...; COMMIT;
--   B: UPDATE ... SET units_in_stock = 9 ...; COMMIT;            -> замість 8 лишилось 9
-- Рішення: SELECT ... FOR UPDATE або атомарний UPDATE units_in_stock = units_in_stock - 1.
--
-- Рівні ізоляції (REPEATABLE READ):
--   A: BEGIN ISOLATION LEVEL REPEATABLE READ; SELECT units_in_stock FROM products WHERE product_id=1;
--   B: UPDATE products SET units_in_stock = units_in_stock + 1 WHERE product_id=1;
--   A: SELECT units_in_stock FROM products WHERE product_id=1;   -> те саме значення
--   A: UPDATE products SET units_in_stock = 0 WHERE product_id=1; -> ERROR 40001 could not serialize
--
-- Взаємне блокування (deadlock): A оновлює товар 1, потім 2; B - товар 2, потім 1.
-- PostgreSQL перериває одну з транзакцій помилкою 40P01.
CREATE TABLE product_versions (
    version_id     SERIAL PRIMARY KEY,
    product_id     INTEGER   NOT NULL,           -- без FK: історія переживає видалення
    version_number INTEGER   NOT NULL,
    data           JSONB     NOT NULL,
    valid_from     TIMESTAMP NOT NULL DEFAULT clock_timestamp(),
    valid_to       TIMESTAMP,
    created_by     VARCHAR(100) NOT NULL DEFAULT session_user,
    UNIQUE (product_id, version_number)
);
CREATE INDEX idx_pver_product ON product_versions (product_id, valid_from);
-- Початкові версії для вже існуючих товарів
INSERT INTO product_versions (product_id, version_number, data)
SELECT product_id, 1, to_jsonb(p) FROM products p;
CREATE OR REPLACE FUNCTION fn_product_version() RETURNS trigger
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
DECLARE
    v_ts  TIMESTAMP := clock_timestamp();
    v_num INTEGER;
    v_id  INTEGER;
BEGIN
    IF TG_OP = 'DELETE' THEN
        UPDATE product_versions SET valid_to = v_ts
        WHERE product_id = OLD.product_id AND valid_to IS NULL;
        RETURN NULL;
    END IF;
    IF TG_OP = 'UPDATE' AND to_jsonb(OLD) IS NOT DISTINCT FROM to_jsonb(NEW) THEN
        RETURN NULL;
    END IF;
    v_id := NEW.product_id;
    UPDATE product_versions SET valid_to = v_ts
    WHERE product_id = v_id AND valid_to IS NULL;
    SELECT COALESCE(MAX(version_number), 0) + 1 INTO v_num
    FROM product_versions WHERE product_id = v_id;
    INSERT INTO product_versions (product_id, version_number, data, valid_from)
    VALUES (v_id, v_num, to_jsonb(NEW), v_ts);
    RETURN NULL;
END;
$$;
CREATE TRIGGER trg_product_version AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW EXECUTE FUNCTION fn_product_version();
-- Допоміжна функція: вставка/оновлення товару з JSON-знімка
CREATE OR REPLACE FUNCTION upsert_product_from_json(j JSONB) RETURNS VOID
LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO products
    SELECT (jsonb_populate_record(NULL::products, j)).*
    ON CONFLICT (product_id) DO UPDATE SET
        product_name      = EXCLUDED.product_name,
        supplier_id       = EXCLUDED.supplier_id,
        category_id       = EXCLUDED.category_id,
        quantity_per_unit = EXCLUDED.quantity_per_unit,
        unit_price        = EXCLUDED.unit_price,
        units_in_stock    = EXCLUDED.units_in_stock,
        units_on_order    = EXCLUDED.units_on_order,
        reorder_level     = EXCLUDED.reorder_level,
        discontinued      = EXCLUDED.discontinued,
        description       = EXCLUDED.description,
        picture_url       = EXCLUDED.picture_url;
END;
$$;
-- Стан товару на певний момент часу
CREATE OR REPLACE FUNCTION get_product_version(p_product_id INTEGER, p_at TIMESTAMP)
RETURNS JSONB LANGUAGE sql STABLE AS $$
    SELECT data FROM product_versions
    WHERE product_id = p_product_id
      AND valid_from <= p_at AND (valid_to IS NULL OR valid_to > p_at)
    ORDER BY version_number DESC
    LIMIT 1;
$$;
-- Повернення товару до вказаної версії
CREATE OR REPLACE FUNCTION restore_product_version(p_product_id INTEGER, p_version INTEGER)
RETURNS BOOLEAN LANGUAGE plpgsql AS $$
DECLARE
    v_data JSONB;
BEGIN
    SELECT data INTO v_data FROM product_versions
    WHERE product_id = p_product_id AND version_number = p_version;
    IF v_data IS NULL THEN
        RETURN FALSE;
    END IF;
    PERFORM upsert_product_from_json(v_data);
    RETURN TRUE;
END;
$$;
-- Демонстрація
UPDATE products SET unit_price = 14999.00 WHERE product_id = 3;
UPDATE products SET unit_price = 11999.00, units_in_stock = 40 WHERE product_id = 3;
SELECT version_number, data->>'unit_price' AS price, data->>'units_in_stock' AS stock,
       valid_from, valid_to, created_by
FROM product_versions WHERE product_id = 3 ORDER BY version_number;
```

**Результат виконання:**

```text
CREATE TABLE
CREATE INDEX
INSERT 0 26
CREATE FUNCTION
CREATE TRIGGER
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
UPDATE 1
UPDATE 1
 version_number |  price   | stock |         valid_from         |          valid_to          | created_by
----------------+----------+-------+----------------------------+----------------------------+------------
              1 | 13499.00 | 48    | 2026-11-08 14:10:41.915084 | 2026-11-08 14:10:41.916699 | postgres
              2 | 14999.00 | 48    | 2026-11-08 14:10:41.916699 | 2026-11-08 14:10:41.917297 | postgres
              3 | 11999.00 | 40    | 2026-11-08 14:10:41.917297 |                            | postgres
(3 rows)
```

**Запит:**

```sql
SELECT get_product_version(3, clock_timestamp()::TIMESTAMP) ->> 'unit_price' AS price_now;
```

**Результат виконання:**

```text
 price_now
-----------
 11999.00
(1 row)
```

**Запит:**

```sql
SELECT get_product_version(3, (SELECT valid_from FROM product_versions
                               WHERE product_id = 3 AND version_number = 1) + INTERVAL '1 millisecond')
       ->> 'unit_price' AS price_in_v1;
```

**Результат виконання:**

```text
 price_in_v1
-------------
 13499.00
(1 row)
```

**Запит:**

```sql
SELECT restore_product_version(3, 1);
```

**Результат виконання:**

```text
 restore_product_version
-------------------------
 t
(1 row)
```

**Запит:**

```sql
SELECT product_id, unit_price, units_in_stock FROM products WHERE product_id = 3;
```

**Результат виконання:**

```text
 product_id | unit_price | units_in_stock
------------+------------+----------------
          3 |   13499.00 |             48
(1 row)
```

### 3.2 Відкладені операції
Таблиця `scheduled_operations` та `execute_scheduled_operations()` (`FOR UPDATE SKIP LOCKED`, обробка помилок кожної операції окремо). Типи: `SET_PRICE`, `EXPIRE_RESERVATIONS`, `CANCEL_ORDER`, `CREATE_CHECKPOINT`. Автозапуск через `pg_cron` (команди в скрипті закоментовані).

**Запит:**

```sql
CREATE TABLE scheduled_operations (
    operation_id   SERIAL PRIMARY KEY,
    operation_type VARCHAR(50) NOT NULL,
    operation_data JSONB       NOT NULL DEFAULT '{}'::jsonb,
    scheduled_for  TIMESTAMP   NOT NULL,
    status         VARCHAR(20) NOT NULL DEFAULT 'Pending'
                   CHECK (status IN ('Pending','Executed','Failed','Cancelled')),
    executed_at    TIMESTAMP,
    error_message  TEXT,
    created_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_sched_pending ON scheduled_operations (scheduled_for) WHERE status = 'Pending';
CREATE OR REPLACE FUNCTION execute_scheduled_operations() RETURNS INTEGER
LANGUAGE plpgsql AS $$
DECLARE
    v_op    RECORD;
    v_count INTEGER := 0;
BEGIN
    FOR v_op IN
        SELECT * FROM scheduled_operations
        WHERE status = 'Pending' AND scheduled_for <= LOCALTIMESTAMP
        ORDER BY scheduled_for
        FOR UPDATE SKIP LOCKED
    LOOP
        BEGIN   -- вкладений блок = SAVEPOINT: збій однієї операції не ламає решту
            CASE v_op.operation_type
                WHEN 'SET_PRICE' THEN
                    UPDATE products
                    SET unit_price = (v_op.operation_data->>'unit_price')::NUMERIC
                    WHERE product_id = (v_op.operation_data->>'product_id')::INTEGER;
                WHEN 'EXPIRE_RESERVATIONS' THEN
                    UPDATE product_reservations SET status = 'Expired'
                    WHERE status = 'Active' AND expires_at < LOCALTIMESTAMP;
                WHEN 'CANCEL_ORDER' THEN
                    CALL cancel_order((v_op.operation_data->>'order_id')::INTEGER);
                WHEN 'CREATE_CHECKPOINT' THEN
                    PERFORM create_checkpoint(COALESCE(v_op.operation_data->>'name', 'scheduled'), 'scheduled');
                ELSE
                    RAISE EXCEPTION 'Невідомий тип операції: %', v_op.operation_type;
            END CASE;
            UPDATE scheduled_operations
            SET status = 'Executed', executed_at = clock_timestamp()
            WHERE operation_id = v_op.operation_id;
            v_count := v_count + 1;
        EXCEPTION WHEN OTHERS THEN
            UPDATE scheduled_operations
            SET status = 'Failed', executed_at = clock_timestamp(), error_message = SQLERRM
            WHERE operation_id = v_op.operation_id;
        END;
    END LOOP;
    RETURN v_count;
END;
$$;
```

**Результат виконання:**

```text
CREATE TABLE
CREATE INDEX
CREATE FUNCTION
```

### 3.3 Збережені процедури
`place_order(...)` (перевірка залишків, блокування товарів, автознижка для компаній, атомарний відкат при помилці) та `cancel_order(...)` (`PROCEDURE`, викликається через `CALL`).

**Запит:**

```sql
-- Автозапуск щохвилини (Supabase / розширення pg_cron):
-- CREATE EXTENSION IF NOT EXISTS pg_cron;
-- SELECT cron.schedule('exec-scheduled', '* * * * *', 'SELECT execute_scheduled_operations()');
-- SELECT cron.schedule('daily-checkpoint', '0 3 * * *', $$SELECT create_checkpoint('daily','pg_cron')$$);
-- Оформлення замовлення: перевірка залишків, знижка, блокування товарів
CREATE OR REPLACE FUNCTION place_order(
    p_customer_id INTEGER,
    p_employee_id INTEGER,
    p_items       JSONB,                 -- [{"product_id":3,"quantity":2,"discount":0.05}, ...]
    p_ship_via    VARCHAR DEFAULT 'Нова Пошта'
) RETURNS INTEGER
LANGUAGE plpgsql AS $$
DECLARE
    v_cust     customers;
    v_prod     products;
    v_item     RECORD;
    v_order_id INTEGER;
    v_disc     NUMERIC;
BEGIN
    SELECT * INTO v_cust FROM customers WHERE customer_id = p_customer_id;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Клієнта % не знайдено', p_customer_id;
    END IF;
    IF jsonb_typeof(p_items) IS DISTINCT FROM 'array' OR jsonb_array_length(p_items) = 0 THEN
        RAISE EXCEPTION 'Список позицій порожній';
    END IF;
    INSERT INTO orders (customer_id, employee_id, required_date, ship_via, freight, ship_name,
                        ship_address, ship_city, ship_region_id, ship_postal_code, order_status)
    VALUES (p_customer_id, p_employee_id, CURRENT_DATE + 5, p_ship_via, 100,
            COALESCE(v_cust.company_name, v_cust.contact_name), v_cust.address, v_cust.city,
            v_cust.region_id, v_cust.postal_code, 'processing')
    RETURNING order_id INTO v_order_id;
    FOR v_item IN
        SELECT * FROM jsonb_to_recordset(p_items) AS x(product_id INT, quantity INT, discount NUMERIC)
    LOOP
        SELECT * INTO v_prod FROM products WHERE product_id = v_item.product_id FOR UPDATE;
        IF NOT FOUND THEN
            RAISE EXCEPTION 'Товар % не знайдено', v_item.product_id;
        END IF;
        IF v_prod.discontinued THEN
            RAISE EXCEPTION 'Товар "%" знято з виробництва', v_prod.product_name;
        END IF;
        IF v_prod.units_in_stock < v_item.quantity THEN
            RAISE EXCEPTION 'Недостатньо товару "%": на складі %, потрібно %',
                v_prod.product_name, v_prod.units_in_stock, v_item.quantity;
        END IF;
        -- Автознижка 10 % для компаній при кількості від 3 шт, якщо знижку не задано
        v_disc := COALESCE(v_item.discount,
                           CASE WHEN v_cust.customer_type = 'company' AND v_item.quantity >= 3
                                THEN 0.10 ELSE 0 END);
        INSERT INTO order_items (order_id, product_id, unit_price, quantity, discount)
        VALUES (v_order_id, v_prod.product_id, v_prod.unit_price, v_item.quantity, v_disc);
        UPDATE products SET units_in_stock = units_in_stock - v_item.quantity
        WHERE product_id = v_prod.product_id;
    END LOOP;
    RETURN v_order_id;
END;
$$;
-- Скасування замовлення з поверненням товару на склад
CREATE OR REPLACE PROCEDURE cancel_order(p_order_id INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
    v_status TEXT;
BEGIN
    SELECT order_status INTO v_status FROM orders WHERE order_id = p_order_id FOR UPDATE;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Замовлення % не знайдено', p_order_id;
    END IF;
    IF v_status NOT IN ('pending', 'processing') THEN
        RAISE EXCEPTION 'Замовлення % у статусі "%" скасувати неможливо', p_order_id, v_status;
    END IF;
    UPDATE products p SET units_in_stock = p.units_in_stock + oi.quantity
    FROM order_items oi
    WHERE oi.order_id = p_order_id AND oi.product_id = p.product_id;
    UPDATE orders SET order_status = 'cancelled' WHERE order_id = p_order_id;
END;
$$;
-- Використання
SELECT product_id, units_in_stock FROM products WHERE product_id IN (3, 16);
```

**Результат виконання:**

```text
CREATE FUNCTION
CREATE PROCEDURE
 product_id | units_in_stock
------------+----------------
         16 |             14
          3 |             48
(2 rows)
```

**Запит:**

```sql
SELECT place_order(3, 2, '[{"product_id":3,"quantity":3},{"product_id":16,"quantity":1,"discount":0.05}]'::jsonb)
       AS new_order_id;    -- компанія, 3 шт => автознижка 10 % на товар 3
```

**Результат виконання:**

```text
 new_order_id
--------------
           34
(1 row)
```

**Запит:**

```sql
SELECT oi.product_id, oi.quantity, oi.discount FROM order_items oi
WHERE oi.order_id = (SELECT MAX(order_id) FROM orders);
```

**Результат виконання:**

```text
 product_id | quantity | discount
------------+----------+----------
          3 |        3 |     0.10
         16 |        1 |     0.05
(2 rows)
```

**Запит:**

```sql
SELECT product_id, units_in_stock FROM products WHERE product_id IN (3, 16);   -- залишки зменшились
```

**Результат виконання:**

```text
 product_id | units_in_stock
------------+----------------
          3 |             45
         16 |             13
(2 rows)
```

**Запит:**

```sql
DO $$
DECLARE v_id INTEGER := (SELECT MAX(order_id) FROM orders);   -- CALL не приймає підзапит
BEGIN
    CALL cancel_order(v_id);
END $$;
SELECT product_id, units_in_stock FROM products WHERE product_id IN (3, 16);   -- залишки повернулись
```

**Результат виконання:**

```text
DO
 product_id | units_in_stock
------------+----------------
          3 |             48
         16 |             14
(2 rows)
```

**Запит:**

```sql
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: недостатньо товару -> усе замовлення відкочується атомарно
SELECT place_order(1, 2, '[{"product_id":3,"quantity":1},{"product_id":23,"quantity":1000}]'::jsonb);
```

**Результат виконання:**

```text
ERROR:  Недостатньо товару "PlayStation 5 825GB": на складі 3, потрібно 1000
CONTEXT:  PL/pgSQL function place_order(integer,integer,jsonb,character varying) line 35 at RAISE
```

**Запит:**

```sql
SELECT COUNT(*) AS orders_total FROM orders;   -- нового замовлення немає
```

**Результат виконання:**

```text
 orders_total
--------------
           34
(1 row)
```

**Запит:**

```sql
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: доставлене замовлення скасувати не можна
CALL cancel_order(1);
```

**Результат виконання:**

```text
ERROR:  Замовлення 1 у статусі "delivered" скасувати неможливо
CONTEXT:  PL/pgSQL function cancel_order(integer) line 10 at RAISE
```

**Запит:**

```sql
-- Відкладена операція (тепер, коли процедури існують)
INSERT INTO scheduled_operations (operation_type, operation_data, scheduled_for) VALUES
  ('SET_PRICE', '{"product_id": 3, "unit_price": 12499}', LOCALTIMESTAMP - INTERVAL '1 minute'),
  ('EXPIRE_RESERVATIONS', '{}', LOCALTIMESTAMP - INTERVAL '1 minute'),
  ('BAD_TYPE', '{}', LOCALTIMESTAMP - INTERVAL '1 minute'),
  ('SET_PRICE', '{"product_id": 3, "unit_price": 12999}', LOCALTIMESTAMP + INTERVAL '1 day');
SELECT execute_scheduled_operations() AS executed;
```

**Результат виконання:**

```text
INSERT 0 4
 executed
----------
        2
(1 row)
```

**Запит:**

```sql
SELECT operation_id, operation_type, scheduled_for, status, error_message FROM scheduled_operations ORDER BY 1;
```

**Результат виконання:**

```text
 operation_id |   operation_type    |       scheduled_for        |  status  |          error_message
--------------+---------------------+----------------------------+----------+----------------------------------
            1 | SET_PRICE           | 2026-11-08 14:09:41.926314 | Executed |
            2 | EXPIRE_RESERVATIONS | 2026-11-08 14:09:41.926314 | Executed |
            3 | BAD_TYPE            | 2026-11-08 14:09:41.926314 | Failed   | Невідомий тип операції: BAD_TYPE
            4 | SET_PRICE           | 2026-11-09 14:10:41.926314 | Pending  |
(4 rows)
```

### 3.4 Ролі
`tm_viewer` (читання) → `tm_sales` (оформлення замовлень, права на стовпці `units_in_stock`, `units_on_order`, без DELETE) → `tm_manager` (керування каталогом) → `tm_admin` (усе). Перевірено через `SET ROLE`.

**Запит:**

```sql
DO $$
DECLARE r TEXT;
BEGIN
    FOREACH r IN ARRAY ARRAY['tm_viewer', 'tm_sales', 'tm_manager', 'tm_admin'] LOOP
        IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = r) THEN
            EXECUTE format('CREATE ROLE %I NOLOGIN', r);
        END IF;
    END LOOP;
END $$;
-- tm_viewer: лише читання
GRANT USAGE ON SCHEMA public TO tm_viewer, tm_sales, tm_manager, tm_admin;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO tm_viewer;
-- tm_sales: читає все + оформлює замовлення, змінює лише залишки товарів, нічого не видаляє
GRANT tm_viewer TO tm_sales;
```

**Результат виконання:**

```text
DO
GRANT
GRANT
GRANT ROLE
```

**Запит:**

```sql
GRANT INSERT, UPDATE ON customers, orders, order_items, product_reservations TO tm_sales;
GRANT UPDATE (units_in_stock, units_on_order) ON products TO tm_sales;     -- права на рівні стовпців
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO tm_sales;
-- tm_manager: + керування каталогом та видалення бронювань
GRANT tm_sales TO tm_manager;
```

**Результат виконання:**

```text
GRANT
GRANT
GRANT
GRANT ROLE
```

**Запит:**

```sql
GRANT INSERT, UPDATE, DELETE ON products TO tm_manager;
GRANT DELETE ON product_reservations TO tm_manager;
-- tm_admin: усе
GRANT tm_manager TO tm_admin;
```

**Результат виконання:**

```text
GRANT
GRANT
GRANT ROLE
```

**Запит:**

```sql
GRANT ALL ON ALL TABLES IN SCHEMA public TO tm_admin;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO tm_admin;
-- Щоб поточний користувач міг робити SET ROLE (потрібно в PostgreSQL 16+ / Supabase):
GRANT tm_viewer, tm_sales, tm_manager, tm_admin TO CURRENT_USER;
```

**Результат виконання:**

```text
GRANT
GRANT
GRANT ROLE
```

**Запит:**

```sql
-- Тестування
SET ROLE tm_viewer;
SELECT COUNT(*) FROM products;                                         -- OK
```

**Результат виконання:**

```text
SET
 count
-------
    26
(1 row)
```

**Запит:**

```sql
UPDATE products SET units_in_stock = 5 WHERE product_id = 1;           -- [ПОМИЛКА] ПЕРЕДБАЧЕНО
```

**Результат виконання:**

```text
ERROR:  permission denied for table products
```

**Запит:**

```sql
RESET ROLE;
SET ROLE tm_sales;
UPDATE products SET units_in_stock = units_in_stock WHERE product_id = 1;   -- OK
UPDATE products SET unit_price = 1 WHERE product_id = 1;               -- [ПОМИЛКА] ПЕРЕДБАЧЕНО (немає права на цей стовпець)
```

**Результат виконання:**

```text
RESET
SET
UPDATE 1
ERROR:  permission denied for table products
```

**Запит:**

```sql
DELETE FROM orders WHERE order_id = 1;                                 -- [ПОМИЛКА] ПЕРЕДБАЧЕНО
```

**Результат виконання:**

```text
ERROR:  permission denied for table orders
```

**Запит:**

```sql
SELECT place_order(2, 2, '[{"product_id":16,"quantity":1}]'::jsonb);  -- OK: оформлення через функцію
```

**Результат виконання:**

```text
 place_order
-------------
          36
(1 row)
```

**Запит:**

```sql
RESET ROLE;
SET ROLE tm_manager;
UPDATE products SET unit_price = unit_price WHERE product_id = 1;      -- OK
DELETE FROM customers WHERE customer_id = 1;                           -- [ПОМИЛКА] ПЕРЕДБАЧЕНО
```

**Результат виконання:**

```text
RESET
SET
UPDATE 1
ERROR:  permission denied for table customers
```

**Запит:**

```sql
RESET ROLE;
-- Перевірка: хто що може
SELECT grantee, table_name, string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privileges
FROM information_schema.table_privileges
WHERE grantee LIKE 'tm\_%' AND table_name IN ('products', 'orders', 'order_items')
GROUP BY grantee, table_name ORDER BY grantee, table_name;
```

**Результат виконання:**

```text
RESET
  grantee   | table_name  |                          privileges
------------+-------------+---------------------------------------------------------------
 tm_admin   | order_items | DELETE, INSERT, REFERENCES, SELECT, TRIGGER, TRUNCATE, UPDATE
 tm_admin   | orders      | DELETE, INSERT, REFERENCES, SELECT, TRIGGER, TRUNCATE, UPDATE
 tm_admin   | products    | DELETE, INSERT, REFERENCES, SELECT, TRIGGER, TRUNCATE, UPDATE
 tm_manager | products    | DELETE, INSERT, UPDATE
 tm_sales   | order_items | INSERT, UPDATE
 tm_sales   | orders      | INSERT, UPDATE
 tm_viewer  | order_items | SELECT
 tm_viewer  | orders      | SELECT
 tm_viewer  | products    | SELECT
(9 rows)
```

### 3.5 Контрольні точки
Таблиця `checkpoints`, функція `create_checkpoint()`, `restore_products_from_checkpoint()` і statement-тригер, який автоматично створює точку перед будь-яким `DELETE` з `products`. Для повного відновлення після збою описано налаштування WAL-архівації та PITR.

**Запит:**

```sql
CREATE TABLE checkpoints (
    checkpoint_id SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    reason        TEXT,
    created_at    TIMESTAMP NOT NULL DEFAULT clock_timestamp(),
    created_by    VARCHAR(100) NOT NULL DEFAULT session_user,
    snapshot      JSONB NOT NULL
);
CREATE OR REPLACE FUNCTION create_checkpoint(p_name TEXT, p_reason TEXT DEFAULT NULL)
RETURNS INTEGER LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
DECLARE
    v_id INTEGER;
BEGIN
    INSERT INTO checkpoints (name, reason, snapshot)
    SELECT p_name, p_reason, jsonb_build_object(
        'products',    (SELECT COALESCE(jsonb_agg(to_jsonb(t)), '[]') FROM products t),
        'customers',   (SELECT COALESCE(jsonb_agg(to_jsonb(t)), '[]') FROM customers t),
        'orders',      (SELECT COALESCE(jsonb_agg(to_jsonb(t)), '[]') FROM orders t),
        'order_items', (SELECT COALESCE(jsonb_agg(to_jsonb(t)), '[]') FROM order_items t))
    RETURNING checkpoint_id INTO v_id;
    RETURN v_id;
END;
$$;
-- Відновлення каталогу товарів зі знімка (відсутні рядки повертаються, змінені - відновлюються)
CREATE OR REPLACE FUNCTION restore_products_from_checkpoint(p_checkpoint_id INTEGER)
RETURNS INTEGER LANGUAGE plpgsql AS $$
DECLARE
    v_row   RECORD;
    v_count INTEGER := 0;
BEGIN
    IF NOT EXISTS (SELECT 1 FROM checkpoints WHERE checkpoint_id = p_checkpoint_id) THEN
        RAISE EXCEPTION 'Контрольну точку % не знайдено', p_checkpoint_id;
    END IF;
    FOR v_row IN
        SELECT j AS item
        FROM checkpoints c CROSS JOIN LATERAL jsonb_array_elements(c.snapshot->'products') AS j
        WHERE c.checkpoint_id = p_checkpoint_id
    LOOP
        PERFORM upsert_product_from_json(v_row.item);
        v_count := v_count + 1;
    END LOOP;
    RETURN v_count;
END;
$$;
-- Автоматична контрольна точка перед будь-яким DELETE по products
CREATE OR REPLACE FUNCTION fn_auto_checkpoint() RETURNS trigger
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
BEGIN
    PERFORM create_checkpoint('auto: before DELETE on products', 'statement trigger');
    RETURN NULL;
END;
$$;
CREATE TRIGGER trg_products_auto_checkpoint BEFORE DELETE ON products
    FOR EACH STATEMENT EXECUTE FUNCTION fn_auto_checkpoint();
-- Демонстрація: "випадкове" видалення й відновлення
INSERT INTO products (product_name, supplier_id, category_id, unit_price, units_in_stock)
VALUES ('ТЕСТ-L3: товар для відновлення', 1, 8, 999.00, 7)
RETURNING product_id;
```

**Результат виконання:**

```text
CREATE TABLE
CREATE FUNCTION
CREATE FUNCTION
CREATE FUNCTION
CREATE TRIGGER
 product_id
------------
         30
(1 row)

INSERT 0 1
```

**Запит:**

```sql
SELECT create_checkpoint('manual', 'перед видаленням');
```

**Результат виконання:**

```text
 create_checkpoint
-------------------
                 1
(1 row)
```

**Запит:**

```sql
DELETE FROM products WHERE product_name = 'ТЕСТ-L3: товар для відновлення';   -- спрацює авто-точка
SELECT COUNT(*) AS found_after_delete FROM products WHERE product_name = 'ТЕСТ-L3: товар для відновлення';  -- 0
```

**Результат виконання:**

```text
DELETE 1
 found_after_delete
--------------------
                  0
(1 row)
```

**Запит:**

```sql
SELECT checkpoint_id, name, reason, created_at FROM checkpoints ORDER BY checkpoint_id;
```

**Результат виконання:**

```text
 checkpoint_id |              name               |      reason       |         created_at
---------------+---------------------------------+-------------------+----------------------------
             1 | manual                          | перед видаленням  | 2026-11-08 14:10:41.938184
             2 | auto: before DELETE on products | statement trigger | 2026-11-08 14:10:41.940302
(2 rows)
```

**Запит:**

```sql
SELECT restore_products_from_checkpoint((SELECT MAX(checkpoint_id) FROM checkpoints));
```

**Результат виконання:**

```text
 restore_products_from_checkpoint
----------------------------------
                               27
(1 row)
```

**Запит:**

```sql
SELECT product_id, product_name, unit_price FROM products WHERE product_name = 'ТЕСТ-L3: товар для відновлення';  -- повернувся
```

**Результат виконання:**

```text
 product_id |          product_name          | unit_price
------------+--------------------------------+------------
         30 | ТЕСТ-L3: товар для відновлення |     999.00
(1 row)
```

### 3.6 Інтерфейс історії
Представлення `v_change_history` (людинозрозумілий опис змін), `v_product_price_history`, функція `get_changes_report()` та `revert_product_changes(від, до)` для відкату змін за період.

**Запит:**

```sql
-- Для повного відновлення БД після збою (PITR) на рівні сервера:
--   wal_level = replica; archive_mode = on; archive_command = 'cp %p /backup/wal/%f'
--   pg_basebackup -D /backup/base -Ft -z -P
--   recovery: restore_command = 'cp /backup/wal/%f %p'; recovery_target_time = '2024-08-20 12:00:00'
CREATE OR REPLACE VIEW v_change_history AS
SELECT a.log_id, a.table_name, a.operation, a.record_id, a.changed_at, a.changed_by,
       CASE a.operation
           WHEN 'INSERT' THEN 'Додано запис'
           WHEN 'DELETE' THEN 'Видалено запис'
           ELSE 'Змінено: ' || COALESCE((
                SELECT string_agg(k || ': ' || COALESCE(a.old_data->>k, 'NULL') || ' → ' ||
                                  COALESCE(a.new_data->>k, 'NULL'), '; ')
                FROM jsonb_object_keys(a.new_data) AS k
                WHERE a.old_data->k IS DISTINCT FROM a.new_data->k), '-')
       END AS description
FROM audit_log a;
CREATE OR REPLACE VIEW v_product_price_history AS
SELECT product_id, version_number,
       data->>'product_name'                              AS product_name,
       (data->>'unit_price')::NUMERIC                     AS unit_price,
       LAG((data->>'unit_price')::NUMERIC) OVER (PARTITION BY product_id ORDER BY version_number) AS prev_price,
       valid_from, valid_to, created_by
FROM product_versions;
GRANT SELECT ON v_change_history, v_product_price_history TO tm_viewer;
SELECT * FROM v_change_history WHERE table_name = 'products' ORDER BY log_id DESC LIMIT 10;
```

**Результат виконання:**

```text
CREATE VIEW
CREATE VIEW
GRANT
 log_id | table_name | operation | record_id |         changed_at         | changed_by |                            description
--------+------------+-----------+-----------+----------------------------+------------+-------------------------------------------------------------------
     20 | products   | INSERT    |        30 | 2026-11-08 14:10:41.944402 | postgres   | Додано запис
     19 | products   | DELETE    |        30 | 2026-11-08 14:10:41.94181  | postgres   | Видалено запис
     18 | products   | INSERT    |        30 | 2026-11-08 14:10:41.937636 | postgres   | Додано запис
     17 | products   | UPDATE    |        16 | 2026-11-08 14:10:41.93275  | postgres   | Змінено: units_in_stock: 14 → 13
     15 | products   | UPDATE    |         3 | 2026-11-08 14:10:41.927107 | postgres   | Змінено: unit_price: 13499.00 → 12499.00
     11 | products   | UPDATE    |        16 | 2026-11-08 14:10:41.924947 | postgres   | Змінено: units_in_stock: 13 → 14
     10 | products   | UPDATE    |         3 | 2026-11-08 14:10:41.924766 | postgres   | Змінено: units_in_stock: 45 → 48
      9 | products   | UPDATE    |        16 | 2026-11-08 14:10:41.923485 | postgres   | Змінено: units_in_stock: 14 → 13
      8 | products   | UPDATE    |         3 | 2026-11-08 14:10:41.923232 | postgres   | Змінено: units_in_stock: 48 → 45
      6 | products   | UPDATE    |         3 | 2026-11-08 14:10:41.918446 | postgres   | Змінено: unit_price: 11999.00 → 13499.00; units_in_stock: 40 → 48
(10 rows)
```

**Запит:**

```sql
SELECT * FROM v_product_price_history WHERE product_id = 3 ORDER BY version_number;
```

**Результат виконання:**

```text
 product_id | version_number |             product_name             | unit_price | prev_price |         valid_from         |          valid_to          | created_by
------------+----------------+--------------------------------------+------------+------------+----------------------------+----------------------------+------------
          3 |              1 | Xiaomi Redmi Note 13 Pro 128GB Синій |   13499.00 |            | 2026-11-08 14:10:41.915084 | 2026-11-08 14:10:41.916699 | postgres
          3 |              2 | Xiaomi Redmi Note 13 Pro 128GB Синій |   14999.00 |   13499.00 | 2026-11-08 14:10:41.916699 | 2026-11-08 14:10:41.917297 | postgres
          3 |              3 | Xiaomi Redmi Note 13 Pro 128GB Синій |   11999.00 |   14999.00 | 2026-11-08 14:10:41.917297 | 2026-11-08 14:10:41.918469 | postgres
          3 |              4 | Xiaomi Redmi Note 13 Pro 128GB Синій |   13499.00 |   11999.00 | 2026-11-08 14:10:41.918469 | 2026-11-08 14:10:41.923247 | postgres
          3 |              5 | Xiaomi Redmi Note 13 Pro 128GB Синій |   13499.00 |   13499.00 | 2026-11-08 14:10:41.923247 | 2026-11-08 14:10:41.924784 | postgres
          3 |              6 | Xiaomi Redmi Note 13 Pro 128GB Синій |   13499.00 |   13499.00 | 2026-11-08 14:10:41.924784 | 2026-11-08 14:10:41.927123 | postgres
          3 |              7 | Xiaomi Redmi Note 13 Pro 128GB Синій |   12499.00 |   13499.00 | 2026-11-08 14:10:41.927123 |                            | postgres
(7 rows)
```

**Запит:**

```sql
-- Звіт про зміни за період
CREATE OR REPLACE FUNCTION get_changes_report(
    p_start TIMESTAMP,
    p_end   TIMESTAMP DEFAULT clock_timestamp()
) RETURNS TABLE (change_day DATE, tbl TEXT, total_changes BIGINT, inserts BIGINT, updates BIGINT, deletes BIGINT)
LANGUAGE sql STABLE AS $$
    SELECT a.changed_at::DATE, a.table_name::TEXT, COUNT(*),
           COUNT(*) FILTER (WHERE a.operation = 'INSERT'),
           COUNT(*) FILTER (WHERE a.operation = 'UPDATE'),
           COUNT(*) FILTER (WHERE a.operation = 'DELETE')
    FROM audit_log a
    WHERE a.changed_at BETWEEN p_start AND p_end
    GROUP BY a.changed_at::DATE, a.table_name
    ORDER BY 1 DESC, 2;
$$;
SELECT * FROM get_changes_report(CURRENT_DATE - INTERVAL '30 days');
```

**Результат виконання:**

```text
CREATE FUNCTION
 change_day |   tbl    | total_changes | inserts | updates | deletes
------------+----------+---------------+---------+---------+---------
 2026-11-08 | orders   |             4 |       2 |       2 |       0
 2026-11-08 | products |            14 |       2 |      11 |       1
(2 rows)
```

**Запит:**

```sql
-- Відкат змін товарів за період: кожен товар повертається до стану, який був на початок періоду
CREATE OR REPLACE FUNCTION revert_product_changes(p_from TIMESTAMP, p_to TIMESTAMP DEFAULT clock_timestamp())
RETURNS INTEGER LANGUAGE plpgsql AS $$
DECLARE
    r RECORD;
    n INTEGER := 0;
BEGIN
    FOR r IN
        SELECT DISTINCT ON (record_id) record_id, operation, old_data
        FROM audit_log
        WHERE table_name = 'products' AND changed_at BETWEEN p_from AND p_to
        ORDER BY record_id, changed_at, log_id          -- найраніша подія періоду
    LOOP
        IF r.operation = 'INSERT' THEN
            DELETE FROM products WHERE product_id = r.record_id;   -- товар з'явився в періоді
        ELSE
            PERFORM upsert_product_from_json(r.old_data);          -- UPDATE / DELETE
        END IF;
        n := n + 1;
    END LOOP;
    RETURN n;
END;
$$;
-- Демонстрація відкату
CREATE TEMP TABLE _mark AS SELECT clock_timestamp()::TIMESTAMP AS t;
UPDATE products SET unit_price = unit_price * 2 WHERE category_id = 8;
UPDATE products SET units_in_stock = 0 WHERE product_id = 1;
SELECT product_id, unit_price, units_in_stock FROM products WHERE product_id IN (1, 20, 21, 22);
```

**Результат виконання:**

```text
CREATE FUNCTION
SELECT 1
UPDATE 4
UPDATE 1
 product_id | unit_price | units_in_stock
------------+------------+----------------
         20 |   18897.90 |             30
         21 |    1467.90 |            100
         22 |    8397.90 |             18
          1 |   29999.00 |              0
(4 rows)
```

**Запит:**

```sql
SELECT * FROM v_change_history WHERE changed_at >= (SELECT t FROM _mark) ORDER BY log_id;
```

**Результат виконання:**

```text
 log_id | table_name | operation | record_id |         changed_at         | changed_by |               description
--------+------------+-----------+-----------+----------------------------+------------+-----------------------------------------
     21 | products   | UPDATE    |        20 | 2026-11-08 14:10:41.948627 | postgres   | Змінено: unit_price: 9448.95 → 18897.90
     22 | products   | UPDATE    |        21 | 2026-11-08 14:10:41.948848 | postgres   | Змінено: unit_price: 733.95 → 1467.90
     23 | products   | UPDATE    |        22 | 2026-11-08 14:10:41.94893  | postgres   | Змінено: unit_price: 4198.95 → 8397.90
     24 | products   | UPDATE    |        30 | 2026-11-08 14:10:41.948981 | postgres   | Змінено: unit_price: 999.00 → 1998.00
     25 | products   | UPDATE    |         1 | 2026-11-08 14:10:41.949362 | postgres   | Змінено: units_in_stock: 24 → 0
(5 rows)
```

**Запит:**

```sql
SELECT revert_product_changes((SELECT t FROM _mark)) AS reverted_products;
```

**Результат виконання:**

```text
 reverted_products
-------------------
                 5
(1 row)
```

**Запит:**

```sql
SELECT product_id, unit_price, units_in_stock FROM products WHERE product_id IN (1, 20, 21, 22);  -- стан до змін
```

**Результат виконання:**

```text
 product_id | unit_price | units_in_stock
------------+------------+----------------
          1 |   29999.00 |             24
         20 |    9448.95 |             30
         21 |     733.95 |            100
         22 |    4198.95 |             18
(4 rows)
```

**Запит:**

```sql
-- Доступ до службових функцій лише адміністратору
REVOKE EXECUTE ON FUNCTION revert_product_changes(TIMESTAMP, TIMESTAMP)  FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION restore_products_from_checkpoint(INTEGER)     FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION create_checkpoint(TEXT, TEXT)                 FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION execute_scheduled_operations()                FROM PUBLIC;
GRANT  EXECUTE ON FUNCTION revert_product_changes(TIMESTAMP, TIMESTAMP)  TO tm_admin;
GRANT  EXECUTE ON FUNCTION restore_products_from_checkpoint(INTEGER)     TO tm_admin;
GRANT  EXECUTE ON FUNCTION create_checkpoint(TEXT, TEXT)                 TO tm_admin;
GRANT  EXECUTE ON FUNCTION execute_scheduled_operations()                TO tm_admin;
-- Кінець лабораторної роботи 03
```

**Результат виконання:**

```text
REVOKE
REVOKE
REVOKE
REVOKE
GRANT
GRANT
GRANT
GRANT
```

---

## Висновки

_Опишіть своїми словами, що вдалося, які обмеження й тригери найбільше допомогли захистити дані, які складнощі виникли._

---

## Відповіді на контрольні запитання

**1. Різниця між INSERT, UPDATE, DELETE. Обмеження.**
INSERT додає нові рядки, UPDATE змінює значення існуючих, DELETE видаляє рядки. Усі три перевіряються на NOT NULL, CHECK, UNIQUE, зовнішні ключі та тригери; DELETE і зміна PK залежать ще й від правил `ON DELETE`/`ON UPDATE`.

**2. Чому критична умова WHERE.**
Без неї оператор діє на всі рядки таблиці: `UPDATE products SET unit_price = 0` обнулить ціни всього каталогу, `DELETE FROM orders` видалить усі замовлення. Тому спочатку виконують SELECT з тією самою умовою, а зміни роблять у транзакції.

**3. Транзакція та ACID.**
Транзакція виконується як єдине ціле. Атомарність: замовлення з трьох операцій (замовлення, позиції, списання залишку) або виконується повністю, або відкочується. Консистентність: після транзакції діють усі обмеження (залишок не від'ємний). Ізольованість: паралельні оформлення не бачать проміжних станів одне одного. Довговічність: після `COMMIT` дані збережуться навіть при збої сервера (WAL).

**4. COMMIT і ROLLBACK.**
`COMMIT` фіксує всі зміни транзакції, `ROLLBACK` скасовує. `ROLLBACK` застосовують при помилці або зміні рішення, `COMMIT` після успішної перевірки результатів. У PostgreSQL `COMMIT` у транзакції після помилки виконає відкат.

**5. SAVEPOINT.**
Це позначка всередині транзакції, до якої можна відкотитись (`ROLLBACK TO SAVEPOINT`), не втрачаючи попередніх операцій. Простий `ROLLBACK` скасовує все. У лабі: клієнт і замовлення збережені, помилкову позицію скасовано.

**6. Типи обмежень цілісності.**
PRIMARY KEY (унікальна ідентифікація), FOREIGN KEY (посилальна цілісність), UNIQUE (унікальність, наприклад email), CHECK (допустимі значення: `quantity > 0`), NOT NULL (обов'язковість), DEFAULT (значення за замовчуванням).

**7. Каскадні операції.**
`CASCADE` (змінити/видалити залежні рядки), `SET NULL`, `SET DEFAULT`, `RESTRICT` / `NO ACTION` (заборонити). Застосовуються до `ON DELETE` та `ON UPDATE`.

**8. Тригери.**
Автоматично виконуються при INSERT/UPDATE/DELETE. Приклади: аудит змін (`audit_log`), версіонування товарів, контроль статусу замовлення, автоматична контрольна точка перед видаленням.

**9. Блокування.**
Без блокувань дві транзакції можуть прочитати той самий залишок і обидві списати з нього: втрачене оновлення, перепродаж товару. `SELECT ... FOR UPDATE` змушує другу транзакцію чекати. Надмірні або різнопорядкові блокування можуть призвести до взаємного блокування (deadlock).

**10. Рівні ізоляції.**
`READ UNCOMMITTED` (у PostgreSQL працює як `READ COMMITTED`), `READ COMMITTED` (немає брудного читання), `REPEATABLE READ` (немає неповторюваного читання; у PostgreSQL і фантомів), `SERIALIZABLE` (немає аномалій, можливі помилки серіалізації з повтором). Для звичайних операцій достатньо `READ COMMITTED`, для звітів, що мають бачити єдиний знімок, `REPEATABLE READ`, для критичних фінансових операцій `SERIALIZABLE`.
