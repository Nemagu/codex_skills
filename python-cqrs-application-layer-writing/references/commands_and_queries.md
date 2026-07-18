# Commands & Queries

## Форма dataclass

Команда и запрос — простые dataclass-ы для пересечения границы слоя в направлении «presentation → application». Это входные данные use case-а.

```python
from dataclasses import dataclass
from uuid import UUID

from application.dto.paginator import LimitOffsetPaginator


@dataclass(slots=True, frozen=True)
class UpdateTenantCommand:
    initiator_id: UUID
    tenant_id: UUID
    status: str


@dataclass(slots=True, frozen=True)
class GetTenantLastVersionQuery:
    initiator_id: UUID
    tenant_id: UUID


@dataclass(slots=True, frozen=True)
class ListTenantLastVersionsQuery:
    initiator_id: UUID
    paginator: LimitOffsetPaginator
    tenant_ids: list[UUID] | None
    statuses: list[str] | None
    states: list[str] | None
```

**Правила:**
- `@dataclass(slots=True, frozen=True)` — обязательно.
- Все поля публичные (без `_`).
- Чистый dataclass без методов поведения. Никаких `__post_init__` для валидации, никаких computed properties, никаких фабрик.
- Поля либо обязательные, либо `T | None`. Дефолтные значения через `field(default=...)` — допустимо для опциональных, но не как способ «угадать» отсутствующий ввод (лучше явный `None`).

## Допустимые типы полей

| Категория | Примеры |
|---|---|
| Скалярные примитивы | `UUID`, `int`, `str`, `bool`, `float`, `Decimal`, `bytes` |
| Дата/время (stdlib) | `datetime`, `date`, `time` |
| Контейнеры | `list[T]`, `dict[str, T]`, `tuple[T, ...]` |
| Опциональные | `T | None` |
| Application-level DTO | `LimitOffsetPaginator` и другие input-формы из `application/dto/` (без доменных ссылок и без поведения) |

**Запрещено в полях команд/запросов:**
- Доменные VO (`TenantID`, `TenantStatus` и т.п.) — они принадлежат domain, конверсия делается в `execute`.
- Доменные сущности (`Tenant`, `User` и т.п.).
- Выходные DTO (`TenantDTO` и т.п. — они описывают **выход** use case, не вход).
- Типы из presentation (pydantic-модели и подобное) или infrastructure.

## Конверсия в VO в use case, не в команде

Команда/запрос несёт **примитивы через границу слоёв**; преобразование в domain-форму — задача use case-а.

```python
# ✓ верно
@dataclass(slots=True, frozen=True)
class UpdateTenantCommand:
    initiator_id: UUID
    tenant_id: UUID
    status: str

# в execute:
tenant_id = TenantID(command.tenant_id)
status = TenantStatus.from_str(command.status)
```

```python
# ✗ неверно: VO в команде делает presentation зависимым от domain
@dataclass(slots=True, frozen=True)
class UpdateTenantCommand:
    initiator_id: UserID
    tenant_id: TenantID
    status: TenantStatus
```

## Что возвращает use case — никогда domain

Граница слоя симметрична: ни вход, ни выход не утечают domain-типов.

| ✓ Разрешено | ✗ Запрещено |
|---|---|
| `-> TenantDTO` | `-> Tenant` (domain entity) |
| `-> tuple[list[TenantDTO], int]` | `-> list[Tenant]` |
| `-> None` | `-> TenantID` (VO тоже domain) |
| | `-> User` |

Если presentation нужен только id созданного объекта — он получает его через DTO (`dto.tenant_id`), а не `TenantID` напрямую.

## Контракт `execute` — 0 или 1 аргумент

`execute` принимает **либо одну** команду/запрос, **либо ничего**. Никогда несколько.

```python
# ✓ команда/запрос на входе
async def execute(self, command: UpdateTenantCommand) -> TenantDTO: ...

# ✓ без входа (publisher use case)
async def execute(self) -> None: ...

# ✗ запрещено
async def execute(self, command: XxxCommand, extra: SomethingElse) -> ...: ...
```

