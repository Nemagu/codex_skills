---
name: python-cqrs-application-layer-writing
description: Используй при реализации или правке прикладного слоя в Python-проекте по гексагональной архитектуре с CQRS. Триггеры — добавление или изменение заданного use case, команды, запроса, DTO, порта, UnitOfWork, преобразования ошибок и механизма version/outbox в `application/`. Не применять для определения требований и слоёв `domain/`, `infrastructure/`, `presentation/`.
---

# Python CQRS Application Layer Writing

## Quick Start

1. Извлеки из задачи точные команды, запросы, входы, результаты, порты, порядок шагов, авторизацию, границы согласованности и публичные исходы. Не дополняй контракт типовыми элементами.
2. Размести операцию по бизнес-возможности; `user`/`system` используй только при заданном различии.
3. Объяви Command, Query и публичные DTO как глубоко неизменяемые `@dataclass(slots=True, frozen=True)`: коллекции — `tuple`, вложения — другие неизменяемые DTO.
4. Для partial update различай `UNSET`, `None` и переданное значение. Не используй truthiness вместо проверки контракта.
5. Преобразуй вход в domain VO до транзакции, если преобразованию не нужен I/O.
6. Инжектируй только требуемые порты. Для заданной атомарной группы используй обязательную фабрику нового UoW; независимый порт передавай напрямую.
7. Для нового агрегата получай техническое значение ID через типизированный
   исходящий порт, оборачивай его в доменный VO и передавай фабрике. Не требуй
   новый ID во входной команде без заданной внешней семантики.
8. Реализуй только заданную авторизацию и порядок доменных вызовов. Не добавляй инициатора, роли и policy по шаблону.
9. Явно преобразуй ожидаемые `DomainError` в публичные application-исходы; неизвестный доменный отказ — в `AppInternalError`.
10. Техническую ошибку адаптер преобразует в `AppPortError`, а use case — в публичный исход своей операции.
11. Формируй самодостаточные неизменяемые version/outbox DTO в заданных точках сохранения; не реализуй event sourcing.
12. Храни в use case только зависимости, а данные вызова — в локальных переменных. Не кешируй результаты в экземпляре.
13. Не создавай application-сценарии и не вызывай один use case из другого. Реализуй однозначно заданные части без придумывания семантики.

## When to Apply

### Триггеры активации

**По расположению файлов** — любая правка/создание в `application/`:
- `application/command/<business_capability>/{user,system}/<action>.py`
- `application/query/<business_capability>/{user,system}/<action>.py`
- `application/command/base.py`, `application/query/base.py`
- `application/dto/<aggregate>.py`, `application/dto/paginator.py`
- `application/port/unit_of_work.py`, `application/port/event_publisher.py`
- `application/port/repository/__init__.py`, `application/port/repository/<aggregate>.py`
- `application/error.py`

**По концепциям, упомянутым пользователем:**
- CQRS-термины: «use case», «сценарий», «command», «команда» в смысле прикладного действия, «query», «запрос».
- DTO, маппинг доменного объекта в DTO.
- Unit of Work, UoW, граница транзакции на уровне сценария.
- Порт, port, адаптер; EventPublisher; outbox-публикация.
- Иерархия прикладных ошибок (`AppError`, `AppNotFoundError`, `AppInvalidDataError`, `AppInternalError`).
- Авторизация инициатора, role check.
- Прикладной слой, application layer.

**По типам задач:**
- Добавление нового use case (команды или запроса).
- Добавление нового DTO.
- Расширение `UnitOfWork` под новый агрегат.
- Добавление нового интерфейса репозитория или новой группы (`outbox`, `subscription`).
- Добавление события в outbox + соответствующий publisher use case.
- Расширение иерархии прикладных ошибок.
- Правка списочных запросов (filter helper, paginator).
- Добавление источника идентификаторов, часов, кеша или другого исходящего порта.

### Анти-триггеры

Скил не активируется, когда правка относится к слоям `domain/`, `infrastructure/` или `presentation/` — для редактирования каждого из этих слоёв используется свой скил:
- `domain/` — доменная модель: агрегаты, value objects, фабрики, доменные сервисы, абстрактные domain-интерфейсы.
- `infrastructure/` — реализации портов: репозитории к БД, адаптеры к брокеру сообщений, миграции схемы БД.
- `presentation/` — точки входа в систему: HTTP-эндпоинты, фоновые worker-ы, схемы валидации входящих данных.
- Составные сценарии и pipeline-описания: отдельные классы сценариев в
  `application/` не создаются, один use case из другого не вызывается.

