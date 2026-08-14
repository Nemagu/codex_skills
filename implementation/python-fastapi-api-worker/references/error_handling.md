# Обработка ошибок

## Содержание

- [Граница слоёв](#граница-слоёв)
- [Регистрация](#регистрация)
- [Непредвиденные ошибки](#непредвиденные-ошибки)
- [Логирование](#логирование)
- [Проверки](#проверки)

## Граница слоёв

HTTP-слой знает только публичные application-ошибки и транспортные ошибки. Если
из application выходит domain-ошибка, исправить преобразование на границе
application, а не добавлять domain handler в FastAPI.

Для каждого внешнего исхода требования должны определить:

- HTTP status;
- стабильный внешний code, если он есть;
- безопасные detail и data;
- обязательные response headers;
- уровень и состав логирования.

Не выводить внешний формат из имени Python-класса автоматически.

## Регистрация

Сохранять handlers небольшими и передавать регистрацию в фабрику:

```python
from collections.abc import Callable
from dataclasses import dataclass

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse


@dataclass(frozen=True, slots=True)
class ErrorResponse:
    status_code: int
    content: dict[str, object]
    headers: dict[str, str] | None = None


ApplicationErrorMapper = Callable[[Exception], ErrorResponse]


def install_error_handlers(
    app: FastAPI,
    map_application_error: ApplicationErrorMapper,
) -> None:
    @app.exception_handler(PublicApplicationError)
    async def handle_application_error(
        request: Request,
        error: PublicApplicationError,
    ) -> JSONResponse:
        response = map_application_error(error)
        return JSONResponse(
            status_code=response.status_code,
            content=response.content,
            headers=response.headers,
        )
```

`PublicApplicationError` здесь обозначает согласованный корень публичных ошибок.
Не вводить его в проект только ради совпадения с примером.

Transport validation errors обрабатывать отдельно, если стандартный FastAPI
ответ не соответствует внешнему контракту.

## Непредвиденные ошибки

Внешняя error boundary:

1. формирует или извлекает request ID;
2. логирует событие один раз через `logger.exception(...)` либо
   `logger.error(..., exc_info=error)`;
3. возвращает согласованный безопасный `500`;
4. не включает exception text и stack trace в body;
5. не пытается продолжить response, если его отправка уже началась.

Учитывать встроенные `ServerErrorMiddleware` и `ExceptionMiddleware`: не
создавать вторую конкурирующую границу поверх них без необходимости.

## Логирование

Разделять события:

- `http_access` — итог запроса без stack trace;
- `http_expected_error` — ожидаемый внешний отказ, только если нужен;
- `http_unhandled_error` — непредвиденная ошибка со stack trace;
- `api_startup_failed` и `api_shutdown_failed` — lifecycle.

Не преобразовывать исходное исключение только в `str(error)`: для внутренней
диагностики сохранять traceback через `exc_info`.

Не логировать:

- Authorization, Cookie и Set-Cookie;
- plaintext secrets;
- request/response body по умолчанию;
- полные DTO или settings;
- внутренние данные в клиентском error response.

## Проверки

- Каждый application outcome получает ожидаемый status и body.
- Неизвестная ошибка даёт безопасный `500`.
- Stack trace присутствует только в серверном логе.
- Одна ошибка не создаёт два error-события.
- Request ID совпадает в ответе и лог-контексте.
- Validation error соответствует внешнему контракту.
