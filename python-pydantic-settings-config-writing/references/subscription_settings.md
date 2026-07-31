# Настройки воркера подписок

## Модель

```python
from pathlib import Path
from typing import Annotated

from pydantic import Field


PositiveSeconds = Annotated[float, Field(gt=0)]
ExternalPath = Annotated[Path, Field(strict=False)]


class SubscriptionSettings(ConfigModel):
    healthcheck_file: ExternalPath
    interval_seconds: PositiveSeconds
```

Добавлять batch size, startup delay и retry policy только по требованиям
конкретного worker-а. Настройки его Postgres/read-порта композировать отдельным
`PostgresSettings`, не дублировать.

## YAML

```yaml
subscription:
  healthcheck_file: /app/run/subscription_worker_healthbeat
  interval_seconds: 10.0
```

## Проверки

- Положительный interval.
- Абсолютный runtime path.
- Startup preflight существующего writable parent.
- Worker сам выполняет atomic write heartbeat; settings не создаёт файл.