Также не активируется на косметические правки внутри `application/` (только docstrings, переименование без изменения контракта, импорт-рефакторинг).

### Именование структуры

- Предпочитай единственное число в каталогах и файлах: `command`, `query`, `port`, `repository`, `error.py`.
- Сохраняй уже принятую единообразную конвенцию проекта, если пользователь не согласовал переименование.
- Не включай массовое переименование в функциональную задачу без явного согласия.
- Все пути далее — иллюстрации; подставляй согласованную форму единственного или множественного числа последовательно.

### Предусловия

- Гексагональная архитектура с выделенным `application/`-слоем.
- CQRS: команды и запросы разнесены по разным деревьям.
- `application/` импортирует только из `domain/` и стандартной библиотеки. Запрещены импорты из `infrastructure/`, `presentation/`. Внешние библиотеки (web-фреймворки, драйверы БД, клиенты брокеров и т.п.) не используются.
- Состав и семантика application-операций заданы входными требованиями. Скил
  выбирает Python-реализацию, но не проектирует новые поля, исходы и шаги.
- Прикладная авторизация использует только заданные сведения и domain-контракты.
  Аутентификация и transport context остаются во входящем адаптере.

## Package Structure

```
application/
├── __init__.py                              ← пустой
├── error.py                                 ← иерархия AppError
│
├── dto/
│   ├── __init__.py                          ← пустой
│   ├── unset.py                             ← UNSET для partial update
│   ├── paginator.py                         ← LimitOffsetPaginator и общие пагинаторы
│   └── <meaning>.py                         ← публичные application DTO
├── error_mapping/
│   └── <business_capability>.py             ← переиспользуемые трансляторы
│
├── port/
│   ├── __init__.py                          ← пустой
│   ├── unit_of_work.py                      ← UnitOfWork + обязательная UnitOfWorkFactory
│   ├── identifier.py                        ← только требуемые типизированные источники ID
│   ├── event_publisher.py                   ← EventPublisher ABC + TypeAlias событий
│   ├── dto/                                 ← самодостаточные port DTO
│   │   └── <meaning>.py
│   └── repository/
│       ├── __init__.py                      ← только re-export + __all__
│       └── <aggregate>.py                   ← интерфейсы + <Aggregate>Repositories
│
├── command/
│   ├── __init__.py                          ← пустой
│   ├── base.py                              ← только реально общие helpers
│   └── <business_capability>/
│       ├── __init__.py                      ← публичная витрина возможности
│       ├── user/                            ← только при заданном разделении
│       │   └── <action>.py
│       └── system/                          ← только при заданном разделении
│           └── <action>.py
│
└── query/
    ├── __init__.py                          ← пустой
    ├── base.py                              ← только реально общие helpers
    └── <business_capability>/
        ├── __init__.py                      ← публичная витрина возможности
        ├── user/
        │   └── <action>.py
        └── system/
            └── <action>.py
```

Для малого набора операций сохраняй принятую плоскую структуру. Группируй по
бизнес-возможности при росте: более 12 файлов на уровне, минимум две устойчивые
группы с двумя операциями, повторяющиеся префиксы, необходимость индекса или
очевидное скорое достижение этих условий. Не перемещай существующие файлы
автоматически. Уровень bounded context добавляй только при явно выделенных
нескольких контекстах и устойчивой группировке возможностей.

### Назначение узлов

Ветки `user` и `system` одинаково применяются к командам и запросам:

- `user` и `system` отражают заданный источник операции, но сами по себе не
  добавляют `initiator_id`, роль или policy. Включай только сведения и проверки
  конкретного контракта.

Эта классификация не определяет доступность операции через HTTP или брокер и не
является заменой модификаторам видимости.

