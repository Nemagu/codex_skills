---
name: python-ddd-domain-layer-writing
description: Используй при реализации или правке доменного слоя в Python-проекте по DDD/гексагональной архитектуре. Триггеры — добавление или изменение агрегата, проекции, объекта-значения, доменного сервиса, фабрики, абстрактного repository-интерфейса в `domain/`; реализация заданных инвариантов, поведения, версионирования, доменных ошибок и состояний. Не применять для определения требований и слоёв `application/`, `infrastructure/`, `presentation/`.
---

# Python DDD Domain Layer Writing

Общие правила оформления Python-кода брать из `$python-code-style-writing` и не
дублировать здесь; этот скил определяет только контракты domain-слоя.

## Quick Start

1. Выдели из задачи точный состав агрегатов, сущностей, проекций, полей, операций, состояний, инвариантов и отказов. Не дополняй его типовыми элементами.
2. Создай для каждого заданного поля отдельный неизменяемый объект-значение. Внутри доменной модели не используй сырые примитивы.
3. Создай поддиректорию `domain/<name>/` и последовательно: `value_object.py` → `aggregate.py` (или `projection.py`) → обязательный `factory.py` → только требуемые `repository.py` и `service.py`.
4. Разреши создание и восстановление агрегата или проекции только через фабрику. `new()` и `restore()` принимают готовые VO; идентификатор всегда передаёт вызывающий слой.
5. Реализуй только заданные состояния, переходы, инварианты, поведение, семантику повторов и проверки версий.
6. После всех проверок выполни мутацию агрегата и увеличь версию ровно один раз. При отказе состояние и версия не меняются.
7. Проекция принимает заданную версию извне и самостоятельно её не увеличивает. Не добавляй ID-parity без явного требования.
8. Различимые доменные бизнес-отказы представь отдельными типами.
9. Не создавай коллекцию доменных событий: тип сохранённого изменения, version storage, outbox и публикация принадлежат внешним слоям.
10. Обнови витрину `__init__.py` поддиректории.
11. Внутри `domain/` запрещены импорты из `application/`, `infrastructure/`, `presentation/` и внешних библиотек.
12. Все приватные поля начинаются с `_` и экспонируются без возможности внешней мутации.
13. Реализуй однозначно заданные части; не придумывай недостающую бизнес-семантику для остальных.

## When to Apply

### Триггеры активации

**По расположению файлов** — любая правка/создание в `domain/`:
- `domain/<aggregate>/aggregate.py`, `domain/<aggregate>/factory.py`, `domain/<aggregate>/service.py`, `domain/<aggregate>/repository.py`, `domain/<aggregate>/value_object.py`
- `domain/<projection>/projection.py` и связанные
- Общие файлы: `domain/value_object.py`, `domain/error.py`, `domain/aggregate.py`, `domain/projection.py`

**По концепциям, упомянутым пользователем:**
- DDD-термины: «агрегат», «aggregate», «value object», «VO», «доменная сущность», «entity», «проекция», «projection», «доменный сервис», «domain service», «фабрика», «factory», «репозиторий-интерфейс».
- Бизнес-инварианты, инкапсуляция бизнес-логики, неизменяемость состояния.
- Оптимистичный параллелизм / версионирование агрегата.
- Управление состоянием сущности (`active`/`deleted`/`frozen`).
- Read-модели, проекции из других bounded context.

**По типам задач:**
- Добавление нового агрегата или проекции.
- Добавление/изменение VO.
- Добавление/изменение метода поведения агрегата (нового мутатора).
- Добавление доменного сервиса для проверки конкретного бизнес-правила.
- Добавление абстрактного repository-интерфейса в domain.
- Расширение иерархии доменных ошибок.

### Анти-триггеры

- Правка `application/` (use cases, commands, queries, DTO, ports).
- Правка `infrastructure/` (psycopg-репозитории, NATS-адаптеры, миграции).
- Правка `presentation/` (FastAPI routers, background workers).
- Импорт-рефакторинг, переименование без изменения структуры.

### Предусловия

