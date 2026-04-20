# 35. (П17) Багатопоточне завантаження даних із мережі

## Передумови

- Прочитана [Лекція 32 — Основи асинхронного програмування та конкурентності в Python](/ua/courses/programming-2sem/module3/32-async-basics-lecture/)
- Прочитана [Лекція 34 — Асинхронність усередині Python: GIL, подієвий цикл і сокети](/ua/courses/programming-2sem/module3/34-event-loop-sockets-lecture/)
- Встановлена бібліотека `requests`: `pip install requests`

!!! warning "Персоналізація"
    У файлі оголосіть константу `STUDENT = "Imya Prizvysche"` (ваші дані латиницею) і виводьте її у першому рядку кожного запуску програми: `print(f"Student: {STUDENT}")`. На скріншотах у звіті повинно бути видно ваше ім'я.

## Завдання

### 1.1 Послідовне завантаження (baseline)

Створіть файл `threaded_download.py`. Напишіть функцію `download(url: str) -> None`, яка виконує HTTP GET-запит і друкує статус-код та розмір відповіді разом з іменем поточного потоку.

```python
import threading
import time
import requests

STUDENT = "Imya Prizvysche"


def download(url: str) -> None:
    name = threading.current_thread().name
    print(f"[{name}] start: {url}")
    response = requests.get(url)
    print(f"[{name}] done: {url} -> {response.status_code}, {len(response.content)} bytes")


if __name__ == "__main__":
    print(f"Student: {STUDENT}")

    urls = [
        "https://httpbin.org/delay/3",
        "https://httpbin.org/delay/2",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/2",
    ]

    # Послідовне завантаження
    start = time.perf_counter()
    for url in urls:
        download(url)
    print(f"Sequential: {time.perf_counter() - start:.2f}s")
```

Завдання: Доповніть `threaded_download.py`: виконайте ті ж запити паралельно через `threading.Thread`. Кожному потоку дайте ім'я `"downloader-N"`.


## Результат

Після виконання роботи здайте звіт зі скріншотами
