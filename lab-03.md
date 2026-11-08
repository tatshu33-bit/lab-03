# Лабораторна робота 03 Модифікація даних та транзакції

## 🎯 Мета роботи

Здобути практичні навички безпечної модифікації даних у реляційних базах даних, опанувати механізми транзакцій для забезпечення цілісності даних, навчитися працювати з обмеженнями цілісності та каскадними операціями при зміні взаємопов'язаних записів.

## ✅ Завдання

### Рівень 1

Створити базу даних для управління бібліотекою з наступними таблицями: автори, книги, читачі, видачі книг. Реалізувати базові операції модифікації даних.

**Конкретні завдання:**

1. Створити структуру бази даних з відповідними обмеженнями цілісності.
2. Реалізувати операції додавання нових записів з перевіркою обмежень.
3. Виконати безпечне оновлення записів з використанням умов WHERE.
4. Здійснити видалення записів з урахуванням зовнішніх ключів.
5. Використати прості транзакції для групування операцій.
6. Продемонструвати роботу механізму ROLLBACK при помилках.

### Рівень 2

Виконати завдання рівня 1. Розширити базу даних додатковими сутностями та реалізувати складніші сценарії модифікації з використанням транзакцій.

**Конкретні завдання:**

1. Додати таблиці для обліку бронювань книг та історії змін.
2. Реалізувати складні транзакції з множинними операціями модифікації.
3. Впровадити каскадні операції для автоматичного оновлення пов'язаних записів.
4. Використати точки збереження SAVEPOINT для часткового відкату транзакцій.
5. Створити тригери для автоматичного ведення історії змін.
6. Реалізувати механізми блокування записів для паралельного доступу.

### Рівень 3

Виконати завдання рівня 2. Розробити повноцінну систему управління даними з розширеними можливостями модифікації та аудиту змін.

**Конкретні завдання:**

1. Впровадити систему версіонування записів з можливістю відстеження всіх змін.
2. Створити механізм відкладених операцій з автоматичним виконанням.
3. Реалізувати складну бізнес-логіку через збережені процедури.
4. Впровадити систему прав доступу на рівні операцій модифікації.
5. Розробити механізм автоматичного відновлення даних після збоїв.
6. Створити інтерфейс для перегляду та відкату змін за певний період.


## 🖥️ Програмне забезпечення

