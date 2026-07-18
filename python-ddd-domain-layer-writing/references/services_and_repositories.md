# Domain Services and Repository Interfaces

## Domain Services

### Назначение и расположение

Domain service — место для логики, которая не помещается ни в один агрегат, ни в проекцию: либо требует доступа к репозиторию, либо находится «между» агрегатами и не принадлежит ни одному из них.

Расположение: `domain/<aggregate>/service.py`. Один файл на агрегат, в нём — все сервисы, относящиеся к этому агрегату. Если файл становится большим (>200 строк) — можно разделить на отдельные модули, но это редкий случай.

### Когда нужен domain service vs другие места

| Логика | Куда положить |
|---|---|
| Затрагивает только один агрегат и его поля | Метод самого агрегата (`new_X`, `activate`) |
| Требует данных из БД (проверка существования, чтение другого агрегата по полю) | **Domain service** |
| Валидация против другого агрегата, который **передаётся** в метод из use case | Метод того агрегата, чей инвариант проверяется (`PersonalTransaction.validate_categories`) |
| Кросс-агрегатное правило, не принадлежащее ни одному из них | **Domain service** (Policy) |
| Простое создание из данных пользователя без внешних проверок | Фабрика напрямую, без сервиса |
| Чисто техническая работа (сериализация, кэш, мэппинг) | Не domain — другой слой |

**Принцип «передаётся → метод агрегата»:** если обе сущности уже у нас в памяти и нужно проверить отношение между ними, инициатор которого — один из них (его инвариант), это метод этого агрегата. Если отношение симметричное и не принадлежит ни одному — это policy service.

### Общий контракт

Любой domain-сервис подчиняется одному набору правил, независимо от назначения. Если задача не помещается ни в один из перечисленных ниже типов — оформляем сервис по общему контракту и при необходимости расширяем скил новым типом.

**1. Когда вообще создавать сервис.** Сервис нужен, если выполнено хотя бы одно:
- Логика затрагивает ≥ 2 агрегатов/проекций и не принадлежит ни одному из них.
- Логика требует обращения к репозиторию (чтение по полю, проверка существования, выборка по условию).
- Логика выполняет что-то «между» VO/агрегатом — оркестрация без владельца.

Если логика помещается в метод самого агрегата (трогает только его поля или принимает другой агрегат-параметром без I/O) — это **не** сервис, а метод агрегата.

**2. Имя класса:** отражает конкретное бизнес-правило: `UserUniquenessService`, `TransactionOwnershipPolicy`.

**3. Состояние класса:**
- **Без I/O** → `@staticmethod` методы, нет конструктора, нет полей. Класс — namespace.
- **С I/O** → конструктор принимает только узкие repo-интерфейсы из `domain/<aggregate>/repository.py`, поля только для них. Никакого изменяемого состояния — сервис идемпотентен между вызовами.

**4. Метод(ы):**
- Имя — глагол, отражающий действие: `create`, `validate_<field>`, `raise_<rule>`, `calculate_<thing>`, `synchronize_<thing>`.
- Параметры — full-сущности, проекции, VO. **Никогда** примитивы (`UUID`, `str`, `int`).
- Все аргументы передаются по имени.
- Возврат:
  - Результат доменного расчёта — **либо**
  - `None` для validate-/raise- методов (успех молчаливый, нарушение через доменную ошибку).
- `async` если внутри метода есть обращения к репозиторию; `sync` иначе.

**5. Ошибки:** бросаем доменные ошибки напрямую (`EntityAlreadyExistsError`, `EntityPolicyError`, `EntityInvalidDataError`). Не оборачиваем чужие, не подменяем класс. `try/except` внутри сервиса не используем.

**6. Запреты:**
- Импорт из `application/`, `infrastructure/`, `presentation/`.
- Создание и сохранение агрегатов.
- Генерация ID.
- Вызов фабрики после успешной проверки.
- Любое I/O помимо вызовов узких read-репозиториев из domain (никаких HTTP/брокеров/файлов).

