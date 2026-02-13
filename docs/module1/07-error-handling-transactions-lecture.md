# 7. (Л) Обробка помилок і транзакцій у Python при роботі з PostgreSQL

## Зміст лекції

1. Ієрархія винятків psycopg2
2. Обробка помилок підключення
3. Обробка помилок SQL-запитів
4. Транзакції в PostgreSQL
5. Управління транзакціями через psycopg2
6. SAVEPOINT — вкладені транзакції
7. Контекстні менеджери та транзакції
8. Режим autocommit
9. Практичний приклад: безпечне переведення коштів

### Підготовка бази даних для прикладів

Перед тим як працювати з прикладами цієї лекції, створіть усі потрібні таблиці та заповніть їх початковими даними:

```python
import psycopg2

def populate(conn):
    """Створення таблиць і заповнення початковими даними для всіх прикладів лекції"""
    with conn.cursor() as cursor:
        # Таблиця користувачів (приклади обробки помилок)
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                email VARCHAR(255) UNIQUE
            )
        ''')

        # Таблиця рахунків (приклади транзакцій)
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS accounts (
                id SERIAL PRIMARY KEY,
                owner VARCHAR(100) NOT NULL,
                balance NUMERIC(12, 2) NOT NULL DEFAULT 0,
                CHECK (balance >= 0)
            )
        ''')

        # Таблиця замовлень (приклад SAVEPOINT)
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS orders (
                id SERIAL PRIMARY KEY,
                user_id INTEGER REFERENCES users(id),
                total NUMERIC(12, 2) NOT NULL
            )
        ''')

        # Таблиця бонусів (приклад SAVEPOINT)
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS bonuses (
                id SERIAL PRIMARY KEY,
                user_id INTEGER REFERENCES users(id),
                amount NUMERIC(12, 2) NOT NULL
            )
        ''')

        # Таблиця логів (приклади autocommit та контекстних менеджерів)
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS logs (
                id SERIAL PRIMARY KEY,
                message TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        # Заповнення початковими даними (тільки якщо таблиці порожні)
        cursor.execute("SELECT COUNT(*) FROM users")
        if cursor.fetchone()[0] == 0:
            cursor.execute(
                "INSERT INTO users (name, email) VALUES (%s, %s), (%s, %s), (%s, %s)",
                ("Alice", "alice@example.com", "Bob", "bob@example.com", "Charlie", "charlie@example.com")
            )

        cursor.execute("SELECT COUNT(*) FROM accounts")
        if cursor.fetchone()[0] == 0:
            cursor.execute(
                "INSERT INTO accounts (owner, balance) VALUES (%s, %s), (%s, %s)",
                ("Олена", 1000, "Тарас", 500)
            )

    conn.commit()
    print("База даних готова до роботи!")

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
populate(conn)
```

### Ієрархія винятків psycopg2