- [PostgreSQL](https://www.postgresql.org)

!!! tip "Рекомендації"

    Для виконання лабораторної роботи рекомендується використовувати БД в Docker-контейнері

    - [Docker Desktop](https://www.docker.com/products/docker-desktop)
    - [postgres official images](https://hub.docker.com/_/postgres)

## 👥 Форма виконання роботи

Форма виконання роботи **індивідуальна**.

## 📝 Критерії оцінювання

- середній рівень (оцінка "задовільно") - виконано завдання рівня 1.
    - Створено коректну структуру бази даних з усіма необхідними обмеженнями цілісності.
    - Реалізовано базові операції INSERT для всіх таблиць з коректними даними.
    - Виконано безпечні операції UPDATE з обов'язковою умовою WHERE.
    - Здійснено операції DELETE з перевіркою впливу на пов'язані записи.
    - Використано прості транзакції для групування операцій з COMMIT.
    - Продемонстровано роботу ROLLBACK при виникненні помилок.
    - Підготовлено звіт з описом виконаних операцій та скріншотами результатів.
- достатній рівень (оцінка "добре") - виконано всі вимоги рівня 2.
    - Створено додаткові таблиці з правильними зв'язками та обмеженнями.
    - Реалізовано складні транзакції з множинними операціями різних типів.
    - Впроваджено каскадні операції для всіх типів зв'язків.
    - Використано точки збереження SAVEPOINT для гнучкого управління транзакціями.
    - Створено тригери для автоматичного ведення історії змін.
    - Продемонстровано механізми блокування при паралельному доступі.
    - Підготовлено розширений звіт з аналізом роботи транзакцій та обмежень.
- високий рівень (оцінка "відмінно") - виконано завдання рівня 3.
    - Впроваджено систему версіонування з можливістю отримання історичних станів.
    - Реалізовано механізм відкладених операцій з автоматичним виконанням.
    - Створено збережені процедури для реалізації складної бізнес-логіки.
    - Впроваджено систему ролей з різними правами доступу.
    - Розроблено механізм автоматичного створення контрольних точок.
    - Створено зручний інтерфейс для перегляду історії змін через представлення.

## ⏰ Політика щодо дедлайнів

При порушенні встановленого терміну здачі лабораторної роботи максимальна можлива оцінка становить "добре", незалежно від якості виконаної роботи. Винятки можливі лише за поважних причин, підтверджених документально.

## 📚 Теоретичні відомості

### Операції модифікації даних

Мова SQL надає три основні оператори для модифікації даних у таблицях: INSERT для додавання нових записів, UPDATE для зміни існуючих записів та DELETE для видалення записів.

#### Оператор INSERT

Оператор INSERT використовується для додавання нових рядків до таблиці. Існує кілька варіантів його використання.

**Вставка одного запису з явним переліком стовпців:**

```sql
INSERT INTO books (isbn, title, author_id, publication_year, genre)
VALUES ('978-0-123456-78-9', 'Основи баз даних', 1, 2023, 'Технічна література');
```

**Вставка без переліку стовпців:**

```sql
INSERT INTO books
VALUES (1, '978-0-123456-78-9', 'Основи баз даних', 1, 2023, 'Технічна література', 5);
```

У цьому випадку необхідно вказати значення для всіх стовпців у порядку їх визначення в таблиці.

**Вставка множинних записів:**

```sql
INSERT INTO authors (first_name, last_name, birth_year, country)
VALUES
    ('Іван', 'Петренко', 1975, 'Україна'),
    ('Марія', 'Коваленко', 1982, 'Україна'),
    ('Олександр', 'Сидоров', 1968, 'Україна');
```

**Вставка даних з результатів запиту:**

```sql
INSERT INTO archive_books (isbn, title, archive_date)
SELECT isbn, title, CURRENT_DATE
FROM books
WHERE publication_year < 2000;
```

#### Оператор UPDATE

Оператор UPDATE використовується для зміни значень у існуючих записах. Обов'язковою частиною безпечного UPDATE є умова WHERE.

**Оновлення одного поля:**

```sql
UPDATE books
SET available_copies = available_copies - 1
WHERE isbn = '978-0-123456-78-9';
```

**Оновлення кількох полів одночасно:**

```sql
UPDATE readers
SET email = 'new.email@example.com',
    phone = '+380501234567',
    updated_at = CURRENT_TIMESTAMP
WHERE reader_id = 15;
```

**Оновлення на основі підзапиту:**

```sql
UPDATE books
SET genre = 'Класична література'
WHERE author_id IN (
    SELECT author_id
    FROM authors
    WHERE birth_year < 1900
);
```

**Оновлення з використанням JOIN:**

```sql
UPDATE books b
SET b.available_copies = b.available_copies + 1
FROM lendings l
WHERE b.book_id = l.book_id
    AND l.return_date = CURRENT_DATE;
```

#### Оператор DELETE

Оператор DELETE видаляє записи з таблиці. Використання умови WHERE є критично важливим для уникнення випадкового видалення всіх даних.

**Видалення конкретних записів:**

```sql
DELETE FROM lendings
WHERE return_date < CURRENT_DATE - INTERVAL '1 year';
```

**Видалення з підзапитом:**

```sql
DELETE FROM books
WHERE author_id IN (
    SELECT author_id
    FROM authors
    WHERE country = 'Невідома'
);
```

**Видалення з використанням JOIN:**

```sql
DELETE l
FROM lendings l
INNER JOIN readers r ON l.reader_id = r.reader_id
WHERE r.status = 'Blocked';
```

### Безпечні операції модифікації

Безпечна модифікація даних передбачає використання кількох важливих практик.

#### Використання умови WHERE

Найважливішою практикою є завжди використовувати умову WHERE в операціях UPDATE та DELETE, якщо не потрібно модифікувати всі записи в таблиці.

**Небезпечна операція без WHERE:**

```sql
-- УВАГА: видалить всі книги з таблиці
DELETE FROM books;
```

**Безпечна операція з WHERE:**

```sql
-- Видалить тільки книги певного автора
DELETE FROM books
WHERE author_id = 5;
```

#### Перевірка перед модифікацією

Перед виконанням операції модифікації корисно виконати SELECT з тією ж умовою WHERE для перевірки, які записи будуть змінені.

```sql
-- Спочатку перевіряємо, які записи будуть оновлені
SELECT book_id, title, available_copies
FROM books
WHERE genre = 'Художня література';

-- Після перевірки виконуємо UPDATE
UPDATE books
SET available_copies = available_copies + 5
WHERE genre = 'Художня література';
```

#### Використання RETURNING

В PostgreSQL можна використовувати конструкцію RETURNING для отримання даних модифікованих записів.

```sql
UPDATE books
SET available_copies = available_copies - 1
WHERE isbn = '978-0-123456-78-9'
RETURNING book_id, title, available_copies;
```

### Транзакції

Транзакція є логічною одиницею роботи, яка містить одну або більше операцій з базою даних. Транзакції забезпечують виконання властивостей ACID.

#### Властивості ACID

**Атомарність (Atomicity)** означає, що транзакція виконується повністю або не виконується взагалі. Якщо одна з операцій в транзакції не вдається, всі попередні операції скасовуються.

**Консистентність (Consistency)** гарантує, що транзакція переводить базу даних з одного узгодженого стану в інший, зберігаючи всі обмеження цілісності.

**Ізольованість (Isolation)** забезпечує, що одночасні транзакції не впливають одна на одну. Результат виконання паралельних транзакцій еквівалентний їх послідовному виконанню.

**Довговічність (Durability)** гарантує, що після підтвердження транзакції її результати зберігаються навіть у випадку збою системи.

#### Основні команди транзакцій

**BEGIN або START TRANSACTION** розпочинає нову транзакцію.

```sql
BEGIN;
-- або
START TRANSACTION;
```

**COMMIT** підтверджує всі зміни, виконані в поточній транзакції.

```sql
BEGIN;
INSERT INTO authors (first_name, last_name) VALUES ('Тарас', 'Шевченко');
INSERT INTO books (title, author_id) VALUES ('Кобзар', LASTVAL());
COMMIT;
```

**ROLLBACK** скасовує всі зміни, виконані в поточній транзакції.

```sql
BEGIN;
DELETE FROM books WHERE publication_year < 1950;
-- Помилка або зміна рішення
ROLLBACK;
```

#### Точки збереження SAVEPOINT

Точки збереження дозволяють частково відкотити транзакцію до певної точки без скасування всієї транзакції.

```sql
BEGIN;

INSERT INTO authors (first_name, last_name) VALUES ('Леся', 'Українка');

SAVEPOINT after_author_insert;

INSERT INTO books (title, author_id) VALUES ('Лісова пісня', LASTVAL());

-- Якщо виникла помилка з книгою, можна відкотити тільки вставку книги
ROLLBACK TO SAVEPOINT after_author_insert;

-- Автор залишиться доданим, можна додати іншу книгу
INSERT INTO books (title, author_id) VALUES ('Камінний господар', LASTVAL());

COMMIT;
```

#### Рівні ізоляції транзакцій

SQL стандарт визначає чотири рівні ізоляції транзакцій, які визначають, які аномалії можуть виникати при паралельному виконанні транзакцій.

**READ UNCOMMITTED** дозволяє читати незафіксовані зміни інших транзакцій. Можливі всі типи аномалій.

**READ COMMITTED** дозволяє читати тільки зафіксовані дані. Запобігає брудному читанню, але можливі неповторювані читання та фантомні записи.

**REPEATABLE READ** гарантує, що повторне читання в межах транзакції поверне ті самі дані. Запобігає брудному та неповторюваному читанню, але можливі фантомні записи.

**SERIALIZABLE** найвищий рівень ізоляції. Транзакції виконуються так, ніби вони виконуються послідовно. Запобігає всім аномаліям.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- операції з даними
COMMIT;
```

### Обмеження цілісності

Обмеження цілісності є правилами, які забезпечують коректність та узгодженість даних у базі даних.

#### PRIMARY KEY

Первинний ключ унікально ідентифікує кожен запис у таблиці. Не може містити NULL значення.

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    isbn VARCHAR(17) UNIQUE NOT NULL,
    title VARCHAR(200) NOT NULL
);
```

#### FOREIGN KEY

Зовнішній ключ створює зв'язок між таблицями та забезпечує посилальну цілісність.

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    author_id INTEGER NOT NULL,
    FOREIGN KEY (author_id) REFERENCES authors(author_id)
);
```

#### UNIQUE

Обмеження унікальності забезпечує, що значення в стовпці або комбінації стовпців є унікальними.

```sql
CREATE TABLE readers (
    reader_id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    library_card_number VARCHAR(20) UNIQUE NOT NULL
);
```

#### CHECK

Обмеження перевірки забезпечує, що значення в стовпці задовольняє певну умову.

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    publication_year INTEGER CHECK (publication_year >= 1450 AND publication_year <= EXTRACT(YEAR FROM CURRENT_DATE)),
    available_copies INTEGER CHECK (available_copies >= 0)
);
```

#### NOT NULL

Обмеження забороняє NULL значення в стовпці.

```sql
CREATE TABLE authors (
    author_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL
);
```

### Каскадні операції

Каскадні операції автоматично виконують певні дії над пов'язаними записами при зміні або видаленні батьківського запису.

#### ON DELETE CASCADE

Автоматично видаляє всі залежні записи при видаленні батьківського запису.

```sql
CREATE TABLE lendings (
    lending_id SERIAL PRIMARY KEY,
    book_id INTEGER NOT NULL,
    reader_id INTEGER NOT NULL,
    FOREIGN KEY (book_id) REFERENCES books(book_id) ON DELETE CASCADE,
    FOREIGN KEY (reader_id) REFERENCES readers(reader_id) ON DELETE CASCADE
);
```

При видаленні книги з таблиці books автоматично видаляться всі записи про видачі цієї книги.

#### ON UPDATE CASCADE

Автоматично оновлює значення зовнішнього ключа при зміні значення первинного ключа в батьківській таблиці.

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    author_id INTEGER NOT NULL,
    FOREIGN KEY (author_id) REFERENCES authors(author_id) ON UPDATE CASCADE
);
```

#### ON DELETE SET NULL

Встановлює NULL у зовнішньому ключі при видаленні батьківського запису.

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    publisher_id INTEGER,
    FOREIGN KEY (publisher_id) REFERENCES publishers(publisher_id) ON DELETE SET NULL
);
```

#### ON DELETE SET DEFAULT

Встановлює значення за замовчуванням у зовнішньому ключі при видаленні батьківського запису.

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    category_id INTEGER DEFAULT 1,
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE SET DEFAULT
);
```

#### ON DELETE RESTRICT

Забороняє видалення батьківського запису, якщо існують залежні записи. Це поведінка за замовчуванням.

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    author_id INTEGER NOT NULL,
    FOREIGN KEY (author_id) REFERENCES authors(author_id) ON DELETE RESTRICT
);
```

