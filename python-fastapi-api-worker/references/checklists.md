# Чеклист

## Сборка приложения

- [ ] Фабрика отделена от запуска процесса.
- [ ] Повторный вызов создаёт независимое приложение.
- [ ] Фабрика не выполняет I/O.
- [ ] Параметры минимальны, типизированы и неизменяемы.
- [ ] Routers, handlers и middleware переданы явно.
- [ ] Внешний settings не распространяется по presentation.
- [ ] Route names и operation IDs не конфликтуют.

## Зависимости и жизненный цикл

- [ ] Конкретные адаптеры выбраны в composition root.
- [ ] Presentation не импортирует persistence реализации.
- [ ] Unit of Work не передаётся endpoint-ам.
- [ ] Один lifespan управляет всеми process-scoped ресурсами.
- [ ] Cleanup регистрируется сразу после создания ресурса.
- [ ] Частичный startup освобождает созданные ресурсы.
- [ ] Shutdown идёт в обратном порядке.
- [ ] Runtime context один, типизирован и неизменяем.
- [ ] Cancellation и startup-ошибки не подавляются.

## Промежуточное ПО и ошибки

- [ ] Состав middleware задан требованиями.
- [ ] Порядок описан для входящего и исходящего пути.
- [ ] Собственные middleware stateless.
- [ ] Ограничения `BaseHTTPMiddleware` учтены.
- [ ] CORS включён только при необходимости.
- [ ] Domain-ошибки не обрабатываются в presentation.
- [ ] Внешний error response не раскрывает внутренние данные.
- [ ] Unhandled exception логируется один раз с traceback.
- [ ] Access log не содержит secrets и полного query/body.

## Uvicorn и окружение

- [ ] Готовый app используется только в single-process режиме.
- [ ] Reload/workers используют import string или factory.
- [ ] Аргументы сверены с установленной версией Uvicorn.
- [ ] Graceful shutdown согласован с внешним supervisor.
- [ ] Forwarded headers принимаются только от доверенных proxy.
- [ ] Root path и OpenAPI/docs URLs конфигурируемы.

## Проверки состояния

- [ ] Liveness не вызывает внешние зависимости.
- [ ] Readiness использует дешёвое runtime-состояние.
- [ ] Health responses не раскрывают инфраструктуру.
- [ ] Startup, частичный startup и shutdown проверены.
- [ ] Middleware и error mapping проверены интеграционно.
- [ ] Выбранный режим runner-а проверен без production bind.
