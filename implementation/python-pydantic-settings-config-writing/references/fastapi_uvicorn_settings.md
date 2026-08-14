# Настройки FastAPI, Uvicorn и CORS

## Модели

```python
import re
from typing import Annotated, Literal, Self

from pydantic import Field, field_validator, model_validator


Port = Annotated[int, Field(ge=1, le=65535)]


class CorsSettings(ConfigModel):
    allow_origins: Annotated[tuple[str, ...], Field(strict=False)]
    allow_origin_regex: str | None = None
    allow_credentials: bool = False
    allow_methods: Annotated[tuple[str, ...], Field(strict=False)]
    allow_headers: Annotated[tuple[str, ...], Field(strict=False)]
    expose_headers: Annotated[
        tuple[str, ...],
        Field(strict=False),
    ] = ()
    max_age_seconds: Annotated[int, Field(ge=0)] = 600

    @field_validator("allow_origin_regex")
    @classmethod
    def validate_origin_regex(cls, value: str | None) -> str | None:
        if value is None:
            return value
        try:
            re.compile(value)
        except re.error as error:
            raise ValueError("некорректное регулярное выражение CORS") from error
        return value

    @model_validator(mode="after")
    def validate_credentials(self) -> Self:
        if self.allow_credentials and any(
            "*" in values
            for values in (
                self.allow_origins,
                self.allow_methods,
                self.allow_headers,
            )
        ):
            raise ValueError(
                "credentials несовместимы с wildcard CORS-параметрами"
            )
        return self


class FastApiSettings(ConfigModel):
    user_id_header_name: str
    request_id_header_name: str
    cors: CorsSettings


class UvicornSettings(ConfigModel):
    host: str
    port: Port
    workers: Annotated[int, Field(gt=0)]
    reload: bool = False
    loop: Literal["asyncio", "uvloop"] = "uvloop"
```

Имена headers и CORS являются внешним контрактом; брать их из требований.
Settings не создаёт `FastAPI`/`CORSMiddleware`/uvicorn config.

## YAML

```yaml
fastapi:
  user_id_header_name: x-user-id
  request_id_header_name: x-request-id
  cors:
    allow_origins:
      - https://frontend.example
    allow_origin_regex: null
    allow_credentials: true
    allow_methods:
      - GET
      - POST
    allow_headers:
      - authorization
      - content-type
    expose_headers: []
    max_age_seconds: 600

uvicorn:
  host: 0.0.0.0
  port: 8000
  workers: 1
  reload: false
  loop: uvloop
```

Production-модель не должна молча разрешать `"*"` через default. Конкретное
окружение задаёт CORS явно.

## Проверки

- Компилируемый regex.
- Согласованность credentials и wildcard.
- Допустимые port/workers/loop.
- Преобразование tuple в типы, ожидаемые Starlette/Uvicorn, выполняет adapter.
- Reload/workers compatibility проверять по актуальному способу запуска.
