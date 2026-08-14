# Маршрутизаторы и OpenAPI

## Группировка

Группировать по внешним ресурсам или API-возможностям:

```text
presentation/api/
└── public/
    └── v1/
        └── project/
            ├── router.py
            ├── model.py
            ├── mapping.py
            └── dependency.py
```

Агрегат domain не обязан совпадать с HTTP resource. Имена каталогов отражают
внешний контракт.

## Фабрика маршрутизатора

Фабрика возвращает новый router и принимает только сборочные зависимости:

```python
def create_project_router(
    dependencies: ProjectHTTPDependencies,
) -> APIRouter:
    router = APIRouter(
        prefix="/projects",
        tags=["projects"],
    )
    # register endpoints
    return router
```

`ProjectHTTPDependencies` неизменяем и содержит dependency callables/mappers, но
не runtime clients и не общий service locator.

Если фабрика ухудшает существующую простую структуру, сохранить статический
router с узкими dependencies.

## Версии

- V1 и V2 имеют отдельные request/response models.
- Mappings версии переводят их в существующие application contracts.
- Общую application-операцию не дублировать.
- Breaking HTTP change не скрывать внутри прежней модели.
- Deprecated endpoint помечать согласно требованиям.

## Маршруты

- Operation ID задавать стабильно и уникально.
- Route name использовать осмысленно для reverse lookup.
- Статический `/projects/search` не должен перехватываться
  `/{project_id}`.
- Prefix определять на одном уровне и не дублировать.
- Slash policy соблюдать единообразно.

## OpenAPI

Для операции явно фиксировать заданные:

- status code;
- response model;
- public error responses;
- headers;
- tags;
- summary/description;
- deprecated;
- operation ID.

Не добавлять error response, которого операция не возвращает. Не включать
внутренние исключения и application DTO в schemas.

## Проверка уникальности

После сборки приложения проверить:

- дубликаты method + path;
- дубликаты operation ID;
- конфликтующие route names, если они используются;
- соответствие фактического status/response схеме;
- отсутствие внутренних routes в публичной OpenAPI.

Snapshot всей схемы допустим только если команда готова поддерживать шумные
изменения. Предпочитать точечные assertions публично значимых операций и schemas.

## Рост структуры

Разделить `model.py` на `model/request.py`, `model/response.py` и `model/common.py`,
а `mapping.py` на input/output, когда файл стал большим или группы меняются
независимо. Не создавать такую иерархию заранее для двух моделей.
