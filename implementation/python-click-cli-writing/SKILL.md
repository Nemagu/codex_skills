---
name: python-click-cli-writing
description: "Используй при реализации или рефакторинге presentation-слоя CLI на Python с Click: дерева групп и команд, аргументов и опций, command-local моделей, Click context/state, тонких command handlers и общей границы выполнения. Не применять для одной только документации CLI, application/domain-логики, реализации конфигурации, адаптеров ресурсов или общего logging runtime."
---

# Реализация CLI на Python с Click

## Область скила

Скил реализует Click presentation-слой поверх уже определённых CLI-контрактов и
публичных application-операций. Его результат — выполняемое дерево Click, где
каждая конечная команда разбирает внешний ввод, вызывает одну application-
операцию и возвращает стабильный консольный исход.

Не определять в этом скиле бизнес-семантику, транзакции, repositories,
технологические settings, lifecycle конкретного адаптера или полный logging-
контракт.

## Порядок работы

1. Извлечь из документации полный путь команды, параметры, типы, defaults,
   сочетания опций, help, результат и exit statuses.
2. Найти одну публичную application-операцию, её вход, результат и ошибки.
3. Изучить существующие CLI entrypoint, структуру presentation и способы
   получения зависимостей в проекте.
4. Реализовать parsing и dispatch средствами Click.
5. Собрать короткую конечную команду и подключить её к общей границе выполнения.
6. Добавить проверки наблюдаемого Click-контракта и запустить проверки проекта.

Не менять документацию, application/domain или адаптеры без отдельного запроса.
Если обязательного контракта либо application entry point нет, остановиться и
сообщить о блокере.

## Click tree и parsing

- Описывать selector, группы, команды, аргументы и опции через Click; не
  выполнять предварительный ручной разбор `sys.argv`.
- Использовать типы Click (`UUID`, `Path`, `IntRange`, `DateTime`) и `Choice`
  для закрытых наборов вместо ручной проверки строк.
- Межполевые transport-ограничения проверять после parsing без I/O.
- Если одиночную опцию запрещено повторять, отклонять повтор явно: стандартное
  поведение Click «последнее значение победило» может нарушать контракт.
- Взаимоисключающие общие flags фиксировать через единый callback/state, а не
  проверять заново в каждой команде.
- Делать help eager, чтобы его показ не запускал command handler и не разрешал
  зависимости.
- Общие опции подключать на каждом уровне дерева, где контракт разрешает их
  указывать; выбранное значение сохранять один раз в state.

Help должен отражать реальные Click types. Для enum использовать `Choice`, чтобы
в сигнатуре были перечислены все варианты. Если контракт требует один шаблон
команды, показывать один пример и перечислять в нём доступные enum-варианты возле
соответствующих flags, не дублируя пример для каждого значения.

## Структура

Для нетривиального CLI использовать минимально необходимое разбиение:

```text
entrypoints/cli.py              process entrypoint
presentation/cli/
|-- app.py                     Click tree и вызов без неявного SystemExit
|-- options.py                 только общие Click options/callbacks
|-- state.py                   state одного запуска и pass decorator
|-- runtime.py                 общая граница выполнения команд
|-- dependencies.py            интерфейс ленивого разрешения возможностей
|-- errors.py                  CLI presentation errors
|-- render.py                  общий rendering, если он действительно общий
`-- commands/
    `-- resource_action/
        |-- command.py         Click-команда и её application-handler
        `-- models.py          модели, используемые только этой командой
```

Сохранять существующую структуру проекта и не создавать пустые модули. Каждую
конечную команду помещать в отдельный пакет, когда у неё есть локальные модели
или mapping.

Не выносить application-handler из `command.py` только ради уменьшения файла.
Отдельный `handler.py` оправдан лишь самостоятельной ответственностью или
реальным повторным использованием.

## Click context и state

Click должен владеть `click.Context`. Состояние одного CLI-вызова хранить в
`Context.obj` и передавать типизированным decorator-ом на основе
`make_pass_decorator`.

В общем state допустимы:

- время начала вызова;
- выбранные общие output/verbosity параметры;
- метаданные текущей команды и correlation ID;
- dependency injector либо набор общих providers.

Не хранить в state идентификаторы и поля одной конкретной команды. Они
принадлежат command-local invocation и при необходимости предоставляют свой
безопасный logging context.

Минимальная связка state и Click context:

```python
@dataclass(slots=True)
class CLIState:
    injector: CLIDependencyInjector = field(default_factory=CLIDependencyInjector)
    output_format: OutputFormat = OutputFormat.TEXT


pass_state = make_pass_decorator(CLIState)


