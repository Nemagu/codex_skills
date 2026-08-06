---
name: python-fastapi-http-endpoint-writing
description: "Используй при реализации или правке обычных HTTP endpoint-ов на FastAPI: routers, внешних Pydantic request/response-моделей, path/query/header/cookie параметров, transport validation, caller identity, явных mappings в публичные application-входы и результаты, пагинации, идемпотентности и OpenAPI. Не применять для WebSocket/SSE, сборки API-процесса, application/domain-логики, Unit of Work и исходящих адаптеров."
---

# HTTP-обработчики на FastAPI

## Порядок работы

1. Извлечь точный HTTP-контракт и публичную application-операцию.
2. Изучить существующую структуру routers, models, mappings и dependencies.
3. Реализовать независимые transport request/response-модели.
4. Реализовать явные mappings между HTTP и application-контрактами.
5. Собрать endpoint как одну последовательность transport-шагов.
6. Подключить идентичность, параметры, status, headers и error outcomes.
7. Обновить router/OpenAPI и интеграционные проверки.

Не задавать повторных вопросов, если требования однозначны. Остановиться и
уточнить только противоречие, небезопасное решение или отсутствующий выбор,
меняющий внешний контракт.

## Граница ответственности

Скил реализует обычное HTTP request/response взаимодействие. Он не определяет:

- новые application-команды, запросы, результаты и ошибки;
- бизнес-валидацию, авторизацию и транзакционные границы;
- Unit of Work, repositories и исходящие адаптеры;
- фабрику FastAPI-приложения, lifespan, middleware и Uvicorn;
- WebSocket, SSE и общий streaming protocol;
- отсутствующую бизнес-семантику.

Presentation зависит только от публичных application entry points и контрактов.
Не импортировать domain и конкретные infrastructure-реализации. Если нужного
application-контракта нет, сообщить о блокере, а не создавать его внутри
presentation.

## Одна операция

Один HTTP-запрос вызывает одну публичную application-операцию:

```text
HTTP input
  -> transport validation
  -> caller identity
  -> explicit input mapping
  -> one application call
  -> explicit output mapping
  -> HTTP response
```

Не оркестрировать несколько use case-ов, не открывать транзакцию и не передавать
Unit of Work в endpoint. Составной бизнес-процесс должен быть представлен одной
application-операцией.

## Транспортные модели

- Request/response-модели являются самостоятельным внешним контрактом.
- Не наследовать их от application DTO и не передавать внутрь application.
- Не возвращать application DTO напрямую.
- Использовать immutable Pydantic models и `extra="forbid"` по умолчанию.
- Настраивать aliases, strict/coercion и serialization по HTTP-контракту.
- Не использовать `model_dump()`/`**dto` как mapping между слоями.
- Не использовать `from_attributes=True` для скрытого чтения domain/DTO.

Не дублировать domain-инварианты в validators. Pydantic проверяет transport
формат, наличие, диапазон, длину и внешние cross-field ограничения без I/O.

Подробности: [модели и преобразования](references/models_and_mapping.md).

## Преобразования

По умолчанию размещать преобразования в отдельных чистых функциях:

- request + path/query/header/identity → application input;
- application result → response model;
- application public error → централизованный внешний outcome.

Transport-модель не должна знать application-классы. Метод-фабрика модели
допустим только для простого локального mapping, если это устойчивая конвенция
проекта.

Каждое поле сопоставлять явно. Не переносить внутренние поля в ответ и не
заполнять application-вход данными, которых нет в требованиях.

## Обработчик и зависимости

Endpoint:

- получает уже разобранные transport-данные и типизированную caller identity;
- получает конкретный application entry point через `Depends` или router factory;
- вызывает entry point один раз;
- не содержит общего `try/except` и не логирует перед `raise`;
- не читает runtime context, headers или settings вручную, если для этого есть
  dependency;
- завершает application-операцию до успешного ответа.

