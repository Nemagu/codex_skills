# API Worker Template

Эталонная сборка HTTP API-воркера для Python-сервиса с гексагональной архитектурой. Все файлы — в `src/presentation/api/`, за исключением фрагмента entrypoint-а. Имена `<ServiceError>`/`PostgresConnectionManager` подстраивай под проект.

## `src/presentation/api/server.py`

```python
"""Модуль `presentation/api/server.py` слоя представления."""

from logging import getLogger

import uvicorn
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from infrastructure.config import APIWorkerSettings
from presentation.api.dependencies import APILifespan
from presentation.api.error_handler import setup_error_handler
from presentation.api.middlewares import (
    LoggingMiddleware,
    PerformanceMiddleware,
    RequestIDMiddleware,
)
from presentation.api.routers import main_router

logger = getLogger(__name__)


class APIWorker:
    """Инициализирует и запускает API-процесс FastAPI."""

    def __init__(self, settings: APIWorkerSettings) -> None:
        self.settings = settings
        logger.info("init api worker lifespan")
        lifespan = APILifespan(self.settings)

        logger.info("init fastapi app")
        self.app = FastAPI(lifespan=lifespan.lifespan)

        self.app.add_middleware(LoggingMiddleware)
        self.app.add_middleware(
            PerformanceMiddleware,
            header_name=self.settings.fastapi.process_time_header_name,
            header_name_ms=self.settings.fastapi.process_time_ms_header_name,
        )
        self.app.add_middleware(
            RequestIDMiddleware,
            header_name=self.settings.fastapi.request_id_header_name,
        )
        self.app.add_middleware(
            CORSMiddleware,
            allow_origins=self.settings.fastapi.cors.allow_origins,
            allow_origin_regex=self.settings.fastapi.cors.allow_origin_regex,
            allow_credentials=self.settings.fastapi.cors.allow_credentials,
            allow_methods=self.settings.fastapi.cors.allow_methods,
            allow_headers=self.settings.fastapi.cors.allow_headers,
            expose_headers=self.settings.fastapi.cors.expose_headers,
            max_age=self.settings.fastapi.cors.max_age,
        )

        setup_error_handler(self.app)
        self.app.include_router(main_router)

    def run(self) -> None:
        config = uvicorn.Config(
            app=self.app,
            host=self.settings.uvicorn.host,
            port=self.settings.uvicorn.port,
            reload=self.settings.uvicorn.reload,
            loop=self.settings.uvicorn.loop,
            workers=self.settings.uvicorn.workers,
            access_log=False,
        )
        server = uvicorn.Server(config)
        logger.info("start api server")
        server.run()
```

## `src/presentation/api/dependencies/lifespan.py`

```python
"""Модуль `presentation/api/dependencies/lifespan.py` слоя представления."""

from contextlib import asynccontextmanager
from typing import AsyncGenerator

from fastapi import FastAPI

from infrastructure.config import APIWorkerSettings
from infrastructure.db.postgres import PostgresConnectionManager


class APILifespan:
    def __init__(self, settings: APIWorkerSettings) -> None:
        self._settings = settings

    @asynccontextmanager
    async def lifespan(self, app: FastAPI) -> AsyncGenerator:
        connection_manager = PostgresConnectionManager(self._settings.db)
        await connection_manager.init()
        app.state.fastapi_settings = self._settings.fastapi
        app.state.db_connection_manager = connection_manager
        yield
        await connection_manager.close()
```

## `src/presentation/api/error_handler.py`

```python
"""Модуль `presentation/api/error_handler.py` слоя представления."""

from fastapi import FastAPI, Request, status
from fastapi.encoders import jsonable_encoder
from fastapi.responses import JSONResponse

from application.errors import AppError, AppInternalError, AppNotFoundError
from domain.errors import DomainError, EntityPolicyError


def setup_error_handler(app: FastAPI) -> None:
    @app.exception_handler(AppError)
    async def handle_app_error(request: Request, exc: AppError) -> JSONResponse:
        error_context: dict[str, object] = {
            "detail": exc.msg,
            "action": exc.action,
            "struct_name": None,
            "data": exc.data,
        }
        content = {"detail": exc.msg, "data": exc.data}
        if isinstance(exc, AppInternalError):
            status_code = status.HTTP_500_INTERNAL_SERVER_ERROR
            if exc.wrap_error is not None:
                error_context["wrap_error"] = str(exc.wrap_error)
        elif isinstance(exc, AppNotFoundError):
            status_code = status.HTTP_404_NOT_FOUND
        else:
            status_code = status.HTTP_400_BAD_REQUEST
        request.state.error_context = error_context
        return JSONResponse(status_code=status_code, content=jsonable_encoder(content))

    @app.exception_handler(DomainError)
    async def handle_domain_error(request: Request, exc: DomainError) -> JSONResponse:
        request.state.error_context = {
            "detail": exc.msg,
            "action": None,
            "struct_name": exc.struct_name,
            "data": exc.data,
        }
        content = {"detail": exc.msg, "data": exc.data}
        if isinstance(exc, EntityPolicyError):
            status_code = status.HTTP_403_FORBIDDEN
        else:
            status_code = status.HTTP_400_BAD_REQUEST
        return JSONResponse(status_code=status_code, content=jsonable_encoder(content))
```

## `src/presentation/api/routers/__init__.py`

```python
"""Модуль `presentation/api/routers/__init__.py` слоя представления."""

from fastapi import APIRouter

from presentation.api.routers.health import health_router
from presentation.api.routers.public import public_router

main_router = APIRouter()

main_router.include_router(health_router, prefix="/health")
main_router.include_router(public_router)

__all__ = ["main_router"]
```

## `src/presentation/api/routers/health.py`

