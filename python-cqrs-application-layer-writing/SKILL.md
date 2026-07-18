---
name: python-cqrs-application-layer-writing
description: Используй при проектировании или правке прикладного слоя в Python-проекте по гексагональной архитектуре с CQRS. Триггеры — добавление или изменение use case (команды/запроса), DTO, портов (UnitOfWork, EventPublisher, repository interfaces), иерархии прикладных ошибок в `application/`. Не применять для слоёв `domain/`, `infrastructure/`, `presentation/`.
---

# Python CQRS Application Layer Writing

## Quick Start

1. Определи тип use case: команда (мутирует состояние, возвращает DTO или `None`) или запрос (только чтение, возвращает DTO либо `tuple[list[DTO], int]`).
2. Определи аудиторию: public (вызывается пользователем, требует initiator + проверку роли) или private (системный/событийный, без initiator).
3. Сначала проверь конвенцию имён проекта. По умолчанию предпочитай единственное число: `application/command/...`, `application/query/...`, `application/port/repository/...`. Если проект последовательно использует множественное число, предложи пользователю сохранить его или согласовать отдельное переименование; не смешивай формы автоматически.
4. Выбери зависимости по сценарию: UoW — для согласованных изменений в нескольких репозиториях, прямой порт — для одного ресурса или stateless-операции. Выноси зависимости в базовый класс только при реальном переиспользовании.
5. Объяви команду/запрос как `@dataclass(slots=True, frozen=True)` с примитивами (`UUID`, `int`, `str`, `list[...] | None`). Доменные VO в команды/запросы не помещаем.
6. Тело `execute` начинается с `action: str` (русское описание для контекста ошибок), затем конверсия примитивов в VO (всё, что может упасть и не требует БД — до открытия транзакции), затем `async with self._uow as uow:`.
7. В public use case первым делом: получить инициатора и проверить его роль. Helper-методы, работающие с репозиториями, принимают `uow` параметром (`__aenter__` UoW может вернуть транзакционно-связанный объект, отличный от `self._uow`).
8. Загружай агрегаты через `uow.<x>_repositories.read.by_id(...)`. На `None` бросай `AppNotFoundError` (для основного ресурса операции — цели) или `AppInvalidDataError` (для зависимости/контекста). Initiator — всегда `AppInvalidDataError`.
9. Авторизуй конкретные агрегаты через доменный per-aggregate сервис (если есть в домене). Это дополняет глобальную проверку роли инициатора, проверяя право над *этими* объектами.
10. Вызови мутатор агрегата (для query — пропусти), затем атомарно сохрани согласованные изменения через `read`, `version` и `outbox`-порты. Не открывай транзакцию, если сценарий не выполняет согласованных изменений.
11. Разделяй входные Command/Query, выходные DTO для presentation и внутренние port DTO. Называй DTO по смыслу данных (`UserDTO`, `UserVersionDTO`, `UserEventDTO`, `ClaimsDTO`), не по слою-получателю.
12. Не лови исключения в use case. Адаптер сам преобразует техническую ошибку в application-ошибку, заполняет `action` и сохраняет исходную причину в `wrap_error`.
13. Запрещено: f-строки в `msg=` ошибок, бизнес-логика в `execute`, импорты из `infrastructure/` или `presentation/`, доменные типы во входных и выходных DTO, обращения к репозиторию вне активного UoW.

## When to Apply

### Триггеры активации

**По расположению файлов** — любая правка/создание в `application/`:
- `application/command/{public,private}/<aggregate>/<action>.py`
- `application/query/<aggregate>/<action>.py`
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

### Анти-триггеры

