# 40. (Л) Проблеми виконання асинхронного коду, ручне керування подійним циклом та налагодження asyncio

## Зміст лекції

1. Типові помилки в асинхронному коді
2. Ручне керування event loop
3. Режим налагодження asyncio
4. Логування та діагностика
5. Як знайти «повільну» корутину

## Типові помилки в асинхронному коді

Більшість проблем у asyncio зводяться до кількох повторюваних шаблонів. Розберімося з найчастішими.

### Помилка 1: забути `await`

```python
import asyncio


async def greet() -> str:
    return "hello"


async def main() -> None:
    result = greet()        # 🛑 без await
    print(result)


asyncio.run(main())
```

Вивід:

```text
<coroutine object greet at 0x...>
RuntimeWarning: coroutine 'greet' was never awaited
```

Без `await` ми отримуємо об'єкт корутини, а не результат. Python зазвичай попереджає про це через `RuntimeWarning`.

✅ Правильно:

```python
result = await greet()
```

### Помилка 2: блокувальний виклик усередині async-функції

```python
import asyncio
import time


async def fetch_data() -> None:
    time.sleep(3)           # 🛑 блокує весь event loop
    print("done")


async def main() -> None:
    await asyncio.gather(fetch_data(), fetch_data())


asyncio.run(main())
```

Тут `time.sleep` зупиняє **увесь** event loop на 3 секунди. Інші задачі чекають, ніби програма зависла.

✅ Правильно — використовуйте `asyncio.sleep` або винесіть блокувальний код у потік:

```python
await asyncio.sleep(3)
# або
await asyncio.to_thread(time.sleep, 3)
```

!!! warning "Що ще блокує event loop"
    - `requests.get(...)` (треба `aiohttp` або `httpx.AsyncClient`)
    - синхронні драйвери БД (psycopg2, sqlite3) — використовуйте `asyncpg` або `to_thread`
    - читання великих файлів через `open(...).read()`
    - довгі CPU-обчислення (хешування, парсинг великих JSON)

### Помилка 3: не зберігати посилання на `Task`

```python
async def background() -> None:
    await asyncio.sleep(1)
    print("done")


async def main() -> None:
    asyncio.create_task(background())   # 🛑 посилання втрачено
    await asyncio.sleep(2)
```

Збирач сміття може видалити задачу до її завершення. Завжди зберігайте посилання:

```python
task = asyncio.create_task(background())
await task
```

### Помилка 4: змішування sync і async

```python
async def get_data() -> str:
    return "data"


def main() -> None:
    result = get_data()     # 🛑 не можна викликати async з sync без run
    print(result)
```

Async-функцію не можна викликати «звичайним» способом. Або робіть `main` теж асинхронною, або запускайте через `asyncio.run`.

## Ручне керування event loop

Зазвичай нам достатньо `asyncio.run`, але іноді треба контролювати event loop напряму — наприклад, для інтеграції з GUI, тестами або довгоживими додатками.

### Що робить `asyncio.run`

`asyncio.run(coro)` під капотом виконує приблизно таке:

```python
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)
try:
    loop.run_until_complete(coro)
finally:
    loop.close()
```

Тобто — створює loop, виконує корутину до кінця, закриває loop.

### Створення власного event loop

```python
import asyncio


async def work() -> str:
    await asyncio.sleep(1)
    return "result"


def main() -> None:
    loop = asyncio.new_event_loop()
    try:
        result = loop.run_until_complete(work())
        print(result)
    finally:
        loop.close()


main()
```

Тут ми робимо те саме, що `asyncio.run`, але кожен крок видно явно.

### `run_until_complete` vs `run_forever`

| Метод | Поведінка |
|---|---|
| `run_until_complete(coro)` | Виконує корутину, повертає її результат, потім зупиняється |
| `run_forever()` | Працює, поки хтось не викличе `loop.stop()` |

`run_forever` корисний для серверів, які мають слухати запити необмежено довго.

```python
import asyncio


async def tick() -> None:
    while True:
        print("tick")
        await asyncio.sleep(1)


def main() -> None:
    loop = asyncio.new_event_loop()
    loop.create_task(tick())
    try:
        loop.run_forever()
    except KeyboardInterrupt:
        print("stopping")
    finally:
        loop.close()


main()
```

Натиснувши `Ctrl+C`, ми перериваємо `run_forever` і коректно закриваємо loop.

```mermaid
graph LR
    A["new_event_loop"] --> B["create_task"]
    B --> C["run_forever"]
    C --> D{"Ctrl+C?"}
    D -->|ні| C
    D -->|так| E["loop.close"]

    style A fill:#339af0,stroke:#333,color:#fff
    style E fill:#fa5252,stroke:#333,color:#fff
```

!!! tip "Коли НЕ робити ручне керування"
    Для більшості програм `asyncio.run` — найкращий вибір. Ручне керування потрібне лише коли:

    - Ви інтегруєтесь із зовнішнім event loop (GUI, веб-сервери).
    - Пишете тестовий фреймворк для async-коду.
    - Маєте довготривалий процес, що приймає підключення.

## Режим налагодження asyncio

asyncio має вбудований **debug mode**, що допомагає знайти типові проблеми: повільні корутини, забуті `await`, блокувальні виклики.

### Як увімкнути

Три способи:

**1. Через `asyncio.run`:**

```python
asyncio.run(main(), debug=True)
```

**2. Через змінну середовища:**

