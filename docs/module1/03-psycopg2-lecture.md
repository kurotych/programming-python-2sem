# 3. (Л) Python та PostgreSQL. Робота з бібліотекою psycopg2

## Зміст лекції

1. Встановлення бібліотеки psycopg2
2. Підключення до бази даних
3. Що таке Cursor?
4. Створення таблиці
5. Коли потрібен commit()?
6. Основні операції CRUD (додавання, отримання даних)
7. Закриття з'єднання
8. Відмінності від sqlite3

### Встановлення

На відміну від sqlite3, бібліотека `psycopg2` не входить до стандартної бібліотеки Python і потребує встановлення.

Перед встановленням створіть та активуйте віртуальне середовище:

```bash
python3 -m venv env
source env/bin/activate
pip install psycopg2-binary
```

**Примітка:** `psycopg2-binary` - це попередньо скомпільована версія, яка простіша в установці. Для production-середовища рекомендується використовувати `psycopg2` (потребує компіляції та наявності PostgreSQL development headers).

```python
import psycopg2
```

### Розгортання PostgreSQL

Перед підключенням потрібно мати запущений сервер PostgreSQL. Найпростіший спосіб — використати Docker:

```bash
docker run -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=mydb -p 5432:5432 -d postgres
```

Ця команда:

- Створює контейнер з іменем `my-postgres`
- Встановлює пароль `secret` для користувача `postgres`
- Створює базу даних `mydb`
- Прокидає порт 5432 на localhost

### Підключення до бази даних

```python
import psycopg2

# Підключення до бази даних PostgreSQL
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    database="mydb",
    user="postgres",
    password="secret"
)
cursor = conn.cursor()
```

**Важливо:**

- На відміну від SQLite, PostgreSQL є клієнт-серверною СУБД, тому потрібно вказати параметри підключення
- `host` - адреса сервера (localhost для локального сервера)
- `port` - порт PostgreSQL (за замовчуванням 5432)
- `database` - назва бази даних (повинна існувати заздалегідь)
- `user` та `password` - облікові дані для автентифікації
- База даних повинна бути створена заздалегідь (на відміну від SQLite, який створює файл автоматично)

```python
# Альтернативний спосіб - через connection string (DSN)
conn = psycopg2.connect("host=localhost port=5432 dbname=mydb user=postgres password=secret")

# Або у форматі URI
conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
```

### Що таке Cursor?

**Cursor (курсор)** працює аналогічно до sqlite3 - це об'єкт для виконання SQL-запитів та отримання результатів.

Основні можливості курсора:

- Виконання SQL-команд через метод `execute()`
- Отримання результатів: `fetchall()`, `fetchone()`, `fetchmany()`
- Можна створити кілька курсорів для одного з'єднання
- Курсор зберігає контекст виконання та результати останнього запиту

```python
# Приклад створення кількох курсорів
cursor1 = conn.cursor()
cursor2 = conn.cursor()
```

### Створення таблиці

```python
cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        email VARCHAR(255) UNIQUE
    )
''')
conn.commit()
```

**Відмінності від SQLite:**

- `SERIAL` (або `BIGSERIAL`) замість `INTEGER PRIMARY KEY AUTOINCREMENT` - автоматично генерує унікальні значення
- `VARCHAR(n)` замість `TEXT` - рядок обмеженої довжини (хоча `TEXT` теж підтримується)
- PostgreSQL має строгішу типізацію даних

### Коли потрібен commit()?

Правила такі ж, як і для sqlite3. `commit()` потрібен для операцій, які **змінюють** дані:

**Потребують commit():**

- `INSERT` - додавання даних
- `UPDATE` - оновлення даних
- `DELETE` - видалення даних
- `CREATE TABLE` - створення таблиць
- `DROP TABLE` - видалення таблиць
- `ALTER TABLE` - зміна структури таблиць

**НЕ потребують commit():**

- `SELECT` - читання даних

```python
# Приклад: SELECT не потребує commit
cursor.execute("SELECT * FROM users")
rows = cursor.fetchall()  # commit() не потрібен!

# Приклад: INSERT потребує commit
cursor.execute("INSERT INTO users (name, email) VALUES (%s, %s)",
               ("Alice", "alice@example.com"))
conn.commit()  # Обов'язково!
```

**Автоматичний commit:**

```python
# Увімкнення autocommit режиму
conn.autocommit = True

# Або через контекстний менеджер
with psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb") as conn:
    with conn.cursor() as cursor:
        cursor.execute("INSERT INTO users (name, email) VALUES (%s, %s)",
                       ("Bob", "bob@example.com"))
        # commit() відбудеться автоматично при виході з блоку with
```

### Додавання даних

```python
# Один запис
cursor.execute("INSERT INTO users (name, email) VALUES (%s, %s)",
               ("John Doe", "john@example.com"))
conn.commit()

# Декілька записів одночасно
users_data = [
    ("Alice", "alice@example.com"),
    ("Bob", "bob@example.com"),
    ("Charlie", "charlie@example.com")
]
cursor.executemany("INSERT INTO users (name, email) VALUES (%s, %s)", users_data)
conn.commit()
```

**Важлива відмінність:** psycopg2 використовує `%s` як плейсхолдер для параметрів, тоді як sqlite3 використовує `?`.

### Отримання даних

```python
cursor.execute("SELECT * FROM users")
rows = cursor.fetchall()
for row in rows:
    print(row)

# Отримання одного запису
cursor.execute("SELECT * FROM users WHERE id = %s", (1,))
user = cursor.fetchone()
print(user)

# Отримання певної кількості записів
cursor.execute("SELECT * FROM users")
first_five = cursor.fetchmany(5)
```

### Закриття з'єднання

```python
cursor.close()  # Закриття курсора
conn.close()    # Закриття з'єднання
```

**Чому потрібно закривати з'єднання?**

- PostgreSQL має обмежену кількість одночасних з'єднань (за замовчуванням ~100)
- Кожне з'єднання споживає пам'ять на сервері
- Незакриті з'єднання можуть блокувати ресурси бази даних
- У production використовують connection pooling для ефективного управління з'єднаннями

**Рекомендована практика з контекстними менеджерами:**

```python
# Найкращий підхід - автоматичне управління ресурсами
with psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb") as conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM users")
        rows = cursor.fetchall()
        # cursor автоматично закриється тут
    # conn автоматично закриється тут
```

**Без контекстного менеджера:**

```python
# Ручне управління - потрібно пам'ятати про закриття
conn = psycopg2.connect("postgresql://postgres:secret@localhost:5432/mydb")
cursor = conn.cursor()
try:
    cursor.execute("SELECT * FROM users")
    rows = cursor.fetchall()
finally:
    cursor.close()
    conn.close()
```

## Корисні посилання

- [Документація psycopg2](https://www.psycopg.org/docs/)
- [PostgreSQL Tutorial](https://www.postgresqltutorial.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

## Домашнє завдання

Прочитати документацію [psycopg2](https://www.psycopg.org/docs/)
