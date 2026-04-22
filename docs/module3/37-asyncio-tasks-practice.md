# 37. (П18) Асинхронне виконання задач із використанням asyncio

## Передумови

- Прочитана [Лекція 34 — Асинхронність усередині Python: GIL, подієвий цикл і сокети](/ua/courses/programming-2sem/module3/34-event-loop-sockets-lecture/)
- Прочитана [Лекція 36 — Основи asyncio. Корутини, затримки та виконання задач](/ua/courses/programming-2sem/module3/36-asyncio-basics-lecture/)
- Встановлена бібліотека `aiohttp`: `pip install aiohttp`

!!! warning "Персоналізація"
    У кожному файлі оголосіть константу `STUDENT = "Imya Prizvysche"` (ваші дані латиницею) і виводьте її у першому рядку кожного запуску програми: `print(f"Student: {STUDENT}")`. На скріншотах у звіті повинно бути видно ваше ім'я.

## Завдання

Навчитися запускати корутини через `asyncio.run`, порівнювати послідовне та конкурентне виконання через `asyncio.gather`, а також виконувати реальні HTTP-запити асинхронно через `aiohttp`.

---

## Частина 1: Перша корутина

### 1.1 Запуск однієї корутини

Створіть файл `async_basic.py`. Напишіть корутину `greet(name: str) -> str`, яка чекає 1 секунду (`asyncio.sleep(1)`) і повертає рядок `f"Hello, {name}"`. Запустіть її через `asyncio.run` і надрукуйте результат.

```python
import asyncio

STUDENT = "Imya Prizvysche"


async def greet(name: str) -> str:
    await asyncio.sleep(1)
    return f"Hello, {name}"


async def main() -> None:
    print(f"Student: {STUDENT}")
    message = await greet("asyncio")
    print(message)


asyncio.run(main())
```

Очікуваний вивід:

```text
Student: Imya Prizvysche
Hello, asyncio
```

### 1.2 Помилка: виклик без await

Щоб відчути типову помилку, тимчасово замініть рядок `message = await greet("asyncio")` на `message = greet("asyncio")` і запустіть. Зафіксуйте у звіті попередження `RuntimeWarning: coroutine 'greet' was never awaited` і поясніть, чому код усередині корутини не виконався. Поверніть `await` на місце.

---

## Частина 2: Послідовно vs конкурентно

### 2.1 Послідовне виконання

Створіть файл `async_sequential.py`. Напишіть корутину `fake_download(url: str, delay: float) -> str`, яка друкує `f"start {url}"`, чекає `delay` секунд через `asyncio.sleep`, друкує `f"done {url}"` і повертає рядок `f"<content of {url}>"`.

Викличте її **послідовно** для трьох URL із затримками 3, 1 і 2 секунди. Виміряйте час виконання через `time.perf_counter`.

```python
import asyncio
import time

STUDENT = "Imya Prizvysche"


async def fake_download(url: str, delay: float) -> str:
    print(f"start {url}")
    await asyncio.sleep(delay)
    print(f"done {url}")
    return f"<content of {url}>"


async def main() -> None:
    print(f"Student: {STUDENT}")

    urls = [
        ("https://example.com/a", 3),
        ("https://example.com/b", 1),
        ("https://example.com/c", 2),
    ]

    start = time.perf_counter()
    results = []
    for url, delay in urls:
        result = await fake_download(url, delay)
        results.append(result)
    print(f"Sequential: {time.perf_counter() - start:.2f}s")


asyncio.run(main())
```

Очікуваний час: ~6 секунд (сума всіх затримок).

### 2.2 Конкурентне виконання через `asyncio.gather`

Створіть файл `async_gather.py`. Використайте ту саму корутину `fake_download` і ті самі URL, але запустіть їх **конкурентно** через `asyncio.gather`. Виміряйте час.

Очікуваний час: ~3 секунди (найдовша затримка).

