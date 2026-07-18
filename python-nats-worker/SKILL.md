---
name: python-nats-worker
description: Используй при проектировании или реализации фонового воркера на Python поверх NATS / JetStream в сервисах с гексагональной архитектурой. Триггеры — сборка `NatsConsumerWorker` (pull-подписки, классификация ошибок в ack/nak/term) или `NatsPublisherWorker` (streams creation, publish-loop, publisher-адаптер), общий каркас `BackgroundBaseWorker` (lifecycle, signals, heartbeat, task cancellation), `NatsBaseWorker` (подключение NATS+DB, reconnect). Не применять для HTTP API-воркера ([python-fastapi-api-worker]) и для других background-воркеров без NATS (например, periodic-loop поверх `BackgroundBaseWorker` — паттерн такой же, но NATS-специфика не нужна).
---

# Python NATS Worker

Скил про сборку background-воркера поверх NATS/JetStream в Python-сервисе. Воркер — отдельный runtime-процесс, который вычитывает события (consumer) или публикует их из БД-outbox (publisher). Общий каркас фонового воркера (signal handling, heartbeat, task lifecycle) описан здесь же, потому что NATS-специфика наследует его и не существует отдельно.

За HTTP API отвечает [python-fastapi-api-worker]; за структуру `infrastructure/config/` (включая `NatsSettings`, `NatsConsumerStreamSettings`, `NatsPublisherStreamSettings`) — [python-pydantic-settings-config-writing].

## Quick Start

1. Заведи `BackgroundBaseWorker` (`src/presentation/background/base.py`), если его ещё нет: общий каркас под все фоновые воркеры — `shutdown_event`, SIGTERM/SIGINT, `_cancel_tasks`, `_update_heartbeat`, абстрактные `setup`/`complete`/`_create_tasks`.
2. Заведи `NatsBaseWorker` (`src/presentation/background/nats/base.py`) — наследник `BackgroundBaseWorker`. Хранит `NatsSettings` и `PostgresConnectionManager`; в `setup` поднимает БД и `_connect_nats()`; в `complete` корректно закрывает. `_connect_nats()` крутит цикл реконнекта до `_shutdown_event`. `_events_after_connected()` — пустой extension point.
3. Для consumer-воркера наследуйся от `NatsBaseWorker`: в `_create_tasks` зарегистрируй пары `(subject, handler, name)` через `asyncio.create_task(self._consume(...))`; в `_consume` сделай `pull_subscribe → fetch → handler → классификация ошибки → ack/nak/term` (см. ниже).
4. Для publisher-воркера — наследник `NatsBaseWorker`: переопредели `_events_after_connected()` для создания streams через `StreamConfig`; в `_create_tasks` зарегистрируй pairs `(handler, name)` через `asyncio.create_task(self._publish(...))`; в `_publish` крути heartbeat → handler → log → sleep.
5. Validation входящего payload — pydantic-модель + `_InvalidPayloadError` (приватный наследник `AppInternalError`) для term-on-decode.
6. Idempotency живёт **в use case** (через `INSERT ... ON CONFLICT`, version-чек агрегата), а не в воркере. Воркер просто прогоняет повторные сообщения через use case, который их абсорбирует.
7. Entrypoint (`src/main.py`) запускает воркер через `asyncio.Runner(loop_factory=uvloop.new_event_loop)` под нужным `MODE`.

Полный эталон — [nats_worker_template.md](references/nats_worker_template.md). Матрица ack/nak/term — [error_classification.md](references/error_classification.md).

## When to Apply

### Триггеры активации

- Создание или правка `src/presentation/background/base.py` (`BackgroundBaseWorker`).
- Создание или правка `src/presentation/background/nats/base.py` (`NatsBaseWorker`): подключение, reconnect, общие настройки.
- Создание или правка `src/presentation/background/nats/consumer.py` или `publisher.py`: добавление новой подписки/публикатора, изменение pull-цикла или классификации ошибок.
- Добавление новой пары `(subject, handler)` для consumer-а или `(handler)` для publisher-а.
- Добавление payload-схемы (pydantic-модель + `_extract_*_payload`).
- Изменение entrypoint-логики для NATS-режимов (`MODE=nats_consumer`/`MODE=nats_publisher`).

### Анти-триггеры

