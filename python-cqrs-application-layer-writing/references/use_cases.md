# Use Cases

## Допущения раздела

- Раздел опирается на существующий domain-слой (агрегаты, VO, доменные сервисы) и принятый в проекте паттерн `UnitOfWork`.
- Конкретные имена базовых классов, имена доменных VO и сервисов в примерах — **иллюстрация**. Универсальна только структура.
- Если в проекте нет домена / нет базовых классов / нет UoW-паттерна — это **гейт**: до согласования с пользователем не вводим эти абстракции сами.

## Базовые классы — пример конвенции

Выделяй базовый класс только для реально повторяющихся зависимостей или helper-ов. Не вводи его ради одного use case или одного поля.

**Пример минимального набора:**

```python
# application/command/base.py
from abc import ABC

from application.error import AppInvalidDataError
from application.port.event_publisher import EventPublisher
from application.port.unit_of_work import UnitOfWork
from domain.user import User, UserID


class BaseUseCase(ABC):
    def __init__(self, uow: UnitOfWork) -> None:
        self._uow = uow

    async def _initiator(
        self, uow: UnitOfWork, initiator_id: UserID, action: str
    ) -> User:
        initiator = await uow.user_repositories.read.by_id(initiator_id)
        if initiator is None:
            raise AppInvalidDataError(
                msg="инициатор не существует",
                action=action,
                data={"user": {"user_id": initiator_id.user_id}},
            )
        return initiator


class PublisherUseCase(BaseUseCase):
    def __init__(self, uow: UnitOfWork, event_publisher: EventPublisher) -> None:
        super().__init__(uow)
        self._event_publisher = event_publisher
```

```python
# application/query/base.py — точная копия BaseUseCase, без PublisherUseCase
class BaseUseCase(ABC):
    def __init__(self, uow: UnitOfWork) -> None:
        self._uow = uow

    async def _initiator(
        self, uow: UnitOfWork, initiator_id: UserID, action: str
    ) -> User:
        ...  # та же реализация
```

**Альтернативные варианты:**
- Базовых классов нет вовсе — каждый use case принимает зависимости через свой `__init__`.
- Один базовый класс на всё.
- Много базовых — на каждую категорию (отдельно для user/system/publisher и т.п.).

Скил предписывает **принцип**: общие зависимости и helper-ы выносятся в базовые классы; конкретный набор — конвенция конкретного проекта. Если базовых классов в проекте ещё нет — согласуй с пользователем, какой минимум нужно ввести.

## Конструктор use case

Use case наследует общий конструктор, если базовый класс оправдан переиспользованием. Иначе принимай нужный UoW или специализированный порт напрямую.

```python
class UpdateTenantUseCase(BaseUseCase):
    async def execute(self, command: UpdateTenantCommand) -> TenantDTO:
        ...
```

Если у проекта нет базовых классов — конструктор принимает зависимости напрямую (`UnitOfWork`, при необходимости `EventPublisher`), но всё равно use case содержит только `async def execute(...)` как публичный метод.

## Шаблон `execute` — универсальные пункты и точки согласования

### Универсально (без согласования)

- Сигнатура `async def execute(self, command_or_query) -> ...` (или `async def execute(self) -> ...` для use case без входа).
- Тип возврата — DTO либо `tuple[list[DTO], int]` для list-запросов; никакой утечки domain-объектов.
- **Пред-транзакционная фаза** — все операции, которые могут бросить исключение и не требуют доступа к БД, выполняются **до** открытия транзакции:
  - конверсия примитивов команды/запроса в VO (конструктор VO, `StrEnum.from_str`);
  - синхронная валидация формата/диапазона значений, не зависящая от состояния БД;
  - вычисление производных значений из входа.
- **Границы транзакций** (при наличии согласованных изменений):
  - **Команда** (мутация): один `async with self._uow as uow:` блок на весь `execute`. Атомарность мутаций критична.
  - **Запрос** (чтение): не открывай UoW без транзакционной потребности; для одного источника инжектируй read-порт напрямую.
  - В обоих случаях запрещены **вложенные** блоки.
