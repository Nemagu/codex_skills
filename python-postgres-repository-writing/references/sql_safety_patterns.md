# SQL Safety Patterns (psycopg)

## Parameterized Query

```python
from psycopg import sql
from psycopg.rows import DictRow, dict_row

query = sql.SQL("SELECT id, status FROM subscriptions WHERE tenant_id = %s AND status = %s")
params = (tenant_id, status)
cursor.execute(query, params)
```

## Dict rows

Настраивай соединение или пул с `row_factory=dict_row`. Типизируй результаты
`fetchone`/`fetchall` как `DictRow`/`list[DictRow]` и обращайся к данным по
именам колонок.

## Dynamic Filters

```python
from psycopg import sql

conditions = [sql.SQL("tenant_id = %s")]
params: list[object] = [tenant_id]

if status is not None:
    conditions.append(sql.SQL("status = %s"))
    params.append(status)

where_sql = sql.SQL(" AND ").join(conditions)
query = sql.SQL("SELECT id FROM subscriptions WHERE ") + where_sql
cursor.execute(query, params)
```

## Rules

- Не использовать f-строки для SQL.
- Всегда отделять SQL-шаблон и параметры.
- Выполнять SQL только в приватных методах репозитория.
- Для пакетных операций использовать `executemany`.
- Запрещено маскировать конфликты в SQL через `ON CONFLICT DO NOTHING/DO UPDATE`.
- Ошибки psycopg оборачивать в `AppInternalError`, обязательно сохраняя исходное исключение в `wrap_error`.
- Генерацию UUID и других несеквенциальных ID выполнять в коде приложения, не через SQL-запросы.
- В list/count сценариях избегать повторных тяжелых запросов без необходимости.
