 Лабораторна робота 03. Модифікація даних та транзакції
-- База даних: "ТехноМарт" (PostgreSQL 14+ / Supabase)
-- Передумова: виконано скрипт створення БД "ТехноМарт".
--
-- ВАЖЛИВО: у psql / pgAdmin / DBeaver скрипт можна запускати блоками.
-- Рядки з позначкою  -- [ПОМИЛКА]  ПЕРЕДБАЧЕНО: вони навмисно викликають
-- помилку (для демонстрації обмежень). У Supabase SQL Editor кожен такий
-- блок запускайте окремо, бо помилка зупиняє весь запуск.
-- =====================================================================


-- #####################################################################
-- РІВЕНЬ 1
-- #####################################################################

-- ---------------------------------------------------------------------
-- 1.1 Структура та обмеження цілісності
-- Основні таблиці вже існують (PK, FK, UNIQUE, CHECK). Додаємо ще правила.
-- ---------------------------------------------------------------------
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

-- ---------------------------------------------------------------------
-- 1.2 INSERT: додавання записів
-- ---------------------------------------------------------------------
-- Одиночний INSERT з RETURNING
INSERT INTO categories (category_name, description, picture_url)
VALUES ('Електротранспорт', 'Електросамокати, гіроскутери, електровелосипеди', 'https://example.com/e-transport.jpg')
RETURNING category_id, category_name;

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

-- Новий клієнт
INSERT INTO customers (company_name, contact_name, contact_title, address, city, region_id,
                       postal_code, phone, email, customer_type)
VALUES ('ТОВ "Нова Хвиля"', 'Захарченко Богдан Ігорович', 'Директор', 'вул. Шота Руставелі, 10', 'Київ',
        25, '01001', '+380441112233', 'zakharchenko@novakhvylia.ua', 'company')
RETURNING customer_id, contact_name;

-- INSERT ... SELECT: архів дешевих товарів
CREATE TABLE products_archive (LIKE products INCLUDING DEFAULTS, archived_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
INSERT INTO products_archive
SELECT p.*, CURRENT_TIMESTAMP FROM products p WHERE p.unit_price < 1000;
SELECT product_id, product_name, unit_price, archived_at FROM products_archive;

-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: порушення UNIQUE (дубль email)
BEGIN;
INSERT INTO customers (contact_name, email) VALUES ('Дублікат', 'petrov.ivan@gmail.com');
ROLLBACK;

-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: порушення CHECK (від'ємна ціна)
BEGIN;
INSERT INTO products (product_name, unit_price) VALUES ('Помилковий товар', -10);
ROLLBACK;

-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: порушення FOREIGN KEY (неіснуюча категорія)
BEGIN;
INSERT INTO products (product_name, category_id, unit_price) VALUES ('Сирота', 9999, 100);
ROLLBACK;

-- ---------------------------------------------------------------------
-- 1.3 UPDATE: безпечне оновлення (SELECT -> UPDATE -> SELECT)
-- ---------------------------------------------------------------------
-- Перевіряємо, які рядки буде змінено
SELECT product_id, product_name, unit_price FROM products WHERE category_id = 8;

-- Підвищення цін на аксесуари на 5 %
UPDATE products
SET unit_price = ROUND(unit_price * 1.05, 2)
WHERE category_id = 8
RETURNING product_id, product_name, unit_price;

-- Оновлення кількох полів одночасно
UPDATE customers
SET phone = '+380501112233', city = 'Львів', region_id = 2
WHERE email = 'zakharchenko@novakhvylia.ua'
RETURNING customer_id, phone, city;

-- UPDATE ... FROM: поповнення замовлення для товарів із малим залишком
UPDATE products p
SET units_on_order = p.reorder_level * 2
FROM suppliers s
WHERE p.supplier_id = s.supplier_id
  AND p.units_in_stock <= p.reorder_level + 1      -- майже вичерпані товари
RETURNING p.product_id, p.product_name, p.units_in_stock, p.units_on_order;

-- Захист умовою: залишок не піде в мінус
UPDATE products
SET units_in_stock = units_in_stock - 1
WHERE product_id = 1 AND units_in_stock > 0
RETURNING product_id, units_in_stock;

-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: CHECK (units_in_stock >= 0)
BEGIN;
UPDATE products SET units_in_stock = units_in_stock - 1000 WHERE product_id = 1;
ROLLBACK;

-- ---------------------------------------------------------------------
-- 1.4 DELETE: видалення з урахуванням зовнішніх ключів
-- ---------------------------------------------------------------------
-- Спочатку SELECT: товари, що ніколи не продавались
SELECT product_id, product_name
FROM products p
WHERE product_name LIKE 'ТЕСТ:%'
  AND NOT EXISTS (SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id);

DELETE FROM products p
WHERE product_name LIKE 'ТЕСТ:%'
  AND NOT EXISTS (SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id)
RETURNING product_id, product_name;

-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: товар №1 є в замовленнях -> порушення FK
BEGIN;
DELETE FROM products WHERE product_id = 1;
ROLLBACK;

-- Скільки замовлень блокують видалення клієнта
SELECT c.customer_id, c.contact_name, COUNT(o.order_id) AS orders_cnt
FROM customers c LEFT JOIN orders o USING (customer_id)
GROUP BY c.customer_id, c.contact_name
ORDER BY orders_cnt DESC
LIMIT 5;

-- ---------------------------------------------------------------------
-- 1.5 Проста транзакція: оформлення замовлення (COMMIT)
-- ---------------------------------------------------------------------
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

COMMIT;

-- ---------------------------------------------------------------------
-- 1.6 ROLLBACK
-- ---------------------------------------------------------------------
-- (а) Свідомий відкат небезпечних змін
SELECT COUNT(*) AS items_before FROM order_items;

BEGIN;
UPDATE products SET unit_price = 0;          -- "випадково" без WHERE
DELETE FROM order_items;                     -- "випадково" без WHERE
SELECT COUNT(*) AS items_inside_tx FROM order_items;   -- 0
ROLLBACK;

SELECT COUNT(*) AS items_after FROM order_items;       -- як було
SELECT MIN(unit_price) AS min_price FROM products;     -- ціни не змінились

-- (б) Відкат через помилку: після помилки COMMIT фактично стає ROLLBACK
BEGIN;
INSERT INTO categories (category_name) VALUES ('Тимчасова категорія');
UPDATE products SET units_in_stock = -1 WHERE product_id = 1;   -- [ПОМИЛКА] ПЕРЕДБАЧЕНО
COMMIT;   -- повідомить ROLLBACK
SELECT * FROM categories WHERE category_name = 'Тимчасова категорія';   -- 0 рядків


-- #####################################################################
-- РІВЕНЬ 2
-- #####################################################################

-- ---------------------------------------------------------------------
-- 2.1 Нові таблиці: бронювання та історія змін
-- ---------------------------------------------------------------------
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

-- ---------------------------------------------------------------------
-- 2.2 Складна транзакція: скасування замовлення
--   1) блокуємо замовлення  2) міняємо статус  3) повертаємо товар на склад
--   4) виконуємо найстарше активне бронювання по повернутих товарах
-- ---------------------------------------------------------------------
BEGIN;

