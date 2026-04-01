# 27. (П13) Розробка та виконання тестів для REST API з використанням pytest

## Передумови

- Прочитана [Лекція 26 — Тестування REST API за допомогою pytest](/ua/courses/programming-2sem/module2/26-testing-rest-api-lecture/)
- Виконана [Практична 11 — Реалізація CRUD операцій і робота з базою даних у Flask](/ua/courses/programming-2sem/module2/23-flask-crud-postgres-practice/)
- Встановлений Docker

!!! warning "Персоналізація"
    У полі `created_by` при створенні книг повинно бути **ваше ім'я та прізвище**. На скріншотах у звіті повинно бути видно ваше ім'я у відповідях API та у виводі pytest.

## Завдання

Написати автоматичні тести для **Book Library API** з [Практичної 11](/ua/courses/programming-2sem/module2/23-flask-crud-postgres-practice/). Замість ручного тестування через curl, ви створите набір pytest-тестів, які перевіряють коректність роботи API з PostgreSQL.

### Підготовка середовища

#### PostgreSQL у Docker

```bash
docker run -d \
  --name library-test-postgres \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  postgres:17
```

#### Проєкт

```bash
python3 -m venv env
source env/bin/activate
pip install flask psycopg2-binary pytest
```

### Підготовка застосунку

