# Модели и преобразования

## Содержание

- [Базовые модели](#базовые-модели)
- [Входное преобразование](#входное-преобразование)
- [Выходное преобразование](#выходное-преобразование)
- [Разные версии](#разные-версии)
- [Валидация](#валидация)
- [Ответ без тела](#ответ-без-тела)

## Базовые модели

Задать строгие внешние модели без зависимости от application:

```python
from pydantic import BaseModel, ConfigDict


class HTTPModel(BaseModel):
    model_config = ConfigDict(
        extra="forbid",
        frozen=True,
        validate_default=True,
    )


class CreateProjectRequest(HTTPModel):
    name: str


class ProjectResponse(HTTPModel):
    project_id: str
    name: str
    version: int
```

`strict=True` включать целиком либо на отдельных полях согласно JSON/HTTP
контракту. Не запрещать стандартное декодирование внешнего строкового формата,
если оно предусмотрено.

Коллекции в immutable-моделях объявлять как `tuple`. Defaults использовать только
если они являются частью внешнего контракта.

## Входное преобразование

Mapping объединяет только предусмотренные источники:

```python
def to_create_project_command(
    request: CreateProjectRequest,
    caller: CallerIdentity,
) -> CreateProjectCommand:
    return CreateProjectCommand(
        initiator_id=caller.subject_id,
        name=request.name,
    )
```

ID нового проекта отсутствует: application-операция получает его через порт.
Path/query/header значения добавлять параметрами mapper-а, если они входят в
application input.

Не использовать:

```python
CreateProjectCommand(**request.model_dump())
```

Явный mapping защищает application от добавления нового transport-поля.

## Выходное преобразование

```python
def to_project_response(
    result: ProjectResult,
) -> ProjectResponse:
    return ProjectResponse(
        project_id=str(result.project_id),
        name=result.name,
        version=result.version,
    )
```

Формат UUID, enum, datetime, decimal и bytes задавать явно. Не использовать
`from_attributes=True` как замену mapping.

## Разные версии

Каждая HTTP-версия имеет собственные модели и mapping:

```text
v1 request -> v1 mapper --+
                          +-> one application operation
v2 request -> v2 mapper --+
```

Общий helper допустим только для действительно одинаковой transport-семантики.
Не заставлять новую версию наследоваться от старой.

## Валидация

Transport model проверяет:

- обязательность и внешний тип;
- формат строки, длину и диапазон;
- допустимые внешние enum values;
- внешние cross-field ограничения.

Она не проверяет:

- существование aggregate;
- uniqueness;
- права вызывающей стороны;
- допустимость domain transition;
- состояние внешней системы.

Validators остаются чистыми и не выполняют I/O.

## Ответ без тела

Для `204 No Content` не объявлять фиктивную response-модель и не возвращать
`None` как JSON. Endpoint должен завершиться пустым response способом,
соответствующим FastAPI и контракту.