| Узел | Что внутри | Канон |
|---|---|---|
| `error.py` | `AppError` + `AppNotFoundError`, `AppInvalidDataError`, `AppInternalError` | один файл, не дробится |
| `dto/<meaning>.py` | Публичные application DTO, названные по смыслу данных | по связному представлению |
| `dto/unset.py` | Общий `UNSET` для partial update | один файл |
| `dto/paginator.py` | общие пагинаторы (`LimitOffsetPaginator`) | один на проект |
| `port/unit_of_work.py` | `UnitOfWork` ABC + обязательная фабрика нового экземпляра | один файл |
| `port/identifier.py` | типизированные источники новых ID | только при потребности |
| `port/event_publisher.py` | `EventPublisher` ABC + `TypeAlias` допустимых event DTO | один файл |
| `port/dto/<meaning>.py` | Самодостаточные внутренние DTO портов | создаётся по необходимости |
| `port/repository/__init__.py` | Только re-export и алфавитный `__all__` | публичная витрина |
| `port/repository/<aggregate>.py` | Repository-интерфейсы и `<Aggregate>Repositories` | по одному файлу на агрегат |
| `command/base.py` / `query/base.py` | базовые классы use case-ов (если приняты в проекте) | пример конвенции, не предписание |
| `command/<business_capability>/.../<action>.py` | один use case = один файл | группировка по бизнес-возможности |
| `query/<business_capability>/.../<action>.py` | dataclass-запрос + use case-класс | то же |
| `error_mapping/<business_capability>.py` | Повторяющиеся трансляторы `DomainError` | только при переиспользовании |

### Конвенции по `__init__.py`

| Уровень | Состояние | Содержимое |
|---|---|---|
| `application/__init__.py` | пустой | — |
| `application/{command,query,dto,port}/__init__.py` | пустой | — |
| `application/{command,query}/<business_capability>/__init__.py` | публичная витрина | операции возможности |
| **`application/command/<business_capability>/__init__.py`** | **не пустой** | re-export публичных use case и Command возможности |
| **`application/query/<business_capability>/__init__.py`** | **не пустой** | re-export публичных use case и Query возможности |
| `application/dto/__init__.py` | пустой | DTO импортируются напрямую из `application.dto.<aggregate>` |
| `application/port/__init__.py` | пустой | — |
| **`application/port/repository/__init__.py`** | **не пустой** | только re-export контрактов и алфавитный `__all__` |

Входящие адаптеры импортируют операции через витрину бизнес-возможности.
Изнутри application используй прямые пути.

### Naming convention для use case-ов (CQRS verb-first)

**Команды — императивный verb + аггрегат:**

| Файл | Command | UseCase |
|---|---|---|
| `create.py` | `Create<Aggregate>Command` | `Create<Aggregate>UseCase` |
| `update.py` | `Update<Aggregate>Command` | `Update<Aggregate>UseCase` |
| `delete.py` | `Delete<Aggregate>Command` | `Delete<Aggregate>UseCase` |
| `restore.py` | `Restore<Aggregate>Command` | `Restore<Aggregate>UseCase` |
| `publish.py` | — (нет входа) | `Publish<Aggregate>VersionsUseCase` |
| `<verb>_<object>.py` (доменный) | `<Verb><Aggregate><Object>Command` | `<Verb><Aggregate><Object>UseCase` |

**Запросы — `Get` / `List` + описание возвращаемого результата:**

| Файл | Query | UseCase |
|---|---|---|
| `retrieve_last_version.py` | `Get<Aggregate>LastVersionQuery` | `Get<Aggregate>LastVersionUseCase` |
| `retrieve_version.py` | `Get<Aggregate>VersionQuery` | `Get<Aggregate>VersionUseCase` |
| `list_last_versions.py` | `List<Aggregate>LastVersionsQuery` | `List<Aggregate>LastVersionsUseCase` |
| `list_versions.py` | `List<Aggregate>VersionsQuery` | `List<Aggregate>VersionsUseCase` |

Имена файлов следуют столбцу «Файл» — это часть структуры слоя, не имя класса.

Тип сохранённого изменения называй по смыслу (`ProjectChange`, `TenantChange`) и
размещай рядом с version DTO. Это application-метаданные снимка, не доменное
событие и не transport message.

### Чего в layout-е нет и почему

- `system` и `user` обозначают источник сценария, а не публичность Python API или транспортную доступность.
- `query/system/` создавай только при наличии реального системного сценария чтения; пустую ветку заранее не добавляй.
- Не создавай event-sourcing API, `events.py` и коллекцию событий агрегата.

## Errors Hierarchy

### Полная иерархия

```
AppError (msg, action, data)
├── AppNotFoundError       — основной ресурс не найден
├── AppInvalidDataError    — зависимость/контекст невалиден или отсутствует
├── <PublicOutcomeError>   — различимый публичный исход операции
├── AppInternalError       — непредусмотренный внутренний исход (+ wrap_error)
└── AppPortError           — внутренняя ошибка исходящего порта (+ wrap_error)
```

