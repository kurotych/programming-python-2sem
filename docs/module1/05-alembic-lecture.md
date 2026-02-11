# 5. (Л) Міграції у PostgreSQL з використанням Python та бібліотеки Alembic

## Зміст лекції

1. Що таке міграції бази даних?
2. Встановлення Alembic та SQLAlchemy
3. Ініціалізація Alembic у проєкті
4. Конфігурація Alembic
5. Створення міграцій
6. Застосування та відкат міграцій
7. Автогенерація міграцій на основі моделей
8. Найкращі практики

### Що таке міграції бази даних?

**Міграція бази даних** — це контрольований спосіб внесення змін до структури бази даних (схеми). Замість ручного виконання SQL-команд `ALTER TABLE`, `CREATE TABLE` тощо, міграції дозволяють:

- **Версіонувати** зміни схеми БД разом з кодом
- **Відстежувати** історію змін структури бази даних
- **Відкатувати** зміни у разі помилок
- **Синхронізувати** схему БД між різними середовищами (розробка, тестування, production)
- **Автоматизувати** розгортання змін

**Приклад проблеми без міграцій:**

```
Розробник 1: Додав колонку `phone` до таблиці `users`
Розробник 2: Не знає про зміну, його код падає
Production: Структура БД відрізняється від розробки (коду проєкту)
```

**З міграціями:**

```
Розробник 1: Створює міграцію "add_phone_to_users"
Розробник 2: Запускає міграції, отримує актуальну схему
Production: Міграції застосовуються автоматично при деплої (запуску сервіса/утиліти)
```

### Що таке Alembic?

**Alembic** — це інструмент для міграцій баз даних для Python. Він підтримує:

- Генерацію міграцій
- Застосування та відкат міграцій
- Розгалуження та злиття міграцій
- Підтримку різних баз даних (PostgreSQL, MySQL, SQLite тощо)

### Встановлення

Перед встановленням створіть та активуйте віртуальне середовище:

```bash
python3 -m venv env
source env/bin/activate
pip install alembic sqlalchemy psycopg2-binary
```

Перевірка встановлення:

```bash
alembic --version
```

### Ініціалізація Alembic у проєкті

```bash
# У корені проєкту виконайте:
alembic init alembic
```
Ця команда створить:

- `alembic.ini` — головний конфігураційний файл
- `alembic/` — директорію з:
  - `env.py` — скрипт середовища міграцій
  - `script.py.mako` — шаблон для нових міграцій
  - `versions/` — папку для файлів міграцій

Типова структура проєкту з Alembic:

```
my_project/
├── alembic/                  # Директорія Alembic
│   ├── versions/             # Файли міграцій
│   │   ├── 001_create_users.py
│   │   └── 002_add_phone.py
│   ├── env.py                # Конфігурація середовища
│   └── script.py.mako        # Шаблон для міграцій
├── alembic.ini               # Головний конфіг Alembic
├── models.py                 # SQLAlchemy моделі
└── main.py                   # Основний код програми
```

### Конфігурація Alembic

