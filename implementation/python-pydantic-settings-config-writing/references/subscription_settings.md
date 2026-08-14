# Настройки задачи подписок

## Модель

```python
from pathlib import Path
from typing import Annotated

from pydantic import Field


PositiveMilliseconds = Annotated[int, Field(gt=0)]
ExternalPath = Annotated[Path, Field(strict=False)]


class SubscriptionTaskSettings(ConfigModel):
    progress_heartbeat_file: ExternalPath
    interval_ms: PositiveMilliseconds
    operation_timeout_ms: PositiveMilliseconds
```

Добавлять batch size, startup delay и retry policy только по требованиям
конкретной задачи. Общий shutdown timeout хранить в runtime-блоке, а настройки
Postgres/read-порта — в отдельном `PostgresSettings`, не дублировать.

## YAML

```yaml
subscription:
  progress_heartbeat_file: /app/run/subscription_worker_heartbeat
  interval_ms: 10000
  operation_timeout_ms: 30000
```

## Проверки

- Положительный interval.
- Положительный timeout операции.
- Абсолютный runtime path.
- Startup preflight существующего writable parent.
- Задача сама выполняет atomic write progress heartbeat после завершённого цикла;
  settings не создаёт файл.
