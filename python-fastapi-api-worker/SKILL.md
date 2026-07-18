---
name: python-fastapi-api-worker
description: Используй при проектировании или реализации HTTP API-воркера на FastAPI в Python-сервисах с гексагональной архитектурой. Триггеры — сборка `APIWorker`-класса, регистрация middleware (включая CORS как outermost), error handlers, lifespan для shared-ресурсов, маршрутизация роутеров, запуск через uvicorn. Не применять для background-воркеров (NATS-consumer/publisher, periodic loops) и для реализации конкретных endpoint-ов.
---

# Python FastAPI API Worker

Скил про сборку HTTP API-воркера на FastAPI: класс `APIWorker`, который инкапсулирует FastAPI-приложение, навешивает обязательный набор middleware (включая CORS), error handlers, lifespan для shared-ресурсов, маршрутизирует роутеры и запускает Uvicorn. За структурой конфига отвечает [python-pydantic-settings-config-writing], за CI-обвязку — [python-gitlab-ci-pipeline]; здесь только сам воркер.

## Quick Start

1. Создай класс `APIWorker` в `src/presentation/api/server.py`, принимающий `APIWorkerSettings` в `__init__`.
2. В `__init__` собери приложение через `FastAPI(lifespan=...)`, передав в lifespan `APILifespan` (`presentation/api/dependencies/lifespan.py`).
3. Зарегистрируй обязательный набор middleware в строго определённом порядке (см. раздел «Middleware»). **CORS — последним**, чтобы оказаться outermost.
4. Подключи `setup_error_handler(app)` (см. «Error Handlers»).
5. Подключи `main_router` через `app.include_router(...)`.
6. В методе `run()` собери `uvicorn.Config(loop="uvloop", access_log=False, ...)` и запусти `uvicorn.Server(config).run()`.
7. Точку входа `src/main.py` свяжи с воркером через ветку `MODE=api`.

Готовый эталон — [api_worker_template.md](references/api_worker_template.md). Бери целиком, адаптируй имена.

## When to Apply

### Триггеры активации

- Создание или правка `src/presentation/api/server.py` (`APIWorker` или эквивалент).
- Добавление новой middleware в `src/presentation/api/middlewares/`.
- Изменение error handler-ов в `src/presentation/api/error_handler.py`.
- Правка `APILifespan` (shared-ресурсы, попадающие в `app.state`).
- Регистрация новых верхнеуровневых роутеров в `src/presentation/api/routers/__init__.py`.
- Запуск через Uvicorn (`access_log`, `loop`, `workers`).

### Анти-триггеры

- Background-воркеры на FastAPI lifespan (polling-loops, NATS-consumer/publisher, periodic-jobs) — это другой паттерн, не описан здесь.
- Реализация конкретных endpoint-ов и бизнес-логики роутеров.
- Структура `infrastructure/config/` (включая `CORSSettings`/`FastAPISettings`/`UvicornSettings`) — это [python-pydantic-settings-config-writing].
- CI/CD-обвязка и сборка образа — [python-gitlab-ci-pipeline].

## Package Structure

```
src/presentation/api/
├── server.py                   ← APIWorker — собирает FastAPI-app
├── error_handler.py            ← setup_error_handler — маппинг исключений в HTTP-ответы
├── dependencies/
│   ├── __init__.py             ← re-export Depends-функций и APILifespan
│   ├── lifespan.py             ← APILifespan — инициализирует shared-ресурсы в app.state
│   ├── db.py                   ← Depends-провайдер UnitOfWork из app.state.db_connection_manager
│   └── user_id_extractor.py    ← Depends-функция авторизации (читает заголовок, возвращает UUID)
├── middlewares/
│   ├── __init__.py             ← re-export классов middleware
│   ├── logging.py              ← LoggingMiddleware
│   ├── performance.py          ← PerformanceMiddleware
│   └── request_id.py           ← RequestIDMiddleware
├── models/                     ← pydantic-схемы запросов и ответов (HTTP-контракт)
│   ├── <aggregate>.py          ← <Aggregate>Response, <Aggregate>VersionResponse и т.п.
│   └── paginator_result.py     ← Generic LimitOffsetPaginatorResult и аналогичные обёртки
└── routers/                    ← APIRouter-ы, иерархия по слоям доступа
    ├── __init__.py             ← main_router = APIRouter(); include_router(...)
    ├── health.py               ← health_router
    └── public/                 ← публичные роуты с версионированием
        ├── __init__.py         ← public_router с prefix="/public"
        └── v1/
            ├── __init__.py     ← v1_router с prefix="/v1"; include роутеров агрегатов
            └── <aggregate>.py  ← <aggregate>_router с prefix="/<aggregates>"
```

