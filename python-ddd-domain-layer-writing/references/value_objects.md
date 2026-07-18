# Value Objects

## Общие требования ко всем VO

1. **Иммутабельность.** Всегда `@dataclass(frozen=True)` для dataclass-VO; `StrEnum` для enum-VO. Никогда не используем обычный класс с `__init__`.
2. **Только примитивы и другие VO внутри.** Никаких ссылок на агрегаты, репозитории, сервисы или внешние модели (Pydantic, ORM).
3. **Валидация — в `__post_init__`** (для dataclass) или в `from_str` (для enum). Никогда не в фабрике, use case'е или презентации.
4. **Ошибка валидации — всегда `ValueObjectInvalidDataError`**, с `msg` / `subject` / `data` через kwargs.
5. **`subject`** — человекочитаемая русская метка предмета: `"версия агрегата"`, `"количество средств"`, `"состояние арендатора"`. Не имя класса/поля.
6. **Имена полей внутри VO — snake_case на английском**: `tenant_id`, `transaction_id`, `name`, `amount`, `currency`. Это поля, которые попадут в `data` и в API.
7. **Никаких сеттеров и мутирующих методов.** Если нужна модификация — метод должен возвращать новый экземпляр (`def with_amount(self, amount: Decimal) -> Self`).

## Тип A — Identity VO

Типобезопасная обёртка над идентификатором (обычно `UUID`). Без валидации, потому что валидность UUID гарантирует тип.

```python
from dataclasses import dataclass
from uuid import UUID


@dataclass(frozen=True)
class TenantID:
    tenant_id: UUID
```

**Когда применять:** для каждого агрегата и проекции — обязательно. ID-VO нужен, чтобы агрегаты ссылались друг на друга через типизированные ID, а не через голые `UUID`.

**Соглашения:**
- Имя класса — `<Aggregate>ID` (`PersonalTransactionID`, `TransactionCategoryID`).
- Имя поля внутри — `<aggregate>_id` (`tenant_id`, `category_id`). Совпадает с именем колонки в БД.
- Тип поля — `UUID` (если в проекте используется UUID-стратегия).

**Чего не делать:** не добавлять `__post_init__` с проверкой `if not isinstance(...)` — это работа аннотации типа.

## Тип B — Validated VO

Однополевый VO с нормализацией и/или валидацией.

### B.1 — Нормализация + валидация

```python
from dataclasses import dataclass

from domain.error import ValueObjectInvalidDataError


@dataclass(frozen=True)
class PersonalTransactionName:
    name: str

    def __post_init__(self) -> None:
        object.__setattr__(self, "name", self.name.strip())
        if len(self.name) > 100:
            raise ValueObjectInvalidDataError(
                msg="название транзакции не может содержать более 100 символов",
                subject="название персональной транзакции",
                data={"name": self.name},
            )
```

**Ключевые моменты:**

- **Нормализация — первым шагом.** Через `object.__setattr__(self, "field", normalized_value)` — потому что `frozen=True` запрещает обычное присваивание. Типичная нормализация: `.strip()` для строк.
- **Валидация — после нормализации.** Иначе пробельная строка `"   "` пройдёт проверку «не пустое».
- **Все проверки в одном `__post_init__`** — последовательно, каждая бросает свой `ValueObjectInvalidDataError` со своим `msg`. Не объединяем несколько ошибок в одну.
- **`data` содержит уже нормализованное значение** — то, что реально стало полем.
- **`subject` — описание самого VO**, не агрегата, к которому он принадлежит. `"название персональной транзакции"`, не `"персональная транзакция"`.

### B.2 — Только нормализация

Если бизнес-правил нет, но нужна нормализация — всё равно создаём отдельный VO, не используем сырой `str`:

```python
@dataclass(frozen=True)
class PersonalTransactionDescription:
    description: str

    def __post_init__(self) -> None:
        object.__setattr__(self, "description", self.description.strip())
```

**Зачем не сырой `str`:** типизация на границах защищает от случайной передачи строки не в то поле; если завтра появится правило (например, лимит длины) — добавляется в одно место.

### B.3 — Без валидации и без нормализации

Если поле — типизированная обёртка над одним примитивом без правил:

```python
from datetime import datetime


@dataclass(frozen=True)
class PersonalTransactionTime:
    transaction_time: datetime
```

Это нормально и допустимо. Не нужно искусственно добавлять `__post_init__`.

## Тип C — Enum VO

Для значений из закрытого множества — статусов, типов, валют, состояний.

