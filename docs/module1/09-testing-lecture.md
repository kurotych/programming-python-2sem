# 9. (Л) Тестування програм, що працюють із базами даних (PostgreSQL) за допомогою pytest

## Зміст лекції

1. Навіщо тестувати код, що працює з базою даних
2. Нагадування: основи pytest
3. Фікстури pytest
4. Організація тестової бази даних
5. Фікстури для роботи з PostgreSQL
6. Тестування CRUD-операцій
7. Ізоляція тестів

## Навіщо тестувати код, що працює з базою даних

Код, що працює з базою даних, особливо вразливий до помилок:

- SQL-запити можуть містити синтаксичні помилки, які виявляються лише під час виконання
- Зміни схеми БД можуть зламати існуючі запити
- Транзакції можуть працювати некоректно — дані губляться або зберігаються частково
- Граничні випадки (порожня таблиця, дублікати, NULL-значення) часто залишаються непокритими

Автоматичні тести дозволяють:

- Виявляти помилки одразу після внесення змін
- Безпечно рефакторити код
- Документувати очікувану поведінку
- Запобігати регресіям — повторному появленню виправлених помилок

## Нагадування: основи pytest

Ви вже працювали з pytest у першому семестрі. Коротко нагадаємо ключові моменти.

### Підготовка середовища

pytest потрібно запускати у віртуальному середовищі (venv):

```bash
# Створення віртуального середовища
python3 -m venv env

# Активація (Linux/macOS)
source env/bin/activate

# Встановлення залежностей
pip install pytest psycopg2-binary
```

### Коротке нагадування

- Файли тестів: `test_*.py` або `*_test.py`
- Функції тестів починаються з `test_`
- Перевірки — через `assert`
- Перевірка винятків — через `pytest.raises`
- Запуск: `pytest` або `pytest -v` (детальний вивід)

```python
import pytest

def test_simple():
    assert 2 + 2 == 4

def test_exception():
    with pytest.raises(ValueError, match="invalid"):
        int("not_a_number")  # ValueError: invalid literal...
```

## Фікстури pytest

