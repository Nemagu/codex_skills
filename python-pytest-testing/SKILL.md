---
name: python-pytest-testing
description: Используй при проектировании, написании или проверке автотестов на Python через pytest и pytest-cov. Триггеры — добавление/правка unit и integration тестов, организация фикстур и фабрик, параметризация через pytest.mark.parametrize, маркировка интеграционных тестов и поддержка двух режимов поднятия инфраструктуры (локальный docker compose vs внешняя инфра, готовая снаружи).
---

# Python Pytest Testing

## Quick Start

1. Определи тип проверки: unit, integration или смешанный сценарий.
2. Сформируй список проверяемых контрактов и инвариантов до написания кода тестов.
3. Выбери структуру фикстур и фабрик; исключи дублирование через общий `conftest.py` на минимально подходящем уровне.
4. Напиши тесты с явными входными данными и наблюдаемым результатом.
5. Объедини схожие кейсы через `pytest.mark.parametrize` и добавь `ids`.
6. Маркируй интеграционные тесты `@pytest.mark.integration`, чтобы их можно было запускать прицельно через `-m integration`.
7. Запускай тесты через `uv run pytest`. Удали неиспользуемые фикстуры, стабилизируй flaky-поведение, зафиксируй итоговый DoD.

## Running Tests

Канонический раннер — `uv run pytest`. Никаких обёрток поверх него скил не требует.

```bash
# все тесты
uv run pytest

# только unit (исключение интеграционных делаем директорией, без -m)
uv run pytest src/tests/units

# только integration по маркеру
uv run pytest -m integration

# coverage с отчётом по строкам
uv run pytest --cov=src --cov-report=term-missing

# конкретный кейс
uv run pytest src/tests/units/domain/test_project.py::test_state_transition
```

> Не передавай в флаги значения с пробелами (например, `-m "not integration"`). Это вынуждает повсюду ставить кавычки и рвётся в скриптах. Используй односложные маркеры (`-m integration`) и директории (`src/tests/units`) для исключения.

Если в окружении нет `uv`, замени префикс на `pytest` напрямую.

## External Infrastructure Mode

Скил не знает о конкретной CI-системе. Знает только, что фикстуры могут запускаться в окружении, где инфраструктура (Postgres, NATS и т. п.) уже поднята снаружи. В таком окружении фикстуры **не должны** пытаться поднимать docker compose.

Канонический переключатель — переменная окружения `INTEGRATION_USE_EXTERNAL_INFRA=1`:

- `INTEGRATION_USE_EXTERNAL_INFRA` не задана или `=0` — фикстура поднимает инфраструктуру локально (например, через `docker compose`).
- `INTEGRATION_USE_EXTERNAL_INFRA=1` — фикстура читает параметры подключения из переменных окружения и не запускает свои контейнеры.

Имена переменных подключения по умолчанию используют префикс `INTEGRATION_*` (`INTEGRATION_PG_HOST`, `INTEGRATION_PG_PORT`, `INTEGRATION_NATS_HOST`, …). Это явно отделяет тестовые подключения от рантайм-настроек приложения.

> **Перед применением:** если в проекте уже есть устоявшаяся конвенция имён для тестовых переменных подключения, используй её. Иначе спроси пользователя, какие имена принять, прежде чем закреплять их в фикстурах.

Полный пример двухрежимной фикстуры — в [config_and_infrastructure.md](references/config_and_infrastructure.md).

## Workflow

### 1. Classify Test Scope

- Помечай тест как `unit`, если внешние зависимости заменены doubles/fakes и проверяется локальный контракт.
- Помечай тест как `integration`, если проверяется связка модулей или реальная инфраструктура.
- Не смешивай в одном тесте unit и integration цели.
- Все интеграционные тесты помечай маркером `@pytest.mark.integration` (и регистрируй маркер в `pyproject.toml`, если включён `--strict-markers`).

### 2. Design Cases Before Implementation

- Определи минимум: happy path, граничные условия, невалидные входы, отказоустойчивость.
- Формулируй поведение через контракт: вход, действие, ожидаемый эффект.
- Для багфикса сначала добавляй воспроизводящий тест, затем исправляй код.

### 3. Build Fixtures and Factories

- Держи фикстуры узкими по ответственности.
- Выноси общие фикстуры в `conftest.py` только при реальном переиспользовании.
- Создавай фабрики для агрегатов/команд/DTO, чтобы тест не зависел от лишних полей.
- Для интеграционной инфраструктуры используй session-scoped фикстуру + autouse function-scoped очистку данных. Подробности — в [config_and_infrastructure.md](references/config_and_infrastructure.md).
- Если интеграция трогает message broker с глобальным неймспейсом (NATS streams, Kafka topics, RabbitMQ exchanges) — не хардкодь их имена в тестах и фикстурах: рандомизируй per-session суффиксом через config, иначе параллельные прогоны на shared-брокере дают плавающие падения. См. секцию «Hermetic Streams/Topics» в [config_and_infrastructure.md](references/config_and_infrastructure.md).
- Decision tree: [fixture_and_factory_patterns.md](references/fixture_and_factory_patterns.md).

### 4. Parameterize Similar Scenarios

- Используй `pytest.mark.parametrize` для повторяемых сценариев с одинаковой структурой шагов.
- Всегда задавай `ids`, чтобы названия тестов отражали смысл кейса.
- Пример паттерна смотри в [pytest_best_practices.md](references/pytest_best_practices.md).

### 5. Validate and Harden

- На итерации запускай узкий target (`uv run pytest src/tests/units/...` или конкретный файл/кейс).
- Перед завершением запускай полный набор и `--cov`.
- Проверяй стабильность: тест не должен зависеть от порядка запуска и времени выполнения.

## Definition of Done

- Есть тесты на ключевые контракты и регрессионные риски.
- Схожие сценарии параметризованы и читаемы по `ids`.
- Фикстуры и фабрики не дублируются между модулями.
- Интеграционные тесты явно отделены от unit-тестов директорией и маркером `@pytest.mark.integration`.
- Интеграционные фикстуры корректно работают в двух режимах: локальное поднятие инфраструктуры и `INTEGRATION_USE_EXTERNAL_INFRA=1` (инфра поднята снаружи).
- Если тесты используют message broker — имена стримов/топиков/очередей уникальны per сессия (per-session суффикс), а ассерты ожидаемых subject-ов выводятся из настроек, а не хардкодятся.
- Тестовый набор проходит локально через `uv run pytest`; метрики покрытия соответствуют целям команды (порог не обязателен на уровне CI — это ориентир, к которому стремимся).

## References

- Практики и анти-паттерны: [pytest_best_practices.md](references/pytest_best_practices.md)
- Стратегия покрытия: [coverage_strategy.md](references/coverage_strategy.md)
- Decision tree для фикстур и фабрик: [fixture_and_factory_patterns.md](references/fixture_and_factory_patterns.md)
- Конфиг-файлы и инфраструктура в тестах (включая режим внешней инфры): [config_and_infrastructure.md](references/config_and_infrastructure.md)