Иерархия живёт в `application/error.py`. Создавай отдельный публичный тип для
каждого исхода, который вызывающая сторона должна различать. Не создавай разные
типы для одинаковой публичной семантики. `AppPortError` не пересекает входящую
границу: use case всегда преобразует его в публичный исход или `AppInternalError`.

### Базовый класс

```python
from typing import Any


class AppError(Exception):
    def __init__(
        self,
        msg: str,
        action: str,
        data: dict[str, Any] | None = None,
        *args: object,
    ) -> None:
        super().__init__(msg, *args)
        self.msg = msg
        self.action = action
        self.data = data or {}


class AppNotFoundError(AppError):
    pass


class AppInvalidDataError(AppError):
    pass


class AppInternalError(AppError):
    def __init__(
        self,
        msg: str,
        action: str,
        data: dict[str, Any] | None = None,
        wrap_error: BaseException | None = None,
        *args: object,
    ) -> None:
        super().__init__(msg, action, data, *args)
        self.wrap_error = wrap_error


class AppPortError(AppInternalError):
    pass
```

### Поля

- **`msg`** — короткое человекочитаемое описание на русском, без переменных. Пример: `"арендатор не существует"`, `"транзакция уже опубликована"`, `"инициатор не существует"`.
- **`action`** — контекст операции, в которой возникла ошибка. Совпадает с локальной `action` use case-а. Пример: `"обновление арендатора"`, `"удаление транзакции"`, `"получение последних версий категорий"`.
- **`data`** — безопасный структурированный application-контекст. Это не готовое
  тело API-ответа и не transport-контракт.
- **`wrap_error`** (только `AppInternalError`) — оригинальное исключение, если ошибка оборачивает техническое.

### Конвенция формирования `data`

**Ключи верхнего уровня — английские, в snake_case, отражают тип сущности:**

| Тип ошибки | Ключ верхнего уровня | Значение |
|---|---|---|
| Ошибка одного агрегата | имя сущности в ед. ч.: `"tenant"`, `"transaction"`, `"category"`, `"user"` | `dict` с полями этой сущности |
| Ошибка с участием нескольких сущностей одного типа | имя во мн. ч.: `"categories"`, `"transactions"` | `list[dict]` |
| Ошибка отдельного значения / поля | имя поля: `"version"`, `"event"`, `"status"` | примитив или `dict` |

**Внутри блока сущности — поля в snake_case, значения — примитивы, stdlib-типы
или неизменяемые application DTO:**

```python
# Ошибка одного агрегата
{"tenant": {"tenant_id": UUID(...)}}

# Ошибка с двумя сущностями разных типов
{
    "tenant": {"tenant_id": UUID(...)},
    "transaction": {"transaction_id": UUID(...)},
}

# Ошибка с коллекцией зависимостей
{
    "transaction": {"transaction_id": UUID(...)},
    "categories": [
        {"category_id": UUID(...)},
        {"category_id": UUID(...)},
    ],
}

# Ошибка отдельного значения
{"event": "wrong_value"}
{"version": 0}
```

Не помещай в `data` domain-объекты, VO, технические исключения, stack trace,
секреты и детали адаптера. Для коллекций используй `tuple`. Исходная причина
хранится только в `wrap_error`; внешний адаптер сам строит безопасный формат.

### Выбор типа ошибки

Выбирай тип только по публичной семантике заданного исхода. Не определяй его
механически по цели, зависимости, инициатору или имени use case. Пустой результат,
отсутствие объекта, конфликт, отказ авторизации и недоступность порта реализуй
ровно так, как задано операцией.

Ожидаемые domain-ошибки преобразуй локальным `try/except` вокруг минимального
доменного вызова. Повторяющуюся карту вынеси в функцию с возвратом `NoReturn`.
Неизвестный `DomainError` преобразуй в `AppInternalError`. Адаптер преобразует
ожидаемую ошибку зависимости в `AppPortError`; use case придаёт ей публичный смысл
и свой `ACTION`. Не перехватывай `BaseException`, `CancelledError` и публичный
`AppError`; внутренний `AppPortError` является отдельным разрешённым случаем.

### Правила вызова

- **Всегда через kwargs.** `raise AppNotFoundError(msg="...", action=action, data={...})`.
- **`data` пропускаем, если её нет** — не передавать `data={}` явно.
- **`wrap_error` пропускаем, если его нет** — не передавать `wrap_error=None` явно.
- **`AppError` напрямую не инстанцируется** — только подклассы.