- Проект следует гексагональной архитектуре с выделенным `domain/`-слоем.
- В `domain/` запрещены импорты из `application/`, `infrastructure/`, `presentation/`. Допустимы только импорты внутри `domain/` и из стандартной библиотеки.
- Состав и семантика доменной модели заданы входными требованиями. Скил выбирает
  способ реализации на Python, но не проектирует новые бизнес-контракты.

## Package Structure

```
domain/
├── __init__.py                  ← пустой, не делаем re-export верхнего уровня
│
├── value_object.py             ← общие VO (Version, DomainObjectName, State)
├── error.py                    ← иерархия доменных ошибок
├── aggregate.py                  ← базовые AggregateRoot / AggregateRootWithState
├── projection.py               ← базовые Projection / ProjectionWithState
│
├── <aggregate_name>/            ← поддиректория на агрегат
│   ├── __init__.py              ← re-export публичного API через __all__
│   ├── aggregate.py
│   ├── value_object.py
│   ├── factory.py
│   ├── repository.py            ← только для требуемого доменного поиска
│   └── service.py               ← только для заданного межобъектного правила
│
└── <projection_name>/           ← поддиректория на проекцию
    ├── __init__.py
    ├── projection.py            ← вместо aggregate.py!
    ├── value_object.py
    ├── factory.py
    ├── repository.py            ← только для требуемого доменного поиска
    └── service.py               ← только для заданного межобъектного правила
```

### Содержимое верхнеуровневых файлов

**`domain/value_object.py`** — VO, разделяемые между агрегатами/проекциями. Правило: VO попадает сюда, **только если используется ≥ 2 разными модулями ИЛИ базовыми классами**. Иначе — в `domain/<name>/value_object.py`.

**`domain/error.py`** — единая публичная иерархия доменных ошибок, включая
специализированные типы различимых бизнес-исходов.

**`domain/aggregate.py`** — базовые `AggregateRoot` и `AggregateRootWithState`. Других классов в этом файле быть не должно.

**`domain/projection.py`** — базовые `Projection` и `ProjectionWithState`.

### Реализация заданного типа

**Агрегат:**
- Реализуй заданную границу и только перечисленное поведение.
- Изменяй принадлежащие объекты только через корень агрегата.
- После успешной изменяющей операции увеличивай версию ровно один раз.

**Проекция:**
- Реализуй ровно перечисленные поля и операции локального представления.
- Принимай новую версию извне и не инкрементируй её внутри domain.
- Не принимай transport-модели или application DTO.
- В файле используй `projection.py` и базовый `Projection` / `ProjectionWithState`.

### Правила импортов внутри `domain/`

**Разрешено:**
- Импорт из стандартной библиотеки (`uuid`, `decimal`, `datetime`, `enum`, `dataclasses`, `typing`, `abc`).
- Импорт между модулями `domain/` (например, `from domain.tenant import TenantID` в `personal_transaction/aggregate.py`).
- Импорт из общих файлов (`domain.value_object`, `domain.error`, `domain.aggregate`, `domain.projection`).

**Запрещено:**
- Импорты из `application/`, `infrastructure/`, `presentation/`.
- Импорты внешних библиотек (никаких `pydantic`, `psycopg`, `fastapi`).
- Циклические импорты между поддиректориями. При риске цикла — выносим общий VO в `domain/value_object.py`.

### Именование

- Модуль агрегата/проекции — **snake_case в единственном числе** (`personal_transaction`, не `personal_transactions`).
- Класс агрегата — **PascalCase, существительное в единственном числе** (`PersonalTransaction`, `Tenant`).
- ID-VO — **`<Aggregate>ID`** (`PersonalTransactionID`, не `PersonalTransactionId`).
- Фабрика — **`<Aggregate>Factory`**.
- Repository-интерфейс — **`<Aggregate>ReadRepository`**.
- Сервис именуется по конкретному бизнес-правилу: `UserUniquenessService`, `TransactionOwnershipPolicy`.

## Error Hierarchy

### Полная иерархия

```
DomainError (msg, subject, data)
├── ValueObjectError
│   └── ValueObjectInvalidDataError
└── EntityError
    ├── EntityInvalidDataError
    ├── EntityVersionError
    ├── EntityIdempotentError
    ├── EntityPolicyError
    ├── EntityAlreadyExistsError
    └── EntityNotFoundError
```

