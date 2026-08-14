# Жизненный цикл и контекст выполнения

## Содержание

- [Контракты](#контракты)
- [Сборка ресурсов](#сборка-ресурсов)
- [Интеграция с FastAPI](#интеграция-с-fastapi)
- [Готовность](#готовность)
- [Завершение](#завершение)

## Контракты

Задавать зависимости через минимальные интерфейсы:

```python
from collections.abc import Callable, Mapping
from contextlib import AbstractAsyncContextManager
from dataclasses import dataclass
from typing import Protocol


class ApplicationEntryPoints(Protocol):
    """Публичные операции приложения, доступные HTTP-слою."""


class Readiness(Protocol):
    @property
    def is_ready(self) -> bool: ...


@dataclass(frozen=True, slots=True)
class APIRuntimeContext:
    entry_points: ApplicationEntryPoints
    readiness: Readiness
    shared: Mapping[str, object]


RuntimeContextFactory = Callable[
    [],
    AbstractAsyncContextManager[APIRuntimeContext],
]
```

Если набор ресурсов известен, предпочитать явные типизированные поля вместо
`Mapping[str, object]`. Не указывать в presentation типы конкретных адаптеров.

## Сборка ресурсов

Composition root может собрать context factory через `AsyncExitStack`:

```python
from collections.abc import AsyncIterator
from contextlib import AsyncExitStack, asynccontextmanager


@asynccontextmanager
async def create_runtime_context() -> AsyncIterator[APIRuntimeContext]:
    async with AsyncExitStack() as stack:
        resource_a = await stack.enter_async_context(create_resource_a())
        resource_b = await stack.enter_async_context(create_resource_b(resource_a))
        entry_points = create_entry_points(resource_a, resource_b)
        readiness = create_readiness(resource_a, resource_b)
        yield APIRuntimeContext(
            entry_points=entry_points,
            readiness=readiness,
            shared={},
        )
```

Каждая resource factory возвращает async context manager либо предоставляет
явный async cleanup, который сразу регистрируется через
`stack.callback`/`stack.push_async_callback`.

## Интеграция с FastAPI

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request


def create_lifespan(
    context_factory: RuntimeContextFactory,
):
    @asynccontextmanager
    async def lifespan(app: FastAPI) -> AsyncIterator[None]:
        async with context_factory() as context:
            app.state.runtime_context = context
            yield

    return lifespan


def get_runtime_context(request: Request) -> APIRuntimeContext:
    context = getattr(request.app.state, "runtime_context", None)
    if not isinstance(context, APIRuntimeContext):
        raise RuntimeError("API runtime context is not initialized")
    return context
```

Если ASGI app обёрнута внешней middleware, request всё равно должен получать
state внутреннего FastAPI-приложения. Не копировать context в глобальную
переменную.

## Готовность

- До завершения context factory приложение не принимает запросы.
- Readiness отражает состояние обязательных ресурсов после startup.
- Изменяемое состояние readiness инкапсулировать в отдельном объекте; сам
  `APIRuntimeContext` остаётся неизменяемым.
- Не выполнять сетевой probe при каждом чтении `is_ready`, если требования не
  предписывают обратное.

## Завершение

- Освобождение происходит в обратном порядке регистрации.
- Частичный startup автоматически вызывает cleanup.
- Не перехватывать `CancelledError` общим `except`.
- Ошибка cleanup одного ресурса не должна отменять попытки `AsyncExitStack`
  закрыть остальные.
- Ошибку startup пробрасывать вызывающему серверу.