```python
"""Модуль `presentation/api/routers/health.py` слоя представления."""

from fastapi import APIRouter

health_router = APIRouter(tags=["health"])


@health_router.get("")
def health_check() -> dict:
    return {"status": "ok"}
```

## `src/presentation/api/routers/public/__init__.py`

```python
"""Модуль `presentation/api/routers/public/__init__.py` слоя представления."""

from fastapi import APIRouter

from presentation.api.routers.public.v1 import v1_router

public_router = APIRouter(prefix="/public", tags=["Public"])
public_router.include_router(v1_router)

__all__ = ["public_router"]
```

## `src/presentation/api/routers/public/v1/__init__.py`

```python
"""Модуль `presentation/api/routers/public/v1/__init__.py` слоя представления."""

from fastapi import APIRouter

from presentation.api.routers.public.v1.<aggregate> import <aggregate>_router

v1_router = APIRouter(prefix="/v1", tags=["V1"])

v1_router.include_router(<aggregate>_router)

__all__ = ["v1_router"]
```

## `src/presentation/api/dependencies/__init__.py`

```python
"""Модуль `presentation/api/dependencies/__init__.py` слоя представления."""

from presentation.api.dependencies.db import db_unit_of_work
from presentation.api.dependencies.lifespan import APILifespan
from presentation.api.dependencies.user_id_extractor import user_id_extractor

__all__ = ["APILifespan", "db_unit_of_work", "user_id_extractor"]
```

## `src/presentation/api/dependencies/db.py`

```python
"""Модуль `presentation/api/dependencies/db.py` слоя представления."""

from fastapi import Request

from infrastructure.db.postgres import PostgresUnitOfWork


async def db_unit_of_work(request: Request) -> PostgresUnitOfWork:
    async with request.app.state.db_connection_manager.connection() as conn:
        return PostgresUnitOfWork(conn)
```

## `src/presentation/api/dependencies/user_id_extractor.py`

```python
"""Модуль `presentation/api/dependencies/user_id_extractor.py` слоя представления."""

from uuid import UUID

from fastapi import HTTPException, Request, status

from infrastructure.config import FastAPISettings


async def user_id_extractor(request: Request) -> UUID:
    settings: FastAPISettings = request.app.state.fastapi_settings
    raw = request.headers.get(settings.user_id_header_name)
    if raw is None:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "id пользователя не передан")
    try:
        return UUID(raw)
    except (TypeError, ValueError):
        raise HTTPException(
            status.HTTP_401_UNAUTHORIZED, "некорректный id пользователя"
        )
```

## Endpoint pattern (фрагмент)

```python
# src/presentation/api/routers/public/v1/<aggregate>.py
from uuid import UUID

from fastapi import APIRouter, Depends

from application.queries.public.<aggregate> import (
    <Aggregate>LastVersionQuery,
    <Aggregate>LastVersionUseCase,
)
from infrastructure.db.postgres import PostgresUnitOfWork
from presentation.api.dependencies import db_unit_of_work, user_id_extractor
from presentation.api.models.<aggregate> import <Aggregate>SimpleResponse

<aggregate>_router = APIRouter(prefix="/<aggregates>", tags=["<Aggregate>"])


@<aggregate>_router.get("/{<aggregate>_id}")
async def get_<aggregate>(
    <aggregate>_id: UUID,
    user_id: UUID = Depends(user_id_extractor),
    uow: PostgresUnitOfWork = Depends(db_unit_of_work),
) -> <Aggregate>SimpleResponse:
    query = <Aggregate>LastVersionQuery(initiator_id=user_id, <aggregate>_id=<aggregate>_id)
    use_case = <Aggregate>LastVersionUseCase(uow)
    dto = await use_case.execute(query)
    return <Aggregate>SimpleResponse.from_dto(dto)
```

## `src/main.py` (фрагмент)

```python
mode = getenv("MODE", "api")
match mode:
    case "api":
        api_settings = APIWorkerSettings()
        apply_db_migrations(api_settings.db)
        APIWorker(api_settings).run()
    case "nats_consumer":
        ...
```

## Заметки по адаптации

- Подтипы исключений (`AppInternalError`, `AppNotFoundError`, `EntityPolicyError`) свои в каждом сервисе — добавляй ветки `isinstance` под локальную иерархию. Базовые корни — `AppError` и `DomainError`.
- `PostgresConnectionManager` — пример shared-ресурса. Другие (NATS-клиент, Redis-пул и т.п.) добавляются аналогично: init на startup, close на shutdown в обратном порядке.
- Все CORS-параметры — `self.settings.fastapi.cors.*`. Если `CORSSettings` ещё не заведён в `infrastructure/config/fastapi.py` — заведи через [python-pydantic-settings-config-writing] (`references/tech_examples.md` → FastAPI / Uvicorn) **до** правки `server.py`, иначе `add_middleware(CORSMiddleware, ...)` упадёт на чтении несуществующих полей.
- Замени `<aggregate>` / `<Aggregate>` / `<aggregates>` на реальные имена агрегатов проекта во всех файлах (`routers/public/v1/...`, endpoint pattern, models).
- `db_unit_of_work` — Postgres-вариант через `PostgresUnitOfWork`. Если используется другой UnitOfWork — поменяй тип, шаблон тот же: контекст-менеджер connection-manager-а из `app.state`.
- `user_id_extractor` — простой `X-User-Id`-вариант. Для JWT/OAuth добавь отдельную Depends-функцию рядом и используй её через `Depends(...)` в нужных эндпоинтах.
- Миграции (`apply_db_migrations`) применяются в `main.py` до старта app, не в lifespan — иначе несколько воркеров будут гоняться за блокировкой.
