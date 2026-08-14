# Фабрика приложения

## Контракт

Фабрика получает только готовые параметры и компоненты сборки:

```python
from collections.abc import Callable, Sequence
from contextlib import AbstractAsyncContextManager
from dataclasses import dataclass

from fastapi import APIRouter, FastAPI
from starlette.middleware import Middleware
from starlette.types import ASGIApp


ErrorHandlersInstaller = Callable[[FastAPI], None]
Lifespan = Callable[[FastAPI], AbstractAsyncContextManager[None]]


@dataclass(frozen=True, slots=True)
class APIParameters:
    title: str
    root_path: str
    openapi_url: str | None
    docs_url: str | None
    redoc_url: str | None


@dataclass(frozen=True, slots=True)
class APIComponents:
    routers: Sequence[APIRouter]
    middleware: Sequence[Middleware]
    install_error_handlers: ErrorHandlersInstaller
    lifespan: Lifespan
    outer_wrappers: Sequence[Callable[[ASGIApp], ASGIApp]] = ()
```

Имена являются примером. Не создавать универсальный контейнер, если проект уже
имеет эквивалентные типизированные структуры.

## Сборка

```python
def create_api_app(
    parameters: APIParameters,
    components: APIComponents,
) -> ASGIApp:
    app = FastAPI(
        title=parameters.title,
        root_path=parameters.root_path,
        openapi_url=parameters.openapi_url,
        docs_url=parameters.docs_url,
        redoc_url=parameters.redoc_url,
        lifespan=components.lifespan,
        middleware=list(components.middleware),
    )
    components.install_error_handlers(app)
    for router in components.routers:
        app.include_router(router)

    result: ASGIApp = app
    for wrapper in reversed(components.outer_wrappers):
        result = wrapper(result)
    return result
```

`outer_wrappers` использовать только для middleware, которая по требованиям
должна оборачивать встроенный `ServerErrorMiddleware`, например для глобального
CORS. Порядок последовательности описывать как внешний → внутренний.

## Правила

- Один вызов возвращает новое приложение.
- Фабрика не читает конфигурацию и не открывает ресурсы.
- Routers и installers передаются явно.
- Параметры не зависят от модели внешнего settings.
- Не импортировать конкретные application handlers и infrastructure factories
  внутри универсальной фабрики.
- Не хранить собранное приложение в module-level singleton, если требуется
  factory mode.

## Проверки

- Два вызова фабрики не разделяют изменяемое состояние.
- Состав routes и middleware детерминирован.
- OpenAPI/docs URLs соответствуют параметрам.
- Outer wrappers расположены в согласованном порядке.
- Создание приложения не вызывает I/O.
