# 24. (Л) Сесії у Flask. Збереження стану користувача між запитами

## Зміст лекції

1. Проблема: HTTP — протокол без стану
2. Що таке сесія
3. Два підходи до реалізації сесій
4. Як працюють сесії у Flask
5. Налаштування секретного ключа
6. Запис і читання даних сесії
7. Видалення даних сесії
8. Практичний приклад: система логіну
9. Час життя сесії
10. Сесії та REST API
11. JWT — токени для REST API

## Проблема: HTTP — протокол без стану

HTTP — це **stateless**-протокол. Це означає, що кожен запит є незалежним: сервер не "пам'ятає" попередні запити від того самого клієнта.

```mermaid
sequenceDiagram
    participant B as Браузер
    participant S as Flask-сервер

    B->>S: POST /login (username=taras, password=secret)
    S->>B: 200 OK — "Welcome, taras!"

    B->>S: GET /dashboard
    S->>B: 401 — "Who are you?"
    Note right of S: Сервер не знає,<br/>хто зробив цей запит
```

Після успішного логіну сервер одразу "забуває" про користувача. Наступний запит для сервера — це абсолютно новий, незнайомий клієнт.

Але в реальних застосунках нам потрібно зберігати стан між запитами: хто залогінений, які налаштування сайту обрав користувач і т.д.. Саме для цього існують **сесії**.

## Що таке сесія

**Сесія (session)** — це механізм збереження даних, прив'язаних до конкретного користувача, між кількома HTTP-запитами.

Типові приклади використання сесій:

- **Автентифікація** — зберігаємо ім'я або ID залогіненого користувача
- **Кошик покупок** — зберігаємо список товарів між переходами по сторінках
- **Налаштування** — мова інтерфейсу, тема оформлення
- **Flash-повідомлення** — одноразові повідомлення ("Задачу створено!")

## Два підходи до реалізації сесій

Існує два основних способи зберігання даних сесії:

### Cookie-based sessions (клієнтські сесії)

**Всі дані** сесії зберігаються безпосередньо у cookie браузера. Сервер нічого не зберігає — він лише підписує дані, щоб клієнт не міг їх підробити.

### Server-side sessions (серверні сесії)