```bash
PYTHONASYNCIODEBUG=1 python my_script.py
```

**3. Через виклик методу:**

```python
loop = asyncio.new_event_loop()
loop.set_debug(True)
```

### Що дає debug mode

Після увімкнення asyncio починає:

- **Попереджати про незачекані корутини** (`coroutine was never awaited`).
- **Логувати «повільні» callback'и** — функції, що виконуються довше 100 мс і блокують loop.
- **Показувати джерело створення задач** у трасуваннях винятків.
- **Перевіряти**, чи викликаються `loop`-методи з правильного потоку.

Приклад попередження про повільний callback:

```text
Executing <Task pending name='Task-2' coro=<work() running at script.py:5>
   wait_for=<Future pending cb=[...]>
created at script.py:12> took 0.350 seconds
```

Це означає: задача затримала event loop на 350 мс — імовірно, всередині є блокувальний виклик.

### Зміна порогу повільності

За замовчуванням попередження з'являється при затримці > 100 мс. Для діагностики можна знизити поріг:

```python
loop = asyncio.get_running_loop()
loop.slow_callback_duration = 0.01  # 10 мс
```

## Логування та діагностика

Для нетривіальних async-програм `print` швидко перестає допомагати. Стандартний модуль `logging` працює і в синхронному, і в асинхронному коді.

### Базове налаштування

```python
import asyncio
import logging


logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
log = logging.getLogger("app")


async def fetch(url: str) -> str:
    log.info("fetching %s", url)
    await asyncio.sleep(1)
    log.info("done %s", url)
    return url


async def main() -> None:
    await asyncio.gather(fetch("a"), fetch("b"))


asyncio.run(main(), debug=True)
```

Вивід (приблизно):

```text
2026-05-01 10:00:00 [DEBUG] asyncio: Using selector: EpollSelector
2026-05-01 10:00:00 [INFO] app: fetching a
2026-05-01 10:00:00 [INFO] app: fetching b
2026-05-01 10:00:01 [INFO] app: done a
2026-05-01 10:00:01 [INFO] app: done b
```

Зверніть увагу на рядки від логера `asyncio` — у debug-режимі він сам розповідає про свою роботу.

### Корисні логери

| Логер | Що показує |
|---|---|
| `asyncio` | Стан event loop, повільні задачі |
| `app` (ваш) | Бізнес-логіка вашого коду |
| `aiohttp.client` | Деталі HTTP-запитів (якщо використовуєте aiohttp) |

## Як знайти «повільну» корутину

Замірюємо час кожного кроку через `time.perf_counter` — одразу видно, де вузьке місце.

```python
import asyncio
import time


async def fetch_data() -> str:
    await asyncio.sleep(0.2)
    return "data"


async def parse_data() -> str:
    time.sleep(1.5)             # 🛑 блокувальний виклик
    return "parsed"


async def save_data() -> None:
    await asyncio.sleep(0.3)


async def step(name: str, coro):
    start = time.perf_counter()
    result = await coro
    elapsed = time.perf_counter() - start
    print(f"[{name}] {elapsed:.3f}s")
    return result


async def main() -> None:
    await step("fetch", fetch_data())
    await step("parse", parse_data())
    await step("save",  save_data())


asyncio.run(main())
```

Вивід:

```text
[fetch] 0.200s
[parse] 1.500s
[save]  0.300s
```

Видно, що `parse` — найповільніший крок. Заглядаємо всередину і знаходимо `time.sleep` — блокувальний виклик. Виносимо його у потік через `asyncio.to_thread`:

```python
async def parse_data() -> str:
    await asyncio.to_thread(time.sleep, 1.5)
    return "parsed"
```

## Підсумок

| Тема | Ключова думка |
|---|---|
| Типові помилки | Забутий `await`, блокувальний виклик, втрачене посилання на `Task` |
| `asyncio.run` | За кулісами створює, запускає і закриває event loop |
| `loop.run_until_complete` | Виконує корутину до завершення |
| `loop.run_forever` | Працює, поки `loop.stop()` не зупинить |
| `debug=True` | Вмикає попередження про повільні задачі та забуті `await` |
| `slow_callback_duration` | Поріг (сек.), коли asyncio логує повільну задачу |
| `logging` | Звичайний модуль `logging` повноцінно працює і в async-коді |
| `asyncio.all_tasks()` | Список живих задач — для діагностики «зомбі» |
| `asyncio.to_thread` | Винести блокувальний код у потік |

Ключові принципи:

- **Завжди `await`** — Python попереджає, але краще ловити це самому.
- **Жоден блокувальний виклик** не повинен бути в async-функції без `to_thread`.
- **Debug mode** — перший інструмент при будь-яких підозрах на повільність.
- **`logging`** замість `print` для нетривіальних програм.
- **Ручне керування loop** — лише коли справді треба (сервери, тести, GUI-інтеграція).

## Корисні посилання

- [Python docs — Developing with asyncio](https://docs.python.org/3/library/asyncio-dev.html)
- [Python docs — Event Loop](https://docs.python.org/3/library/asyncio-eventloop.html)
- [Python docs — asyncio debug mode](https://docs.python.org/3/library/asyncio-dev.html#debug-mode)
- [Python docs — asyncio.to_thread](https://docs.python.org/3/library/asyncio-task.html#asyncio.to_thread)
- [Python docs — logging](https://docs.python.org/3/library/logging.html)
