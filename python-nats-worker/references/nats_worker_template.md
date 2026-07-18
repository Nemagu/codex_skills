# NATS Worker Template

Эталонные файлы для NATS-воркера в Python-сервисе с гексагональной архитектурой. Бери целиком, заменяй `<aggregate>` / `<Aggregate>` плейсхолдеры на реальные имена агрегатов и use case-ы проекта.

## `src/presentation/background/base.py`

```python
"""Модуль `presentation/background/base.py` слоя представления."""

import asyncio
import os
from abc import ABC, abstractmethod
from logging import getLogger
from signal import SIGINT, SIGTERM

logger = getLogger(__name__)


class BackgroundBaseWorker(ABC):
    """Общий каркас фоновых воркеров: сигналы, lifecycle, heartbeat."""

    def __init__(self, healthcheck_file: str) -> None:
        self._healthcheck_file = healthcheck_file
        self._shutdown_event = asyncio.Event()
        self._tasks: list[asyncio.Task] = []

    @abstractmethod
    async def setup(self) -> None: ...

    @abstractmethod
    async def complete(self) -> None: ...

    @abstractmethod
    def _create_tasks(self) -> None: ...

    async def run(self) -> None:
        loop = asyncio.get_running_loop()
        loop.add_signal_handler(SIGTERM, self._handle_signal)
        loop.add_signal_handler(SIGINT, self._handle_signal)
        await self.setup()
        if self._shutdown_event.is_set():
            await self.complete()
            return
        logger.info("background worker started")
        self._create_tasks()
        await self._shutdown_event.wait()
        logger.info("shutting down...")
        await self._cancel_tasks()
        await self.complete()
        logger.info("background worker stopped")

    async def _cancel_tasks(self) -> None:
        for task in self._tasks:
            task.cancel()
        errors = await asyncio.gather(*self._tasks, return_exceptions=True)
        for error in errors:
            if isinstance(error, asyncio.CancelledError):
                continue
            if isinstance(error, BaseException):
                logger.error("task cancelled error: %s", error)
        self._tasks.clear()

    def _handle_signal(self) -> None:
        logger.info("shutting down...")
        self._shutdown_event.set()

    def _update_heartbeat(self) -> None:
        try:
            open(self._healthcheck_file, "a").close()
            os.utime(self._healthcheck_file, None)
        except Exception as e:
            logger.error("failed to update heartbeat: %s", e)
```

## `src/presentation/background/nats/base.py`

```python
"""Модуль `presentation/background/nats/base.py` слоя представления."""

import asyncio
from logging import getLogger

from nats import connect
from nats.aio.client import Client
from nats.errors import (
    ConnectionClosedError,
    ConnectionDrainingError,
    ConnectionReconnectingError,
    StaleConnectionError,
)
from nats.js import JetStreamContext

from infrastructure.config import NatsSettings, PostgresSettings
from infrastructure.db.postgres import PostgresConnectionManager
from presentation.background.base import BackgroundBaseWorker

logger = getLogger(__name__)

NATS_CONNECTION_ERRORS = (
    ConnectionClosedError,
    ConnectionDrainingError,
    ConnectionReconnectingError,
    StaleConnectionError,
)


class NatsBaseWorker(BackgroundBaseWorker):
    """Базовый воркер с подключением к NATS и PostgreSQL."""

    def __init__(
        self, nats_settings: NatsSettings, db_settings: PostgresSettings
    ) -> None:
        super().__init__(nats_settings.healthcheck_file)
        self._nats_settings = nats_settings
        self._db_manager = PostgresConnectionManager(db_settings)
        self._nc: Client | None = None
        self._js: JetStreamContext | None = None
        self._nats_lock = asyncio.Lock()

    async def setup(self) -> None:
        await self._db_manager.init()
        await self._connect_nats()

    async def complete(self) -> None:
        if self._nc:
            try:
                await asyncio.wait_for(self._nc.close(), timeout=5.0)
            except Exception:
                pass
        await self._db_manager.close()

    async def _events_after_connected(self) -> None:
        return None

    async def _connect_nats(self) -> None:
        async with self._nats_lock:
            if self._nc is not None and self._nc.is_connected:
                return
            logger.info("connecting to nats")
            while not self._shutdown_event.is_set():
                self._update_heartbeat()
                try:
                    self._nc = await asyncio.wait_for(
                        connect(
                            self._nats_settings.url,
                            name=self._nats_settings.connect_name,
                            connect_timeout=self._nats_settings.connect_timeout,
                            reconnect_time_wait=self._nats_settings.reconnect_time_wait,
                            max_reconnect_attempts=0,
                            ping_interval=self._nats_settings.ping_interval,
                            max_outstanding_pings=self._nats_settings.max_outstanding_pings,
                        ),
                        timeout=self._nats_settings.connect_timeout,
                    )
                    self._js = self._nc.jetstream()
                    await self._events_after_connected()
                    logger.info("nats connected")
                    return
                except BaseException as e:
                    logger.warning("nats connection failed: %s", e)
                try:
                    await asyncio.wait_for(
                        self._shutdown_event.wait(),
                        timeout=self._nats_settings.reconnect_time_wait,
                    )
                except asyncio.TimeoutError:
                    pass
```