Фікстури — це механізм pytest для підготовки та очищення ресурсів, потрібних для тестів (з'єднання з БД, тестові дані тощо).

### Базовий приклад

```python
import pytest

@pytest.fixture
def sample_users():
    """Фікстура повертає список тестових користувачів"""
    return [
        {"name": "Alice", "email": "alice@example.com"},
        {"name": "Bob", "email": "bob@example.com"},
    ]

def test_user_count(sample_users):
    assert len(sample_users) == 2

def test_first_user(sample_users):
    assert sample_users[0]["name"] == "Alice"
```

Коли pytest бачить параметр `sample_users` у сигнатурі тесту, він автоматично викликає фікстуру з такою ж назвою та передає результат як аргумент.

### Фікстури з підготовкою та очищенням (yield)

```python
import pytest

@pytest.fixture
def temp_file():
    # Підготовка (setup)
    import tempfile, os
    path = tempfile.mktemp(suffix=".txt")
    with open(path, "w") as f:
        f.write("test data")

    yield path  # Тест отримає це значення

    # Очищення (teardown) — виконується після тесту
    os.remove(path)

def test_read_temp_file(temp_file):
    with open(temp_file) as f:
        assert f.read() == "test data"
```

Код **до** `yield` виконується перед тестом (setup), код **після** `yield` — після тесту (teardown). Це гарантує очищення ресурсів незалежно від результату тесту.

### Область дії фікстур (scope)

```python
import pytest

@pytest.fixture(scope="session")
def expensive_resource():
    """Створюється один раз для всієї тестової сесії"""
    print("Створення ресурсу...")
    resource = {"connection": "established"}
    yield resource
    print("Звільнення ресурсу...")

@pytest.fixture(scope="function")  # За замовчуванням
def per_test_data():
    """Створюється заново для кожного тесту"""
    return {"counter": 0}
```

| Scope | Опис |
|---|---|
| `function` | Нова фікстура для кожного тесту (за замовчуванням) |
| `class` | Одна фікстура на клас тестів |
| `module` | Одна фікстура на файл тестів |
| `session` | Одна фікстура на всю тестову сесію |

## Організація тестової бази даних

**Важливо:** тести ніколи не повинні працювати з продакшн-базою даних. Для тестів завжди використовують окрему базу.

### Стратегії

1. **Окрема тестова база даних** — створюється перед запуском тестів, видаляється після. Тести працюють з реальним PostgreSQL.
2. **Мокування (mocking)** — підміна реального з'єднання імітацією. Тести не потребують запущеної БД, але менш реалістичні.

У цій лекції ми розглянемо перший підхід.

## Фікстури для роботи з PostgreSQL

### Фікстура з'єднання з тестовою базою

```python
import pytest
import psycopg2

TEST_DB_URL = "postgresql://postgres:secret@localhost:5432/mydb"

@pytest.fixture(scope="session")
def db_connection():
    """Створення з'єднання з тестовою базою на всю сесію тестів"""
    conn = psycopg2.connect(TEST_DB_URL)
    conn.autocommit = True  # Кожен запит виконується автономно
    yield conn
    conn.close()

@pytest.fixture(scope="session", autouse=True)
def create_tables(db_connection):
    """Створення таблиць перед запуском тестів"""
    with db_connection.cursor() as cursor:
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                email VARCHAR(255) UNIQUE NOT NULL
            )
        ''')
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS accounts (
                id SERIAL PRIMARY KEY,
                owner VARCHAR(100) NOT NULL,
                balance NUMERIC(12, 2) NOT NULL DEFAULT 0,
                CHECK (balance >= 0)
            )
        ''')
```

Параметр `autouse=True` означає, що фікстура застосовується автоматично до всіх тестів без потреби явно вказувати її в аргументах.

`autocommit = True` — кожен SQL-запит виконується та фіксується автоматично, без явного виклику `commit()`. Це спрощує тести: після помилки (наприклад, `IntegrityError`) з'єднання залишається робочим, бо немає «зламаної» транзакції.

### Повний файл conftest.py

pytest автоматично завантажує фікстури з файлу `conftest.py`. Розміщуйте спільні фікстури в цьому файлі, щоб вони були доступні всім тестам:

```python
# conftest.py
import pytest
import psycopg2

TEST_DB_URL = "postgresql://postgres:secret@localhost:5432/mydb"

@pytest.fixture(scope="session")
def db_connection():
    """З'єднання з тестовою базою"""
    conn = psycopg2.connect(TEST_DB_URL)
    conn.autocommit = True
    yield conn
    conn.close()

@pytest.fixture(scope="session", autouse=True)
def create_tables(db_connection):
    """Створення таблиць"""
    with db_connection.cursor() as cur:
        cur.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                email VARCHAR(255) UNIQUE NOT NULL
            )
        ''')
        cur.execute('''
            CREATE TABLE IF NOT EXISTS accounts (
                id SERIAL PRIMARY KEY,
                owner VARCHAR(100) NOT NULL,
                balance NUMERIC(12, 2) NOT NULL DEFAULT 0,
                CHECK (balance >= 0)
            )
        ''')

@pytest.fixture(autouse=True)
def clean_tables(db_connection):
    """Очищення таблиць перед кожним тестом"""
    with db_connection.cursor() as cur:
        cur.execute("DELETE FROM users")
        cur.execute("DELETE FROM accounts")
```

Фікстура `clean_tables` з `autouse=True` автоматично очищає всі таблиці після кожного тесту. Це гарантує, що дані, створені одним тестом, не впливають на інші. Завдяки `autocommit = True` фікстура працює надійно — навіть якщо тест спровокував помилку БД (наприклад, `IntegrityError`), з'єднання залишається робочим.

## Тестування CRUD-операцій

### Модуль, який тестуємо

```python
# user_repository.py
import psycopg2

class UserRepository:
    def __init__(self, conn):
        self.conn = conn

    def create(self, name, email):
        """Створити нового користувача"""
        with self.conn.cursor() as cur:
            cur.execute(
                "INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id",
                (name, email)
            )
            return cur.fetchone()[0]

    def get_by_id(self, user_id):
        """Отримати користувача за ID"""
        with self.conn.cursor() as cur:
            cur.execute("SELECT id, name, email FROM users WHERE id = %s", (user_id,))
            row = cur.fetchone()
            if row is None:
                return None
            return {"id": row[0], "name": row[1], "email": row[2]}

    def get_all(self):
        """Отримати всіх користувачів"""
        with self.conn.cursor() as cur:
            cur.execute("SELECT id, name, email FROM users ORDER BY id")
            return [{"id": r[0], "name": r[1], "email": r[2]} for r in cur.fetchall()]

    def update_email(self, user_id, new_email):
        """Оновити email користувача"""
        with self.conn.cursor() as cur:
            cur.execute(
                "UPDATE users SET email = %s WHERE id = %s",
                (new_email, user_id)
            )
            return cur.rowcount > 0

    def delete(self, user_id):
        """Видалити користувача"""
        with self.conn.cursor() as cur:
            cur.execute("DELETE FROM users WHERE id = %s", (user_id,))
            return cur.rowcount > 0
```

### Тести для CRUD-операцій

```python
# test_user_repository.py
import pytest
import psycopg2
from user_repository import UserRepository

@pytest.fixture
def repo(db_connection):
    """Фікстура: екземпляр UserRepository"""
    return UserRepository(db_connection)

# --- CREATE ---

def test_create_user(repo):
    user_id = repo.create("Alice", "alice@example.com")
    assert user_id is not None
    assert isinstance(user_id, int)

def test_create_user_duplicate_email(repo, db_connection):
    repo.create("Alice", "alice@example.com")
    with pytest.raises(psycopg2.IntegrityError):
        repo.create("Bob", "alice@example.com")

# --- READ ---

def test_get_by_id(repo):
    user_id = repo.create("Bob", "bob@example.com")
    user = repo.get_by_id(user_id)

    assert user is not None
    assert user["name"] == "Bob"
    assert user["email"] == "bob@example.com"

def test_get_by_id_not_found(repo):
    user = repo.get_by_id(99999)
    assert user is None

def test_get_all(repo):
    repo.create("Alice", "alice@example.com")
    repo.create("Bob", "bob@example.com")

    users = repo.get_all()
    assert len(users) == 2
    assert users[0]["name"] == "Alice"
    assert users[1]["name"] == "Bob"

# --- UPDATE ---

def test_update_email(repo):
    user_id = repo.create("Charlie", "charlie@example.com")
    result = repo.update_email(user_id, "new_charlie@example.com")

    assert result is True
    user = repo.get_by_id(user_id)
    assert user["email"] == "new_charlie@example.com"

def test_update_nonexistent_user(repo):
    result = repo.update_email(99999, "no@example.com")
    assert result is False

# --- DELETE ---

def test_delete_user(repo):
    user_id = repo.create("Dave", "dave@example.com")
    result = repo.delete(user_id)

    assert result is True
    assert repo.get_by_id(user_id) is None

def test_delete_nonexistent_user(repo):
    result = repo.delete(99999)
    assert result is False
```


#### Приклад виконання тестів

```
❯ pytest  -v
===================================================================================== test session starts ======================================================================================
platform linux -- Python 3.13.5, pytest-9.0.2, pluggy-1.6.0 -- /home/kurotych/sources/python-postgre/env/bin/python3
cachedir: .pytest_cache
rootdir: /home/kurotych/sources/python-postgre
collected 9 items                                                                                                                                                                              

test_user_repository.py::test_create_user PASSED                                                                                                                                         [ 11%]
test_user_repository.py::test_create_user_duplicate_email PASSED                                                                                                                         [ 22%]
test_user_repository.py::test_get_by_id PASSED                                                                                                                                           [ 33%]
test_user_repository.py::test_get_by_id_not_found PASSED                                                                                                                                 [ 44%]
test_user_repository.py::test_get_all PASSED                                                                                                                                             [ 55%]
test_user_repository.py::test_update_email PASSED                                                                                                                                        [ 66%]
test_user_repository.py::test_update_nonexistent_user PASSED                                                                                                                             [ 77%]
test_user_repository.py::test_delete_user PASSED                                                                                                                                         [ 88%]
test_user_repository.py::test_delete_nonexistent_user PASSED                                                                                                                             [100%]
```


Зверніть увагу: завдяки фікстурі `clean_tables` кожен тест починає з порожньої таблиці `users`. Тести не впливають один на одного.

## Ізоляція тестів

Ізоляція тестів — критично важлива концепція. Кожен тест повинен працювати незалежно від інших: порядок запуску не повинен впливати на результат.

З'єднання з `autocommit = True` та видалення даних після кожного тесту:

```python
@pytest.fixture(scope="session")
def db_connection():
    conn = psycopg2.connect(TEST_DB_URL)
    conn.autocommit = True
    yield conn
    conn.close()

@pytest.fixture(autouse=True)
def clean_tables(db_connection):
    """Очищення таблиць перед кожним тестом"""
    with db_connection.cursor() as cur:
        cur.execute("DELETE FROM users")
        cur.execute("DELETE FROM accounts")
```

Перед кожним тестом фікстура видаляє всі рядки з таблиць, гарантуючи чистий стан. Завдяки `autocommit` немає проблеми зі «зламаними» транзакціями після помилок БД.


### Приклад тестування класів.

```bash
# Створення та активація venv
python3 -m venv env
source env/bin/activate

# Встановлення залежностей
pip install pytest psycopg2-binary

# Також потрібно створити базу даних mydb (якщо ще не створена)
```

### account_service.py

```python
import psycopg2

class InsufficientFundsError(Exception):
    pass

class AccountNotFoundError(Exception):
    pass

class AccountService:
    def __init__(self, conn):
        self.conn = conn

    def create_account(self, owner, balance=0):
        with self.conn.cursor() as cur:
            cur.execute(
                "INSERT INTO accounts (owner, balance) VALUES (%s, %s) RETURNING id",
                (owner, balance)
            )
            return cur.fetchone()[0]

    def get_balance(self, account_id):
        with self.conn.cursor() as cur:
            cur.execute("SELECT balance FROM accounts WHERE id = %s", (account_id,))
            row = cur.fetchone()
            if row is None:
                raise AccountNotFoundError(f"Рахунок {account_id} не знайдено")
            return float(row[0])

    def transfer(self, from_id, to_id, amount):
        if amount <= 0:
            raise ValueError("Сума переказу повинна бути додатною")

        balance = self.get_balance(from_id)
        self.get_balance(to_id)  # Перевірка існування отримувача

        if balance < amount:
            raise InsufficientFundsError(f"Недостатньо коштів: {balance} < {amount}")

        with self.conn.cursor() as cur:
            cur.execute(
                "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                (amount, from_id)
            )
            cur.execute(
                "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                (amount, to_id)
            )
```

### conftest.py

```python
import pytest
import psycopg2

TEST_DB_URL = "postgresql://postgres:secret@localhost:5432/mydb"

@pytest.fixture(scope="session")
def db_connection():
    conn = psycopg2.connect(TEST_DB_URL)
    conn.autocommit = True
    yield conn
    conn.close()

@pytest.fixture(scope="session", autouse=True)
def create_tables(db_connection):
    with db_connection.cursor() as cur:
        cur.execute('''
            CREATE TABLE IF NOT EXISTS accounts (
                id SERIAL PRIMARY KEY,
                owner VARCHAR(100) NOT NULL,
                balance NUMERIC(12, 2) NOT NULL DEFAULT 0,
                CHECK (balance >= 0)
            )
        ''')

@pytest.fixture(autouse=True)
def clean_tables(db_connection):
    """Очищення таблиць перед кожним тестом"""
    with db_connection.cursor() as cur:
        cur.execute("DELETE FROM accounts")
```

### test_account_service.py

```python
import pytest
from account_service import AccountService, InsufficientFundsError, AccountNotFoundError

@pytest.fixture
def service(db_connection):
    return AccountService(db_connection)

@pytest.fixture
def funded_accounts(service):
    """Два рахунки: Олена (1000 грн), Тарас (500 грн)"""
    id1 = service.create_account("Олена", 1000)
    id2 = service.create_account("Тарас", 500)
    return id1, id2

class TestCreateAccount:
    def test_create_with_default_balance(self, service):
        acc_id = service.create_account("Нова людина")
        assert service.get_balance(acc_id) == 0.0

    def test_create_with_initial_balance(self, service):
        acc_id = service.create_account("Багатій", 10000)
        assert service.get_balance(acc_id) == 10000.0

class TestGetBalance:
    def test_existing_account(self, service, funded_accounts):
        id1, _ = funded_accounts
        assert service.get_balance(id1) == 1000.0

    def test_nonexistent_account(self, service):
        with pytest.raises(AccountNotFoundError):
            service.get_balance(99999)

class TestTransfer:
    def test_successful_transfer(self, service, funded_accounts):
        id1, id2 = funded_accounts
        service.transfer(id1, id2, 300)

        assert service.get_balance(id1) == 700.0
        assert service.get_balance(id2) == 800.0

    def test_transfer_all_money(self, service, funded_accounts):
        id1, id2 = funded_accounts
        service.transfer(id1, id2, 1000)

        assert service.get_balance(id1) == 0.0
        assert service.get_balance(id2) == 1500.0

    def test_insufficient_funds(self, service, funded_accounts):
        id1, id2 = funded_accounts

        with pytest.raises(InsufficientFundsError):
            service.transfer(id1, id2, 2000)

        # Баланси залишились незмінними
        assert service.get_balance(id1) == 1000.0
        assert service.get_balance(id2) == 500.0

    def test_negative_amount(self, service, funded_accounts):
        id1, id2 = funded_accounts
        with pytest.raises(ValueError, match="додатною"):
            service.transfer(id1, id2, -100)

    def test_zero_amount(self, service, funded_accounts):
        id1, id2 = funded_accounts
        with pytest.raises(ValueError):
            service.transfer(id1, id2, 0)

    def test_nonexistent_sender(self, service, funded_accounts):
        _, id2 = funded_accounts
        with pytest.raises(AccountNotFoundError):
            service.transfer(99999, id2, 100)

    def test_nonexistent_receiver(self, service, funded_accounts):
        id1, _ = funded_accounts
        with pytest.raises(AccountNotFoundError):
            service.transfer(id1, 99999, 100)

    def test_multiple_transfers(self, service, funded_accounts):
        id1, id2 = funded_accounts

        service.transfer(id1, id2, 100)  # Олена: 900, Тарас: 600
        service.transfer(id2, id1, 50)   # Олена: 950, Тарас: 550

        assert service.get_balance(id1) == 950.0
        assert service.get_balance(id2) == 550.0
```

### Запуск тестів

```bash
#~/sources/pp2 via 🐍 v3.13.5 (env) took 8s
❯ pytest -v
=============================================================================================== test session starts ================================================================================================
platform linux -- Python 3.13.5, pytest-9.0.2, pluggy-1.6.0 -- /home/kurotych/sources/pp2/env/bin/python3
cachedir: .pytest_cache
rootdir: /home/kurotych/sources/pp2
collected 12 items

test_account_service.py::TestCreateAccount::test_create_with_default_balance PASSED                                                                                                                          [  8%]
test_account_service.py::TestCreateAccount::test_create_with_initial_balance PASSED                                                                                                                          [ 16%]
test_account_service.py::TestGetBalance::test_existing_account PASSED                                                                                                                                        [ 25%]
test_account_service.py::TestGetBalance::test_nonexistent_account PASSED                                                                                                                                     [ 33%]
test_account_service.py::TestTransfer::test_successful_transfer PASSED                                                                                                                                       [ 41%]
test_account_service.py::TestTransfer::test_transfer_all_money PASSED                                                                                                                                        [ 50%]
test_account_service.py::TestTransfer::test_insufficient_funds PASSED                                                                                                                                        [ 58%]
test_account_service.py::TestTransfer::test_negative_amount PASSED                                                                                                                                           [ 66%]
test_account_service.py::TestTransfer::test_zero_amount PASSED                                                                                                                                               [ 75%]
test_account_service.py::TestTransfer::test_nonexistent_sender PASSED                                                                                                                                        [ 83%]
test_account_service.py::TestTransfer::test_nonexistent_receiver PASSED                                                                                                                                      [ 91%]
test_account_service.py::TestTransfer::test_multiple_transfers PASSED                                                                                                                                        [100%]

================================================================================================ 12 passed in 0.07s ================================================================================================ Активуйте venv, якщо ще не активоване
source env/bin/activate

# Запуск усіх тестів
pytest -v

# Очікуваний результат:
# test_account_service.py::TestCreateAccount::test_create_with_default_balance PASSED
# test_account_service.py::TestCreateAccount::test_create_with_initial_balance PASSED
# test_account_service.py::TestGetBalance::test_existing_account PASSED
# test_account_service.py::TestGetBalance::test_nonexistent_account PASSED
# test_account_service.py::TestTransfer::test_successful_transfer PASSED
# test_account_service.py::TestTransfer::test_transfer_all_money PASSED
# test_account_service.py::TestTransfer::test_insufficient_funds PASSED
# test_account_service.py::TestTransfer::test_negative_amount PASSED
# test_account_service.py::TestTransfer::test_zero_amount PASSED
# test_account_service.py::TestTransfer::test_nonexistent_sender PASSED
# test_account_service.py::TestTransfer::test_nonexistent_receiver PASSED
# test_account_service.py::TestTransfer::test_multiple_transfers PASSED
# ========================= 12 passed =========================
```

## Корисні посилання

- [Документація pytest](https://docs.pytest.org/)
- [pytest: Fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
- [unittest.mock](https://docs.python.org/3/library/unittest.mock.html)
- [Документація psycopg2](https://www.psycopg.org/docs/)

## Домашнє завдання

Прочитати документацію [pytest: Fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