def run_cli(arguments: Sequence[str], state: CLIState) -> int:
    status = cli.main(
        args=list(arguments),
        standalone_mode=False,
        obj=state,
    )
    return int(status or 0)
```

## Command-local модели

Invocation представляет уже разобранный внешний ввод и выполняет только
transport-validation без I/O. Она может явно преобразовываться в публичный
application input/command.

Result представляет стабильный успешный результат команды и явно создаётся из
application DTO. Не передавать Click types или invocation в application и не
возвращать application DTO напрямую в renderer.

Общие CLI-модели заводить только для полей и поведения, разделяемых несколькими
командами. Command-specific модели хранить рядом с `command.py` в `models.py`.

## Тонкая конечная команда

Command callback должен быть сопоставим по ответственности с HTTP endpoint:

1. получить типизированные значения Click;
2. создать command-local invocation;
3. передать state, invocation, тип use case и локальный handler общему executor;
4. вернуть exit status.

Он не читает config, не создаёт pool/UoW/repository, не настраивает logging, не
содержит общий `try/except` и не форматирует результат.

Локальный async handler остаётся в `command.py`: явно преобразует invocation,
вызывает use case ровно один раз и преобразует application result в CLI result.

Пример формы `command.py` без инфраструктурной сборки:

```python
@command(name="update")
@option("--resource-id", required=True, type=UUIDType)
@option(
    "--status",
    type=Choice(tuple(item.value for item in ResourceStatus)),
)
@pass_state
def update_resource(
    state: CLIState,
    resource_id: UUID,
    status: str | None,
) -> int:
    invocation = UpdateInvocation(
        resource_id=resource_id,
        status=status,
        output_format=state.output_format,
    )
    return execute_command(state, invocation, UpdateUseCase, handle_update)


async def handle_update(
    invocation: UpdateInvocation,
    use_case: UpdateUseCase,
) -> UpdateResult:
    result = await use_case.execute(invocation.to_application_input())
    return UpdateResult.from_application(result)
```

## Общая граница выполнения

Один общий executor должен:

- установить общие метаданные state;
- выполнить межполевую transport-validation invocation до разрешения ресурсов;
- получить готовый use case через injector/provider;
- выполнить локальный handler;
- передать результат общему renderer-у;
- единообразно сопоставить input, dependency, public application и unexpected
  ошибки в безопасный результат и exit status.

Не копировать эту механику по конечным командам. Для async application-кода
event-loop runner и lifecycle общей команды также принадлежат executor-у.

Конкретные тексты ошибок, logging events, streams и форматы результата брать из
CLI-контракта проекта. Не включать сырые значения неизвестных опций или exception
details в пользовательский результат.

## Зависимости без выхода в смежные области

State предоставляет ленивый injector, а команда запрашивает готовый use case,
не config и не низкоуровневые ресурсы. Help, импорт модулей и построение Click
tree не должны разрешать зависимости.

Если выбранная команда требует возможность, которой нет в конфигурации текущего
процесса, общая граница получает типизированную dependency error. Детальную
реализацию выполнять профильными скилами:

- типизированные settings и выбор нужных секций —
  `python-pydantic-settings-config-writing`;
- PostgreSQL pool/UoW и connection lifecycle —
  `python-psycopg-yoyo-persistence-writing`;
- logging configuration, context и error records —
  `python-service-logging-writing`;
- Python style и docstrings — `python-code-style-writing`;
- устройство pytest fixtures и test suite — `python-pytest-testing`;
- проектирование либо изменение Markdown-контракта CLI —
  `cli-documentation-writing`.

Этот скил определяет только место подключения этих возможностей к Click CLI и
не повторяет их внутренние правила.

## Проверки Click-контракта

Через реальное Click tree и подменяемый state/injector проверить:

- help каждого уровня без разрешения зависимостей;
- допустимые пути команд и расположение общих опций;
- types, enum choices, обязательность, повторы и взаимоисключения;
- mapping parsed values в один application call;
- text/json и verbosity modes, если они входят в контракт;
- единый безопасный error outcome и exit status;
- process entrypoint: явные arguments и `sys.argv[1:]`.

Не тестировать внутренности Click. Детальную организацию, fixtures,
параметризацию и coverage выполнять по `python-pytest-testing`.

## Критерии готовности

- Parsing и dispatch полностью выполняет Click.
- Help соответствует фактическим типам, enum choices и иерархии.
- Конечная команда остаётся короткой и не собирает инфраструктуру.
- Command-local поля не хранятся в общем state.
- Help не читает config и не открывает ресурсы.
- Все конечные команды используют общую границу выполнения и ошибок.
- Каждая команда вызывает одну публичную application-операцию.
- Профильные и полные проверки проекта проходят.