## APIWorker

Класс-обёртка над `FastAPI`. Получает все настройки воркера, собирает приложение, экспортирует `run()`.

```python
class APIWorker:
    def __init__(self, settings: APIWorkerSettings) -> None:
        self.settings = settings
        lifespan = APILifespan(self.settings)
        self.app = FastAPI(lifespan=lifespan.lifespan)
        # ... middleware, error handlers, routers

    def run(self) -> None:
        config = uvicorn.Config(app=self.app, ..., access_log=False)
        uvicorn.Server(config).run()
```

Один FastAPI-app = один `APIWorker` = один YAML-конфиг (`APIWorkerSettings`). Не запускай несколько приложений в одном процессе. Полный код — [api_worker_template.md](references/api_worker_template.md).

## Middleware

Регистрируются через `app.add_middleware(...)`. **Starlette применяет middleware в обратном порядке регистрации:** последний `add_middleware` — самый внешний. Это влияет на то, кто видит что.

### Обязательный набор

| Слой (внутри → наружу) | Middleware | Назначение |
|---|---|---|
| 1 (innermost) | `LoggingMiddleware` | Структурный лог запроса+ответа, подбирает `error_context` от error handler |
| 2 | `PerformanceMiddleware` | Заголовки `x-process-time` / `x-process-time-ms` |
| 3 | `RequestIDMiddleware` | Сквозной `request_id` в `request.state` + заголовок ответа |
| 4 (outermost) | `CORSMiddleware` | CORS-политика (см. ниже) |

Порядок в коде:

```python
app.add_middleware(LoggingMiddleware)
app.add_middleware(PerformanceMiddleware, ...)
app.add_middleware(RequestIDMiddleware, ...)
app.add_middleware(CORSMiddleware, ...)   # последним → outermost
```

Реализации Logging/Performance/RequestID — в [middlewares_catalog.md](references/middlewares_catalog.md).

### CORS (обязательно)

`CORSMiddleware` обязателен в каждом API-воркере. Без него браузерные клиенты с другого origin режутся.

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=self.settings.fastapi.cors.allow_origins,
    allow_origin_regex=self.settings.fastapi.cors.allow_origin_regex,
    allow_credentials=self.settings.fastapi.cors.allow_credentials,
    allow_methods=self.settings.fastapi.cors.allow_methods,
    allow_headers=self.settings.fastapi.cors.allow_headers,
    expose_headers=self.settings.fastapi.cors.expose_headers,
    max_age=self.settings.fastapi.cors.max_age,
)
```

- Все параметры читаются из `settings.fastapi.cors.*`. **Хардкод в `server.py` запрещён** — иначе нельзя сузить политику в проде без релиза.
- Структура `CORSSettings` и валидаторы (запрет `credentials=True` с `*`, проверка regex) поставляются [python-pydantic-settings-config-writing] → секция «FastAPI / Uvicorn» в `references/tech_examples.md`. Если структуры в проекте ещё нет — заведи её, а не подставляй литералы.
- CORS — outermost: даже на 5xx из downstream-обработчиков ответ должен унести CORS-заголовки, иначе браузер не покажет тело ошибки клиенту.

## Error Handlers

Один файл `error_handler.py`, одна функция `setup_error_handler(app)`. Внутри регистрируются `@app.exception_handler(...)` для базовых иерархий исключений.

```python
def setup_error_handler(app: FastAPI) -> None:
    @app.exception_handler(AppError)
    async def handle_app_error(request: Request, exc: AppError) -> JSONResponse:
        error_context: dict[str, object] = {
            "detail": exc.msg, "action": exc.action, ...
        }
        if isinstance(exc, AppInternalError):
            status_code = status.HTTP_500_INTERNAL_SERVER_ERROR
            if exc.wrap_error is not None:
                error_context["wrap_error"] = str(exc.wrap_error)
        elif isinstance(exc, AppNotFoundError):
            status_code = status.HTTP_404_NOT_FOUND
        else:
            status_code = status.HTTP_400_BAD_REQUEST
        request.state.error_context = error_context
        return JSONResponse(status_code=status_code, content=jsonable_encoder({...}))

    @app.exception_handler(DomainError)
    async def handle_domain_error(request: Request, exc: DomainError) -> JSONResponse:
        ...
