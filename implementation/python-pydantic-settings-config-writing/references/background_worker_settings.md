# Настройки runtime и фоновых задач

## Владение параметрами

Top-level settings одного процесса композирует три разных вида блоков:

|Блок|Владеет|Не владеет|
|---|---|---|
|Runtime|Именем процесса, общим graceful shutdown и параметрами сигналов|Расписанием и timeout application-операции|
|Конкретная задача|Интервалом, timeout операции, задержками по результатам, progress heartbeat и readiness|Lifecycle других задач|
|Технология|Подключением, пулом и timeout конкретного клиента|Расписанием рабочей задачи|

Одинаковые значения у двух задач не образуют общий параметр. Общий блок допустим
только при общей семантике и требовании изменять значение для всех задач атомарно.

## Пример моделей

```python
from typing import Annotated

from pydantic import Field


PositiveMilliseconds = Annotated[int, Field(gt=0)]


class RuntimeSettings(ConfigModel):
    shutdown_timeout_ms: PositiveMilliseconds


class PublishEventsTaskSettings(ConfigModel):
    heartbeat_file: Path
    readiness_file: Path
    heartbeat_timeout_ms: PositiveMilliseconds
    idle_delay_ms: PositiveMilliseconds
    retry_delay_ms: PositiveMilliseconds
    operation_timeout_ms: PositiveMilliseconds
    failed_cycles_before_not_ready: Annotated[int, Field(gt=0)]


class CleanupTaskSettings(ConfigModel):
    interval_ms: PositiveMilliseconds
    operation_timeout_ms: PositiveMilliseconds


class BackgroundWorkerSettings(WorkerSettings):
    runtime: RuntimeSettings
    publish_events: PublishEventsTaskSettings
    cleanup: CleanupTaskSettings
    postgres: PostgresSettings
    nats: NatsSettings
```

Не создавать общий `task_timeout_ms`: timeout одной задачи не ограничивает
другую. Не дублировать в task settings `connect_timeout` PostgreSQL или NATS —
это параметры технологических клиентов.

## Пример YAML

```yaml
runtime:
  shutdown_timeout_ms: 30000

publish_events:
  heartbeat_file: /app/run/publisher-heartbeat
  readiness_file: /app/run/publisher-ready
  heartbeat_timeout_ms: 60000
  idle_delay_ms: 1000
  retry_delay_ms: 1000
  operation_timeout_ms: 30000
  failed_cycles_before_not_ready: 3

cleanup:
  interval_ms: 60000
  operation_timeout_ms: 10000
```

Если процесс содержит только одну задачу, её блок всё равно отделять от runtime:
это сохраняет владельца параметров и не превращает timeout операции в свойство
всего процесса.
