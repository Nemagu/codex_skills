---
name: python-pydantic-settings-config-writing
description: Используй при проектировании или правке конфигурации в слое адаптеров (`infrastructure/config/`) Python-проекта на pydantic-settings. Триггеры — добавление или изменение настроек технологии (БД, брокер, web-сервер, и т.п.), top-level worker-настроек, source-customization для YAML/env, file-based secrets, иерархических composition-блоков. Не применять для логики `domain/`, `application/`, `presentation/`.
---

# Python Pydantic-Settings Config Writing

## Quick Start

1. Размести конфиг в `infrastructure/config/`. Per-technology блок — отдельный файл `<concern>.py` (snake_case singular).
2. Two-tier правило: **`BaseSettings`** (через общий `AppBaseSettings`) — **только** для top-level worker-настроек. **`BaseModel`** — для всех вложенных concern-блоков.
3. Все top-level воркер-классы и `AppBaseSettings` живут в **одном файле** `infrastructure/config/workers.py`. Не разносим по файлам.
4. `infrastructure/config/__init__.py` — **витрина**, только re-export через `__all__`. Никаких определений.
5. Источник истины — **YAML-only**, путь читается из `CONFIG_FILE` env. `settings_customise_sources` явно отключает env/dotenv/secrets.
6. Per-worker top-level: один воркер = один config-класс = один YAML-файл. Воркер не таскает лишних блоков.
7. Composition вложенных блоков — через `Field(default_factory=NestedSettings)`.
8. Computed values (`url`, `subjects`, derived flags) — `@property`, **не** поля. Чтение secret-файлов — тоже `@property`.
9. Секреты (пароли, токены, ключи) — **file-based**: поле `<x>_file: str`, `@property <x>` читает файл, `@model_validator(mode="after")` проверяет существование. **Plaintext в YAML — anti-pattern.**
   - Дефолтные пути файлов **никогда не в `/tmp`** (и `/var/tmp`, `/dev/shm`) — мир-перезаписываемые каталоги. Secret-файлы → `/run/secrets/<name>`; рантайм-файлы, которые процесс пишет (healthcheck-маячки, pid) → писабельная директория под рабочей папкой процесса (`/app/run/<name>`), создаваемая в образе под non-root.
10. Naming: блоки `<Concern>Settings` / `<Concern><Part>Settings`; top-level — `<WorkerKind>WorkerSettings`.
11. Поля конкретного блока (имена, типы, дефолты) **определяются с пользователем** — не выдумывай. Если технология типовая (Postgres, NATS, Redis, FastAPI/Uvicorn и т.п.) — проактивно предложи стандартный набор с опорой на `references/tech_examples.md`.
12. Запрещено: `BaseSettings` во вложенных блоках, env-переменные для отдельных полей помимо `CONFIG_FILE`, plaintext-секреты в YAML, определения top-level воркеров в `__init__.py`.

## When to Apply

### Триггеры активации

**По расположению файлов** — любая правка/создание в:
- `infrastructure/config/__init__.py`
- `infrastructure/config/workers.py`
- `infrastructure/config/<concern>.py`
- `configs/*.yaml` (YAML-примеры воркеров)

**По концепциям, упомянутым пользователем:**
- pydantic-settings, BaseSettings, BaseModel.
- Настройки приложения, environment-конфиг, YAML-конфиг.
- Подключение к БД / брокеру / web-серверу — параметры подключения.
- Pool-настройки, retry/timeout, healthcheck-файлы.
- Секреты, password file, file-based secret.
- Source customization, settings_customise_sources.

**По типам задач:**
- Добавление нового concern-блока (новая технология/инфраструктурный компонент).
- Добавление нового top-level worker-конфига.
- Добавление нового поля в существующий блок.
- Изменение source-стратегии (YAML / env / mixed).
- Добавление computed property или валидатора.
- Перевод plaintext-секрета в file-based.

### Анти-триггеры

Скил не активируется, когда правка относится к:
- `domain/` — доменная модель.
- `application/` — use cases, ports, application DTO.
- `presentation/` — реализация воркеров (HTTP-handlers, NATS-consumers, и т.п.). Сами файлы воркеров **используют** settings, но не определяют структуру конфига.
- Косметические правки (docstrings, переименование без изменения контракта).

### Предусловия

- Гексагональная архитектура с выделенным `infrastructure/`-слоем.
- Используется `pydantic` + `pydantic-settings`.
- Один runtime-процесс = один воркер с одним конфигом, читаемым из YAML по `CONFIG_FILE`.
- Секреты доступны как файлы (через secret-mount, vault-CSI, или dev-bind).

## Package Structure

```
infrastructure/config/
├── __init__.py                              ← НЕ пустой: ТОЛЬКО re-export через __all__
├── workers.py                               ← AppBaseSettings + все <Worker>WorkerSettings
└── <concern>.py                             ← BaseModel-блоки на технологию
                                               (db.py, nats.py, fastapi.py, ...)
```

