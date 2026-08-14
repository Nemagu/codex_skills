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

Version DTO и outbox DTO — разные application-контракты. Первый описывает
исторический снимок, второй — намерение публикации и содержит стабильный
идентификатор внешнего события. Не передавай version DTO напрямую в outbox.
Сформируй outbox DTO явно из version DTO и идентификатора, полученного через
отдельный application-порт; persistence- и transport-адаптеры этот идентификатор
не генерируют.

Если эти ответственности представлены тремя отдельными портами, use case обязан
явно вызвать порт текущего состояния, порт истории и outbox-порт внутри одного
UoW. Не скрывай один вызов внутри реализации другого порта. Для каждой созданной
версии последовательно сохраняй её текущее состояние, version DTO и
соответствующую outbox-запись; при нескольких версиях повторяй все три шага.

Не публикуй событие в брокер из обычной команды. Если проект сознательно работает без outbox, зафиксируй модель доставки и требования к идемпотентности отдельно; открытая DB-транзакция не устраняет окно рассогласования.

## Publisher use case

Реализуй заданный процесс. Вариант для одного следующего события:

```text
прочитать одно следующее событие без транзакции
→ опубликовать событие
→ при успехе открыть короткий UoW
→ отметить событие опубликованным
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
        event = await self._outbox_reader.next()
        if event is None:
            return

        await self._event_publisher.publish(event)

        async with self._uow_factory() as uow:
            await uow.event_repository.outbox.mark_as_published(event)
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
```

Добавляй только используемые методы. Пакетный метод появляется лишь при
фактическом пакетном application-сценарии, а не для удобства адаптера или тестов.
Не расширяй порт агрегатными методами `publish_users`, `publish_tenants`:
infrastructure выполняет исчерпывающий dispatch по runtime-типу DTO, выбирает
serializer и subject. Неизвестный тип приводит к `AppInternalError`.

Этот `TypeAlias` обязателен и для единого outbox-порта, если он сохраняет или
возвращает те же конкретные outbox DTO. Не заменяй alias общим базовым DTO, `Any` либо
перечислением типов ресурсов. Application use case передаёт конкретный DTO между
outbox и publisher без transport-поведения; адаптеры выполняют dispatch по его
runtime-типу.

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
