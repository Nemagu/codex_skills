# Проверки состояния

## Жизнеспособность

Liveness отвечает на вопрос: «процесс и event loop способны обработать
запрос?».

- Не обращаться к БД, брокеру и внешним API.
- Не выполнять файловые записи.
- Возвращать минимальный стабильный ответ.
- Не включать сведения о конфигурации и зависимостях.

Пример формы определяется HTTP-требованиями:

```python
from fastapi import APIRouter, status
from fastapi.responses import JSONResponse


def create_liveness_router(path: str) -> APIRouter:
    router = APIRouter()

    @router.get(path, include_in_schema=False)
    async def liveness() -> JSONResponse:
        return JSONResponse(
            status_code=status.HTTP_200_OK,
            content={"status": "ok"},
        )

    return router
```

Не фиксировать `{"status": "ok"}`, если контракт задаёт другое тело.

## Готовность

Readiness отвечает на вопрос: «следует ли направлять процессу новый трафик?».

```python
from fastapi import Depends, status
from fastapi.responses import JSONResponse


@router.get(readiness_path, include_in_schema=False)
async def readiness(
    context: APIRuntimeContext = Depends(get_runtime_context),
) -> JSONResponse:
    if context.readiness.is_ready:
        return JSONResponse(
            status_code=status.HTTP_200_OK,
            content={"status": "ready"},
        )
    return JSONResponse(
        status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
        content={"status": "not_ready"},
    )
```

Readiness state создаётся lifespan и обновляется владельцами ресурсов. Endpoint
не запускает тяжёлую проверку каждого соединения.

## Запуск и завершение

- До успешного завершения startup сервер не должен принимать запросы.
- При начале graceful shutdown readiness может перейти в not ready до cleanup.
- Liveness не должна падать только из-за временной недоступности внешней системы.
- Если конкретная зависимость не обязательна для обслуживания трафика, её сбой
  не обязан делать весь процесс not ready.

## Диагностика

Подробный endpoint добавлять отдельно и защищать авторизацией или сетевой
политикой. Даже защищённый ответ не должен раскрывать:

- DSN и credentials;
- secret paths и значения;
- stack traces;
- внутренние адреса без необходимости;
- полные исключения библиотек.

## Проверки

- Liveness не вызывает внешние адаптеры.
- Readiness возвращает `200` и `503` согласно состоянию.
- Ответы соответствуют контракту и не содержат внутренних данных.
- Health routes могут быть исключены из OpenAPI по требованиям.
