# Ports

## Допущения раздела

**Универсально для `port/`:**
- Все классы — абстрактные (`ABC` или потомки), методы — `@abstractmethod` с телом `...`.
- Реализация портов **всегда** в `infrastructure/`, не здесь.

**Импорты в `port/`:**

| Можно | Нельзя |
|---|---|
| `domain.*` (VO, entity, projection, абстрактный domain repo для наследования) | `infrastructure.*` |
| `application.dto.paginator`, `application.error` | `presentation.*` |
| stdlib (`abc`, `dataclasses`, `datetime`, `enum`, `typing`) | внешние библиотеки |

**Специфично для проекта:**
- Конкретный набор типов репозиториев на агрегат (`read` / `version` / `outbox` / `subscription`).
- Структура UoW (одна на проект; разбита на группы по агрегатам).
- Состав `<Aggregate>Event`.
- Какие агрегаты публикуются наружу, какие — подписаны на чужие/внутренние события.

При отсутствии существующей конвенции — согласование с пользователем.

## Структура `port/`

```
port/
├── __init__.py                      ← пустой
├── unit_of_work.py                  ← UnitOfWork ABC
├── event_publisher.py               ← EventPublisher ABC + Union
├── dto/                             ← внутренние DTO для нескольких портов
│   └── <aggregate>.py
└── repository/
    ├── __init__.py                  ← только re-export + __all__
    └── <aggregate>.py               ← интерфейсы + <Aggregate>Repositories
```

## `unit_of_work.py`

```python
from abc import ABC, abstractmethod
from types import TracebackType
from typing import Self

from application.port.repository import (
    TenantRepositories,
    UserRepositories,
    # ... другие группы
)


class UnitOfWork(ABC):
    @abstractmethod
    async def __aenter__(self) -> Self: ...

    @abstractmethod
    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: TracebackType | None,
    ) -> None: ...

    @property
    @abstractmethod
    def tenant_repositories(self) -> TenantRepositories: ...

    @property
    @abstractmethod
    def user_repositories(self) -> UserRepositories: ...

    # ... по property на каждый агрегат
```

**Правила:**
- `__aenter__` возвращает `Self` — допускает реализацию-«self» или транзакционно-связанный объект.
- `__aexit__` обязан откатывать при исключении (контракт инфраструктурной реализации).
- Группы репозиториев — через `@property @abstractmethod`, не через прямые поля. Лениво, абстрактно, реализация решает как поднимать.
- Один `UnitOfWork` на проект, **не** один на агрегат. Все группы — атрибуты одного UoW.

## `event_publisher.py`

```python
from abc import ABC, abstractmethod
from typing import Sequence, TypeAlias

from application.port.repository import (
    TenantVersionDTO,
    TransactionVersionDTO,
    # ... только те, что публикуются наружу
)


PublishEventDTO: TypeAlias = (
    TenantVersionDTO
    | TransactionVersionDTO
    # | другие port-level VersionDTO агрегатов, публикуемых наружу
)


class EventPublisher(ABC):
    @abstractmethod
    async def publish(self, event: PublishEventDTO) -> None: ...

    @abstractmethod
    async def batch_publish(self, events: Sequence[PublishEventDTO]) -> None: ...
```

**Правила:**
- В `Union` попадают **только те** `<Aggregate>VersionDTO`, чьи изменения проект публикует наружу. Агрегаты-проекции / read-only локальные сущности в `Union` не входят.
- `publish` — единичная публикация, `batch_publish` — массовая. Добавляй только реально используемые операции.
- Используется только publisher use case-ами.
- Не добавляй агрегатные методы `publish_users`, `publish_tenants`: application передаёт типизированный union, а infrastructure выбирает serializer и subject исчерпывающим dispatch по типу.
- Неизвестный runtime-тип не игнорируется и приводит к `AppInternalError`.

## `repository/<aggregate>.py` — каркас файла

Каждый файл содержит **подмножество** следующих сущностей (что именно — определяется ролью агрегата):

