# 15. (П7) Розгортання PostgreSQL у Docker-контейнері та наповнення бази тестовими даними

## Передумови

- Виконана [Практична 6 — Встановлення Docker. Базові команди](/ua/courses/programming-2sem/module1/13-docker-practice/)
- Прочитана [Лекція 14 — Керування контейнерами. Порти, volumes, змінні середовища](/ua/courses/programming-2sem/module1/14-docker-management-lecture/)

## Завдання

Розгорнути PostgreSQL у Docker-контейнері з використанням volumes, env-файлу та прокидання портів. Написати Python-скрипт, що створює схему бази даних та наповнює її тестовими даними. Переконатися, що дані зберігаються після перестворення контейнера.

### Крок 1. Підготовка проєкту

Створити директорію проєкту з такою структурою:

```
docker-postgres/
├── .env
├── .env.example
├── .gitignore
├── seed.py
└── requirements.txt
```

#### Файл `.env`

```bash
POSTGRES_USER=student
POSTGRES_PASSWORD=secret123
POSTGRES_DB=university_db
STUDENT_NAME=Тарас Шевченко
```

#### Файл `.env.example`

```bash
POSTGRES_USER=student
POSTGRES_PASSWORD=CHANGE_ME
POSTGRES_DB=university_db
STUDENT_NAME=Ваше Ім'я та Прізвище
```

#### Файл `.gitignore`

```
.env
__pycache__/
*.pyc
env/
venv/
```

#### Файл `requirements.txt`

```
psycopg2-binary
```

### Крок 2. Запуск PostgreSQL у Docker

- Створити іменований volume та запустити контейнер PostgreSQL з використанням `--env-file`:

- Перевірити, що контейнер працює

- Перевірити з'єднання. Приклад:

```bash
docker exec -it uni-postgres psql -U student -d university_db -c "SELECT version();"
```

### Крок 3. Написання Python-скрипта `seed.py`

Написати скрипт `seed.py`, який:

1. Підключається до PostgreSQL у контейнері
2. Створює таблиці
3. Наповнює їх тестовими даними
4. Виводить підсумкову інформацію

#### Схема бази даних

Створити три таблиці:

**`students`** — студенти:

| Колонка | Тип | Обмеження |
|---|---|---|
| id | SERIAL | PRIMARY KEY |
| first_name | VARCHAR(50) | NOT NULL |
| last_name | VARCHAR(50) | NOT NULL |
| email | VARCHAR(100) | UNIQUE, NOT NULL |
| enrollment_year | INTEGER | NOT NULL |

**`courses`** — курси:

| Колонка | Тип | Обмеження |
|---|---|---|
| id | SERIAL | PRIMARY KEY |
| name | VARCHAR(100) | NOT NULL |
| credits | INTEGER | NOT NULL |

**`enrollments`** — записи студентів на курси:

| Колонка | Тип | Обмеження |
|---|---|---|
| id | SERIAL | PRIMARY KEY |
| student_id | INTEGER | REFERENCES students(id) |
| course_id | INTEGER | REFERENCES courses(id) |
| grade | INTEGER | NULL (від 0 до 100) |

#### Вимоги до скрипта

- Параметри підключення та ім'я студента зчитувати з файлу `.env` (вручну або через модуль `os.environ`)
- Використовувати `host="localhost"`, `port=5432`
- Перед створенням таблиць — видаляти їх, якщо вони існують (`DROP TABLE IF EXISTS ... CASCADE`)
- **Першим студентом** у таблиці `students` має бути **ви** (ваше ім'я та прізвище зі змінної `STUDENT_NAME`)
- Додати щонайменше **5 студентів** (включно з вами), **4 курси** та **10 записів на курси** (enrollments)
- Дані мають бути реалістичними (українські імена, назви курсів)
- Використовувати **транзакцію** для вставки даних (один `commit` після всіх `INSERT`)
- Після наповнення — вивести персоналізований підсумок з вашим ім'ям

Приклад очікуваного виводу скрипта:

```
============================================
  Seed скрипт — Тарас Шевченко
============================================

Підключення до бази university_db...
Таблиці створено.
Додано 5 студентів.
Додано 4 курси.
Додано 10 записів на курси.
---
Підсумок (Тарас Шевченко):
  students:    5
  courses:     4
  enrollments: 10
Готово!
```

### Крок 4. Запуск скрипта та перевірка даних

Встановити залежності, завантажити змінні середовища та запустити скрипт. Приклад:

```bash
source env/bin/activate
pip install -r requirements.txt

# Завантажити змінні з .env у поточну сесію терміналу
export $(grep -v '^#' .env | xargs)

python seed.py
```

Скрипт має вивести ваше ім'я у заголовку та підсумку.

Перевірити дані через `docker exec`. Приклад:

```bash
# Переконатися, що ваш запис є в базі (замініть на ваше прізвище)
docker exec -it uni-postgres psql -U student -d university_db \
  -c "SELECT * FROM students WHERE id = 1;"

# Список курсів
docker exec -it uni-postgres psql -U student -d university_db \
  -c "SELECT * FROM courses;"

# Ваші курси (замініть на ваш student_id)
docker exec -it uni-postgres psql -U student -d university_db \
  -c "SELECT s.first_name, s.last_name, c.name AS course, e.grade
      FROM enrollments e
      JOIN students s ON e.student_id = s.id
      JOIN courses c ON e.course_id = c.id
      ORDER BY s.last_name;"
```

### Крок 5. Перевірка збереження даних (volumes)

Переконатися, що дані зберігаються після видалення контейнера. Приклад:

```bash
# 1. Видалити контейнер
docker rm -f uni-postgres

# 2. Переконатися, що контейнер видалено
docker ps -a | grep uni-postgres

# 3. Запустити НОВИЙ контейнер з тим самим volume
docker run -d \
  --name uni-postgres \
  --env-file .env \
  -v uni-pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:17

# 4. Перевірити, що дані на місці
docker exec -it uni-postgres psql -U student -d university_db \
  -c "SELECT count(*) FROM students;"
```

Якщо кількість студентів збігається — volume працює коректно.

### Крок 6. Діагностика контейнера

Виконати команди діагностики та зафіксувати результати. Приклад команд:

```bash
# Переглянути змінні середовища контейнера
docker exec uni-postgres env | grep POSTGRES

# Переглянути підключені volumes
docker inspect --format='{{json .Mounts}}' uni-postgres

# Перевірити прокинуті порти
docker inspect --format='{{json .NetworkSettings.Ports}}' uni-postgres

# Статистика використання ресурсів (вийти через Ctrl+C)
docker stats uni-postgres
```

## Результат

Після виконання роботи студент має:

- Практичний досвід розгортання PostgreSQL у Docker з volumes та env-файлом
- Працюючий Python-скрипт для створення схеми БД та наповнення тестовими даними
- Розуміння збереження даних через Docker volumes
- Досвід діагностики контейнерів (`docker inspect`, `docker stats`)
- Практичний досвід роботи з командами: `volume create`, `volume ls`, `volume rm`, `run` з прапорцями `-v`, `-p`, `--env-file`, `inspect`, `stats`
