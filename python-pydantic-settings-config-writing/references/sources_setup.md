# Sources Setup

## AppBaseSettings — общий предок top-level воркер-классов

Один файл `infrastructure/config/workers.py` содержит `AppBaseSettings` и все `<Worker>WorkerSettings`. `AppBaseSettings` фиксирует:
- `model_config` (yaml_file path, encoding, extra-policy).
- `settings_customise_sources` — отключает все источники кроме YAML.

```python
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

Все `<Worker>WorkerSettings` в `workers.py` наследуются от `AppBaseSettings`, не от `BaseSettings` напрямую — чтобы единая customization автоматически применилась к каждому.

## CONFIG_FILE env

Путь к YAML — единственная переменная окружения, которую читает приложение. Воркер запускается как:

```bash
CONFIG_FILE=/etc/app/api_worker.yaml python -m presentation.api_worker
```

`getenv("CONFIG_FILE")` возвращает `None`, если переменная не задана. В этом случае `YamlConfigSettingsSource` вернёт пустые значения, и каждое поле возьмёт свой default. Это допустимо в тестах (с фикстурой, заполняющей всё через `default_factory`), но неприемлемо в production — добавь в `presentation/`-слой явную проверку наличия `CONFIG_FILE` на старте.

## Параметры `model_config`

```python
model_config = SettingsConfigDict(
    yaml_file=getenv("CONFIG_FILE"),
    yaml_file_encoding="utf-8",
    extra="ignore",
)
```

| Параметр | Значение | Зачем |
|---|---|---|
| `yaml_file` | path из `CONFIG_FILE` | путь к YAML-источнику |
| `yaml_file_encoding` | `"utf-8"` | для надёжного чтения unicode-значений |
| `extra` | `"ignore"` | YAML может содержать ключи под другие воркеры/среды; не падаем на лишних |

`extra="ignore"` — намеренно: один YAML может быть общим для нескольких воркеров (если так удобно ops), и каждый top-level забирает только свои блоки, остальное игнорирует.

## settings_customise_sources

Метод-классметод pydantic-settings, возвращающий tuple источников **в порядке приоритета** (первым — наивысший).

В YAML-only каноне tuple содержит **только** `YamlConfigSettingsSource`. Это значит:
- env-переменные для отдельных полей — игнорируются (нет `env_settings` в tuple).
- dotenv-файлы — игнорируются.
- pydantic secret-файлы (через `secrets_dir`) — игнорируются (используется свой file-based secrets паттерн через `@property`, см. `nested_settings.md`).
- init-значения из `__init__` — игнорируются (нельзя передать поля прямо в `<Worker>WorkerSettings(field=value)`, кроме как для тестов с подмешиванием через `model_construct`).

Результат: единственный источник — YAML. Полностью предсказуемо, только один файл.

## Когда отступать от YAML-only канона

YAML-only — канон, но возможны исключения **по согласованию с пользователем**:

### Тесты

В тестовых фикстурах settings не должны зависеть от файловой системы. Варианты:

```python
# Способ 1: model_construct (без валидации)
settings = APIWorkerSettings.model_construct(
    fastapi=FastAPISettings(),
    uvicorn=UvicornSettings(host="127.0.0.1", port=18000),
    db=PostgresSettings.model_construct(...),
)

# Способ 2: переопределение settings_customise_sources в test-фикстуре
class TestAPIWorkerSettings(APIWorkerSettings):
    @classmethod
    def settings_customise_sources(cls, settings_cls, *args, **kwargs):
        return (init_settings,)  # init_settings — единственный источник
```

### Локальная разработка с dotenv

Если для dev-окружения хочется `.env`-файл — добавь `DotEnvSettingsSource` в tuple **с явной пометкой** в коде и docstring-ом, что включается только в dev-режиме:

```python
@classmethod
def settings_customise_sources(
    cls,
    settings_cls,
    init_settings,
    env_settings,
    dotenv_settings,
    file_secret_settings,
):
    if getenv("ENV") == "dev":
        return (dotenv_settings, YamlConfigSettingsSource(...))
    return (YamlConfigSettingsSource(...),)
```

### Cloud-провайдер с native secret-store

Для AWS Parameter Store / GCP Secret Manager / HashiCorp Vault — кастомный `PydanticBaseSettingsSource` подкласс, добавляется в tuple дополнительно к YAML. Реализация подкласса — отдельная задача, выходит за рамки скила.

В каждом случае:
- Изменение `settings_customise_sources` явно прокомментировано.
- Приоритеты источников описаны: что может перебить что.
- Документация воркера обновлена.

## Anti-patterns

- `os.getenv("DB_HOST")` где-то в коде помимо `CONFIG_FILE` — все настройки идут через YAML.
- `yaml.safe_load(...)` в обход pydantic-settings — теряется валидация и type coercion.
- Mixed-source tuple, где одни поля идут из YAML, другие из env — выбираем один источник.
- `dotenv_settings` в production-tuple без явной dev-ветки.
- Чтение `CONFIG_FILE` где-то ещё для других целей — переменная закреплена за конфигом.
- Наследование top-level воркера от `BaseSettings` напрямую (минуя `AppBaseSettings`) — теряется единообразие customise_sources.