- HTTP API-воркер — это [python-fastapi-api-worker].
- Реализация use case-ов, агрегатов, репозиториев — это `domain/`/`application/`/`infrastructure/`-уровни, не presentation.
- Структура `infrastructure/config/nats.py` (`NatsSettings`, stream-настройки) — это [python-pydantic-settings-config-writing] (`references/tech_examples.md` → NATS / JetStream).
- Адаптер `EventNatsPublisher` (`infrastructure/message_broker/nats/publisher.py`) — это инфраструктура; здесь только то, как воркер его использует.

## Package Structure

```
src/presentation/background/
├── base.py                     ← BackgroundBaseWorker — общий каркас фоновых воркеров
└── nats/
    ├── __init__.py             ← re-export NatsConsumerWorker, NatsPublisherWorker
    ├── base.py                 ← NatsBaseWorker — NATS+DB подключение, reconnect, extension point
    ├── consumer.py             ← NatsConsumerWorker — fetch-loop, классификация, ack/nak/term
    └── publisher.py            ← NatsPublisherWorker — streams creation, publish-loop

src/infrastructure/message_broker/nats/
└── publisher.py                ← EventNatsPublisher — низкоуровневый publish (за рамками этого скила)
```

`SubscriptionWorker` (`src/presentation/background/subscriptions.py`) — другой наследник `BackgroundBaseWorker` без NATS-специфики; пример того, что каркас переиспользуется и для periodic-loop воркеров. В этом скиле — только как референс к `BackgroundBaseWorker`.

## BackgroundBaseWorker

Общий каркас всех фоновых воркеров — независимо от NATS. Один экземпляр на runtime-процесс.

Что предоставляет:

- `_shutdown_event: asyncio.Event` — сигнал остановки, на который смотрят все циклы.
- `_tasks: list[asyncio.Task]` — реестр запущенных подзадач.
- `_update_heartbeat()` — `touch`-ит файл `healthcheck_file` (из `<concern>Settings.healthcheck_file`); вызывать в каждой итерации каждой задачи.
- `run()` — главный цикл: ставит SIGTERM/SIGINT-хендлеры → `setup()` → `_create_tasks()` → ждёт `_shutdown_event` → `_cancel_tasks()` → `complete()`.

Что обязан реализовать наследник:

- `setup()` — асинхронная инициализация shared-ресурсов (БД, NATS, etc.).
- `complete()` — корректное закрытие в обратном порядке.
- `_create_tasks()` — создание подзадач через `asyncio.create_task(...)` и регистрация в `self._tasks`.

Особенности:

- `_handle_signal` ставит `_shutdown_event`; долгие `await` в подзадачах должны прерываться через `wait_for(_shutdown_event.wait(), timeout=...)`, а не голым `asyncio.sleep`, если задача может «спать» дольше допустимого времени shutdown.
- `_cancel_tasks` зовёт `task.cancel()` на каждой подзадаче и собирает результаты через `asyncio.gather(..., return_exceptions=True)`. `CancelledError` глотается, остальные ошибки логируются.
- Если `setup()` уже увидел сигнал и `_shutdown_event` поставлен — `run()` сразу зовёт `complete()` и выходит, не создавая tasks.

Полный код — [nats_worker_template.md](references/nats_worker_template.md) → `BackgroundBaseWorker`.

## NatsBaseWorker

Специализация под NATS. Хранит:

- `NatsSettings` (из `infrastructure/config/nats.py`).
- `PostgresConnectionManager` (БД нужна почти всегда — handler-ы делегируют в use case через `UnitOfWork`).
- `nc: Client | None`, `js: JetStreamContext | None`, `_nats_lock: asyncio.Lock`.

Что переопределяет:

- `setup()`: `await self._db_manager.init()` → `await self._connect_nats()`.
- `complete()`: закрыть NATS (через `wait_for(nc.close(), timeout=5)`), затем `db_manager.close()`.

Что добавляет:

- `_connect_nats()` — крутит цикл подключения до `_shutdown_event`. При неудаче — `wait_for(_shutdown_event.wait(), timeout=reconnect_time_wait)` (так шатдаун прерывает ожидание). `max_reconnect_attempts=0` у клиента — реконнект мы делаем сами в цикле.
- `_events_after_connected()` — default no-op; publisher-наследник переопределяет для создания streams.
- `NATS_CONNECTION_ERRORS` — кортеж исключений, на которые нужно реконнектиться: `ConnectionClosedError`, `ConnectionDrainingError`, `ConnectionReconnectingError`, `StaleConnectionError`. Экспортируется из модуля для использования в consumer/publisher.

Важно: `_events_after_connected` зовётся **после** каждого успешного `_connect_nats` (включая реконнекты). Если publisher проверяет наличие stream — он это сделает после каждого реконнекта; если действие идемпотентно (как `stream_info → add_stream`/`update_stream` с проверкой недостающих subjects), это OK; если нет — защищай флагом.

## NatsConsumerWorker

Pull-consumer: цикл `fetch → handler → ack/nak/term`. На каждую подписку — свой `asyncio.Task`.

### Регистрация подписок

```python
def _create_tasks(self) -> None:
    subjects_handlers = [
        (self._streams.<agg>.creation_subject, self._handle_<agg>_creation, "<agg>_creation"),
        (self._streams.<agg>.deletion_subject, self._handle_<agg>_deletion, "<agg>_deletion"),
        # ...
    ]
    for subject, handler, name in subjects_handlers:
        self._tasks.append(asyncio.create_task(self._consume(subject, handler, name)))
```

- `name` — короткий идентификатор для логов (`[<name>]`-префикс).
- Subject читаются из `<aggregate>NatsConsumerStreamSettings.<event>_subject` (см. [python-pydantic-settings-config-writing] → NATS).

### Fetch-loop

```python
async def _consume(self, subject: str, handler: Handler, name: str) -> None:
    sub = None
    while not self._shutdown_event.is_set():
        self._update_heartbeat()
        try:
            if sub is None:
                sub = await self._js.pull_subscribe(subject)
            msg = await self._fetch_message(sub, name)
        except NATS_CONNECTION_ERRORS:
            sub = None
            await self._connect_nats()
            continue
        # ...
        if msg is None:
            await asyncio.sleep(self._nats_settings.loop_sleep_duration)
            continue

        processing_error: BaseException | None = None
        try:
            await handler(json.loads(msg.data))
        except BaseException as err:
            processing_error = err

        response_type = self._resolve_message_response_type(processing_error)
        self._log_processing_error(name, processing_error)
        response_sent = await self._handle_message_response(msg, response_type, name)
        if not response_sent:
            sub = None
            await self._connect_nats()

        await asyncio.sleep(self._nats_settings.loop_sleep_duration)
```

Ключевые правила:

- `pull_subscribe` пересоздаётся при потере соединения (`sub = None`) — старый subscription после реконнекта не действителен.
- `_fetch_message` ловит `NatsTimeoutError` и возвращает `None` (нормальный кейс — нет сообщений); пробрасывает `NATS_CONNECTION_ERRORS` и `CancelledError` наружу.
- Декодинг `json.loads(msg.data)` сделан **до** handler-а, чтобы JSON-ошибка попала в общий `try/except` и классифицировалась как TERM.
- `_resolve_message_response_type` смотрит на тип исключения и возвращает `MessageResponseType` (см. ниже).
- `_handle_message_response` шлёт `ack`/`nak`/`term` в NATS; если сетевая ошибка — сообщение не подтверждено, переподключаемся и продолжаем (брокер при таймауте перезаложит).
- `loop_sleep_duration` между итерациями — задаётся в `NatsSettings`.

### Классификация ошибок (matrix)

```python
def _resolve_message_response_type(self, error: BaseException | None) -> MessageResponseType:
    if error is None:
        return MessageResponseType.ACK
    if isinstance(error, (_InvalidPayloadError, JSONDecodeError)):
        return MessageResponseType.TERM
    if isinstance(error, (AppNotFoundError, AppInternalError)):
        return MessageResponseType.NAK
    if isinstance(error, (DomainError, AppError)):
        return MessageResponseType.TERM
    return MessageResponseType.NAK
```

Полная матрица с объяснением «почему NAK, а не TERM» — [error_classification.md](references/error_classification.md). Главное правило: **TERM** — сообщение «дальше пытаться бесполезно» (битый payload, нарушение доменного инварианта); **NAK** — «временно не получилось, пусть JetStream доставит снова» (БД недоступна, related-агрегат ещё не приехал).

