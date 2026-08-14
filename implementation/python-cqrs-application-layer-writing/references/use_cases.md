# Варианты использования

## Допущения раздела

- Раздел опирается на существующий domain-слой (агрегаты, VO, доменные сервисы) и принятый в проекте паттерн `UnitOfWork`.
- Конкретные имена базовых классов, имена доменных VO и сервисов в примерах — **иллюстрация**. Универсальна только структура.
- Не вводи domain, UoW, инициатора, авторизацию и порты, отсутствующие во входном
  контракте операции.

## Базовые классы — пример конвенции

Выделяй базовый класс только для реально повторяющихся зависимостей или helper-ов. Не вводи его ради одного use case или одного поля.

**Пример минимального набора:**

```python
# application/command/base.py
from abc import ABC
from typing import ClassVar

from application.error import AppInvalidDataError
from application.port.event_publisher import EventPublisher
from application.port.unit_of_work import UnitOfWork, UnitOfWorkFactory
from domain.user import User, UserID


class BaseUseCase(ABC):
    ACTION: ClassVar[str]

    def __init__(self, uow_factory: UnitOfWorkFactory) -> None:
        self._uow_factory = uow_factory

    async def _load_initiator(
        self, uow: UnitOfWork, initiator_id: UserID
    ) -> User:
        initiator = await uow.user_repositories.read.by_id(initiator_id)
        if initiator is None:
            raise AppInvalidDataError(
                msg="инициатор не существует",
                action=self.ACTION,
                data={"user": {"user_id": initiator_id.user_id}},
            )
        return initiator


class PublisherUseCase(BaseUseCase):
    def __init__(
        self,
        uow_factory: UnitOfWorkFactory,
        event_publisher: EventPublisher,
    ) -> None:
        super().__init__(uow_factory)
        self._event_publisher = event_publisher
```

```python
# application/query/base.py — точная копия BaseUseCase, без PublisherUseCase
class BaseUseCase(ABC):
    ACTION: ClassVar[str]

    def __init__(self, uow_factory: UnitOfWorkFactory) -> None:
        self._uow_factory = uow_factory

    async def _load_initiator(
        self, uow: UnitOfWork, initiator_id: UserID
    ) -> User:
        ...  # та же реализация
```

**Альтернативные варианты:**
- Базовых классов нет вовсе — каждый use case принимает зависимости через свой `__init__`.
- Один базовый класс на всё.
- Много базовых — на каждую категорию (отдельно для user/system/publisher и т.п.).

Общие зависимости и helpers выноси только при фактическом повторении. Не создавай
иерархию базовых классов по предполагаемому будущему использованию.

## Конструктор use case

Use case наследует общий конструктор, если базовый класс оправдан переиспользованием. Иначе принимай обязательную фабрику UoW или специализированный порт напрямую.

```python
class UpdateTenantUseCase(BaseUseCase):
    async def execute(self, command: UpdateTenantCommand) -> TenantDTO:
        ...
```

Если у проекта нет базовых классов — конструктор принимает зависимости напрямую (`UnitOfWork`, при необходимости `EventPublisher`), но всё равно use case содержит только `async def execute(...)` как публичный метод.

Храни в экземпляре только зависимости. Command, Query, локальный UoW, агрегаты,
результаты и ошибки держи в локальных переменных. Не кешируй результат вызова в
`self`; заданный кеш реализуй отдельным портом. Не используй изменяемые class
attributes.

Для стабильного контекста ошибок объявляй:

```python
from typing import ClassVar


class UpdateTenantUseCase:
    ACTION: ClassVar[str] = "обновление арендатора"
```

ID, текущее время, случайность и другие недетерминированные значения получай через
отдельные минимальные порты. Источник ID возвращает заданный технический тип
(`UUID`, `int`, `str` или другой), а use case оборачивает значение в конкретный
доменный ID-VO до вызова фабрики. Не вызывай `uuid4()`/`datetime.now()` в use case
и не размещай `next_id()` в repository-порте.

Для создания агрегата не требуй его новый ID во входной команде, если ID не
выбирается вызывающей стороной по внешней семантике. Верни назначенный ID в
публичном application-результате. Если источник резервирует DB sequence,
используй его внутри того же UoW, когда резервирование входит в транзакционную
границу; не открывай скрытое независимое соединение.

Для набора не вызывай порт в цикле: используй реально требуемый batch-метод с
`tuple`. Не предполагай порядок результата без гарантии контракта.

## Шаблон `execute`

### Универсально (без согласования)

- Сигнатура `async def execute(self, command_or_query) -> ...` (или `async def execute(self) -> ...` для use case без входа).
- Тип возврата — точный публичный DTO; для limit/offset с total допустим
  `tuple[tuple[DTO, ...], int]`. Domain-объекты наружу не возвращаются.
