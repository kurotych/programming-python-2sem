# 39. (П19) Асинхронне створення, виконання та керування задачами в asyncio

## Передумови

- Прочитана [Лекція 38 — Керування асинхронними задачами та вимірювання продуктивності в asyncio](/ua/courses/programming-2sem/module3/38-asyncio-tasks-management-lecture/)
- Виконана [Практика 18 — Асинхронне виконання задач із використанням asyncio](/ua/courses/programming-2sem/module3/37-asyncio-tasks-practice/)
- Python **3.11+** (для `asyncio.timeout`)

!!! warning "Персоналізація"
    У кожному файлі оголосіть константу `STUDENT = "Imya Prizvysche"` (ваші дані латиницею) і виводьте її у першому рядку кожного запуску програми: `print(f"Student: {STUDENT}")`. На скріншотах у звіті повинно бути видно ваше ім'я.

## Завдання

Реалізувати дві невеликі програми, які демонструють керування асинхронними задачами: вибір способу очікування групи (`wait` vs `as_completed`) та робота з тайм-аутами і скасуванням. Код пишете самостійно — нижче чітко описано, які функції оголосити, що друкувати і який вивід має вийти.

---

## Частина 1: `wait` vs `as_completed`

Файл `waiting_strategies.py`.

Оголосіть корутину:

```python
async def fetch(name: str, delay: float) -> str
```

Поведінка:

1. на старті друкує `f"start {name}"`;
2. чекає `delay` секунд через `asyncio.sleep`;
3. на завершенні друкує `f"done {name}"`;
4. повертає рядок `f"data-{name}"`.

У `main` викличте її **двічі підряд** для трьох задач із затримками 3, 1 і 2 секунди:

**Блок A — `asyncio.wait` із `FIRST_COMPLETED`:**

- виміряйте час через `time.perf_counter`;
- після пробудження надрукуйте результат **найшвидшої** задачі у форматі `f"first done: {result}"`;
- скасуйте решту через `task.cancel()` і дочекайтесь їх через `await asyncio.gather(*pending, return_exceptions=True)`;
- надрукуйте `f"block A elapsed: {elapsed:.2f}s"`.

**Блок B — `asyncio.as_completed`:**

- виміряйте час;
- у циклі `for coro in asyncio.as_completed(...)` друкуйте `f"got: {await coro}"`;
- після циклу надрукуйте `f"block B elapsed: {elapsed:.2f}s"`.

Очікуваний час: блок A ≈ 1с, блок B ≈ 3с.

У звіті — повний вивід та 2 речення відповіді: чому `wait(FIRST_COMPLETED)` завершується за 1с, а `as_completed` — за 3с, хоча обидва запускають ті самі три задачі.

---

## Частина 2: тайм-аути та скасування

Файл `timeouts_and_cancel.py`. Дві незалежні демонстрації, обидві у тому самому `main`.

### Частина 2A — `asyncio.timeout`

Оголосіть три корутини без аргументів:

| Корутина | Затримка | Друкує на старті |
|---|---|---|
| `step_one` | 0.5с | `"step one: start"` |
| `step_two` | 0.5с | `"step two: start"` |
| `step_three` | 2с | `"step three: start"` |

Викличте їх **послідовно** через `await` всередині `async with asyncio.timeout(1.5):`. Перехопіть `TimeoutError` і надрукуйте `"timeout reached"`.

Очікуваний вивід:

```text
step one: start
step two: start
step three: start
timeout reached
```

### Частина 2B — ручне скасування

Оголосіть корутину `worker()`, що в нескінченному циклі друкує `f"tick {i}"` (де `i` — лічильник з 1) і чекає 1 секунду.

У блоці `try/except asyncio.CancelledError`:

- надрукуйте `f"worker: cleanup after {i} ticks"`;
- зробіть `raise`.

У `main` запустіть її через `create_task`, зачекайте 2.5 секунди, викличте `task.cancel()` і обробіть `CancelledError` від `await task` рядком `"main: task cancelled"`.

Очікуваний вивід:

```text
tick 1
tick 2
tick 3
worker: cleanup after 3 ticks
main: task cancelled
```

У звіті у 2 реченнях поясніть, **що зміниться, якщо прибрати `raise`** з блоку `except CancelledError`.

## Результат

Після виконання роботи здайте звіт зі скріншотами:

- **Частина 1:** повний вивід `waiting_strategies.py` з обома блоками; пояснення різниці часу
- **Частина 2:** вивід `timeouts_and_cancel.py`; пояснення наслідків прибирання `raise`