### Payload validation

Каждый handler начинается с распаковки payload через **отдельную pydantic-модель** в этом же файле и helper-метод `_extract_*_payload`. ValidationError превращается в `_InvalidPayloadError` (приватный наследник `AppInternalError` в `consumer.py`) — этот тип ловится `_resolve_message_response_type` и даёт TERM:

```python
class CompanyCreationPayload(BaseModel):
    company_id: UUID
    state: str
    version: int


def _extract_company_creation_payload(self, payload: dict) -> tuple[UUID, str, int]:
    try:
        data = CompanyCreationPayload.model_validate(payload)
    except ValidationError as err:
        raise _InvalidPayloadError(
            msg="некорректный payload компании для потребления",
            action="извлечение данных из сообщения о компании",
            data={"payload": payload, "errors": err.errors()},
            wrap_error=err,
        )
    return data.company_id, data.state, data.version
```

- Модель — payload-only (`event_id`/`occurred_at` — на уровне JetStream-headers, не в теле). Если проект кладёт metadata в тело — поля добавляются в модель.
- `_InvalidPayloadError` — приватный класс на уровне `consumer.py`. Это не application-уровень; payload-ошибка специфична для транспортного слоя.
- `wrap_error=err` сохраняет оригинальную `ValidationError` в `__cause__` для логов.

### Handler делегирует в use case

```python
async def _handle_company_creation(self, payload: dict) -> None:
    company_id, state, version = self._extract_company_creation_payload(payload)
    command = CompanyCreationCommand(company_id=company_id, state=state, version=version)
    async with self._db_manager.connection() as conn:
        await CompanyCreationUseCase(PostgresUnitOfWork(conn)).execute(command)
```

Шаги всегда в одном порядке: extract → Command → `async with db_manager.connection()` → UseCase(uow).execute(command). Никакой логики в handler-е помимо этого.

## NatsPublisherWorker

Публикатор outbox-событий: цикл `heartbeat → handler → sleep`. На каждый агрегат — свой `asyncio.Task`.

### Streams creation в `_events_after_connected`

```python
async def _events_after_connected(self) -> None:
    await self._create_streams()

async def _create_streams(self) -> None:
    streams_subjects: dict[str, list[str]] = {}
    for stream_settings in (
        self._publishers_settings.<agg1>,
        self._publishers_settings.<agg2>,
    ):
        streams_subjects.setdefault(stream_settings.stream_name, []).extend(stream_settings.subjects)
    for stream_name, subjects in streams_subjects.items():
        try:
            info = await self._js.stream_info(stream_name)
        except NotFoundError:
            logger.info("creating nats stream: %s", stream_name)
            await self._js.add_stream(config=StreamConfig(name=stream_name, subjects=subjects))
            continue

        existing = info.config.subjects or []
        missing = [subject for subject in subjects if subject not in existing]
        if not missing:
            continue
        logger.info("updating nats stream subjects: %s", stream_name)
        info.config.subjects = existing + missing
        await self._js.update_stream(config=info.config)
```

- Один stream объединяет subjects всех агрегатов с одинаковым `stream_name` (сервисный stream — обычно один на сервис).
- **Создания stream недостаточно — нужно сверять subjects уже существующего stream.** Если stream уже есть, `stream_info` не бросает `NotFoundError`, и при «голом» `stream_info → add_stream` новые subjects (добавленные в конфиг позже — например, при появлении нового агрегата) никогда не доедут до живущего на сервере stream. Тогда `js.publish(subject)` на незарегистрированный subject упрётся в `NoRespondersError`, который JetStream-клиент превращает в `NoStreamResponseError` → ошибка `nats: no response from stream`. Симптом: публикатор «нового» агрегата падает, а старые работают.
- Поэтому при существующем stream сверяй желаемые subjects с `info.config.subjects` и недостающие добавляй через `js.update_stream`. **Мутируй `info.config`** (полученный от сервера) и дополняй только `subjects`, а не пересоздавай `StreamConfig` с нуля — иначе сбросишь retention/storage/replicas и прочие параметры stream к дефолтам клиента. Список subjects только расширяем (удаление subject с непрочитанными сообщениями сервер и так отклонит).
- `stream_info → add_stream`/`update_stream` — идемпотентная связка; повторный вызов на реконнект безопасен (на втором проходе `missing` пуст → апдейта нет).
- Конфигурацию stream (retention, max_msgs, и т.п.) добавляй в `StreamConfig` явно при `add_stream`, не оставляй дефолты, если они влияют на семантику.