```

- Хэндлятся два корня иерархий: `AppError` (`application/errors`) и `DomainError` (`domain/errors`). Конкретные подтипы маппятся на HTTP-коды через `isinstance`.
- Контекст ошибки кладётся в `request.state.error_context` — `LoggingMiddleware` подбирает и пишет в структурный лог. Без этого ошибки уйдут в логи без бизнес-контекста.
- Для `AppInternalError` дополнительно кладётся `wrap_error=str(exc.wrap_error)` (если поле не `None`) — это исходное исключение из инфраструктуры. Без него 5xx-логи показывают только user-facing `detail`, и операторы вынуждены поднимать стэки, чтобы понять root cause. Для клиентских подтипов (`AppNotFoundError`, `AppUnauthorizedError`, `DomainError` и т.п.) `wrap_error` **не выставляется** — там нет инфраструктурного контекста, а если бы и был, не должен утечь в общий лог-контракт.
- Тело ответа: `{"detail": ..., "data": ...}`. `jsonable_encoder` — для безопасной сериализации pydantic-моделей внутри `data`. `wrap_error` уходит **только в лог**, не в тело ответа — иначе клиент увидит трассу инфраструктуры.
- В error handler **нет** бизнес-логики, обращений в БД, чтения внешних сервисов: только маппинг исключения в HTTP-ответ.

Полный код — [api_worker_template.md](references/api_worker_template.md).

## Lifespan

Shared-ресурсы (DB-пул, NATS-клиент и т.п.) живут на уровне процесса, не запроса. Инициализируются в lifespan и кладутся в `app.state.*`.

```python
class APILifespan:
    def __init__(self, settings: APIWorkerSettings) -> None:
        self._settings = settings

    @asynccontextmanager
    async def lifespan(self, app: FastAPI) -> AsyncGenerator:
        manager = PostgresConnectionManager(self._settings.db)
        await manager.init()
        app.state.fastapi_settings = self._settings.fastapi
        app.state.db_connection_manager = manager
        yield
        await manager.close()
```

- Класс-обёртка вокруг `@asynccontextmanager` нужен, чтобы передать `settings` через `__init__` (lifespan-функция от FastAPI получает только `FastAPI`-аргумент).
- Shared-ресурсы инициализируются на startup и кладутся в `app.state.<name>`; на shutdown закрываются в обратном порядке.
- В `app.state` лежат **shared-ресурсы и settings-блоки** (DB connection manager, NATS-клиент, `fastapi_settings` для извлечения имён заголовков). Per-request объекты (UnitOfWork, текущий пользователь) — через `Depends`, не через `app.state`.

## Routers

Один `main_router`, который `APIWorker` подключает целиком; внутри — иерархия подроутеров с префиксами по слоям доступа и версионированием.

```python
# presentation/api/routers/__init__.py
main_router = APIRouter()
main_router.include_router(health_router, prefix="/health")
main_router.include_router(public_router)   # внутри своя структура /public/v1/...
```

```python
# presentation/api/routers/public/__init__.py
public_router = APIRouter(prefix="/public", tags=["Public"])
public_router.include_router(v1_router)
```

```python
# presentation/api/routers/public/v1/__init__.py
v1_router = APIRouter(prefix="/v1", tags=["V1"])
v1_router.include_router(member_router)
v1_router.include_router(project_router)
# ...
```

```python
# presentation/api/routers/public/v1/<aggregate>.py
<aggregate>_router = APIRouter(prefix="/<aggregates>", tags=["<Aggregate>"])

