# DTO

## Категории

| Категория | Назначение | Допустимые данные |
|---|---|---|
| Command / Query | Публичный вход application-операции | примитивы, stdlib, application DTO |
| Публичный DTO | Публичный результат операции | примитивы, stdlib, application DTO |
| Port DTO | Самодостаточный контракт исходящего порта | также domain-объекты и VO |

Реализуй ровно заданные поля. Не добавляй сведения, которые операция получает
через порт, transport metadata и поля «на будущее».

## Глубокая неизменяемость

Каждый DTO оформляй как `@dataclass(slots=True, frozen=True)`. `frozen=True` не
делает неизменяемыми вложенные `list` и `dict`, поэтому:

- коллекции храни в `tuple` или `frozenset`;
- структурированные вложения оформляй отдельными frozen DTO;
- не изменяй входной DTO внутри use case;
- выходные коллекции возвращай как `tuple`.

Для `limit/offset` с общим количеством используй
`tuple[tuple[DTO, ...], int]`; пустой результат — `((), 0)`. Другую форму
пагинации реализуй буквально и не добавляй total без требования.

## Частичное обновление

Если `None` означает явную очистку, отличай его от отсутствующего поля:

```python
from enum import Enum, auto


class Unset(Enum):
    VALUE = auto()


UNSET = Unset.VALUE
```

Поле имеет тип `T | None | Unset`. Входящий адаптер преобразует отсутствие в
`UNSET`, явный null — в `None`. Не используй truthiness для различения вариантов.

## Публичные DTO

- Не используй domain entity и VO, Pydantic-модели и типы адаптеров.
- Не добавляй HTTP-, NATS- и serialization-поведение.
- Называй DTO по смыслу результата, а не `OutputDTO`, `ResponseDTO` или слою.
- Используй `from_domain()` только для устойчивого чистого преобразования.
- Если представлений несколько, используй уже заданные различимые имена.

## Port DTO

Port DTO должен содержать минимально достаточный набор данных, чтобы адаптер
выполнил операцию без дополнительного обращения к агрегату или другому порту.
Domain-типы допустимы, поскольку адаптер реализует application-порт.

Для сохранения версии предпочитай два неизменяемых контракта:

```python
@dataclass(slots=True, frozen=True)
class ProjectSnapshotDTO:
    project_id: ProjectID
    status: ProjectStatus
    version: Version


@dataclass(slots=True, frozen=True)
class ProjectVersionRecordDTO:
    snapshot: ProjectSnapshotDTO
    change: ProjectChange
    editor_id: MemberID | None
```

Snapshot DTO фиксирует VO и неизменяемые коллекции в конкретной точке и не хранит
ссылку на изменяемый агрегат. Outbox DTO также самодостаточен для сохранения:
включай ID, версию, тип изменения, редактора, publication ID и другие заданные
метаданные. Не включай subject, stream, payload и headers без application-смысла.

Размещай переиспользуемые контракты в `application/port/dto/<meaning>.py`, а
контракт единственного порта — рядом с ним.

## Запреты

- Изменяемые коллекции внутри frozen DTO.
- Domain-типы в публичных Command, Query и result DTO.
- Transport-валидация, `to_dict`, `to_json`, `model_dump`.
- DTO, хранящий изменяемый агрегат как снимок нескольких будущих версий.
- Техническое имя вместо смысла данных.
