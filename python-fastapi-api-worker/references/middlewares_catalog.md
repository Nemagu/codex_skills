# Middlewares Catalog

Стандартный набор middleware HTTP API-воркера. Все наследуются от `starlette.middleware.base.BaseHTTPMiddleware`. Файлы — в `src/presentation/api/middlewares/<name>.py`, экспорт — через `middlewares/__init__.py`.

## Порядок регистрации

```python
# в APIWorker.__init__
app.add_middleware(LoggingMiddleware)
app.add_middleware(PerformanceMiddleware, header_name=..., header_name_ms=...)
app.add_middleware(RequestIDMiddleware, header_name=...)
app.add_middleware(CORSMiddleware, ...)
```

Starlette применяет middleware **в обратном порядке регистрации**: последний `add_middleware` оказывается самым внешним. Это критично — кто стоит снаружи, тот видит ответы downstream (включая ответы error handler-а) и может навесить заголовки.

| Слой | Middleware | Видит `request.state` от | Пишет в `request.state` | Заголовки ответа |
|---|---|---|---|---|
| 1 (innermost) | `LoggingMiddleware` | `error_context` (включая `wrap_error` для `AppInternalError`), `request_id` | — | — |
| 2 | `PerformanceMiddleware` | `request_id` | — | `x-process-time`, `x-process-time-ms` |
| 3 | `RequestIDMiddleware` | — | `request_id` | `x-request-id` |
| 4 (outermost) | `CORSMiddleware` | — | — | `access-control-*` |

Источник истины для имён заголовков — `FastAPISettings` ([python-pydantic-settings-config-writing]); в middleware-классы передаются явно через `__init__`-аргументы, а не читаются из `settings` внутри `dispatch`.

## LoggingMiddleware

Структурный лог каждого запроса после прогона downstream. Подбирает `error_context`, который выставил error handler.

```python
"""Модуль `presentation/api/middlewares/logging.py` слоя представления."""

from logging import getLogger
from time import perf_counter
from typing import Any, Callable

from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.types import ASGIApp

logger = getLogger(__name__)


class LoggingMiddleware(BaseHTTPMiddleware):
    def __init__(self, app: ASGIApp) -> None:
        super().__init__(app)

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        start_time = perf_counter()
        try:
            response = await call_next(request)
        except BaseException as err:
            duration_ms = round((perf_counter() - start_time) * 1000, 2)
            context = self._build_context(request, None, duration_ms)
            context["unhandled_error"] = str(err)
            logger.exception("http_request %s", context)
            raise

        duration_ms = round((perf_counter() - start_time) * 1000, 2)
        context = self._build_context(request, response.status_code, duration_ms)
        if response.status_code >= 500:
            logger.error("http_request %s", context)
        elif response.status_code >= 400:
            logger.warning("http_request %s", context)
        else:
            logger.info("http_request %s", context)
        return response

    @staticmethod
    def _build_context(
        request: Request, status_code: int | None, duration_ms: float
    ) -> dict[str, Any]:
        error_context = getattr(request.state, "error_context", None)
        if not isinstance(error_context, dict):
            error_context = {}
        return {
            "request_id": getattr(request.state, "request_id", None),
            "method": request.method,
            "path": request.url.path,
            "query": request.url.query,
            "status_code": status_code,
            "duration_ms": duration_ms,
            "action": error_context.get("action"),
            "struct_name": error_context.get("struct_name"),
            "detail": error_context.get("detail"),
            "data": error_context.get("data"),
            "wrap_error": error_context.get("wrap_error"),
        }
```

Замечания:
- Уровень лога — от status code: ≥500 → `error`, ≥400 → `warning`, остальное → `info`.
- `BaseException`-ветка нужна для unhandled-крашей до того, как Starlette превратит исключение в 500.
- `error_context` ставит error handler (`setup_error_handler`); middleware его только читает. Без error handler-а middleware всё равно работает — `error_context` будет пустым словарём.
- `LoggingMiddleware` — innermost: видит status code ответа уже после error handler-а и подбирает выставленный им контекст.
- Поле `wrap_error` имеет смысл только для `AppInternalError` (5xx) — там лежит `str(exc.wrap_error)` с исходным инфраструктурным исключением. Для остальных кейсов handler его не выставляет и в логе оно остаётся `None`. Тело ответа `wrap_error` **не содержит** — это поле строго для лога, иначе клиент увидит трассу инфраструктуры.

