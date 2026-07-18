# DTO

## Категории по направлению

Разделяй DTO по роли в процессе, а не по используемой технологии.

| Категория | Направление | Содержимое |
|---|---|---|
| Command / Query | presentation → application | Входные примитивы и stdlib-типы |
| Output DTO | application → presentation | Плоские данные без domain-ссылок |
| Port DTO | application ↔ adapters | Внутренний типизированный контракт; может содержать domain-объекты |

Command и Query описаны отдельно в `commands_and_queries.md` и не получают дополнительный суффикс `DTO`.

## Выходные DTO

Оформляй выходной DTO как неизменяемый снимок:

```python
from dataclasses import dataclass
from typing import Self
from uuid import UUID

from domain.user import User


@dataclass(slots=True, frozen=True)
class UserDTO:
    user_id: UUID
    external_user_id: int
    state: str
    version: int

    @classmethod
    def from_domain(cls, user: User) -> Self:
        return cls(
            user_id=user.user_id.user_id,
            external_user_id=user.external_user_id.user_id,
            state=user.state.value,
            version=user.version.version,
        )
```

Правила:

- Используй только примитивы, stdlib-типы и композицию других выходных DTO, когда она нужна сценарию.
- Не возвращай domain entity или VO в presentation.
- Оставляй только `from_*`-фабрики; не добавляй поведение, I/O и сериализацию.
- Размещай в `application/dto/<aggregate>.py` или `application/dto/<scenario>.py`.
- Оставляй `application/dto/__init__.py` пустым, если проект не использует публичную витрину DTO.

## Port DTO

Port DTO связывает use case с несколькими портами или адаптерами и может сохранять доменную форму данных:

```python
from dataclasses import dataclass

from domain.user import User, UserID


@dataclass(slots=True, frozen=True)
class UserEventDTO:
    user: User
    editor_id: UserID | None
    event: UserEvent
```

Используй port DTO, когда агрегат, событие и контекст образуют единый контракт для version repository, outbox или publisher. Формируй такой DTO один раз в use case и не реконструируй его независимо в адаптерах.

Размещение:

- `application/port/dto/<aggregate>.py` — если DTO используют два и более порта/адаптера;
- рядом с единственным портом — если отдельный пакет не даёт ценности.

Port DTO не является выходом application → presentation. Наличие domain-ссылок внутри него не нарушает границу слоя: infrastructure реализует application-порт и зависит от application/domain.

## Именование

Называй DTO по смыслу содержащихся данных:

- `UserDTO` — основное текущее представление пользователя;
- `UserVersionDTO` — исторический снимок;
- `UserEventDTO` — агрегат и контекст события;
- `ClaimsDTO` — результат самостоятельного сценария.

Не закрепляй обязательные `Simple`, `Detail`, `Full`, `Short` без явной системы парных представлений. Не используй технические имена `OutputDTO`, `PortDTO`, `RequestDTO`, `ResponseDTO`: направление видно из расположения и сигнатуры, а имя должно объяснять содержимое.

Если у сущности несколько одинаково естественных представлений, согласуй схему имён с пользователем до создания новых DTO и применяй её единообразно.

## Запреты

- Domain-типы в выходном DTO для presentation.
- Pydantic-модели и типы infrastructure/presentation в application DTO.
- Валидация входа, вычисление бизнес-значений, `to_dict`, `to_json`, `model_dump`.
- Мутация DTO после создания.
- Имена, описывающие транспорт или слой вместо смысла данных.
