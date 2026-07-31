# Конструкция publisher-адаптера

## Поток данных

```text
DTO исходящего порта
        ↓
явный реестр маршрутов
        ↓
чистый mapper в transport-модель
        ↓
детерминированный serializer
        ↓
subject + headers + bytes
        ↓
JetStream publish
        ↓
publish acknowledgement
```

## Минимальные роли

```python
@dataclass(frozen=True, slots=True)
class PublishRoute:
    stream: str
    subject: str
    mapper: PayloadMapper
    serializer: PayloadSerializer


type RouteRegistry = Mapping[type[PublishDTO], PublishRoute]
```

Конкретные протоколы, DTO и ошибки бери из проекта. Реестр должен быть неизменяемым после сборки.

## Ошибки

| Этап | Категория |
|---|---|
| Неизвестный DTO | Невосстановимая ошибка контракта |
| Недостаточные metadata | Невосстановимая ошибка контракта |
| Ошибка mapping/валидации | Невосстановимая ошибка сообщения |
| Некорректный маршрут | Ошибка конфигурации startup |
| Недоступный JetStream | Временная ошибка публикации |
| Потерянный PubAck | Неопределённый результат публикации |
| Несовместимый stream | Ошибка топологии startup |

Используй классы существующего порта. Не создавай инфраструктурные исключения, которые должен понимать application-слой.

## Проверяемый публичный сценарий

1. Подготовить адаптер и топологию.
2. Передать DTO порта в `publish`.
3. Получить сообщение из настоящего JetStream.
4. Проверить subject, headers и декодированный payload.
5. Убедиться, что метод завершился только после подтверждения.
