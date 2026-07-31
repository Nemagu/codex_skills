# Порты

## Общие правила

Создавай только исходящие возможности, необходимые заданным application-операциям.
Порт выражает смысл возможности, а не технологию или устройство адаптера.

- Используй `ABC`, `@abstractmethod`, `async` для I/O и тело только `...`.
- Не импортируй infrastructure и presentation.
- Добавляй только используемые методы, параметры, результаты и гарантии.
- Domain-объекты и VO допустимы для хранения и получения доменного состояния.
- Для сложного обмена используй самодостаточный immutable port DTO.
- Не наследуй application-порт от domain-порта без полного совпадения смысла.
- Не добавляй batch, retry, timeout и порядок без требования.
- Для коллекций используй `tuple`.

## Unit of Work

Включай в UoW только порты одной заданной транзакционной границы:

```python
from abc import ABC, abstractmethod
from types import TracebackType
from typing import Self


class UnitOfWork(ABC):
    @abstractmethod
    async def __aenter__(self) -> Self:
        ...

    @abstractmethod
    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc_value: BaseException | None,
        traceback: TracebackType | None,
    ) -> None:
        ...


class UnitOfWorkFactory(ABC):
    @abstractmethod
    def __call__(self) -> UnitOfWork:
        ...
```

Контракт:

- `__aenter__()` возвращает транзакционно связанный экземпляр;
- успешный выход выполняет commit;
- исключение и отмена задачи выполняют rollback;
- `__aexit__()` не подавляет исключение;
- use case не вызывает commit/rollback вручную;
- `UnitOfWorkFactory` создаёт новый экземпляр для каждого транзакционного вызова;
- use case хранит фабрику, но не хранит и не переиспользует UoW;
- один экземпляр UoW используется один раз;
- репозитории недействительны после выхода;
- UoW закрывает только принадлежащие ему ресурсы.

Независимый read-, cache-, publisher- или stateless-порт инжектируй напрямую.

## Repository-порты

Создавай узкие контракты чтения и сохранения по фактическим операциям. Не добавляй
автоматически `read`, `version`, `outbox` и `subscription` каждому агрегату.

```python
class ProjectRepository(ABC):
    @abstractmethod
    async def by_id(self, project_id: ProjectID) -> Project | None:
        ...

    @abstractmethod
    async def save(self, project: Project) -> None:
        ...
```

Если use case работает с набором, не вызывай `by_id()` или `save()` в цикле.
Создай используемый batch-метод и определи соответствие результатов входам:

```python
class ProjectBatchReader(ABC):
    @abstractmethod
    async def by_ids(
        self,
        project_ids: tuple[ProjectID, ...],
    ) -> tuple[Project, ...]:
        ...
```

Не предполагай сохранение порядка без явной гарантии. Отсутствующие элементы
обрабатывай согласно публичному исходу операции.

## Источники значений

Генерация ID не относится к repository. Для каждого требуемого источника создавай
отдельный минимальный порт:

```python
class ProjectIDProvider(ABC):
    @abstractmethod
    async def next_id(self) -> UUID:
        ...


class Clock(ABC):
    @abstractmethod
    def now(self) -> datetime:
        ...
```

Аналогично оформляй случайность и внешнее решение. Не создавай общий `UtilsPort`.
Use case преобразует значение в VO и передаёт ID доменной фабрике.

## Кеш

Кеширование реализуй только отдельным портом и по заданной семантике ключа,
свежести, cache miss, недоступности и инвалидирования. Не храни кешированные данные
в полях use case.

- Ключ учитывает tenant, область доступа и версию, если они влияют на результат.
- Порт принимает application DTO или специализированный port DTO.
- Технологию, сериализацию и TTL реализует адаптер.
- Точка инвалидирования относительно commit задаётся операцией.

## Version DTO и repository

Не передавай в `save()` изменяемый агрегат вместе с отдельными `event` и
`editor_id`. Сформируй единый immutable record в заданной точке сохранения:

```python
@dataclass(slots=True, frozen=True)
class ProjectSnapshotDTO:
    project_id: ProjectID
    status: ProjectStatus
    version: Version


@dataclass(slots=True, frozen=True)
class ProjectVersionRecordDTO:
    snapshot: ProjectSnapshotDTO
    change: ProjectChange
    editor_id: MemberID | None


class ProjectVersionRepository(ABC):
    @abstractmethod
    async def save(self, record: ProjectVersionRecordDTO) -> None:
        ...

    @abstractmethod
    async def batch_save(
        self,
        records: tuple[ProjectVersionRecordDTO, ...],
    ) -> None:
        ...
```

Snapshot DTO фиксирует VO и immutable-коллекции в конкретной точке. Он не хранит
ссылку на изменяемый агрегат. Реализуй ровно заданные точки сохранения: несколько
доменных операций могут образовать одну версию либо несколько самостоятельных
версий.

`ProjectChange` — тип сохранённого изменения application-слоя, а не доменное
событие. Event sourcing и коллекция событий агрегата не используются.

## Outbox

Outbox-порт принимает единый immutable DTO с полным минимально достаточным набором
данных для сохранения намерения без дополнительного чтения агрегата или другого
порта:

```python
class ProjectOutboxRepository(ABC):
    @abstractmethod
    async def save(self, record: ProjectOutboxRecordDTO) -> None:
        ...
```

В DTO допустимы ID, версия, тип изменения, редактор, publication ID и другие
заданные метаданные. Полный снимок включай только если он действительно хранится
в outbox. Не включай subject, stream, serialized payload и headers без
application-смысла.

Version record и связанный outbox record сохраняй в одной заданной транзакции.
Outbox создавай только для публикуемой версии.

## Publisher

Application-порт публикации принимает application/port DTO. Адаптер выбирает
serializer, payload, subject и transport headers.

```python
class EventPublisher(ABC):
    @abstractmethod
    async def publish(self, event: PublishEventDTO) -> None:
        ...

    @abstractmethod
    async def batch_publish(
        self,
        events: tuple[PublishEventDTO, ...],
    ) -> None:
        ...
```

Добавляй только реально используемый метод. Размер batch, порядок, повторы,
частичный успех и отметку публикации реализуй согласно операции.

## Ошибки портов

Адаптер перехватывает только ожидаемые исключения зависимости и создаёт
`AppPortError` с безопасным контекстом операции порта и `wrap_error`. Не
перехватывай `BaseException`, `CancelledError` и публичный `AppError`.

Use case перехватывает `AppPortError` вокруг минимального вызова и преобразует её
в заданный публичный исход со своим `ACTION`. `AppPortError` не должен достигать
входящего адаптера.

## Группы портов

Создавай dataclass-группу внутри UoW только для нескольких портов одной устойчивой
области и транзакции. Группа — `@dataclass(slots=True)` без `frozen=True`.
Не группируй порты только по агрегату, если их application-смысл различается.
