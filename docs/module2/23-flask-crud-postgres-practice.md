# 23. (П11) Реалізація CRUD операцій і робота з базою даних у Flask

## Передумови

- Прочитана [Лекція 22 — CRUD операції у Flask REST API з використанням PostgreSQL](/ua/courses/programming-2sem/module2/22-flask-crud-postgres-lecture/)
- Виконана [Практична 10 — Завдання з передачі даних в HTTP-запиті](/ua/courses/programming-2sem/module2/21-flask-request-data-practice/)
- Встановлений Python 3.10+
- Встановлений Docker

!!! warning "Персоналізація"
    У всіх запитах, де передається поле `created_by`, повинно бути **ваше ім'я та прізвище**. На скріншотах у звіті повинно бути видно ваше ім'я.

## Завдання

Побудувати REST API для управління бібліотекою книг (**Book Library API**) з використанням Flask та PostgreSQL. На відміну від попередніх практичних, дані зберігаються не в пам'яті, а в базі даних. API підтримує повний набір CRUD-операцій для книг і авторів, фільтрацію та пошук.

### Підготовка середовища

#### PostgreSQL у Docker

```bash
docker run -d \
  --name library-postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=library_db \
  -p 5432:5432 \
  postgres:17
```

#### Проєкт

```bash
python3 -m venv env
source env/bin/activate
pip install flask psycopg2-binary
```

### Структура бази даних

Застосунок працює з двома таблицями:

**Таблиця `authors`:**

| Поле | Опис |
|---|---|
| `id` | Унікальний ідентифікатор |
| `name` | Ім'я автора (обов'язкове) |
| `birth_year` | Рік народження |

**Таблиця `books`:**