### Тригери для аудиту змін

Тригери дозволяють автоматично виконувати певні дії при виникненні подій модифікації даних.

```sql
-- Таблиця для зберігання історії змін
CREATE TABLE books_audit (
    audit_id SERIAL PRIMARY KEY,
    book_id INTEGER NOT NULL,
    operation VARCHAR(10) NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by VARCHAR(100) DEFAULT CURRENT_USER
);

-- Функція тригера
CREATE OR REPLACE FUNCTION audit_books_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO books_audit (book_id, operation, new_data)
        VALUES (NEW.book_id, 'INSERT', row_to_json(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO books_audit (book_id, operation, old_data, new_data)
        VALUES (NEW.book_id, 'UPDATE', row_to_json(OLD), row_to_json(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO books_audit (book_id, operation, old_data)
        VALUES (OLD.book_id, 'DELETE', row_to_json(OLD));
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Створення тригера
CREATE TRIGGER books_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON books
FOR EACH ROW
EXECUTE FUNCTION audit_books_changes();
```

## ▶️ Хід роботи

### Підготовка середовища

1. Переконатися, що PostgreSQL встановлено та запущено на локальному комп'ютері або є доступ до віддаленого сервера.
2. Відкрити інструмент для роботи з базою даних: pgAdmin, DBeaver або командний рядок psql.
3. Створити нову базу даних для лабораторної роботи.
    ```sql
    CREATE DATABASE library_lab3;
    ```