Зафіксуйте у звіті обидва варіанти з часом і порядком виведення рядків `start` / `done` — зверніть увагу, як змінюється порядок у конкурентній версії.

### 2.3 Порівняльна таблиця

У звіті складіть таблицю:

| Виконання | Час | Пояснення |
|---|---|---|
| Послідовно | ~6.0с | сума затримок |
| Конкурентно (`gather`) | ~3.0с | найдовша затримка |

---

## Частина 3: Реальні HTTP-запити через `aiohttp`

Тепер замінимо `asyncio.sleep` на справжні мережеві запити. Звичайний `requests` блокуватиме event loop, тому використовуємо асинхронний клієнт `aiohttp`.

### 3.1 Конкурентне завантаження

Створіть файл `async_downloads.py`. Напишіть корутину `download(session: aiohttp.ClientSession, url: str) -> tuple[str, int, int]`, яка виконує GET-запит і повертає кортеж `(url, status_code, content_length)`. Використайте `async with session.get(url) as response:`.

```python
import asyncio
import time
import aiohttp

STUDENT = "Imya Prizvysche"


async def download(session: aiohttp.ClientSession, url: str) -> tuple[str, int, int]:
    print(f"start {url}")
    async with session.get(url) as response:
        content = await response.read()
        print(f"done {url} -> {response.status}, {len(content)} bytes")
        return url, response.status, len(content)


async def main() -> None:
    print(f"Student: {STUDENT}")

    urls = [
        "https://httpbin.org/delay/3",
        "https://httpbin.org/delay/2",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/2",
    ]

    start = time.perf_counter()
    async with aiohttp.ClientSession() as session:
        results = await asyncio.gather(
            *[download(session, url) for url in urls]
        )
    elapsed = time.perf_counter() - start

    for url, status, size in results:
        print(f"{url}: {status}, {size} bytes")
    print(f"Concurrent: {elapsed:.2f}s")


asyncio.run(main())
```

Очікуваний час: близько до найдовшого запиту (~3 секунди), а не сумарно (~8 секунд).

!!! tip "Одна сесія на всі запити"
    `aiohttp.ClientSession` створюється один раз і передається в усі корутини. Це дає переваги: повторне використання TCP-з'єднань (connection pooling), спільні cookies, однакові заголовки за замовчуванням.

### 3.2 Додайте послідовну версію для порівняння

У тому ж файлі `async_downloads.py` додайте послідовне виконання (через цикл `for` з `await`) для тих самих URL. Виміряйте час.

Зафіксуйте у звіті обидва результати:

| Виконання | Час | Запити |
|---|---|---|
| Послідовно | ~8с | 3+2+1+2 |
| Конкурентно | ~3с | max(3,2,1,2) |

---

## Частина 4: Порівняння з потоками

У [практиці 35](/ua/courses/programming-2sem/module3/35-threaded-downloads-practice/) ви завантажували ті самі URL через `threading`. Тепер порівняйте підходи.

У звіті складіть підсумкову таблицю для 4 запитів `httpbin.org/delay/*`:

| Підхід | Час | Кількість потоків ОС |
|---|---|---|
| Послідовно (requests) | ~8с | 1 |
| `threading` (4 потоки) | ~3с | 4 |
| `asyncio` + `aiohttp` | ~3с | 1 |

Коротко (2–3 речення) поясніть:

- чому `threading` і `asyncio` дають приблизно однакове прискорення для I/O-bound задач
- у чому перевага `asyncio` при масштабуванні до сотень / тисяч запитів

## Результат

Після виконання роботи здайте звіт зі скріншотами:

- **Частина 1:** вивід `async_basic.py` та скриншот із попередженням `RuntimeWarning` із пункту 1.2
- **Частина 2:** вивід `async_sequential.py` і `async_gather.py`, порівняльна таблиця часу
- **Частина 3:** вивід `async_downloads.py` з обома варіантами, таблиця часу
- **Частина 4:** підсумкова таблиця та пояснення