Візьміть `app.py` з Практичної 11 і перетворіть його на **application factory** (як у [лекції 26](/ua/courses/programming-2sem/module2/26-testing-rest-api-lecture/#application-factory)):

- Винесіть конфігурацію БД у `DEFAULT_DB_CONFIG`
- Обгорніть усі маршрути у функцію `create_app(db_config=None)`
- Замініть глобальний `app = create_app()` на блок `if __name__ == "__main__":`

Це дозволить тестам підставити окрему тестову базу даних.

### Структура проєкту

```
library-api/
├── app.py                # Flask-застосунок (application factory)
├── conftest.py           # Фікстури pytest
├── test_authors.py       # Тести для авторів
├── test_books.py         # Тести для книг
└── requirements.txt
```

### Фікстури (conftest.py)

Створіть файл `conftest.py` з фікстурами за зразком з [лекції 26](/ua/courses/programming-2sem/module2/26-testing-rest-api-lecture/#_7):

- `test_db` (scope=session) — створює тестову базу `library_test_db`, видаляє після всіх тестів
- `app` (scope=session) — створює Flask-застосунок із тестовою БД
- `client` (scope=function) — очищує таблиці `books` та `authors` перед кожним тестом, повертає тестовий клієнт

!!! note "Порядок очищення таблиць"
    При очищенні спочатку видаляйте `books` (бо має зовнішній ключ на `authors`), потім `authors`. Або використовуйте `TRUNCATE books, authors RESTART IDENTITY CASCADE`.

### Частина 1: Тести для авторів

Створіть файл `test_authors.py` з класом `TestAuthors`. Напишіть тести для таких сценаріїв:

| Тест | Що перевіряє | Очікуваний статус |
|---|---|---|
| `test_get_authors_empty` | Список авторів порожній | 200, `[]` |
| `test_create_author` | Створення автора з `name` та `birth_year` | 201 |
| `test_create_author_without_name` | Створення без обов'язкового поля | 400 |
| `test_get_author_by_id` | Отримання автора за ID | 200 |
| `test_get_author_not_found` | Запит неіснуючого автора | 404 |
| `test_delete_author` | Видалення автора | 204 |
| `test_delete_author_not_found` | Видалення неіснуючого автора | 404 |

**Вимоги до тестів:**

- При створенні автора перевіряйте, що у відповіді є поля `id`, `name`, `birth_year`
- Після видалення автора перевіряйте через GET, що він більше недоступний (404)

**Приклад тесту:**

```python
class TestAuthors:
    def test_create_author(self, client):
        response = client.post("/api/authors", json={
            "name": "Тарас Шевченко",
            "birth_year": 1814,
        })

        assert response.status_code == 201
        data = response.get_json()
        assert data["name"] == "Тарас Шевченко"
        assert data["birth_year"] == 1814
        assert "id" in data
```

### Частина 2: Тести для книг

Створіть файл `test_books.py` з класом `TestBooks`. Напишіть тести для таких сценаріїв:

| Тест | Що перевіряє | Очікуваний статус |
|---|---|---|
| `test_get_books_empty` | Список книг порожній | 200, `[]` |
| `test_create_book` | Створення книги з усіма полями | 201 |
| `test_create_book_without_title` | Створення без `title` | 400 |
| `test_create_book_without_created_by` | Створення без `created_by` | 400 |
| `test_create_book_with_author` | Створення книги з прив'язкою до автора | 201 |
| `test_create_book_with_nonexistent_author` | Прив'язка до неіснуючого автора | 400 |
| `test_get_book_by_id` | Отримання книги за ID | 200 |
| `test_get_book_not_found` | Запит неіснуючої книги | 404 |
| `test_delete_book` | Видалення книги | 204 |

**Вимоги до тестів:**

- У полі `created_by` використовуйте **ваше ім'я та прізвище**
- При створенні книги з автором спочатку створіть автора через POST, потім використайте його `id`
- Після видалення книги перевіряйте через GET, що вона більше недоступна

**Приклад тесту:**

```python
class TestBooks:
    def test_create_book_with_author(self, client):
        # Спочатку створюємо автора
        author = client.post("/api/authors", json={
            "name": "Леся Українка",
            "birth_year": 1871,
        }).get_json()

        # Створюємо книгу з прив'язкою до автора
        response = client.post("/api/books", json={
            "title": "Lisova Pisnia",
            "genre": "drama",
            "year_published": 1911,
            "author_id": author["id"],
            "created_by": "Ім'я Прізвище",
        })

        assert response.status_code == 201
        data = response.get_json()
        assert data["author_id"] == author["id"]
        assert data["created_by"] == "Ім'я Прізвище"
```

### Частина 3: Тести для фільтрації та зв'язків

Додайте до `test_books.py` клас `TestBooksFilter` та до `test_authors.py` клас `TestAuthorBooks`:

**Фільтрація книг (`TestBooksFilter`):**

| Тест | Що перевіряє |
|---|---|
| `test_filter_by_genre` | `GET /api/books?genre=poetry` повертає лише книги з жанром `poetry` |
| `test_filter_by_author_id` | `GET /api/books?author_id=1` повертає лише книги автора |
| `test_search_by_title` | `GET /api/books?q=kobzar` знаходить книгу за частковим збігом |
| `test_filter_no_results` | Фільтрація за неіснуючим жанром повертає `[]` |

**Книги автора (`TestAuthorBooks`):**

| Тест | Що перевіряє |
|---|---|
| `test_get_author_books` | `GET /api/authors/<id>/books` повертає книги автора |
| `test_get_author_books_empty` | Автор без книг повертає `[]` |
| `test_get_author_books_not_found` | Неіснуючий автор повертає 404 |

**Приклад тесту з підготовкою даних:**

```python
class TestBooksFilter:
    def test_filter_by_genre(self, client):
        # Підготовка: створюємо книги різних жанрів
        client.post("/api/books", json={
            "title": "Kobzar",
            "genre": "poetry",
            "created_by": "Ім'я Прізвище",
        })
        client.post("/api/books", json={
            "title": "Tygrolovy",
            "genre": "novel",
            "created_by": "Ім'я Прізвище",
        })

        # Фільтрація
        response = client.get("/api/books?genre=poetry")
        data = response.get_json()

        assert len(data) == 1
        assert data[0]["title"] == "Kobzar"
```

### Частина 4: Тест видалення автора з книгами

Додайте до `test_authors.py` тест, що перевіряє каскадну поведінку `ON DELETE SET NULL`:

| Тест | Що перевіряє |
|---|---|
| `test_delete_author_keeps_books` | Після видалення автора його книги залишаються, але `author_id` стає `null` |

```python
def test_delete_author_keeps_books(self, client):
    # Створюємо автора та книгу
    author = client.post("/api/authors", json={
        "name": "Іван Франко",
    }).get_json()

    book = client.post("/api/books", json={
        "title": "Zakhar Berkut",
        "author_id": author["id"],
        "created_by": "Ім'я Прізвище",
    }).get_json()

    # Видаляємо автора
    client.delete(f"/api/authors/{author['id']}")

    # Книга залишилась, але без автора
    response = client.get(f"/api/books/{book['id']}")
    assert response.status_code == 200
    assert response.get_json()["author_id"] is None
```

### Запуск тестів

```bash
pytest -v
```

Очікуваний результат:

```
test_authors.py::TestAuthors::test_get_authors_empty PASSED
test_authors.py::TestAuthors::test_create_author PASSED
test_authors.py::TestAuthors::test_create_author_without_name PASSED
test_authors.py::TestAuthors::test_get_author_by_id PASSED
test_authors.py::TestAuthors::test_get_author_not_found PASSED
test_authors.py::TestAuthors::test_delete_author PASSED
test_authors.py::TestAuthors::test_delete_author_not_found PASSED
test_authors.py::TestAuthorBooks::test_get_author_books PASSED
test_authors.py::TestAuthorBooks::test_get_author_books_empty PASSED
test_authors.py::TestAuthorBooks::test_get_author_books_not_found PASSED
test_authors.py::TestAuthors::test_delete_author_keeps_books PASSED
test_books.py::TestBooks::test_get_books_empty PASSED
test_books.py::TestBooks::test_create_book PASSED
test_books.py::TestBooks::test_create_book_without_title PASSED
test_books.py::TestBooks::test_create_book_without_created_by PASSED
test_books.py::TestBooks::test_create_book_with_author PASSED
test_books.py::TestBooks::test_create_book_with_nonexistent_author PASSED
test_books.py::TestBooks::test_get_book_by_id PASSED
test_books.py::TestBooks::test_get_book_not_found PASSED
test_books.py::TestBooks::test_delete_book PASSED
test_books.py::TestBooksFilter::test_filter_by_genre PASSED
test_books.py::TestBooksFilter::test_filter_by_author_id PASSED
test_books.py::TestBooksFilter::test_search_by_title PASSED
test_books.py::TestBooksFilter::test_filter_no_results PASSED

========================= 24 passed =========================
```

## Результат

Після виконання роботи студент здає:

- **`app.py`** — Flask-застосунок з application factory
- **`conftest.py`** — фікстури для тестування з PostgreSQL
- **`test_authors.py`** — тести для авторів (класи `TestAuthors`, `TestAuthorBooks`)
- **`test_books.py`** — тести для книг (класи `TestBooks`, `TestBooksFilter`)
- Звіт зі скріншотом виконання `pytest -v`, де **всі тести проходять** та видно ваше ім'я у назвах тестових даних