4. Підключитися до створеної бази даних.

### Рівень 1

#### Крок 1. Створення структури бази даних

Створити таблиці для системи управління бібліотекою з відповідними обмеженнями цілісності.

```sql
-- Таблиця авторів
CREATE TABLE authors (
    author_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    birth_year INTEGER CHECK (birth_year >= 1000 AND birth_year <= EXTRACT(YEAR FROM CURRENT_DATE)),
    country VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблиця книг
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    isbn VARCHAR(17) UNIQUE NOT NULL,
    title VARCHAR(200) NOT NULL,
    author_id INTEGER NOT NULL,
    publication_year INTEGER CHECK (publication_year >= 1450),
    genre VARCHAR(50),
    total_copies INTEGER DEFAULT 1 CHECK (total_copies >= 0),
    available_copies INTEGER DEFAULT 1 CHECK (available_copies >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (author_id) REFERENCES authors(author_id) ON DELETE RESTRICT,
    CHECK (available_copies <= total_copies)
);

-- Таблиця читачів
CREATE TABLE readers (
    reader_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    library_card_number VARCHAR(20) UNIQUE NOT NULL,
    registration_date DATE DEFAULT CURRENT_DATE,
    status VARCHAR(20) DEFAULT 'Active' CHECK (status IN ('Active', 'Blocked', 'Inactive'))
);

-- Таблиця видач книг
CREATE TABLE lendings (
    lending_id SERIAL PRIMARY KEY,
    book_id INTEGER NOT NULL,
    reader_id INTEGER NOT NULL,
    lending_date DATE DEFAULT CURRENT_DATE,
    due_date DATE NOT NULL,
    return_date DATE,
    FOREIGN KEY (book_id) REFERENCES books(book_id) ON DELETE RESTRICT,
    FOREIGN KEY (reader_id) REFERENCES readers(reader_id) ON DELETE RESTRICT,
    CHECK (due_date > lending_date),
    CHECK (return_date IS NULL OR return_date >= lending_date)
);
```

#### Крок 2. Додавання тестових даних

Вставити початкові дані у створені таблиці.

```sql
-- Додавання авторів
INSERT INTO authors (first_name, last_name, birth_year, country)
VALUES
    ('Тарас', 'Шевченко', 1814, 'Україна'),
    ('Іван', 'Франко', 1856, 'Україна'),
    ('Леся', 'Українка', 1871, 'Україна'),
    ('Михайло', 'Коцюбинський', 1864, 'Україна'),
    ('Панас', 'Мирний', 1849, 'Україна');

-- Додавання книг
INSERT INTO books (isbn, title, author_id, publication_year, genre, total_copies, available_copies)
VALUES
    ('978-966-03-4561-2', 'Кобзар', 1, 1840, 'Поезія', 10, 8),
    ('978-966-03-4562-9', 'Захар Беркут', 2, 1883, 'Історична проза', 5, 4),
    ('978-966-03-4563-6', 'Лісова пісня', 3, 1911, 'Драма', 7, 7),
    ('978-966-03-4564-3', 'Тіні забутих предків', 4, 1911, 'Повість', 6, 5),
    ('978-966-03-4565-0', 'Хіба ревуть воли, як ясла повні', 5, 1880, 'Повість', 4, 4);

-- Додавання читачів
INSERT INTO readers (first_name, last_name, email, phone, library_card_number)
VALUES
    ('Олена', 'Петренко', 'olena.petrenko@email.com', '+380501234567', 'LIB-2024-001'),
    ('Андрій', 'Коваленко', 'andrii.kovalenko@email.com', '+380502345678', 'LIB-2024-002'),
    ('Марія', 'Сидоренко', 'maria.sydorenko@email.com', '+380503456789', 'LIB-2024-003'),
    ('Іван', 'Мельник', 'ivan.melnyk@email.com', '+380504567890', 'LIB-2024-004');
```

#### Крок 3. Безпечне оновлення записів

Виконати операції UPDATE з обов'язковою перевіркою через SELECT.

```sql
-- Перевірка перед оновленням
SELECT book_id, title, available_copies
FROM books
WHERE book_id = 1;

-- Оновлення кількості доступних примірників
UPDATE books
SET available_copies = available_copies - 1
WHERE book_id = 1 AND available_copies > 0;

-- Перевірка результату
SELECT book_id, title, available_copies
FROM books
WHERE book_id = 1;
```

#### Крок 4. Операції з транзакціями

Виконати видачу книги читачу як транзакцію з кількох операцій.

```sql
BEGIN;

-- Видача книги
INSERT INTO lendings (book_id, reader_id, due_date)
VALUES (1, 1, CURRENT_DATE + INTERVAL '14 days');

-- Зменшення кількості доступних примірників
UPDATE books
SET available_copies = available_copies - 1
WHERE book_id = 1 AND available_copies > 0;

-- Перевірка результатів перед підтвердженням
SELECT * FROM lendings WHERE lending_id = LASTVAL();
SELECT available_copies FROM books WHERE book_id = 1;

COMMIT;
```

#### Крок 5. Демонстрація ROLLBACK

Показати відкат транзакції при виникненні помилки.

```sql
BEGIN;

-- Спроба видалити автора, у якого є книги
DELETE FROM authors WHERE author_id = 1;

-- Ця операція викличе помилку через обмеження FOREIGN KEY
-- Транзакція автоматично відкотиться

ROLLBACK;

-- Перевірка, що дані не змінилися
SELECT * FROM authors WHERE author_id = 1;
```

