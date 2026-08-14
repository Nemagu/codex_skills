# Частичные обновления и идемпотентность

## Содержание

- [Три состояния PATCH](#три-состояния-patch)
- [Полная замена](#полная-замена)
- [Ключ идемпотентности](#ключ-идемпотентности)
- [Транспортный тайм-аут](#транспортный-тайм-аут)
- [Условные заголовки](#условные-заголовки)
- [Проверки](#проверки)

## Три состояния PATCH

Pydantic сохраняет множество фактически переданных полей в
`model_fields_set`. Mapper обязан различить omission, `null` и значение:

```python
def to_update_project_command(
    project_id: str,
    request: UpdateProjectRequest,
    caller: CallerIdentity,
) -> UpdateProjectCommand:
    name = (
        request.name
        if "name" in request.model_fields_set
        else UNSET
    )
    description = (
        request.description
        if "description" in request.model_fields_set
        else UNSET
    )
    return UpdateProjectCommand(
        initiator_id=caller.subject_id,
        project_id=project_id,
        name=name,
        description=description,
    )
```

Если `None` запрещён конкретному полю, transport validation отклоняет его.

Не использовать `if request.name`, потому что пустая строка, `0` и `False` могут
быть валидными значениями. Не полагаться на default `None`: он стирает различие
между omission и явным `null`.

## Полная замена

Для PUT реализовывать полную замену только если это задано контрактом. Не
переносить PATCH-семантику в PUT автоматически.

Клиентский aggregate ID допустим, если он является частью PUT-контракта.
Application получает его как внешний вход и не генерирует другой ID.

## Ключ идемпотентности

Transport-слой:

- извлекает ключ из заданного header;
- проверяет формат, длину и обязательность;
- передаёт значение application-команде;
- не выполняет lookup и не хранит response;
- не повторяет application-вызов.

```python
def to_command(
    request: CreatePaymentRequest,
    caller: CallerIdentity,
    idempotency_key: str,
) -> CreatePaymentCommand:
    return CreatePaymentCommand(
        initiator_id=caller.subject_id,
        idempotency_key=idempotency_key,
        amount=request.amount,
    )
```

Application определяет:

- что входит в значимый fingerprint операции;
- что возвращать при точном повторе;
- что делать при совпавшем ключе и другом входе;
- срок и границу хранения результата.

## Транспортный тайм-аут

Timeout означает, что клиент не получил ответ. Он не доказывает rollback или
неуспех application-операции.

- Не запускать скрытый retry изменяющей операции.
- Возвращать результат повтора согласно application outcome.
- Не генерировать новый idempotency key вместо клиентского.
- Не генерировать aggregate ID в presentation для стабилизации повтора.

## Условные заголовки

`If-Match` и ETag применять только по требованиям:

```text
If-Match -> parse expected version -> application command
application conflict -> public error -> HTTP 412 (если так задано)
```

Не сравнивать актуальную версию в endpoint и не обращаться к repository.
Разобрать только допустимый формат ETag, включая правила weak/strong comparison,
если они являются частью контракта.

## Проверки

- Omission, `null` и значение дают разные application inputs.
- False/zero/empty value не теряются.
- Idempotency key передаётся без скрытого хранения.
- Один HTTP-запрос вызывает application entry point один раз.
- Conditional header корректно преобразуется или отклоняется.