Бібліотека psycopg2 має чітку ієрархію винятків, яка відповідає специфікації Python DB-API 2.0 ([PEP 249](https://peps.python.org/pep-0249/)):

```
Exception
└── psycopg2.Error
    ├── psycopg2.DatabaseError
    │   ├── psycopg2.DataError            # Невірні дані (наприклад, ділення на 0)
    │   ├── psycopg2.OperationalError     # Проблеми з БД (з'єднання, пам'ять)
    │   ├── psycopg2.IntegrityError       # Порушення обмежень (UNIQUE, FK, NOT NULL)
    │   ├── psycopg2.InternalError        # Внутрішня помилка БД
    │   ├── psycopg2.ProgrammingError     # Помилки SQL-синтаксису, таблиця не існує
    │   └── psycopg2.NotSupportedError    # Непідтримувана операція
    ├── psycopg2.InterfaceError           # Проблеми інтерфейсу (курсор закрито)
    └── psycopg2.Warning                  # Попередження
```

Кожен виняток має корисні атрибути:

```python
import psycopg2

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
cursor = conn.cursor()

try:
    cursor.execute("INSERT INTO users (email) VALUES (%s)", ("duplicate@example.com",))
    conn.commit()
except psycopg2.Error as e:
    print(f"Код помилки (pgcode): {e.pgcode}")    # наприклад, '23505' для UNIQUE violation
    print(f"Повідомлення: {e.pgerror}")            # детальний опис від PostgreSQL
    print(f"Діагностика: {e.diag.message_detail}") # додаткові деталі
    conn.rollback()
```

### Обробка помилок підключення

Помилки підключення — найпоширеніший тип помилок. Вони виникають, коли сервер PostgreSQL недоступний, вказано невірні облікові дані або база даних не існує.

```python
import psycopg2

try:
    conn = psycopg2.connect(
        host="localhost",
        port=5432,
        database="mydb",
        user="postgres",
        password="secret"
    )
    print("Підключення успішне!")
except psycopg2.OperationalError as e:
    print(f"Не вдалося підключитися до бази даних: {e}")
    # Типові причини:
    # - Сервер не запущено
    # - Невірний хост або порт
    # - Невірний пароль
    # - База даних не існує
```

### Обробка помилок SQL-запитів

#### ProgrammingError — помилки синтаксису та структури

```python
import psycopg2

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
cursor = conn.cursor()

try:
    cursor.execute("SELET * FROM users")  # Помилка: SELET замість SELECT
    conn.commit()
except psycopg2.ProgrammingError as e:
    print(f"Помилка SQL: {e}")
    conn.rollback()  # Обов'язково! Після помилки транзакція "зламана"
```

#### IntegrityError — порушення обмежень

```python
import psycopg2

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
cursor = conn.cursor()

try:
    cursor.execute(
        "INSERT INTO users (name, email) VALUES (%s, %s)",
        ("Alice", "alice@example.com")
    )
    conn.commit()
except psycopg2.IntegrityError as e:
    if e.pgcode == '23505':  # unique_violation
        print("Користувач з таким email вже існує!")
    elif e.pgcode == '23502':  # not_null_violation
        print("Обов'язкове поле не заповнено!")
    elif e.pgcode == '23503':  # foreign_key_violation
        print("Порушення зовнішнього ключа!")
    conn.rollback()
```

**Важливо:** після будь-якої помилки в psycopg2 поточна транзакція переходить у стан `ABORTED` (зламана). Перш ніж виконувати нові запити, **обов'язково** потрібно викликати `conn.rollback()`. Інакше всі наступні запити будуть повертати помилку:

```
psycopg2.errors.InFailedSqlTransaction: current transaction is aborted,
commands ignored until end of transaction block
```

### Транзакції в PostgreSQL

**Транзакція** — це група SQL-операцій, які виконуються як єдине ціле. Або всі операції виконуються успішно, або жодна з них не застосовується.

**Властивості ACID:**

| Властивість | Опис |
|---|---|
| **Atomicity** (Атомарність) | Всі операції виконуються або не виконується жодна |
| **Consistency** (Узгодженість) | БД переходить з одного коректного стану в інший |
| **Isolation** (Ізоляція) | Паралельні транзакції не впливають одна на одну |
| **Durability** (Довговічність) | Після commit дані збережені назавжди |

**Приклад:** переказ грошей з одного рахунку на інший. Потрібно виконати дві операції: зняти з одного рахунку та додати на інший. Якщо між ними стається збій — гроші не повинні зникнути.

```sql
-- На рівні SQL транзакції виглядають так:
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Якщо щось пішло не так:
ROLLBACK;
```

### Управління транзакціями через psycopg2

За замовчуванням psycopg2 **автоматично починає** нову транзакцію перед першим SQL-запитом. Транзакція залишається відкритою, поки ви не викличете `commit()` або `rollback()`.

```python
import psycopg2

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
cursor = conn.cursor()

# Транзакція починається автоматично з першим запитом
cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = %s", (1,))
cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = %s", (2,))

# Підтвердити транзакцію — обидві зміни збережуться
conn.commit()
print("Зміни успішно збережені")

# АБО відкатити — жодна зміна не збережеться
# conn.rollback()
```

**Шаблон try/except/finally для безпечної роботи з транзакціями:**

```python
import psycopg2

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
cursor = conn.cursor()

try:
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = %s", (1,))
    cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = %s", (2,))
    conn.commit()
    print("Транзакція успішна")
except psycopg2.Error as e:
    conn.rollback()
    print(f"Помилка, транзакцію відкачено: {e}")
finally:
    cursor.close()
    conn.close()
```

### SAVEPOINT — вкладені транзакції

SAVEPOINT дозволяє створити «точку збереження» всередині транзакції. Якщо частина операцій не вдалася, можна відкатити лише до цієї точки, а не всю транзакцію.
```python
import psycopg2

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
cursor = conn.cursor()

try:
    cursor.execute("INSERT INTO orders (user_id, total) VALUES (%s, %s)", (1, 500))

    # Створюємо savepoint
    cursor.execute("SAVEPOINT sp_bonus")

    try:
        # Спроба нарахувати бонус (може не вдатись)
        cursor.execute("INSERT INTO bonuses (user_id, amount) VALUES (%s, %s)", (1, 50))
    except psycopg2.Error:
        # Відкатуємо тільки до savepoint — замовлення залишається
        cursor.execute("ROLLBACK TO SAVEPOINT sp_bonus")
        print("Бонус не нараховано, але замовлення збережено")

    conn.commit()
except psycopg2.Error as e:
    conn.rollback()
    print(f"Вся транзакція відкачена: {e}")
finally:
    cursor.close()
    conn.close()
```

### Контекстні менеджери та транзакції

Контекстний менеджер `with` для з'єднання psycopg2 автоматично викликає `commit()` при успішному виході або `rollback()` при винятку:

```python
import psycopg2

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")

try:
    with conn:
        # Транзакція починається автоматично
        with conn.cursor() as cursor:
            cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = %s", (1,))
            cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = %s", (2,))
        # commit() викличеться автоматично при виході з блоку with

    with conn:
        # Нова транзакція
        with conn.cursor() as cursor:
            cursor.execute("INSERT INTO logs (message) VALUES (%s)", ("Переказ виконано",))
        # commit() автоматично
finally:
    conn.close()  # Контекстний менеджер НЕ закриває з'єднання!
```

**Важливо:** `with conn` **не закриває** з'єднання — він лише управляє транзакцією (commit/rollback). З'єднання потрібно закрити самостійно.

### Режим autocommit

У режимі autocommit кожен SQL-запит виконується в окремій транзакції та автоматично підтверджується:

```python
import psycopg2

conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
conn.autocommit = True  # Увімкнути autocommit

cursor = conn.cursor()

# Кожен запит — це окрема транзакція
cursor.execute("INSERT INTO logs (message) VALUES (%s)", ("Подія 1",))  # Відразу збережено
cursor.execute("INSERT INTO logs (message) VALUES (%s)", ("Подія 2",))  # Відразу збережено

cursor.close()
conn.close()
```

**Коли використовувати autocommit:**

- Для операцій, які не можуть виконуватись всередині транзакції (наприклад, `CREATE DATABASE`, `VACUUM`)
- Для простих одноразових запитів, де атомарність не потрібна
- Для читання даних (SELECT), де транзакції не критичні

**Коли НЕ використовувати autocommit:**

- Коли потрібно виконати кілька пов'язаних операцій атомарно
- Коли важлива цілісність даних між кількома запитами

### Практичний приклад: безпечне переведення коштів

```python
import psycopg2

def create_tables(conn):
    """Створення таблиці рахунків"""
    with conn.cursor() as cursor:
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS accounts (
                id SERIAL PRIMARY KEY,
                owner VARCHAR(100) NOT NULL,
                balance NUMERIC(12, 2) NOT NULL DEFAULT 0,
                CHECK (balance >= 0)
            )
        ''')
    conn.commit()

def transfer_money(conn, from_id, to_id, amount):
    """
    Безпечний переказ коштів між рахунками.
    Використовує транзакцію — або переказ виконується повністю, або не виконується зовсім.
    """
    try:
        with conn:  # commit() автоматично при успіху, rollback() при помилці
            with conn.cursor() as cursor:
                # Перевірка балансу відправника
                cursor.execute("SELECT balance FROM accounts WHERE id = %s", (from_id,))
                row = cursor.fetchone()
                if row is None:
                    raise ValueError(f"Рахунок {from_id} не знайдено")

                balance = row[0]
                if balance < amount:
                    raise ValueError(f"Недостатньо коштів: {balance} < {amount}")

                # Зняття з рахунку відправника
                cursor.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, from_id)
                )

                # Зарахування на рахунок отримувача
                cursor.execute(
                    "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to_id)
                )

                print(f"Переказ {amount} грн з рахунку {from_id} на рахунок {to_id} успішний!")
    except psycopg2.IntegrityError:
        print("Помилка: порушення обмежень (можливо, від'ємний баланс)")
    except ValueError as e:
        print(f"Помилка: {e}")


# Використання
conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
try:
    create_tables(conn)
    transfer_money(conn, from_id=1, to_id=2, amount=100)
finally:
    conn.close()
```

## Корисні посилання

- [Документація psycopg2: Transactions](https://www.psycopg.org/docs/usage.html#transactions-control)
- [Документація psycopg2: Exceptions](https://www.psycopg.org/docs/errors.html)
- [PostgreSQL: Error Codes](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [PostgreSQL: Transaction Management](https://www.postgresql.org/docs/current/tutorial-transactions.html)

## Домашнє завдання

Прочитати документацію [psycopg2: Transactions control](https://www.psycopg.org/docs/usage.html#transactions-control)
