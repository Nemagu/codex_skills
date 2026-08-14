# Настройки логирования

## Модель

```python
from enum import StrEnum
from typing import Annotated, Literal

from pydantic import Field


class LoggingLevel(StrEnum):
    DEBUG = "debug"
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"
    CRITICAL = "critical"


class LoggingSettings(ConfigModel):
    level: Annotated[LoggingLevel, Field(strict=False)]
    format: Literal["json", "console"]
    include_traceback: bool
```

Добавлять только поддерживаемые logging adapter-ом поля. Не помещать в config
список secret keys как попытку компенсировать небезопасное логирование.

## YAML

```yaml
logging:
  level: info
  format: json
  include_traceback: true
```

## Правила

- Этот блок только конфигурирует логер; settings и adapter factory не создают
  записей.
- Использовать закрытый набор уровней, не свободный `str`.
- Преобразовывать enum в значение, ожидаемое logging library, в adapter factory.
- Не настраивать handlers при импорте settings.
- Не хранить mutable dict произвольного logging config без точного контракта.
- Не логировать весь settings-объект после настройки.
