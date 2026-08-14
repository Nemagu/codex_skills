# Промежуточное ПО

## Содержание

- [Проектирование цепочки](#проектирование-цепочки)
- [Чистое ASGI](#чистое-asgi)
- [Журнал доступа](#журнал-доступа)
- [CORS](#cors)
- [Ограничения](#ограничения)

## Проектирование цепочки

Для каждой middleware составить краткую таблицу:

| Middleware | Входные данные | Создаёт | Потребители | Требуемое положение |
|---|---|---|---|---|
| Request context | headers | request ID/context | logs, handlers | охватывает потребителей |
| Access log | scope, status, duration | log event | observability | видит итоговый ответ |
| Error boundary | exception | safe 500 | client, error log | охватывает приложение |
| CORS | origin, response | CORS headers | browser | глобально, если требуется |

Не копировать эту таблицу как обязательный набор. Добавлять только компоненты из
требований.

## Чистое ASGI

Для сквозного request context использовать stateless ASGI middleware:

```python
from collections.abc import Callable
from contextvars import ContextVar

from starlette.datastructures import MutableHeaders
from starlette.types import ASGIApp, Message, Receive, Scope, Send


request_id_context: ContextVar[str | None] = ContextVar(
    "request_id",
    default=None,
)


class RequestIDMiddleware:
    def __init__(
        self,
        app: ASGIApp,
        *,
        header_name: str,
        id_factory: Callable[[], str],
    ) -> None:
        self._app = app
        self._header_name = header_name
        self._id_factory = id_factory

    async def __call__(
        self,
        scope: Scope,
        receive: Receive,
        send: Send,
    ) -> None:
        if scope["type"] != "http":
            await self._app(scope, receive, send)
            return

        request_id = self._id_factory()
        token = request_id_context.set(request_id)

        async def send_with_request_id(message: Message) -> None:
            if message["type"] == "http.response.start":
                headers = MutableHeaders(scope=message)
                headers[self._header_name] = request_id
            await send(message)

        try:
            await self._app(scope, receive, send_with_request_id)
        finally:
            request_id_context.reset(token)
```

Политику входного request ID задавать отдельно:

- принимать только корректное значение согласованного формата;
- определять, доверен ли внешний источник;
- генерировать новое значение при отсутствии или недоверенном источнике;
- при необходимости хранить внешний correlation ID отдельно.

## Журнал доступа

Логировать после получения `http.response.start` или завершения приложения:

- method и нормализованный route template, если он доступен;
- status code и duration;
- request/correlation ID;
- разрешённые идентификаторы вызывающей стороны;
- размер запроса/ответа только при необходимости.

Не включать query string целиком по умолчанию: она может содержать токены и
персональные данные. Не читать body ради логирования.

Unhandled exception логирует error boundary с `exc_info`; access middleware
фиксирует итог запроса без второго stack trace.

## CORS

CORS нужен только браузерному cross-origin взаимодействию, заданному
требованиями.

- Передавать origins, methods, headers, credentials и max age явно.
- При credentials не использовать wildcard там, где он запрещён контрактом.
- Для CORS-заголовков на неожиданных `500` оборачивать всё ASGI-приложение.
- Не включать CORS как способ исправить серверную авторизацию.

## Ограничения

- Middleware instance не хранит per-request mutable state.
- Не обращаться к БД и внешним API.
- Не подавлять исключения, если middleware не является согласованной error
  boundary.
- Не изменять request/response body без явного требования и поддержки streaming.
- Проверять HTTP, WebSocket и lifespan scopes либо явно пропускать неподдерживаемые.