## `src/presentation/background/nats/consumer.py`

```python
"""Модуль `presentation/background/nats/consumer.py` слоя представления."""

import asyncio
import json
from collections.abc import Callable, Coroutine
from enum import StrEnum
from json import JSONDecodeError
from logging import getLogger
from typing import Any
from uuid import UUID

from nats.aio.msg import Msg
from nats.errors import TimeoutError as NatsTimeoutError
from nats.js.client import JetStreamContext
from pydantic import BaseModel, ValidationError

from application.commands.private.<aggregate> import (
    <Aggregate>CreationCommand,
    <Aggregate>CreationUseCase,
)
from application.errors import AppError, AppInternalError, AppNotFoundError
from domain.errors import DomainError
from infrastructure.config import MessageBrokerConsumerSettings
from infrastructure.db.postgres import PostgresUnitOfWork
from presentation.background.nats.base import NATS_CONNECTION_ERRORS, NatsBaseWorker

logger = getLogger(__name__)

Handler = Callable[[dict], Coroutine[Any, Any, None]]


class MessageResponseType(StrEnum):
    ACK = "ack"
    NAK = "nak"
    TERM = "term"


class _InvalidPayloadError(AppInternalError):
    """Ошибка валидации входящего payload сообщения."""

    pass


class <Aggregate>CreationPayload(BaseModel):
    <aggregate>_id: UUID
    state: str
    version: int


class NatsConsumerWorker(NatsBaseWorker):
    def __init__(self, settings: MessageBrokerConsumerSettings) -> None:
        super().__init__(settings.nats, settings.db)
        self._streams = settings.consumers

    def _create_tasks(self) -> None:
        subjects_handlers = [
            (
                self._streams.<aggregate>.creation_subject,
                self._handle_<aggregate>_creation,
                "<aggregate>_creation",
            ),
            # (subject, handler, name) ...
        ]
        for subject, handler, name in subjects_handlers:
            self._tasks.append(
                asyncio.create_task(self._consume(subject, handler, name))
            )

    async def _fetch_message(
        self, sub: JetStreamContext.PullSubscription, name: str
    ) -> Msg | None:
        try:
            messages = await sub.fetch(
                1, timeout=self._nats_settings.loop_sleep_duration
            )
            return messages[0]
        except NatsTimeoutError:
            return None
        except NATS_CONNECTION_ERRORS:
            raise
        except asyncio.CancelledError:
            raise
        except BaseException as err:
            logger.error("[%s] fetch message error: %s", name, err)
            return None

    async def _consume(self, subject: str, handler: Handler, name: str) -> None:
        sub = None
        while not self._shutdown_event.is_set():
            self._update_heartbeat()
            try:
                if sub is None:
                    sub = await self._js.pull_subscribe(subject)
                msg = await self._fetch_message(sub, name)
            except NATS_CONNECTION_ERRORS:
                logger.warning("[%s] nats connection lost, reconnecting...", name)
                sub = None
                await self._connect_nats()
                continue
            except asyncio.CancelledError:
                return
            except BaseException as err:
                logger.error("[%s] subscribe error: %s", name, err)
                await asyncio.sleep(self._nats_settings.reconnect_time_wait)
                continue

            if msg is None:
                await asyncio.sleep(self._nats_settings.loop_sleep_duration)
                continue

            processing_error: BaseException | None = None
            try:
                await handler(json.loads(msg.data))
            except asyncio.CancelledError:
                return
            except BaseException as err:
                processing_error = err

            response_type = self._resolve_message_response_type(processing_error)
            self._log_processing_error(name, processing_error)
            try:
                response_sent = await self._handle_message_response(
                    msg=msg,
                    response_type=response_type,
                    name=name,
                )
            except asyncio.CancelledError:
                return

            if not response_sent:
                sub = None
                await self._connect_nats()

            await asyncio.sleep(self._nats_settings.loop_sleep_duration)

    async def _handle_<aggregate>_creation(self, payload: dict) -> None:
        <aggregate>_id, state, version = self._extract_<aggregate>_creation_payload(payload)
        command = <Aggregate>CreationCommand(
            <aggregate>_id=<aggregate>_id, state=state, version=version
        )
        async with self._db_manager.connection() as conn:
            await <Aggregate>CreationUseCase(PostgresUnitOfWork(conn)).execute(command)

    def _extract_<aggregate>_creation_payload(
        self, payload: dict
    ) -> tuple[UUID, str, int]:
        try:
            data = <Aggregate>CreationPayload.model_validate(payload)
        except ValidationError as err:
            raise _InvalidPayloadError(
                msg="некорректный payload <aggregate> для потребления",
                action="извлечение данных из сообщения о <aggregate>",
                data={"payload": payload, "errors": err.errors()},
                wrap_error=err,
            )
        return data.<aggregate>_id, data.state, data.version

    def _resolve_message_response_type(
        self, error: BaseException | None
    ) -> MessageResponseType:
        if error is None:
            return MessageResponseType.ACK
        if isinstance(error, (_InvalidPayloadError, JSONDecodeError)):
            return MessageResponseType.TERM
        if isinstance(error, (AppNotFoundError, AppInternalError)):
            return MessageResponseType.NAK
        if isinstance(error, (DomainError, AppError)):
            return MessageResponseType.TERM
        return MessageResponseType.NAK

    async def _handle_message_response(
        self,
        msg: Msg,
        response_type: MessageResponseType,
        name: str,
    ) -> bool:
        try:
            match response_type:
                case MessageResponseType.ACK:
                    await msg.ack()
                case MessageResponseType.NAK:
                    await msg.nak()
                case MessageResponseType.TERM:
                    await msg.term()
            return True
        except NATS_CONNECTION_ERRORS:
            logger.warning(
                "[%s] nats connection lost on message response (%s), reconnecting...",
                name,
                response_type.value,
            )
            return False
        except asyncio.CancelledError:
            raise
        except BaseException as err:
            logger.error(
                "[%s] message response (%s) failed: %s",
                name,
                response_type.value,
                err,
            )
            return True

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
        if isinstance(error, _InvalidPayloadError):
            logger.error(
                "[%s] invalid payload: %s, action: %s, data=%s",
                name,
                error.msg,
                error.action,
                error.data,
            )
            return
        if isinstance(error, JSONDecodeError):
            logger.error("[%s] invalid json payload: %s", name, error)
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

## `src/presentation/background/nats/publisher.py`

```python
"""Модуль `presentation/background/nats/publisher.py` слоя представления."""