Скил не активируется, когда правка относится к слоям `domain/`, `infrastructure/` или `presentation/` — для редактирования каждого из этих слоёв используется свой скил:
- `domain/` — доменная модель: агрегаты, value objects, фабрики, доменные сервисы, абстрактные domain-интерфейсы.
- `infrastructure/` — реализации портов: репозитории к БД, адаптеры к брокеру сообщений, миграции схемы БД.
- `presentation/` — точки входа в систему: HTTP-эндпоинты, фоновые worker-ы, схемы валидации входящих данных.

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
- В проекте есть domain-слой с агрегатами, VO, доменными сервисами и абстрактными domain-репозиториями. При отсутствии или неполноте домена — согласование с пользователем до начала работы.
- Прикладная авторизация опирается на доменную модель ролей и политик, если она нужна сценарию. HTTP-аутентификацию, извлечение токена и формирование request context по умолчанию оставляй в presentation.

## Package Structure

```
application/
├── __init__.py                              ← пустой
├── error.py                                 ← иерархия AppError
│
├── dto/
│   ├── __init__.py                          ← пустой
│   ├── paginator.py                         ← LimitOffsetPaginator и общие пагинаторы
│   └── <aggregate>.py                       ← выходные DTO для presentation
│
├── port/
│   ├── __init__.py                          ← пустой
│   ├── unit_of_work.py                      ← UnitOfWork ABC + property на репо-группы
│   ├── event_publisher.py                   ← EventPublisher ABC + TypeAlias событий
│   ├── dto/                                 ← внутренние port DTO при 2+ потребителях
│   │   └── <aggregate>.py
│   └── repository/
│       ├── __init__.py                      ← только re-export + __all__
│       └── <aggregate>.py                   ← интерфейсы + <Aggregate>Repositories
│
├── command/
│   ├── __init__.py                          ← пустой
│   ├── base.py                              ← BaseUseCase + PublisherUseCase (пример конвенции)
│   ├── public/
│   │   ├── __init__.py                      ← пустой
│   │   └── <aggregate>/
│   │       ├── __init__.py                  ← НЕ пустой: re-export use case-ов агрегата
│   │       ├── base.py                      ← опционально: общий helper для 2+ use case-ов
│   │       └── <action>.py                  ← один use case на файл
│   └── private/
│       ├── __init__.py                      ← пустой
│       └── <aggregate>/
│           ├── __init__.py                  ← НЕ пустой: re-export use case-ов агрегата
│           └── <action>.py
│
└── query/
    ├── __init__.py                          ← пустой
    ├── base.py                              ← BaseUseCase (без PublisherUseCase)
    └── public/
        ├── __init__.py                      ← пустой
        └── <aggregate>/
            ├── __init__.py                  ← НЕ пустой: re-export use case-ов агрегата
            ├── base.py                      ← опционально: общий helper для 2+ use case-ов
            └── <action>.py
```

### Назначение узлов

| Узел | Что внутри | Канон |
|---|---|---|
| `error.py` | `AppError` + `AppNotFoundError`, `AppInvalidDataError`, `AppInternalError` | один файл, не дробится |
| `dto/<aggregate>.py` | Выходные DTO для presentation, названные по смыслу данных | по одному файлу на агрегат или сценарий |
| `dto/paginator.py` | общие пагинаторы (`LimitOffsetPaginator`) | один на проект |
| `port/unit_of_work.py` | `UnitOfWork` ABC: `__aenter__`/`__aexit__` + property на группы | один файл |
| `port/event_publisher.py` | `EventPublisher` ABC + `TypeAlias` допустимых event DTO | один файл |
| `port/dto/<aggregate>.py` | Внутренние DTO, общие для нескольких портов/адаптеров | создаётся по необходимости |
| `port/repository/__init__.py` | Только re-export и алфавитный `__all__` | публичная витрина |
| `port/repository/<aggregate>.py` | Repository-интерфейсы и `<Aggregate>Repositories` | по одному файлу на агрегат |
| `command/base.py` / `query/base.py` | базовые классы use case-ов (если приняты в проекте) | пример конвенции, не предписание |
| `command/{public,private}/<aggregate>/<action>.py` | один use case = один файл: dataclass-команда + use case-класс | каждый файл публичный |
| `query/<aggregate>/<action>.py` | dataclass-запрос + use case-класс | то же |
| `command/{public,private}/<aggregate>/base.py` <br> `query/<aggregate>/base.py` | **опционально**: общий helper для 2+ use case-ов одной группы | создаётся, только когда 2+ use case-а делят helper |

