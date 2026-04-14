# 33. (П16) Створення процесів та потоків: threading і multiprocessing

## Передумови

- Прочитана [Лекція 32 — Основи асинхронного програмування та конкурентності в Python](/ua/courses/programming-2sem/module3/32-async-basics-lecture/)
- Python 3.10+

!!! warning "Персоналізація"
    У кожному файлі оголосіть константу `STUDENT = "Ім'я Прізвище"` (ваші дані) і виводьте її у першому рядку кожного запуску програми: `print(f"Студент: {STUDENT}")`. На скріншотах у звіті повинно бути видно ваше ім'я.

## Завдання

Освоїти базові інструменти конкурентності Python: створювати потоки та процеси, синхронізувати доступ до спільних даних, вимірювати продуктивність.

---

## Частина 1: Перші потоки з `threading`

### 1.1 Запуск кількох потоків

Створіть файл `threads_basic.py`. Напишіть функцію `download(url: str, delay: float) -> None`, яка імітує завантаження файлу: виводить `"Завантаження: <url>"`, чекає `delay` секунд (`time.sleep`), виводить `"Готово: <url>"`.

Запустіть три потоки для трьох різних URL із затримками 3, 1 і 2 секунди — **спочатку послідовно**, потім **через `threading.Thread`**. Виміряйте час кожного варіанту за допомогою `time.perf_counter`.

```python
import threading
import time


def download(url: str, delay: float) -> None:
    print(f"Завантаження: {url}")
    time.sleep(delay)
    print(f"Готово: {url}")


urls = [
    ("https://example.com/file1", 3),
    ("https://example.com/file2", 1),
    ("https://example.com/file3", 2),
]

# Послідовне виконання
start = time.perf_counter()
for url, delay in urls:
    download(url, delay)
print(f"Послідовно: {time.perf_counter() - start:.2f}с\n")

# TODO: замінити на потокове виконання
```

Очікуваний результат: послідовне виконання ~6 секунд, потокове ~3 секунди.

### 1.2 Передача аргументів і `name`

Доповніть `threads_basic.py`. Дайте кожному потоку ім'я (`name="downloader-1"` тощо) і виводьте його в повідомленнях функції `download` через `threading.current_thread().name`.

Очікуваний вивід (порядок рядків може різнитись):

```
[downloader-1] Завантаження: https://example.com/file1
[downloader-2] Завантаження: https://example.com/file2
[downloader-3] Завантаження: https://example.com/file3
[downloader-2] Готово: https://example.com/file2
[downloader-3] Готово: https://example.com/file3
[downloader-1] Готово: https://example.com/file1
Потоки: 3.01с
```

---

## Частина 2: Перші процеси з `multiprocessing`

### 2.1 Запуск процесів

Створіть файл `processes_basic.py`. Напишіть CPU-bound функцію `compute(n: int) -> int`, яка рахує суму квадратів від 0 до n: `sum(i * i for i in range(n))`. Поверніть результат і надрукуйте його разом з іменем процесу.

```python
import multiprocessing
import time


def compute(n: int) -> None:
    result = sum(i * i for i in range(n))
    name = multiprocessing.current_process().name
    print(f"[{name}] sum of squares(0..{n}) = {result}")


if __name__ == "__main__":
    tasks = [10_000_000, 8_000_000, 12_000_000]

    # Послідовно
    start = time.perf_counter()
    for n in tasks:
        compute(n)
    print(f"Послідовно: {time.perf_counter() - start:.2f}с\n")

    # TODO: через multiprocessing.Process
```

!!! warning "`if __name__ == '__main__':`"
    Цей захист **обов'язковий** для `multiprocessing` на Windows і macOS. Без нього запуск нових процесів рекурсивно породить нові процеси і програма зависне.

Запустіть обидва варіанти. Переконайтеся, що процеси дають прискорення на CPU-bound задачі (на відміну від потоків).

## Результат

Після виконання роботи здайте звіт зі скріншотами:

- **Частина 1:** вивід потокового завантаження з іменами потоків; порівняння часу послідовного і потокового виконання
- **Частина 2:** порівняльна таблиця часу (послідовно / потоки / процеси) для CPU-bound задачі