@<aggregate>_router.get("")
async def get_<aggregates>(...): ...
```

- Префиксы фиксируются в подсборщиках рядом с ресурсами (`/public` → `public/__init__.py`, `/v1` → `v1/__init__.py`, `/members` → `member.py`), а не в `main_router`. Так версионирование и группировка живут локально и не требуют правок в `server.py`.
- `tags=[...]` навешивается на уровне `APIRouter(...)` — попадает в Swagger-секции автоматически.
- `APIWorker` ничего не знает про конкретные роуты — только про `main_router`. Добавление нового агрегата = `<aggregate>.py` + строка `include_router` в соответствующем `__init__.py`.

## Dependencies (Depends)

Per-request DI идёт через `fastapi.Depends`-функции из `presentation/api/dependencies/`. Они:

1. Читают `request.app.state.<resource>` для доступа к shared-ресурсам (без модульных глобалов).
2. Возвращают **уже готовый объект** для использования в эндпоинте (UnitOfWork, UUID юзера, и т.п.).
3. Никакой бизнес-логики — только извлечение/проверка из запроса.

### UnitOfWork-провайдер

```python
# presentation/api/dependencies/db.py
from fastapi import Request

from infrastructure.db.postgres import PostgresUnitOfWork


async def db_unit_of_work(request: Request) -> PostgresUnitOfWork:
    async with request.app.state.db_connection_manager.connection() as conn:
        return PostgresUnitOfWork(conn)
```

- `db_connection_manager` — shared-ресурс из lifespan. Берётся из `app.state`, не импортируется как модульный глобал.
- На каждый запрос — свежий `UnitOfWork` с собственным соединением; коммит/роллбэк управляются внутри use case.

### Извлечение пользователя

```python
# presentation/api/dependencies/user_id_extractor.py
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

- Имя заголовка читается из `FastAPISettings`, который положен в `app.state.fastapi_settings` на startup; не хардкодится.
- `HTTPException` — единственное место в `presentation/api/dependencies/`, где разрешено бросать HTTP-ошибки напрямую (auth-проверка — не доменная ошибка).

### Использование в эндпоинте

```python
@member_router.get("/{member_id}")
async def get_member(
    member_id: UUID,
    user_id: UUID = Depends(user_id_extractor),
    uow: PostgresUnitOfWork = Depends(db_unit_of_work),
) -> MemberSimpleResponse: ...
```

- Порядок: path-параметры → query-параметры → `Depends`-ы.
- В обработчик попадают только готовые объекты; чтения `request.headers`/`request.app.state` внутри обработчика быть не должно.

## Models

Pydantic-схемы запросов и ответов лежат в `presentation/api/models/`. Это **HTTP-контракт** — отдельный от `application/dto/`. DTO прикладного слоя нельзя возвращать напрямую: HTTP-схема должна меняться независимо от внутреннего DTO.

### Response-схема с конструктором из DTO

```python
# presentation/api/models/member.py
from typing import Self
from uuid import UUID

from pydantic import BaseModel

from application.dto.member import MemberSimpleDTO


class MemberSimpleResponse(BaseModel):
    member_id: UUID
    tenant_id: UUID
    status: str
    state: str
    version: int

    @classmethod
    def from_dto(cls, dto: MemberSimpleDTO) -> Self:
        return cls(
            member_id=dto.member_id,
            tenant_id=dto.tenant_id,
            status=dto.status,
            state=dto.state,
            version=dto.version,
        )
```

