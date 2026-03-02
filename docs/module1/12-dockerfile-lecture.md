# 12. (Л) Dockerfile. Інструкції (FROM, COPY, RUN, CMD), шари, збірка образів

## Зміст лекції

1. Що таке Dockerfile?
2. Основні інструкції: FROM, COPY, RUN, CMD
3. Додаткові інструкції: WORKDIR, EXPOSE, ENV, ENTRYPOINT, ARG
4. Як працюють шари
5. Оптимізація шарів та кешування
6. Збірка образу: docker build
7. Файл .dockerignore
8. Практичний приклад: контейнеризація Python-застосунку

## Що таке Dockerfile?

У попередній лекції ми використовували готові образи з Docker Hub (наприклад, `postgres:17`). Але як створити **власний** образ для свого застосунку?

**Dockerfile** — це текстовий файл з інструкціями для побудови Docker-образу. Кожна інструкція описує один крок: яку базу взяти, що встановити, які файли скопіювати, яку команду виконати при запуску.

### Як називати файл

За домовленістю файл називається просто **`Dockerfile`** — без розширення, з великої літери `D`. Він розміщується в кореневій директорії проєкту:

```
my-project/
├── Dockerfile          ← саме так
├── app.py
├── requirements.txt
└── ...
```

Docker за замовчуванням шукає файл із саме такою назвою. Якщо потрібно мати кілька Dockerfile (наприклад, для різних середовищ), використовують суфікси: `Dockerfile.dev`, `Dockerfile.prod`, `Dockerfile.test`. У такому випадку при збірці потрібно явно вказати файл через прапорець `-f`:

```bash
docker build -f Dockerfile.dev -t myapp:dev .
```

```
Dockerfile  →  docker build  →  Docker-образ  →  docker run  →  Контейнер
(рецепт)       (збірка)         (шаблон)          (запуск)       (працюючий екземпляр)
```

## Основні інструкції

### FROM — базовий образ

Кожен Dockerfile **починається** з інструкції `FROM`. Вона визначає базовий образ, на основі якого буде побудовано ваш образ.

```dockerfile
# Повноцінний Python-образ (~1 ГБ)
FROM python:3.13

# Полегшений Python-образ (~130 МБ) — рекомендовано
FROM python:3.13-slim

# Мінімальний Python-образ (~50 МБ)
FROM python:3.13-alpine
```

**Який обрати?**

| Образ | Розмір | Коли використовувати |
|---|---|---|
| `python:3.13` | ~1 ГБ | Потрібні системні бібліотеки для компіляції (numpy, psycopg2) |
| `python:3.13-slim` | ~130 МБ | Більшість проєктів — оптимальний баланс |
| `python:3.13-alpine` | ~50 МБ | Мінімальний розмір, але можуть бути проблеми зі сумісністю |

Для наших проєктів з `psycopg2-binary` рекомендується `python:3.13-slim`.

### RUN — виконання команд під час збірки

Інструкція `RUN` виконує команду всередині образу **під час збірки**. Результат зберігається як новий шар образу.

```dockerfile
# Встановити системний пакет
RUN apt-get update && apt-get install -y curl

# Встановити Python-залежності
RUN pip install flask psycopg2-binary

# Створити директорію
RUN mkdir -p /app/data
```

**Важливо:** кожен `RUN` створює окремий шар. Тому часто об'єднують команди через `&&`:

```dockerfile
# Погано — 3 шари
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Добре — 1 шар
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

Символ `\` дозволяє перенести довгу команду на наступний рядок.

### COPY — копіювання файлів

Інструкція `COPY` копіює файли або директорії з вашого комп'ютера (контексту збірки) в образ.

```dockerfile
# Скопіювати один файл
COPY requirements.txt /app/requirements.txt

# Скопіювати файл у робочу директорію (якщо встановлено WORKDIR)
COPY requirements.txt .

# Скопіювати всю директорію
COPY . /app/

# Скопіювати кілька файлів
COPY app.py config.py /app/
```

**Контекст збірки** — це директорія, яку ви вказуєте при виклику `docker build`. Зазвичай це коренева директорія проєкту. `COPY` може копіювати **лише** файли з контексту збірки — вийти за його межі неможливо.

### CMD — команда запуску контейнера

Інструкція `CMD` визначає команду, яка виконується **при запуску контейнера** (не під час збірки!).

```dockerfile
# Запустити Python-скрипт
CMD ["python", "app.py"]

# Запустити з аргументами
CMD ["python", "-m", "flask", "run", "--host=0.0.0.0"]