**7. Размещение:** `domain/<aggregate>/service.py`. Несколько сервисов одного агрегата живут в одном файле, пока он не разрастается.

### Распространённые типы

Дальше — два распространённых примера. Если задача не вписывается в них, назови сервис по конкретному бизнес-правилу и следуй общему контракту выше.

---

#### Тип A — Uniqueness service

**Когда применять:** нужно проверить, что агрегат с заданным атрибутом не существует (или конкретный — существует). Требует обращения в БД через read-репозиторий.

**Конвенции:**
- Имя класса: `<Aggregate>UniquenessService`.
- Зависимости — read-репозиторий, инъекция через конструктор.
- Метод(ы): `async def validate_<field>(self, ...) -> None`.
- Возврат — `None` при успехе. При нарушении — `EntityAlreadyExistsError`.
- Несколько `validate_*` методов в одном сервисе допустимы.

**Пример (один метод):**

```python
class UserUniquenessService:
    def __init__(self, read_repository: UserReadRepository) -> None:
        self._read_repo = read_repository

    async def validate_user_id(self, user_id: UserID) -> None:
        existing = await self._read_repo.by_id(user_id)
        if existing:
            raise EntityAlreadyExistsError(
                msg=f'пользователь с id "{existing.user_id.user_id}" уже существует',
                subject=existing.projection_name.name,
                data={"user": {"user_id": existing.user_id.user_id}},
            )
```

**Пример (с дополнительной семантикой состояния):**

```python
class TransactionCategoryUniquenessService:
    def __init__(self, read_repository: TransactionCategoryReadRepository) -> None:
        self._read_repo = read_repository

    async def validate_name(
        self,
        tenant: Tenant,
        name: TransactionCategoryName,
    ) -> None:
        category = await self._read_repo.by_owner_id_name(tenant.tenant_id, name)
        if category is None:
            return

        if category.state.is_deleted():
            raise EntityAlreadyExistsError(
                msg="название категории транзакции уже используется, но она была ранее удалена",
                subject=category.aggregate_name.name,
                data={
                    "category_id": category.category_id.category_id,
                    "name": name.name,
                },
            )
        raise EntityAlreadyExistsError(
            msg="название категории транзакции уже используется",
            subject=category.aggregate_name.name,
            data={
                "category_id": category.category_id.category_id,
                "name": name.name,
            },
        )
```

**Правила:**
- Параметры — VO (или связанные агрегаты), не примитивы.
- `subject` берётся из `aggregate_name.name` найденного агрегата.
- При различающихся бизнес-сообщениях (как с `is_deleted`) — отдельная ветка.

---

#### Тип B — Policy service

**Когда применять:** правило между двумя или более агрегатами/проекциями, не принадлежащее ни одному. Не требует обращения в БД — обе сущности уже в памяти.

**Конвенции:**
- Имя класса отражает правило, например `TransactionOwnershipPolicy`.
- Зависимостей нет — конструктор отсутствует.
- Все методы — `@staticmethod`.
- Метод: `def raise_<rule>(...) -> None`.
- Возврат — `None` при успехе. При нарушении — `EntityPolicyError`.

```python
class TransactionOwnershipPolicy:
    @staticmethod
    def raise_owner(tenant: Tenant, category: TransactionCategory) -> None:
        if tenant.tenant_id != category.owner_id:
            raise EntityPolicyError(
                msg="только владелец может работать с категорией транзакции",
                subject=tenant.aggregate_name.name,
                data={
                    "tenant": {"tenant_id": tenant.tenant_id.tenant_id},
                    "category": {
                        "category_id": category.category_id.category_id,
                        "owner_id": category.owner_id.tenant_id,
                    },
                },
            )
```

**Правила:**
- Параметры — оба агрегата целиком, не их части.
- Имя метода — `raise_<rule>` (как у методов-предикатов агрегата).
- В `data` кладём оба агрегата с их идентификаторами и полями, которые участвовали в нарушении.
- `subject` обычно — имя того агрегата, чьё право проверяется, но допустимо и имя ущемляемой стороны.

