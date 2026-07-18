# Checklists

Пошаговые чек-листы добавления типовых артефактов слоя `infrastructure/config/`.

## Добавление нового concern-блока

1. Согласуй с пользователем набор полей. Если технология типовая — предложи стандартный набор из `tech_examples.md`.
2. Создай файл `infrastructure/config/<concern>.py`. Имя файла — snake_case singular (`db.py`, `nats.py`, `redis.py`).
3. Определи главный блок:
   ```python
   from pydantic import BaseModel, Field


   class <Concern>Settings(BaseModel):
       host: str = "localhost"
       port: int = ...
       # ... согласованные поля
   ```
4. Если есть подразделы (pool, retry policy, и т.п.) — отдельные `<Concern><Part>Settings(BaseModel)`, композируй через `Field(default_factory=...)`.
5. Если есть секреты — паттерн `<x>_file: str` + `@property <x>` + `@model_validator(mode="after")` (см. `nested_settings.md` → File-based secrets).
6. Computed values (URL, derived names) — `@property`, не поля.
7. Обнови `infrastructure/config/__init__.py`: добавь импорты и имена в алфавитный `__all__`.
8. Если блок используется в существующем воркере — обнови соответствующий `<Worker>WorkerSettings` в `workers.py`, добавь поле `<concern>: <Concern>Settings = Field(default_factory=<Concern>Settings)`.
9. Обнови YAML-примеры воркеров (`configs/<worker>.example.yaml`), включи новый блок.

## Добавление нового top-level worker-конфига

1. Определи, какие concern-блоки нужны воркеру.
2. Открой `infrastructure/config/workers.py`.
3. Добавь класс:
   ```python
   class <WorkerKind>WorkerSettings(AppBaseSettings):
       <concern1>: <Concern1>Settings = Field(default_factory=<Concern1>Settings)
       <concern2>: <Concern2>Settings = Field(default_factory=<Concern2>Settings)
       # ... только нужные блоки
   ```
4. Обнови `infrastructure/config/__init__.py`: добавь импорт и имя в алфавитный `__all__`.
5. Создай YAML-пример `configs/<worker_kind>.example.yaml` с наполнением всех полей.
6. В presentation-слое (точка входа воркера) — инстанцируй `<WorkerKind>WorkerSettings()`. Конструктор сам прочитает YAML по `CONFIG_FILE`.

## Добавление нового file-based secret

1. Согласуй с пользователем имя секрета (`password`, `token`, `private_key`) и путь к файлу.
2. В соответствующем `<concern>.py` добавь:
   ```python
   <secret>_file: str = "/run/secrets/<concern>_<secret>"
   ```
3. Добавь validator существования файла:
   ```python
   from os import path
   from typing import Self

   from pydantic import model_validator


   @model_validator(mode="after")
   def validate_<secret>_file(self) -> Self:
       if not path.isfile(self.<secret>_file):
           raise ValueError(f"<Secret> file not found: {self.<secret>_file}")
       return self
   ```
4. Добавь `@property` для чтения:
   ```python
   @property
   def <secret>(self) -> str:
       with open(self.<secret>_file, "r", encoding="utf-8") as file:
           return file.read().strip()
   ```
5. Если секрет используется в URL/connection-string — обнови соответствующий `@property`.
6. **Не** добавляй секрет как отдельное поле в YAML.
7. Обнови dev/staging/prod конвенцию доставки секрета (mounts, vault, и т.п.) — это вне application-слоя, но в документации проекта.

## Добавление новой технологии в `tech_examples.md`

1. Согласуй финальный набор полей с пользователем.
2. Открой `references/tech_examples.md`.
3. Добавь новую секцию `## <Tech name>` (по алфавиту с существующими секциями).
4. Помести полный код-блок с реалистичными дефолтами (где безопасно — `host="localhost"`, `port=...`).
5. Если есть pool / sub-blocks — отдельные класс-блоки внутри одного code-fence.
6. Если есть file-based secret — паттерн полностью.
7. Computed properties — обязательно, если есть URL / составные имена.
8. После кода — список «Типичные поля» с краткими аннотациями.
9. Не добавляй блок в `__init__.py` — `tech_examples.md` это reference, а не часть кодовой базы.

## Добавление нового поля в существующий блок

1. Определи: поле обязательное или опциональное?
2. Если обязательное и без дефолта — все YAML-конфиги воркеров, использующих этот блок, должны включить поле; обнови `configs/*.yaml` шаблоны.
3. Если опциональное / с дефолтом — добавь default в коде; YAML может опустить.
4. Если поле несёт чувствительное значение — применяй file-based secrets паттерн (см. соответствующий чек-лист).
5. Если поле участвует в derived value — добавь/обнови `@property`.
6. Обнови документацию воркера.

## Anti-patterns checklist (что не делать при работе с конфигом)

- [ ] Добавил `BaseSettings` во вложенном блоке? → нет, только `BaseModel`.
- [ ] Положил top-level worker-настройки в `__init__.py`? → нет, только в `workers.py`.
- [ ] Чтение env-вара для отдельного поля? → нет, всё через YAML по `CONFIG_FILE`.
- [ ] Plaintext-пароль в YAML? → нет, file-based secret.
- [ ] Поле `password: str` для секрета? → нет, `<x>_file` + `@property`.
- [ ] Derived value в поле? → нет, `@property`.
- [ ] Mutable default напрямую? → нет, `Field(default_factory=...)`.
- [ ] Добавил `<Concern>` без суффикса `Settings`? → имя класса: `<Concern>Settings`.
- [ ] Top-level воркер без суффикса `WorkerSettings`? → имя класса: `<WorkerKind>WorkerSettings`.
- [ ] Переиспользовал секрет через копирование `<x>_file` → `<x>`? → нет, `@property` всегда читает файл, не кеширует.