Вся публичная иерархия живёт в `domain/error.py`. Для каждого бизнес-исхода,
который вызывающий слой должен отличать от остальных, создавай отдельный тип,
наследуя его от подходящей общей категории. Не создавай разные типы для внутренних
проверок с одинаковым наблюдаемым смыслом.

### Базовый класс

```python
from typing import Any


class DomainError(Exception):
    def __init__(
        self,
        msg: str,
        subject: str,
        data: dict[str, Any] | None = None,
    ) -> None:
        super().__init__(msg)
        self.msg = msg
        self.subject = subject
        self.data = data or {}

    def __repr__(self) -> str:
        return (
            f"{self.__class__.__name__}("
            f"msg={self.msg!r}, subject={self.subject!r}, data={self.data!r})"
        )


class ValueObjectError(DomainError):
    pass


class ValueObjectInvalidDataError(ValueObjectError):
    pass


class EntityError(DomainError):
    pass


class EntityInvalidDataError(EntityError):
    pass


class EntityVersionError(EntityError):
    pass


class EntityIdempotentError(EntityError):
    pass


class EntityPolicyError(EntityError):
    pass


class EntityAlreadyExistsError(EntityError):
    pass


class EntityNotFoundError(EntityError):
    pass
```

### Поля

- **`msg`** — сообщение об ошибке на языке домена. Пример: `"новое состояние идентично текущему"`, `"арендатор удален"`, `"только владелец может работать с категорией"`.
- **`subject`** — человекочитаемая метка предмета ошибки на языке домена. Это **не** имя класса/модуля, а название агрегата/VO/проекции в терминах бизнеса. Для агрегатов и проекций берётся из общего `domain_object_name.name`, для остальных VO задаётся явно. Примеры: `"арендатор"`, `"категория транзакций"`, `"проекция пользователя"`, `"название категории"`, `"версия агрегата"`.
- **`data`** — безопасный структурированный контекст ошибки для вызывающего слоя.
  Опционально, по умолчанию `{}`. Domain не логирует ошибки и не определяет их
  transport-представление.

### Конвенция формирования `data`

**Ключи верхнего уровня — английские, в snake_case, отражают тип сущности:**

| Тип ошибки | Ключ верхнего уровня | Значение |
|---|---|---|
| Ошибка одного агрегата | имя сущности в ед. ч.: `"tenant"`, `"transaction"`, `"category"`, `"user"` | `dict` с полями этой сущности |
| Ошибка с участием нескольких сущностей одного типа | имя во мн. ч.: `"categories"`, `"transactions"` | `list[dict]` |
| Ошибка VO | имя поля VO: `"version"`, `"name"`, `"transaction_type"` | примитив или `dict` |

**Внутри блока сущности — поля в snake_case (английские), значения — примитивы для JSON:**

```python
# Ошибка одного агрегата
{"tenant": {"tenant_id": UUID(...), "state": "active"}}

# Ошибка с двумя агрегатами разных типов
{
    "tenant": {"tenant_id": UUID(...)},
    "transaction": {"owner_id": UUID(...)},
}

# Ошибка с коллекцией
{
    "transaction": {"transaction_id": UUID(...)},
    "categories": [
        {"category_id": UUID(...), "name": "еда"},
        {"category_id": UUID(...), "name": "транспорт"},
    ],
}

# Ошибка VO
{"version": 0}
{"transaction_type": "wrong_value"}
{"money_amount": {"amount": "-10.50", "currency": "ruble"}}
```

**Правила значений:**
- `UUID` оставляем как `UUID`-объект, не строкой — сериализатор API сам приведёт.
- `Decimal` приводим к `str` (`str(amount)`), чтобы избежать потери точности при JSON-сериализации.
- `datetime` оставляем как есть, сериализатор API приведёт.
- `Enum` берём `.value` (строку), не сам enum.

### Семантика подклассов

