# Codex Skills

Персональные скилы для проектирования, разработки, проверки и документирования
Python-сервисов с DDD, CQRS и гексагональной архитектурой.

## Структура

```text
skill-library/
├── AGENTS.md        # глобальные инструкции Codex
├── implementation/  # реализация, конфигурация, runtime, CI и тесты
├── documentation/   # проектирование и ведение документации
├── orchestration/   # сквозные циклы реализации и доработки документации
└── review/          # независимые проверки реализации и документации
```

Каждый непосредственный дочерний каталог этих групп является самостоятельным
скилом и содержит `SKILL.md`. Каталог репозитория можно размещать в любом месте;
для обнаружения скилов Codex создать символические ссылки в глобальном каталоге
`skills`.

## Установка на Linux

Клонировать репозиторий:

```bash
git clone git@github.com:Nemagu/codex_skills.git ~/.codex/skill-library
cd ~/.codex/skill-library
```

Создать ссылки на все скилы, не перезаписывая существующие пути:

```bash
repo_dir="$(pwd)"
codex_dir="${CODEX_HOME:-$HOME/.codex}"
skills_dir="$codex_dir/skills"
mkdir -p "$skills_dir"

find "$repo_dir/implementation" "$repo_dir/documentation" \
  "$repo_dir/orchestration" "$repo_dir/review" \
  -mindepth 1 -maxdepth 1 -type d -print0 |
while IFS= read -r -d '' skill_dir; do
  link_path="$skills_dir/$(basename "$skill_dir")"
  if [ -e "$link_path" ] || [ -L "$link_path" ]; then
    printf 'Пропущен существующий путь: %s\n' "$link_path"
    continue
  fi
  ln -s "$skill_dir" "$link_path"
done
```

Подключить глобальные инструкции, если путь ещё не занят:

```bash
agents_path="$codex_dir/AGENTS.md"
if [ -e "$agents_path" ] || [ -L "$agents_path" ]; then
  printf 'Пропущен существующий путь: %s\n' "$agents_path"
else
  ln -s "$repo_dir/AGENTS.md" "$agents_path"
fi
```

## Установка на Windows

В PowerShell клонировать репозиторий:

```powershell
git clone git@github.com:Nemagu/codex_skills.git "$env:USERPROFILE\.codex\skill-library"
Set-Location "$env:USERPROFILE\.codex\skill-library"
```

Включить Developer Mode или запустить PowerShell с правами, разрешающими создание
символических ссылок. Затем выполнить:

```powershell
$RepoDir = (Get-Location).Path
$CodexDir = if ($env:CODEX_HOME) {
    $env:CODEX_HOME
} else {
    Join-Path $env:USERPROFILE ".codex"
}
$SkillsDir = Join-Path $CodexDir "skills"
New-Item -ItemType Directory -Force -Path $SkillsDir | Out-Null

$Groups = @("implementation", "documentation", "orchestration", "review")
foreach ($Group in $Groups) {
    Get-ChildItem -LiteralPath (Join-Path $RepoDir $Group) -Directory |
        ForEach-Object {
            $LinkPath = Join-Path $SkillsDir $_.Name
            if (Test-Path -LiteralPath $LinkPath) {
                Write-Host "Пропущен существующий путь: $LinkPath"
            } else {
                New-Item -ItemType SymbolicLink -Path $LinkPath -Target $_.FullName |
                    Out-Null
            }
        }
}
```

Подключить глобальные инструкции, если путь ещё не занят:

```powershell
$AgentsPath = Join-Path $CodexDir "AGENTS.md"
if (Test-Path -LiteralPath $AgentsPath) {
    Write-Host "Пропущен существующий путь: $AgentsPath"
} else {
    New-Item -ItemType SymbolicLink -Path $AgentsPath `
        -Target (Join-Path $RepoDir "AGENTS.md") | Out-Null
}
```

Команды установки не перезаписывают существующие пути. Если `AGENTS.md` уже
существует, сначала перенести его актуальное содержимое в файл репозитория, а затем
заменить его ссылкой. После добавления новых скилов повторно выполнить
соответствующую команду создания ссылок. Если новый скил или изменённые глобальные
инструкции не появились в Codex автоматически, перезапустить Codex.

## Добавление скила

Создавать новый скил сразу в подходящей группе:

```text
implementation/<skill-name>/
documentation/<skill-name>/
orchestration/<skill-name>/
review/<skill-name>/
```

Не создавать каноническую копию непосредственно в глобальном каталоге `skills`.
После создания и валидации скила повторно выполнить команду создания ссылок для
своей операционной системы.

## Состав

- документирование бизнес-контекстов, слоёв и внешних контрактов;
- обсуждение и фиксация решений разных слоёв (`decision-record-writing`);
- реализация domain-, application-, infrastructure- и presentation-компонентов;
- Postgres, FastAPI, NATS, фоновые процессы и конфигурация;
- pytest, GitLab CI, logging и документация репозитория;
- проверка реализации по скилам и документации;
- проверка полноты и непротиворечивости документации.

## Записи решений

Скил [decision-record-writing](documentation/decision-record-writing/SKILL.md)
помогает исследовать варианты, согласовать выбор и сохранить его обоснование.
Запись создаётся по решению пользователя. При изменении домена, работающего
в production, глобальные инструкции предписывают предложить её создание.

Документация служит первичным источником истины; реализацию можно читать только
с явного разрешения пользователя. Записи понятны вне репозитория: область
изменения описывается через слои и контракты, без обязательных ссылок на документы
или коммиты. Актуальные требования остаются в спецификациях слоёв.

Решение одного слоя размещается в его `decisions/`, нескольких — в общем
`decisions/` документации. ID включает область, например `DR-DOMAIN-0001` или
`DR-COMMON-0001`; каждая область имеет отдельную нумерацию. Правила пересмотра,
статусов и адаптируемый шаблон входят в скил.
