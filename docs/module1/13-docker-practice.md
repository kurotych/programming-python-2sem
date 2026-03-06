# 13. (П6) Встановлення Docker. Базові команди (build, run, stop, rm, logs, exec)

## Передумови

- Прочитані лекції [11 — Вступ до Docker](/ua/courses/programming-2sem/module1/11-docker-lecture/) та [12 — Dockerfile](/ua/courses/programming-2sem/module1/12-dockerfile-lecture/)
- Встановлений Docker ([інструкція встановлення](#крок-1-встановлення-docker))

## Завдання

Встановити Docker, створити персоналізований Python-застосунок, запакувати його в Docker-образ та відпрацювати базові команди управління контейнерами.

### Крок 1. Встановлення Docker

Встановити Docker Engine, дотримуючись офіційної інструкції для вашої ОС:

- **Linux (Ubuntu/Debian):** [docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- **macOS:** [docs.docker.com/desktop/setup/install/mac-install](https://docs.docker.com/desktop/setup/install/mac-install/)
- **Windows:** [docs.docker.com/desktop/setup/install/windows-install](https://docs.docker.com/desktop/setup/install/windows-install/)

Перевірити, що Docker встановлено коректно:

```bash
docker --version
# Docker version 28.x.x, build ...

docker run hello-world
# Hello from Docker!
# This message shows that your installation appears to be working correctly.
```

### Крок 2. Створення персоналізованого застосунку

Створити директорію проєкту та файли:

```
docker-practice/
├── app.py
├── Dockerfile
└── .dockerignore
```

#### Файл `app.py`

Написати програму, яка:

1. Виводить привітання з **вашим ім'ям та прізвищем** (українською)
2. Показує поточну дату та час
3. Виводить інформацію про систему (версія Python, ОС)
4. Працює у циклі — кожні 5 секунд виводить повідомлення з часом (щоб контейнер не завершувався одразу і можна було практикувати команди `logs`, `stop`, `exec`)

Приклад виводу:

```
============================================
  Привіт! Мене звати Тарас Шевченко
  Дата: 2026-03-06 14:30:00
  Python: 3.13.2
  Платформа: Linux-6.1.0-amd64
============================================

[14:30:05] Тарас Шевченко — програма працює...
[14:30:10] Тарас Шевченко — програма працює...
[14:30:15] Тарас Шевченко — програма працює...
```

Підказки:

- Використовуйте модулі `datetime`, `platform`, `sys`, `time`
- Ім'я можна зберігати у змінній або зчитувати зі змінної середовища `STUDENT_NAME`
- `time.sleep(5)` для паузи між повідомленнями

#### Dockerfile

Написати Dockerfile на основі `python:3.13-slim`, що:

- Встановлює `WORKDIR`
- Копіює код застосунку
- Використовує `ENV` для змінної `STUDENT_NAME` зі значенням за замовчуванням — **вашим ім'ям**
- Задає команду запуску через `CMD`

#### Файл `.dockerignore`

```
__pycache__/
*.pyc
.git/
env/
venv/
Dockerfile
.dockerignore
```

### Крок 3. Збірка образу (`docker build`)

Зібрати Docker-образ з тегом, що містить ваше прізвище (латиницею):

```bash
# Замініть shevchenko на ваше прізвище
docker build -t shevchenko-app:1.0 .
```

Перевірити, що образ створено:

```bash
docker images | grep shevchenko
```

Переглянути шари образу:

```bash
docker history shevchenko-app:1.0
```

### Крок 4. Запуск контейнера (`docker run`)

Запустити контейнер у різних режимах:

**Запуск на передньому плані (foreground):**

```bash
docker run --name my-app shevchenko-app:1.0
```

Побачити вивід програми. Зупинити через `Ctrl+C`.

**Запуск у фоновому режимі (detached):**

```bash
docker run -d --name my-app-bg shevchenko-app:1.0
```

**Запуск з перевизначенням імені через змінну середовища:**

```bash
docker run -d --name my-app-env -e STUDENT_NAME="Інше Ім'я" shevchenko-app:1.0
```

### Крок 5. Перегляд логів (`docker logs`)

Переглянути логи фонового контейнера:

```bash
# Всі логи
docker logs my-app-bg

# Останні 5 рядків
docker logs --tail 5 my-app-bg

# Логи у реальному часі (як tail -f)
docker logs -f my-app-bg
```

Вийти зі слідкування за логами через `Ctrl+C` (контейнер продовжує працювати).

### Крок 6. Виконання команд у контейнері (`docker exec`)

Виконати команди всередині працюючого контейнера:

```bash
# Перевірити версію Python у контейнері
docker exec my-app-bg python --version

# Подивитися файли в робочій директорії
docker exec my-app-bg ls -la

# Відкрити інтерактивний shell у контейнері
docker exec -it my-app-bg bash
```

Усередині контейнера виконати:

```bash
# Перевірити змінну середовища
echo $STUDENT_NAME

# Подивитися процеси
ps aux

# Перевірити вміст файлу
cat app.py

# Вийти з контейнера
exit
```

### Крок 7. Зупинка та видалення (`docker stop`, `docker rm`)

```bash
# Подивитися працюючі контейнери
docker ps

# Зупинити контейнер
docker stop my-app-bg

# Подивитися всі контейнери (включно зі зупиненими)
docker ps -a

# Видалити зупинений контейнер
docker rm my-app-bg

# Видалити всі зупинені контейнери за раз
docker rm my-app my-app-env
```

### Крок 8. Повний цикл

Виконати повний цикл роботи з контейнером та зафіксувати результати:

1. Зібрати образ версії `2.0` (можна змінити щось у програмі)
2. Запустити контейнер у фоновому режимі
3. Переглянути логи та переконатися, що відображається ваше ім'я
4. Виконати `docker exec` для перевірки змінної `STUDENT_NAME`
5. Зупинити та видалити контейнер
6. Видалити образ:

```bash
docker rmi shevchenko-app:1.0 shevchenko-app:2.0
```

## Результат

Після виконання роботи студент має:

- Встановлений та працюючий Docker
- Персоналізований Docker-образ з вашим ім'ям
- Практичний досвід роботи з командами: `build`, `run`, `stop`, `rm`, `logs`, `exec`, `ps`, `images`, `rmi`, `history`