### Назначение узлов

| Узел | Что внутри |
|---|---|
| `__init__.py` | re-export всех settings через `__all__` (алфавитный) |
| `workers.py` | `AppBaseSettings` (наследник `BaseSettings` с `settings_customise_sources`) + все `<Worker>WorkerSettings` |
| `<concern>.py` | `BaseModel`-наследники: главный блок + (опционально) под-блоки и наследники для per-aggregate вариаций |

### Naming convention

| Тип | Файл | Класс |
|---|---|---|
| Общая база | `workers.py` | `AppBaseSettings` (наследник `BaseSettings`) |
| Top-level worker | `workers.py` (там же) | `<WorkerKind>WorkerSettings` (наследник `AppBaseSettings`) |
| Concern-блок главный | `<concern>.py` | `<Concern>Settings` (наследник `BaseModel`) |
| Под-блок concern-а | `<concern>.py` (там же) | `<Concern><Part>Settings` (наследник `BaseModel`) |
| Per-aggregate variation | `<concern>.py` (там же) | `<Aggregate><Concern><Part>Settings` (наследник базового либо общего блока) |

## Two-tier паттерн: BaseSettings vs BaseModel

| Уровень | Базовый класс | Где живёт | Что умеет |
|---|---|---|---|
| **Top-level** (worker config) | `BaseSettings` (через `AppBaseSettings`) | `workers.py` | читает источники (YAML), композирует concern-блоки |
| **Nested** (concern-блок и под-блоки) | `BaseModel` | `<concern>.py` | структура + валидация + `@property` derived values |

**Правило:** только top-level — `BaseSettings`. Все вложенные блоки — `BaseModel`. Никогда не вкладывайте `BaseSettings` друг в друга.

Почему: `BaseSettings` обращается к источникам (YAML/env/secrets) при инстанцировании. Вложение `BaseSettings` приводит к множественным независимым попыткам чтения источников — поведение хрупкое и зависит от порядка инициализации.

`BaseModel` для вложенных — чистая структура, без побочных эффектов, инициализируется родителем через `Field(default_factory=...)`.

## Sources of truth — YAML-only канон

Один воркер читает **один YAML-файл**, путь — из переменной окружения `CONFIG_FILE`. Все остальные источники (env-вары для отдельных полей, dotenv, secret-файлы pydantic) — отключены.

```python
# infrastructure/config/workers.py (фрагмент)
from os import getenv

from pydantic_settings import (
    BaseSettings,
    PydanticBaseSettingsSource,
    SettingsConfigDict,
    YamlConfigSettingsSource,
)


class AppBaseSettings(BaseSettings):
    model_config = SettingsConfigDict(
        yaml_file=getenv("CONFIG_FILE"),
        yaml_file_encoding="utf-8",
        extra="ignore",
    )

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        return (
            YamlConfigSettingsSource(
                settings_cls=settings_cls,
                yaml_file=getenv("CONFIG_FILE"),
                yaml_file_encoding="utf-8",
            ),
        )
```

**Почему YAML-only канон:**
- Reproducibility: весь конфиг воркера — в одном текстовом файле, версионируется и читается глазами.
- Простота для ops: разные среды (dev/stage/prod) — разные YAML-ы, без чтения переменных по списку.
- Локальный контроль: один файл — один точечный диф.

Подробнее, включая «когда отступать», — `references/sources_setup.md`.

## Per-worker top-level

Один runtime-процесс = один top-level `<Worker>WorkerSettings`. Каждый top-level композирует **только нужные** concern-блоки.

```python
class APIWorkerSettings(AppBaseSettings):
    fastapi: FastAPISettings = Field(default_factory=FastAPISettings)
    uvicorn: UvicornSettings = Field(default_factory=UvicornSettings)
    db: PostgresSettings = Field(default_factory=PostgresSettings)


class BrokerPublisherWorkerSettings(AppBaseSettings):
    nats: NatsSettings = Field(default_factory=NatsSettings)
    publishers: NatsPublisherStreamSettings = Field(default_factory=NatsPublisherStreamSettings)
    db: PostgresSettings = Field(default_factory=PostgresSettings)
```

API-воркер не знает про NATS; broker-publisher не знает про FastAPI. YAML-файл каждого воркера содержит только свой набор блоков.

## File-based secrets

Секреты (пароли, токены, ключи) **никогда** не лежат в YAML в открытом виде. Паттерн:

1. Поле `<secret>_file: str` — путь к файлу с секретом.
2. `@property <secret>` — читает содержимое файла при доступе.
3. `@model_validator(mode="after")` — проверяет существование файла на старте.

**Почему файл, а не env-вар:**
- Secret-mount tools (Docker secrets, K8s secret-volumes, Vault CSI) экспортируют секреты как файлы.
- Файл не попадает в `os.environ`, не утечёт в логи через дамп окружения.
- Ротация — пересоздание файла, без рестарта процесса (если код читает при каждом доступе через `@property`).

