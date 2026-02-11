# 2. (П1) Виконання завдань з Python та sqlite3

Виконувати лише за допомогою Python коду (без прямого втручання в БД)

Розробити сервіс, який зберігає інформацію про погоду в місті Луцьк (Lutsk) в sqlite бд
використовуючи [wttr.in](https://github.com/chubin/wttr.in) API.
Якщо wttr.in не працює, можна використати любий інший сервіс наприклад:  
`https://api.open-meteo.com/v1/forecast?latitude=50.7472&longitude=25.3254&current_weather=true`.  

Ваш сервіс повинен підтримувати два command line аргументи:

- service - запускає процес, який кожну хвилину зберігає інформацію про температуру в бд
- history - виводить в термінал збережену інформацію (історію) в форматі:  
```
Weather history from: <ВАШІ ім'я та прізвище>:  
<Час> <температура>  
...  
<Час> <температура>  
```
Час має бути відсортований у спадному порядку.

Приклад запиту для отримання актуальної температури (wttr.in):
```bash
curl "https://wttr.in/Lutsk?format=%t"
```

Приклад запиту для отримання актуальної температури (open-meteo):
```bash
curl "https://api.open-meteo.com/v1/forecast?latitude=50.7472&longitude=25.3254&current_weather=true"
```
