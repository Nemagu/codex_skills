# Источники и загрузчик

## Выбор источников

`settings_customise_sources` возвращает источники от высшего приоритета к
низшему. Перечислять только согласованные источники.

Для источников, встроенных в класс, переопределять
`settings_customise_sources`. Не читать selector источника в class body:

```python
@classmethod
def settings_customise_sources(
    cls,
    settings_cls: type[BaseSettings],
    init_settings: PydanticBaseSettingsSource,
    env_settings: PydanticBaseSettingsSource,
    dotenv_settings: PydanticBaseSettingsSource,
    file_secret_settings: PydanticBaseSettingsSource,
) -> tuple[PydanticBaseSettingsSource, ...]:
    return (env_settings, init_settings)
```

Первый элемент имеет высший приоритет. Конкретный набор определяется контрактом.

## Загрузчик

Loader проверяет selector источника до создания модели:

```python
from collections.abc import Mapping
from os import environ
from pathlib import Path
from typing import TypeVar

from pydantic_settings import YamlConfigSettingsSource


SettingsT = TypeVar("SettingsT", bound=WorkerSettings)


class SettingsLoader:
    def __init__(self, environment: Mapping[str, str] = environ) -> None:
        self._environment = environment

    def load(self, settings_type: type[SettingsT]) -> SettingsT:
        raw_path = self._environment.get("CONFIG_FILE")
        if raw_path is None:
            raise ValueError("CONFIG_FILE не задан")
        config_file = Path(raw_path)
        if not config_file.is_absolute():
            raise ValueError("CONFIG_FILE должен быть абсолютным путём")
        if not config_file.is_file():
            raise ValueError("файл конфигурации недоступен")
        source = YamlConfigSettingsSource(
            settings_cls=settings_type,
            yaml_file=config_file,
            yaml_file_encoding="utf-8",
        )
        return settings_type.model_validate(source())
```

Так loader и source используют один Path, а тест передаёт собственный
`Mapping[str, str]`. YAML всё равно читает штатный source pydantic-settings.

## Другие стратегии

- YAML + env overrides: собрать оба source и явно задать приоритет.
- Init-only tests: использовать отдельный test settings/source, а не
  `model_construct()`.
- Dotenv: включать только в согласованном локальном режиме.
- Cloud secret store: отдельный `PydanticBaseSettingsSource`.

Не смешивать источники без таблицы приоритетов и теста одинакового поля из
нескольких источников.

## Ошибки

Не подавлять `KeyError`, `SettingsError`, YAML parser errors и `ValidationError`.
Startup-граница логирует один безопасный отчёт и завершает процесс до создания
runtime-ресурсов.

Не включать содержимое YAML или исходное значение поля в собственные сообщения.