Подробнее — `references/nested_settings.md` → File-based secrets.

## Computed properties и validators

**`@property`** — для производных значений: `url`, составные subject-имена, derived flags, **чтение secret-файлов**.

**`@model_validator(mode="after")`** — для cross-field валидации, проверки существования внешних артефактов.

**`@field_validator`** — для валидации одного поля (диапазоны, форматы, нормализация). Использовать сдержанно — type-coercion от pydantic уже покрывает базу.

Подробнее с примерами — `references/nested_settings.md`.

## Поведение при добавлении нового config-блока

Поля конкретного блока (имена, типы, дефолты) определяются пользователем для конкретной технологии и сценария. **Агент:**

1. Спрашивает у пользователя, какие поля нужны.
2. Если технология типовая (Postgres, NATS, Redis, FastAPI/Uvicorn, RabbitMQ и т.п.) — **проактивно предлагает** стандартный набор полей с опорой на `references/tech_examples.md`.
3. Не выдумывает поля без согласования; не копирует дефолты слепо без контекста проекта (особенно `host`, `port`, имена пользователей, пути файлов). Пути файлов в дефолтах не указывает в `/tmp` (см. anti-patterns).
4. Если технология новая — после согласования полей **добавляет** её в `references/tech_examples.md` для последующих использований.

## Общий пример блока

```python
# infrastructure/config/<concern>.py
from pydantic import BaseModel, Field


class <Tech>PoolSettings(BaseModel):
    # Поля пула — определяются с пользователем
    # (типичный набор: min_size, max_size, timeouts)
    min_size: int = ...
    max_size: int = ...


class <Tech>Settings(BaseModel):
    # Поля подключения — определяются с пользователем
    host: str = "localhost"
    port: int = ...

    pool: <Tech>PoolSettings = Field(default_factory=<Tech>PoolSettings)

    @property
    def url(self) -> str:
        # Формат — определяется технологией
        return f"<scheme>://{self.host}:{self.port}"
```

Полные примеры на конкретных технологиях — `references/tech_examples.md`.

## Anti-patterns

### Структура
- `BaseSettings` во вложенном блоке — только top-level worker-классы.
- Top-level worker-настройки в `__init__.py` — только в `workers.py`.
- Определения в `__init__.py` — только re-export.
- Один config-класс на всё приложение — per-worker top-level.
- Разнесение top-level воркеров по разным файлам — все в `workers.py`.

### Источники
- env-переменные для отдельных полей помимо `CONFIG_FILE`.
- dotenv-загрузка автоматически (без явного `settings_customise_sources`).
- Mixed YAML+env с приоритетом — выбираем один источник.
- Чтение source-источников вручную (`yaml.safe_load(...)` в обход pydantic-settings).

### Секреты
- Plaintext-пароль/токен в YAML.
- Plaintext-секрет в env-переменной.
- `password: str` поле без `_file`-суффикса для чувствительных значений.
- Чтение secret-файла в `field_validator(mode="before")` для подмены поля на содержимое — раскрывает секрет в process memory как обычное поле и теряет преимущество lazy-чтения.

### Поля и валидация
- f-строки для derived URL прямо в полях — только через `@property`.
- Cross-field логика в `__init__` или фабрике — `@model_validator(mode="after")`.
- Mutable default напрямую: `pool: <Concern>PoolSettings = <Concern>PoolSettings()` — используем `Field(default_factory=...)`.
- Дефолтный путь файла в `/tmp` / `/var/tmp` / `/dev/shm` — мир-перезаписываемые каталоги (предсказуемое имя файла → symlink/precreate-атака; bandit `B108` это валит). Secret-файлы → `/run/secrets/<name>`; рантайм-файлы, которые процесс пишет, → писабельная папка под рабочей директорией процесса (`/app/run/<name>`), создаваемая в образе под non-root. Никогда не `/tmp/...`.

### Naming
- `<Concern>` без `Settings`-суффикса.
- Top-level воркер без `WorkerSettings`-суффикса.
- Per-aggregate variation без префикса агрегата (`StreamSettings` вместо `<Aggregate>StreamSettings`).

## References

- **`references/sources_setup.md`** — `AppBaseSettings`, `settings_customise_sources`, YAML-источник, `CONFIG_FILE` env, отступления от YAML-only канона.
- **`references/nested_settings.md`** — паттерны `BaseModel`: композиция через `Field(default_factory=...)`, `@property` derived values, `@model_validator`, file-based secrets, наследование per-aggregate.
- **`references/tech_examples.md`** — опорные технологические примеры (Postgres, NATS, FastAPI/Uvicorn, и пополняемый набор). Используются агентом как источник подсказок при выборе полей. Содержит шаблон для добавления новой технологии.
- **`references/checklists.md`** — пошаговые чек-листы добавления нового concern-блока, нового top-level worker-конфига, нового file-based secret-а, новой технологии в `tech_examples.md`.
