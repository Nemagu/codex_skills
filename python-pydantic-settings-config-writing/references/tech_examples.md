# Technology Examples

Опорные технологические примеры. **Не предписание** — используются агентом как источник подсказок при добавлении нового concern-блока: типичный набор полей, дефолты, computed properties, секреты.

При добавлении нового config-блока:
1. Если технология присутствует ниже — предложи пользователю стандартный набор полей из соответствующей секции.
2. Если технологии нет — после согласования полей с пользователем добавь её сюда (см. секцию **Шаблон для новой технологии** в конце).

Файл пополняется по мере появления новых технологий в проектах.

## Logging

```python
# infrastructure/config/logging.py
from enum import StrEnum

from pydantic import BaseModel


class LoggingLevel(StrEnum):
    DEBUG = "debug"
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"
    CRITICAL = "critical"


class LoggingSettings(BaseModel):
    level: LoggingLevel = LoggingLevel.INFO
```

**Типичные поля:** `level` (через `LoggingLevel` StrEnum, значения в нижнем регистре).

**Требование:** уровень логирования всегда задаётся через `StrEnum`, не через `str`. Значения enum — строго нижний регистр.

## FastAPI / Uvicorn

```python
# infrastructure/config/fastapi.py
import re
from typing import Self

from pydantic import BaseModel, Field, field_validator, model_validator


class CORSSettings(BaseModel):
    allow_origins: list[str] = ["*"]
    allow_origin_regex: str | None = None
    allow_credentials: bool = False
    allow_methods: list[str] = ["*"]
    allow_headers: list[str] = ["*"]
    expose_headers: list[str] = []
    max_age: int = Field(default=600, ge=0)

    @field_validator("allow_origin_regex")
    @classmethod
    def _validate_origin_regex(cls, value: str | None) -> str | None:
        if value is None:
            return value
        try:
            re.compile(value)
        except re.error as exc:
            raise ValueError(f"allow_origin_regex is not a valid regex: {exc}") from exc
        return value

    @model_validator(mode="after")
    def _forbid_credentials_with_wildcards(self) -> Self:
        if not self.allow_credentials:
            return self
        offenders = [
            name
            for name, values in (
                ("allow_origins", self.allow_origins),
                ("allow_methods", self.allow_methods),
                ("allow_headers", self.allow_headers),
            )
            if "*" in values
        ]
        if offenders:
            raise ValueError(
                "allow_credentials=True is incompatible with '*' in: "
                + ", ".join(offenders)
                + ". Replace wildcards with explicit values."
            )
        return self


class FastAPISettings(BaseModel):
    user_id_header_name: str = "x-user-id"
    request_id_header_name: str = "x-request-id"
    process_time_header_name: str = "x-process-time"
    process_time_ms_header_name: str = "x-process-time-ms"
    cors: CORSSettings = Field(default_factory=CORSSettings)


class UvicornSettings(BaseModel):
    host: str = "localhost"
    port: int = 8000
    workers: int = 1
    reload: bool = False
    loop: str = "uvloop"
```

**Типичные поля FastAPI:** имена заголовков для middleware (user-id, request-id, process-time) и вложенный блок `cors: CORSSettings`.

**Типичные поля CORS:** `allow_origins`, `allow_origin_regex`, `allow_credentials`, `allow_methods`, `allow_headers`, `expose_headers`, `max_age` — один-в-один параметры Starlette `CORSMiddleware`.

**Defaults у CORS — позволительные** (`origins/methods/headers=["*"]`, `credentials=False`): подходят для локальной разработки, не требуют объяснения с фронтом «почему запросы режутся». В проде сужают `allow_origins` до конкретного списка. Если фронту нужны cookies или `Authorization` — `allow_credentials: true` плюс **явные** origins/methods/headers (см. валидатор ниже).

**Запрещённые сочетания (валидируются на загрузке конфига):**

- `allow_credentials=True` + `"*"` в `allow_origins`/`allow_methods`/`allow_headers` — по спецификации браузер не примет ответ с `Access-Control-Allow-Origin: *` и `Access-Control-Allow-Credentials: true` одновременно; падаем на старте процесса, а не получаем тихие отказы в рантайме.
- `allow_origin_regex` должен компилироваться как Python-regex.

**Регистрация мидлвары (presentation, не config):** `CORSMiddleware` подключают через `app.add_middleware(CORSMiddleware, ...)` **последним** — Starlette применяет мидлвары в обратном порядке регистрации, и таким образом CORS оказывается самым внешним и навешивает заголовки даже на 5xx из downstream-обработчиков.