- Все обращения к репозиториям — через локальный `uow`, не через `self._uow`. Внутри блока: `uow.<aggregate>_repositories.read.<method>(...)`.
- Helper-методы, трогающие репозитории (`_initiator`, `_filtering_data`, любой пользовательский), **принимают `uow` параметром** — `__aenter__` UoW может вернуть транзакционно-связанный объект, отличный от `self._uow`.

### Согласовать с пользователем (опираясь на domain-слой)

- Нужна ли авторизация инициатора и в какой форме (есть ли в домене аналог `User` / role-check).
- Нужен ли per-aggregate policy-check и через какой сервис домена.
- Какие агрегаты загружаются и в каком порядке; что считается primary, что — зависимостью.
- Какие validate-сервисы домена применяются и где в потоке.
- Какие мутаторы агрегата вызываются.
- Какие репозиторные операции выполняются на запись (`read.save`, `version.save`, `outbox.save`) — определяется наличием соответствующих полей в `<Aggregate>Repositories`.
- Какой DTO возвращается и через какую фабрику (`from_domain` / `from_dto`).

### Референс-пример (типовой паттерн)

Шаблон, который в большинстве проектов работает как отправная точка для user command. **Не предписание**, а основа для согласования.

```python
async def execute(self, command: UpdateTenantCommand) -> TenantDTO:
    # ── Phase 1: pre-transaction setup ────────────────────────
    action = "обновление арендатора"
    initiator_id = UserID(command.initiator_id)
    tenant_id = TenantID(command.tenant_id)
    new_status = TenantStatus.from_str(command.status)
    # ──────────────────────────────────────────────────────────
    async with self._uow as uow:
        # ── Phase 2: initiator + role (user only) ─────────────
        initiator = await self._initiator(uow, initiator_id, action)
        initiator.raise_admin()

        # ── Phase 3: load primary ─────────────────────────────
        tenant = await uow.tenant_repositories.read.by_id(tenant_id)
        if tenant is None:
            raise AppNotFoundError(
                msg="арендатор не существует",
                action=action,
                data={"tenant": {"tenant_id": tenant_id.tenant_id}},
            )

        # ── Phase 4: per-aggregate authorization ──────────────
        TenantPolicyService().edit(initiator, [tenant])

        # ── Phase 5: domain mutation ──────────────────────────
        tenant.new_status(new_status)

        # ── Phase 6: save ─────────────────────────────────────
        await uow.tenant_repositories.read.save(tenant)
        await uow.tenant_repositories.version.save(
            tenant, TenantEvent.UPDATED, initiator.user_id
        )

        # ── Phase 7: return DTO ───────────────────────────────
        return TenantDTO.from_domain(tenant)
```

### Применимость фаз по типу use case

| Фаза | user command | system command | user query |
|---|:-:|:-:|:-:|
| 1. action + VO + pre-transaction | ✓ | ✓ | ✓ |
| 2. initiator + role | ✓ (`raise_admin`) | — | ✓ (`raise_reader`) |
| 3. load primary | ✓ | ✓ | ✓ |
| 4. per-aggregate authorization | ✓ (`.edit`) | — | ✓ (`.read`) |
| 5. domain mutation | ✓ | ✓ | — |
| 6. save | ✓ | ✓ (если есть в `<Aggregate>Repositories`) | — |
| 7. return | DTO | DTO или `None` | DTO или `(list[DTO], int)` |

## Initiator

Условия применения:
- **User use case (command/query)** — обязательный шаг сразу после открытия UoW.
- **System use case** — отсутствует. Initiator не передаётся в команду или запрос.
- **Publisher use case** (подтип system) — отсутствует.

Использование:

```python
initiator = await self._initiator(uow, UserID(command.initiator_id), action)
initiator.raise_admin()   # для команд: требуется роль admin
# или
initiator.raise_reader()  # для запросов: достаточно reader
```

