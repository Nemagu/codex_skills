# Запуск Uvicorn

## Содержание

- [Выбор режима](#выбор-режима)
- [Параметры](#параметры)
- [Доверие прокси](#доверие-прокси)
- [OpenAPI и документация](#openapi-и-документация)
- [Плавное завершение](#плавное-завершение)
- [Проверки](#проверки)

## Выбор режима

| Требование | Представление приложения | Запуск |
|---|---|---|
| Один процесс, без reload | готовый ASGI object | программный `Server` |
| Reload | import string/factory | CLI или `uvicorn.run` |
| Несколько workers | import string/factory | Uvicorn process manager |
| Внешний supervisor | import string/factory | параметры задаёт supervisor |

Не передавать готовый объект приложения в multi-worker/reload режим: дочерний
процесс должен импортировать или создать приложение самостоятельно.

## Параметры

Описать отдельный неизменяемый контракт runner-а и сопоставить его с Uvicorn
явно:

```python
from dataclasses import dataclass

import uvicorn
from starlette.types import ASGIApp


@dataclass(frozen=True, slots=True)
class UvicornParameters:
    host: str
    port: int
    loop: str
    proxy_headers: bool
    forwarded_allow_ips: tuple[str, ...]
    timeout_graceful_shutdown_seconds: int


def run_single_process(
    app: ASGIApp,
    parameters: UvicornParameters,
) -> None:
    config = uvicorn.Config(
        app=app,
        host=parameters.host,
        port=parameters.port,
        loop=parameters.loop,
        proxy_headers=parameters.proxy_headers,
        forwarded_allow_ips=list(parameters.forwarded_allow_ips),
        timeout_graceful_shutdown=(
            parameters.timeout_graceful_shutdown_seconds
        ),
        workers=1,
        reload=False,
    )
    uvicorn.Server(config).run()
```

Перед реализацией сверять имена и типы параметров с установленной версией
Uvicorn. Не передавать settings через `model_dump()`.

Для factory mode использовать import string вида `package.bootstrap:create_app`
и включать factory-флаг способом, поддержанным выбранным запуском. Factory
должна быть import-safe: не открывать ресурсы до lifespan.

## Доверие прокси

- `proxy_headers=True` не означает доверие любому источнику.
- `forwarded_allow_ips` задаёт согласованный список proxy IPs/networks.
- Не принимать `X-Forwarded-For`, `X-Forwarded-Proto` и host от недоверенного
  клиента.
- Согласовать поведение Unix socket и локального proxy отдельно.
- `root_path` FastAPI задавать согласно path prefix reverse proxy.

## OpenAPI и документация

Конфигурировать независимо:

- `openapi_url`;
- `docs_url`;
- `redoc_url`;
- `root_path`;
- servers в OpenAPI, если они нужны внешнему контракту.

`None` означает отключение соответствующего endpoint. Не включать production
docs автоматически.

## Плавное завершение

Согласовать:

- таймаут завершения активных запросов;
- keep-alive timeout;
- лимит конкурентности, если нужен;
- внешний termination grace period;
- время cleanup ресурсов.

Внешний grace period должен покрывать drain и cleanup. Не добавлять собственные
signal handlers поверх Uvicorn без требования.

## Проверки

- Single-process runner получает готовый app и `workers=1`, `reload=False`.
- Multi-worker/reload использует import string или factory.
- Несовместимая комбинация отклоняется до запуска.
- Factory не создаёт process-scoped ресурсы при импорте.
- Proxy headers не доверяются произвольным адресам.
- Все параметры соответствуют установленному Uvicorn.
