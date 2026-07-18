---
name: python-subscription-worker
description: Используй при проектировании или реализации фонового воркера-«подписчика» (periodic-loop без брокера сообщений) в Python-сервисах с гексагональной архитектурой — типично для синхронизации внутренних проекций между bounded contexts внутри одного сервиса. Триггеры — сборка `SubscriptionWorker` (наследник `BackgroundBaseWorker`), периодический цикл heartbeat → handler → log → sleep, делегирование в use case через `UnitOfWork`. Не применять для HTTP API ([python-fastapi-api-worker]) и для NATS-воркеров ([python-nats-worker]).
---

# Python Subscription Worker

Скил про periodic-loop background worker без брокера сообщений. В этом проекте паттерн называется `SubscriptionWorker` и используется для **синхронизации внутренних проекций между bounded contexts**: один use case вычитывает изменения из одной проекции и применяет к другой; воркер просто крутит use case по расписанию, не имеет внешнего триггера и не разговаривает с NATS.

Тот же шаблон подходит для любого «бесконечного pull-цикла без брокера»: scheduled-задач, periodic-rebuild-проекций, GC/cleanup-таск, и т.п. — поэтому скил описан общим, конкретный пример `SubscriptionWorker` идёт через `<Subscription>` плейсхолдеры в шаблоне.

Общий каркас `BackgroundBaseWorker` (сигналы, lifecycle, heartbeat, task cancellation) описан в [python-nats-worker] — здесь не дублируется. Этот скил описывает специфичное: настройки, цикл подписки, делегирование в use case.

## Quick Start

1. Заведи `BackgroundBaseWorker` в `src/presentation/background/base.py`, если его ещё нет (см. [python-nats-worker] → `BackgroundBaseWorker`).
2. Создай класс `SubscriptionWorker` (`src/presentation/background/subscriptions.py`) — наследник `BackgroundBaseWorker`, принимающий `SubscriptionWorkerSettings` в `__init__`.
3. Заведи `SubscriptionSettings` (`infrastructure/config/subscriptions.py`) — `healthcheck_file` + `loop_sleep_duration`. Top-level `SubscriptionWorkerSettings` композирует `subscription: SubscriptionSettings` и `db: PostgresSettings`.
4. В `setup()` подними `PostgresConnectionManager`; в `complete()` закрой.
5. В `_create_tasks()` зарегистрируй список `(handler, name)` через `asyncio.create_task(self._subscribe(...))`.
6. В `_subscribe(handler, name)` крути цикл: `heartbeat → handler → log_error → sleep loop_sleep_duration`. Никакого pull/fetch/ack — handler сам решает, есть работа или нет.
7. Каждый handler делает `async with db_manager.connection() as conn: await <Aggregate><Operation>UseCase(PostgresUnitOfWork(conn)).execute()` — use case отвечает за «есть ли что синхронизировать».
8. Entrypoint (`src/main.py`) запускает воркер через `asyncio.Runner(loop_factory=uvloop.new_event_loop)` под `MODE=subscriptions_worker`.

Полный эталон — [subscription_worker_template.md](references/subscription_worker_template.md).

## When to Apply

### Триггеры активации

- Создание или правка `src/presentation/background/subscriptions.py` (или любого другого periodic-loop воркера на `BackgroundBaseWorker`).
- Добавление новой пары `(handler, name)` в список подписок.
- Изменение `loop_sleep_duration` или политики ошибок в `_subscribe`.
- Создание/правка `SubscriptionSettings` / `SubscriptionWorkerSettings` в `infrastructure/config/`.
- Изменение entrypoint-логики для режима `subscriptions_worker` (или эквивалентного периодического).

### Анти-триггеры

- HTTP API-воркер — [python-fastapi-api-worker].
- NATS consumer/publisher — [python-nats-worker].
- Реализация use case-ов, репозиториев, агрегатов — это `domain/`/`application/`/`infrastructure/`-уровни.
- Структура `infrastructure/config/subscriptions.py` сама по себе — это [python-pydantic-settings-config-writing].

## Связь с `python-nats-worker`

Оба скила опираются на один и тот же `BackgroundBaseWorker`. Контракт каркаса описан в `python-nats-worker` → `BackgroundBaseWorker` (signal handling, `_shutdown_event`, `_cancel_tasks`, `_update_heartbeat`, abstract `setup`/`complete`/`_create_tasks`).

`SubscriptionWorker` отличается от `NatsBaseWorker` тем, что:

- Не подключается к NATS — только к Postgres.
- Не делает pull/fetch — handler сам генерирует свою работу (читает «есть ли что синхронизировать» из таблицы внутри use case).
- Не имеет ack/nak/term — нет брокера, нет «доставки сообщения». Ошибка логируется и цикл переходит к следующей итерации.
- Не имеет reconnect-логики — Postgres-пул сам переподключается; если соединение не поднимается, упадёт `setup()` и воркер пойдёт на рестарт через k8s.

