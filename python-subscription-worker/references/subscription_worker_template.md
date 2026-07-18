# Subscription Worker Template

Эталонные файлы periodic-loop background-воркера без брокера сообщений. Имена `<aggregate>` / `<Aggregate>` / `<operation>` подставляй под реальные синхронизируемые проекции/агрегаты проекта.

`BackgroundBaseWorker` (`src/presentation/background/base.py`) тут не дублируется — берётся из [python-nats-worker] → `references/nats_worker_template.md`.

## `src/infrastructure/config/subscriptions.py`

```python
"""Модуль `infrastructure/config/subscriptions.py` слоя инфраструктуры."""

from pydantic import BaseModel


class SubscriptionSettings(BaseModel):
    healthcheck_file: str = "/app/run/subscription_worker_healthbeat"
    loop_sleep_duration: int = 10
```

## `src/infrastructure/config/workers.py` (фрагмент)

```python
class SubscriptionWorkerSettings(AppBaseSettings):
    subscription: SubscriptionSettings = Field(default_factory=SubscriptionSettings)
    db: PostgresSettings = Field(default_factory=PostgresSettings)
```

## `src/presentation/background/subscriptions.py`

```python
"""Модуль `presentation/background/subscriptions.py` слоя представления."""

import asyncio
from collections.abc import Callable, Coroutine
from logging import getLogger
from typing import Any

from application.commands.private.<aggregate> import (
    <Aggregate>CreationUseCase,
    <Aggregate>UpdateUseCase,
)
from application.errors import AppError, AppInternalError
from domain.errors import DomainError
from infrastructure.config import SubscriptionWorkerSettings
from infrastructure.db.postgres import PostgresConnectionManager, PostgresUnitOfWork
from presentation.background.base import BackgroundBaseWorker

logger = getLogger(__name__)

Handler = Callable[[], Coroutine[Any, Any, None]]


class SubscriptionWorker(BackgroundBaseWorker):
    """Periodic-loop воркер синхронизации внутренних проекций."""

    def __init__(self, settings: SubscriptionWorkerSettings) -> None:
        super().__init__(settings.subscription.healthcheck_file)
        self._subscription_settings = settings.subscription
        self._db_manager = PostgresConnectionManager(settings.db)

    async def setup(self) -> None:
        await self._db_manager.init()

    async def complete(self) -> None:
        await self._db_manager.close()

    def _create_tasks(self) -> None:
        handlers = [
            (self._handle_<aggregate>_creation, "<aggregate>_creation"),
            (self._handle_<aggregate>_update, "<aggregate>_update"),
            # (handler, name) ...
        ]
        for handler, name in handlers:
            self._tasks.append(asyncio.create_task(self._subscribe(handler, name)))

    async def _subscribe(self, handler: Handler, name: str) -> None:
        while not self._shutdown_event.is_set():
            self._update_heartbeat()
            processing_error: BaseException | None = None
            try:
                await handler()
            except asyncio.CancelledError:
                return
            except BaseException as err:
                processing_error = err
            self._log_processing_error(name, processing_error)
            await asyncio.sleep(self._subscription_settings.loop_sleep_duration)

    async def _handle_<aggregate>_creation(self) -> None:
        async with self._db_manager.connection() as conn:
            await <Aggregate>CreationUseCase(PostgresUnitOfWork(conn)).execute()

    async def _handle_<aggregate>_update(self) -> None:
        async with self._db_manager.connection() as conn:
            await <Aggregate>UpdateUseCase(PostgresUnitOfWork(conn)).execute()

    def _log_processing_error(self, name: str, error: BaseException | None) -> None:
        if error is None:
            return
        if isinstance(error, DomainError):
            logger.warning(
                "[%s] domain error: %s, struct_name: %s, data=%s",
                name,
                error.msg,
                error.struct_name,
                error.data,
            )
            return
        if isinstance(error, AppInternalError):
            logger.error(
                "[%s] app internal error: %s, action: %s, data=%s",
                name,
                error.msg,
                error.action,
                error.data,
            )
            return
        if isinstance(error, AppError):
            logger.warning(
                "[%s] app error: %s, action: %s, data=%s",
                name,
                error.msg,
                error.action,
                error.data,
            )
            return
        logger.error("[%s] handler error: %s", name, error)
```

## `src/main.py` (фрагмент)

```python
from asyncio import Runner
from os import getenv

from uvloop import new_event_loop

from infrastructure.config import SubscriptionWorkerSettings
from presentation.background.subscriptions import SubscriptionWorker


def main() -> None:
    mode = getenv("MODE", "api")
    match mode:
        case "subscriptions_worker":
            subscriptions_settings = SubscriptionWorkerSettings()
            apply_db_migrations(subscriptions_settings.db)
            worker = SubscriptionWorker(subscriptions_settings)
            with Runner(loop_factory=new_event_loop) as runner:
                runner.run(worker.run())
```

## `configs/subscriptions.worker.example.yaml` (фрагмент)

```yaml
subscription:
  healthcheck_file: /app/run/subscription_worker_healthbeat
  loop_sleep_duration: 10

db:
  host: 127.0.0.1
  port: 5432
  user: <service>
  database: <service>
  password_file: /run/secrets/db_password
  pool:
    min_size: 5
    max_size: 20
    timeout: 20
```

## Заметки по адаптации

- Замени `<aggregate>` / `<Aggregate>` / `<operation>` на реальные имена синхронизируемых проекций (`tenant`, `Tenant`, `creation` / `update` / `sync`).
- Подписок (`handlers` в `_create_tasks`) может быть сколько угодно — обычно 3–10 на воркер. Один use case = один handler = одна `asyncio.Task`.
- Use case-ы импортируются по локальной структуре проекта; типично это `application.commands.private.<aggregate>` — но если в проекте subscription-use case-ы живут в отдельном пакете (`application.subscriptions.<aggregate>`), используй его.
- `execute()` без аргументов — стандарт для subscription-use case-ов: они сами читают «что синхронизировать» из таблицы. Если use case требует аргумент — это не subscription, посмотри на NATS-consumer-паттерн ([python-nats-worker]).
- Если в проекте появится non-subscription periodic-loop (например, GC/cleanup) — копируй этот же шаблон, переименовав файл и класс; `BackgroundBaseWorker` остаётся общим.
- Дефолт `healthcheck_file` не оставляй в `/tmp`: писабельная папка под рабочей директорией процесса, создаваемая в образе под non-root (см. [python-pydantic-settings-config-writing] → anti-patterns).
- Миграции применяются в `main.py` до `worker.run()`, чтобы инстансы не гонялись за yoyo-блокировкой.