#### Крок 6. Безпечне видалення

Виконати видалення з перевіркою.

```sql
-- Перевірка перед видаленням
SELECT * FROM readers WHERE status = 'Inactive' AND reader_id NOT IN (
    SELECT DISTINCT reader_id FROM lendings WHERE return_date IS NULL
);

-- Видалення неактивних читачів без боргів
DELETE FROM readers
WHERE status = 'Inactive' AND reader_id NOT IN (
    SELECT DISTINCT reader_id FROM lendings WHERE return_date IS NULL
);
```

### Рівень 2

#### Крок 1. Додавання нових таблиць

Створити таблиці для бронювань та історії змін.

```sql
-- Таблиця бронювань
CREATE TABLE reservations (
    reservation_id SERIAL PRIMARY KEY,
    book_id INTEGER NOT NULL,
    reader_id INTEGER NOT NULL,
    reservation_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expiration_date TIMESTAMP NOT NULL,
    status VARCHAR(20) DEFAULT 'Active' CHECK (status IN ('Active', 'Fulfilled', 'Cancelled', 'Expired')),
    FOREIGN KEY (book_id) REFERENCES books(book_id) ON DELETE CASCADE,
    FOREIGN KEY (reader_id) REFERENCES readers(reader_id) ON DELETE CASCADE
);

-- Таблиця історії змін
CREATE TABLE audit_log (
    log_id SERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    operation VARCHAR(10) NOT NULL,
    record_id INTEGER NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by VARCHAR(100) DEFAULT CURRENT_USER
);
```

#### Крок 2. Складні транзакції

Реалізувати процес повернення книги з оновленням кількох таблиць.

```sql
BEGIN;

-- Оновлення запису про видачу
UPDATE lendings
SET return_date = CURRENT_DATE
WHERE lending_id = 1 AND return_date IS NULL;

-- Збільшення кількості доступних примірників
UPDATE books
SET available_copies = available_copies + 1
WHERE book_id = (SELECT book_id FROM lendings WHERE lending_id = 1);

-- Перевірка чи є активні бронювання на цю книгу
SELECT reservation_id, reader_id
FROM reservations
WHERE book_id = (SELECT book_id FROM lendings WHERE lending_id = 1)
    AND status = 'Active'
ORDER BY reservation_date
LIMIT 1;

-- Якщо є бронювання, змінюємо його статус
UPDATE reservations
SET status = 'Fulfilled'
WHERE reservation_id = (
    SELECT reservation_id
    FROM reservations
    WHERE book_id = (SELECT book_id FROM lendings WHERE lending_id = 1)
        AND status = 'Active'
    ORDER BY reservation_date
    LIMIT 1
);

COMMIT;
```

#### Крок 3. Використання SAVEPOINT

Демонструвати частковий відкат транзакції.

```sql
BEGIN;

-- Додаємо нового автора
INSERT INTO authors (first_name, last_name, birth_year, country)
VALUES ('Ольга', 'Кобилянська', 1863, 'Україна');

SAVEPOINT after_author;

-- Додаємо книгу
INSERT INTO books (isbn, title, author_id, publication_year, genre, total_copies, available_copies)
VALUES ('978-966-03-4566-7', 'Земля', LASTVAL(), 1902, 'Роман', 3, 3);

SAVEPOINT after_book;

-- Спроба додати дублікат ISBN викличе помилку
INSERT INTO books (isbn, title, author_id, publication_year, genre)
VALUES ('978-966-03-4566-7', 'Інша книга', LASTVAL(), 1905, 'Роман');

-- Відкат до точки після додавання автора, але до помилки з книгою
ROLLBACK TO SAVEPOINT after_author;

-- Додаємо книгу з правильним ISBN
INSERT INTO books (isbn, title, author_id, publication_year, genre, total_copies, available_copies)
VALUES ('978-966-03-4567-4', 'Земля', LASTVAL(), 1902, 'Роман', 3, 3);

COMMIT;
```

#### Крок 4. Каскадні операції

Продемонструвати роботу каскадного видалення.

```sql
-- Створюємо тестові дані для демонстрації каскаду
BEGIN;

INSERT INTO authors (first_name, last_name, birth_year, country)
VALUES ('Тестовий', 'Автор', 1900, 'Україна');

INSERT INTO books (isbn, title, author_id, publication_year, genre, total_copies, available_copies)
VALUES ('978-966-00-0000-0', 'Тестова книга', LASTVAL(), 2000, 'Тест', 1, 0);

INSERT INTO reservations (book_id, reader_id, expiration_date)
VALUES (LASTVAL(), 1, CURRENT_TIMESTAMP + INTERVAL '7 days');

COMMIT;

-- Перевірка зв'язків
SELECT b.book_id, b.title, r.reservation_id
FROM books b
LEFT JOIN reservations r ON b.book_id = r.book_id
WHERE b.isbn = '978-966-00-0000-0';

-- Видалення книги з каскадним видаленням бронювань
DELETE FROM books WHERE isbn = '978-966-00-0000-0';

-- Перевірка, що бронювання також видалилися
SELECT * FROM reservations WHERE book_id NOT IN (SELECT book_id FROM books);
```

#### Крок 5. Тригери для аудиту

Створити тригер для автоматичного логування змін.