Следствие: вся вариативность входа умещается в полях одного dataclass-а. Если входов «логически два» (например, command + контекст вызова) — оба поля живут внутри одной команды.

## Command vs Query — таблица

| Аспект | Command | Query |
|---|---|---|
| Цель | мутация состояния | чтение состояния |
| Return type | `DTO` или `None` | `DTO` или `tuple[list[DTO], int]` |
| Side effects | да: `read.save` / `version.save` / `outbox.save` | нет |
| Авторизация (public) | `raise_admin` или иное mutating-право | `raise_reader` |
| Per-aggregate authorization | `.edit(initiator, [primary])` | `.read(initiator, items)` |
| Расположение | `command/{public,private}/<aggregate>/<action>.py` | `query/<aggregate>/<action>.py` |
| Suffix dataclass | `Command` | `Query` |
| Verb form | императив (`Create`, `Update`, ...) | `Get` / `List` |
| Транзакционная граница | один UoW для согласованных изменений | без UoW, если транзакция не нужна |

Определяй вид сценария по эффекту, а не по используемому порту или расположению файла:

- Command изменяет состояние системы или инициирует прикладной побочный эффект.
- Query читает или вычисляет данные без изменения состояния.
- Обращение к JWT, внешнему API или другому read-only порту само по себе не превращает Query в Command.

Называй запрос существительным с `Get`/`List`: `GetClaimsQuery`, а не `GettingClaimsCommand`.

## Выбор зависимостей

- Инжектируй `UnitOfWork`, когда нужны атомарные изменения нескольких репозиториев.
- Инжектируй специализированный порт напрямую для одного ресурса или stateless-операции.
- Не открывай UoW в Query без транзакционной потребности.
- Не вводи базовый use case-класс только ради единственной зависимости; выноси её при реальном переиспользовании.

## Helper `_filtering_data` для list-запросов

**Назначение:** конверсия опциональных фильтров запроса → kwargs для `repository.filters(...)`. Без helper-а получается портянка `if not None:` внутри `execute`.

**Где живёт:** приватный метод **самого use case-а**, не в базовом классе и не в отдельном модуле. Helper специфичен для каждого list-запроса.

```python
from typing import Any


class ListTenantLastVersionsUseCase(BaseUseCase):
    async def execute(
        self, query: ListTenantLastVersionsQuery
    ) -> tuple[list[TenantDTO], int]:
        action = "получение последних версий арендаторов"
        initiator_id = UserID(query.initiator_id)
        async with self._uow as uow:
            initiator = await self._initiator(uow, initiator_id, action)
            initiator.raise_reader()
            filtering_data = self._filtering_data(query)
            tenants, count = await uow.tenant_repositories.read.filters(
                **filtering_data
            )
            TenantPolicyService().read(initiator, tenants)
            return [
                TenantDTO.from_domain(tenant) for tenant in tenants
            ], count

    def _filtering_data(
        self, query: ListTenantLastVersionsQuery
    ) -> dict[str, Any]:
        filtering_data: dict[str, Any] = dict()
        if query.tenant_ids is not None:
            filtering_data["tenant_ids"] = [
                TenantID(tid) for tid in query.tenant_ids
            ]
        if query.statuses is not None:
            filtering_data["statuses"] = [
                TenantStatus.from_str(s) for s in query.statuses
            ]
        if query.states is not None:
            filtering_data["states"] = [State.from_str(s) for s in query.states]
        filtering_data["paginator"] = query.paginator
        return filtering_data
```

**Когда нужен:**
- 3+ опциональных фильтра — обязательно.
- 1–2 опциональных — можно inline в `execute`, но без потери читаемости (если получается громоздко — выноси).

**Принцип:** helper отвечает только за **сборку фильтров**. Никакой бизнес-логики, никаких обращений к репо, никакой авторизации в нём не место.