Внутри `_initiator`:
- Загрузка `User` (или эквивалентного агрегата-инициатора) из `uow.<user>_repositories.read.by_id(...)`.
- Если `None` — `AppInvalidDataError` (initiator всегда `InvalidData`, не `NotFound`).
- `uow` параметром обязателен — внутри `__aenter__` UoW может вернуть транзакционно-связанный объект, отличный от `self._uow`.

Порядок: `_initiator` → `raise_<role>` → загрузка остальных агрегатов. Обратный порядок (сначала загрузка primary, потом проверка роли) недопустим — нельзя делать I/O от лица неавторизованного инициатора.

## Per-aggregate authorization (PolicyService)

Что это: доменный сервис из `domain/<aggregate>/services.py`, проверяет, имеет ли инициатор право выполнять конкретное действие над конкретным набором агрегатов.

Различие с `raise_admin/reader`:
- `initiator.raise_admin()` — глобальная роль («может что-то менять вообще»).
- `<Aggregate>PolicyService().edit(initiator, [primary])` — право над *этим* объектом (например, только в своём tenant).

Где в шаблоне: **сразу после загрузки соответствующего агрегата, до мутации**. Для list-запроса — после получения списка, над всеми элементами разом: `policy.read(initiator, items)`.

Применимость:
- user command → `.edit(initiator, [primary])`.
- user query → `.read(initiator, items)`.
- system — отсутствует.

Если в домене такого сервиса нет — согласуй с пользователем форму авторизации.

## Транзакционные границы (UoW)

### Когда UoW нужен

- Use case делает **несколько write-операций**, требующих атомарности.
- Use case обращается к **нескольким репозиториям** — вместо инжекции N репозиториев через `__init__` инжектируется один UoW.

### Когда без него

- Use case трогает один репозиторий, write-операция атомарна сама по себе.
- Read-only use case с одним источником.
- Stateless-операция через специализированный порт.

UoW не является универсальной базой use case-а. Не создавай транзакцию, если сценарий не выполняет согласованных изменений.

UoW владеет транзакционной границей, но не обязательно соединением. Если infrastructure передаёт готовое соединение, UoW не закрывает чужой ресурс. Один экземпляр UoW используй для одной прикладной операции и не переиспользуй после выхода из контекста.

### Правила границ блока

При наличии UoW:
- Один `execute` команды = **один** `async with self._uow as uow:` блок (атомарность мутаций).
- Запрос не открывает UoW без транзакционной потребности; для одного источника использует read-порт напрямую.
- Запрещены **вложенные** блоки.
- Все обращения к репозиториям — через локальный `uow`, не через `self._uow`.
- Helper-методы, работающие с репо, принимают `uow` параметром.
- `uow` не сохраняется в state объекта (валиден только в пределах своего блока).

## Обработка ошибок

### Универсально (без согласования)

Use case не ловит application-, domain- и технические исключения. Не добавляй `try/except` ради `action`, логирования или повторной классификации.

- Доменные исключения пробрасываются как есть.
- Технические исключения ловятся в infrastructure-адаптерах и преобразуются в application-ошибки. Адаптер сразу заполняет `action`, а исходную причину сохраняет в `wrap_error`.
- Порт либо возвращает успешный результат, либо бросает исключение; не возвращай `AppError | None` как результат операции.

Логирование внутри use case **не делается** — это не уровень application.

### Anti-patterns

- `try/except AppError:` в use case.
- `try/except DomainError:` в use case.
- `try/except Exception:` оборачивание тех. исключений в use case (это уровень infrastructure).
- Логирование ошибок в use case.
- Catch-and-ignore доменных ошибок.

## Аутентификация и авторизация

- HTTP-аутентификацию, извлечение токена и формирование request context по умолчанию выполняет presentation.
- Передавай в Command/Query уже извлечённый идентификатор инициатора.
- В application проверяй прикладную авторизацию: существование инициатора, состояние, роль и право над конкретным агрегатом.
- Не создавай application use case только для механического переноса идентификатора из JWT в заголовок или request context.
- Порт аутентификации в application нужен только для самостоятельного прикладного сценария, а не для middleware/dependency presentation.
