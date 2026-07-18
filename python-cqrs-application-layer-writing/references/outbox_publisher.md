# Outbox и publisher use case

## Граница application

Application описывает процесс публикации через DTO и порты, но не предписывает устройство таблиц. Отдельная outbox-таблица, version log с маркерами и другая схема хранения — детали infrastructure, если они удовлетворяют прикладному контракту.

Outbox нужен, когда изменение состояния и намерение опубликовать событие должны фиксироваться атомарно. DB-транзакция вокруг сетевого вызова брокера не делает БД и брокер атомарными: уже опубликованное сообщение нельзя откатить при неуспешном commit.

## Команда-мутатор

В одной короткой транзакции сохрани:

1. Текущее состояние агрегата.
2. Версию/аудит, если они предусмотрены.
3. Намерение публикации через outbox-порт.

Не публикуй событие в брокер из обычной команды. Если проект сознательно работает без outbox, зафиксируй модель доставки и требования к идемпотентности отдельно; открытая DB-транзакция не устраняет окно рассогласования.

## Publisher use case

Предпочтительный процесс:

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
        uow: UnitOfWork,
        event_publisher: EventPublisher,
    ) -> None:
        self._outbox_reader = outbox_reader
        self._uow = uow
        self._event_publisher = event_publisher

    async def execute(self) -> None:
        events = await self._outbox_reader.not_published()
        if not events:
            return

        await self._event_publisher.batch_publish(events)

        async with self._uow as uow:
            await uow.event_repository.outbox.mark_as_published(events)
```

Имена портов и repository-групп зависят от проекта; универсальны границы процесса.

## EventPublisher

Используй общий контракт и отдельный alias типов:

```python
from collections.abc import Sequence
from typing import TypeAlias


PublishEventDTO: TypeAlias = UserEventDTO | TenantEventDTO | ProjectEventDTO


class EventPublisher(ABC):
    @abstractmethod
    async def publish(self, event: PublishEventDTO) -> None: ...

    @abstractmethod
    async def batch_publish(
        self,
        events: Sequence[PublishEventDTO],
    ) -> None: ...
```

Добавляй только используемые методы. Не расширяй порт агрегатными методами `publish_users`, `publish_tenants`: infrastructure выполняет исчерпывающий dispatch по runtime-типу DTO, выбирает serializer и subject. Неизвестный тип приводит к `AppInternalError`.

Преобразование DTO в payload, сериализация и выбор subject полностью принадлежат infrastructure. Application только композирует чтение, публикацию и отметку.

## Batch-семантика

Частичный успех считай неуспехом всего batch:

- не отмечай опубликованным ни один элемент при ошибке;
- пробрасывай ошибку адаптера;
- повторный запуск получает тот же batch;
- используй стабильный идентификатор события;
- обеспечивай дедупликацию брокером или идемпотентность потребителя.

Физически опубликованные до ошибки сообщения нельзя отозвать, поэтому повторная доставка является ожидаемой частью модели at-least-once.

## Запреты

- DB-транзакция на время сетевой публикации без необходимости.
- Публикация из обычной команды вместо outbox.
- Отметка `published` до полного успеха batch.
- Скрытие частичной ошибки адаптером.
- Формирование broker payload или subject в application.
- Создание UoW для пустого batch.