### Publish-loop

```python
async def _publish(self, handler: Handler, name: str) -> None:
    while not self._shutdown_event.is_set():
        self._update_heartbeat()
        processing_error: BaseException | None = None
        try:
            await handler()
        except NATS_CONNECTION_ERRORS:
            await self._connect_nats()
            continue
        except asyncio.CancelledError:
            return
        except BaseException as err:
            processing_error = err
        self._log_processing_error(name, processing_error)
        await asyncio.sleep(self._nats_settings.loop_sleep_duration)
```

Никаких subscriptions/fetch — handler сам решает что публиковать (через выборку outbox-таблицы из БД).

### Handler делегирует в use case с publisher-адаптером

```python
async def _handle_<agg>_publish(self) -> None:
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
```

- `EventNatsPublisher` — адаптер из `infrastructure/message_broker/nats/publisher.py`, отвечает за низкоуровневый publish с retry/ack JetStream-а. Воркер только передаёт его в use case.
- Use case вычитывает outbox, маппит в payload, зовёт `publisher.publish(...)`, помечает outbox-записи опубликованными в той же транзакции.
- Идемпотентность публикации — на уровне outbox (запись помечена опубликованной → use case её больше не отдаёт).

## Idempotency

Идемпотентность — **в use case**, не в воркере.

- **Consumer**: повторная доставка прогоняется через use case. Use case проверяет инварианты (`Already exists` → `AppError`, который классифицируется как TERM/NAK по матрице) или использует `INSERT ... ON CONFLICT DO NOTHING` / version-чек агрегата. Воркер про дубли не знает и dedup-стораджа не держит.
- **Publisher**: outbox-таблица. Use case в одной транзакции читает «неопубликованные», публикует и помечает «опубликованные». Если воркер крашнется между publish и mark — на следующей итерации публикатор пошлёт повторно; consumer-side это абсорбирует своей идемпотентностью.

Скил **не** прописывает «dedup до handler-а через event_id»: это не паттерн репозитория. Если конкретный use case требует явный dedup (например, side-effect нельзя повторить) — это решается внутри use case через выделенную dedup-таблицу, не на уровне воркера.

## Runtime

uvloop поднимается в entrypoint через `asyncio.Runner(loop_factory=new_event_loop)`. Глобальная `set_event_loop_policy` не нужна и не используется.

```python
from asyncio import Runner

from uvloop import new_event_loop

with Runner(loop_factory=new_event_loop) as runner:
    runner.run(worker.run())
```

- `Runner` создаёт цикл через переданную фабрику, прогоняет корутину, закрывает loop при выходе из контекста.
- Воркер сам должен быть async (`async def run`), как `BackgroundBaseWorker.run`.

## Entrypoint

NATS-режимы — ветки в `src/main.py`:

```python
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

- `getLogger("nats").setLevel(CRITICAL)` — отключаем шум NATS-клиента; своё логирование делает воркер.
- Миграции применяются **до** старта воркера, не в `setup()` — иначе несколько инстансов будут гоняться за блокировкой.
- Settings (`MessageBrokerConsumerSettings`, `MessageBrokerPublisherSettings`) — top-level worker-настройки из [python-pydantic-settings-config-writing].

## Anti-patterns

- **`asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())`** — глобальная policy. Используем `Runner(loop_factory=new_event_loop)`.
- **Ack до завершения handler-а** — обработка ещё может крашнуться, но JetStream уже считает сообщение принятым.
- **`msg.term()` для временных ошибок** (БД недоступна, related-агрегат ещё не приехал) — теряем сообщение навсегда. TERM только когда ретрай заведомо не поможет.
- **`msg.nak()` для битого payload** — JetStream будет упорно его доставлять до `max_deliver`. Битый payload → TERM.
- **`stream_info → add_stream` без сверки subjects существующего stream** — новые subjects, добавленные в конфиг позже, не попадут в уже созданный stream, и `publish` упадёт с `nats: no response from stream`. При существующем stream сверяй и дополняй subjects через `update_stream`.
- **Пересоздание `StreamConfig` при `update_stream`** вместо мутации `info.config` — сбрасывает retention/storage/replicas и прочие параметры к дефолтам клиента. Бери конфиг от `stream_info` и меняй только `subjects`.
- **Бизнес-логика в handler-е воркера** (мутации, обращения в БД помимо use case) — handler только распаковывает payload и зовёт use case.
- **Внешний dedup-сторадж в воркере** до прохода в use case — идемпотентность живёт в use case (через invariants/outbox), не в transport layer.
- **Reconnect через `max_reconnect_attempts > 0` у NATS-клиента** — конфликтует со своим циклом реконнекта в `_connect_nats`. Используй `max_reconnect_attempts=0` и крути реконнект руками с учётом `_shutdown_event`.
- **Голый `asyncio.sleep(big)` в фоновом цикле** — на shutdown воркер будет ждать его до конца. Длинные ожидания — через `wait_for(_shutdown_event.wait(), timeout=...)`.
- **DLQ через отдельный `publish_to_dlq(...)` subject** — в JetStream DLQ-поведение даёт `max_deliver` (после которого сообщение либо ack-нется автоматически, либо попадёт в dead-letter subject стрима). Не плодим параллельную реализацию.
- **Не вызывать `_update_heartbeat()` в горячем цикле** — k8s-liveness-probe порежет процесс.

## Definition of Done

- `BackgroundBaseWorker` есть в `src/presentation/background/base.py` и обрабатывает SIGTERM/SIGINT, ведёт `_shutdown_event`, отменяет подзадачи в `_cancel_tasks`, имеет `_update_heartbeat`. `setup`/`complete`/`_create_tasks` — абстрактные.
- `NatsBaseWorker` в `src/presentation/background/nats/base.py` управляет подключением к NATS+DB, держит цикл реконнекта в `_connect_nats` под `_shutdown_event`, экспортирует `NATS_CONNECTION_ERRORS` и `_events_after_connected` как extension point.
- `NatsConsumerWorker`: `_create_tasks` регистрирует подписки списком `(subject, handler, name)`; `_consume` делает `pull_subscribe → fetch → decode → handler → resolve_response → ack/nak/term`, реконнектится на `NATS_CONNECTION_ERRORS`, пересоздаёт subscription. Payload-схемы — pydantic-модели; `ValidationError` оборачивается в `_InvalidPayloadError`.
- Классификация исключений в `_resolve_message_response_type` соответствует матрице из [error_classification.md] (битый payload → TERM, временное → NAK, доменная ошибка → TERM).
- `NatsPublisherWorker`: `_events_after_connected` идемпотентно создаёт streams через `stream_info → add_stream` **и сверяет subjects существующего stream, дополняя недостающие через `update_stream`** (мутируя `info.config`); `_publish` крутит `heartbeat → handler → sleep`; handler делегирует в `<Aggregate>PublisherUseCase(uow, publisher).execute()`.
- Idempotency — в use case (invariants/outbox), не в воркере.
- Entrypoint поднимает loop через `asyncio.Runner(loop_factory=uvloop.new_event_loop)`; режимы `MODE=nats_consumer`/`MODE=nats_publisher` следуют шаблону: settings → миграции → `worker.run()`.
- `getLogger("nats").setLevel(CRITICAL)` стоит до старта воркера, чтобы не дублировать сетевой шум NATS-клиента.

## References

- **[nats_worker_template.md](references/nats_worker_template.md)** — полный эталон: `presentation/background/base.py`, `nats/base.py`, `nats/consumer.py`, `nats/publisher.py`, фрагмент `main.py`. Стартовая точка для нового сервиса.
- **[error_classification.md](references/error_classification.md)** — матрица «исключение → ACK/NAK/TERM», правила «когда TERM, а когда NAK», что делает JetStream после `term()`/`nak()`/превышения `max_deliver`, как использовать `_InvalidPayloadError`.
- Структура `NatsSettings` / `NatsConsumerStreamSettings` / `NatsPublisherStreamSettings` — [python-pydantic-settings-config-writing] (`references/tech_examples.md` → NATS / JetStream).