**1. Впевніться що у вас запущений інстанс postgresql**  
[Команда запуску PostgreSQL за допомогою docker](/ua/courses/programming-2sem/module1/04-psycopg2-practice/#postgresql-docker)

**2. Налаштування підключення до БД через змінні середовища:**

Відредагуйте `alembic/env.py`:


(Функцію run_migrations_offline можна видалити/ігнорувати)

```python
import os
from alembic import context
from sqlalchemy import engine_from_config, pool

# Отримання URL з змінної середовища
def get_url():
    return os.environ.get("DATABASE_URL", "postgresql://postgres:secret@localhost:5432/mydb")

def run_migrations_online():
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url() ## Ось тут добавили виклик get_url() який повертає connection string до бд

    connectable = engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    # ... решта коду
```

### Створення міграцій

```bash
# Створення нової міграції
alembic revision -m "create users table"
```

Ця команда створить файл у `alembic/versions/` з унікальним ідентифікатором:

```python
# alembic/versions/a1b2c3d4e5f6_create_users_table.py

"""create users table

Revision ID: a1b2c3d4e5f6
Revises:
Create Date: 2026-02-10 10:30:00.000000

"""
from alembic import op
import sqlalchemy as sa

# Ідентифікатори ревізій
revision = 'a1b2c3d4e5f6'
down_revision = None  # Попередня міграція (None для першої)
branch_labels = None
depends_on = None


def upgrade():
    """Застосування міграції (вперед)"""
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('name', sa.String(100), nullable=False),
        sa.Column('email', sa.String(255), unique=True),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now())
    )


def downgrade():
    """Відкат міграції (назад)"""
    op.drop_table('users')
```

**Основні операції Alembic:**

```python
from alembic import op
import sqlalchemy as sa

# Створення таблиці
op.create_table(
    'products',
    sa.Column('id', sa.Integer(), primary_key=True),
    sa.Column('name', sa.String(200), nullable=False),
    sa.Column('price', sa.Numeric(10, 2))
)

# Видалення таблиці
op.drop_table('products')

# Додавання колонки
op.add_column('users', sa.Column('phone', sa.String(20)))

# Видалення колонки
op.drop_column('users', 'phone')

# Перейменування колонки
op.alter_column('users', 'name', new_column_name='full_name')

# Зміна типу колонки
op.alter_column('users', 'phone',
    type_=sa.String(30),
    existing_type=sa.String(20))

# Створення індексу
op.create_index('ix_users_email', 'users', ['email'])

# Видалення індексу
op.drop_index('ix_users_email', 'users')

# Додавання зовнішнього ключа
op.create_foreign_key(
    'fk_orders_user_id',    # Назва constraint
    'orders',                # Таблиця з FK
    'users',                 # Таблиця, на яку посилається
    ['user_id'],             # Колонки FK
    ['id']                   # Колонки, на які посилається
)

# Видалення зовнішнього ключа
op.drop_constraint('fk_orders_user_id', 'orders', type_='foreignkey')
```

### Застосування та відкат міграцій

**Застосування міграцій:**

```bash
# Застосувати всі нові міграції
alembic upgrade head

# Застосувати до конкретної ревізії
alembic upgrade a1b2c3d4e5f6

# Застосувати одну міграцію вперед
alembic upgrade +1
```

**Відкат міграцій:**

```bash
# Відкатити одну міграцію
alembic downgrade -1

# Відкатити до конкретної ревізії
alembic downgrade a1b2c3d4e5f6

# Відкатити всі міграції (обережно!)
alembic downgrade base
```

**Перегляд стану:**

```bash
# Поточна версія бази даних
alembic current

# Історія міграцій
alembic history

# Детальна історія
alembic history --verbose
```

**3. Застосування міграції:**
```bash
❯ alembic upgrade head
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 192c4b9ae498, create users table
```

Alembic порівняє поточний стан БД з моделями та згенерує відповідну міграцію.


```bash
❯ alembic upgrade head
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
```

### Приклад повного робочого процесу

```bash
# 1. Ініціалізація Alembic
alembic init alembic

# 2. Налаштування alembic.ini та env.py

# 3. Перша міграція (автогенерація)
alembic revision -m "create users table"

# 4. Застосування міграції
alembic upgrade head
```

### Найкращі практики

**1. Називайте міграції описово:**

```bash
# Погано
alembic revision -m "update"

# Добре
alembic revision -m "add phone column to users table"
```

**2. По можливості добавляйте відкат транзакції (downgrade()):**

```python
def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20)))

def downgrade():
    op.drop_column('users', 'phone')  # Важливо для відкату!
```

**3. Не редагуйте застосовані міграції:**

Якщо міграція вже застосована (особливо на production), не змінюйте її. Створіть нову міграцію для виправлення.

**4. Тестуйте міграції:**

```bash
# Застосувати
alembic upgrade head

# Відкатити
alembic downgrade base

# Знову застосувати
alembic upgrade head
```

**5. Використовуйте транзакції:**

Alembic за замовчуванням виконує міграції у транзакції. Якщо щось пішло не так — зміни відкотяться автоматично.

**6. Зберігайте міграції у системі контролю версій (git):**

Файли міграцій — це частина коду проєкту. Вони повинні бути у Git разом з іншим кодом.

### Таблиця alembic_version

Alembic створює спеціальну таблицю `alembic_version` у базі даних для відстеження поточної версії схеми:

```sql
SELECT * FROM alembic_version;
-- Результат:
-- version_num
-- -----------
-- a1b2c3d4e5f6
```

Ця таблиця містить ідентифікатор останньої застосованої міграції.

## Корисні посилання

- [Протестований код до лекції з однією міграцією](https://github.com/kurotych/programming-2sem-code/tree/main/alembic-migration) 
- [Документація Alembic](https://alembic.sqlalchemy.org/)
- [Alembic Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

## Домашнє завдання

Прочитати документацію [Alembic Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html)