```sql
-- Функція тригера
CREATE OR REPLACE FUNCTION log_books_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, record_id, new_data)
        VALUES ('books', 'INSERT', NEW.book_id, row_to_json(NEW)::jsonb);
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_data, new_data)
        VALUES ('books', 'UPDATE', NEW.book_id, row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, record_id, old_data)
        VALUES ('books', 'DELETE', OLD.book_id, row_to_json(OLD)::jsonb);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Створення тригера
CREATE TRIGGER books_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON books
FOR EACH ROW
EXECUTE FUNCTION log_books_changes();

-- Тестування тригера
UPDATE books SET available_copies = available_copies + 1 WHERE book_id = 1;

-- Перегляд логу
SELECT log_id, operation, record_id,
      new_data->>'title' as title,
      old_data->>'available_copies' as old_copies,
      new_data->>'available_copies' as new_copies,
      changed_at
FROM audit_log
WHERE table_name = 'books'
ORDER BY changed_at DESC;
```

#### Крок 6. Блокування записів

Продемонструвати використання блокувань для паралельного доступу.

```sql
-- Транзакція 1 (виконується в одному з'єднанні)
BEGIN;

-- Блокування запису для оновлення
SELECT * FROM books WHERE book_id = 1 FOR UPDATE;

-- Імітація довгої операції
SELECT pg_sleep(5);

UPDATE books SET available_copies = available_copies - 1 WHERE book_id = 1;

COMMIT;

-- Транзакція 2 (виконується в іншому з'єднанні одночасно)
-- Буде очікувати завершення транзакції 1
BEGIN;
SELECT * FROM books WHERE book_id = 1 FOR UPDATE;
UPDATE books SET available_copies = available_copies - 1 WHERE book_id = 1;
COMMIT;
```

### Рівень 3

#### Крок 1. Система версіонування

Реалізувати механізм збереження всіх версій записів.

```sql
-- Таблиця версій книг
CREATE TABLE books_versions (
    version_id SERIAL PRIMARY KEY,
    book_id INTEGER NOT NULL,
    version_number INTEGER NOT NULL,
    data JSONB NOT NULL,
    valid_from TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    valid_to TIMESTAMP,
    created_by VARCHAR(100) DEFAULT CURRENT_USER,
    UNIQUE(book_id, version_number)
);

-- Функція для створення нової версії
CREATE OR REPLACE FUNCTION create_book_version()
RETURNS TRIGGER AS $$
DECLARE
    v_version_number INTEGER;
BEGIN
    -- Закриваємо попередню версію
    UPDATE books_versions
    SET valid_to = CURRENT_TIMESTAMP
    WHERE book_id = NEW.book_id AND valid_to IS NULL;

    -- Визначаємо номер нової версії
    SELECT COALESCE(MAX(version_number), 0) + 1
    INTO v_version_number
    FROM books_versions
    WHERE book_id = NEW.book_id;

    -- Створюємо нову версію
    INSERT INTO books_versions (book_id, version_number, data)
    VALUES (NEW.book_id, v_version_number, row_to_json(NEW)::jsonb);

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER book_versioning_trigger
AFTER INSERT OR UPDATE ON books
FOR EACH ROW
EXECUTE FUNCTION create_book_version();

-- Функція для отримання версії на певну дату
CREATE OR REPLACE FUNCTION get_book_version(p_book_id INTEGER, p_date TIMESTAMP)
RETURNS JSONB AS $$
BEGIN
    RETURN (
        SELECT data
        FROM books_versions
        WHERE book_id = p_book_id
            AND valid_from <= p_date
            AND (valid_to IS NULL OR valid_to > p_date)
        LIMIT 1
    );
END;
$$ LANGUAGE plpgsql;
```

#### Крок 2. Відкладені операції

Створити механізм планування операцій на майбутнє.

```sql
-- Таблиця відкладених операцій
CREATE TABLE scheduled_operations (
    operation_id SERIAL PRIMARY KEY,
    operation_type VARCHAR(50) NOT NULL,
    target_table VARCHAR(50) NOT NULL,
    operation_data JSONB NOT NULL,
    scheduled_for TIMESTAMP NOT NULL,
    status VARCHAR(20) DEFAULT 'Pending' CHECK (status IN ('Pending', 'Executed', 'Failed', 'Cancelled')),
    executed_at TIMESTAMP,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Функція виконання відкладених операцій
CREATE OR REPLACE FUNCTION execute_scheduled_operations()
RETURNS INTEGER AS $$
DECLARE
    v_operation RECORD;
    v_count INTEGER := 0;
    v_sql TEXT;
BEGIN
    FOR v_operation IN
        SELECT * FROM scheduled_operations
        WHERE status = 'Pending' AND scheduled_for <= CURRENT_TIMESTAMP
    LOOP
        BEGIN
            -- Формуємо SQL на основі типу операції
            CASE v_operation.operation_type
                WHEN 'UPDATE_BOOK_STATUS' THEN
                    UPDATE books
                    SET available_copies = (v_operation.operation_data->>'available_copies')::INTEGER
                    WHERE book_id = (v_operation.operation_data->>'book_id')::INTEGER;

                WHEN 'EXPIRE_RESERVATION' THEN
                    UPDATE reservations
                    SET status = 'Expired'
                    WHERE reservation_id = (v_operation.operation_data->>'reservation_id')::INTEGER;

                WHEN 'BLOCK_READER' THEN
                    UPDATE readers
                    SET status = 'Blocked'
                    WHERE reader_id = (v_operation.operation_data->>'reader_id')::INTEGER;
            END CASE;

            -- Позначаємо операцію як виконану
            UPDATE scheduled_operations
            SET status = 'Executed', executed_at = CURRENT_TIMESTAMP
            WHERE operation_id = v_operation.operation_id;

            v_count := v_count + 1;
        EXCEPTION WHEN OTHERS THEN
            UPDATE scheduled_operations
            SET status = 'Failed',
                executed_at = CURRENT_TIMESTAMP,
                error_message = SQLERRM
            WHERE operation_id = v_operation.operation_id;
        END;
    END LOOP;

    RETURN v_count;
END;
$$ LANGUAGE plpgsql;

-- Додавання відкладеної операції
INSERT INTO scheduled_operations (operation_type, target_table, operation_data, scheduled_for)
VALUES (
    'EXPIRE_RESERVATION',
    'reservations',
    '{"reservation_id": 1}'::jsonb,
    CURRENT_TIMESTAMP + INTERVAL '1 hour'
);
```