- **Пред-транзакционная фаза** — все операции, которые могут бросить исключение и не требуют доступа к БД, выполняются **до** открытия транзакции:
  - конверсия примитивов команды/запроса в VO (конструктор VO, `StrEnum.from_str`);
  - синхронная валидация формата/диапазона значений, не зависящая от состояния БД;
  - вычисление производных значений из входа.
- **Границы транзакций** (при наличии согласованных изменений):
  - **Команда**: один новый UoW из обязательной фабрики только для заданной атомарной группы.
  - **Запрос** (чтение): не открывай UoW без транзакционной потребности; для одного источника инжектируй read-порт напрямую.
  - В обоих случаях запрещены **вложенные** блоки.
- Use case хранит только stateless `UnitOfWorkFactory`; каждый `execute` вызывает её заново.
- Все обращения к репозиториям — через локальный `uow`, не через поле use case. Внутри блока: `uow.<aggregate>_repositories.read.<method>(...)`.
- Helper-методы, трогающие репозитории (`_load_initiator`, `_load_resource`,
  любой пользовательский), **принимают `uow` параметром**.

### Ответственность helper-методов

`execute` явно вызывает шаги сценария в порядке контракта: преобразование входа,
загрузку каждого объекта, авторизацию, доменную операцию, создание version DTO и
сохранение состояния, версии и outbox.

Каждый helper выполняет только одну функцию:

- `_load_initiator` и `_load_<resource>` загружают один объект и преобразуют его
  отсутствие в заданную application-ошибку;
- `_authorize_<operation>` только вызывает заданную domain-policy;
- `_make_version` только создаёт immutable version DTO;
- чистый filtering helper только преобразует входные фильтры.

Не создавай `_entities`/`_owners`, возвращающие tuple разнородных объектов; не
передавай флаг `initiator` или callback/policy для переключения поведения. Не
скрывай в `_save` последовательность state/version/outbox/`mark_persisted` — эти
точки должны быть видны в `execute`.

### Реализовать из входного контракта

- точный вход и результат;
- заданную авторизацию и порядок загрузки;
- доменные операции и преобразования их отказов;
- транзакционную границу и точки сохранения;
- version/outbox DTO и публичные исходы.

### Референс-пример (типовой паттерн)

Пример показывает форму кода, но не добавляет операции типовые шаги.

```python
@handle_domain_errors
async def execute(self, command: UpdateTenantCommand) -> TenantDTO:
    # ── Phase 1: pre-transaction setup ────────────────────────
    initiator_id = UserID(command.initiator_id)
    tenant_id = TenantID(command.tenant_id)
    new_status = TenantStatus.from_str(command.status)
    # ──────────────────────────────────────────────────────────
    async with self._uow_factory() as uow:
        # ── Phase 2: initiator + role (user only) ─────────────
        initiator = await self._load_initiator(uow, initiator_id)
        initiator.raise_admin()

        # ── Phase 3: load primary ─────────────────────────────
        tenant = await uow.tenant_repositories.read.by_id(tenant_id)
        if tenant is None:
            raise AppNotFoundError(
                msg="арендатор не существует",
                action=self.ACTION,
                data={"tenant": {"tenant_id": tenant_id.tenant_id}},
            )

        # ── Phase 4: per-aggregate authorization ──────────────
        TenantPolicyService().edit(initiator, (tenant,))

        # ── Phase 5: domain mutation ──────────────────────────
        tenant.new_status(new_status)

        # ── Phase 6: save ─────────────────────────────────────
        await uow.tenant_repositories.read.save(tenant)
        version_record = TenantVersionRecordDTO(
            snapshot=TenantSnapshotDTO.from_domain(tenant),
            change=TenantChange.UPDATED,
            editor_id=initiator.user_id,
        )
        await uow.tenant_repositories.version.save(version_record)

        # ── Phase 7: return DTO ───────────────────────────────
        return TenantDTO.from_domain(tenant)
```

### Применимость фаз по типу use case

| Фаза | user command | system command | user query |
|---|:-:|:-:|:-:|
| 1. action + VO + pre-transaction | ✓ | ✓ | ✓ |
| 2. initiator + role | только по контракту | только по контракту | только по контракту |
| 3. load primary | ✓ | ✓ | ✓ |
| 4. domain-полномочие | только по контракту | только по контракту | только по контракту |
| 5. domain mutation | ✓ | ✓ | — |
| 6. save | заданные точки | заданные точки | — |
| 7. return | заданный DTO | заданный DTO или `None` | заданный DTO |

## Initiator

Добавляй инициатора и его загрузку только по заданному контракту. `user`/`system`
не определяют автоматически поле `initiator_id`, роль и форму ошибки.

Использование:

```python
initiator = await self._load_initiator(uow, UserID(command.initiator_id))
initiator.raise_admin()   # для команд: требуется роль admin
# или
initiator.raise_reader()  # для запросов: достаточно reader
```