Дані сесії зберігаються **на сервері** (у базі даних, Redis, Memcached або навіть у пам'яті процесу). Клієнт отримує лише **session ID** — унікальний ідентифікатор, за яким сервер знаходить відповідні дані.

```mermaid
graph LR
    subgraph "Cookie-based"
        A1["Cookie<br/>Усі дані (підписані)"] -->|"Кожен запит"| S1["Сервер<br/>Нічого не зберігає"]
    end

    subgraph "Server-side"
        A2["Cookie<br/>Лише session_id"] -->|"Кожен запит"| S2["Сервер<br/>Шукає дані за ID"]
        S2 --- DB["Сховище<br/>(Redis, БД, пам'ять)"]
    end

    style A1 fill:#ffd43b,stroke:#333,color:#000
    style S1 fill:#51cf66,stroke:#333,color:#000
    style A2 fill:#ffd43b,stroke:#333,color:#000
    style S2 fill:#51cf66,stroke:#333,color:#000
    style DB fill:#339af0,stroke:#333,color:#fff
```

| Критерій | Cookie-based | Server-side |
|---|---|---|
| **Де зберігаються дані** | У cookie клієнта | На сервері |
| **Що отримує клієнт** | Підписані дані | Лише session ID |
| **Обмеження розміру** | ~4 КБ (ліміт cookie) | Практично необмежений |
| **Видимість даних** | Клієнт бачить дані | Клієнт бачить лише ID |
| **Відкликання сесії** | Неможливо до закінчення терміну | Можна видалити на сервері |
| **Масштабування** | Просто (нічого на сервері) | Потрібне спільне сховище |

Flask за замовчуванням використовує **cookie-based sessions** — це простіший підхід, який не потребує додаткової інфраструктури. Для серверних сесій у Flask існує бібліотека [Flask-Session](https://flask-session.readthedocs.io/), яка підтримує Redis, Memcached, SQLAlchemy та інші бекенди.

## Як працюють сесії у Flask

Як ми вже знаємо, Flask використовує **cookie-based sessions** — дані сесії зберігаються безпосередньо у cookie браузера клієнта.

```mermaid
sequenceDiagram
    participant B as Браузер
    participant S as Flask-сервер

    B->>S: POST /login (username=taras)
    Note right of S: session["user"] = "taras"<br/>Flask підписує дані
    S->>B: 200 OK<br/>Header: Set-Cookie: session=eyJ...(підписані дані)

    B->>S: GET /dashboard<br/>Header: Cookie: session=eyJ...
    Note right of S: Flask розшифровує cookie<br/>session["user"] → "taras"
    S->>B: 200 OK — "Dashboard for taras"
```

### Як це працює крок за кроком

1. Сервер записує дані в об'єкт `session` (наприклад, `session["user"] = "taras"`)
2. Flask **серіалізує** дані в JSON і **підписує** їх криптографічним ключем
3. Підписані дані відправляються клієнту як cookie `session`
4. Браузер автоматично додає цю cookie до кожного наступного запиту
5. Flask **перевіряє підпис** і **десеріалізує** дані назад у об'єкт `session`

```mermaid
graph LR
    A["Python dict<br/>{'user': 'taras'}"] -->|"Серіалізація + підпис"| B["Cookie<br/>eyJfdWVyIjoidGF..."]
    B -->|"Перевірка підпису + десеріалізація"| A

    style A fill:#51cf66,stroke:#333,color:#000
    style B fill:#ffd43b,stroke:#333,color:#000
```

!!! warning "Cookie-сесії: підписані, але НЕ зашифровані"
    Flask **підписує** cookie, щоб клієнт не міг змінити дані (наприклад, змінити `user` з `"taras"` на `"admin"`). Якщо хтось змінить cookie, підпис не збігатиметься, і Flask відхилить ці дані.

    Але дані **не зашифровані** — їх можна декодувати та прочитати. Тому **ніколи не зберігайте в сесії паролі, токени чи інші секрети**. Зберігайте лише ідентифікатори (наприклад, ID користувача), а чутливі дані тримайте на сервері (у базі даних).

## Налаштування секретного ключа

Для підпису cookie Flask потребує **секретний ключ**. Без нього сесії не працюватимуть.

```python
from flask import Flask

app = Flask(__name__)

# Секретний ключ для підпису cookie
app.secret_key = "my-secret-key-change-in-production"
```

!!! danger "Секретний ключ у production"
    У прикладах ми записуємо ключ прямо в коді — це прийнятно лише для навчання. У production секретний ключ повинен бути:

    - **Довгим і випадковим**
    - **Збережений у змінній оточення** — не в коді:

    ```python
    import os

    app.secret_key = os.environ["SECRET_KEY"]
    ```

    Якщо зловмисник дізнається секретний ключ, він зможе підробити будь-яку сесію.

## Запис і читання даних сесії

Об'єкт `session` у Flask працює як звичайний Python-словник:

```python
from flask import Flask, session, request, jsonify

app = Flask(__name__)
app.secret_key = "my-secret-key"


@app.route("/set-language", methods=["POST"])
def set_language():
    data = request.json
    lang = data.get("language", "uk")

    # Записуємо в сесію
    session["language"] = lang

    return jsonify({"message": f"Language set to {lang}"})


@app.route("/get-language")
def get_language():
    # Читаємо з сесії
    lang = session.get("language", "uk")

    return jsonify({"language": lang})
```

Тестування:

```bash
# Встановити мову (curl зберігає cookie у файл)
curl -X POST http://127.0.0.1:5000/set-language \
  -H "Content-Type: application/json" \
  -d '{"language": "en"}' \
  -c cookies.txt

# Прочитати мову (curl надсилає збережену cookie)
curl http://127.0.0.1:5000/get-language \
  -b cookies.txt

# Відповідь: {"language": "en"}
```

!!! note "Прапорці `-c` та `-b` у curl"
    - `-c cookies.txt` — зберігає cookie, отримані від сервера, у файл
    - `-b cookies.txt` — надсилає cookie з файлу разом із запитом

    Браузер робить це автоматично, але curl за замовчуванням не зберігає cookie між запитами.

### Основні операції з `session`

```python
# Записати значення
session["key"] = "value"

# Прочитати значення (KeyError, якщо ключ відсутній)
value = session["key"]

# Безпечне читання (None, якщо ключ відсутній)
value = session.get("key")

# Безпечне читання зі значенням за замовчуванням
value = session.get("key", "default")

# Перевірити наявність ключа
if "key" in session:
    ...

# Видалити один ключ
session.pop("key", None)
```

## Видалення даних сесії

### Видалення окремого ключа

```python
@app.route("/clear-language", methods=["POST"])
def clear_language():
    session.pop("language", None)
    return jsonify({"message": "Language preference cleared"})
```

### Повне очищення сесії

```python
@app.route("/logout", methods=["POST"])
def logout():
    session.clear()
    return jsonify({"message": "Logged out"})
```

`session.clear()` видаляє **всі** дані сесії — корисно при виході користувача.

## Практичний приклад: система логіну

Побудуємо простий застосунок з логіном, захищеною сторінкою та виходом. Цього разу — з HTML-сторінками, які можна відкрити у браузері.

```python
from flask import Flask, session, request, redirect

app = Flask(__name__)
app.secret_key = "my-secret-key"

# Спрощена "база" користувачів
USERS = {
    "taras": "secret123",
    "olena": "password456",
}


@app.route("/")
def index():
    user = session.get("user")

    if user:
        return f"""
        <h1>Hello, {user}!</h1>
        <p>You are logged in.</p>
        <a href="/dashboard">Dashboard</a>
        <form action="/logout" method="post" style="display:inline">
            <button type="submit">Logout</button>
        </form>
        """

    return """
    <h1>Welcome</h1>
    <p>You are not logged in.</p>
    <a href="/login">Login</a>
    """


@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "GET":
        return """
        <h1>Login</h1>
        <form action="/login" method="post">
            <label>Username: <input name="username"></label><br><br>
            <label>Password: <input name="password" type="password"></label><br><br>
            <button type="submit">Login</button>
        </form>
        """

    username = request.form.get("username")
    password = request.form.get("password")

    if not username or not password:
        return "<h1>Error</h1><p>Username and password are required</p>", 400

    # Перевіряємо облікові дані
    if USERS.get(username) != password:
        return """
        <h1>Error</h1>
        <p>Invalid credentials</p>
        <a href="/login">Try again</a>
        """, 401

    # Зберігаємо ім'я користувача в сесії
    session["user"] = username

    return redirect("/dashboard")


@app.route("/dashboard")
def dashboard():
    # Перевіряємо, чи є користувач у сесії
    user = session.get("user")

    if not user:
        return """
        <h1>Access denied</h1>
        <p>Please <a href="/login">login</a> first.</p>
        """, 401

    return f"""
    <h1>Dashboard</h1>
    <p>Welcome to your dashboard, <b>{user}</b>!</p>
    <a href="/">Home</a>
    <form action="/logout" method="post" style="display:inline">
        <button type="submit">Logout</button>
    </form>
    """


@app.route("/logout", methods=["POST"])
def logout():
    session.pop("user", None)
    return redirect("/")
```

### Як це працює у браузері

1. Відкрийте `http://127.0.0.1:5000/` — побачите "You are not logged in"
2. Перейдіть за посиланням **Login** — з'явиться HTML-форма
3. Введіть `taras` / `secret123` і натисніть **Login**
4. Браузер надішле POST-запит, Flask запише `session["user"] = "taras"` і перенаправить на `/dashboard`
5. На dashboard натисніть **Logout** — сесія очиститься, і ви повернетесь на головну

!!! note "`redirect()` — перенаправлення"
    Функція `redirect("/dashboard")` повертає відповідь з кодом `302 Found` і заголовком `Location: /dashboard`. Браузер автоматично переходить на вказаний URL. Це стандартний підхід після успішного POST-запиту (паттерн **Post/Redirect/Get**), який запобігає повторній відправці форми при оновленні сторінки.

### Діаграма потоку

```mermaid
sequenceDiagram
    participant B as Браузер
    participant S as Flask-сервер

    B->>S: GET /dashboard
    S->>B: 401 — "Please login first"

    B->>S: GET /login
    S->>B: 200 — HTML-форма логіну

    B->>S: POST /login (username=taras, password=secret123)
    Note right of S: session["user"] = "taras"
    S->>B: 302 Redirect → /dashboard<br/>Header: Set-Cookie: session=...

    B->>S: GET /dashboard<br/>Header: Cookie: session=...
    Note right of S: session["user"] → "taras"
    S->>B: 200 — "Welcome to your dashboard, taras!"

    B->>S: POST /logout<br/>Header: Cookie: session=...
    Note right of S: session.pop("user")
    S->>B: 302 Redirect → /

    B->>S: GET /
    S->>B: 200 — "You are not logged in"
```

## Час життя сесії

За замовчуванням cookie сесії Flask — це **session cookie**, яка існує, поки браузер відкритий. Коли користувач закриє браузер, cookie видаляється.

### Постійна сесія

Щоб сесія зберігалася після закриття браузера, використовуйте [session.permanent](https://flask.palletsprojects.com/en/stable/config/?utm_source=chatgpt.com#PERMANENT_SESSION_LIFETIME):

```python
from datetime import timedelta

app.permanent_session_lifetime = timedelta(days=7)


@app.route("/login", methods=["POST"])
def login():
    data = request.json
    username = data.get("username")
    password = data.get("password")

    if USERS.get(username) != password:
        return jsonify({"error": "Invalid credentials"}), 401

    session.permanent = True
    session["user"] = username

    return jsonify({"message": f"Welcome, {username}!"})
```

| Параметр | За замовчуванням | Опис |
|---|---|---|
| `session.permanent` | `False` | Якщо `True` — cookie має `Expires`, інакше — session cookie |
| `app.permanent_session_lifetime` | `timedelta(days=31)` | Час життя постійної сесії |

## Сесії та REST API

У попередніх лекціях ([16](../module2/16-http-rest-lecture.md), [22](../module2/22-flask-crud-postgres-lecture.md)) ми будували REST API. Один із принципів REST — **statelessness**: сервер не повинен зберігати стан клієнта між запитами.

Це означає, що сесії **не використовують** у "чистих" REST API. Замість сесій використовують **токени** (наприклад, JWT), які клієнт передає в заголовку `Authorization` з кожним запитом.

```mermaid
graph TD
    A{"Який тип застосунку?"} -->|"Веб-сайт з HTML-сторінками"| B["Сесії (cookies)<br/>session['user'] = 'taras'"]
    A -->|"REST API для мобільних/SPA"| C["Токени (JWT)<br/>Authorization: Bearer eyJ..."]

    style A fill:#339af0,stroke:#333,color:#fff
    style B fill:#51cf66,stroke:#333,color:#000
    style C fill:#ffd43b,stroke:#333,color:#000
```

| Підхід | Де зберігається стан | Коли використовувати |
|---|---|---|
| **Сесії** | Cookie в браузері або сховище на сервері | Веб-застосунки з HTML-сторінками |
| **Токени (JWT)** | Клієнт (localStorage, заголовок) | REST API, мобільні застосунки, SPA |

!!! info "Що таке localStorage?"
    `localStorage` — це вбудоване сховище браузера (Web Storage API), яке дозволяє JavaScript-коду зберігати дані у вигляді key-value пар. На відміну від cookie, дані з `localStorage` **не надсилаються автоматично** з кожним запитом — клієнтський код сам додає токен у заголовок `Authorization` при кожному запиті до API.

!!! note "Чому ми вивчаємо сесії?"
    Хоча для REST API сесії не підходять, вони залишаються фундаментальним механізмом веб-розробки. Розуміння сесій необхідне для:

    - Роботи з веб-застосунками, що рендерять HTML на сервері
    - Розуміння основ автентифікації та авторизації
    - Побудови адмін-панелей та внутрішніх інструментів

## JWT — токени для REST API

**JWT (JSON Web Token)** — це стандарт для передачі автентифікаційних даних у вигляді підписаного токена. Як і cookie-based сесії, JWT не потребує зберігання стану на сервері — вся інформація міститься в самому токені. Але на відміну від сесій, JWT призначений для REST API, де клієнт сам додає токен до кожного запиту.

### Як працює JWT

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Flask-сервер

    C->>S: POST /login {"username": "taras", "password": "secret123"}
    Note right of S: Перевірка пароля ✓<br/>Генерація JWT-токена
    S->>C: 200 OK {"token": "eyJhbGci..."}

    C->>S: GET /api/tasks<br/>Header: Authorization: Bearer eyJhbGci...
    Note right of S: Перевірка підпису токена ✓<br/>Декодування даних користувача
    S->>C: 200 OK [{"id": 1, "title": "..."}]

    C->>S: GET /api/tasks<br/>(без заголовка Authorization)
    S->>C: 401 Unauthorized
```

1. Клієнт надсилає логін і пароль
2. Сервер перевіряє облікові дані та генерує **підписаний токен**
3. Клієнт зберігає токен і додає його до кожного запиту в заголовку `Authorization`
4. Сервер перевіряє підпис токена та витягує дані користувача

### Структура JWT

JWT складається з трьох частин, розділених крапкою:

```
Header:    eyJhbGciOiJIUzI1NiJ9
Payload:   eyJ1c2VyIjoidGFyYXMiLCJleHAiOjE3NDM5ODQ2MDB9
Signature: SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

У токені ці три частини з'єднані крапкою: `Header.Payload.Signature`.

| Частина | Що містить | Приклад (після декодування з Base64) |
|---|---|---|
| **Header** | Алгоритм підпису | `{"alg": "HS256"}` |
| **Payload** | Дані користувача + метадані | `{"user": "taras", "exp": 1743984600}` |
| **Signature** | Підпис (Header + Payload + секретний ключ) | бінарні дані |

Header і Payload — це просто JSON, закодований у Base64. Їх може прочитати будь-хто. **Signature** гарантує, що дані не були змінені — аналогічно до підпису cookie у Flask-сесіях.

!!! warning "JWT, як і Flask-сесії, підписаний, але НЕ зашифрований"
    Payload токена може прочитати будь-хто — достатньо декодувати Base64. Тому не зберігайте в JWT паролі чи інші секрети. Зберігайте лише ідентифікатори: ім'я користувача, ID, роль.

### Приклад: JWT-автентифікація у Flask

Встановимо бібліотеку для роботи з JWT:

```bash
pip install pyjwt
```

Повний приклад:

```python
import datetime

import jwt
from flask import Flask, jsonify, request

app = Flask(__name__)

SECRET_KEY = "my-jwt-secret-key"

USERS = {
    "taras": "secret123",
    "olena": "password456",
}


@app.route("/login", methods=["POST"])
def login():
    data = request.json

    if not data:
        return jsonify({"error": "Request body must be JSON"}), 400

    username = data.get("username")
    password = data.get("password")

    if USERS.get(username) != password:
        return jsonify({"error": "Invalid credentials"}), 401

    # Створюємо JWT-токен
    payload = {
        "user": username,
        "exp": datetime.datetime.now(datetime.timezone.utc)
        + datetime.timedelta(hours=1),
    }
    token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")

    return jsonify({"token": token})


@app.route("/api/tasks")
def get_tasks():
    # Отримуємо токен із заголовка Authorization
    auth_header = request.headers.get("Authorization")

    if not auth_header or not auth_header.startswith("Bearer "):
        return jsonify({"error": "Missing or invalid Authorization header"}), 401

    token = auth_header.split(" ")[1]  # "Bearer eyJ..." → "eyJ..."

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        return jsonify({"error": "Token has expired"}), 401
    except jwt.InvalidTokenError:
        return jsonify({"error": "Invalid token"}), 401

    user = payload["user"]

    return jsonify({
        "user": user,
        "tasks": [
            {"id": 1, "title": "Learn Flask"},
            {"id": 2, "title": "Learn JWT"},
        ],
    })
```

### Тестування через curl

```bash
# Логін — отримуємо токен
curl -X POST http://127.0.0.1:5000/login \
  -H "Content-Type: application/json" \
  -d '{"username": "taras", "password": "secret123"}'

# Відповідь: {"token": "eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoi..."}

# Запит із токеном — працює
curl http://127.0.0.1:5000/api/tasks \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoi..."

# Відповідь: {"user": "taras", "tasks": [...]}

# Запит без токена — помилка
curl http://127.0.0.1:5000/api/tasks

# Відповідь: {"error": "Missing or invalid Authorization header"} (401)
```

### Ключові моменти коду

- **`jwt.encode(payload, key, algorithm)`** — створює підписаний токен із даних `payload`
- **`jwt.decode(token, key, algorithms)`** — перевіряє підпис і декодує дані. Якщо токен змінений або прострочений — викидає виняток
- **`"exp"`** — стандартне поле JWT, яке визначає час закінчення дії токена. Бібліотека `pyjwt` автоматично перевіряє його при `decode()`
- **`Authorization: Bearer <token>`** — стандартний формат заголовка для передачі токена ([RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750))

### Порівняння: сесії vs JWT

| Критерій | Сесії (cookie-based) | Сесії (server-side) | JWT (токени) |
|---|---|---|---|
| **Де зберігається стан** | Cookie в браузері | Сховище на сервері (Redis, БД) | Клієнт (localStorage, змінна, заголовок) |
| **Хто керує відправкою** | Браузер автоматично | Браузер автоматично (session ID) | Клієнт вручну додає заголовок |
| **Стан на сервері** | Ні | Так | Ні |
| **Вихід (logout)** | `session.clear()` — видаляє cookie | Видалення сесії зі сховища | Клієнт видаляє токен (сервер не може "відкликати" токен) |
| **Міжсервісна взаємодія** | Не підходить | Не підходить | Підходить — токен можна передати іншому сервісу |
| **Використання** | Веб-застосунки з HTML | Веб-застосунки з HTML | REST API, мобільні застосунки, SPA |

## Підсумок

| Операція | Код |
|---|---|
| Записати в сесію | `session["key"] = value` |
| Прочитати з сесії | `session.get("key", default)` |
| Видалити один ключ | `session.pop("key", None)` |
| Очистити всю сесію | `session.clear()` |
| Зробити сесію постійною | `session.permanent = True` |

Flask-сесії:

- Зберігаються у **cookie** на стороні клієнта
- **Підписані** секретним ключем (захист від підробки)
- **Не зашифровані** (не зберігайте секрети)
- Працюють як звичайний Python-**словник**
- Підходять для **веб-застосунків**, але не для REST API

## Корисні посилання

- [Flask — Sessions](https://flask.palletsprojects.com/quickstart/#sessions)
- [Flask — API: session](https://flask.palletsprojects.com/api/#flask.session)
- [MDN — HTTP Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [PyJWT — Documentation](https://pyjwt.readthedocs.io/en/stable/)
- [JWT.io — Debugger](https://jwt.io/) — інтерактивний інструмент для декодування та перевірки JWT-токенів

## Домашнє завдання

1. Повторити приклад із системою логіну з лекції. Протестувати через `curl` повний цикл: логін → доступ до захищеного ресурсу → вихід → спроба доступу після виходу.
2. Розширити приклад: додати endpoint `POST /api/cart/add`, який додає товар у "кошик" (`session["cart"]` — список). Додати endpoint `GET /api/cart`, який повертає вміст кошика. Перевірити, що кошик зберігається між запитами.
