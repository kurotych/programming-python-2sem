# 54. (П24) Ехо-сервер на `asyncio`

## Передумови

- Прочитана [Лекція 53 — Архітектура ехо-сервера на `asyncio`](/ua/courses/programming-2sem/module4/53-asyncio-echo-server-lecture/)
- Виконане [Практичне 22 (файл 50) — Неблокуючий ехо-сервер на `selectors`](/ua/courses/programming-2sem/module4/50-non-blocking-echo-server-practice/)

## Завдання

Зробіть **те саме**, що в [П22](/ua/courses/programming-2sem/module4/50-non-blocking-echo-server-practice/) — ту саму поведінку, той самий формат логу, ті самі тести — але на `asyncio` замість `selectors`.

Файл — `echo_server_asyncio.py`. Замість ручного циклу `sel.select()` використайте `asyncio.start_server`, а обробник одного клієнта напишіть як `async def handle(reader, writer)`. Стан клієнта (лічильник `n`) — локальна змінна корутини, **не** глобальний словник.

## Результат

Той самий, що в П22: скріншот сесії з трьох одночасних `nc`-клієнтів і скріншот логу сервера з `Student: <Імʼя>`.