```python
from enum import StrEnum
from typing import Self

from domain.error import ValueObjectInvalidDataError


class PersonalTransactionType(StrEnum):
    EXPENSE = "expense"
    INCOME = "income"

    def is_expense(self) -> bool:
        return self == self.__class__.EXPENSE

    def is_income(self) -> bool:
        return self == self.__class__.INCOME

    @classmethod
    def from_str(cls, value: str) -> Self:
        lower_value = value.lower()
        if lower_value in cls._value2member_map_:
            return cls(lower_value)
        raise ValueObjectInvalidDataError(
            msg=f'не удалось найти тип транзакции по предоставленной строке - "{value}"',
            subject="тип персональной транзакции",
            data={"transaction_type": value},
        )
```

**Обязательные элементы:**

- **Базовый класс — `StrEnum`.** Не `Enum`, не `IntEnum`. Даёт автоматическую сериализацию в строку (для API, БД, логов) и работу `==` со строками.
- **Значения — английские, snake_case** (`"expense"`, `"income"`, `"deleted"`). Это то, что попадёт в БД и API.
- **Предикаты `is_X()`** добавляй только когда они реально используются и делают бизнес-условие понятнее. Не создавай их механически для каждого значения. Использование:
  ```python
  if transaction_type.is_expense(): ...   # правильно
  if transaction_type == PersonalTransactionType.EXPENSE: ...   # допустимо, но менее читаемо
  ```
- **`from_str` classmethod** — единая точка парсинга строк извне. Регистр-нечувствительный (`.lower()`). При невалидной строке — `ValueObjectInvalidDataError`.

**Имя предиката:** `is_<value>()`, всегда без отрицания. Не `is_not_deleted` — пиши `not state.is_deleted()`.

**Чего не делать:**
- Не добавлять `__post_init__` к enum.
- Не парсить enum через `cls(value)` снаружи — всегда через `from_str`, чтобы получить доменную ошибку, а не `ValueError`.

## Тип D — Multi-field VO

VO с несколькими полями, между которыми есть инвариант. Не путать с агрегатом — у multi-field VO нет идентификатора и нет жизненного цикла, он сравнивается по значению.

```python
from dataclasses import dataclass
from decimal import Decimal

from domain.error import ValueObjectInvalidDataError


@dataclass(frozen=True)
class MoneyAmount:
    amount: Decimal
    currency: Currency

    def __post_init__(self) -> None:
        if self.amount < 0:
            raise ValueObjectInvalidDataError(
                msg="количество средств не может быть менее 0",
                subject="количество средств",
                data={
                    "money_amount": {
                        "amount": str(self.amount),
                        "currency": self.currency.value,
                    }
                },
            )
```

**Особенности:**

- **Поля могут быть другими VO** (здесь `Currency` — enum-VO).
- **`data` — вложенный** под ключом `"money_amount"` — следует конвенции «ключ верхнего уровня = имя VO/сущности».
- **`Decimal` приводим к `str` в `data`** — для JSON-сериализации без потери точности.
- **`Enum` приводим к `.value`** — `currency.value`, не `currency`.

**Когда применять:** есть инвариант между полями (`amount` ≥ 0 при любой `currency`), либо поля всегда ходят парой (валюта без суммы бессмысленна).

## Размещение VO

| Где живёт | Когда |
|---|---|
| `domain/value_object.py` | используется ≥ 2 разными агрегатами/проекциями ИЛИ базовыми классами |
| `domain/<aggregate>/value_object.py` | используется только этим агрегатом |

Общими становятся только VO, у которых нет «домашнего» агрегата (например, `Version`, `AggregateName`, `State`).

## Анти-паттерны

❌ **Pydantic-модели как VO.** Pydantic — для границ (API, события). Внутри domain — только `dataclass(frozen=True)` и `StrEnum`.

❌ **Изменяемый `dict`/`list` как поле frozen-dataclass.** `frozen=True` защищает только от переприсваивания поля, но не от мутации содержимого. Если нужен набор — `frozenset`/`tuple`.

❌ **Сравнение enum со строкой в коде:**
```python
if state == "active":  # неправильно
if state.is_active():  # правильно
```

❌ **Валидация через ассерты.** `assert version >= 1` — нет, бросаем `ValueObjectInvalidDataError`. Ассерты могут быть выключены через `python -O`.

❌ **Логика, требующая I/O, в VO.** VO — чистая структура. Никаких обращений в БД/сеть/файлы из `__post_init__` или методов.

❌ **Возврат `None` из методов VO.** Если метод модифицирует значение — он возвращает новый VO, не `None` (потому что VO иммутабелен). Если метод проверяет — возвращает `bool` или бросает доменную ошибку.
