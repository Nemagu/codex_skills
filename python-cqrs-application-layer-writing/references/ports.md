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

Генерация или резервирование ID не относится к repository. Тип значения задаётся
контрактом конкретного источника, а не ограничивается UUID:

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar


IdentifierT = TypeVar("IdentifierT")


class IdentifierProvider(ABC, Generic[IdentifierT]):
    @abstractmethod
    async def next_id(self) -> IdentifierT:
        ...


class Clock(ABC):
    @abstractmethod
    def now(self) -> datetime:
        ...
```

Имя общего generic-порта является примером. Если разные источники имеют разную
семантику, создай отдельные минимальные контракты. Use case преобразует `UUID`,
`int`, `str` или другое полученное значение в конкретный ID-VO и передаёт его
доменной фабрике.

Если `int` резервируется через DB sequence и резервирование входит в атомарную
операцию, источник должен использовать тот же transaction context: включи его в
UoW либо предоставь эквивалентный транзакционно связанный контракт. Не открывай
скрытое независимое соединение и не поручай назначение доменного ID методу
repository `save`.

Create-команда не содержит ID нового агрегата без заданной внешней семантики.
Назначенный ID возвращается через публичный application DTO.

Аналогично оформляй время, случайность и внешнее решение. Не создавай общий
`UtilsPort`.

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

Если единый outbox-порт работает с несколькими конкретными outbox DTO, объяви
их закрытое объединение через явный `TypeAlias` и используй этот alias во всех
методах сохранения, выборки, публикации и изменения состояния. Не вводи
application-перечисление типов ресурсов только для dispatch адаптера.

Добавляй только заданные операции состояния. Для терминальных исходов предпочитай
различимые методы по смыслу (`mark_published`, `mark_rejected`) перед передачей
инфраструктурного status enum. Повтор того же терминального перехода и конфликт
разных переходов реализуй буквально по контракту.

## Publisher

Application-порт публикации принимает application/port DTO. Адаптер выбирает
serializer, payload, subject и transport headers.

```python
class EventPublisher(ABC):
    @abstractmethod
    async def publish(self, event: PublishEventDTO) -> None:
        ...
```

Добавляй batch-метод только когда application-операция действительно передаёт
набор одним вызовом. Не вводи его ради оптимизации адаптера, тестовой подготовки
данных или предполагаемого будущего сценария. Размер batch, порядок, повторы,
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