- Конструктор `from_dto` — единственный мост между application-DTO и HTTP-схемой. Эндпоинт зовёт `Response.from_dto(dto)`, не передаёт DTO в JSON-encoder напрямую.
- `Self` (3.11+) для типа возврата — наследники получают корректный тип.

### Generic-обёртки

Постраничные ответы — generic-обёртка, не отдельный класс на каждый ресурс.

```python
# presentation/api/models/paginator_result.py
from typing import Generic, TypeVar

T = TypeVar("T")


class LimitOffsetPaginatorResult(BaseModel, Generic[T]):
    count: int
    results: list[T]
    next: str | None = None
    previous: str | None = None

    @classmethod
    def create(cls, request, results, total_count, limit, offset) -> Self: ...
```

`LimitOffsetPaginatorResult[MemberSimpleResponse]` в сигнатуре эндпоинта → Swagger получит конкретный тип `results`.

## Endpoint pattern

Эндпоинт — тонкий клей: `Depends → Query/Command → UseCase(uow).execute(...) → Response.from_dto(dto)`. Никакой бизнес-логики.

```python
@member_router.get("/{member_id}")
async def get_member(
    member_id: UUID,
    user_id: UUID = Depends(user_id_extractor),
    uow: PostgresUnitOfWork = Depends(db_unit_of_work),
) -> MemberSimpleResponse:
    query = MemberLastVersionQuery(initiator_id=user_id, member_id=member_id)
    use_case = MemberLastVersionUseCase(uow)
    dto = await use_case.execute(query)
    return MemberSimpleResponse.from_dto(dto)
```

Шаги обработчика, всегда в одном порядке:

1. Собрать `Query`/`Command` из path/query/body + `user_id` из `Depends`.
2. Инстанцировать `UseCase`, передав `uow` из `Depends`.
3. `await use_case.execute(...)`.
4. Конвертировать результат в Response-схему через `from_dto`.

Чего в эндпоинте быть **не должно**:

- Прямых обращений в `infrastructure/` (репозитории, БД-сессии) — только через use case и его UnitOfWork.
- Логики маппинга DTO → Response помимо вызова `from_dto`.
- Обработки `AppError`/`DomainError` через `try/except` — это делает `setup_error_handler`.

## Uvicorn

Запуск через `uvicorn.Server(config)` в `APIWorker.run()`:

```python
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
    uvicorn.Server(config).run()
```

- `loop=self.settings.uvicorn.loop` — обычно `"uvloop"`; Uvicorn сам поднимает uvloop policy при старте. Если платформа `uvloop` не поддерживает — явный fallback на `"asyncio"`.
- `access_log=False` — отключаем стандартный access log Uvicorn-а; вся обсервабельность HTTP идёт через `LoggingMiddleware`. Иначе будет дублирование.
- `reload=True` только локально, в проде — всегда `False`.
- `workers` обычно > 1 в проде; lifespan и shared-ресурсы внутри каждого воркера — независимые.

## Entrypoint

API-воркер — один из режимов в `src/main.py`:

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

- `APIWorkerSettings()` сам читает YAML по `CONFIG_FILE` (см. [python-pydantic-settings-config-writing]).
- Миграции применяются **до** старта app, а не в lifespan: иначе несколько воркеров будут гоняться за блокировкой.
- В `main.py` не должно быть бизнес-логики и обращений в `presentation/api/...` помимо `APIWorker`.

## Anti-patterns