### Конвенции по `__init__.py`

| Уровень | Состояние | Содержимое |
|---|---|---|
| `application/__init__.py` | пустой | — |
| `application/{command,query,dto,port}/__init__.py` | пустой | — |
| `application/command/{public,private}/__init__.py` | пустой | — |
| **`application/command/{public,private}/<aggregate>/__init__.py`** | **не пустой** | re-export всех use case-ов агрегата + Command-dataclasses, объединённый `__all__` |
| **`application/query/<aggregate>/__init__.py`** | **не пустой** | re-export всех use case-ов агрегата + Query-dataclasses, объединённый `__all__` |
| `application/dto/__init__.py` | пустой | DTO импортируются напрямую из `application.dto.<aggregate>` |
| `application/port/__init__.py` | пустой | — |
| **`application/port/repository/__init__.py`** | **не пустой** | только re-export контрактов и алфавитный `__all__` |

Импорты из presentation — через `<aggregate>/__init__.py` витрину: `from application.command.public.tenant import UpdateTenantUseCase, UpdateTenantCommand`. Изнутри одного use case-файла — прямой путь (`from application.command.base import BaseUseCase`).

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

Имена событий (`<Aggregate>Event` в `port/repository/<aggregate>.py` или `port/dto/<aggregate>.py`) живут по отдельному правилу: noun + past (`Created`, `Updated`, `Deleted`, `Restored`).

### Чего в layout-е нет и почему

- По умолчанию нет `query/private/`: системное чтение оформляй отдельным портом или добавляй private-ветку только при реальной потребности.
- Не создавай отдельный `events.py` без нескольких независимых потребителей: держи `<Aggregate>Event` рядом с repository-контрактом или общим event DTO.

## Errors Hierarchy

### Полная иерархия

```
AppError (msg, action, data)
├── AppNotFoundError       — основной ресурс не найден
├── AppInvalidDataError    — зависимость/контекст невалиден или отсутствует
└── AppInternalError       — внутренняя ошибка слоя/инфраструктуры (+ wrap_error)
```

