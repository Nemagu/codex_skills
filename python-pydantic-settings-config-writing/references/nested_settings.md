# Nested Settings

## Базовый класс — BaseModel

Все вложенные блоки наследуются от `pydantic.BaseModel` (не `BaseSettings`).

```python
from pydantic import BaseModel


class <Concern>Settings(BaseModel):
    field_a: str = "default"
    field_b: int = 10
```

Поля типизированы, дефолты явно указаны. `BaseModel` валидирует типы при инстанцировании, не лезет к источникам.

## Композиция через Field(default_factory=...)

Вложенные блоки попадают в родителя через `Field(default_factory=NestedSettings)`:

```python
from pydantic import BaseModel, Field


class <Concern>PoolSettings(BaseModel):
    min_size: int = 5
    max_size: int = 20
    timeout: int = 20


class <Concern>Settings(BaseModel):
    host: str = "localhost"
    port: int = 0
    pool: <Concern>PoolSettings = Field(default_factory=<Concern>PoolSettings)
```

**Почему `default_factory`, а не `pool: <Concern>PoolSettings = <Concern>PoolSettings()`:**
- Default-значение через прямой инстанс — общее для всех экземпляров класса (mutable default бы шарился).
- `default_factory` создаёт новый экземпляр для каждого `<Concern>Settings`, безопасно и предсказуемо.

YAML для такого блока:

```yaml
<concern>:
  host: 127.0.0.1
  port: 5432
  pool:
    min_size: 10
    max_size: 30
```

Если `pool` опущен в YAML — берётся `<Concern>PoolSettings()` со всеми дефолтами.

## @property для derived values

Производные значения (URL, составные имена subject-ов, derived flags) — `@property`, не поля:

```python
class <Concern>Settings(BaseModel):
    host: str
    port: int
    user: str
    database: str
    password_file: str

    @property
    def url(self) -> str:
        return (
            f"<scheme>://{self.user}:{self.password}"
            f"@{self.host}:{self.port}/{self.database}"
        )
```

```python
class <Aggregate>StreamSettings(BaseModel):
    stream_name: str
    creation_subject_name: str = "<aggregate>.created"
    update_subject_name: str = "<aggregate>.updated"

    @property
    def creation_subject(self) -> str:
        return f"{self.stream_name}.{self.creation_subject_name}"

    @property
    def update_subject(self) -> str:
        return f"{self.stream_name}.{self.update_subject_name}"

    @property
    def subjects(self) -> list[str]:
        return [self.creation_subject, self.update_subject]
```

**Почему `@property`, а не поле:**
- Поле сохраняется в YAML и должно дублироваться руками — derived value автоматически пересчитывается при изменении частей.
- Поле требует валидации формата — `@property` собирается из уже валидированных частей.
- Поле в YAML теряет связь с источниками частей.

## File-based secrets

Канон для всех чувствительных значений (пароли, токены, ключи):

```python
from os import path
from typing import Self

from pydantic import BaseModel, model_validator


class <Concern>Settings(BaseModel):
    user: str = "app"
    password_file: str = "/run/secrets/<concern>_password"
    # ... другие поля

    @model_validator(mode="after")
    def validate_password_file(self) -> Self:
        if not path.isfile(self.password_file):
            raise ValueError(f"Password file not found: {self.password_file}")
        return self

    @property
    def password(self) -> str:
        with open(self.password_file, "r", encoding="utf-8") as file:
            return file.read().strip()
```

**Контракт:**
- Поле `<secret>_file: str` — путь, попадает в YAML.
- `@model_validator(mode="after")` — проверяет существование файла на старте процесса (fail-fast при неправильном пути).
- `@property <secret>` — читает файл **при каждом обращении**, не кеширует.
- `.strip()` — снимает trailing newlines, типичную проблему secret-файлов от docker secrets / kubernetes.

**Почему не кешировать:**
- Ротация секрета (внешним процессом, который перезаписывает файл) подхватывается без рестарта.
- Память процесса не хранит plaintext-копию (минус один источник утечки через core dump).

