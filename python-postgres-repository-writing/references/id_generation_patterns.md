# ID Generation Patterns

## Rule

- Если идентификатор не последовательный (не SERIAL/IDENTITY), например UUID, генерируй его в коде приложения до обращения к БД.
- Не делай отдельный SQL-запрос в БД для генерации такого ID.

## UUID Example

```python
from uuid6 import uuid7


entity_id = uuid7()
aggregate = Aggregate(id=entity_id, ...)
repository.save(aggregate)
```

## Notes

- Версию UUID выбирай по архитектурному стандарту проекта (например, uuid7).
- Генерация ID в приложении упрощает тестирование и уменьшает связность с БД.
- Для SERIAL/IDENTITY стратегия иная: ID допускается получать из БД при вставке.