Если helper `_load_initiator` задан и переиспользуется:
- Загрузка `User` (или эквивалентного агрегата-инициатора) из `uow.<user>_repositories.read.by_id(...)`.
- Если `None` — заданный публичный application-исход.
- `uow` параметром обязателен — экземпляр существует только внутри текущего блока.

Авторизацию выполняй до защищаемого действия и необязательной загрузки закрытых
данных, сохраняя точный порядок операции.

## Per-aggregate authorization (PolicyService)

Что это: доменный сервис из `domain/<aggregate>/services.py`, проверяет, имеет ли инициатор право выполнять конкретное действие над конкретным набором агрегатов.

Различие с `raise_admin/reader`:
- `initiator.raise_admin()` — глобальная роль («может что-то менять вообще»).
- `<Aggregate>PolicyService().edit(initiator, [primary])` — право над *этим* объектом (например, только в своём tenant).

Где в шаблоне: **сразу после загрузки соответствующего агрегата, до мутации**. Для list-запроса — после получения списка, над всеми элементами разом: `policy.read(initiator, items)`.

Вызывай только явно заданное доменное полномочие. Не добавляй `.edit()`, `.read()`
или policy-сервис по типу операции.

## Транзакционные границы (UoW)

### Когда UoW нужен

- Use case делает **несколько write-операций**, требующих атомарности.
- Несколько портов участвуют в одной заданной атомарной группе.

### Когда без него

- Use case трогает один репозиторий, write-операция атомарна сама по себе.
- Read-only use case с одним источником.
- Stateless-операция через специализированный порт.

UoW не является универсальной базой use case-а. Не создавай транзакцию, если сценарий не выполняет согласованных изменений.

Use case хранит обязательную `UnitOfWorkFactory`, а не UoW. Фабрика возвращает
новый экземпляр на каждый транзакционный вызов и новую retry-попытку.
`__aenter__()` возвращает транзакционно связанный UoW. Успешный выход выполняет
commit, любое исключение или отмена — rollback. `__aexit__()` не подавляет
исключение. Use case не вызывает commit/rollback вручную, не переиспользует UoW
и его репозитории после выхода и не закрывает ресурс, которым UoW не владеет.

### Правила границ блока

При наличии UoW:
- Один `execute` команды = **один** `async with self._uow_factory() as uow:` блок (атомарность мутаций).
- Запрос не открывает UoW без транзакционной потребности; для одного источника использует read-порт напрямую.
- Запрещены **вложенные** блоки.
- Все обращения к репозиториям — через локальный `uow`, не через поле use case.
- Helper-методы, работающие с репо, принимают `uow` параметром.
- `uow` не сохраняется в state объекта (валиден только в пределах своего блока).
- Фабрика не должна возвращать ранее использованный экземпляр.

## Обработка ошибок

### Преобразование

- Оборачивай весь `execute` явным декоратором application-границы: это покрывает
  конструирование VO, фабрики, агрегаты и доменные сервисы без локальных блоков.
- Общая карта содержит только стабильные соответствия; неоднозначные типы вроде
  `EntityNotFoundError` задавай override-картой конкретного use case.
- Объявляй `ACTION` как `ClassVar[str]`; декоратор использует его для результата.
- Неизвестный `DomainError` преобразуй в `AppInternalError` с `wrap_error`.
- Адаптер преобразует ошибку зависимости в `AppPortError` с операцией порта.
- Use case преобразует `AppPortError` в публичный исход со своим `ACTION`.
- Не перехватывай `BaseException`, `CancelledError` и публичный `AppError`;
  внутренний `AppPortError` преобразуется отдельно.
- Всегда сохраняй цепочку через `raise ... from error`.

Use case и UoW не логируют ошибки. Они сохраняют безопасный контекст и цепочку
причин и пробрасывают заданный результат.

### Anti-patterns

- Повторное оборачивание `AppError`.
- Пропуск `DomainError` или `AppPortError` во входящий адаптер.
- Перехват `BaseException` или отмены задачи.
- Логирование ошибок в use case.
- Логирование обработанной попытки в UoW.
- Локальные `try/except DomainError` вокруг каждого VO или доменного вызова.
- Catch-and-ignore доменных ошибок.

## Аутентификация и авторизация

- HTTP-аутентификацию, извлечение токена и формирование request context по умолчанию выполняет presentation.
- Передавай в Command/Query уже извлечённый идентификатор инициатора.
- В application проверяй прикладную авторизацию: существование инициатора, состояние, роль и право над конкретным агрегатом.
- Не создавай application use case только для механического переноса идентификатора из JWT в заголовок или request context.
- Порт аутентификации в application нужен только для самостоятельного прикладного сценария, а не для middleware/dependency presentation.
