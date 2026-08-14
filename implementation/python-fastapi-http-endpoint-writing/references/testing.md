# Тестирование

## Содержание

- [Уровень](#уровень)
- [Упрощённая точка входа](#упрощённая-точка-входа)
- [Матрица обработчика](#матрица-обработчика)
- [PATCH](#patch)
- [Пагинация](#пагинация)
- [OpenAPI](#openapi)
- [Инфраструктура](#инфраструктура)

## Уровень

Проверять endpoint интеграционно через реальное FastAPI-приложение. Подменять
только application entry point или его dependency, а не FastAPI/Pydantic.

Если правила проекта не требуют unit-тестов presentation wiring, не создавать
их отдельно.

## Упрощённая точка входа

Использовать записывающую реализацию:

```python
class RecordingCreateProject:
    def __init__(self, result: ProjectResult) -> None:
        self.calls: list[CreateProjectCommand] = []
        self._result = result

    async def __call__(
        self,
        command: CreateProjectCommand,
    ) -> ProjectResult:
        self.calls.append(command)
        return self._result
```

В production-коде коллекции остаются immutable; изменяемый список здесь является
тестовым журналом вызовов.

## Матрица обработчика

Проверить:

- минимальный успешный запрос;
- полный успешный запрос;
- отсутствие каждого обязательного входа;
- неверный transport type/format;
- extra body field;
- границы длины/диапазона;
- caller identity;
- точный application input;
- один вызов entry point;
- status, headers и точное JSON body;
- публичные application errors;
- безопасный unexpected error через общую API boundary.

Схожие случаи объединять параметризацией с понятными IDs согласно правилам
проекта.

## PATCH

Отдельно проверить:

- omission → `UNSET`;
- explicit `null` → `None`;
- `False`, `0` и `""` не теряются;
- значение передаётся без неявного default;
- пустой PATCH обрабатывается по контракту.

## Пагинация

Проверить пустую страницу и пример результата:

```json
{
  "items": [
    {
      "project_id": "project-1",
      "name": "Example",
      "version": 3
    }
  ],
  "total": 1,
  "limit": 20,
  "offset": 0
}
```

Поля и формат примера адаптировать к реальному контракту.

## OpenAPI

Точечно проверить:

- method/path;
- operation ID;
- request schema;
- success response schema/status;
- заданные error outcomes и headers;
- отсутствие скрытых/internal операций.

Не проверять внутреннее устройство генератора OpenAPI.

## Инфраструктура

Endpoint-тест с упрощённым application entry point не требует реальной БД.
Если проверяется полный процесс с реальными адаптерами, это отдельный
интеграционный сценарий с автоматически поднятой инфраструктурой согласно
правилам репозитория.
