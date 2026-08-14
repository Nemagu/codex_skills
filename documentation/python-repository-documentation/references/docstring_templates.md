# Docstring Templates

## Short Form

```python
def activate(self) -> None:
    """Активирует подписку и обновляет ее версию."""
```

## With Args

```python
def parse_period(value: str) -> Period:
    """Преобразует строковый период в доменный объект.

    Args:
        value: Строка формата YYYY-MM.
    """
```

## With Raises

```python
def apply(command: ChangePlanCommand) -> None:
    """Применяет изменение плана к агрегату.

    Raises:
        DomainError: Если переход состояния недопустим.
    """
```

## With Returns

```python
def build_snapshot() -> tuple[int, list[Item]]:
    """Возвращает номер версии и отсортированный список элементов.

    Returns:
        Кортеж из версии состояния и списка элементов.
    """
```

## __init__ Focused

```python
class Service:
    def __init__(self, repository: Repository, clock: Clock) -> None:
        """Args:
            repository: Репозиторий для работы с агрегатами.
            clock: Источник текущего времени.
        """
```
