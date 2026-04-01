# 28. (Л) Неперервна інтеграція (CI). Знайомство з GitHub Actions

## Зміст лекції

1. Що таке неперервна інтеграція (CI)
2. Огляд GitHub Actions
3. Структура workflow-файлу
4. CI для Python-проєкту з Flask та PostgreSQL
5. Практичні поради

## Що таке неперервна інтеграція (CI)

**Неперервна інтеграція (Continuous Integration, CI)** — це практика розробки, при якій кожна зміна коду автоматично перевіряється: запускаються тести, лінтери, збірка тощо. Мета — виявляти помилки якнайраніше, ще до того, як код потрапить у головну гілку.

### Як виглядає розробка без CI

```mermaid
graph LR
    D["Розробник пише код"] --> P["git push"]
    P --> M["Merge у main"]
    M --> B["Помилка на production 💥"]

    style B fill:#ff6b6b,stroke:#333,color:#fff
```

Без CI розробник може випадково зламати проєкт — забути запустити тести, не помітити конфлікт із чужими змінами, або пропустити помилку лінтера.

### Як виглядає розробка з CI

```mermaid
graph LR
    D["Розробник пише код"] --> P["git push"]
    P --> CI["CI: тести, лінтери"]
    CI -->|"Все пройшло ✅"| M["Merge у main"]
    CI -->|"Помилка ❌"| F["Виправити код"]
    F --> P

    style CI fill:#339af0,stroke:#333,color:#fff
    style M fill:#51cf66,stroke:#333,color:#000
    style F fill:#ffd43b,stroke:#333,color:#000
```

CI-сервер автоматично запускає перевірки при кожному push або pull request. Якщо перевірки не пройшли — merge блокується, і розробник бачить, що саме зламалось.

### Що зазвичай перевіряє CI

| Перевірка | Інструмент | Що знаходить |
|---|---|---|
| Тести | `pytest` | Баги в логіці |
| Лінтер | `flake8`, `ruff` | Стилістичні помилки, підозрілий код |
| Форматування | `black`, `ruff format` | Неконсистентне форматування |
| Типи | `mypy` | Помилки типізації |

## Огляд GitHub Actions

**GitHub Actions** — це вбудована CI/CD платформа GitHub. Вона дозволяє автоматизувати будь-які процеси прямо в репозиторії — запуск тестів, деплой, створення релізів тощо.

### Основні переваги

- **Вбудовано в GitHub** — не потрібно налаштовувати зовнішній сервіс
- **Безкоштовно** для публічних репозиторіїв та з обмеженнями для приватних
- **Велика екосистема** — тисячі готових дій (actions) від спільноти
- **Підтримка Docker** — можна запускати сервіси (PostgreSQL, Redis тощо) як контейнери

### Ключові концепції

```mermaid
graph TD
    WF["Workflow<br/>(файл .yml)"] --> E["Event<br/>(push, pull_request)"]
    WF --> J1["Job 1"]
    WF --> J2["Job 2"]
    J1 --> S1["Step 1"]
    J1 --> S2["Step 2"]
    J1 --> S3["Step 3"]

    style WF fill:#339af0,stroke:#333,color:#fff
    style E fill:#ffd43b,stroke:#333,color:#000
    style J1 fill:#51cf66,stroke:#333,color:#000
    style J2 fill:#51cf66,stroke:#333,color:#000
```

| Концепція | Опис |
|---|---|
| **Workflow** | Автоматизований процес, описаний у YAML-файлі |
| **Event** | Подія, яка запускає workflow (push, pull request, розклад тощо) |
| **Job** | Набір кроків, що виконуються на одній віртуальній машині |
| **Step** | Окремий крок всередині job — команда або дія (action) |
| **Action** | Готовий блок, який можна перевикористовувати (наприклад, `actions/checkout`) |
| **Runner** | Віртуальна машина, на якій виконується job |

### Де зберігаються workflow-файли

Workflow-файли зберігаються в директорії `.github/workflows/` у корені репозиторію:

```
my-project/
├── .github/
│   └── workflows/
│       └── ci.yml        # CI workflow
├── app.py
├── test_app.py
└── requirements.txt
```

## Структура workflow-файлу

Розглянемо мінімальний CI workflow для Python-проєкту:

```yaml
# .github/workflows/ci.yml

name: CI                          # Назва workflow (відображається в GitHub UI)

on:                               # Коли запускати
  push:                           # При push у ці гілки
    branches: [main]
  pull_request:                   # При створенні/оновленні PR у ці гілки
    branches: [main]

jobs:                             # Список jobs
  test:                           # Ідентифікатор job
    runs-on: ubuntu-latest        # На якій ОС запускати

    steps:                        # Кроки job
      - uses: actions/checkout@v4          # Крок 1: клонувати репозиторій
      - uses: actions/setup-python@v5      # Крок 2: встановити Python
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt  # Крок 3: встановити залежності
      - run: pytest -v                        # Крок 4: запустити тести
```

### Розбір кожної секції

#### `name`

```yaml
name: CI
```

Назва workflow. Відображається у вкладці **Actions** репозиторію на GitHub.

#### `on` — тригери

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

Визначає, коли запускається workflow:

| Тригер | Коли спрацьовує |
|---|---|
| `push` | При push комітів у вказані гілки |
| `pull_request` | При створенні або оновленні PR у вказані гілки |

#### `jobs`

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
```

Кожен job має ідентифікатор (тут `test`) і вказує runner — віртуальну машину для виконання. `ubuntu-latest` — це безкоштовний Linux runner від GitHub.

!!! info "Runner"
    Runner — це віртуальна машина, яку GitHub створює спеціально для виконання вашого workflow. Після завершення workflow машина знищується. Це означає, що кожен запуск починається з чистого стану — ніяких залишків від попередніх запусків.

#### `steps` — кроки

Кроки виконуються послідовно. Є два типи кроків:

**1. `uses` — використання готової дії (action):**

```yaml
- uses: actions/checkout@v4
```

Ця дія клонує ваш репозиторій у runner. Без цього кроку код недоступний.

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
```

Ця дія встановлює Python потрібної версії. Параметр `with` передає вхідні дані до action.

**2. `run` — виконання shell-команди:**

```yaml
- run: pip install -r requirements.txt
- run: pytest -v
```

Виконує довільну команду в терміналі runner.

### Як побачити результат

Після push у репозиторій, перейдіть у вкладку **Actions** на GitHub:

```
https://github.com/<user>/<repo>/actions
```

Там видно список запусків workflow, статус кожного (успіх/помилка), і детальні логи кожного кроку.

## CI для Python-проєкту з Flask та PostgreSQL

Розглянемо повноцінний CI workflow для Flask API з PostgreSQL — проєкту, який ми розробляли в попередніх лекціях.

### Використання сервісів (services)

GitHub Actions дозволяє запускати Docker-контейнери як **сервіси** — наприклад, базу даних PostgreSQL:

```yaml
# .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:                              # Docker-сервіси
      postgres:
        image: postgres:17                 # Образ PostgreSQL
        env:                               # Змінні середовища контейнера
          POSTGRES_PASSWORD: secret
          POSTGRES_DB: tasks_test_db
        ports:
          - 5432:5432                      # Прокидання порту
        options: >-                        # Перевірка готовності
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies        # Можна додати назву кроку
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest -v
```

### Як працюють сервіси

```mermaid
graph TD
    GA["GitHub Actions Runner"] --> PG["PostgreSQL контейнер<br/>localhost:5432"]
    GA --> S["Steps"]
    S --> S1["checkout"]
    S1 --> S2["setup-python"]
    S2 --> S3["pip install"]
    S3 --> S4["pytest -v"]
    S4 -.->|"Тести підключаються<br/>до localhost:5432"| PG

    style GA fill:#339af0,stroke:#333,color:#fff
    style PG fill:#336791,stroke:#333,color:#fff
```

Секція `services` запускає PostgreSQL контейнер **до виконання кроків**. Контейнер доступний по `localhost:5432` — так само, як при локальній розробці з Docker.

Параметр `options` з health check гарантує, що PostgreSQL повністю запущений перед початком тестів:

| Параметр | Значення |
|---|---|
| `--health-cmd pg_isready` | Команда перевірки готовності PostgreSQL |
| `--health-interval 10s` | Перевіряти кожні 10 секунд |
| `--health-timeout 5s` | Таймаут однієї перевірки |
| `--health-retries 5` | Кількість спроб |

### Змінні середовища

Змінні середовища можна задавати на рівні workflow, job або step:

```yaml
env:                                 # Рівень workflow — доступні всюди
  FLASK_ENV: testing

jobs:
  test:
    runs-on: ubuntu-latest
    env:                             # Рівень job
      DATABASE_URL: postgresql://postgres:secret@localhost:5432/tasks_test_db

    steps:
      - run: echo "DB = $DATABASE_URL"
        env:                         # Рівень step
          DEBUG: "1"
```

### Секрети (secrets)

Для зберігання конфіденційних даних (паролі, токени, API-ключі) використовуйте **GitHub Secrets**. Секрети налаштовуються в розділі **Settings > Secrets and variables > Actions** репозиторію.

```yaml
steps:
  - run: deploy.sh
    env:
      API_TOKEN: ${{ secrets.API_TOKEN }}
```

!!! warning "Безпека"
    Ніколи не записуйте паролі, токени чи інші секрети безпосередньо у workflow-файл. Використовуйте GitHub Secrets — вони шифруються та не відображаються в логах.

## Практичні поради

### 1. Починайте з простого

Не намагайтеся одразу побудувати складний CI pipeline. Почніть з мінімального workflow, який просто запускає тести:

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: pytest
```

Додавайте лінтери, перевірку типів та інші кроки поступово.

### 2. Кешуйте залежності

Встановлення пакетів через `pip install` може зайняти час. Кешування прискорює CI:

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    cache: "pip"                     # Кешувати pip-залежності
```

Параметр `cache: "pip"` зберігає завантажені пакети між запусками workflow. Якщо `requirements.txt` не змінився — пакети беруться з кешу.

### 3. Захист гілки main

У налаштуваннях репозиторію (**Settings > Branches > Branch protection rules**) можна увімкнути **обов'язкову перевірку CI** перед merge:

- Require status checks to pass before merging
- Обрати потрібні перевірки (наприклад, `test` і `lint`)

Це гарантує, що код у `main` завжди проходить усі перевірки.

### 4. Корисні готові actions

| Action | Призначення |
|---|---|
| `actions/checkout@v4` | Клонування репозиторію |
| `actions/setup-python@v5` | Встановлення Python |
| `actions/cache@v4` | Кешування файлів між запусками |
| `actions/upload-artifact@v4` | Збереження файлів (звіти, логи) |

## Підсумок

| Концепція | Опис |
|---|---|
| **CI** | Автоматичний запуск перевірок при кожній зміні коду |
| **GitHub Actions** | Вбудована CI/CD платформа GitHub |
| **Workflow** | YAML-файл у `.github/workflows/` |
| **Job** | Набір кроків на одному runner |
| **Step** | Окремий крок: `uses` (action) або `run` (команда) |
| **Services** | Docker-контейнери (PostgreSQL, Redis тощо) для тестів |
| **Secrets** | Безпечне зберігання конфіденційних даних |

Ключові принципи:

- **Автоматизуйте перевірки** — тести та лінтери повинні запускатись автоматично при кожному push
- **Починайте з простого** — мінімальний workflow з тестами, потім додавайте кроки
- **Використовуйте сервіси** — PostgreSQL та інші залежності запускаються як Docker-контейнери
- **Захищайте main** — налаштуйте обов'язкові перевірки перед merge
- **Зберігайте секрети безпечно** — використовуйте GitHub Secrets замість хардкоду

## Корисні посилання

- [GitHub Actions — Documentation](https://docs.github.com/en/actions)
- [GitHub Actions — Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions — Using PostgreSQL service containers](https://docs.github.com/en/actions/using-containerized-services/creating-postgresql-service-containers)
- [actions/setup-python](https://github.com/actions/setup-python)

## Домашнє завдання

1. Створити новий репозиторій на GitHub з Flask API та тестами з [лекції 26](../module2/26-testing-rest-api-lecture.md). Додати workflow `.github/workflows/ci.yml`, який запускає тести з PostgreSQL при push у `main`. Переконатися, що workflow успішно проходить у вкладці Actions.
2. Додати кешування pip-залежностей до workflow (параметр `cache: "pip"` у `actions/setup-python`). Порівняти час виконання workflow до та після додавання кешу.
3. Налаштувати Branch Protection Rule для гілки `main`: увімкнути обов'язкову перевірку CI перед merge. Створити pull request із навмисною помилкою в тестах та переконатися, що merge заблоковано.