| Сущность | Назначение | Когда присутствует |
|---|---|---|
| `<Aggregate>ReadRepository` | чтение/запись текущего состояния, наследник domain-интерфейса | всегда |
| `<Aggregate>Event` (`StrEnum`) | типы событий жизненного цикла + `from_str` | если есть version-репо |
| `<Aggregate>VersionDTO` (dataclass) | port-level snapshot: domain entity + event + editor_id + created_at | если есть version-репо |
| `<Aggregate>VersionRepository` | чтение/запись версионных снапшотов | агрегат имеет аудит-лог |
| `<Aggregate>OutboxRepository` | транзакционный outbox для исходящих событий | агрегат публикуется наружу |
| `<Aggregate>SubscriptionRepository` | inbox для входящих событий (subscription tracking) | агрегат подписан на события другого агрегата (внутреннего того же BC или внешнего) |

Файл **один** на агрегат, всё перечисленное — внутри. Не разбиваем на `events.py` / `version.py` и т.п.

## `<Aggregate>ReadRepository`

```python
from abc import abstractmethod

from application.dto.paginator import LimitOffsetPaginator
from domain.tenant import (
    Tenant,
    TenantID,
    TenantStatus,
    TenantReadRepository as DomainTenantReadRepository,
)
from domain.value_objects import State


class TenantReadRepository(DomainTenantReadRepository):
    @abstractmethod
    async def by_id(self, tenant_id: TenantID) -> Tenant | None: ...

    @abstractmethod
    async def filters(
        self,
        paginator: LimitOffsetPaginator,
        tenant_ids: list[TenantID] | None = None,
        statuses: list[TenantStatus] | None = None,
        states: list[State] | None = None,
    ) -> tuple[list[Tenant], int]: ...

    @abstractmethod
    async def save(self, tenant: Tenant) -> None: ...

    # опционально, при необходимости:
    @abstractmethod
    async def next_id(self) -> TenantID: ...

    @abstractmethod
    async def batch_save(self, tenants: list[Tenant]) -> None: ...
```

**Принцип разделения с domain:**
- Domain-интерфейс (`Domain<Aggregate>ReadRepository`) описывает операции, нужные **доменным сервисам** (uniqueness checks, поиск по ключам и т.п.).
- Application-интерфейс расширяет его операциями, нужными **use case-ам** (`by_id`, `filters` с пагинацией, `save`, опционально `next_id`/`batch_save`).
- Реализация в `infrastructure/` удовлетворяет application-интерфейсу (и тем самым transitively — domain).

`next_id` появляется только у агрегатов, которые сами генерируют id (через репо как источник свежего id-в-транзакции).

## `<Aggregate>Event` + `<Aggregate>VersionDTO` + `<Aggregate>VersionRepository`

Присутствуют **только** у агрегатов, имеющих `<Aggregate>VersionRepository`. Для агрегатов без version-репо (например, read-only локальные проекции) Event и VersionDTO отсутствуют.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from enum import StrEnum
from typing import Self

from application.error import AppInternalError
from domain.tenant import Tenant, TenantID, TenantStatus
from domain.user import UserID
from domain.value_objects import State, Version


class TenantEvent(StrEnum):
    CREATED = "created"
    UPDATED = "updated"
    RESTORED = "restored"
    DELETED = "deleted"

    @classmethod
    def from_str(cls, value: str) -> Self:
        lower_value = value.lower()
        if lower_value in cls._value2member_map_:
            return cls(lower_value)
        raise AppInternalError(
            msg="не удалось сопоставить событие по предоставленной строке",
            action="получение события арендатора",
            data={"event": value},
        )


@dataclass
class TenantVersionDTO:
    tenant: Tenant
    event: TenantEvent
    editor_id: UserID | None
    created_at: datetime


class TenantVersionRepository(ABC):
    @abstractmethod
    async def by_id_version(
        self, tenant_id: TenantID, version: Version
    ) -> TenantVersionDTO | None: ...

    @abstractmethod
    async def filters(
        self,
        paginator: LimitOffsetPaginator,
        tenant_id: TenantID,
        statuses: list[TenantStatus] | None = None,
        states: list[State] | None = None,
        from_version: Version | None = None,
        to_version: Version | None = None,
    ) -> tuple[list[TenantVersionDTO], int]: ...

    @abstractmethod
    async def save(
        self, tenant: Tenant, event: TenantEvent, editor_id: UserID | None
    ) -> None: ...

    # опционально:
    @abstractmethod
    async def batch_save(
        self,
        items: list[tuple[Tenant, TenantEvent, UserID | None]],
    ) -> None: ...
