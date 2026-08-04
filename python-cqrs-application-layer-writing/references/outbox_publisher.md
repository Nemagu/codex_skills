# Outbox и publisher use case

## Граница application

Application описывает процесс публикации через DTO и порты, но не предписывает устройство таблиц. Отдельная outbox-таблица, version log с маркерами и другая схема хранения — детали infrastructure, если они удовлетворяют прикладному контракту.

Добавляй outbox только когда операция требует атомарно зафиксировать изменение и
намерение публикации. Не применяй его автоматически ко всем командам.

## Команда-мутатор

В одной короткой транзакции сохрани:

1. Текущее состояние агрегата.
2. Заданные записи версий.
3. Самодостаточные immutable outbox DTO для публикуемых версий.

Не публикуй событие в брокер из обычной команды. Если проект сознательно работает без outbox, зафиксируй модель доставки и требования к идемпотентности отдельно; открытая DB-транзакция не устраняет окно рассогласования.

## Publisher use case

Реализуй заданный процесс. Распространённый вариант:

```text
прочитать неопубликованный batch без транзакции
→ опубликовать весь batch
→ при полном успехе открыть короткий UoW
→ отметить весь batch опубликованным
→ commit
```

Не удерживай DB-транзакцию во время сетевого I/O, если до успешной публикации нет согласованных изменений состояния.

```python
class PublishEventsUseCase:
    def __init__(
        self,
        outbox_reader: OutboxReader,
        uow_factory: UnitOfWorkFactory,
        event_publisher: EventPublisher,
    ) -> None:
        self._outbox_reader = outbox_reader
        self._uow_factory = uow_factory
        self._event_publisher = event_publisher

    async def execute(self) -> None:
        events = await self._outbox_reader.not_published()
        if not events:
            return

        await self._event_publisher.batch_publish(events)

        async with self._uow_factory() as uow:
            await uow.event_repository.outbox.mark_as_published(events)
```

Имена портов и repository-групп зависят от проекта; универсальны границы процесса.
Возврат `None` в примере допустим только когда контракт не требует различать
ожидаемые результаты. Если вызывающий процесс должен отличать пустой outbox,
успешную публикацию, повтор и окончательный отказ, возвращай заданное публичное
DTO-перечисление и не используй исключение для пустого outbox.

## EventPublisher

Используй общий контракт и отдельный alias типов:

```python
from typing import TypeAlias


PublishEventDTO: TypeAlias = UserEventDTO | TenantEventDTO | ProjectEventDTO


class EventPublisher(ABC):
    @abstractmethod
    async def publish(self, event: PublishEventDTO) -> None: ...

    @abstractmethod
    async def batch_publish(
        self,
        events: tuple[PublishEventDTO, ...],
    ) -> None: ...
```

Добавляй только используемые методы. Не расширяй порт агрегатными методами `publish_users`, `publish_tenants`: infrastructure выполняет исчерпывающий dispatch по runtime-типу DTO, выбирает serializer и subject. Неизвестный тип приводит к `AppInternalError`.

Преобразование DTO в payload, сериализация и выбор subject полностью принадлежат infrastructure. Application только композирует чтение, публикацию и отметку.

## Batch-семантика

Размер batch, порядок, частичный успех, повтор и отметку публикации реализуй
буквально по контракту publisher-операции. Не навязывай whole-batch retry.
Передавай коллекции как `tuple`; стабильный publication ID и дедупликацию добавляй
только при заданной at-least-once модели.

## Запреты

- DB-транзакция на время сетевой публикации без необходимости.
- Публикация из обычной команды вместо outbox.
- Отметка `published` до полного успеха batch.
- Скрытие частичной ошибки адаптером.
- Формирование broker payload или subject в application.
- Создание UoW для пустого batch.
