# Проекции

Реализуй ровно перечисленные во входном контракте проекции, поля, операции,
состояния, отказы и правила версий. Не решай внутри этого навыка, нужна ли
проекция, и не расширяй её типовыми полями или поведением.

Проекция получает заданные доменные сведения и версию извне. Преобразование
transport-модели или application DTO в VO выполняется до вызова domain.

## Принципиальные отличия от агрегата

| Аспект | Агрегат (`Entity`) | Проекция (`Projection`) |
|---|---|---|
| Источник правды | Сам, в текущем BC | Внешний BC |
| Версия | Инкрементит сам через `_update_version` | Получает в `new_version(version)` извне |
| `original_version` | Есть — для оптимистичной блокировки в репо | Нет — проекция перезаписывается атомарно |
| `mark_persisted` | Есть — вызывает репо после save | Нет — версия применилась = состояние записано |
| State-методы | `_update_version()` после мутации | Версию **не** трогают |
| `_check_state` | Применяется для заданных ограничений состояния | Только если такое ограничение задано |
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
| `version == self._version` | `EntityIdempotentError` — повтор переданной версии |
| `version < self._version` | `EntityVersionError` — передана устаревшая версия |
| `version > self._version` | Применяем |

**Зачем regression-проверка:**
События источника правды могут приходить не по порядку (брокер, ребалансировка консьюмеров). Если уже применили версию 7 и пришла 5 — нельзя откатывать состояние.

**Зачем idempotency-проверка:**
Одна и та же входящая версия может быть передана повторно.

### Почему state-методы не инкрементят version

В агрегате `_update_version()` отражает локальное изменение. В проекции версия
приходит извне. Если state-метод инкрементит её локально, счётчик расходится с
источником, поэтому операции проекции версию не изменяют.

### Почему нет `_check_state` в проекции

Не переноси `_check_state` агрегата в проекцию автоматически. Реализуй только
ограничения состояния, заданные для самой проекции.

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

### Операции проекции

Создавай только заданные операции. Метод принимает VO, выполняет все проверки до
мутации и не увеличивает версию локально:

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
| 1 | заданные проверки состояния | заданные проверки состояния |
| 2 | заданная семантика повтора | заданная семантика повтора |
| 3 | мутация | мутация |
| 4 | `_update_version()` | пропускаем — версия применится отдельно через `new_version` |

### Применение входного изменения

```python
user = await uow.user_repo.by_id(user_id)
user.new_state(user_state)
user.new_version(version)
await uow.user_repo.save(user)
```

Вызывающий слой сначала применяет заданное изменение, затем переданную версию.

`new_version` возьмёт на себя:
- Если версия равна текущей — `EntityIdempotentError` (повтор версии).
- Если меньше — `EntityVersionError` (устаревшая версия).
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

Совпадение идентификаторов проекции и другого объекта реализуй только при явном
требовании. Не выводи ID-parity из сходства полей, связи `1:1` или удобства
поиска. Идентификатор проекции всегда представлен отдельным VO.

## Сосуществующий агрегат и проекция

Если входной контракт отдельно задаёт проекцию и агрегат, реализуй их как разные
классы. Совпадение их ID допускается только при явном требовании:

| Аспект | Проекция | Сосуществующий агрегат |
|---|---|---|
| Источник правды | Внешний сервис | Текущий сервис (для своих полей) |
| Что хранит | Поля внешней сущности (status, state, ...) | Локальные поля (display_name, settings, ...) |
| Кто меняет | События из внешнего сервиса | Команды текущего сервиса + реакции на события проекции |
| ID | Собственный VO | Собственный VO; совпадает только по требованию |

### Реакция агрегатов на изменение проекции

Domain предоставляет только заданные операции проекции и агрегата. Порядок
загрузки, вызовов, сохранения и транзакции остаётся за пределами domain.

## Фабрика проекции

Фабрика проекции обязательна и предоставляет отдельные `new()` и `restore()`.
Оба метода принимают переданный вызывающим слоем ID и остальные поля только как
готовые VO:

```python
class UserFactory:
    @staticmethod
    def new(user_id: UserID, state: UserState, version: Version) -> User:
        return User(
            user_id=user_id,
            state=state,
            version=version,
        )

    @staticmethod
    def restore(user_id: UserID, state: UserState, version: Version) -> User:
        return User(
            user_id=user_id,
            state=state,
            version=version,
        )
```

Raw-данные преобразуются до вызова фабрики. Прямой вызов конструктора проекции
за пределами фабрики запрещён.

## Чек-лист добавления операции проекции

1. Имя отражает заданное доменное намерение.
2. Параметры — VO.
3. Возврат — `None`, мутация in-place.
4. Все заданные проверки выполняются до мутации.
5. Семантика повторного вызова реализована буквально.
6. **`_update_version` не вызывать.**
7. Версия передаётся отдельно через `new_version`.

## Анти-паттерны

❌ **Использование `entity.py` как имени файла.** Только `projection.py` — это сигнал читателю о роли класса.

❌ **`_update_version` в state-методе проекции.** Локальный счётчик разойдётся с источником правды.

❌ **`mark_persisted` / `_original_version` в проекции.** Этих понятий у проекции нет.

❌ **Поведение, отсутствующее во входном контракте.** Не расширяй проекцию
типовыми методами или предположениями о внешнем источнике.

❌ **Применение изменения без `new_version`.** Локальная версия останется прежней.

❌ **Слияние state и version в один метод.** `apply_event(state, version)` выглядит удобным, но прячет два разных режима ошибок (idempotent vs version-regression). Оставляем два явных вызова.

❌ **Прямой вызов конструктора проекции.** Используй только `Factory.new()` или
`Factory.restore()`.

❌ **Параметры с `default`, отсутствующим во входном контракте.**
