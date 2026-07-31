# Пагинация и фильтрация

## Содержание

- [Вход](#вход)
- [Закрытые значения](#закрытые-значения)
- [Ответ limit/offset](#ответ-limitoffset)
- [Курсорная пагинация](#курсорная-пагинация)
- [Пустой результат](#пустой-результат)
- [Проверки](#проверки)

## Вход

Имена и ограничения параметров определяются HTTP-контрактом:

```python
from typing import Annotated

from fastapi import Query


Limit = Annotated[int, Query(ge=1, le=100)]
Offset = Annotated[int, Query(ge=0)]
```

Mapper создаёт application query явно:

```python
def to_list_projects_query(
    caller: CallerIdentity,
    limit: int,
    offset: int,
    statuses: tuple[ProjectStatusFilter, ...] | None,
) -> ListProjectsQuery:
    return ListProjectsQuery(
        initiator_id=caller.subject_id,
        limit=limit,
        offset=offset,
        statuses=(
            tuple(status.value for status in statuses)
            if statuses is not None
            else None
        ),
    )
```

Если application использует отдельный paginator DTO, mapper создаёт его вместо
плоских полей.

## Закрытые значения

Фильтры и сортировку описывать внешними `StrEnum`/`Literal`. Их значения являются
HTTP-контрактом и не обязаны совпадать с именами DB columns или domain enum.

```python
from enum import StrEnum


class ProjectSort(StrEnum):
    CREATED_AT = "created_at"
    NAME = "name"
```

Mapper переводит внешний sort key в заданный application input. Не передавать
строку в SQL и не принимать произвольное имя поля.

## Ответ limit/offset

Сохранять пример результата пагинации:

```python
class ProjectPageResponse(HTTPModel):
    items: tuple[ProjectResponse, ...]
    total: int
    limit: int
    offset: int


def to_project_page_response(
    result: ProjectPageResult,
    *,
    limit: int,
    offset: int,
) -> ProjectPageResponse:
    return ProjectPageResponse(
        items=tuple(
            to_project_response(item)
            for item in result.items
        ),
        total=result.total,
        limit=limit,
        offset=offset,
    )
```

Не добавлять `total`, limit/offset echo или links, если их нет во внешнем
контракте.

## Курсорная пагинация

Cursor считать opaque transport value:

- валидировать только внешний формат;
- не раскрывать DB offset или внутренний ключ без требования;
- передавать декодированное значение application query через отдельный mapper;
- следующий cursor формировать из application result по согласованному контракту.

Не сочетать cursor и offset в одном endpoint без заданной семантики.

## Пустой результат

Обычно пустой список является успешным результатом с пустым `items`. `404`
использовать только если HTTP-контракт различает отсутствие коллекции как
отдельный исход.

## Проверки

- Минимум, максимум и неверный тип параметров.
- Несколько значений фильтра и закрытая сортировка.
- Точный application query.
- Пустая и непустая страница.
- Total и cursor только при их наличии в контракте.