**Типичные поля Uvicorn:** `host`, `port`, `workers`, `reload`, `loop`.

## NATS / JetStream

### Базовая конфигурация подключения

```python
# infrastructure/config/nats.py (фрагмент)
from pydantic import BaseModel


class NatsSettings(BaseModel):
    host: str = "localhost"
    port: int = 4222
    healthcheck_file: str = "/app/run/nats_worker_healthbeat"

    loop_sleep_duration: int = 2

    connect_name: str = "app-worker"
    reconnect_time_wait: int = 5
    connect_timeout: int = 5
    ping_interval: int = 120
    max_outstanding_pings: int = 3

    @property
    def url(self) -> str:
        return f"nats://{self.host}:{self.port}"
```

**Типичные поля:** `host`, `port`, `healthcheck_file`, `loop_sleep_duration`, `connect_name`, `reconnect_time_wait`, `connect_timeout`, `ping_interval`, `max_outstanding_pings`.

> `healthcheck_file` — путь, **куда воркер пишет** liveness-маячок. Дефолт не указывать в `/tmp` (и `/var/tmp`, `/dev/shm`) — это мир-перезаписываемые каталоги. Бери писабельную директорию под рабочей папкой процесса (например `/app/run/`), создаваемую в образе под non-root пользователя. См. anti-patterns в SKILL.md.

### Per-aggregate stream-настройки (publisher)

```python
from pydantic import BaseModel, Field


class BasePublisherStreamSettings(BaseModel):
    stream_name: str = "app"
    creation_subject_name: str = "<aggregate>.created"
    update_subject_name: str = "<aggregate>.updated"
    deletion_subject_name: str = "<aggregate>.deleted"
    restoration_subject_name: str = "<aggregate>.restored"

    @property
    def creation_subject(self) -> str:
        return f"{self.stream_name}.{self.creation_subject_name}"

    @property
    def update_subject(self) -> str:
        return f"{self.stream_name}.{self.update_subject_name}"

    @property
    def deletion_subject(self) -> str:
        return f"{self.stream_name}.{self.deletion_subject_name}"

    @property
    def restoration_subject(self) -> str:
        return f"{self.stream_name}.{self.restoration_subject_name}"

    @property
    def subjects(self) -> list[str]:
        return [
            self.creation_subject,
            self.update_subject,
            self.deletion_subject,
            self.restoration_subject,
        ]


class <Aggregate>PublisherStreamSettings(BasePublisherStreamSettings):
    creation_subject_name: str = "<aggregate>.created"
    update_subject_name: str = "<aggregate>.updated"
    deletion_subject_name: str = "<aggregate>.deleted"
    restoration_subject_name: str = "<aggregate>.restored"


class PublisherStreamSettings(BaseModel):
    <aggregate1>: <Aggregate1>PublisherStreamSettings = Field(
        default_factory=<Aggregate1>PublisherStreamSettings
    )
    <aggregate2>: <Aggregate2>PublisherStreamSettings = Field(
        default_factory=<Aggregate2>PublisherStreamSettings
    )
```

### Per-aggregate stream-настройки (consumer)

```python
class <Aggregate>ConsumerStreamSettings(BaseModel):
    stream_name: str = "<source_service>"
    creation_subject_name: str = "<aggregate>.created"
    update_subject_name: str = "<aggregate>.updated"

    @property
    def creation_subject(self) -> str:
        return f"{self.stream_name}.{self.creation_subject_name}"

    @property
    def update_subject(self) -> str:
        return f"{self.stream_name}.{self.update_subject_name}"


class ConsumerStreamSettings(BaseModel):
    <aggregate1>: <Aggregate1>ConsumerStreamSettings = Field(
        default_factory=<Aggregate1>ConsumerStreamSettings
    )
```

**Замечания:**
- Consumer-stream обычно отличается составом subject-ов (приходит то, что соседний сервис публикует).
- Publisher-stream — фиксированный набор lifecycle-событий (created/updated/deleted/restored).
- `stream_name` для publisher-а — имя текущего сервиса; для consumer-а — имя источника.
- Поля `*_subject_name` содержат весь путь после stream через точку, например `user.created`.
- Итоговый subject вычисляется только как `<stream_name>.<subject_name>`; отдельный `main_subject_name` не используется.

## Postgres

