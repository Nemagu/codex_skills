# Projections

## Когда заводить проекцию

**Проекция — компактное локальное представление сущности из другого BC.** Источник правды для проекции находится снаружи (в другом сервисе, BC, источнике событий).

Проекция используется для:

1. **Создания агрегатов на её основе.** ID проекции совпадает с ID будущего агрегата (см. ID-parity ниже). Это инвариант «один локальный агрегат на одну внешнюю сущность».
2. **Синхронизации связанных агрегатов.** Когда из источника правды приходит событие, проекция применяет его, и, если в этом сервисе есть связанный агрегат, ему могут быть применены соответствующие изменения.
3. **Валидации в domain-сервисах.** Например, Uniqueness service проверяет существование внешней сущности через read-репозиторий проекции.

**Когда выбрать проекцию вместо агрегата:**
- Источник правды — снаружи.
- В этом сервисе сущность нужна с минимальным набором полей.

**Проекция остаётся проекцией навсегда** — она не «дорастает» до агрегата. Если в текущем сервисе понадобится локальное поведение поверх внешней сущности — добавляется отдельный сосуществующий агрегат с тем же ID, сохраняя проекцию как канал синхронизации (см. ниже).

## Принципиальные отличия от агрегата

| Аспект | Агрегат (`Entity`) | Проекция (`Projection`) |
|---|---|---|
| Источник правды | Сам, в текущем BC | Внешний BC |
| Версия | Инкрементит сам через `_update_version` | Получает в `new_version(version)` извне |
| `original_version` | Есть — для оптимистичной блокировки в репо | Нет — проекция перезаписывается атомарно |
| `mark_persisted` | Есть — вызывает репо после save | Нет — версия применилась = состояние записано |
| State-методы | `_update_version()` после мутации | Версию **не** трогают |
| `_check_state` | Защищает мутирующие методы от удалённого состояния | Нет — события источника правды всегда применяются |
| Откат версии назад | Невозможен | Защищён `EntityVersionError` |
| `_error_data` контракт | `subject = aggregate_name.name` | `subject = projection_name.name` |
| Имя метки | `AggregateName` | `ProjectionName` |

## Базовые классы `Projection` и `ProjectionWithState`

### Реализация

```python
from abc import ABC, abstractmethod
from typing import Any

from domain.error import (
    EntityIdempotentError,
    EntityVersionError,
)
from domain.value_object import ProjectionName, State, Version


class Projection(ABC):
    def __init__(
        self,
        version: Version,
        projection_name: ProjectionName,
    ) -> None:
        self._version = version
        self._projection_name = projection_name

    @property
    def version(self) -> Version:
        return self._version

    @property
    def projection_name(self) -> ProjectionName:
        return self._projection_name

    def new_version(self, version: Version) -> None:
        if self._version == version:
            raise EntityIdempotentError(
                **self._error_data(
                    msg="новая версия идентична текущей",
                    data={"version": version.version},
                )
            )
        if self._version.version > version.version:
            raise EntityVersionError(
                **self._error_data(
                    msg="новая версия меньше текущей",
                    data={
                        "new_version": version.version,
                        "current_version": self._version.version,
                    },
                )
            )
        self._version = version

    @abstractmethod
    def _error_data(
        self,
        msg: str,
        data: dict[str, Any] | None = None,
    ) -> dict[str, Any]: ...

    def __repr__(self) -> str:
        fields = ", ".join(f"{k}={v!r}" for k, v in vars(self).items())
        return f"{self.__class__.__name__}({fields})"


class ProjectionWithState(Projection):
    def __init__(
        self,
        state: State,
        version: Version,
        projection_name: ProjectionName,
    ) -> None:
        super().__init__(version=version, projection_name=projection_name)
        self._state = state

    @property
    def state(self) -> State:
        return self._state

    def new_state(self, state: State) -> None:
        if self._state == state:
            raise EntityIdempotentError(
                **self._error_data(
                    msg="новое состояние идентично текущему",
                    data={"state": state.value},
                )
            )
        self._state = state

    def activate(self) -> None:
        if self._state.is_active():
            raise EntityIdempotentError(
                **self._error_data(
                    msg=f"{self._projection_name.name} уже активно",
                    data={"state": self._state.value},
                )
            )
        self._state = State.ACTIVE

    def delete(self) -> None:
        if self._state.is_deleted():
            raise EntityIdempotentError(
                **self._error_data(
                    msg=f"{self._projection_name.name} уже удалено",
                    data={"state": self._state.value},
                )
            )
        self._state = State.DELETED
```

