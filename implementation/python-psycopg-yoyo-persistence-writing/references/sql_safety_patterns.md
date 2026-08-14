# Безопасная сборка SQL

## Данные

Передавать значения только параметрами:

```python
query = SQL(
    "SELECT id, status FROM subscriptions "
    "WHERE tenant_id = %s AND status = %s"
)
await cursor.execute(query, (tenant_id, status))
```

Не использовать f-строки, ручное экранирование и конкатенацию данных.

## Идентификаторы

Собирать имена схем, таблиц и колонок через `Identifier`:

```python
query = SQL("SELECT id FROM {}.{} WHERE tenant_id = %s").format(
    Identifier(schema_name),
    Identifier(table_name),
)
```

- Использовать общий `StrEnum` фактических таблиц.
- Сверять enum с миграциями.
- Передавать schema и table разными `Identifier`.
- Выбирать динамические поля и направления сортировки из закрытого mapping.
- Не считать enum/Literal верхнего слоя готовым SQL.
- Не дублировать имена таблиц в конкретных репозиториях.

## Размещение запросов

- Статический запрос хранить рядом с репозиторием как `SQL`-объект или рядом с
  приватным методом.
- Выносить длинный запрос в модуль того же компонента.
- Не создавать глобальный каталог SQL и универсальный query builder.
- Выбирать колонки явно, без `SELECT *`.
- Выносить фрагмент только при реальном переиспользовании.

## Типы SQL-композиции

- `SQL` использовать для статического SQL-литерала.
- `Composable` использовать для параметра, принимающего `SQL`, `Identifier`,
  `Composed` или другой безопасный фрагмент psycopg.
- `Composed` использовать для результата `SQL.format()`, `SQL.join()` и сложения
  SQL-фрагментов.

```python
from psycopg.sql import SQL, Composable, Composed, Identifier


def select_query(condition: Composable) -> Composed:
    return SQL("SELECT id FROM {} WHERE {}").format(
        Identifier("items"),
        condition,
    )
```

Не аннотировать результат `format()` или `join()` как `SQL`: эти методы
возвращают `Composed`.

## Курсоры и типы

- Типизировать async-код через `AsyncConnection[DictRow]` и
  `AsyncCursor[DictRow]`.
- Настраивать `row_factory=dict_row`.
- Читать колонки по именам, не использовать `TupleRow` и числовые индексы.
- Закрывать cursor в приватной DB-операции.
- Не возвращать cursor или ленивый iterator наружу.
- Server-side cursor использовать только для заданного streaming-контракта.