| Поле | Опис |
|---|---|
| `id` | Унікальний ідентифікатор |
| `title` | Назва книги (обов'язкове) |
| `genre` | Жанр |
| `year_published` | Рік публікації |
| `author_id` | Зовнішній ключ на автора |
| `created_by` | Хто створив запис (обов'язкове) |

!!! note "Зовнішній ключ"
    Поле `author_id` у таблиці `books` посилається на `authors(id)`. Конструкція `ON DELETE SET NULL` означає: якщо автора видалити, у книгах поле `author_id` стане `NULL`, а самі книги залишаться.

### Частина 1: CRUD для авторів

| Метод | URL | Опис | Коди відповідей |
|---|---|---|---|
| POST | `/api/authors` | Створити автора | 201, 400 |
| GET | `/api/authors` | Список усіх авторів | 200 |
| GET | `/api/authors/<id>` | Отримати автора за ID | 200, 404 |
| DELETE | `/api/authors/<id>` | Видалити автора | 204, 404 |

**Вимоги:**

- **POST `/api/authors`** — обов'язкове поле: `name`. Необов'язкове поле: `birth_year`. Повертає створеного автора з кодом `201`.

- **GET `/api/authors`** — повертає список усіх авторів, відсортованих за `id`.

- **GET `/api/authors/<id>`** — повертає автора за ID або `404` з JSON-повідомленням про помилку.

- **DELETE `/api/authors/<id>`** — видаляє автора. Повертає порожнє тіло з кодом `204` або `404`.

**Формат відповіді:**

```json
{
    "id": 1,
    "name": "Тарас Шевченко",
    "birth_year": 1814
}
```

**Валідація при створенні:**

- Тіло запиту має бути JSON
- `name` — обов'язкове, не порожній рядок

```json
{"error": "field 'name' is required"}
```

### Частина 2: CRUD для книг

| Метод | URL | Опис | Коди відповідей |
|---|---|---|---|
| POST | `/api/books` | Створити книгу | 201, 400 |
| GET | `/api/books` | Список книг з фільтрацією | 200 |
| GET | `/api/books/<id>` | Отримати книгу за ID | 200, 404 |
| PUT | `/api/books/<id>` | Оновити книгу | 200, 400, 404 |
| DELETE | `/api/books/<id>` | Видалити книгу | 204, 404 |

**Вимоги:**

- **POST `/api/books`** — обов'язкові поля: `title`, `created_by`. Необов'язкові поля: `genre`, `year_published`, `author_id`. Якщо передано `author_id`, перевірити, що автор з таким ID існує (інакше `400`). Повертає створену книгу з кодом `201`.

- **GET `/api/books`** — повертає список книг, відсортованих за `id`. Підтримує фільтрацію через query-параметри:
    - `genre` — фільтрація за жанром (точне співпадіння)
    - `author_id` — фільтрація за автором
    - `q` — пошук за ключовим словом у `title` (регістронезалежний, часткове входження через `ILIKE`)

    Приклад: `GET /api/books?genre=poetry&q=kobzar`

- **GET `/api/books/<id>`** — повертає книгу за ID або `404`.

- **PUT `/api/books/<id>`** — оновлює лише ті поля, які передані в тілі запиту (використовуйте `COALESCE`, як у [лекції 22](/ua/courses/programming-2sem/module2/22-flask-crud-postgres-lecture/)). Повертає оновлену книгу або `404`.

- **DELETE `/api/books/<id>`** — видаляє книгу. Повертає порожнє тіло з кодом `204` або `404`.

**Формат відповіді:**

```json
{
    "id": 1,
    "title": "Kobzar",
    "genre": "poetry",
    "year_published": 1840,
    "author_id": 1,
    "created_by": "Ім'я Прізвище"
}
```

**Валідація при створенні:**

- Тіло запиту має бути JSON
- `title` — обов'язкове, не порожній рядок
- `created_by` — обов'язкове, не порожній рядок
- `author_id` (якщо передано) — автор повинен існувати в базі

```json
{"error": "field 'title' is required"}
```

```json
{"error": "Author with id 999 not found"}
```

### Частина 3: Отримання книг автора

| Метод | URL | Опис | Коди відповідей |
|---|---|---|---|
| GET | `/api/authors/<id>/books` | Книги конкретного автора | 200, 404 |

**Вимоги:**

- Перевірити, що автор з таким ID існує (інакше `404`)
- Повертає список книг, у яких `author_id` відповідає ID автора

```bash
curl http://127.0.0.1:5000/api/authors/1/books
```

### Частина 4: Статистика

| Метод | URL | Опис | Коди відповідей |
|---|---|---|---|
| GET | `/api/stats` | Статистика бібліотеки | 200 |

**Вимоги:**

Повертає загальну статистику бібліотеки, зібрану за допомогою SQL-запитів:

```json
{
    "total_books": 5,
    "total_authors": 3,
    "books_by_genre": {
        "poetry": 2,
        "novel": 3
    },
    "oldest_book_year": 1840,
    "newest_book_year": 2020
}
```

### Тестування через curl

За допомогою curl команд переконатися, що:

**Автори:**

```bash
# Створити автора
curl -X POST http://127.0.0.1:5000/api/authors \
  -H "Content-Type: application/json" \
  -d '{"name": "Тарас Шевченко", "birth_year": 1814}'

# Створити другого автора
curl -X POST http://127.0.0.1:5000/api/authors \
  -H "Content-Type: application/json" \
  -d '{"name": "Леся Українка", "birth_year": 1871}'

# Список авторів
curl http://127.0.0.1:5000/api/authors

# Автор за ID
curl http://127.0.0.1:5000/api/authors/1

# Неіснуючий автор
curl http://127.0.0.1:5000/api/authors/999
```

**Книги:**

```bash
# Створити книгу
curl -X POST http://127.0.0.1:5000/api/books \
  -H "Content-Type: application/json" \
  -d '{"title": "Kobzar", "genre": "poetry", "year_published": 1840, "author_id": 1, "created_by": "Ім'я Прізвище"}'

# Створити книгу без автора
curl -X POST http://127.0.0.1:5000/api/books \
  -H "Content-Type: application/json" \
  -d '{"title": "Unknown Author Book", "genre": "novel", "created_by": "Ім'я Прізвище"}'

# Спроба створити книгу з неіснуючим автором
curl -X POST http://127.0.0.1:5000/api/books \
  -H "Content-Type: application/json" \
  -d '{"title": "Test", "author_id": 999, "created_by": "Ім'я Прізвище"}'

# Список книг з фільтрацією
curl "http://127.0.0.1:5000/api/books?genre=poetry"
curl "http://127.0.0.1:5000/api/books?q=kobzar"
curl "http://127.0.0.1:5000/api/books?author_id=1"

# Оновити книгу
curl -X PUT http://127.0.0.1:5000/api/books/1 \
  -H "Content-Type: application/json" \
  -d '{"genre": "classic poetry"}'

# Видалити книгу
curl -X DELETE http://127.0.0.1:5000/api/books/1
```

**Зв'язки та статистика:**

```bash
# Книги автора
curl http://127.0.0.1:5000/api/authors/1/books

# Статистика
curl http://127.0.0.1:5000/api/stats

# Видалити автора та перевірити, що книги залишились (author_id = null)
curl -X DELETE http://127.0.0.1:5000/api/authors/1
curl http://127.0.0.1:5000/api/books
```

## Результат

Після виконання роботи студент здає:

- **`app.py`** — Flask-застосунок з усіма маршрутами
- Звіт зі списком curl команд, використаних для тестування кожної частини, та **обов'язково** скріншоти виконання команд

!!! warning "Перевірка збереження даних"
    Перезапустіть Flask-сервер та виконайте `GET /api/books` — дані повинні залишитися на місці, оскільки вони зберігаються у PostgreSQL, а не в пам'яті процесу. Покажіть це у звіті.