## Public API of Module

### Что попадает в витрину бизнес-возможности

В `application/{command,query}/<business_capability>/__init__.py` re-export-ятся
публичные use case и соответствующие Command/Query этой возможности.

| Категория | Примеры |
|---|---|
| Use case-классы | `UpdateTenantUseCase`, `GetTenantLastVersionUseCase` |
| Соответствующие Command/Query dataclass-ы | `UpdateTenantCommand`, `GetTenantLastVersionQuery` |

**Не попадает:**
- Helper-методы из `<aggregate>/base.py` (приватные, внутреннее использование).
- Базовые классы из `command/base.py` / `query/base.py` — импортируются напрямую.

### Шаблон витрины бизнес-возможности

```python
from application.command.user.tenant.delete import (
    DeleteTenantCommand,
    DeleteTenantUseCase,
)
from application.command.user.tenant.restore import (
    RestoreTenantCommand,
    RestoreTenantUseCase,
)
from application.command.user.tenant.update import (
    UpdateTenantCommand,
    UpdateTenantUseCase,
)

__all__ = [
    "DeleteTenantCommand",
    "DeleteTenantUseCase",
    "RestoreTenantCommand",
    "RestoreTenantUseCase",
    "UpdateTenantCommand",
    "UpdateTenantUseCase",
]
```

**`__all__` строго в алфавитном порядке.**

### `port/repository/<aggregate>.py` и публичная витрина

```python
from application.port.repository.tenant import (
    TenantEvent,
    TenantOutboxRepository,
    TenantReadRepository,
    TenantVersionDTO,
    TenantVersionRepository,
    TenantRepositories,
)

__all__ = [
    "TenantEvent",
    "TenantOutboxRepository",
    "TenantReadRepository",
    "TenantRepositories",
    "TenantVersionDTO",
    "TenantVersionRepository",
    # ...
]
```

`TenantRepositories` объявляй в `repository/tenant.py` рядом с интерфейсами своего агрегата. В `repository/__init__.py` оставляй только re-export и алфавитный `__all__`.

### Top-level `application/__init__.py` — пустой

Не делай re-export на верхний уровень. Use case импортируй через витрину
бизнес-возможности, DTO, ошибки и базовые классы — прямыми путями.

### Правила импорта

| Где | Откуда импортируем |
|---|---|
| Из `application/command/<business_capability>/.../<action>.py` | Прямые пути к DTO, ошибкам, портам и domain |
| Из `application/query/<business_capability>/.../<action>.py` | Аналогично |
| Из входящего адаптера | Витрина бизнес-возможности |
| Из `infrastructure/` (для имплементации портов) | Прямые пути: `from application.port.repository.tenant import TenantReadRepository` |

## Anti-patterns (сводный чеклист)

### Утечка слоёв
- Импорт из `infrastructure/` или `presentation/` в `application/`.
- Импорт внешних библиотек в `application/`.
- Domain VO/entity в полях команд/запросов.
- Domain VO/entity в return type `execute`.
- Domain VO/entity в значениях `data` ошибки.
- Re-export use case-ов из верхнеуровневых `__init__.py` (только агрегатный уровень).

### Транзакционные границы
- Инъекция или хранение экземпляра UoW вместо stateless-фабрики.
- Фабрика, возвращающая ранее использованный UoW.
- Вложенный `async with self._uow_factory()` в одном `execute`.
- Доступ к репозиториям через поле use case вместо локального `uow`.
- Несколько транзакций для одного набора согласованных изменений.
- UoW в read-only/stateless-сценарии без транзакционной потребности.
- Helper-метод, работающий с репозиториями, без `uow`-параметра.
- Хранение `uow` в state объекта.
- Ручной `commit()`/`rollback()` в use case или подавление исключения в `__aexit__`.
- Использование репозитория после выхода из UoW.

### Авторизация
- Авторизация, initiator или role-check, добавленные только по метке user/system.
- Загрузка агрегатов **до** проверки роли инициатора.
- Глобальная проверка роли без последующего per-aggregate authorization-сервиса (для user-команды над конкретным агрегатом).
- `_initiator` вне `async with` — нужен открытый `uow`.