| Класс | Когда бросать |
|---|---|
| `ValueObjectInvalidDataError` | VO получил невалидные данные при конструировании. В `__post_init__` или `from_str`. |
| `EntityInvalidDataError` | Операция над сущностью невозможна из-за её состояния или входных данных, не подпадающих под идемпотентность/политику. Типичные случаи: попытка изменить удалённую сущность (через `_check_state`), пустые данные на входе, несовместимые входные данные. |
| `EntityVersionError` | Нарушение контракта версии сущности или проекции, включая получение устаревшей версии. |
| `EntityIdempotentError` | Повторный вызов должен завершиться доменным отказом согласно заданной семантике. Не используй его автоматически для каждого равенства значений. |
| `EntityPolicyError` | Нарушение бизнес-политики, не связанное с состоянием самой сущности (только владелец может, только админ может, доступ запрещён из-за состояния субъекта). |
| `EntityAlreadyExistsError` | Бизнес-правило уникальности обнаружило уже существующую сущность. |
| `EntityNotFoundError` | Бизнес-правило требует существования связанной сущности, но она не найдена. |

### Правила вызова

- **`data` пропускаем, если её нет** — не передавать `data={}` явно.
- **В агрегатах используем `_error_data` хелпер** — он подставляет `subject` из `domain_object_name.name` и добавляет ID агрегата в `data`: `raise EntityIdempotentError(**self._error_data(msg="...", data={...}))`.
- **В VO и в сервисах** заполняем поля явно (хелпера нет).

## Public API of a Module (`__init__.py` and `__all__`)

### Что попадает в витрину

| Категория | Примеры |
|---|---|
| Класс агрегата / проекции | `Tenant`, `User` |
| Фабрика | `TenantFactory`, `UserFactory` |
| Все VO агрегата/проекции | `TenantID`, `TenantState`, `TenantStatus` |
| Repository-интерфейс (если есть) | `TenantReadRepository` |
| Domain-сервисы (если есть) | `UserUniquenessService`, `TransactionOwnershipPolicy` |

**Не попадает:**
- Приватные хелперы (имя с `_`).
- Базовые классы из общих файлов (`AggregateRoot`, `Projection`, `DomainError`) — импортируются напрямую.

### Шаблон `__init__.py`

```python
from domain.<aggregate>.entity import <Aggregate>
from domain.<aggregate>.factory import <Aggregate>Factory
from domain.<aggregate>.repository import <Aggregate>ReadRepository
from domain.<aggregate>.service import <Aggregate>UniquenessService
from domain.<aggregate>.value_object import (
    <Aggregate>ID,
    <Aggregate>Name,
    <Aggregate>State,
)

__all__ = [
    "<Aggregate>",
    "<Aggregate>Factory",
    "<Aggregate>ID",
    "<Aggregate>Name",
    "<Aggregate>ReadRepository",
    "<Aggregate>State",
    "<Aggregate>UniquenessService",
]
```

**`__all__` строго в алфавитном порядке.**

### Top-level `domain/__init__.py` — пустой

Не делаем re-export всех агрегатов на верхний уровень. Импорты на стороне идут через поддиректории.

### Правила импорта

| Где | Откуда импортируем |
|---|---|
| Внутри `domain/<aggregate>/*.py` (тот же модуль) | Прямые пути: `from domain.<aggregate>.entity import ...` |
| Из `domain/<other_aggregate>/*.py` | Витрина: `from domain.<other_aggregate> import ...` |
| Из `application/`, `infrastructure/`, `presentation/` | Витрина: `from domain.<aggregate> import ...` |
| Общие файлы (`domain/aggregate.py`, `domain/error.py`, ...) | Прямые пути: `from domain.error import ...` |

## References

- **`references/value_objects.md`** — три типа VO (Identity, Validated, Enum) + Multi-field, правила иммутабельности, нормализация, валидация.
- **`references/aggregates.md`** — базовые классы `AggregateRoot` / `AggregateRootWithState`, паттерны агрегатов, фабрики агрегатов, особый случай расширенного состояния.
- **`references/projections.md`** — базовые классы `Projection` /
  `ProjectionWithState`, операции, фабрики и правила входящей версии.
- **`references/services_and_repositories.md`** — сервисы конкретных бизнес-правил и необходимые им repository-интерфейсы.
- **`references/testing.md`** — unit-тесты domain-инвариантов, версии, фабрик, fixtures и покрытие.
- **`references/checklists.md`** — пошаговые чек-листы добавления агрегата и проекции.
