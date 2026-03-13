# 17. (П8) Проєктування REST API для системи управління задачами

## Передумови

- Прочитана [Лекція 16 — HTTP та архітектурний стиль REST](/ua/courses/programming-2sem/module2/16-http-rest-lecture/)

## Завдання

Спроєктувати REST API для системи управління задачами (Task Manager) у форматі OpenAPI 3.0. Написати специфікацію у YAML, візуалізувати у Swagger Editor та зберегти документацію у PDF.

### Крок 1. Знайомство зі Swagger Editor

1. Відкрийте [Swagger Editor](https://editor.swagger.io/) у браузері
2. Видаліть вміст лівої панелі
3. Вставте мінімальну специфікацію для перевірки:

```yaml
openapi: 3.0.3
info:
  title: Test API
  version: 1.0.0
paths: {}
```

Праворуч має з'явитись візуалізація без помилок. Якщо все працює — переходимо до проєктування.

### Крок 2. Проєктування ресурсів

Система управління задачами має два ресурси:

**`tasks`** — задачі:

| Поле | Тип | Опис |
|---|---|---|
| id | integer | Унікальний ідентифікатор |
| title | string | Назва задачі (обов'язкове) |
| description | string | Опис задачі |
| status | string | Статус: `todo`, `in_progress`, `done` |
| priority | string | Пріоритет: `low`, `medium`, `high` |
| created_by | string | Ім'я автора |
| created_at | string | Дата створення (ISO 8601) |

**`categories`** — категорії задач:

| Поле | Тип | Опис |
|---|---|---|
| id | integer | Унікальний ідентифікатор |
| name | string | Назва категорії (обов'язкове) |
| description | string | Опис категорії |

Задача може належати до категорії.

### Крок 3. Написання OpenAPI-специфікації

У Swagger Editor написати повну специфікацію API. Нижче наведено каркас та один повністю описаний endpoint як приклад. **Решту endpoint-ів студент описує самостійно.**

!!! warning "Персоналізація"
    У полі `info.description` та в `example` для поля `created_by` повинно бути **ваше ім'я та прізвище**.

#### Каркас специфікації

```yaml
openapi: 3.0.3
info:
  title: Task Manager API
  description: |
    REST API для системи управління задачами.
    Автор: Тарас Шевченко
  version: 1.0.0

tags:
  - name: tasks
    description: Операції з задачами
  - name: categories
    description: Операції з категоріями

paths:
  # ----------------------------------------
  # TASKS
  # ----------------------------------------
  /api/tasks:
    get:
      tags: [tasks]
      summary: Отримати список задач
      description: Повертає список усіх задач. Підтримує фільтрацію.
      parameters:
        - name: status
          in: query
          required: false
          description: Фільтр за статусом
          schema:
            type: string
            enum: [todo, in_progress, done]
        - name: priority
          in: query
          required: false
          description: Фільтр за пріоритетом
          schema:
            type: string
            enum: [low, medium, high]
      responses:
        '200':
          description: Список задач
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Task'

    post:
      # ОПИШІТЬ САМОСТІЙНО: створення нової задачі
      # Підказки:
      #   - requestBody з $ref на TaskCreate
      #   - responses: 201 (створено), 400 (невалідні дані)

  /api/tasks/{id}:
    get:
      # ОПИШІТЬ САМОСТІЙНО: отримання задачі за ID
      # Підказки:
      #   - parameters: id (in: path)
      #   - responses: 200 (задача), 404 (не знайдено)

    put:
      # ОПИШІТЬ САМОСТІЙНО: оновлення задачі
      # Підказки:
      #   - parameters: id (in: path)
      #   - requestBody з $ref на TaskCreate
      #   - responses: 200 (оновлено), 404 (не знайдено), 400

    delete:
      # ОПИШІТЬ САМОСТІЙНО: видалення задачі
      # Підказки:
      #   - parameters: id (in: path)
      #   - responses: 204 (видалено), 404 (не знайдено)

  # ----------------------------------------
  # CATEGORIES
  # ----------------------------------------
  /api/categories:
    get:
      # ОПИШІТЬ САМОСТІЙНО: отримання списку категорій

    post:
      # ОПИШІТЬ САМОСТІЙНО: створення категорії

  /api/categories/{id}/tasks:
    get:
      # ОПИШІТЬ САМОСТІЙНО: отримання задач у категорії

components:
  schemas:
    Task:
      type: object
      properties:
        id:
          type: integer
          example: 1
        title:
          type: string
          example: Вивчити REST API
        description:
          type: string
          example: Прочитати лекцію 16 та виконати практичну
        status:
          type: string
          enum: [todo, in_progress, done]
          example: in_progress
        priority:
          type: string
          enum: [low, medium, high]
          example: high
        created_by:
          type: string
          example: Тарас Шевченко
        created_at:
          type: string
          format: date-time
          example: "2026-03-13T10:00:00Z"

    TaskCreate:
      type: object
      required:
        - title
        - status
        - priority
        - created_by
      properties:
        title:
          type: string
          example: Вивчити REST API
        description:
          type: string
          example: Прочитати лекцію 16
        status:
          type: string
          enum: [todo, in_progress, done]
          example: todo
        priority:
          type: string
          enum: [low, medium, high]
          example: high
        created_by:
          type: string
          example: Тарас Шевченко

    Category:
      # ОПИШІТЬ САМОСТІЙНО: схема категорії
      # Поля: id, name, description

    CategoryCreate:
      # ОПИШІТЬ САМОСТІЙНО: схема для створення категорії
      # Обов'язкові поля: name

    Error:
      type: object
      properties:
        error:
          type: string
          example: Resource not found
```

#### Endpoint-и, які потрібно описати

**Задачі (tasks):**

| Операція | Метод | URL | Коди відповідей |
|---|---|---|---|
| Отримати всі задачі | GET | `/api/tasks` | 200 |
| Отримати задачу за ID | GET | `/api/tasks/{id}` | 200, 404 |
| Створити задачу | POST | `/api/tasks` | 201, 400 |
| Оновити задачу | PUT | `/api/tasks/{id}` | 200, 400, 404 |
| Видалити задачу | DELETE | `/api/tasks/{id}` | 204, 404 |

**Категорії (categories):**

| Операція | Метод | URL | Коди відповідей |
|---|---|---|---|
| Отримати всі категорії | GET | `/api/categories` | 200 |
| Створити категорію | POST | `/api/categories` | 201, 400 |
| Задачі в категорії | GET | `/api/categories/{id}/tasks` | 200, 404 |

### Крок 4. Візуалізація у Swagger Editor

1. Переконайтесь, що **немає помилок** (червоних позначок)
2. Перевірте, що всі endpoint-и відображаються коректно
3. Розгорніть кожен endpoint та переконайтесь, що:
    - Описані параметри запиту
    - Показані приклади тіла запиту (для POST, PUT)
    - Перелічені коди відповідей з прикладами
    - Ваше ім'я відображається в описі API та в прикладах `created_by`

### Крок 5. Збереження документації у PDF

Зберегти візуалізовану документацію у PDF:

1. У Swagger Editor переконайтесь, що всі endpoint-и розгорнуті
2. У браузері: **Файл → Друк** (або `Ctrl+P` / `Cmd+P`)
3. Оберіть **"Зберегти як PDF"** замість принтера
4. Збережіть файл як `task-manager-api.pdf`

## Результат

Після виконання роботи студент здає:

- **`task-manager-api.pdf`** — візуалізована документація API зі Swagger Editor, де відображається ваше ім'я в описі та прикладах
