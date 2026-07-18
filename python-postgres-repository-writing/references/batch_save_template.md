# Batch Save Template

```python
def save(self, aggregate: Aggregate) -> None:
    self.batch_save([aggregate])


def batch_save(self, aggregates: list[Aggregate]) -> None:
    to_create: list[Aggregate] = []
    to_update: list[Aggregate] = []

    for aggregate in aggregates:
        if aggregate.version == 1:
            to_create.append(aggregate)
        else:
            to_update.append(aggregate)

    if to_create:
        self._batch_create(to_create)
    if to_update:
        self._batch_update(to_update)
```

## Notes

- Не запускай SQL на каждый агрегат внутри цикла.
- Все SQL-вызовы выполняй только в приватных методах (`_batch_create`, `_batch_update`).
- В `_batch_create/_batch_update` применяй `executemany`.
- Не используй `ON CONFLICT DO NOTHING/DO UPDATE` для скрытия конфликтов.
- Конфликт уникальности должен приводить к `AppInternalError`, содержащей исходное исключение psycopg в `wrap_error`.
- Для UUID и других несеквенциальных ID генерируй идентификатор в коде до вызова batch-вставки.
