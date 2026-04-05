# 31. (П15) Розгортання Flask REST API у Docker: створення Dockerfile

## Передумови

- Прочитана [Лекція 30 — Контейнеризація Flask-застосунків за допомогою Docker](/ua/courses/programming-2sem/module2/30-docker-flask-lecture/)
- Прочитані лекції [11 — Вступ до Docker](/ua/courses/programming-2sem/module1/11-docker-lecture/) та [12 — Dockerfile](/ua/courses/programming-2sem/module1/12-dockerfile-lecture/)
- Встановлений Docker

!!! warning "Персоналізація"
    Застосунок повинен містити **ваше ім'я та прізвище** у відповідях API. Тег Docker-образу повинен містити ваше прізвище (латиницею). На скріншотах у звіті повинно бути видно ваше ім'я у відповідях `curl`.

## Завдання

Створити персоналізований Flask REST API, запакувати його в Docker-образ та продемонструвати роботу контейнера.

### Частина 1: Створення Flask API

Створити директорію проєкту:

```
flask-api-docker/
├── app.py
├── requirements.txt
└── Dockerfile
```

#### Файл `requirements.txt`

```
flask==3.1.1
```

#### Файл `app.py`

Написати Flask-застосунок з такими ендпоінтами:

**1. `GET /` — привітання з вашим ім'ям:**

```json
{
  "message": "API створено: Тарас Шевченко"
}
```

**2. `GET /health` — перевірка стану:**

```json
{
  "status": "ok"
}
```

**3. `GET /api/student` — інформація про студента:**

Повертає JSON з вашими даними:

```json
{
  "first_name": "Тарас",
  "last_name": "Шевченко",
  "group": "КІ-21"
}
```

**4. `GET /api/courses` — список курсів:**

Повертає масив з 3-5 курсами (назви довільні):

```json
{
  "student": "Тарас Шевченко",
  "courses": [
    {"id": 1, "name": "Python програмування"},
    {"id": 2, "name": "Бази даних"},
    {"id": 3, "name": "Веб-розробка"}
  ]
}
```

**5. `GET /api/courses/<id>` — окремий курс за ID:**

```json
{
  "id": 1,
  "name": "Python програмування"
}
```

Якщо курс не знайдено — повернути статус `404`:

```json
{
  "error": "Course not found"
}
```

!!! note "Підказка"
    Курси можна зберігати у звичайному списку Python — база даних не потрібна.

**Важливо:** Flask повинен слухати `0.0.0.0` (пояснення чому — в [лекції 30](/ua/courses/programming-2sem/module2/30-docker-flask-lecture/#app-py)).

### Частина 2: Dockerfile, збірка та запуск

Написати `Dockerfile` та зібрати Docker-образ для застосунку. Як саме написати Dockerfile — дивіться [лекцію 30](/ua/courses/programming-2sem/module2/30-docker-flask-lecture/#dockerfile-flask).

#### Збірка образу

Зібрати образ з тегом, що містить ваше прізвище:

```bash
# Замініть shevchenko на ваше прізвище
docker build -t shevchenko-flask-api:1.0 .
```

Перевірити створений образ:

```bash
docker images | grep shevchenko
```

#### Запуск контейнера

```bash
docker run -d --name my-api -p 5000:5000 shevchenko-flask-api:1.0
```

#### Тестування всіх ендпоінтів

Перевірити кожен ендпоінт через `curl`:

```bash
# Привітання
curl http://localhost:5000/

# Здоров'я
curl http://localhost:5000/health

# Інформація про студента
curl http://localhost:5000/api/student

# Список курсів
curl http://localhost:5000/api/courses

# Окремий курс
curl http://localhost:5000/api/courses/1

# Неіснуючий курс (має повернути 404)
curl http://localhost:5000/api/courses/999
```

#### Перегляд логів

```bash
# Переглянути логи контейнера
docker logs my-api

# Стежити за логами в реальному часі та надіслати кілька запитів
docker logs -f my-api
```

### Частина 4: Перезбірка після змін

Продемонструвати, що зміни в коді вимагають перезбірки образу:

1. Додати ще один курс у список курсів в `app.py`
2. Перезібрати образ з тегом `2.0`:

```bash
docker build -t shevchenko-flask-api:2.0 .
```

3. Зупинити та видалити старий контейнер:

```bash
docker rm -f my-api
```

4. Запустити новий контейнер з образу `2.0`:

```bash
docker run -d --name my-api -p 5000:5000 shevchenko-flask-api:2.0
```

5. Перевірити, що новий курс з'явився:

```bash
curl http://localhost:5000/api/courses
```

### Частина 5: Запуск на іншому порті

Запустити другий контейнер на іншому порті та перевірити, що обидва працюють:

```bash
# Другий контейнер на порті 8080
docker run -d --name my-api-2 -p 8080:5000 shevchenko-flask-api:2.0

# Перевірити обидва
curl http://localhost:5000/api/student
curl http://localhost:8080/api/student
```

### Завершення

Зупинити та видалити всі контейнери:

```bash
docker rm -f my-api my-api-2
```

## Результат

Після виконання роботи студент здає звіт зі скріншотами:

- Вміст файлів `app.py` та `Dockerfile`
- Збірка образу (`docker build`)
- Результати `curl` для **кожного** ендпоінту (у відповідях видно ваше ім'я)
- Результат `curl` для неіснуючого курсу (статус 404)
- Логи контейнера (`docker logs`)
- Перезбірка образу з тегом `2.0` та перевірка змін
- Два контейнери на різних портах (`docker ps` з обома контейнерами)