---

### Зависимости и DI

- Сервис зависит **только от абстракций** из `domain/<aggregate>/repository.py`. Никаких импортов из `application/`, `infrastructure/`, `presentation/`.
- Конкретная реализация репозитория инжектится use case'ом из application-слоя через UoW. Domain-слой не знает, какая БД используется.
- Конструктор сервиса принимает абстрактные репозитории по имени (`read_repository: TenantReadRepository`).
- Никаких глобальных синглтонов, никаких регистраций.

### Правила сигнатур и возврата

| Тип сервиса | Метод async? | Параметры | Возврат | Статичный? |
|---|---|---|---|---|
| Uniqueness | да | VO/full-сущности | `None` | нет (есть state) |
| Policy | нет | full-сущности | `None` | да (`@staticmethod`) |

**Общее:**
- Параметры передаём по имени.
- Никаких `Optional` параметров с дефолтами по типу `name: str | None = None`.
- Не возвращаем `bool` из `raise_*`, не возвращаем `tuple` со статусом — только `None`/целевой объект, либо доменная ошибка.
- Никаких `try/except` внутри сервиса.

### Сервис vs метод агрегата vs метод фабрики — итоговое правило

| Логика | Куда |
|---|---|
| Меняет состояние агрегата | Метод агрегата (`new_X`, `activate`) |
| Валидирует инвариант одного агрегата против переданных данных | Метод агрегата (`validate_categories`) |
| Проверяет уникальность через БД | Uniqueness service |
| Проверяет правило между двумя агрегатами без БД | Policy service |
| Собирает агрегат из VO | Фабрика (`new` / `restore`) |

### Анти-паттерны (services)

❌ **Mixed-сервис с несколькими несвязанными правилами.** Каждое самостоятельное бизнес-правило — отдельный сервис.

❌ **Сервис, который создаёт или сохраняет агрегат.** Сервис только проверяет конкретное бизнес-правило.

❌ **Сервис с `__init__`, принимающий полный UoW или несколько репозиториев из application-слоя.** Сервис зависит от **узких** read-интерфейсов из `domain/<aggregate>/repository.py`.

❌ **Sync метод там, где нужен async.** Если сервис ходит в БД — `async`.

❌ **Возврат `bool` или статуса из `raise_*` или `validate_*`.** Контракт — либо `None`, либо доменная ошибка.

❌ **Импорт `application/`, `infrastructure/`, `presentation/` в сервисе.**

❌ **Логика самого агрегата, вынесенная в сервис.** Если правило затрагивает только один агрегат и его поля — это метод этого агрегата, не сервис.

❌ **Catch-and-rethrow доменных ошибок в сервисе.** Если VO бросил `ValueObjectInvalidDataError`, сервис не должен ловить и переоборачивать.

---

## Repository Interfaces in Domain

### Назначение и расположение

Repository-интерфейс в domain-слое — это **узкий** контракт для domain-сервисов. Он содержит **только** те методы чтения, которые нужны конкретным сервисам этого агрегата/проекции.

Расположение: `domain/<aggregate>/repository.py` или `domain/<projection>/repository.py`. Один файл на агрегат, в нём — один или несколько ABC-интерфейсов.

**Файл создаётся только для проверки бизнес-правила, которому требуется чтение.** Без такого правила repository-интерфейс в domain не создаётся.

### Структура файла и класса

```python
from abc import ABC, abstractmethod

from domain.tenant.entity import Tenant
from domain.tenant.value_objects import TenantID


class TenantReadRepository(ABC):
    @abstractmethod
    async def by_id(self, tenant_id: TenantID) -> Tenant | None: ...
```

**Принципы:**