### Команды, запросы, DTO
- Команда/запрос/DTO без `@dataclass(slots=True, frozen=True)`.
- Изменяемые `list`/`dict` внутри frozen DTO; используй `tuple` и вложенные DTO.
- Поля команд/запросов: VO, domain entity, response-DTO, типы presentation/infrastructure.
- Конверсия примитивов в VO внутри команды (`__post_init__` и т.п.) — конверсия в use case.
- `execute` с двумя+ аргументами — только 0 или 1.
- DTO с методами поведения (валидации, вычисления) — только `from_*` фабрики.
- Domain entity/VO в публичном application DTO.
- `to_dict` / `to_json` / `model_dump` в DTO — сериализация это presentation.
- Использование `None` одновременно как «не передано» и «очистить»; применяй `UNSET`.

### Ports
- Тело в абстрактном методе — должен быть `...`.
- Реализация в `port/` — реализации в `infrastructure/`.
- Беспричинное разбиение `port/repository/<aggregate>.py` на мелкие файлы.
- Изменяемая `<Aggregate>Repositories`; группа должна быть `frozen=True`.
- `next_id()` в repository-порте; используй отдельный ID provider.
- Вызов одного и того же порта в цикле вместо требуемого batch-контракта.

### Ошибки
- f-строки в `msg`.
- Позиционные аргументы при создании ошибки.
- `data={}` явным пустым литералом — параметр опускается.
- VO/entity в значениях `data` — развёрнуто в примитивы.
- Повторное оборачивание публичного `AppError`; `AppPortError` преобразуется отдельно.
- Пропуск `DomainError` или `AppPortError` во входящий адаптер.
- Перехват `BaseException` или `asyncio.CancelledError`.
- Глобальная таблица преобразования всех domain-ошибок без контекста операции.
- Возврат `AppError | None` из порта вместо успешного результата или исключения.
- Пустой `action` в ошибке адаптера; адаптер формирует контекст сразу.
- Прямой `raise AppError(...)` — только подклассы.
- Отсутствие отдельного типа для публично различимого application-исхода.
- Бизнес-логика в `execute`, минуя domain.

### Outbox / publisher
- `event_publisher.publish` прямо из обычной command use case (нарушение outbox-паттерна).
- Publisher use case с командой на вход.
- DB-транзакция на время сетевой публикации без согласованного изменения состояния.
- Отметка `published` до полного успеха batch или частичная отметка при ошибке.
- Формирование broker payload/subject в application вместо infrastructure-адаптера.
- Точки version/outbox-сохранения, отсутствующие в контракте операции.
- Передача изменяемого агрегата и отдельных `change`/`editor_id` вместо
  самодостаточного immutable version DTO.
- Outbox DTO, которому адаптеру не хватает данных для сохранения записи.

### Naming и структура
- Имена use case-ов с `-ion` / `-ing` / agent-noun — только verb-first (`CreateTenant`, `GetTenantLastVersion`).
- Команда без суффикса `Command`, запрос без суффикса `Query`.
- Несколько use case-ов в одном файле.
- Имя файла, не совпадающее с действием.
- Техническое имя DTO (`OutputDTO`, `PortDTO`, `ResponseDTO`) вместо имени по смыслу данных.
- Команда/запрос с `<Aggregate><Action>` ordering вместо verb-first.
- Группировка растущего дерева только по агрегатам или transport-адаптерам.
- Каталог `application/scenario/` и вызов одного use case из другого.
- Динамический или дублирующийся `action`; используй `ClassVar[str] ACTION`.
- Command, Query, агрегат, локальный UoW или результат, сохранённые в `self`.
- Неявное кеширование результата в use case вместо отдельного cache-порта.

## References

- **`references/use_cases.md`** — анатомия use case-а: базовые классы, конструктор, шаблон `execute` с фазами, initiator + role, per-aggregate authorization, транзакционные границы UoW, обработка ошибок.
- **`references/commands_and_queries.md`** — формы команд и запросов, допустимые типы полей, конверсия в VO, return-type правило, контракт `execute`, command vs query, helper `_filtering_data`.
- **`references/dtos.md`** — входные, выходные и внутренние port DTO, допустимые типы и семантическое именование.
- **`references/ports.md`** — порты: `UnitOfWork`, `EventPublisher`, repository-интерфейсы (`Read`, `Version`, `Outbox`, `Subscription`), группы `<Aggregate>Repositories`, импорты в `port/`.
- **`references/outbox_publisher.md`** — outbox pattern: зачем, форма publisher use case, шаблон, отличия от обычной команды, запрет публикации из команды.
- **`references/checklists.md`** — пошаговые чек-листы добавления команды, запроса, DTO, портов нового агрегата, publisher use case-а.
