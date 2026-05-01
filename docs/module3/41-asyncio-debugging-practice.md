# 41. (П20) Налагодження асинхронного коду та робота з подійним циклом

## Передумови

- Прочитана [Лекція 40 — Проблеми виконання асинхронного коду, ручне керування подійним циклом та налагодження asyncio](/ua/courses/programming-2sem/module3/40-asyncio-debugging-lecture/)
- Python **3.11+**

!!! warning "Персоналізація"
    У кожному файлі оголосіть константу `STUDENT = "Imya Prizvysche"` (ваші дані латиницею) і виводьте її у першому рядку кожного запуску програми: `print(f"Student: {STUDENT}")`. На скріншотах у звіті повинно бути видно ваше ім'я.

## Завдання

Виконати дві невеликі програми, які демонструють базові навички налагодження asyncio і ручного керування event loop.

---

## Частина 1: знайти і виправити повільний крок

Файл `slow_step.py`.

Оголосіть три корутини:

| Корутина | Поведінка |
|---|---|
| `load()` | друкує `"load: start"`, чекає 0.3с через `asyncio.sleep`, повертає `"raw"` |
| `process()` | друкує `"process: start"`, викликає `time.sleep(1.2)`, повертає `"clean"` |
| `save()` | друкує `"save: start"`, чекає 0.5с через `asyncio.sleep`, нічого не повертає |

Оголосіть корутину `step(name, coro)`, яка:

1. запам'ятовує час через `time.perf_counter`;
2. чекає `coro` через `await`;
3. друкує `f"[{name}] {elapsed:.2f}s"`;
4. повертає результат корутини.

У `main` послідовно викличте всі три кроки через `step("load", ...)`, `step("process", ...)`, `step("save", ...)`. Запустіть через `asyncio.run(main(), debug=True)`.

Очікуваний вивід:

```text
load: start
[load] 0.30s
process: start
[process] 1.20s
save: start
[save] 0.50s
```

У debug-режимі asyncio додатково попередить про повільний callback у `process` (`took ... seconds`).

**Виправлення.** Зробіть так, щоб `process` не блокувала event loop: оберніть `time.sleep(1.2)` у `asyncio.to_thread`. Запустіть програму ще раз і переконайтесь, що попередження зникло.

У звіті у 1 реченні поясніть, чому до виправлення asyncio попереджав про повільний callback саме у `process`, а не у `load` чи `save`.

---

## Частина 2: ручне керування event loop

Файл `manual_loop.py`.

Оголосіть корутину `tick()`:

1. у нескінченному циклі друкує `f"tick {i}"` (де `i` — лічильник з 1);
2. чекає 1 секунду через `asyncio.sleep`;
3. у блоці `except asyncio.CancelledError` друкує `"tick: stopped"` і робить `raise`.

У `main` (звичайна синхронна функція):

1. створіть новий event loop через `asyncio.new_event_loop()`;
2. зареєструйте задачу через `loop.create_task(tick())`;
3. запустіть `loop.run_forever()` у блоці `try`;
4. у `except KeyboardInterrupt` скасуйте задачу через `task.cancel()` і викличте `loop.run_until_complete(task)` (огорніть у `try/except CancelledError`);
5. у `finally` викличте `loop.close()` і надрукуйте `"loop closed"`.

Запустіть програму, дайте їй надрукувати 3-4 тіки і натисніть `Ctrl+C`.

Очікуваний вивід (приблизно):

```text
tick 1
tick 2
tick 3
^Ctick: stopped
loop closed
```

У звіті у 1-2 реченнях поясніть, навіщо ми викликаємо `loop.run_until_complete(task)` після `task.cancel()` — що було б, якби ми просто закрили loop одразу.

## Результат

Після виконання роботи здайте звіт зі скріншотами:

- **Частина 1:** вивід `slow_step.py` до і після виправлення (з попередженням asyncio і без нього); пояснення, чому попереджало саме про `process`
- **Частина 2:** вивід `manual_loop.py` із зупинкою через `Ctrl+C`; пояснення ролі `run_until_complete` після скасування
