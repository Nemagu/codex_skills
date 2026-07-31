# Настройки PostgreSQL, пула psycopg и yoyo

## Модели

Поля являются примером для согласования:

```python
from pathlib import Path
from typing import Annotated, Self

from pydantic import Field, model_validator


Port = Annotated[int, Field(ge=1, le=65535)]
PositiveSeconds = Annotated[float, Field(gt=0)]
ExternalPath = Annotated[Path, Field(strict=False)]


class PostgresPoolSettings(ConfigModel):
    min_size: Annotated[int, Field(ge=0)]
    max_size: Annotated[int, Field(gt=0)]
    timeout_seconds: PositiveSeconds
    max_lifetime_seconds: PositiveSeconds
    max_idle_seconds: PositiveSeconds
    reconnect_timeout_seconds: PositiveSeconds
    max_waiting: Annotated[int, Field(ge=0)]
    num_workers: Annotated[int, Field(gt=0)]

    @model_validator(mode="after")
    def validate_size(self) -> Self:
        if self.min_size > self.max_size:
            raise ValueError("min_size не может превышать max_size")
        return self


class PostgresSettings(ConfigModel):
    host: str
    port: Port
    user: str
    database: str
    password_file: ExternalPath
    connect_timeout_seconds: PositiveSeconds
    pool: PostgresPoolSettings


class YoyoSettings(ConfigModel):
    migrations_directory: ExternalPath
```

Не добавлять все параметры psycopg pool автоматически. Выбирать только
необходимые установленной версии.

## Явное преобразование в клиент

Settings не импортирует psycopg. Adapter factory строит connection и pool kwargs
явно:

```python
from typing import TypedDict


class PoolKwargs(TypedDict):
    min_size: int
    max_size: int
    timeout: float
    max_lifetime: float
    max_idle: float
    reconnect_timeout: float
    max_waiting: int
    num_workers: int


def pool_kwargs(settings: PostgresPoolSettings) -> PoolKwargs:
    return {
        "min_size": settings.min_size,
        "max_size": settings.max_size,
        "timeout": settings.timeout_seconds,
        "max_lifetime": settings.max_lifetime_seconds,
        "max_idle": settings.max_idle_seconds,
        "reconnect_timeout": settings.reconnect_timeout_seconds,
        "max_waiting": settings.max_waiting,
        "num_workers": settings.num_workers,
    }
```

Создавать `AsyncConnectionPool(..., open=False)` и открывать его в lifecycle
worker-а. Пароль передавать через connection kwargs после чтения provider-ом, а
не встраивать в логируемую строку.

Для yoyo использовать backend `postgresql+psycopg`. DSN/kwargs формирует
migration adapter из тех же connection settings и секрета, но не хранит
plaintext в settings.

## YAML

```yaml
postgres:
  host: postgres.internal
  port: 5432
  user: service_user
  database: service_database
  password_file: /run/secrets/postgres_password
  connect_timeout_seconds: 5.0
  pool:
    min_size: 2
    max_size: 10
    timeout_seconds: 10.0
    max_lifetime_seconds: 3600.0
    max_idle_seconds: 600.0
    reconnect_timeout_seconds: 300.0
    max_waiting: 20
    num_workers: 3

yoyo:
  migrations_directory: /app/migrations
```

Значения pool являются примером для обсуждения, а не универсальными defaults.

## Проверки

- Обязательность всех connection-полей и pool-блока.
- `min_size <= max_size`.
- Положительные durations и допустимый port.
- Preflight password file и migrations directory.
- Точное сопоставление каждого согласованного поля с kwargs установленной версии.
- Отсутствие password/DSN в dump и логах.