SELECT order_id, order_status FROM orders WHERE order_id = 28 FOR UPDATE;

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

COMMIT;

SELECT order_id, order_status FROM orders WHERE order_id = 28;

-- ---------------------------------------------------------------------
-- 2.3 Каскадні операції (для всіх типів зв'язків)
--   CASCADE   : order_items.order_id -> orders
--   SET NULL  : products.supplier_id, employees.reports_to
--   RESTRICT  : order_items.product_id (не можна видалити проданий товар)
--   ON UPDATE CASCADE : orders.customer_id
-- ---------------------------------------------------------------------
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
DELETE FROM orders WHERE order_id = 26;
SELECT COUNT(*) AS items_after  FROM order_items WHERE order_id = 26;   -- 0
ROLLBACK;

-- (б) ON DELETE SET NULL: видалення постачальника
BEGIN;
SELECT COUNT(*) AS products_of_supplier8 FROM products WHERE supplier_id = 8;
DELETE FROM suppliers WHERE supplier_id = 8;
SELECT product_id, supplier_id FROM products WHERE product_id IN (14, 21);   -- supplier_id = NULL
ROLLBACK;

-- (в) ON DELETE SET NULL (самопосилання): видалення керівника
BEGIN;
DELETE FROM employees WHERE employee_id = 1;
SELECT employee_id, reports_to FROM employees WHERE employee_id IN (2,3,4);  -- reports_to = NULL
ROLLBACK;

-- (г) ON UPDATE CASCADE: зміна PK клієнта оновлює замовлення
BEGIN;
UPDATE customers SET customer_id = 1001 WHERE customer_id = 15;
SELECT order_id, customer_id FROM orders WHERE customer_id = 1001;
ROLLBACK;

-- (д) RESTRICT: проданий товар видалити не можна
-- [ПОМИЛКА] ПЕРЕДБАЧЕНО
BEGIN;
DELETE FROM products WHERE product_id = 1;
ROLLBACK;

-- (е) CASCADE на бронюваннях: видалення клієнта прибирає його бронювання
BEGIN;
DELETE FROM product_reservations WHERE customer_id = 1;   -- вручну прибираємо
ROLLBACK;

-- ---------------------------------------------------------------------
-- 2.4 SAVEPOINT: частковий відкат
-- ---------------------------------------------------------------------
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

-- ---------------------------------------------------------------------
-- 2.5 Тригери: автоматична історія змін
-- ---------------------------------------------------------------------
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

-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: тригер забороняє змінювати доставлене замовлення
UPDATE orders SET order_status = 'pending' WHERE order_id = 1;

SELECT log_id, table_name, operation, record_id,
       old_data->>'unit_price' AS old_price, new_data->>'unit_price' AS new_price, changed_at, changed_by
FROM audit_log
ORDER BY log_id DESC;

-- ---------------------------------------------------------------------
-- 2.6 Блокування записів (потрібні ДВА з'єднання)
-- ---------------------------------------------------------------------
-- Сесія A:
--   BEGIN;
--   SELECT * FROM products WHERE product_id = 1 FOR UPDATE;   -- рядок заблоковано
--   SELECT pg_sleep(20);
--   UPDATE products SET units_in_stock = units_in_stock - 1 WHERE product_id = 1;
--   COMMIT;
--
-- Сесія B (запустити відразу після початку A):
--   BEGIN;
--   SELECT * FROM products WHERE product_id = 1 FOR UPDATE;           -- ЧЕКАЄ, доки A завершиться
--   -- інші режими:
--   -- SELECT * FROM products WHERE product_id = 1 FOR UPDATE NOWAIT;      -- одразу помилка 55P03
--   -- SELECT * FROM products WHERE units_in_stock > 0 FOR UPDATE SKIP LOCKED; -- пропускає заблоковані
--   UPDATE products SET units_in_stock = units_in_stock - 1 WHERE product_id = 1;
--   COMMIT;
--
-- Сесія C (поки A тримає блокування) - спостереження:
SELECT pid, state, wait_event_type, pg_blocking_pids(pid) AS blocked_by, left(query, 55) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
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


-- #####################################################################
-- РІВЕНЬ 3
-- #####################################################################

-- ---------------------------------------------------------------------
-- 3.1 Версіонування записів (товари)
-- ---------------------------------------------------------------------
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

SELECT get_product_version(3, clock_timestamp()::TIMESTAMP) ->> 'unit_price' AS price_now;
SELECT get_product_version(3, (SELECT valid_from FROM product_versions
                               WHERE product_id = 3 AND version_number = 1) + INTERVAL '1 millisecond')
       ->> 'unit_price' AS price_in_v1;

SELECT restore_product_version(3, 1);
SELECT product_id, unit_price, units_in_stock FROM products WHERE product_id = 3;

-- ---------------------------------------------------------------------
-- 3.2 Відкладені операції
-- ---------------------------------------------------------------------
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

-- Автозапуск щохвилини (Supabase / розширення pg_cron):
-- CREATE EXTENSION IF NOT EXISTS pg_cron;
-- SELECT cron.schedule('exec-scheduled', '* * * * *', 'SELECT execute_scheduled_operations()');
-- SELECT cron.schedule('daily-checkpoint', '0 3 * * *', $$SELECT create_checkpoint('daily','pg_cron')$$);

-- ---------------------------------------------------------------------
-- 3.3 Збережені процедури / функції бізнес-логіки
-- ---------------------------------------------------------------------
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

SELECT place_order(3, 2, '[{"product_id":3,"quantity":3},{"product_id":16,"quantity":1,"discount":0.05}]'::jsonb)
       AS new_order_id;    -- компанія, 3 шт => автознижка 10 % на товар 3

SELECT oi.product_id, oi.quantity, oi.discount FROM order_items oi
WHERE oi.order_id = (SELECT MAX(order_id) FROM orders);

SELECT product_id, units_in_stock FROM products WHERE product_id IN (3, 16);   -- залишки зменшились

DO $$
DECLARE v_id INTEGER := (SELECT MAX(order_id) FROM orders);   -- CALL не приймає підзапит
BEGIN
    CALL cancel_order(v_id);
END $$;
SELECT product_id, units_in_stock FROM products WHERE product_id IN (3, 16);   -- залишки повернулись

-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: недостатньо товару -> усе замовлення відкочується атомарно
SELECT place_order(1, 2, '[{"product_id":3,"quantity":1},{"product_id":23,"quantity":1000}]'::jsonb);
SELECT COUNT(*) AS orders_total FROM orders;   -- нового замовлення немає

-- [ПОМИЛКА] ПЕРЕДБАЧЕНО: доставлене замовлення скасувати не можна
CALL cancel_order(1);

-- Відкладена операція (тепер, коли процедури існують)
INSERT INTO scheduled_operations (operation_type, operation_data, scheduled_for) VALUES
  ('SET_PRICE', '{"product_id": 3, "unit_price": 12499}', LOCALTIMESTAMP - INTERVAL '1 minute'),
  ('EXPIRE_RESERVATIONS', '{}', LOCALTIMESTAMP - INTERVAL '1 minute'),
  ('BAD_TYPE', '{}', LOCALTIMESTAMP - INTERVAL '1 minute'),
  ('SET_PRICE', '{"product_id": 3, "unit_price": 12999}', LOCALTIMESTAMP + INTERVAL '1 day');

SELECT execute_scheduled_operations() AS executed;
SELECT operation_id, operation_type, scheduled_for, status, error_message FROM scheduled_operations ORDER BY 1;

-- ---------------------------------------------------------------------
-- 3.4 Ролі та права на операції модифікації
-- ---------------------------------------------------------------------
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
GRANT INSERT, UPDATE ON customers, orders, order_items, product_reservations TO tm_sales;
GRANT UPDATE (units_in_stock, units_on_order) ON products TO tm_sales;     -- права на рівні стовпців
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO tm_sales;

-- tm_manager: + керування каталогом та видалення бронювань
GRANT tm_sales TO tm_manager;
GRANT INSERT, UPDATE, DELETE ON products TO tm_manager;
GRANT DELETE ON product_reservations TO tm_manager;

-- tm_admin: усе
GRANT tm_manager TO tm_admin;
GRANT ALL ON ALL TABLES IN SCHEMA public TO tm_admin;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO tm_admin;

-- Щоб поточний користувач міг робити SET ROLE (потрібно в PostgreSQL 16+ / Supabase):
GRANT tm_viewer, tm_sales, tm_manager, tm_admin TO CURRENT_USER;

-- Тестування
SET ROLE tm_viewer;
SELECT COUNT(*) FROM products;                                         -- OK
UPDATE products SET units_in_stock = 5 WHERE product_id = 1;           -- [ПОМИЛКА] ПЕРЕДБАЧЕНО
RESET ROLE;

SET ROLE tm_sales;
UPDATE products SET units_in_stock = units_in_stock WHERE product_id = 1;   -- OK
UPDATE products SET unit_price = 1 WHERE product_id = 1;               -- [ПОМИЛКА] ПЕРЕДБАЧЕНО (немає права на цей стовпець)
DELETE FROM orders WHERE order_id = 1;                                 -- [ПОМИЛКА] ПЕРЕДБАЧЕНО
SELECT place_order(2, 2, '[{"product_id":16,"quantity":1}]'::jsonb);  -- OK: оформлення через функцію
RESET ROLE;

SET ROLE tm_manager;
UPDATE products SET unit_price = unit_price WHERE product_id = 1;      -- OK
DELETE FROM customers WHERE customer_id = 1;                           -- [ПОМИЛКА] ПЕРЕДБАЧЕНО
RESET ROLE;

-- Перевірка: хто що може
SELECT grantee, table_name, string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privileges
FROM information_schema.table_privileges
WHERE grantee LIKE 'tm\_%' AND table_name IN ('products', 'orders', 'order_items')
GROUP BY grantee, table_name ORDER BY grantee, table_name;

-- ---------------------------------------------------------------------
-- 3.5 Контрольні точки та відновлення
-- ---------------------------------------------------------------------
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

SELECT create_checkpoint('manual', 'перед видаленням');

DELETE FROM products WHERE product_name = 'ТЕСТ-L3: товар для відновлення';   -- спрацює авто-точка
SELECT COUNT(*) AS found_after_delete FROM products WHERE product_name = 'ТЕСТ-L3: товар для відновлення';  -- 0

SELECT checkpoint_id, name, reason, created_at FROM checkpoints ORDER BY checkpoint_id;

SELECT restore_products_from_checkpoint((SELECT MAX(checkpoint_id) FROM checkpoints));
SELECT product_id, product_name, unit_price FROM products WHERE product_name = 'ТЕСТ-L3: товар для відновлення';  -- повернувся

-- Для повного відновлення БД після збою (PITR) на рівні сервера:
--   wal_level = replica; archive_mode = on; archive_command = 'cp %p /backup/wal/%f'
--   pg_basebackup -D /backup/base -Ft -z -P
--   recovery: restore_command = 'cp /backup/wal/%f %p'; recovery_target_time = '2024-08-20 12:00:00'

-- ---------------------------------------------------------------------
-- 3.6 Інтерфейс перегляду та відкату змін
-- ---------------------------------------------------------------------
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
SELECT * FROM v_product_price_history WHERE product_id = 3 ORDER BY version_number;

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

SELECT * FROM v_change_history WHERE changed_at >= (SELECT t FROM _mark) ORDER BY log_id;

SELECT revert_product_changes((SELECT t FROM _mark)) AS reverted_products;
SELECT product_id, unit_price, units_in_stock FROM products WHERE product_id IN (1, 20, 21, 22);  -- стан до змін

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