## PerformanceMiddleware

Замеряет общее время downstream-обработки, навешивает два заголовка ответа.

```python
"""Модуль `presentation/api/middlewares/performance.py` слоя представления."""

import time
from typing import Callable

from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.types import ASGIApp


class PerformanceMiddleware(BaseHTTPMiddleware):
    def __init__(
        self,
        app: ASGIApp,
        header_name: str = "x-process-time",
        header_name_ms: str = "x-process-time-ms",
    ) -> None:
        super().__init__(app)
        self.header_name = header_name
        self.header_name_ms = header_name_ms

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        start = time.perf_counter()
        response = await call_next(request)
        duration = time.perf_counter() - start
        response.headers[self.header_name] = str(duration)
        response.headers[self.header_name_ms] = str(round(duration * 1000, 2))
        return response
```

Имена заголовков — параметры конструктора; идут из `FastAPISettings.process_time_header_name` / `process_time_ms_header_name`. Клиент знает имена из API-спеки, не из middleware-кода.

## RequestIDMiddleware

Генерирует `request_id` (uuid7 — лексикографически сортируется по времени), кладёт в `request.state` для downstream и в ответный заголовок для клиента.

```python
"""Модуль `presentation/api/middlewares/request_id.py` слоя представления."""

from typing import Callable
from uuid import uuid7

from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.types import ASGIApp


class RequestIDMiddleware(BaseHTTPMiddleware):
    def __init__(self, app: ASGIApp, header_name: str = "x-request-id") -> None:
        super().__init__(app)
        self.header_name = header_name

    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        request_id = str(uuid7())
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers[self.header_name] = request_id
        return response
```

`uuid7` — выбор по политике проекта; `uuid4` тоже допустим (теряем лексикографическую сортируемость в логах/индексах).

## CORSMiddleware

Поставляется FastAPI/Starlette (`fastapi.middleware.cors.CORSMiddleware`) — свой код не пишем. Параметры — `settings.fastapi.cors.*`, структура `CORSSettings` — в [python-pydantic-settings-config-writing] (`references/tech_examples.md` → FastAPI / Uvicorn).

Регистрируется **последним** в цепочке `add_middleware` (outermost). Иначе ответы из downstream-обработчиков ошибок (5xx, 4xx из `setup_error_handler`) уйдут без CORS-заголовков, и браузер не покажет клиенту тело ошибки.

## Добавление новой middleware

1. Создай `src/presentation/api/middlewares/<name>.py` с классом-наследником `BaseHTTPMiddleware`.
2. В `__init__` принимай только нужные параметры; не лезь в `settings`, не делай side-effects при импорте.
3. В `dispatch` оборачивай `await call_next(request)` минимальным кодом; тяжёлая логика — в эндпоинтах, не в middleware.
4. Экспортируй из `middlewares/__init__.py`.
5. Зарегистрируй в `APIWorker.__init__` в правильном месте цепочки:
   - **Кладёт в `request.state` для downstream** → ставь раньше (`add_middleware` ближе к началу — становится innermost).
   - **Читает результат downstream / навешивает заголовок ответа** → ставь позже (становится более внешним).
   - **CORS остаётся последним** независимо от того, что добавляешь.
6. Если middleware кладёт что-то в `request.state` — задокументируй в таблице в начале этого файла.

## Anti-patterns

- Чтение `settings` внутри `dispatch` (вместо передачи через `__init__`) — middleware становится зависим от глобального state и плохо тестируется.
- Тяжёлая логика (БД-запросы, внешние HTTP-вызовы) в middleware — для этого есть эндпоинты с `Depends`.
- Своя замена `LoggingMiddleware` + включённый Uvicorn access log — дублирование форматов и шумные логи.
- Перехват исключений в middleware **с подавлением** (без `raise`) — error handler уже не получит управления, status code станет произвольным.
- Регистрация CORS не последней.
