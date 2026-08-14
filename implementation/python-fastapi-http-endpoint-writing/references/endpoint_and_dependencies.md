# Обработчик и зависимости

## Содержание

- [Типизированная точка входа](#типизированная-точка-входа)
- [Узкая зависимость](#узкая-зависимость)
- [Обработчик](#обработчик)
- [Идентичность](#идентичность)
- [Фабрика маршрутизатора](#фабрика-маршрутизатора)
- [Ошибки](#ошибки)

## Типизированная точка входа

Выражать требуемую application-операцию минимальным callable-контрактом:

```python
from typing import Protocol


class CreateProjectEntryPoint(Protocol):
    async def __call__(
        self,
        command: CreateProjectCommand,
    ) -> ProjectResult: ...
```

Если application предоставляет use case с `execute`, dependency может возвращать
его конкретный публичный тип. Не добавлять wrapper только ради совпадения с
примером.

## Узкая зависимость

```python
from typing import Annotated

from fastapi import Depends


def get_create_project_entry_point(
    context: APIRuntimeContext = Depends(get_runtime_context),
) -> CreateProjectEntryPoint:
    return context.entry_points.create_project


CreateProjectOperation = Annotated[
    CreateProjectEntryPoint,
    Depends(get_create_project_entry_point),
]
```

Endpoint не получает весь runtime context и не ищет операцию по строковому ключу.

## Обработчик

```python
from fastapi import APIRouter, status


def create_project_router() -> APIRouter:
    router = APIRouter(prefix="/projects")

    @router.post(
        "",
        status_code=status.HTTP_201_CREATED,
        response_model=ProjectResponse,
        operation_id="createProject",
    )
    async def create_project(
        request: CreateProjectRequest,
        caller: Caller,
        operation: CreateProjectOperation,
    ) -> ProjectResponse:
        command = to_create_project_command(request, caller)
        result = await operation(command)
        return to_project_response(result)

    return router
```

Сигнатура и status являются примером. Реализовывать точный контракт.

## Идентичность

Dependency идентичности:

1. извлекает credential или trusted header;
2. проверяет transport format;
3. вызывает предусмотренный authentication-компонент;
4. возвращает immutable caller identity;
5. не выполняет business authorization.

Не передавать endpoint-у raw token, если ему нужна identity. Не доверять proxy
identity header без проверки источника на внешней границе.

## Фабрика маршрутизатора

При явном связывании фабрика может принять dependency callables:

```python
def create_project_router(
    operation_dependency: Callable[..., CreateProjectEntryPoint],
    caller_dependency: Callable[..., CallerIdentity],
) -> APIRouter:
    ...
```

FastAPI `Depends(operation_dependency)` регистрируется внутри фабрики. Сама
фабрика не создаёт use case, pool, repository или adapter.

Статический router допустим, если:

- зависимости остаются узкими и типизированными;
- нет module-level runtime graph;
- dependency overrides позволяют изолировать application entry points;
- импорт не выполняет I/O.

## Ошибки

Не ловить публичные application-ошибки локально. Transport dependency может
выбросить централизованную presentation-ошибку для отсутствующего/некорректного
transport input.

Не логировать ошибку в endpoint, включая вариант catch/log/re-raise.
