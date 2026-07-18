# Read Repository Template (count + pagination + zero-count short-circuit)

```python
from dataclasses import dataclass

from psycopg.rows import DictRow, dict_row
from psycopg import sql


@dataclass(frozen=True)
class Page:
    items: list
    total: int


def list_items(self, filt: Filter) -> Page:
    conditions = [sql.SQL("tenant_id = %s")]
    params: list[object] = [filt.tenant_id]

    if filt.status is not None:
        conditions.append(sql.SQL("status = %s"))
        params.append(filt.status)

    where_sql = sql.SQL(" AND ").join(conditions)

    count_query = sql.SQL("SELECT COUNT(*) FROM subscriptions WHERE ") + where_sql
    total = self._fetch_count(count_query, params)
    if total == 0:
        return Page(items=[], total=0)

    data_query = (
        sql.SQL("SELECT id, status, updated_at FROM subscriptions WHERE ")
        + where_sql
        + sql.SQL(" ORDER BY updated_at DESC LIMIT %s OFFSET %s")
    )
    data_params = [*params, filt.limit, filt.offset]
    rows = self._fetch_rows(data_query, data_params)
    return Page(items=[self._model_to_domain(row) for row in rows], total=total)
```

Соединение или пул для этого репозитория должен использовать
`row_factory=dict_row`, а `_fetch_rows` должен возвращать `list[DictRow]`.

## Rules

- Сначала `COUNT`, затем select данных только если count больше нуля.
- Использовать общий набор условий для count и data-query.
- Ограничивать `limit` и валидировать `offset` до выполнения запроса.
- Добавлять явный `ORDER BY` для стабильной пагинации.
- Обращаться к значениям `DictRow` по именам колонок, не по числовым индексам.