#### Крок 3. Складна бізнес-логіка

Реалізувати збережені процедури для складних операцій.

```sql
-- Процедура автоматичного продовження терміну видачі
CREATE OR REPLACE FUNCTION extend_lending(
    p_lending_id INTEGER,
    p_days INTEGER DEFAULT 7
)
RETURNS TABLE (
    success BOOLEAN,
    message TEXT,
    new_due_date DATE
) AS $$
DECLARE
    v_reader_status VARCHAR(20);
    v_overdue_count INTEGER;
    v_current_due_date DATE;
BEGIN
    -- Перевірка статусу читача
    SELECT r.status, l.due_date
    INTO v_reader_status, v_current_due_date
    FROM lendings l
    JOIN readers r ON l.reader_id = r.reader_id
    WHERE l.lending_id = p_lending_id;

    IF v_reader_status = 'Blocked' THEN
        RETURN QUERY SELECT FALSE, 'Читач заблокований'::TEXT, NULL::DATE;
        RETURN;
    END IF;

    -- Перевірка кількості прострочених книг
    SELECT COUNT(*)
    INTO v_overdue_count
    FROM lendings l
    JOIN readers r ON l.reader_id = r.reader_id
    WHERE r.reader_id = (SELECT reader_id FROM lendings WHERE lending_id = p_lending_id)
        AND l.return_date IS NULL
        AND l.due_date < CURRENT_DATE;

    IF v_overdue_count > 0 THEN
        RETURN QUERY SELECT FALSE, 'У читача є прострочені книги'::TEXT, NULL::DATE;
        RETURN;
    END IF;

    -- Продовження терміну
    UPDATE lendings
    SET due_date = due_date + p_days
    WHERE lending_id = p_lending_id;

    RETURN QUERY SELECT TRUE, 'Термін продовжено'::TEXT, (v_current_due_date + p_days)::DATE;
END;
$$ LANGUAGE plpgsql;

-- Використання процедури
SELECT * FROM extend_lending(1, 14);
```

#### Крок 4. Система прав доступу

Створити ролі з різними правами на операції модифікації.

```sql
-- Створення ролей
CREATE ROLE librarian_read;
CREATE ROLE librarian_write;
CREATE ROLE library_admin;

-- Надання прав для перегляду
GRANT SELECT ON ALL TABLES IN SCHEMA public TO librarian_read;

-- Надання прав для модифікації
GRANT SELECT, INSERT, UPDATE ON books, lendings, reservations TO librarian_write;
GRANT DELETE ON reservations TO librarian_write;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO librarian_write;

-- Повні права для адміністратора
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO library_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO library_admin;

-- Створення користувачів
CREATE USER librarian1 WITH PASSWORD 'secure_password1';
CREATE USER librarian2 WITH PASSWORD 'secure_password2';
CREATE USER admin1 WITH PASSWORD 'admin_password';

-- Призначення ролей
GRANT librarian_read TO librarian1;
GRANT librarian_write TO librarian2;
GRANT library_admin TO admin1;

-- Тестування прав доступу
-- Від імені librarian1 (тільки читання)
SET ROLE librarian1;
SELECT * FROM books; -- Успішно
UPDATE books SET available_copies = 5 WHERE book_id = 1; -- Помилка

-- Від імені librarian2 (читання та запис)
SET ROLE librarian2;
UPDATE books SET available_copies = 5 WHERE book_id = 1; -- Успішно
DELETE FROM authors WHERE author_id = 1; -- Помилка

RESET ROLE;
```

#### Крок 5. Автоматичне відновлення

Реалізувати механізм point-in-time recovery.