**Что НЕ делать:**
- Не читать файл в `field_validator(mode="before")` для подмены `password_file` на содержимое — это раскрывает секрет в `password: str` поле, теряет преимущество lazy.
- Не объединять `password_file` и `password` в одно поле через `Union[str, ...]`.
- Не определять `password` как `str = Field(default_factory=...)` — снова становится материализованным полем.

## @model_validator (cross-field)

Когда валидация требует **нескольких полей** или **внешнего состояния** (файлов, окружения):

```python
from os import path
from typing import Self

from pydantic import BaseModel, model_validator


class <Concern>Settings(BaseModel):
    host: str
    port: int
    healthcheck_file: str

    @model_validator(mode="after")
    def validate_healthcheck_dir(self) -> Self:
        if not path.isdir(path.dirname(self.healthcheck_file)):
            raise ValueError(
                f"Healthcheck dir does not exist: {path.dirname(self.healthcheck_file)}"
            )
        return self
```

`mode="after"` — валидация после применения всех `field_validator`-ов и type coercion. На входе уже типизированный self.

`mode="before"` — реже нужен; для манипуляции **сырых** входных данных перед coercion.

## @field_validator (single field)

Для валидации/нормализации одного поля:

```python
from pydantic import BaseModel, field_validator


class <Concern>Settings(BaseModel):
    timeout: int

    @field_validator("timeout")
    @classmethod
    def validate_timeout(cls, value: int) -> int:
        if value < 1 or value > 300:
            raise ValueError(f"timeout must be in [1, 300], got {value}")
        return value
```

Использовать сдержанно: type-coercion от pydantic уже покрывает базовые случаи. Добавлять, только если есть domain-специфичное правило (диапазон, формат, нормализация).

## Наследование per-aggregate variation

Когда несколько per-aggregate блоков делят один и тот же набор полей с разными значениями (типичный кейс — stream-ы для разных доменных сущностей):

```python
class BasePublisherStreamSettings(BaseModel):
    stream_name: str = "app"
    creation_subject_name: str = "<aggregate>.created"
    update_subject_name: str = "<aggregate>.updated"

    @property
    def creation_subject(self) -> str:
        return f"{self.stream_name}.{self.creation_subject_name}"

    @property
    def update_subject(self) -> str:
        return f"{self.stream_name}.{self.update_subject_name}"


class <Aggregate1>PublisherStreamSettings(BasePublisherStreamSettings):
    creation_subject_name: str = "<aggregate1>.created"
    update_subject_name: str = "<aggregate1>.updated"


class <Aggregate2>PublisherStreamSettings(BasePublisherStreamSettings):
    creation_subject_name: str = "<aggregate2>.created"
    update_subject_name: str = "<aggregate2>.updated"


class PublisherStreamSettings(BaseModel):
    <aggregate1>: <Aggregate1>PublisherStreamSettings = Field(
        default_factory=<Aggregate1>PublisherStreamSettings
    )
    <aggregate2>: <Aggregate2>PublisherStreamSettings = Field(
        default_factory=<Aggregate2>PublisherStreamSettings
    )
```

**Принцип:** общая структура — в базе; per-aggregate variation — в наследниках с переопределёнными `*_subject_name`. Каждое такое поле содержит весь путь после stream через точку, например `<aggregate>.created`. Группа собирает все per-aggregate блоки.

## Anti-patterns

- `BaseSettings` во вложенном блоке.
- Mutable default напрямую: `pool: <Concern>PoolSettings = <Concern>PoolSettings()` — используем `Field(default_factory=...)`.
- Derived-value в поле: `url: str = ...` — используем `@property`.
- Чтение secret-файла в `field_validator(mode="before")` для подмены поля на содержимое.
- `password: str` поле для секрета — только `<x>_file` + `@property`.
- Кеширование секрета в инстансе после первого чтения.
- `@model_validator(mode="before")` для логики, которая нормально живёт в `mode="after"`.
- Дублирование общих полей между per-aggregate блоками вместо вынесения в базу.
