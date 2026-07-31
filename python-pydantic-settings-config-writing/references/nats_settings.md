# Настройки NATS и JetStream

## Подключение

```python
from pathlib import Path
from typing import Annotated

from pydantic import Field


Port = Annotated[int, Field(ge=1, le=65535)]
PositiveSeconds = Annotated[float, Field(gt=0)]
ExternalPath = Annotated[Path, Field(strict=False)]


class NatsServerSettings(ConfigModel):
    host: str
    port: Port

    @property
    def url(self) -> str:
        return f"nats://{self.host}:{self.port}"


class NatsSettings(ConfigModel):
    servers: Annotated[
        tuple[NatsServerSettings, ...],
        Field(strict=False),
    ]
    connection_name: str
    connect_timeout_seconds: PositiveSeconds
    reconnect_wait_seconds: PositiveSeconds
    ping_interval_seconds: PositiveSeconds
    max_outstanding_pings: Annotated[int, Field(gt=0)]
    healthcheck_file: ExternalPath
```

Набор server URLs собирает adapter factory. Не хранить credentials/token в URL;
для них использовать отдельный file secret/provider по требованиям.

## JetStream

Не считать lifecycle subjects универсальными. Описывать только заданные
направления:

```python
class StreamSettings(ConfigModel):
    stream_name: str
    subjects: Annotated[tuple[str, ...], Field(strict=False)]


class ConsumerSettings(ConfigModel):
    stream_name: str
    consumer_name: str
    filter_subjects: Annotated[tuple[str, ...], Field(strict=False)]
    batch_size: Annotated[int, Field(gt=0)]
    fetch_timeout_seconds: PositiveSeconds
```

Не создавать per-aggregate наследника, если различаются только значения YAML.
Отдельная модель оправдана различием структуры или валидации.

## YAML

```yaml
nats:
  servers:
    - host: nats-1.internal
      port: 4222
    - host: nats-2.internal
      port: 4222
  connection_name: service-consumer
  connect_timeout_seconds: 5.0
  reconnect_wait_seconds: 2.0
  ping_interval_seconds: 120.0
  max_outstanding_pings: 3
  healthcheck_file: /app/run/nats_consumer_healthbeat

consumer:
  stream_name: source_service
  consumer_name: current_service
  filter_subjects:
    - source.entity.created
    - source.entity.updated
  batch_size: 100
  fetch_timeout_seconds: 5.0
```

Имена stream/consumer/subjects являются deployment-контрактом и не должны иметь
placeholder-defaults в production-модели.

## Проверки

- Непустой tuple servers и subjects, если это требуется клиенту.
- Уникальность server URLs и subjects по требованиям.
- Положительные durations и batch size.
- Preflight родителя healthcheck-файла.
- Явное преобразование settings в kwargs установленной версии nats-py.
