# 25. (П12) Робота із сесіями у Flask. Персоналізація

## Передумови

- Прочитана [Лекція 24 — Сесії у Flask. Збереження стану користувача між запитами](/ua/courses/programming-2sem/module2/24-flask-sessions-lecture/)
- Виконана [Практична 11 — Реалізація CRUD операцій і робота з базою даних у Flask](/ua/courses/programming-2sem/module2/23-flask-crud-postgres-practice/)

!!! warning "Персоналізація"
    Імена користувачів у словнику `USERS` повинні містити **ваше ім'я та прізвище** (латиницею) як один із користувачів. На скріншотах у звіті повинно бути видно ваше ім'я.

## Завдання

Побудувати веб-застосунок **Personal Notes** з використанням Flask-сесій. Застосунок дозволяє користувачам логінитися, створювати персональні нотатки, налаштовувати інтерфейс та працювати з обраними нотатками. Усі дані зберігаються в пам'яті сервера, а стан користувача — у сесії.

### Підготовка проєкту

```bash
python3 -m venv env
source env/bin/activate
pip install flask
```

### Базова структура застосунку

Створіть файл `app.py` з початковою конфігурацією:

```python
from flask import Flask, session, request, jsonify

app = Flask(__name__)
app.secret_key = "my-secret-key-change-in-production"

# "База" користувачів (замініть одного користувача на своє ім'я)
USERS = {
    "ім'я_прізвище": "password123",
    "taras": "secret456",
}

# Сховище нотаток у пам'яті: {username: [{"id": 1, "title": "...", "text": "..."}]}
NOTES = {}

# Лічильник ID для нотаток
note_id_counter = 0
```

### Частина 1: Автентифікація (логін / вихід)

| Метод | URL | Опис | Коди відповідей |
|---|---|---|---|
| POST | `/login` | Логін користувача | 200, 400, 401 |
| POST | `/logout` | Вихід (очищення сесії) | 200 |
| GET | `/me` | Інформація про поточного користувача | 200, 401 |

**Вимоги:**

- **POST `/login`** — приймає JSON з полями `username` та `password`. Перевіряє облікові дані у словнику `USERS`. При успіху зберігає `username` у сесії та повертає привітання. При невірних даних — `401`.

```json
// Запит
{"username": "ім'я_прізвище", "password": "password123"}

// Успішна відповідь (200)
{"message": "Welcome, ім'я_прізвище!"}

// Помилка (401)
{"error": "Invalid credentials"}
```

- **POST `/logout`** — очищує всю сесію (`session.clear()`). Повертає повідомлення про вихід.

```json
{"message": "Logged out"}
```

- **GET `/me`** — повертає ім'я залогіненого користувача та його налаштування з сесії. Якщо користувач не залогінений — `401`.

```json
// Залогінений (200)
{
    "username": "ім'я_прізвище",
    "settings": {
        "language": "uk",
        "notes_per_page": 5
    }
}

// Не залогінений (401)
{"error": "Not logged in"}
```

### Частина 2: Персональні налаштування (сесія)

| Метод | URL | Опис | Коди відповідей |
|---|---|---|---|
| PUT | `/settings` | Оновити налаштування | 200, 401 |
| DELETE | `/settings` | Скинути налаштування | 200, 401 |

**Вимоги:**

- **PUT `/settings`** — приймає JSON з необов'язковими полями `language` та `notes_per_page`. Зберігає їх у сесії (`session["language"]`, `session["notes_per_page"]`). Оновлює лише ті поля, які передані в запиті. Якщо користувач не залогінений — `401`.

```json
// Запит
{"language": "en", "notes_per_page": 10}

// Відповідь (200)
{
    "message": "Settings updated",
    "settings": {
        "language": "en",
        "notes_per_page": 10
    }
}
```

- **DELETE `/settings`** — скидає налаштування до значень за замовчуванням (`language` → `"uk"`, `notes_per_page` → `5`). Видаляє відповідні ключі з сесії.

```json
{
    "message": "Settings reset to defaults",
    "settings": {
        "language": "uk",
        "notes_per_page": 5
    }
}
```

!!! note "Значення за замовчуванням"
    Якщо в сесії немає налаштувань, використовуйте значення за замовчуванням: `language` — `"uk"`, `notes_per_page` — `5`. Використовуйте `session.get("language", "uk")` для безпечного читання.

### Частина 3: Нотатки користувача

| Метод | URL | Опис | Коди відповідей |
|---|---|---|---|
| POST | `/api/notes` | Створити нотатку | 201, 400, 401 |
| GET | `/api/notes` | Список нотаток користувача | 200, 401 |
| DELETE | `/api/notes/<id>` | Видалити нотатку | 204, 401, 403, 404 |

**Вимоги:**

- Усі ендпоінти вимагають автентифікації (перевірка `session.get("user")`). Якщо користувач не залогінений — `401`.

- **POST `/api/notes`** — створює нотатку для поточного користувача. Обов'язкові поля: `title`, `text`. Нотатка зберігається у словнику `NOTES` під ключем `username`.

```json
// Запит
{"title": "My first note", "text": "This is the content of my note"}

// Відповідь (201)
{
    "id": 1,
    "title": "My first note",
    "text": "This is the content of my note",
    "author": "ім'я_прізвище"
}
```

**Валідація при створенні:**

