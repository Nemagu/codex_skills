# Fixture and Factory Patterns

## Decision Tree

1. Нужен объект только в одном тесте?
- Создай локально в тесте.

2. Нужен в нескольких тестах одного файла?
- Создай fixture в этом файле.

3. Нужен в нескольких файлах одного пакета тестов?
- Вынеси fixture в ближайший `conftest.py`.

4. Нужны варианты объекта с разными параметрами?
- Создай factory-функцию внутри fixture.

5. Начинается дублирование factory между модулями?
- Вынеси factory в общий `conftest.py` соответствующего уровня.

## Factory Pattern Example

```python
import pytest

@pytest.fixture
def user_factory() -> callable:
    def _make(*, name: str = "user", active: bool = True) -> User:
        return User(name=name, active=active)
    return _make
```

## Practical Rules

- Фабрика изменяет только значимые для сценария поля.
- Значения по умолчанию в фабрике должны создавать валидный объект.
- Не прячь в фабрике случайность, если это не проверяется тестом явно.
