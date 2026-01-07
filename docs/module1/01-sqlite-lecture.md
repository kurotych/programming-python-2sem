# 1. (Л) Python та SQLite. Робота з бібліотекою sqlite3

## Зміст лекції

1. Встановлення бібліотеки sqlite3
2. Підключення до бази даних
3. Що таке Cursor?
4. Створення таблиці
5. Коли потрібен commit()?
6. Основні операції CRUD (додавання, отримання даних)
7. Закриття з'єднання

### Встановлення

Бібліотека `sqlite3` є частиною стандартної бібліотеки Python, тому не потрібно встановлювати жодних додаткових пакетів. Просто імпортуйте її:

```python
import sqlite3
```

### Підключення до бази даних

```python
# Створення/підключення до бази даних
conn = sqlite3.connect('example.db')
cursor = conn.cursor()
```

**Важливо:**

- `'example.db'` - це шлях до файлу бази даних. Якщо вказано лише ім'я файлу (без шляху), файл створюється в **поточній робочій директорії** (де запускається Python-скрипт)
- Можна вказати повний або відносний шлях: `'data/example.db'` або `'/home/user/databases/example.db'`
- Якщо файл бази даних не існує, SQLite автоматично створить його
- Можна використовувати спеціальне ім'я `:memory:` для створення бази даних у пам'яті (RAM, зникає після закриття з'єднання)
- `conn` - об'єкт з'єднання (connection), який представляє підключення до бази даних

```python
# Приклади різних шляхів
conn1 = sqlite3.connect('example.db')              # Поточна директорія
conn2 = sqlite3.connect('databases/example.db')    # Відносний шлях
conn3 = sqlite3.connect('/tmp/example.db')         # Абсолютний шлях
conn4 = sqlite3.connect(':memory:')                # База даних у пам'яті
```

### Що таке Cursor?

**Cursor (курсор)** - це об'єкт, який використовується для виконання SQL-запитів та отримання результатів. Курсор створюється з об'єкта з'єднання (connection) за допомогою методу `cursor()`.

Основні можливості курсора:

- Дозволяє виконувати SQL-команди через метод `execute()`
- Використовується для отримання результатів запитів (`fetchall()`, `fetchone()`, `fetchmany()`)
- Можна створити кілька курсорів для одного з'єднання
- Курсор зберігає контекст виконання: результати останнього запиту, поточну позицію читання даних, опис колонок (`cursor.description`)
- Це дозволяє незалежно працювати з різними запитами через різні курсори

```python
# Приклад створення кількох курсорів
cursor1 = conn.cursor()
cursor2 = conn.cursor()
```

### Створення таблиці

```python
cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE
    )
''')
conn.commit()
```

### Коли потрібен commit()?

**Важливе правило:** `commit()` потрібен тільки для операцій, які **змінюють** дані:

**Потребують commit():**

- `INSERT` - додавання даних
- `UPDATE` - оновлення даних
- `DELETE` - видалення даних
- `CREATE TABLE` - створення таблиць
- `DROP TABLE` - видалення таблиць
- `ALTER TABLE` - зміна структури таблиць

**НЕ потребують commit():**

- `SELECT` - читання даних (запити на вибірку)

```python
# Приклад: SELECT не потребує commit
cursor.execute("SELECT * FROM users")
rows = cursor.fetchall()  # commit() не потрібен!

# Приклад: INSERT потребує commit
cursor.execute("INSERT INTO users (name, email) VALUES (?, ?)",
               ("Alice", "alice@example.com"))
conn.commit()  # Обов'язково!
```

**Автоматичний commit:**

```python
# Можна використовувати контекстний менеджер для автоматичного commit
with sqlite3.connect('example.db') as conn:
    cursor = conn.cursor()
    cursor.execute("INSERT INTO users (name, email) VALUES (?, ?)",
                   ("Bob", "bob@example.com"))
    # commit() відбудеться автоматично при виході з блоку with
```

### Додавання даних

```python
cursor.execute("INSERT INTO users (name, email) VALUES (?, ?)",
               ("John Doe", "john@example.com"))
conn.commit()
```

### Отримання даних

```python
cursor.execute("SELECT * FROM users")
rows = cursor.fetchall()
for row in rows:
    print(row)
```

### Закриття з'єднання

```python
cursor.close()  # Закриття курсора
conn.close()    # Закриття з'єднання
```

**Чому потрібно закривати курсор?**

- Звільняє пам'ять та ресурси, зайняті курсором
- Курсор зберігає результати запитів, які можуть займати багато пам'яті
- Хоча курсор автоматично закриється при закритті з'єднання, явне закриття є хорошою практикою
- Особливо важливо при роботі з багатьма курсорами або великими наборами даних

**Чому потрібно закривати з'єднання?**

- **Запис незбережених змін:** При закритті з'єднання без `commit()` всі незбережені зміни будуть втрачені
- **Звільнення блокувань:** SQLite блокує файл бази даних під час з'єднання. Без закриття інші процеси можуть не мати доступу до БД
- **Звільнення системних ресурсів:** Відкрите з'єднання займає файловий дескриптор та пам'ять
- **Запобігання витоку ресурсів:** У довготривалих програмах незакриті з'єднання можуть накопичуватися і вичерпати системні ресурси
- **Гарантія цілісності даних:** SQLite записує зміни на диск при закритті з'єднання

**Рекомендована практика з контекстними менеджерами:**

```python
# Найкращий підхід - автоматичне управління ресурсами
with sqlite3.connect('example.db') as conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM users")
        rows = cursor.fetchall()
        # cursor автоматично закриється тут
    # conn автоматично закриється тут (з commit, якщо були зміни)
```

**Без контекстного менеджера:**

```python
# Ручне управління - потрібно пам'ятати про закриття
conn = sqlite3.connect('example.db')
cursor = conn.cursor()
try:
    cursor.execute("SELECT * FROM users")
    rows = cursor.fetchall()
finally:
    cursor.close()  # Обов'язково закрити
    conn.close()    # Обов'язково закрити
```

## Корисні посилання

- [Документація sqlite3](https://docs.python.org/3/library/sqlite3.html)
- [SQLite Tutorial](https://www.sqlitetutorial.net/)

## Домашнє завдання

Прочитати документацію [sqlite3](https://docs.python.org/3/library/sqlite3.html)