- **Базовый класс — `ABC`.** Не Protocol — потому что в проектах с DDD и явной инверсией зависимостей принято явное наследование, а не структурная типизация.
- **Имя класса — `<Aggregate>ReadRepository`** (или `<Projection>ReadRepository`). Без `Interface`, `Abstract`, `Protocol`-суффиксов.
- **Все методы — `@abstractmethod`.** В domain-репозитории не бывает методов с реализацией.
- **Тело метода — `...`** Никаких `pass`, `raise NotImplementedError`, docstring-ов.
- **Все методы — `async`** — реализация будет I/O-bound.

### Соглашения о методах

#### Имя метода

Префикс **`by_<criteria>`** для методов поиска:
- `by_id(...)` — поиск одной записи по идентификатору.
- `by_<field>(...)` — поиск по другому полю.
- `by_<field1>_<field2>(...)` — поиск по комбинации.

```python
async def by_id(self, tenant_id: TenantID) -> Tenant | None: ...
async def by_owner_id_name(
    self,
    owner_id: TenantID,
    name: TransactionCategoryName,
) -> TransactionCategory | None: ...
```

Никаких `find_*`, `get_*`, `fetch_*`, `lookup_*` — единый префикс `by_`.

#### Параметры

- **Только domain VO** — никогда сырые `UUID`, `str`, `int`.
- Передаются позиционно или по имени.
- Если поиск принимает несколько критериев — каждый отдельным VO-параметром, не группирующим dataclass.

#### Возврат

- **Один результат** → `<Aggregate> | None`. `None` означает «не найдено» — это нормальное доменное состояние, не ошибка. Никаких исключений типа `NotFoundError` из repository.
- **Никогда не возвращаем** сырые строки, dict, DTO или ORM-объекты. Только доменный агрегат/проекция или `None`.
- **Список результатов** → `list[<Aggregate>]`. Пустой список означает «ничего не найдено», не `None`.

### Граница repository-интерфейса

Добавляй только методы чтения, непосредственно необходимые сервису для проверки бизнес-правила. Не добавляй сохранение, генерацию ID, транзакции, пагинацию или универсальные выборки.

### Несколько интерфейсов в одном файле

Допустимо, если у агрегата есть несколько разнородных read-задач:

```python
class TenantReadRepository(ABC):
    @abstractmethod
    async def by_id(self, tenant_id: TenantID) -> Tenant | None: ...


class TenantSubscriptionReadRepository(ABC):
    @abstractmethod
    async def unprocessed(self) -> list[Tenant]: ...
```

### Чек-лист добавления repo-интерфейса

1. Сервису, использующему этот метод, действительно нужен **новый** метод? Если поиск повторяется — переиспользуй существующий.
2. Имя — `by_<criteria>`.
3. Параметры — VO, не примитивы.
4. Возврат — domain-объект или `None` (или `list[...]`).
5. Метод `async`, тело `...`.
6. Метод объявлен в `ABC`-классе с `@abstractmethod`.

### Анти-паттерны (repositories)

❌ **Тело `pass` или `raise NotImplementedError`.** Всегда `...`.

❌ **Параметры-примитивы.** `async def by_id(self, tenant_id: UUID)` — нет, только `TenantID`.

❌ **Возврат raw-данных.** `dict[str, Any]`, ORM-объект, tuple — нет. Только domain-агрегат/проекция.

❌ **Бросать `NotFoundError` или `EntityNotFoundError` вместо возврата `None`.** Отсутствие записи — это состояние, доменная ошибка не нужна.

❌ **Write-методы (`save`, `delete`, `update`) в domain-репозитории.** Только read.

❌ **Импорт реализации.** Domain-репозиторий импортирует только domain. Никаких `psycopg`, `sqlalchemy`, `asyncpg`.

❌ **Неабстрактные helpers/методы в ABC.** ABC должен быть чистым контрактом.

❌ **Методы, не используемые проверкой бизнес-правила.** Контракт должен оставаться узким.

❌ **Доменный repo для агрегата без domain-сервиса.** Если ни одному domain-сервису он не нужен — файла `domain/<aggregate>/repository.py` не должно быть.
