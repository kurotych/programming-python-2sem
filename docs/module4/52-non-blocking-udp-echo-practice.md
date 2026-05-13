# 52. (П23) Мультиплексування TCP + UDP в одному `selectors`-циклі

## Передумови

- Прочитана [Лекція 44 — UDP](/ua/courses/programming-2sem/module4/44-udp-lecture/)
- Прочитана [Лекція 49 — Неблокуючі сокети](/ua/courses/programming-2sem/module4/49-non-blocking-sockets-lecture/)
- **Виконане [Практичне 22 (файл 50) — Неблокуючий TCP ехо-сервер](/ua/courses/programming-2sem/module4/50-non-blocking-echo-server-practice/)** — беремо його `echo_server_nb.py` як основу
- Утиліта `nc`

!!! info "Навіщо це практичне"
    На UDP саме по собі неблокуючий режим виграшу не дає — там нема з'єднань, які б одне одному заважали. Виграш з'являється тоді, коли в **одному event-loop** живуть сокети різних типів. Цього разу ви додасте UDP-сокет у той самий `selectors`, що вже обслуговує TCP-ехо з П22, — і побачите, що `selectors` нічого не змінює: API однаковий для обох транспортів.

!!! warning "Персоналізація"
    Залиште `STUDENT = "Imya Prizvysche"` з П22 і виведіть `print(f"Student: {STUDENT}")` при старті.

## Завдання

Скопіюйте `echo_server_nb.py` з П22 у файл `echo_server_mux.py` і **додайте** до нього UDP-гілку.

Після ваших змін один процес тримає:

- **TCP** на `127.0.0.1:9100` — поведінка **без змін** з П22: вітання `Hello from <STUDENT>!\n`, лічильник `n`, ехо `[<n>] <msg>\n`, акуратне закриття на `b""`.
- **UDP** на `127.0.0.1:9101` — `SOCK_DGRAM`, `setblocking(False)`, `bind`. На кожну отриману датаграму сервер відправляє назад `[udp] <msg>` тому ж відправникові (`sendto(data, addr)`).

Обидва сокети живуть в **одному** `selectors.DefaultSelector` і **одному** циклі `sel.select()`. Розрізнити їх у обробнику зручно через `key.data`:

```python
sel.register(server_tcp, selectors.EVENT_READ, data=None)        # TCP-listener
sel.register(server_udp, selectors.EVENT_READ, data="udp")       # UDP-сокет
# а TCP-клієнти, як і в П22, реєструються з dict у data
```

У головному циклі:

```python
for key, _mask in sel.select():
    if key.data is None:
        accept(...)                          # новий TCP-клієнт
    elif key.data == "udp":
        handle_udp(key.fileobj)              # UDP-датаграма(и)
    else:
        serve(...)                           # TCP-клієнт з П22
```



## Тестування. Обидва транспорти живі одночасно

Запустіть сервер. У двох окремих терміналах:

```text
$ nc 127.0.0.1 9100              # TCP
Hello from Imya Prizvysche!
hello
[1] hello
```

```text
$ nc -u 127.0.0.1 9101            # UDP
ping
[udp] ping
```

Обидва клієнти отримують відповіді **в одному й тому ж процесі сервера**.

## Результат

Скріншот з трьох вікон: лог сервера, TCP-клієнт (`nc`), UDP-клієнт (`nc -u`). У логу мають бути присутні події **обох** транспортів і `Student: <Ім'я>`.