```

**Правила:**
- `<Aggregate>VersionDTO` — **port-level**, держит ссылку на domain entity. **Не** application DTO (тот плоский, см. `dtos.md`).
- `<Aggregate>VersionDTO` оформлен как `@dataclass` без `slots`/`frozen` — это port-level транспорт между репо и use case, не application output.
- `<Aggregate>Event.from_str` обязателен — нужен для конверсии строкового значения из БД/брокера обратно в enum, бросает `AppInternalError` при отсутствии.
- `editor_id` — `None` для системных событий (use case без initiator).

## `<Aggregate>OutboxRepository`

Присутствует только у агрегатов, публикуемых наружу.

```python
class TenantOutboxRepository(ABC):
    @abstractmethod
    async def save(self, tenant: Tenant) -> None: ...

    @abstractmethod
    async def batch_save(self, tenants: list[Tenant]) -> None: ...

    @abstractmethod
    async def not_published_versions(self) -> list[TenantVersionDTO]: ...
```

- `save` / `batch_save` — пишет состояние в outbox-таблицу транзакционно вместе с основной записью use case-а.
- `not_published_versions` — снимает batch для публикации, используется publisher use case-ом.

## `<Aggregate>SubscriptionRepository`

Используется, когда агрегат текущего сервиса **подписан на события другого агрегата** — независимо от того, где находится источник:
- **Внешний источник**: агрегат из другого bounded context, события приходят через брокер.
- **Внутренний источник**: агрегат того же BC, на который текущий агрегат реагирует (например, дочерняя сущность подписана на события родителя).

```python
class UserSubscriptionRepository(ABC):
    @abstractmethod
    async def subscribe(self, subscriber: User, source: SourceAggregate) -> None: ...

    @abstractmethod
    async def batch_subscribe(
        self, items: list[tuple[User, SourceAggregate]]
    ) -> None: ...

    @abstractmethod
    async def mark_as_processed(
        self, subscriber: User, source: SourceAggregate
    ) -> None: ...

    @abstractmethod
    async def batch_mark_as_processed(
        self, items: list[tuple[User, SourceAggregate]]
    ) -> None: ...

    @abstractmethod
    async def new_source_versions(
        self,
    ) -> list[tuple[User, SourceAggregate]]: ...
```

Хранит связь «subscriber ↔ source» и состояние обработки входящих версий. Конкретные имена методов и сигнатуры зависят от того, какой агрегат-источник подписки — общий принцип: subscribe / mark_as_processed / pull pending.

## Группы `<Aggregate>Repositories`

```python
from dataclasses import dataclass

from application.port.repository.tenant import (
    TenantEvent,
    TenantOutboxRepository,
    TenantReadRepository,
    TenantVersionDTO,
    TenantVersionRepository,
)
# ... другие импорты

__all__ = [
    "TenantEvent",
    "TenantOutboxRepository",
    "TenantReadRepository",
    "TenantRepositories",
    "TenantVersionDTO",
    "TenantVersionRepository",
    # ... в алфавитном порядке
]


@dataclass(slots=True)
class TenantRepositories:
    read: TenantReadRepository
    version: TenantVersionRepository
    outbox: TenantOutboxRepository
    # subscription, если применимо
```

**Правила:**
- Объявляй `<Aggregate>Repositories` в `port/repository/<aggregate>.py` рядом с интерфейсами своего агрегата.
- В `port/repository/__init__.py` оставляй только re-export и алфавитный `__all__`.
- `<Aggregate>Repositories` — `@dataclass(slots=True)`, **без** `frozen` (это инжектируемый контейнер зависимостей).
- Доступ из use case-а: `uow.<aggregate>_repositories.read.<method>(...)` — двухуровневый.
- Альтернатива «плоский UoW с десятками property» отвергается — группировка по агрегату делит UoW на читаемые разделы.

## Внутренние port DTO

Выноси DTO в `port/dto/<aggregate>.py`, когда один контракт используют несколько repository/publisher-портов и адаптеров. Port DTO может содержать domain entity и VO, поскольку не пересекает границу application → presentation. Называй его по смыслу (`UserEventDTO`, `TenantVersionDTO`), а не `PortDTO`.