### Версионирование проекции

**Контракт `new_version(version)`:**

| Условие | Поведение |
|---|---|
| `version == self._version` | `EntityIdempotentError` — повторное событие, no-op |
| `version < self._version` | `EntityVersionError` — пришло устаревшее событие, отбрасываем |
| `version > self._version` | Применяем |

**Зачем regression-проверка:**
События источника правды могут приходить не по порядку (брокер, ребалансировка консьюмеров). Если уже применили версию 7 и пришла 5 — нельзя откатывать состояние.

**Зачем idempotency-проверка:**
Системы доставки событий обычно гарантируют at-least-once — одно и то же событие может прийти дважды.

### Почему state-методы не инкрементят version

В агрегате `_update_version()` нужен, чтобы репозиторий понял: «эту запись изменили, при сохранении используй новую версию». В проекции версия не локальная — она приходит вместе с событием. Если бы `new_state` инкрементил версию локально, локальный счётчик разошёлся бы с источником правды. Поэтому в проекциях state-методы версию **не** трогают.

### Почему нет `_check_state` в проекции

В агрегате `_check_state` запрещает мутации над удалённой сущностью. В проекции «мутация» это применение события из источника правды, и эти события **должны применяться независимо от текущего состояния проекции**. Если бы `_check_state` блокировал применение события на удалённой проекции, мы бы поломали синхронизацию с источником правды.

Это **не означает**, что проекция не участвует в бизнес-логике. Domain-сервисы используют состояние проекции для проверок; use case'ы могут отказывать в командах, если проекция в неподходящем состоянии. Проверки доступа на уровне команд — в use case, не в проекции.

## Класс проекции

### Структура файла

- Один класс на файл — `domain/<projection_name>/projection.py` (**не `entity.py`** — имя файла сигнализирует о роли).
- Класс наследует `ProjectionWithState` (стандартное двухсостояние) или `Projection` напрямую (расширенное состояние).
- Имя класса — существительное в единственном числе, без суффикса `Projection` (`User`, не `UserProjection`).

### Конструктор

```python
class User(ProjectionWithState):
    def __init__(
        self,
        user_id: UserID,
        state: State,
        version: Version,
    ) -> None:
        super().__init__(
            state=state,
            version=version,
            projection_name=ProjectionName("проекция пользователя"),
        )
        self._user_id = user_id
```

Принципы те же, что для агрегата: только VO в параметрах, приватные поля с `_`, `super().__init__` первым шагом. Отличие — передаём `projection_name`, не `aggregate_name`.

### Properties

Идентично агрегату — все поля наружу через `@property` без сеттера, имя без `_`, возвращаем VO.

### Методы применения событий

Каждое внешнее событие — отдельный метод. Без `_check_state`, без `_update_version`. Структура:

```python
def new_state(self, state: UserState) -> None:
    if self._state == state:                                 # 1. идемпотентность
        raise EntityIdempotentError(
            **self._error_data(
                msg="новое состояние пользователя идентично текущему",
                data={"state": state.value},
            )
        )
    self._state = state                                       # 2. мутация
```

**Отличия от метода-мутатора агрегата:**

| Шаг | Агрегат | Проекция |
|---|---|---|
| 1 | `_check_state()` | пропускаем — события источника правды применяются всегда |
| 2 | проверка идемпотентности | проверка идемпотентности |
| 3 | мутация | мутация |
| 4 | `_update_version()` | пропускаем — версия применится отдельно через `new_version` |

### Применение события на стороне use case

```python
user = await uow.user_repo.by_id(UserID(event.user_id))
user.new_state(UserState.FROZEN)              # меняем состояние
user.new_version(Version(event.version))       # применяем версию из события
await uow.user_repo.save(user)
```

Use case делает два вызова: сначала state, потом version. **Никогда** не делает обратное.