- **CORS не последний** в цепочке `add_middleware` → не накладывает заголовки на ответы из downstream-обработчиков ошибок; браузер не покажет тело 5xx.
- **Хардкод CORS-параметров** в `server.py` (например `allow_origins=["*"]` прямо в коде) — нельзя сузить политику без релиза.
- **Отсутствие `CORSMiddleware`** вообще — браузерные клиенты с другого origin молча блокируются.
- **Бизнес-логика в `error_handler.py`** (обращения в БД, мутации) — error handler только маппит исключение в HTTP-ответ.
- **Глобальные shared-ресурсы** мимо `app.state` (модульные глобалы DB-пула, NATS-клиента) — теряем graceful shutdown, тесты не могут подменить.
- **Per-request объекты (UoW, текущий пользователь) в `app.state`** — `app.state` для процесса, не для запроса. Per-request → `Depends`.
- **Импорт `infrastructure/` напрямую в эндпоинтах** (репозитории, sessions, клиенты) — только через use case и его `UnitOfWork`.
- **Возврат application-DTO напрямую** из эндпоинта без обёртки в `<Aggregate>Response` — HTTP-контракт начинает зависеть от структуры DTO.
- **`try/except AppError` / `try/except DomainError` в эндпоинте** — обработка ошибок централизована в `setup_error_handler`.
- **Хардкод имён заголовков** (`user-id`, `x-request-id`) в эндпоинтах/middleware — читать из `FastAPISettings` (через `app.state` или `__init__`-аргументы middleware).
- **Несколько `FastAPI`-app в одном процессе** — один воркер = одно приложение.
- **Свой access-log** в Uvicorn в дополнение к `LoggingMiddleware` — дублирование, разные форматы.
- **Применение миграций в lifespan** — гонка между воркерами/инстансами; применяй в entrypoint до старта app.
- **`reload=True` в проде** — пересоздаёт процесс при изменении файлов; в проде это не нужно и опасно.

## Definition of Done

- `APIWorker` в `src/presentation/api/server.py` собирает `FastAPI(lifespan=...)`, включает middleware/error-handlers/routers и предоставляет `run()`.
- Зарегистрированы: `LoggingMiddleware` → `PerformanceMiddleware` → `RequestIDMiddleware` → `CORSMiddleware` (последний — outermost).
- CORS-параметры читаются из `settings.fastapi.cors.*`, не хардкодятся; `CORSSettings` заведён через [python-pydantic-settings-config-writing] (FastAPI / Uvicorn).
- `setup_error_handler(app)` маппит подтипы `AppError`/`DomainError` на HTTP-коды и кладёт `request.state.error_context`; для `AppInternalError` контекст дополнен полем `wrap_error` (значение из `exc.wrap_error`), которое в тело ответа не попадает.
- `APILifespan` инициализирует shared-ресурсы на startup и закрывает на shutdown в обратном порядке; всё попадает в `app.state.*`.
- `main_router` — единственная точка подключения роутов в `APIWorker`; иерархия подроутеров фиксирует префиксы локально (`/public` → `public/__init__.py`, `/v1` → `v1/__init__.py`, `/<aggregates>` → `<aggregate>.py`).
- Depends-провайдеры в `dependencies/` читают `request.app.state` для shared-ресурсов и не содержат бизнес-логики; имена заголовков берутся из `FastAPISettings`.
- HTTP-схемы лежат в `models/`, конструируются из application-DTO через `from_dto`; application-DTO наружу не утекают.
- Эндпоинт — тонкий клей: `Depends → Query/Command → UseCase.execute(...) → Response.from_dto(...)`, без обращений в `infrastructure/` и без `try/except` доменных/прикладных ошибок.
- `uvicorn.Config` использует `loop="uvloop"` (или явный fallback), `access_log=False`.
- `main.py` под `MODE=api` соблюдает порядок: settings → миграции → `APIWorker(...).run()`.

## References

- **[api_worker_template.md](references/api_worker_template.md)** — полный эталон `APIWorker`, `APILifespan`, `setup_error_handler`, `routers/__init__.py` и фрагмент `main.py`. Стартовая точка для нового сервиса.
- **[middlewares_catalog.md](references/middlewares_catalog.md)** — реализации `LoggingMiddleware`, `PerformanceMiddleware`, `RequestIDMiddleware`; таблица «что кладёт в `request.state`»; как добавить новую middleware и куда её ставить в цепочке.
- Структура CORS-настроек и валидаторы — [python-pydantic-settings-config-writing] → `references/tech_examples.md` (FastAPI / Uvicorn).
