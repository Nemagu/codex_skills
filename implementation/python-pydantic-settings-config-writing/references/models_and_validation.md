# Модели и валидация

## Базовые модели

```python
from pydantic import BaseModel, ConfigDict
from pydantic_settings import BaseSettings, SettingsConfigDict


class ConfigModel(BaseModel):
    model_config = ConfigDict(
        extra="forbid",
        frozen=True,
        strict=True,
        validate_default=True,
    )


class WorkerSettings(BaseSettings):
    model_config = SettingsConfigDict(
        extra="forbid",
        frozen=True,
        strict=True,
        validate_default=True,
    )
```

Вложенные блоки наследовать от `ConfigModel`, top-level worker — от
`WorkerSettings`.

## Коллекции и композиция

Использовать неизменяемые коллекции:

```python
from typing import Annotated

from pydantic import Field


class CorsSettings(ConfigModel):
    allow_origins: Annotated[tuple[str, ...], Field(strict=False)]
```

YAML-массив представлен Python-списком. Разрешать только преобразование
контейнера `list → tuple` через `Field(strict=False)`; типы элементов остаются
строгими. Аналогично явно разрешать строковое представление `Path`/`StrEnum`.
Не отключать strict для всей модели.

Обязательную зависимость объявлять без default:

```python
class ApiWorkerSettings(WorkerSettings):
    postgres: PostgresSettings
    uvicorn: UvicornSettings
```

`Field(default_factory=...)` допустим для необязательного блока, который имеет
полный безопасный default. Он не должен скрывать отсутствие обязательной секции.

## Ограничения

```python
from typing import Annotated

from pydantic import Field, model_validator


Port = Annotated[int, Field(ge=1, le=65535)]
PositiveSeconds = Annotated[float, Field(gt=0)]
```

Простые диапазоны выражать декларативно. Cross-field правило:

```python
@model_validator(mode="after")
def validate_pool_size(self) -> Self:
    if self.min_size > self.max_size:
        raise ValueError("min_size не может превышать max_size")
    return self
```

Не выполнять filesystem/network I/O из validator-а.

## Строгость источников

YAML должен содержать значения нужных типов. Для env source строки преобразуются
отдельным контролируемым parser-ом. Не ослаблять все модели ради env.

Проверять defaults. Не использовать `"false"` как bool и `"20"` как int в YAML.

## Закрытые наборы

Использовать `StrEnum` для переиспользуемого контракта и `Literal` для локального
малого набора. Infrastructure enum не импортирует domain/application enum.

## Совместимость

AliasChoices допустим на переходный период. Конфликт старого и нового ключа
должен отклоняться отдельным validator/source check. После миграции alias удалить,
чтобы `extra="forbid"` обнаруживал устаревшую конфигурацию.
