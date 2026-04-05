# 29. (П14) Створення GitHub workflow для автоматичного тестування Python-застосунку

## Передумови

- Прочитана [Лекція 28 — Неперервна інтеграція (CI). Знайомство з GitHub Actions](/ua/courses/programming-2sem/module2/28-ci-github-actions-lecture/)
- Виконана [Практична 13 — Тестування REST API](/ua/courses/programming-2sem/module2/27-testing-rest-api-practice/)
- Встановлений Git
- Є акаунт на GitHub

!!! warning "Персоналізація"
    Репозиторій на GitHub повинен бути створений під **вашим акаунтом**. На скріншотах у звіті повинно бути видно ваше ім'я користувача GitHub, назву репозиторію та результати виконання workflow.

## Завдання

Налаштувати CI за допомогою GitHub Actions для **Book Library API** з [Практичної 13](/ua/courses/programming-2sem/module2/27-testing-rest-api-practice/). При кожному push у репозиторій тести повинні запускатись автоматично з PostgreSQL у GitHub Actions.

### Частина 1: Підготовка репозиторію

#### Створення репозиторію на GitHub

1. Створіть новий публічний репозиторій на GitHub (наприклад, `library-api`)
2. Клонуйте його локально:

```bash
git clone https://github.com/<your-username>/library-api.git
cd library-api
```

#### Структура проєкту

Скопіюйте файли з Практичної 13 у репозиторій:

```
library-api/
├── app.py                # Flask-застосунок (application factory)
├── conftest.py           # Фікстури pytest
├── test_authors.py       # Тести для авторів
├── test_books.py         # Тести для книг
└── requirements.txt
```

#### Файл requirements.txt

Переконайтесь, що `requirements.txt` містить усі залежності:

```
flask
psycopg2-binary
pytest
```

#### Перший коміт

```bash
git add .
git commit -m "Add library API with tests"
git push
```

### Частина 2: Адаптація тестів для CI

У Практичній 13 конфігурація бази даних була захардкоджена в `conftest.py`. Для CI потрібно зробити її гнучкою — щоб тести могли використовувати конфігурацію як з локального середовища, так і з GitHub Actions.

#### Читання конфігурації із змінних середовища

Змініть `conftest.py`, щоб конфігурація БД читалась зі змінних середовища з fallback на локальні значення:

```python
import os
import pytest
import psycopg2
from app import create_app

TEST_DB_CONFIG = {
    "host": os.environ.get("POSTGRES_HOST", "localhost"),
    "port": int(os.environ.get("POSTGRES_PORT", "5432")),
    "database": os.environ.get("POSTGRES_DB", "library_test_db"),
    "user": os.environ.get("POSTGRES_USER", "postgres"),
    "password": os.environ.get("POSTGRES_PASSWORD", "secret"),
}

# ... решта фікстур залишається без змін
```

Тепер тести працюють і локально (з дефолтними значеннями), і в CI (де змінні середовища задаються у workflow).

### Частина 3: Створення CI workflow

Створіть файл `.github/workflows/ci.yml`:

```bash
mkdir -p .github/workflows
```

Напишіть workflow, який:

1. Запускається при **push** у гілку `main` та при **pull request** у `main`
2. Використовує runner `ubuntu-latest`
3. Піднімає **PostgreSQL 17** як сервіс з health check
4. Встановлює **Python 3.12**
5. Встановлює залежності з `requirements.txt`
6. Запускає `pytest -v`

**Вимоги до сервісу PostgreSQL:**

| Параметр | Значення |
|---|---|
| Образ | `postgres:17` |
| `POSTGRES_PASSWORD` | `secret` |
| `POSTGRES_DB` | `library_test_db` |
| Порт | `5432:5432` |
| Health check | `pg_isready`, інтервал 10s, таймаут 5s, 5 спроб |

**Вимоги до кроку запуску тестів:**

Передайте змінні середовища для підключення до бази:

```yaml
- name: Run tests
  run: pytest -v
  env:
    POSTGRES_HOST: localhost
    POSTGRES_PORT: 5432
    POSTGRES_DB: library_test_db
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: secret
```

!!! note "Підказка"
    За основу візьміть приклад з [лекції 28](/ua/courses/programming-2sem/module2/28-ci-github-actions-lecture/#_9). Додайте секцію `env` до кроку запуску тестів.

#### Push та перевірка

```bash
git add .github/workflows/ci.yml conftest.py
git commit -m "Add CI workflow"
git push
```

Після push перейдіть у вкладку **Actions** вашого репозиторію на GitHub та перевірте, що workflow запустився.

### Частина 4: Перевірка CI на pull request

Переконайтесь, що CI працює також для pull request:

1. Створіть нову гілку:

```bash
git checkout -b test-ci-pr
```

2. Додайте новий тест у `test_books.py`:

```python
def test_create_book_default_status(self, client):
    """Книга створюється зі статусом за замовчуванням"""
    response = client.post("/api/books", json={
        "title": "Test Book",
        "created_by": "Ім'я Прізвище",
    })

    assert response.status_code == 201
```

3. Зробіть коміт та push нової гілки:

```bash
git add test_books.py
git commit -m "Add test for default book status"
git push -u origin test-ci-pr
```

4. Створіть Pull Request на GitHub (base: `main`, compare: `test-ci-pr`)
5. Переконайтесь, що CI запустився для PR та успішно пройшов

В інтерфейсі PR ви побачите статус перевірки CI:

- Жовтий індикатор — workflow ще виконується
- Зелена галочка — всі перевірки пройшли
- Червоний хрестик — є помилки

## Результат

Після виконання роботи студент здає:

- Посилання на **GitHub-репозиторій** з кодом та workflow
- Звіт зі скріншотами:
    - Вкладка Actions з успішним workflow (зелена галочка)
    - Логи кроку "Run tests" з виводом pytest
    - Pull request зі статусом CI-перевірки