Не передавать endpoint-у общий service locator. Создавать узкие типизированные
dependencies по операциям или возможностям. Stateless application handler можно
разделять между запросами; request-scoped экземпляр создавать только по
требованию.

Подробности: [endpoint и зависимости](references/endpoint_and_dependencies.md).

## Идентичность

- Извлечь credential или trusted identity из заданного header/cookie.
- Проверить только transport-формат и обязательность.
- Делегировать проверку credential предусмотренному authentication-компоненту.
- Вернуть типизированную caller identity, а не сырой header.
- Передать identity в application input.
- Не проверять роли, membership, tenant ownership и другие правила авторизации
  в endpoint.
- Доверять identity header только на явно заданной доверенной границе.

## Идентификатор нового агрегата

Presentation не генерирует ID нового агрегата и не включает его в create input,
если ID не является частью внешней бизнес-семантики. Application-операция
получает ID через собственный типизированный порт и возвращает созданный ID в
публичном результате.

Endpoint использует возвращённый ID для response и `Location`. Клиентский ID
передавать только по контракту, например в идемпотентном `PUT`.

## Параметры и частичное обновление

- Не определять одно поле противоречиво в path, query, header и body.
- Имена, defaults и ограничения параметров брать из HTTP-контракта.
- Для PATCH различать отсутствующее поле, `null` и значение.
- Преобразовать их явно в application `UNSET | None | value`.
- Не использовать truthiness и не подменять отсутствие default-значением.
- Не использовать `model_dump(exclude_unset=True)` как межслойный контракт.

Подробности: [частичные обновления и идемпотентность](references/partial_updates_and_idempotency.md).

## Результат HTTP

- Status, наличие body и headers брать из внешнего контракта.
- Не выводить status автоматически из имени команды.
- Не возвращать `200` с `null`, если предусмотрен `204`.
- Формировать `Location` из ID application-результата.
- Объявлять точную response model, кроме заданных `204`, file или streaming
  responses.
- Сериализовать UUID, enum, datetime, decimal и bytes в заданном формате.
- Не отключать response validation ради несовместимого результата.

`ETag`, `If-Match`, `If-None-Match` и version headers реализовывать только по
контракту. Presentation разбирает условие и передаёт минимальный application
вход; optimistic locking остаётся во внутренних слоях.

## Пагинация, фильтры и сортировка

- Преобразовать query-параметры в отдельный application query/DTO.
- Не передавать Pydantic-модель в application.
- Не смешивать limit/offset и cursor без требования.
- Response pagination содержит только заданные items, total, cursor, limit,
  offset или links.
- Представлять sort/filter fields закрытыми внешними наборами.
- Не принимать произвольные DB column names.
- Пустую коллекцию не превращать в `404`, если это не задано.

Подробности: [пагинация и фильтрация](references/pagination_and_filters.md).

## Идемпотентность и повторы

Idempotency key поддерживать только по контракту:

- извлечь и валидировать ключ;
- передать его application-операции;
- не хранить результат и не определять семантику повтора в presentation;
- не повторять изменяющий application-вызов автоматически;
- не считать transport timeout доказательством неуспешной операции.

Одинаковый ключ с другим значимым входом обрабатывается application-операцией
согласно заданному публичному исходу.

## Маршрутизаторы и версии

Группировать routers по внешнему ресурсу или API-возможности, не обязательно по
доменному агрегату. Версии API имеют отдельные transport-модели и mappings, но
могут вызывать один application entry point.

- Делать route names и OpenAPI operation IDs стабильными и уникальными.
- Учитывать порядок статических и параметризованных путей.
- Разделять public/internal/admin только при различии контрактов или доступа.
- Не создавать пустые уровни каталогов.
- Сохранять существующую структуру, если она обеспечивает границы.

Router factory — рекомендуемый способ явной сборки. Она принимает типизированные
dependency callables/mappers и возвращает новый `APIRouter`, но не создаёт
application handlers и адаптеры. Статический router допустим при эквивалентной
изоляции и поддержке dependency overrides.