- Тіло запиту має бути JSON
- `title` — обов'язкове, не порожній рядок
- `text` — обов'язкове, не порожній рядок

```json
{"error": "field 'title' is required"}
```

- **GET `/api/notes`** — повертає список нотаток поточного користувача.

```json
{
    "notes": [
        {"id": 1, "title": "My first note", "text": "...", "author": "ім'я_прізвище"},
        {"id": 2, "title": "Second note", "text": "...", "author": "ім'я_прізвище"}
    ]
}
```

- **DELETE `/api/notes/<id>`** — видаляє нотатку. Видалити можна лише свою нотатку (інакше — `403`). Якщо не знайдена — `404`. Повертає порожнє тіло з кодом `204`.

### Частина 4: Обрані нотатки (сесія)

| Метод | URL | Опис | Коди відповідей |
|---|---|---|---|
| POST | `/api/favorites/add` | Додати нотатку в обрані | 200, 400, 401, 404 |
| DELETE | `/api/favorites/<note_id>` | Видалити нотатку з обраних | 200, 401 |
| GET | `/api/favorites` | Список обраних нотаток | 200, 401 |

**Вимоги:**

- Список обраних нотаток зберігається **в сесії** (`session["favorites"]` — список ID нотаток). Обрані прив'язані до поточної сесії і зникнуть після виходу.

- **POST `/api/favorites/add`** — приймає JSON з полем `note_id`. Перевіряє, що нотатка існує та належить поточному користувачу. Якщо нотатка вже в обраних — повертає повідомлення без дублювання.

```json
// Запит
{"note_id": 1}

// Відповідь (200)
{"message": "Note added to favorites", "favorites": [1]}

// Якщо вже в обраних (200)
{"message": "Note already in favorites", "favorites": [1]}
```

- **DELETE `/api/favorites/<note_id>`** — видаляє нотатку з обраних.

- **GET `/api/favorites`** — повертає повну інформацію про обрані нотатки (не лише ID, а й дані нотаток).

```json
{
    "favorites": [
        {"id": 1, "title": "My first note", "text": "...", "author": "ім'я_прізвище"}
    ]
}
```

### Тестування через curl

За допомогою curl команд переконатися, що:

**Автентифікація та налаштування:**

```bash
# Логін (зберігаємо cookie)
curl -X POST http://127.0.0.1:5000/login \
  -H "Content-Type: application/json" \
  -d '{"username": "ім'я_прізвище", "password": "password123"}' \
  -c cookies.txt

# Перевірка: хто я?
curl http://127.0.0.1:5000/me -b cookies.txt

# Спроба без логіну (без cookie)
curl http://127.0.0.1:5000/me

# Змінити налаштування
curl -X PUT http://127.0.0.1:5000/settings \
  -H "Content-Type: application/json" \
  -d '{"language": "en", "notes_per_page": 2}' \
  -b cookies.txt -c cookies.txt

# Перевірити налаштування через /me
curl http://127.0.0.1:5000/me -b cookies.txt

# Скинути налаштування
curl -X DELETE http://127.0.0.1:5000/settings \
  -b cookies.txt -c cookies.txt
```

**Нотатки та обрані:**

```bash
# Створити нотатки
curl -X POST http://127.0.0.1:5000/api/notes \
  -H "Content-Type: application/json" \
  -d '{"title": "Flask Sessions", "text": "Learning about sessions in Flask"}' \
  -b cookies.txt -c cookies.txt

curl -X POST http://127.0.0.1:5000/api/notes \
  -H "Content-Type: application/json" \
  -d '{"title": "Python Tips", "text": "Useful Python tips and tricks"}' \
  -b cookies.txt -c cookies.txt

# Список нотаток
curl http://127.0.0.1:5000/api/notes -b cookies.txt

# Додати до обраних
curl -X POST http://127.0.0.1:5000/api/favorites/add \
  -H "Content-Type: application/json" \
  -d '{"note_id": 1}' \
  -b cookies.txt -c cookies.txt

# Список обраних
curl http://127.0.0.1:5000/api/favorites -b cookies.txt

# Видалити з обраних
curl -X DELETE http://127.0.0.1:5000/api/favorites/1 \
  -b cookies.txt -c cookies.txt

# Видалити нотатку
curl -X DELETE http://127.0.0.1:5000/api/notes/2 -b cookies.txt -c cookies.txt
```

**Ізоляція та вихід:**

```bash
# Логін іншим користувачем
curl -X POST http://127.0.0.1:5000/login \
  -H "Content-Type: application/json" \
  -d '{"username": "taras", "password": "secret456"}' \
  -c cookies2.txt

# Перевірити, що нотатки першого користувача недоступні
curl http://127.0.0.1:5000/api/notes -b cookies2.txt

# Вихід
curl -X POST http://127.0.0.1:5000/logout -b cookies.txt -c cookies.txt

# Перевірити, що сесія очищена (обрані зникли)
curl http://127.0.0.1:5000/me -b cookies.txt
```

## Результат

Після виконання роботи студент здає:

- **`app.py`** — Flask-застосунок з усіма маршрутами
- Звіт зі списком curl команд, використаних для тестування кожної частини, та **обов'язково** скріншоти виконання команд

!!! warning "Перевірка ізоляції"
    У звіті продемонструйте, що два різних користувачі мають окремі нотатки та обрані. Залогіньтесь двома користувачами (у різних файлах cookie) та покажіть, що дані не перетинаються.
