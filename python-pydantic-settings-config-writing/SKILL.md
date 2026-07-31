---
name: python-pydantic-settings-config-writing
description: "Используй при реализации или правке типизированной конфигурации Python-сервиса в infrastructure через pydantic-settings и pydantic: top-level settings worker-а, вложенных технологических блоков, YAML/env/custom sources, загрузчика CONFIG_FILE, строгой валидации, file-based secrets, startup preflight и примеров конфигурации. Не применять для domain/application-логики, реализации runtime-клиентов и presentation worker-ов."
---

# Конфигурация через pydantic-settings

## Порядок работы

1. Извлечь точные поля, типы, обязательность, источники, приоритеты и безопасные
   значения по умолчанию.
2. Предложить типовой набор для известной технологии, но не добавлять
   несогласованные поля.
3. Разделить top-level settings worker-а и вложенные технологические блоки.
4. Реализовать immutable-модели, loader и startup preflight.
5. Обновить YAML-примеры и deployment-конфигурацию, затронутую контрактом.
6. Проверить модели, источники, примеры, ошибки загрузки и preflight тестами.

## Граница ответственности

- Конфигурация является infrastructure-адаптером и не импортируется из domain или
  application.
- Settings содержат данные и чистые производные значения, но не создают pool,
  client, worker, logger и другие runtime-ресурсы.
- Adapter factory преобразует settings в точные kwargs клиента; не передавать
  `model_dump()` вслепую.
- Сетевые проверки, чтение секретов и writable-проверки не выполнять в pydantic
  validators.
- Не определять поля и defaults, отсутствующие в требованиях.

## Структура

Для малого сервиса допустима плоская структура:

```text
infrastructure/config/
├── __init__.py
├── base.py
├── workers.py
├── postgres.py
└── nats.py
```

При более чем трёх top-level worker settings либо смешении несвязанных
зависимостей перейти к:

```text
infrastructure/config/
├── __init__.py
├── base.py
├── worker/
│   ├── api.py
│   ├── nats_consumer.py
│   └── publisher.py
├── postgres.py
└── nats.py
```

- `base.py` содержит общие базовые модели, loader и source strategy.
- Один top-level worker config композирует только нужные ему concern-блоки.
- Concern-модуль называть по технологии: `postgres.py`, а не общий `db.py`.
- `__init__.py` содержит только re-export и алфавитный `__all__`.
- Не выполнять массовое перемещение автоматически в небольшой задаче.

## Модели

### Два уровня

- Только top-level worker settings наследуется от `BaseSettings`.
- Вложенные concern-блоки наследуются от общей immutable `BaseModel`.
- Не вкладывать `BaseSettings` в `BaseSettings`.

### Неизменяемость и строгость

- Использовать `frozen=True`, `extra="forbid"`, `strict=True` и
  `validate_default=True`.
- Использовать `tuple`, а не `list`, для коллекций.
- Не менять settings после загрузки и не применять `model_copy(update=...)` как
  runtime reload.
- Для env-источника отдельно реализовать контролируемое преобразование строк:
  строгая YAML-валидация не означает произвольную env-coercion.
- Неизвестный ключ и неверный тип должны останавливать запуск.

### Обязательность

- Обязательный блок объявлять без `default_factory`:

  ```python
  postgres: PostgresSettings
  ```

- `default_factory` использовать только для полностью необязательного блока с
  безопасными defaults.
- Адреса внешних систем, имена БД/stream-ов, пользователи и важные пути не
  угадывать.
- Не использовать пустую строку вместо обязательного значения.

Подробности: [модели и валидация](references/models_and_validation.md).

## Источники и загрузчик

- Состав и приоритет источников определять требованиями.
- Перечислять источники явно через `settings_customise_sources`; первый элемент
  tuple имеет высший приоритет.
- Не смешивать YAML, env, dotenv и secrets автоматически.
- YAML-only через `CONFIG_FILE` остаётся рекомендуемым вариантом для текущих
  сервисов, но не универсальным правилом.
- Не читать `CONFIG_FILE` при импорте модуля.
- Загрузчик при вызове проверяет env, абсолютный путь, существование и читаемость,
  затем создаёт новый settings-объект.
- Не создавать глобальный `settings = ...` и не кешировать loader неявно.
- Один worker получает один immutable top-level config в composition root.

Подробности: [источники и загрузчик](references/sources_and_loader.md).

## Секреты и внешние файлы