`new_version` возьмёт на себя:
- Если версия равна текущей — `EntityIdempotentError` (повтор события).
- Если меньше — `EntityVersionError` (устаревшее событие).
- Если больше — применит.

Вызывающий код должен решать, как обработать каждый случай: `EntityIdempotentError` обычно — log-and-skip, `EntityVersionError` — log-and-skip с алертом, успех — продолжить.

### Реализация `_error_data`

```python
def _error_data(
    self,
    msg: str,
    data: dict[str, Any] | None = None,
) -> dict[str, Any]:
    data = data or {}
    data["user_id"] = self._user_id.user_id
    return {
        "msg": msg,
        "subject": self._projection_name.name,
        "data": {"user": data},
    }
```

Конвенция формирования `data` — общая (английский ключ-сущность на верхнем уровне, snake_case полей, примитивы внутри). Отличие — `subject` берётся из `projection_name`, не `aggregate_name`.

## Особый случай — расширенное состояние

Если у проекции состояний больше двух (например, `User` имеет `ACTIVE` / `FROZEN` / `DELETED`):

- Создаём свой enum в `domain/<projection>/value_object.py` (`UserState`).
- Класс наследует **`Projection`**, не `ProjectionWithState`.
- Реализуем `new_state(state)` сами — с проверкой идемпотентности, **без** `_update_version`.
- Если бизнесу нужны узкие методы (`freeze()`, `activate()`, `delete()`) — добавляем по необходимости. Для проекций такие узкие методы часто избыточны: достаточно `new_state(state)`.
- `_DELETED_MSG` / `_FROZEN_MSG` **не нужны** — `_check_state` в проекции отсутствует.

```python
class User(Projection):
    def __init__(
        self,
        user_id: UserID,
        state: UserState,
        version: Version,
    ) -> None:
        super().__init__(
            version=version,
            projection_name=ProjectionName("проекция пользователя"),
        )
        self._user_id = user_id
        self._state = state

    @property
    def user_id(self) -> UserID:
        return self._user_id

    @property
    def state(self) -> UserState:
        return self._state

    def new_state(self, state: UserState) -> None:
        if self._state == state:
            raise EntityIdempotentError(
                **self._error_data(
                    msg="новое состояние пользователя идентично текущему",
                    data={"state": self._state.value},
                )
            )
        self._state = state

    def _error_data(
        self,
        msg: str,
        data: dict[str, Any] | None = None,
    ) -> dict[str, Any]:
        data = data or {}
        data["user_id"] = self._user_id.user_id
        return {
            "msg": msg,
            "subject": self._projection_name.name,
            "data": {"user": data},
        }
```

## Соглашение об ID-parity

**Если проекция представляет сущность, на основе которой в этом сервисе создаётся агрегат, ИЛИ если в этом сервисе появляется сосуществующий агрегат с локальным поведением поверх той же внешней сущности — ID проекции и ID агрегата(ов) совпадают.**

```python
# domain/user/value_object.py
@dataclass(frozen=True)
class UserID:
    user_id: UUID

# domain/tenant/value_object.py
@dataclass(frozen=True)
class TenantID:
    tenant_id: UUID

# Проверка правила и создание остаются отдельными операциями.
tenant_id = TenantID(user.user_id.user_id)  # ID parity
await uniqueness_service.validate_tenant_id(tenant_id=tenant_id)
tenant = TenantFactory.new(tenant_id=tenant_id)
```

**Зачем:**

- **Дешёвая эволюция модели.** Если в этом сервисе понадобятся локальные поля поверх внешней сущности, мы добавляем сосуществующий агрегат с тем же `UUID` — без миграций FK, без переписывания событий.
- **Прямой lookup связанного агрегата** при обработке события проекции (без отдельного маппинга).
- **Прозрачность инварианта 1:1** между внешней сущностью и её локальным представлением.

**Когда не применять:** если связь между внешней сущностью и локальной модели — не один-к-одному. Например, один внешний user может владеть многими локальными счетами — у счёта свой ID, `user_id` хранится отдельным полем.

## Сосуществующий агрегат и проекция

Когда в этом сервисе появляются локальные поля/поведение поверх внешней сущности — заводится **отдельный** агрегат **рядом** с проекцией, с тем же ID. Проекция и агрегат сосуществуют и играют разные роли:

| Аспект | Проекция | Сосуществующий агрегат |
|---|---|---|
| Источник правды | Внешний сервис | Текущий сервис (для своих полей) |
| Что хранит | Поля внешней сущности (status, state, ...) | Локальные поля (display_name, settings, ...) |
| Кто меняет | События из внешнего сервиса | Команды текущего сервиса + реакции на события проекции |
| ID | Совпадает с внешним | Совпадает с проекцией (= внешним) |

### Реакция агрегатов на изменение проекции

Domain-слой предоставляет необходимые операции на проекции (`new_state`, `new_version`) и на связанных агрегатах (`freeze`, `delete`, `new_state` и т.п.). **Сам механизм синхронизации** — кто, когда и в какой транзакции вызывает эти методы — лежит в application и presentation слоях.

Типовая схема (вне domain-слоя):
- Один воркер консумит события из брокера и применяет их к проекциям.
- Другой воркер опрашивает «необработанные» проекции и применяет соответствующие изменения к связанным агрегатам в отдельных транзакциях.

Domain-скил **не диктует** этот pipeline — он только даёт строительные блоки. Принцип «одна транзакция = один агрегат» соблюдается на уровне use case'ов и компоновки воркеров.

## Фабрика проекции

В отличие от агрегата, у проекции **нет семантической разницы между `new` и `restore`** — версия и состояние всегда приходят извне:
- При первом появлении сущности (событие «X создан») — берём из события.
- При чтении из локальной БД — берём из таблицы.

Сигнатура одинаковая, реализация одинаковая. Поэтому в фабрике проекции — **один метод**, без дублирования:

```python
class UserFactory:
    @staticmethod
    def restore(user_id: UserID, state: UserState, version: Version) -> User:
        return User(
            user_id=user_id,
            state=state,
            version=version,
        )
```

**Имя метода:** `restore` — потому что восстанавливается известное состояние. Все параметры — готовые VO; raw-данные преобразуются до вызова фабрики.

**Вызывают:**
- Репозиторий проекции — при чтении из БД.
- Обработчик события — при создании проекции по событию «X создан». В обоих случаях фабрика получает готовые VO.

## Чек-лист добавления нового метода применения события

1. Имя — `new_<field>` или, если событие меняет конкретный аспект, узкий метод (`activate`, `delete`).
2. Параметры — VO.
3. Возврат — `None`, мутация in-place.
4. **Первая строка** — проверка идемпотентности через `==`.
5. **Вторая** — мутация.
6. **`_update_version` не вызывать.**
7. **`_check_state` не вызывать.**
8. Версия применяется отдельно через `new_version` в use case.

## Анти-паттерны

❌ **Использование `entity.py` как имени файла.** Только `projection.py` — это сигнал читателю о роли класса.

❌ **`_update_version` в state-методе проекции.** Локальный счётчик разойдётся с источником правды.

❌ **`_check_state` в проекции.** Проекция всегда применяет приходящие события. Бизнес-проверки доступа — в use case.

❌ **`mark_persisted` / `_original_version` в проекции.** Этих понятий у проекции нет.

❌ **Богатые методы поведения в проекции.** Если хочется добавить `transfer_to`, `recalculate_balance` — это попытка сделать из проекции агрегат. Источник правды снаружи; такая логика должна жить либо в исходном BC, либо в сосуществующем агрегате.

❌ **Применение события без `new_version`.** Если use case вызвал `new_state` и не вызвал `new_version` — версия проекции не сдвинется, при следующем приходе того же события состояние применится повторно или будет выглядеть как regression.

❌ **Слияние state и version в один метод.** `apply_event(state, version)` выглядит удобным, но прячет два разных режима ошибок (idempotent vs version-regression). Оставляем два явных вызова.

❌ **Дублирование тела `new`/`restore` в фабрике проекции.** Если методы идентичны — оставляем один (`restore`).

❌ **Параметры с `default` в фабрике проекции.** Все поля проекции приходят извне — обязательные.

❌ **Проекция, у которой нет ни одного использования в domain-логике.** Если проекция нужна только для read-запроса (присоединить имя пользователя в выдаче) — это не доменная проекция, а DTO в репозитории чтения.