# Запустити bash (для налагодження)
CMD ["bash"]
```

**Важливі правила:**

- У Dockerfile може бути **лише один** `CMD`. Якщо їх кілька — працює останній.
- `CMD` можна перевизначити при запуску контейнера:

```bash
# Замість CMD з Dockerfile виконає bash
docker run myapp bash
```

### Перший повний Dockerfile

Зберемо все разом:

```dockerfile
# Базовий образ
FROM python:3.13-slim

# Встановити залежності
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Скопіювати код застосунку
COPY app.py .

# Команда запуску
CMD ["python", "app.py"]
```

## Додаткові інструкції

### WORKDIR — робоча директорія

Встановлює робочу директорію для всіх наступних інструкцій (`RUN`, `COPY`, `CMD` тощо). Якщо директорія не існує — вона буде створена автоматично.

```dockerfile
FROM python:3.13-slim

# Всі наступні команди виконуються в /app
WORKDIR /app

# Це еквівалентно COPY requirements.txt /app/requirements.txt
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

**Рекомендація:** завжди використовуйте `WORKDIR` замість `RUN mkdir ... && cd ...`. Це робить Dockerfile чистішим та зрозумілішим.

### EXPOSE — документація портів

`EXPOSE` повідомляє, на якому порті працює застосунок усередині контейнера. Це лише **документація** — порт не стає доступним автоматично. Для реального прокидання порту потрібен прапорець `-p` при `docker run`.

```dockerfile
# Застосунок слухає порт 5000
EXPOSE 5000
```

```bash
# Прокинути порт при запуску
docker run -p 5000:5000 myapp
```

### ENV — змінні середовища

Встановлює змінні середовища, які доступні як **під час збірки**, так і **під час роботи** контейнера.

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
```

- `PYTHONDONTWRITEBYTECODE=1` — Python не створює `.pyc` файли (не потрібні в контейнері)
- `PYTHONUNBUFFERED=1` — вивід Python не буферизується (логи видно одразу в `docker logs`)

### ENTRYPOINT — фіксована команда

`ENTRYPOINT` схожий на `CMD`, але його **не можна** перевизначити через аргументи `docker run`. Часто `ENTRYPOINT` і `CMD` використовують разом:

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

```bash
# Запустить: python app.py (CMD за замовчуванням)
docker run myapp

# Запустить: python test.py (CMD перевизначено)
docker run myapp test.py
```

### ARG — аргументи збірки

`ARG` визначає змінні, доступні **лише під час збірки** (на відміну від `ENV`, який доступний і в контейнері).

```dockerfile
ARG PYTHON_VERSION=3.13
FROM python:${PYTHON_VERSION}-slim
```

```bash
# Збірка з іншою версією Python
docker build --build-arg PYTHON_VERSION=3.12 .
```

## Як працюють шари

Кожна інструкція в Dockerfile створює новий **шар** (layer) образу. Шари зберігаються окремо та є незмінними (read-only).

### Візуалізація шарів

```dockerfile
FROM python:3.13-slim         # Шар 1: базовий образ (~130 МБ)
WORKDIR /app                  # Шар 2: створення директорії (~0 Б)
COPY requirements.txt .       # Шар 3: файл залежностей (~1 КБ)
RUN pip install -r req...     # Шар 4: встановлені пакети (~50 МБ)
COPY . .                      # Шар 5: код застосунку (~10 КБ)
CMD ["python", "app.py"]      # Шар 6: метадані (~0 Б)
```

```
┌──────────────────────────────┐
│  Шар 6: CMD (метадані)       │  ← read-only
├──────────────────────────────┤
│  Шар 5: COPY . .             │  ← read-only
├──────────────────────────────┤
│  Шар 4: RUN pip install      │  ← read-only (найбільший)
├──────────────────────────────┤
│  Шар 3: COPY requirements    │  ← read-only
├──────────────────────────────┤
│  Шар 2: WORKDIR /app         │  ← read-only
├──────────────────────────────┤
│  Шар 1: python:3.13-slim     │  ← read-only (базовий)
└──────────────────────────────┘
```

При запуску контейнера Docker додає **тонкий записуваний шар** (writable layer) поверх усіх шарів образу. Усі зміни в контейнері (створення файлів, зміна даних) потрапляють у цей шар. При видаленні контейнера записуваний шар знищується.

### Кешування шарів

Docker кешує кожен шар. При повторній збірці він перевіряє: чи змінилися вхідні дані для цього шару? Якщо ні — використовує кеш.

**Як це працює:**

```
Збірка 1 (перша, повна):
  Шар 1: FROM python:3.13-slim     → завантажити
  Шар 2: COPY requirements.txt .   → скопіювати
  Шар 3: RUN pip install ...       → виконати (довго!)
  Шар 4: COPY . .                  → скопіювати
  Шар 5: CMD                       → записати