- Хранить путь к секрету как обязательный `Path`.
- Не хранить plaintext в YAML/env, если контракт использует file-based secret.
- Не создавать property с DSN, содержащим открытый пароль.
- Секрет читает отдельный provider при создании клиента; для временного хранения
  использовать `SecretStr`.
- Не включать секреты в repr, dump, логи, metrics, traces и health responses.
- Startup preflight проверяет файлы, директории и другие внешние артефакты после
  структурной валидации.
- Ротация требует явного перечитывания и пересоздания зависимого клиента/pool;
  property с повторным чтением файла сама по себе ротацию не обеспечивает.

Подробности: [секреты и preflight](references/secrets_and_preflight.md).

## Валидация

- Простые ограничения задавать через `Annotated` и `Field`.
- `field_validator` использовать для необходимой нормализации одного поля.
- `model_validator(mode="after")` использовать для связей нескольких полей.
- Проверять `min_size <= max_size` и другие заданные взаимосвязи.
- Единицы выражать `timedelta` либо суффиксом `_seconds`, `_bytes` и т. п.
- Конечные наборы задавать `StrEnum` или `Literal`, а не свободным `str`.
- Не нормализовать значимое значение без требования.
- Собственные сообщения об ошибках писать по-русски без исходных секретов.

## Совместимость

- Считать имена и типы полей deployment-контрактом worker-а.
- При добавлении обязательного поля обновлять все затронутые examples и manifests.
- Переименование/удаление согласовывать с порядком развёртывания.
- Временный alias добавлять только на обозначенный переходный период.
- Одновременную передачу старого и нового имени считать ошибкой.
- Не менять единицу измерения под прежним именем поля.

## Безопасное логирование и ошибки

- Не логировать `model_dump()` целиком.
- Startup-summary строить только из явно разрешённых полей.
- Loader/preflight не подавляют ошибки и не переходят к defaults после сбоя.
- На startup-границе логировать ошибку один раз: worker, источник и путь поля без
  секретного значения.
- Завершать процесс до создания runtime-ресурсов.

## Примеры технологий

Читать только относящийся к задаче материал:

- [PostgreSQL, пул psycopg и yoyo](references/postgres_settings.md)
- [NATS и JetStream](references/nats_settings.md)
- [FastAPI, Uvicorn и CORS](references/fastapi_uvicorn_settings.md)
- [Логирование](references/logging_settings.md)
- [Worker подписок](references/subscription_settings.md)
- [Шаблон новой технологии](references/example_template.md)

Примеры являются предложением полей, а не контрактом. Сверять параметры с
официальным API установленной версии. Не добавлять все параметры клиента на
будущее.

## Тестирование

- Создавать модели обычной валидацией; не использовать `model_construct()` как
  стандартную фикстуру.
- Отдельно тестировать structural validation, source priority и preflight.
- Параметризованно загружать каждый `*.example.yaml` соответствующим классом.
- Проверять неизвестный ключ, неверный тип, отсутствие обязательной секции,
  повреждённый YAML и недоступный внешний файл.
- Проверять immutable-модели и отсутствие mutable collections.
- Использовать `tmp_path` для preflight file tests.
- Не проверять внутреннюю реализацию pydantic-settings.

## Антипаттерны

- `BaseSettings` во вложенном блоке.
- Mutable settings и коллекции.
- `extra="ignore"` внутри выбранного worker config.
- `getenv()` в class body или module-level settings instance.
- `Field(default_factory=...)` для обязательной внешней зависимости.
- Production-sensitive default, похожий на рабочее значение.
- I/O или создание клиента в validator/property.
- Полный DSN с паролем в обычной строке.
- `model_dump()` конфигурации в логах.
- Автоматическое смешение источников.
- `model_construct()` как основной путь тестирования.
- Универсальный `config.py` со всеми технологиями и worker-ами.

## Критерии готовности

- Реализованы только согласованные поля и источники.
- Top-level и вложенные модели глубоко неизменяемы.
- Неизвестные поля запрещены, defaults валидируются.
- Обязательные блоки нельзя пропустить.
- Loader не читает env во время импорта.
- Секреты не материализуются в serializable settings.
- Preflight отделён от structural validation.
- Клиентские kwargs строятся явным adapter factory.
- Примеры проходят валидацию и не содержат plaintext-секреты.
- Изменение deployment-контракта отражено во всех потребителях.
- Unit-тесты и проверки репозитория прошли.

## Материалы

- [Модели и валидация](references/models_and_validation.md)
- [Источники и загрузчик](references/sources_and_loader.md)
- [Секреты и preflight](references/secrets_and_preflight.md)
- [Чеклисты](references/checklists.md)