```sql
-- Таблиця для зберігання контрольних точок
CREATE TABLE backup_points (
    backup_id SERIAL PRIMARY KEY,
    backup_name VARCHAR(100) NOT NULL,
    backup_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    description TEXT,
    data_snapshot JSONB
);

-- Функція створення контрольної точки
CREATE OR REPLACE FUNCTION create_backup_point(p_name VARCHAR, p_description TEXT DEFAULT NULL)
RETURNS INTEGER AS $$
DECLARE
    v_backup_id INTEGER;
    v_snapshot JSONB;
BEGIN
    -- Створюємо знімок критичних таблиць
    SELECT jsonb_build_object(
        'books', (SELECT jsonb_agg(row_to_json(b)) FROM books b),
        'authors', (SELECT jsonb_agg(row_to_json(a)) FROM authors a),
        'readers', (SELECT jsonb_agg(row_to_json(r)) FROM readers r),
        'lendings', (SELECT jsonb_agg(row_to_json(l)) FROM lendings l)
    ) INTO v_snapshot;

    INSERT INTO backup_points (backup_name, description, data_snapshot)
    VALUES (p_name, p_description, v_snapshot)
    RETURNING backup_id INTO v_backup_id;

    RETURN v_backup_id;
END;
$$ LANGUAGE plpgsql;

-- Функція відновлення з контрольної точки
CREATE OR REPLACE FUNCTION restore_from_backup(p_backup_id INTEGER)
RETURNS BOOLEAN AS $$
DECLARE
    v_snapshot JSONB;
    v_record JSONB;
BEGIN
    -- Отримуємо знімок
    SELECT data_snapshot INTO v_snapshot
    FROM backup_points
    WHERE backup_id = p_backup_id;

    IF v_snapshot IS NULL THEN
        RAISE EXCEPTION 'Backup point not found';
    END IF;

    -- Відновлюємо дані (спрощена версія для демонстрації)
    BEGIN
        -- Очищуємо існуючі дані
        TRUNCATE TABLE lendings CASCADE;
        TRUNCATE TABLE books CASCADE;
        TRUNCATE TABLE readers CASCADE;
        TRUNCATE TABLE authors CASCADE;

        -- Відновлюємо авторів
        FOR v_record IN SELECT * FROM jsonb_array_elements(v_snapshot->'authors')
        LOOP
            INSERT INTO authors
            SELECT * FROM jsonb_populate_record(NULL::authors, v_record);
        END LOOP;

        -- Відновлюємо інші таблиці аналогічно

        RETURN TRUE;
    EXCEPTION WHEN OTHERS THEN
        RAISE EXCEPTION 'Restore failed: %', SQLERRM;
        RETURN FALSE;
    END;
END;
$$ LANGUAGE plpgsql;

-- Створення контрольної точки
SELECT create_backup_point('Before major update', 'Backup before system upgrade');
```

#### Крок 6. Інтерфейс перегляду історії

Створити представлення для зручного перегляду історії змін.

```sql
-- Представлення для аналізу історії змін книг
CREATE VIEW books_change_history AS
SELECT
    a.log_id,
    a.operation,
    a.record_id as book_id,
    COALESCE(a.new_data->>'title', a.old_data->>'title') as title,
    CASE
        WHEN a.operation = 'INSERT' THEN 'Додано нову книгу'
        WHEN a.operation = 'UPDATE' THEN
            'Оновлено: ' ||
            CASE
                WHEN a.old_data->>'available_copies' <> a.new_data->>'available_copies'
                THEN 'Доступні примірники: ' || a.old_data->>'available_copies' || ' → ' || a.new_data->>'available_copies'
                ELSE 'Інші поля'
            END
        WHEN a.operation = 'DELETE' THEN 'Видалено книгу'
    END as change_description,
    a.changed_at,
    a.changed_by
FROM audit_log a
WHERE a.table_name = 'books'
ORDER BY a.changed_at DESC;

-- Функція отримання історії змін за період
CREATE OR REPLACE FUNCTION get_changes_report(
    p_start_date TIMESTAMP,
    p_end_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
RETURNS TABLE (
    change_date DATE,
    total_changes BIGINT,
    inserts BIGINT,
    updates BIGINT,
    deletes BIGINT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        DATE(changed_at) as change_date,
        COUNT(*) as total_changes,
        COUNT(*) FILTER (WHERE operation = 'INSERT') as inserts,
        COUNT(*) FILTER (WHERE operation = 'UPDATE') as updates,
        COUNT(*) FILTER (WHERE operation = 'DELETE') as deletes
    FROM audit_log
    WHERE changed_at BETWEEN p_start_date AND p_end_date
    GROUP BY DATE(changed_at)
    ORDER BY change_date DESC;
END;
$$ LANGUAGE plpgsql;

-- Використання
SELECT * FROM get_changes_report(CURRENT_DATE - INTERVAL '30 days');
```
### Підготовка звіту

1. Створити файл `lab03-report.md` ([📑 Завантажити шаблон](assets/lab03-report-template.md)). Додати опис виконаних запитів та результатів. Включити скріншоти.
2. Завантажити звіт зі скріншотами в репозиторій на GitHub.
3. Як відповідь на завдання в LMS Moodle вставити посилання на репозиторій.
4. Захистити лабораторну перед викладачем.

[🔼 Здати лабораторну роботу](https://moodle.vcolnuft.volyn.ua/moodle/course/view.php?id=32#section-2)

## ❓ Контрольні запитання

1. У чому полягає різниця між операторами INSERT, UPDATE та DELETE? Які обмеження необхідно враховувати при їх використанні?
2. Чому використання умови WHERE є критично важливим в операціях UPDATE та DELETE? Які наслідки можуть бути від її пропуску?
3. Що таке транзакція та які властивості ACID вона забезпечує? Поясніть кожну властивість на конкретному прикладі.
4. Яка різниця між командами COMMIT та ROLLBACK? У яких ситуаціях використовується кожна з них?
5. Що таке точки збереження SAVEPOINT і як вони дозволяють керувати транзакціями більш гнучко порівняно з простим ROLLBACK?
6. Які типи обмежень цілісності ви знаєте? Як кожен з них допомагає підтримувати коректність даних у базі?
7. Що таке каскадні операції та які варіанти їх поведінки існують при видаленні або оновленні батьківських записів?
8. Яку роль відіграють тригери в забезпеченні цілісності даних та автоматизації операцій? Наведіть приклади їх практичного застосування.
9. Чому необхідно використовувати блокування записів при паралельному доступі? Які проблеми можуть виникнути без них?
10. Як рівні ізоляції транзакцій впливають на можливі аномалії при паралельному виконанні операцій? Який рівень слід обирати для різних сценаріїв?
