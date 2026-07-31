# Чеклист

## Контракт

- [ ] Method, path, media type, status, headers и body заданы требованиями.
- [ ] Application entry point и публичные input/result/error существуют.
- [ ] Один запрос вызывает одну application-операцию.
- [ ] Не добавлены новые business outcomes.

## Модели и преобразования

- [ ] Request/response models независимы от application DTO.
- [ ] Модели immutable, extra fields запрещены по умолчанию.
- [ ] Transport validation не содержит I/O и business rules.
- [ ] Каждое поле преобразовано явно.
- [ ] Application DTO не возвращается напрямую.
- [ ] Serialization сложных типов соответствует контракту.

## Обработчик

- [ ] Entry point получен узкой типизированной dependency.
- [ ] Caller identity извлечена и передана application.
- [ ] Unit of Work/repository отсутствуют.
- [ ] ID нового aggregate не генерируется в presentation.
- [ ] Нет общего catch/log/re-raise.
- [ ] Async endpoint не выполняет blocking I/O.

## Внешнее поведение

- [ ] Status, response model и headers заданы явно.
- [ ] `204` не содержит JSON body.
- [ ] `Location` использует ID application-результата.
- [ ] PATCH различает omission, `null` и значение.
- [ ] Pagination/filter/sort закрыты контрактом.
- [ ] Idempotency и conditional headers не реализованы скрыто.
- [ ] Error body не раскрывает внутренние данные.

## Маршрутизатор и OpenAPI

- [ ] Route/operation IDs уникальны и стабильны.
- [ ] Статические и параметризованные пути не конфликтуют.
- [ ] Версии имеют отдельные transport contracts.
- [ ] OpenAPI описывает реальные success/error outcomes.
- [ ] Internal routes не утекли в публичную схему.

## Проверки

- [ ] Parsing, mapping, result и errors проверены интеграционно.
- [ ] Проверены extra, missing, null и граничные значения.
- [ ] Entry point вызван ровно один раз.
- [ ] Пагинация содержит пустой и непустой пример.
- [ ] Публично значимая OpenAPI проверена точечно.