import asyncio
from collections.abc import Callable, Coroutine
from logging import getLogger
from typing import Any

from nats.js.api import StreamConfig
from nats.js.errors import NotFoundError

from application.commands.private.<aggregate>.publish import <Aggregate>PublisherUseCase
from application.errors import AppError, AppInternalError
from domain.errors import DomainError
from infrastructure.config import MessageBrokerPublisherSettings
from infrastructure.db.postgres import PostgresUnitOfWork
from infrastructure.message_broker.nats import EventNatsPublisher
from presentation.background.nats.base import NATS_CONNECTION_ERRORS, NatsBaseWorker

logger = getLogger(__name__)

Handler = Callable[[], Coroutine[Any, Any, None]]


class NatsPublisherWorker(NatsBaseWorker):
    def __init__(self, settings: MessageBrokerPublisherSettings) -> None:
        super().__init__(settings.nats, settings.db)
        self._publishers_settings = settings.publishers

    async def _events_after_connected(self) -> None:
        await self._create_streams()

    async def _create_streams(self) -> None:
        streams_subjects: dict[str, list[str]] = {}
        for stream_settings in (
            self._publishers_settings.<aggregate>,
            # ... остальные publisher stream-настройки
        ):
            streams_subjects.setdefault(stream_settings.stream_name, []).extend(
                stream_settings.subjects
            )
        for stream_name, subjects in streams_subjects.items():
            try:
                info = await self._js.stream_info(stream_name)
            except NotFoundError:
                logger.info("creating nats stream: %s", stream_name)
                await self._js.add_stream(
                    config=StreamConfig(name=stream_name, subjects=subjects)
                )
                continue

            # stream уже есть — дополняем недостающие subjects, иначе
            # publish на новый subject упадёт с "no response from stream".
            existing = info.config.subjects or []
            missing = [subject for subject in subjects if subject not in existing]
            if not missing:
                continue
            logger.info("updating nats stream subjects: %s", stream_name)
            info.config.subjects = existing + missing
            await self._js.update_stream(config=info.config)

    def _create_tasks(self) -> None:
        handlers = [
            (self._handle_<aggregate>_publish, "<aggregate>_publish"),
            # (handler, name) ...
        ]
        for handler, name in handlers:
            self._tasks.append(asyncio.create_task(self._publish(handler, name)))

    async def _publish(self, handler: Handler, name: str) -> None:
        while not self._shutdown_event.is_set():
            self._update_heartbeat()
            processing_error: BaseException | None = None
            try:
                await handler()
            except NATS_CONNECTION_ERRORS:
                logger.warning("[%s] nats connection lost, reconnecting...", name)
                await self._connect_nats()
                continue
            except asyncio.CancelledError:
                return
            except BaseException as err:
                processing_error = err
            self._log_processing_error(name, processing_error)
            await asyncio.sleep(self._nats_settings.loop_sleep_duration)

    async def _handle_<aggregate>_publish(self) -> None:
        async with self._db_manager.connection() as conn:
            await <Aggregate>PublisherUseCase(
                PostgresUnitOfWork(conn),
                self._publisher(),
            ).execute()

    def _publisher(self) -> EventNatsPublisher:
        return EventNatsPublisher(
            stream_settings=self._publishers_settings,
            nc=self._nc,
            js=self._js,
        )

    def _log_processing_error(self, name: str, error: BaseException | None) -> None:
        # ... тот же что в consumer, без _InvalidPayloadError/JSONDecodeError веток
        ...