Подробности: [routers и OpenAPI](references/routers_and_openapi.md).

## Типы содержимого и крупные тела

- Принимать только предусмотренные media types.
- Не интерпретировать произвольный текст как JSON.
- Применять body limits на согласованной server/proxy границе.
- Обрабатывать крупные payload/file uploads потоково по требованиям.
- Считать filename, content type и checksum недоверенными.
- Не передавать `Request`, `UploadFile` или stream в application, если контракт
  ожидает извлечённые данные.
- Проектировать streaming response отдельно: после начала отправки обычный error
  response невозможен.

## Асинхронность

- Использовать `async def` для асинхронного application entry point.
- Не выполнять блокирующий I/O в event loop.
- Не создавать лишние tasks вокруг одного вызова.
- Не использовать `BackgroundTasks` для надёжной бизнес-операции.
- Не сообщать успех до завершения application-вызова.

## Ошибки

- Публичные application-ошибки передавать централизованному HTTP mapping.
- Не перехватывать domain/infrastructure errors в endpoint.
- `HTTPException` или presentation error использовать только для transport
  отказа.
- Не собирать одинаковый error body вручную в каждом router.
- Не включать internal detail и traceback во внешний ответ.

## OpenAPI

- Получать схему из реальных routers и transport-моделей.
- Явно задавать response model, status, error responses и нужные headers.
- Не полагаться на случайное имя функции как operation ID.
- Добавлять examples и deprecated только по требованиям.
- Скрывать health/internal routes по контракту.
- Проверять соответствие фактического ответа схеме.

## Структура

Начинать с `router.py`, `model.py`, `mapping.py`, `dependency.py` внутри внешней
API-возможности. Разделять на подпапки `router/`, `model/`, `mapping/`, когда:

- в нём более 8–10 endpoint-ов;
- command/query части стали устойчивыми группами;
- смешиваются несколько API-версий;
- независимые контракты регулярно меняют один большой файл.

Не выполнять массовое перемещение без необходимости.

## Тестирование

Проверять HTTP-контракт интеграционно через реальное FastAPI-приложение:

- parsing и validation каждого источника входа;
- request → application input mapping;
- application result/error → status, headers и body;
- omission, `null`, extra fields и границы;
- caller identity, pagination, idempotency и conditional headers;
- OpenAPI schema/operation IDs;
- отсутствие внутренних полей в ответе.

Заменять application entry points упрощёнными реализациями через dependency или
router factory. Не тестировать framework internals отдельно.

Подробности: [тестирование](references/testing.md).

## Антипаттерны

- Application DTO как FastAPI response model.
- Pydantic request внутри application.
- Автоматический `model_dump()` mapping.
- Domain imports в presentation.
- Unit of Work или repository в endpoint.
- Несколько application-вызовов на один HTTP-запрос.
- Генерация aggregate ID в presentation.
- Авторизация по ролям в router.
- I/O в Pydantic validator.
- Общий service locator в endpoint.
- Локальный catch/log/re-raise application error.
- Скрытая retry или idempotency логика.

## Критерии готовности

- HTTP-контракт реализован без расширения требований.
- Transport-модели независимы и строго валидируют внешний вход.
- Mappings явны и не пропускают лишние поля.
- Endpoint вызывает одну application-операцию.
- Identity и dependencies типизированы.
- Aggregate ID создаётся application-портом.
- Status, headers, errors и OpenAPI соответствуют контракту.
- Интеграционные проверки проходят.

## Материалы

- [Модели и преобразования](references/models_and_mapping.md)
- [Обработчик и зависимости](references/endpoint_and_dependencies.md)
- [Частичные обновления и идемпотентность](references/partial_updates_and_idempotency.md)
- [Пагинация и фильтрация](references/pagination_and_filters.md)
- [Маршрутизаторы и OpenAPI](references/routers_and_openapi.md)
- [Тестирование](references/testing.md)
- [Чеклист](references/checklists.md)
