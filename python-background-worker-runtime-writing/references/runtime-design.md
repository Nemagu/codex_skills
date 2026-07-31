# Конструкция runtime

## Минимальные роли

```python
@dataclass(frozen=True, slots=True)
class WorkerTaskSpec:
    name: str
    run: WorkerTask


@dataclass(frozen=True, slots=True)
class RuntimeOptions:
    process_name: str
    heartbeat_interval_seconds: float
    shutdown_timeout_seconds: float
```

`WorkerTask` принимает готовый контекст и `asyncio.Event`. Конкретный тип контекста определяется composition root.

## Последовательность запуска

1. Runner создаёт stop event и устанавливает signal handlers.
2. `AsyncExitStack` последовательно подготавливает обязательные ресурсы.
3. Формируется неизменяемый runtime-контекст.
4. В одном `TaskGroup` запускаются heartbeat и все обязательные задачи.
5. Stop event просит задачи закончить работу.
6. Ошибка задачи отменяет соседние задачи.
7. После выхода из `TaskGroup` ресурсы закрываются в обратном порядке.
8. Внешняя граница классифицирует завершение и возвращает exit code.

## Обязательные тесты

| Сценарий | Ожидаемое поведение |
|---|---|
| Успешный startup | Задачи видят полностью готовый контекст |
| Ошибка ресурса | Задачи не запущены, созданные ресурсы закрыты |
| Ошибка задачи | Соседние задачи отменены |
| Первый сигнал | Установлен stop event, начат graceful shutdown |
| Повторный сигнал | Выполнена немедленная отмена |
| Shutdown timeout | Процесс завершён с ненулевым кодом |
| Heartbeat error | Runtime завершается fail-fast |
| Ожидание интервала | Stop event прерывает ожидание |