```

## `src/main.py` (фрагмент)

```python
from asyncio import Runner
from logging import CRITICAL, getLogger
from os import getenv

from uvloop import new_event_loop

from infrastructure.config import (
    APIWorkerSettings,
    MessageBrokerConsumerSettings,
    MessageBrokerPublisherSettings,
    PostgresSettings,
)
from presentation.background.nats import NatsConsumerWorker, NatsPublisherWorker


def apply_db_migrations(db_settings: PostgresSettings) -> None:
    if getenv("APPLY_MIGRATIONS", "1") == "1":
        from infrastructure.db.postgres import apply_migrations

        apply_migrations(db_settings)


def main() -> None:
    mode = getenv("MODE", "api")
    match mode:
        case "nats_consumer":
            getLogger("nats").setLevel(CRITICAL)
            consumer_settings = MessageBrokerConsumerSettings()
            apply_db_migrations(consumer_settings.db)
            worker = NatsConsumerWorker(consumer_settings)
            with Runner(loop_factory=new_event_loop) as runner:
                runner.run(worker.run())
        case "nats_publisher":
            getLogger("nats").setLevel(CRITICAL)
            publisher_settings = MessageBrokerPublisherSettings()
            apply_db_migrations(publisher_settings.db)
            worker = NatsPublisherWorker(publisher_settings)
            with Runner(loop_factory=new_event_loop) as runner:
                runner.run(worker.run())
```

## Заметки по адаптации

- Замени `<aggregate>` / `<Aggregate>` на реальные имена агрегатов проекта (`company`, `Company`, и т.п.) в payload-моделях, handler-ах, command-ах, use case-ах.
- На один consumer-воркер обычно несколько subjects (creation/deletion/restoration/...) и несколько агрегатов; добавляй пары в `subjects_handlers` и handler-ы по тому же шаблону.
- На publisher-воркер — несколько агрегатов, один handler на агрегат. `_create_streams` объединяет subjects по `stream_name`, чтобы публикация всего сервиса жила в одном или нескольких именованных streams. При добавлении нового агрегата/subject в уже задеплоенный сервис `_create_streams` обязан дополнять subjects существующего stream через `update_stream` (см. сверку `info.config.subjects`), иначе публикация нового subject падёт с `nats: no response from stream` — stream был создан до появления subject и его список не обновлялся.
- Если payload содержит поля помимо id/state/version (timestamps, denormalised data) — расширяй pydantic-модель и `_extract_*_payload`. Главное правило: одна модель на одно сочетание `(subject, payload-schema)`.
- `_InvalidPayloadError` — оставляй приватным внутри `consumer.py`. Это не application-уровень, его не должны импортировать use case-ы.
- Если в проекте появится non-NATS background-воркер (как `SubscriptionWorker`) — наследуй его прямо от `BackgroundBaseWorker`, минуя `NatsBaseWorker`; этот шаблон в скиле не описан, но `BackgroundBaseWorker` спроектирован переиспользуемым.
- Миграции в `main.py` до `worker.run()` — иначе несколько инстансов будут конкурировать за yoyo-блокировку.