Иерархия живёт в одном файле — `application/error.py`. Per-aggregate подклассов **по умолчанию не вводим** — агрегат живёт в `data`, не в имени класса. Расширение иерархии (новые подклассы для специфичных случаев) — по согласованию с пользователем.

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
```

### Поля

- **`msg`** — короткое человекочитаемое описание на русском, без переменных. Пример: `"арендатор не существует"`, `"транзакция уже опубликована"`, `"инициатор не существует"`.
- **`action`** — контекст операции, в которой возникла ошибка. Совпадает с локальной `action` use case-а. Пример: `"обновление арендатора"`, `"удаление транзакции"`, `"получение последних версий категорий"`.
- **`data`** — структурированный контекст ошибки для логов и API-ответов. Опционально, по умолчанию `{}`.
- **`wrap_error`** (только `AppInternalError`) — оригинальное исключение, если ошибка оборачивает техническое.

### Конвенция формирования `data`

**Ключи верхнего уровня — английские, в snake_case, отражают тип сущности:**

| Тип ошибки | Ключ верхнего уровня | Значение |
|---|---|---|
| Ошибка одного агрегата | имя сущности в ед. ч.: `"tenant"`, `"transaction"`, `"category"`, `"user"` | `dict` с полями этой сущности |
| Ошибка с участием нескольких сущностей одного типа | имя во мн. ч.: `"categories"`, `"transactions"` | `list[dict]` |
| Ошибка отдельного значения / поля | имя поля: `"version"`, `"event"`, `"status"` | примитив или `dict` |

**Внутри блока сущности — поля в snake_case (английские), значения — примитивы для JSON:**

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

**Правила значений:**
- `UUID` оставляем как `UUID`-объект, не строкой — сериализатор API сам приведёт.
- `Decimal` приводим к `str` (`str(amount)`), чтобы избежать потери точности при JSON-сериализации.
- `datetime` оставляем как есть, сериализатор API приведёт.
- `Enum` берём `.value` (строку), не сам enum.
- Доменные VO в значениях не появляются — всегда разворачиваем в примитив (`tenant_id.tenant_id`, `status.value`).

### Выбор типа ошибки

| Условие | Тип |
|---|---|
| Ресурс — цель операции (существительное в имени use case-а) | `AppNotFoundError` |
| Ресурс — зависимость/контекст (упоминается в команде, но не цель) | `AppInvalidDataError` |
| Initiator (всегда) | `AppInvalidDataError` |
| Use case создания: цели ещё нет, все загружаемые суть зависимости | `AppInvalidDataError` |
| List-запрос: пустой список валиден | ничего не бросается |
| Невалидное внутреннее состояние, impossible-кейс, обёртка тех.исключения | `AppInternalError` |

Цель операции определяется механически: существительное в имени класса use case-а. `UpdateTenantUseCase` → цель = `Tenant` → `tenant_id` даёт NotFound. `AppointUserAdminUseCase` → цель = `User` → `user_id` даёт NotFound. Зависимости (`category_id` в `CreateTransactionUseCase`) → InvalidData.

### Правила вызова

- **Всегда через kwargs.** `raise AppNotFoundError(msg="...", action=action, data={...})`.
- **`data` пропускаем, если её нет** — не передавать `data={}` явно.
- **`wrap_error` пропускаем, если его нет** — не передавать `wrap_error=None` явно.
- **`AppError` напрямую не инстанцируется** — только подклассы.

## Public API of Module

### Что попадает в витрину `<aggregate>/__init__.py`

В `application/command/{public,private}/<aggregate>/__init__.py` и `application/query/<aggregate>/__init__.py` re-export-ятся:

| Категория | Примеры |
|---|---|
| Use case-классы | `UpdateTenantUseCase`, `GetTenantLastVersionUseCase` |
| Соответствующие Command/Query dataclass-ы | `UpdateTenantCommand`, `GetTenantLastVersionQuery` |

**Не попадает:**
- Helper-методы из `<aggregate>/base.py` (приватные, внутреннее использование).
- Базовые классы из `command/base.py` / `query/base.py` — импортируются напрямую.

### Шаблон `<aggregate>/__init__.py`

```python
from application.command.public.tenant.delete import (
    DeleteTenantCommand,
    DeleteTenantUseCase,
)
from application.command.public.tenant.restore import (
    RestoreTenantCommand,
    RestoreTenantUseCase,
)
from application.command.public.tenant.update import (
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

Не делаем re-export на верхний уровень. Импорты на стороне идут через `<aggregate>/__init__.py` витрины (для use case-ов) или прямые пути (для DTO, ошибок, базовых классов).

### Правила импорта

| Где | Откуда импортируем |
|---|---|
| Из `application/command/<...>/<aggregate>/<action>.py` | Прямые пути: `from application.command.base import BaseUseCase`, `from application.dto.tenant import TenantDTO`, `from application.error import AppNotFoundError`, `from domain.tenant import TenantID, TenantPolicyService` |
| Из `application/query/<aggregate>/<action>.py` | Аналогично |
| Из `presentation/` | Витрина: `from application.command.public.tenant import UpdateTenantUseCase, UpdateTenantCommand` |
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
- Вложенный `async with self._uow` в одном `execute`.
- Доступ к репозиториям через `self._uow.<...>` вместо локального `uow` после `async with`.
- Несколько транзакций для одного набора согласованных изменений.
- UoW в read-only/stateless-сценарии без транзакционной потребности.
- Helper-метод, работающий с репозиториями, без `uow`-параметра.
- Хранение `uow` в state объекта.

### Авторизация
- Любая авторизация в private use case (там нет initiator).
- Загрузка агрегатов **до** проверки роли инициатора.
- Глобальная проверка роли без последующего per-aggregate authorization-сервиса (для public-команды над конкретным агрегатом).
- `_initiator` вне `async with` — нужен открытый `uow`.

### Команды, запросы, DTO
- Команда/запрос/DTO без `@dataclass(slots=True, frozen=True)`.
- Поля команд/запросов: VO, domain entity, response-DTO, типы presentation/infrastructure.
- Конверсия примитивов в VO внутри команды (`__post_init__` и т.п.) — конверсия в use case.
- `execute` с двумя+ аргументами — только 0 или 1.
- DTO с методами поведения (валидации, вычисления) — только `from_*` фабрики.
- Domain entity/VO во внешнем DTO для presentation.
- `to_dict` / `to_json` / `model_dump` в DTO — сериализация это presentation.

### Ports
- Тело в абстрактном методе — должен быть `...`.
- Реализация в `port/` — реализации в `infrastructure/`.
- Беспричинное разбиение `port/repository/<aggregate>.py` на мелкие файлы.
- `<Aggregate>Event` без `from_str`.
- `<Aggregate>Repositories` с `frozen=True`.

### Ошибки
- f-строки в `msg`.
- Позиционные аргументы при создании ошибки.
- `data={}` явным пустым литералом — параметр опускается.
- VO/entity в значениях `data` — развёрнуто в примитивы.
- `try/except AppError` в use case — control flow слоя ловить нельзя.
- `try/except DomainError` или технической ошибки в use case.
- Возврат `AppError | None` из порта вместо успешного результата или исключения.
- Пустой `action` в ошибке адаптера; адаптер формирует контекст сразу.
- Прямой `raise AppError(...)` — только подклассы.
- Per-aggregate подклассы (`TenantNotFoundError` и т.п.) по умолчанию не вводятся.
- Бизнес-логика в `execute`, минуя domain.

### Outbox / publisher
- `event_publisher.publish` прямо из обычной command use case (нарушение outbox-паттерна).
- Publisher use case с командой на вход.
- DB-транзакция на время сетевой публикации без согласованного изменения состояния.
- Отметка `published` до полного успеха batch или частичная отметка при ошибке.
- Формирование broker payload/subject в application вместо infrastructure-адаптера.
- `version.save` без предшествующего `read.save` — состояние и аудит идут парой.
- `outbox.save` у агрегата без `outbox`-поля в `<Aggregate>Repositories`.

### Naming и структура
- Имена use case-ов с `-ion` / `-ing` / agent-noun — только verb-first (`CreateTenant`, `GetTenantLastVersion`).
- Команда без суффикса `Command`, запрос без суффикса `Query`.
- Несколько use case-ов в одном файле.
- Имя файла, не совпадающее с действием.
- Техническое имя DTO (`OutputDTO`, `PortDTO`, `ResponseDTO`) вместо имени по смыслу данных.
- Команда/запрос с `<Aggregate><Action>` ordering вместо verb-first.

## References

- **`references/use_cases.md`** — анатомия use case-а: базовые классы, конструктор, шаблон `execute` с фазами, initiator + role, per-aggregate authorization, транзакционные границы UoW, обработка ошибок.
- **`references/commands_and_queries.md`** — формы команд и запросов, допустимые типы полей, конверсия в VO, return-type правило, контракт `execute`, command vs query, helper `_filtering_data`.
- **`references/dtos.md`** — входные, выходные и внутренние port DTO, допустимые типы и семантическое именование.
- **`references/ports.md`** — порты: `UnitOfWork`, `EventPublisher`, repository-интерфейсы (`Read`, `Version`, `Outbox`, `Subscription`), группы `<Aggregate>Repositories`, импорты в `port/`.
- **`references/outbox_publisher.md`** — outbox pattern: зачем, форма publisher use case, шаблон, отличия от обычной команды, запрет публикации из команды.
- **`references/checklists.md`** — пошаговые чек-листы добавления команды, запроса, DTO, портов нового агрегата, publisher use case-а.