В остальном — тот же каркас.

## Package Structure

```
src/presentation/background/
├── base.py                       ← BackgroundBaseWorker (см. python-nats-worker)
└── subscriptions.py              ← SubscriptionWorker — periodic-loop воркер

src/infrastructure/config/
└── subscriptions.py              ← SubscriptionSettings (healthcheck_file, loop_sleep_duration)
```

`SubscriptionWorkerSettings` — top-level worker-конфиг в `infrastructure/config/workers.py` (или `__init__.py`); композирует `subscription: SubscriptionSettings` + `db: PostgresSettings`. Подробнее — [python-pydantic-settings-config-writing].

## SubscriptionWorker

Наследник `BackgroundBaseWorker`. Хранит:

- `SubscriptionSettings` — `healthcheck_file` (передаётся в `super().__init__`), `loop_sleep_duration`.
- `PostgresConnectionManager` — единственный shared-ресурс; поднимается в `setup()`, закрывается в `complete()`.

Что переопределяет:

- `setup()` — `await self._db_manager.init()`.
- `complete()` — `await self._db_manager.close()`.
- `_create_tasks()` — регистрирует пары `(handler, name)` через `asyncio.create_task(self._subscribe(handler, name))`.

Что добавляет:

- `_subscribe(handler, name)` — цикл подписки (см. ниже).
- Несколько `_handle_*` методов — handler-ы, каждый делегирует в свой use case.
- `_log_processing_error(name, error)` — логирование по типу ошибки.

## Settings

```python
# infrastructure/config/subscriptions.py
from pydantic import BaseModel


class SubscriptionSettings(BaseModel):
    healthcheck_file: str = "/app/run/subscription_worker_healthbeat"
    loop_sleep_duration: int = 10
```

- `healthcheck_file` — путь, который воркер `touch`-ит в каждой итерации каждой подписки. Дефолт **не в `/tmp`** (см. anti-patterns в [python-pydantic-settings-config-writing]).
- `loop_sleep_duration` — пауза между итерациями (секунды). Конкретное значение — продуктовое решение; зависит от того, насколько свежими должны быть проекции (обычно 5–30 секунд).

Top-level настройки:

```python
# infrastructure/config/workers.py
class SubscriptionWorkerSettings(AppBaseSettings):
    subscription: SubscriptionSettings = Field(default_factory=SubscriptionSettings)
    db: PostgresSettings = Field(default_factory=PostgresSettings)
```

NATS-блока здесь нет — воркер с брокером не разговаривает.

## Subscribe-loop

```python
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
```

Ключевые свойства:

- **Heartbeat перед работой**, не после — если handler виснет, файл будет старым; k8s-liveness порежет процесс. Если бы мы трогали файл в конце — зависший handler выглядел бы здоровым до начала следующей итерации.
- **Все исключения ловятся в `processing_error`** и потом логируются. Никаких ack/nak — handler либо завершился, либо нет; в обоих случаях цикл идёт дальше.
- **`CancelledError` пробрасывается через `return`** — это shutdown-путь; не логируем как ошибку.
- **`asyncio.sleep(loop_sleep_duration)`** — sleep после каждой итерации, включая успешную. Если хочется адаптивности (нет работы → дольше спать, есть работа → сразу повтор) — это решается **внутри handler-а/use case-а**, не в `_subscribe`.

Каждая пара `(handler, name)` живёт в своей `asyncio.Task`, поэтому handler-ы независимы: ошибка в одном не аффектит другие.

## Handler → use case

Handler — тонкий клей: открыть БД-соединение, инстанцировать use case, выполнить.

```python
async def _handle_<aggregate>_<operation>(self) -> None:
    async with self._db_manager.connection() as conn:
        await <Aggregate><Operation>UseCase(PostgresUnitOfWork(conn)).execute()
```

- Имя use case-а — типично `<Aggregate>CreationUseCase` / `<Aggregate>UpdateUseCase` / `<Aggregate>SyncUseCase`. Импортируется из `application.commands.private.<aggregate>` (или другого слоя — зависит от проекта).
- `execute()` без аргументов: у subscription-use case нет внешнего ввода — он сам читает «что синхронизировать» из таблицы.
- Никаких параметров в handler-е, никаких payload-ов: всё внутри use case.
- Имена handler-ов в `_create_tasks` — `<aggregate>_<operation>` (`tenant_creation`, `member_update`), чтобы лог-префикс `[<name>]` был осмысленным.

## Logging by error type

`_log_processing_error` логирует по типу ошибки, аналогично NATS-воркерам:

| Тип | Уровень |
|---|---|
| `None` (нет ошибки) | — (тихо) |
| `DomainError` | `warning` |
| `AppInternalError` | `error` |
| `AppError` (прочее) | `warning` |
| Неклассифицированное | `error` |

`_InvalidPayloadError`/`JSONDecodeError` в этом воркере **не возникают** — нет payload. Поэтому их веток в log-методе нет.

Префикс `[<name>]` (имя из `_create_tasks`) проставляется во все сообщения — по логу видно, какая подписка генерит шум.

## Runtime

uvloop — через `asyncio.Runner(loop_factory=new_event_loop)` в entrypoint, без глобальной policy. Тот же паттерн, что и в [python-nats-worker]:

```python
from asyncio import Runner

from uvloop import new_event_loop

with Runner(loop_factory=new_event_loop) as runner:
    runner.run(worker.run())
```

## Entrypoint

В `src/main.py` — отдельная ветка `match`-а:

```python
case "subscriptions_worker":
    subscriptions_settings = SubscriptionWorkerSettings()
    apply_db_migrations(subscriptions_settings.db)
    worker = SubscriptionWorker(subscriptions_settings)
    with Runner(loop_factory=new_event_loop) as runner:
        runner.run(worker.run())
```

Миграции применяются до старта воркера, не в `setup()` (избежать гонок между инстансами за yoyo-блокировку).

## Anti-patterns

- **`asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())`** — глобальная policy. Используем `Runner(loop_factory=new_event_loop)`.
- **Heartbeat в конце итерации** — зависший handler выглядит здоровым. Heartbeat **в начале** каждой итерации.
- **Логика «когда работать» в `_subscribe`** (адаптивный sleep, проверка времени суток, чтение БД для решения «есть работа или нет») — это знание use case, не воркера. `_subscribe` тупо крутит цикл; знает только `loop_sleep_duration`.
- **Бизнес-логика в handler-е** (мутации, обращения в репозитории напрямую) — handler только открывает соединение и зовёт use case.
- **Аргументы в use case `.execute()`** для periodic-цикла — у subscription-use case нет внешнего ввода. Если нужно «обработать одну запись» — это другая семантика, и use case должен зваться по-другому (или это вообще NATS-handler, а не subscription).
- **Голый `asyncio.sleep(big)` для скейла периода** — длинные периоды не дадут воркеру нормально шатдауниться. Если нужен период >30 секунд, делай это через `wait_for(_shutdown_event.wait(), timeout=...)` в `_subscribe` вместо чистого `sleep`.
- **`max_reconnect_attempts > 0` у psycopg/Postgres-пула с собственным reconnect-циклом** — здесь его и нет, но если будешь подключаться к другим бэкендам (Redis, например), помни про конфликт.
- **Несколько `SubscriptionWorker` в одном процессе** — один воркер = один runtime-процесс.
- **Healthcheck-файл в `/tmp`** — anti-pattern из [python-pydantic-settings-config-writing]. Брать писабельную папку под рабочей директорией процесса.

## Definition of Done

- `SubscriptionWorker` в `src/presentation/background/subscriptions.py` наследует `BackgroundBaseWorker`, принимает `SubscriptionWorkerSettings`.
- `setup` поднимает `PostgresConnectionManager`, `complete` его закрывает.
- `_create_tasks` регистрирует пары `(handler, name)` через `asyncio.create_task(self._subscribe(handler, name))`.
- `_subscribe` крутит цикл `heartbeat → handler → log → sleep`, где heartbeat — **в начале** итерации.
- `CancelledError` обрабатывается через `return`, остальные `BaseException` логируются по типу через `_log_processing_error`.
- Каждый handler делегирует в `<Aggregate><Operation>UseCase(PostgresUnitOfWork(conn)).execute()` без аргументов.
- `SubscriptionSettings` (`healthcheck_file`, `loop_sleep_duration`) задан в `infrastructure/config/subscriptions.py`; путь heartbeat-файла не в `/tmp`.
- `SubscriptionWorkerSettings` собран из `subscription: SubscriptionSettings` + `db: PostgresSettings`, без NATS-блока.
- Entrypoint поднимает loop через `asyncio.Runner(loop_factory=uvloop.new_event_loop)`; режим `MODE=subscriptions_worker` следует шаблону `settings → migrations → worker.run()`.

## References

- **[subscription_worker_template.md](references/subscription_worker_template.md)** — полный эталон `SubscriptionWorker`, `SubscriptionSettings`, `SubscriptionWorkerSettings`, фрагмент `main.py`.
- `BackgroundBaseWorker` (общий каркас) — [python-nats-worker] → `BackgroundBaseWorker` и `references/nats_worker_template.md` → `presentation/background/base.py`.
- `SubscriptionSettings` структура (стандартные поля periodic-loop воркера) — [python-pydantic-settings-config-writing] → `references/tech_examples.md` → Subscription Worker.