```python
# infrastructure/config/db.py
from os import path
from typing import Self

from pydantic import BaseModel, Field, model_validator


class PostgresPoolSettings(BaseModel):
    min_size: int = 5
    max_size: int = 20
    max_inactive_connection_lifetime: int = 300
    max_connection_lifetime: int = 3600
    timeout: int = 20


class PostgresSettings(BaseModel):
    host: str = "localhost"
    port: int = 5432
    user: str = "app"
    password_file: str = "/run/secrets/db_password"
    database: str = "app_db"

    pool: PostgresPoolSettings = Field(default_factory=PostgresPoolSettings)

    @model_validator(mode="after")
    def validate_password_file(self) -> Self:
        if not path.isfile(self.password_file):
            raise ValueError(f"Password file not found: {self.password_file}")
        return self

    @property
    def password(self) -> str:
        with open(self.password_file, "r", encoding="utf-8") as file:
            return file.read().strip()

    @property
    def url(self) -> str:
        return (
            f"postgresql://{self.user}:{self.password}"
            f"@{self.host}:{self.port}/{self.database}"
        )

    @property
    def url_with_psycopg(self) -> str:
        return (
            f"postgresql+psycopg://{self.user}:{self.password}"
            f"@{self.host}:{self.port}/{self.database}"
        )
```

**Типичные поля:** `host`, `port`, `user`, `password_file`, `database`, `pool` (под-блок).

**Pool-параметры:** `min_size`, `max_size`, `max_inactive_connection_lifetime`, `max_connection_lifetime`, `timeout`.

**Производные:** `password` (через файл), `url`, `url_with_psycopg` (или с другими драйверами по необходимости).

## SMTP

```python
# infrastructure/config/smtp.py
from os import path
from typing import Self

from pydantic import BaseModel, model_validator


class SmtpSettings(BaseModel):
    host: str = "localhost"
    port: int = 587
    username: str = ""
    password_file: str = "/run/secrets/smtp_password"
    sender: str = ""
    use_tls: bool = True
    timeout: int = 10

    @model_validator(mode="after")
    def validate_password_file(self) -> Self:
        if not path.isfile(self.password_file):
            raise ValueError(f"Password file not found: {self.password_file}")
        return self

    @property
    def password(self) -> str:
        with open(self.password_file, encoding="utf-8") as f:
            return f.read().strip()
```

**Типичные поля:** `host`, `port`, `username`, `password_file`, `sender`, `use_tls`, `timeout`.

**Производные:** `password` (через файл, file-based secret).

**Замечания:** `use_tls` используется как флаг для `aiosmtplib`. `sender` — адрес отправителя (`From:`).

## Subscription Worker

```python
# infrastructure/config/subscriptions.py
from pydantic import BaseModel


class SubscriptionSettings(BaseModel):
    healthcheck_file: str = "/app/run/subscription_worker_healthbeat"
    loop_sleep_duration: int = 10
```

**Типичные поля:** `healthcheck_file` (путь для liveness-probe), `loop_sleep_duration` (период main-loop-а).

Применимо для любого фонового воркера, основанного на periodic-loop без внешнего брокера.

## Шаблон для новой технологии

При добавлении новой технологии (Redis, RabbitMQ, S3, OpenSearch, и т.п.):

1. Согласуй с пользователем набор полей. Опирайся на:
   - Параметры подключения: `host`, `port`, `protocol`, `tls`, и т.п.
   - Аутентификация: `user` + `password_file` (если требуется).
   - Pool / connection-параметры: timeout, retry, max-connections.
   - Healthcheck / observability: пути файлов, имена для трассировки.
   - Per-aggregate variation: если есть smart-routing/partitioning.
2. Добавь блок в этот файл по образцу выше:
   - Заголовок секции `## <Tech name>` (по алфавиту с существующими секциями).
   - Полный код-блок (`<concern>.py`).
   - Список «Типичные поля» в конце.
3. Если есть pool-настройки — отдельный `<Tech>PoolSettings` под-блок.
4. Если есть file-based secret — паттерн `<x>_file` + `@property` + `@model_validator`.
5. Computed values — через `@property`.

### Шаблон секции

```markdown
## <Tech>

\`\`\`python
# infrastructure/config/<concern>.py
from pydantic import BaseModel, Field


class <Tech>Settings(BaseModel):
    host: str = "localhost"
    port: int = ...
    # ... другие согласованные поля

    @property
    def url(self) -> str:
        return f"<scheme>://{self.host}:{self.port}"
\`\`\`

**Типичные поля:** ...
```