Збірка 2 (змінився лише код app.py):
  Шар 1: FROM python:3.13-slim     → кеш ✓
  Шар 2: COPY requirements.txt .   → кеш ✓ (файл не змінився)
  Шар 3: RUN pip install ...       → кеш ✓ (requirements не змінились)
  Шар 4: COPY . .                  → перебудувати (код змінився)
  Шар 5: CMD                       → перебудувати
```

**Важливе правило:** якщо шар змінився — всі наступні шари також перебудовуються (кеш інвалідується).

## Оптимізація шарів та кешування

### Правило: спочатку залежності, потім код

Це найважливіший прийом оптимізації Dockerfile:

```dockerfile
# ПРАВИЛЬНО: залежності окремо від коду
FROM python:3.13-slim
WORKDIR /app

# Спочатку копіюємо лише файл залежностей
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Потім копіюємо код (змінюється частіше)
COPY . .

CMD ["python", "app.py"]
```

```dockerfile
# НЕПРАВИЛЬНО: копіюємо все разом
FROM python:3.13-slim
WORKDIR /app

# Будь-яка зміна коду інвалідує кеш pip install
COPY . .
RUN pip install --no-cache-dir -r requirements.txt

CMD ["python", "app.py"]
```

У правильному варіанті при зміні коду (`app.py`) `pip install` **не виконується повторно**, бо `requirements.txt` не змінився. Це може заощадити хвилини при кожній збірці.

### Прапорець --no-cache-dir

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

`--no-cache-dir` вказує pip не зберігати завантажені пакети в кеші. Це зменшує розмір образу, бо кеш pip не потрібен усередині контейнера — пакети вже встановлені.

### Мінімізація кількості шарів

Об'єднуйте пов'язані команди в один `RUN`:

```dockerfile
# Один шар замість трьох
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*
```

`--no-install-recommends` — не встановлювати рекомендовані (необов'язкові) пакети. `rm -rf /var/lib/apt/lists/*` — видалити кеш apt для зменшення розміру шару.

## Збірка образу: docker build

### Базовий синтаксис

```bash
# Збірка з поточної директорії (де лежить Dockerfile)
docker build .

# Задати ім'я та тег образу (-t)
docker build -t myapp:1.0 .

# Задати лише ім'я (тег буде latest)
docker build -t myapp .
```

Крапка `.` в кінці — це **контекст збірки** (директорія, файли з якої доступні для `COPY`).

### Процес збірки

```bash
$ docker build -t myapp:1.0 .

[+] Building 25.3s (10/10) FINISHED
 => [1/5] FROM python:3.13-slim@sha256:abc...       0.0s (cached)
 => [2/5] WORKDIR /app                               0.1s
 => [3/5] COPY requirements.txt .                    0.1s
 => [4/5] RUN pip install --no-cache-dir -r req...  22.8s
 => [5/5] COPY . .                                   0.1s
 => exporting to image                               2.1s
```

### Перевірка результату

```bash
# Показати створений образ
docker images myapp

REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
myapp        1.0       d4e5f6a7b8c9   10 seconds ago   185MB

# Запустити контейнер з нового образу
docker run myapp:1.0
```

### Вказати інший Dockerfile

```bash
# Якщо Dockerfile має нестандартне ім'я
docker build -f Dockerfile.dev -t myapp:dev .
```

## Файл .dockerignore

Аналог `.gitignore` для Docker. Визначає файли та директорії, які **не потрапляють** у контекст збірки (і, відповідно, не копіюються через `COPY . .`).

```
# .dockerignore

# Віртуальне середовище Python
env/
venv/
.venv/

# Git
.git/
.gitignore

# Кеш Python
__pycache__/
*.pyc

# IDE
.vscode/
.idea/

# Docker
Dockerfile
docker-compose.yml

# Інше
*.md
.env
```

**Навіщо потрібен .dockerignore:**

1. **Безпека** — не копіювати `.env` з секретами в образ
2. **Розмір** — не включати `env/`, `.git/` та інші великі директорії
3. **Швидкість** — менший контекст збірки = швидша збірка
4. **Кешування** — зміни в ігнорованих файлах не інвалідують кеш `COPY . .`

## Практичний приклад: контейнеризація Python-застосунку

Створимо простий Python-застосунок та запакуємо його в Docker-образ.

### Структура проєкту

```
my-app/
├── app.py
├── requirements.txt
├── Dockerfile
└── .dockerignore
```

### Файл app.py

```python
import psycopg2
import os
import time


def main():
    db_host = os.environ.get("DB_HOST", "localhost")
    db_port = os.environ.get("DB_PORT", "5432")
    db_name = os.environ.get("DB_NAME", "mydb")
    db_user = os.environ.get("DB_USER", "postgres")
    db_password = os.environ.get("DB_PASSWORD", "secret")

    print(f"Підключення до {db_host}:{db_port}/{db_name}...")

    conn = psycopg2.connect(
        host=db_host,
        port=db_port,
        dbname=db_name,
        user=db_user,
        password=db_password,
    )

    cur = conn.cursor()
    cur.execute("SELECT version();")
    version = cur.fetchone()[0]
    print(f"PostgreSQL версія: {version}")

    cur.close()
    conn.close()
    print("Готово!")


if __name__ == "__main__":
    main()
```

### Файл requirements.txt

```
psycopg2-binary==2.9.10
```

### Dockerfile

```dockerfile
# Базовий образ
FROM python:3.13-slim

# Змінні для оптимізації Python в контейнері
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Робоча директорія
WORKDIR /app

# Спочатку копіюємо залежності (для кешування)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Потім копіюємо код
COPY . .

# Команда запуску
CMD ["python", "app.py"]
```

### Файл .dockerignore

```
env/
venv/
.git/
__pycache__/
*.pyc
.env
Dockerfile
.dockerignore
```

### Збірка та запуск

```bash
# 1. Збираємо образ
docker build -t my-python-app .

# 2. Запускаємо PostgreSQL (з лекції 11)
docker run -d \
  --name postgres-dev \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  postgres:17

# 3. Запускаємо наш застосунок
#    --rm — видалити контейнер після завершення
#    --network host — використовувати мережу хоста (щоб бачити localhost:5432)
docker run --rm \
  --network host \
  -e DB_HOST=localhost \
  -e DB_PASSWORD=secret \
  my-python-app
```

```
Підключення до localhost:5432/mydb...
PostgreSQL версія: PostgreSQL 17.4 (Debian 17.4-1.pgdg120+2) ...
Готово!
```

### Перегляд шарів образу

```bash
# Переглянути історію шарів образу
docker history my-python-app
```

```
IMAGE          CREATED         CREATED BY                                      SIZE
d4e5f6a7b8c9   2 minutes ago   CMD ["python" "app.py"]                         0B
a3b4c5d6e7f8   2 minutes ago   COPY . . # buildkit                             1.2kB
f1e2d3c4b5a6   2 minutes ago   RUN pip install --no-cache-dir -r requireme…    12.5MB
e7f8a9b0c1d2   2 minutes ago   COPY requirements.txt . # buildkit              32B
b8c9d0e1f2a3   2 minutes ago   WORKDIR /app                                    0B
c5d6e7f8a9b0   3 weeks ago     ENV PYTHONUNBUFFERED=1                          0B
...
```

## Підсумок інструкцій

| Інструкція | Коли виконується | Призначення |
|---|---|---|
| `FROM` | Збірка | Базовий образ |
| `WORKDIR` | Збірка | Встановити робочу директорію |
| `COPY` | Збірка | Скопіювати файли в образ |
| `RUN` | Збірка | Виконати команду (встановити пакети тощо) |
| `ENV` | Збірка + Запуск | Встановити змінну середовища |
| `ARG` | Лише збірка | Аргумент збірки |
| `EXPOSE` | Документація | Задокументувати порт |
| `CMD` | Запуск | Команда за замовчуванням |
| `ENTRYPOINT` | Запуск | Фіксована команда |

## Корисні посилання

- [Готовий приклад з лекції](https://github.com/kurotych/programming-2sem-code/tree/main/docker/dockerfile-example)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Best practices for writing Dockerfiles](https://docs.docker.com/build/building/best-practices/)
- [Офіційні образи Python на Docker Hub](https://hub.docker.com/_/python)

## Домашнє завдання

1. Написати Dockerfile для одного з ваших попередніх проєктів (наприклад, програми з практики по psycopg2).
2. Зібрати образ та запустити контейнер.
3. Переконатися, що застосунок успішно підключається до PostgreSQL, запущеного в іншому контейнері.
