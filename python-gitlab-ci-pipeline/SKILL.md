---
name: python-gitlab-ci-pipeline
description: Используй при создании или правке .gitlab-ci.yml для Python-сервисов на uv (DDD/CQRS-бэкенды). Триггеры — добавление/изменение джоб lint, test (unit и integration), build образа; настройка кэша uv, стадий, workflow.rules; авто-отмена устаревших пайплайнов при новом push'е (`workflow.auto_cancel.on_new_commit` + `interruptible`); сборка docker-образа в CI (Kaniko, теги по ветке, registry mirror); подключение отчётов (JUnit, coverage, code quality) и security-гейтов (bandit, pip-audit, trivy); деплой в Docker Swarm через SSH с обновлением существующего стека на сервере (`docker stack deploy` поверх compose на хосте, deploy token для registry, environments и manual-gate для prod). Не применять для написания самих тестов — для этого есть python-pytest-testing.
---

# Python GitLab CI Pipeline

Скил про `.gitlab-ci.yml` прикладного Python-сервиса на `uv`: стадии, джобы, кэш, отчёты, сборка образа. Сам скил тесты не пишет — за «что именно запускать в тестовых джобах» отвечает `python-pytest-testing`; здесь только то, как обернуть это в CI.

## Quick Start

1. Опиши стадии: `lint → test → build → scan → deploy`.
2. Сделай `workflow.rules` для дедупа пайплайнов branch/MR + `workflow.auto_cancel.on_new_commit: interruptible` для отмены устаревших пайплайнов на новый push.
3. Заведи `default:`-блок: `image: python:3.14-slim` + установка `uv` + кэш по `uv.lock` + `interruptible: true`, и `variables:` с `UV_*`.
4. Стадия `lint`: `ruff` (+ codequality-отчёт), `ruff-format`, `bandit` (gate), `deps-audit` (gate).
5. Стадия `test`: `unit-tests`, затем `integration-tests` с `needs: ["unit-tests"]`, `services:` и `INTEGRATION_USE_EXTERNAL_INFRA=1` — сверься с `python-pytest-testing`. Оба отдают JUnit + cobertura.
6. Стадия `build`: `build-image` на Kaniko — тег `dev` на `develop`, `prod` на `main`, плюс `$CI_COMMIT_SHORT_SHA`; на других ветках не билдим.
7. Стадия `scan`: `container-scan` (Trivy по собранному образу, gate на HIGH/CRITICAL).
8. Стадия `deploy`: шаблон `.deploy` (alpine + openssh-client) + один job `deploy`, у которого rules различают ветки и environment; на `develop` авто, на `main` `when: manual`. Подключение по SSH к swarm-manager и `docker stack deploy` поверх существующего compose на сервере.
9. Согласуй таймауты с пользователем (см. раздел «Таймауты»): один в `default:` и индивидуальные на тяжёлых джобах.
10. Прогони пайплайн, убедись что отчёты подхватились, зафиксируй DoD.

Готовый эталонный файл целиком — в [pipeline_skeleton.md](references/pipeline_skeleton.md). Бери его как стартовую точку и адаптируй под проект.

## Режимы значений по умолчанию

Скилл умеет работать в двух режимах. **Перед началом работы агент обязан спросить, какой режим**, и не выбирать сам:

- **quick** — все значения берутся из Default-таблиц этого скила без вопросов; агент только сообщает в одном сообщении, какие defaults применил, и где можно пересогласовать постфактум.
- **interactive** — на каждый пункт, для которого в скиле есть Default-таблица, агент задаёт вопрос с default'ом как первой опцией («Recommended»). Пользователь либо принимает, либо вводит своё.

Категории, на которые распространяются режимы (для них есть Default-таблицы ниже): **таймауты джоб**, **базовые образы**, **имена и пути в deploy**. Всё остальное (имена сервисов под проект, токены, hostnames серверов, scope-флаги переменных) — это входные данные конкретного проекта; агент берёт их у пользователя в обоих режимах.

В quick-режиме агент **не должен** молча менять схему YAML, состав джоб, права/типы переменных и прочие архитектурные решения скила — defaults касаются только конкретных значений из таблиц.

## Структура пайплайна

Стадии идут строго по зависимостям, внутри стадии джобы параллельны:

- **`lint`** — статика и быстрые гейты: `ruff check`, `ruff format --check`, `bandit` (SAST), `pip-audit` (зависимости). Всё параллельно, падает рано и дёшево.
- **`test`** — `unit-tests`, потом `integration-tests`. Для `integration-tests` обязательно задавай `needs: ["unit-tests"]`: Postgres/NATS не должны подниматься, пока не прошли быстрые юниты.
- **`build`** — `build-image`: собираем и пушим образ. Только после зелёных lint+test — в реестр не должен попадать образ из непрошедшего кода.
- **`scan`** — `container-scan`: сканируем уже собранный образ. Отдельная стадия, потому что нужен артефакт билда.
- **`deploy`** — `deploy`: ssh-ом подключаемся к swarm-manager целевого окружения и катим `docker stack deploy` поверх существующего compose на сервере. Только после зелёного `scan` — уязвимый образ не должен попадать в прод/дев.

Имя стадии — `lint`, не `check` (так в `companies/backend` и `users/backend`; единообразие важнее).

## `default:` и `variables:`

Один общий блок, чтобы не дублировать установку `uv` и кэш по джобам:

```yaml
default:
  image: python:3.14-slim
  timeout: <согласовано>   # см. раздел «Таймауты»; перекрывается per-job для длинных джоб
  interruptible: true       # см. раздел «Отмена устаревших pipeline'ов»; перекрывается на deploy
  before_script:
    - pip install --no-cache-dir --disable-pip-version-check --root-user-action=ignore uv
  cache:
    key:
      files:
        - uv.lock
    paths:
      - .uv-cache/

variables:
  UV_CACHE_DIR: $CI_PROJECT_DIR/.uv-cache
  UV_LINK_MODE: copy
  UV_FROZEN: "1"
  UV_NO_PROGRESS: "1"
  PIP_DISABLE_PIP_VERSION_CHECK: "1"
  PIP_NO_CACHE_DIR: "1"
```

- Кэшируем `.uv-cache/` (скачанные дистрибутивы), а не `.venv/`. Каждая джоба делает свой `uv sync` с нужной группой зависимостей (`--no-install-project`, без самого проекта) — это и есть «per-job sync». Подробности и сравнение с подходом «`uv sync` в `before_script` + кэш `.venv/`» — в [caching_and_uv.md](references/caching_and_uv.md).
- `UV_FROZEN=1` — `uv sync` падает, если `uv.lock` не соответствует `pyproject.toml` (lock не обновляется молча в CI).
- `UV_LINK_MODE=copy` — не ругаться на hardlink между кэшем и `.venv` на разных ФС раннера.
- Ключ кэша по `uv.lock` — кэш инвалидируется при смене зависимостей.

Джобы, которым `uv` не нужен (`build-image` на образе Kaniko, `container-scan` на образе Trivy), переопределяют `image:`, и им обязательно `before_script: []` и `cache: []`, чтобы сбросить `default`.

## workflow

Чтобы не плодить два пайплайна (на push в ветку и на сам MR одновременно) и сразу включить авто-отмену устаревших pipeline'ов при новом push'е в ту же ветку:

```yaml
workflow:
  auto_cancel:
    on_new_commit: interruptible
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS == null
    - if: $CI_COMMIT_TAG
```

(паттерн из `users/backend`; строку про теги добавь, если по тегам тоже нужны пайплайны)

Подробнее про `auto_cancel` и взаимодействие с `interruptible:` на джобах — см. раздел «Отмена устаревших pipeline'ов».

## Отмена устаревших pipeline'ов

При push'е поверх ветки, на которой уже идёт pipeline, по умолчанию GitLab продолжает гонять старый pipeline до конца — runner-минуты сжигаются впустую на коде, который уже не актуален. Особенно неприятно для тяжёлых стадий `integration-tests`, `build-image`, `container-scan`. Лечится двумя настройками:

- **`workflow.auto_cancel.on_new_commit: interruptible`** — глобальный режим: при новом push'е в ту же ветку GitLab автоматически отменяет старый pipeline. Альтернативы: `conservative` (default — только pending, running не трогает) и `none` (никогда не отменять).
- **`interruptible: true` на джобе** — пометка «эту джобу безопасно прерывать». Без неё `auto_cancel.on_new_commit: interruptible` ничего не отменит. По умолчанию GitLab считает все джобы non-interruptible.

Практика: ставим `interruptible: true` в `default:` (применяется ко всем джобам) и явно переопределяем `interruptible: false` только на тех, где прерывание опасно.

### Какие джобы interruptible, какие нет

| Стадия | `interruptible` | Почему |
|---|---|---|
| `lint`, `test`, `build`, `scan` | `true` (через default) | Полностью идемпотентны: пересборка с нуля даёт тот же результат; runner-минут на повторе тратится мало; старый pipeline на устаревшем коде ничего ценного не даёт. |
| `deploy` | **`false`** (override) | `docker stack deploy` стартует rolling update на swarm; SIGTERM CI-джобы в момент апдейта оставит часть сервисов на новом digest, часть — на старом, либо оборвёт docker socket-сессию в неопределённом состоянии. Откатывать такое руками неприятно. Лучше дать deploy-у доехать. |

В YAML это выглядит так:

```yaml
default:
  interruptible: true

.deploy:
  interruptible: false      # переопределяет default для deploy
```

(полные блоки — в [pipeline_skeleton.md](references/pipeline_skeleton.md))

### Замечания

- `interruptible` действует только при `auto_cancel.on_new_commit: interruptible` — иначе пометка просто игнорируется.
- Manual-джобы (`when: manual`, как наш `deploy` на `main`) и так не запускаются автоматически — если кто-то ещё не нажал play, новый push отменит весь pipeline целиком вместе с этой manual-джобой как pending; если уже нажал и deploy пошёл — `interruptible: false` защитит его от прерывания.
- Project-level UI («Auto-cancel redundant pipelines» в Settings → CI/CD → General pipelines) — старая версия того же механизма, работает в режиме `conservative` и не учитывает `interruptible:`. `workflow.auto_cancel` в YAML перекрывает project-level настройку и предпочтительнее, потому что лежит вместе с пайплайном и виден в review.

## Таймауты

Дефолтный project-level таймаут GitLab — 1 час, и это слишком много для большинства джоб: зависший pytest или Kaniko-build будет жечь раннер впустую, пока кто-то не отменит пайплайн руками. Поэтому всегда задаём явные таймауты: один общий в `default:` и индивидуальный — на джобах, которые заведомо длиннее.

**Принцип значения:** `timeout` ≈ 2–3× от типичного времени джобы на этом проекте/раннере. Это даёт запас на выбросы (медленный pull базового образа, тормозящий сервис в `services:`), но при реальном зависании джоба падает за минуты, а не за час.

### Default-таблица

| Где | Default | Когда уточнить у пользователя |
|---|---|---|
| `default:` | `10m` | Большинство джоб на проекте сейчас идёт > 4–5 минут; либо у пользователя есть наблюдение, что 10m мало для конкретного раннера. |
| `integration-tests` | `25m` | В проекте крупная интеграционная матрица, поднимаются дополнительные services кроме postgres+nats, или известно, что pull базовых образов медленный (без registry-mirror). |
| `build-image` | `15m` | Большой `Dockerfile` с компиляцией C-расширений, отсутствие registry-mirror, медленный канал до Docker Hub. |
| `container-scan` | `10m` | Очень большой образ; либо Trivy сканит первым прогоном (скачивает БД уязвимостей). |
| `deploy` | `10m` | Стек содержит много сервисов; rolling update идёт долго; ожидание `--detach=false` (если включат). |

Формат `timeout` — человекочитаемый: `30s`, `5m`, `1h 30m`. Project-level лимит (Settings → CI/CD → Timeout) при этом всё ещё работает как верхняя граница — джоба не может задать `timeout` больше него.

В quick-режиме агент применяет таблицу как есть, в interactive — задаёт по одному вопросу на джобу с дефолтом как первой опцией.

Синтаксис:

```yaml
default:
  timeout: 10m

integration-tests:
  timeout: 25m
```

## Базовые образы

Эталонные версии для уже знакомых компонентов пайплайна; в quick-режиме применяем их без вопросов, в interactive — спрашиваем подтверждение. **Перед коммитом обязательно проверить**, что лейбл всё ещё актуален: версия python и alpine могут уйти вперёд за полгода-год. Источники для сверки — Docker Hub / image registry (`hub.docker.com/_/<image>`).

| Компонент | Default | Когда уточнить |
|---|---|---|
| `default.image` (CI с uv) | `python:3.14-slim` | На проекте `pyproject.toml` фиксирует другой `requires-python`; либо проект ещё на 3.13/3.12. |
| `.deploy.image` | `alpine:3.23` | На раннере нет интернета к Docker Hub; раннер требует другой минимальный образ; либо политика безопасности на distroless/wolfi. |
| Postgres в `integration-tests.services` | `postgres:18-alpine` | Прод-Postgres ниже 18 — выровнять минор; либо команда тестируется на конкретной major-версии. |
| NATS в `integration-tests.services` | `nats:2.12-alpine` | Прод-NATS на другой версии; используется конкретная фича JetStream, появившаяся позже/раньше. |
| `build-image.image` | `gcr.io/kaniko-project/executor:debug` | Команда переехала на `chainguard/kaniko`; `:debug`-тег больше не нужен (но тогда теряем `sh`, что ломает шаги-обёртки). |
| `container-scan.image` | `aquasec/trivy:latest` | Хочется зафиксированный тег версии вместо `:latest` (для воспроизводимости); либо переход на `chainguard/trivy`. |

## Джоба lint

Минимум — `ruff check` и `ruff format --check` по `src` (не по `.` — лишний шум на конфигах и кэшах). Плюс генерим Code Quality-отчёт для виджета в MR:

```yaml
ruff:
  stage: lint
  script:
    - uv sync --only-group lint --no-install-project
    - uv run ruff check src
    - uv run ruff check src --output-format=gitlab > gl-code-quality-report.json
  artifacts:
    when: always
    reports:
      codequality: gl-code-quality-report.json

ruff-format:
  stage: lint
  script:
    - uv sync --only-group lint --no-install-project
    - uv run ruff format --check src
```

Первый прогон `ruff check src` даёт читаемый вывод в лог и роняет джобу при ошибках; второй пишет машинный отчёт (он ничего не валит сам — гейтом служит первый прогон). Держать их разными джобами (`ruff` и `ruff-format`) — чтобы в UI было видно, что именно сломалось.

`bandit` (SAST) и `pip-audit` (зависимости) — тоже в стадии `lint`, как gate-джобы (падают на находках). Их конфигурация, форматы отчётов и tier-ограничения — в [reports_and_gates.md](references/reports_and_gates.md).

## Джобы test

Что запускать (какие директории — unit по слоям, integration по маркеру/директории), как устроен режим внешней инфраструктуры (`INTEGRATION_USE_EXTERNAL_INFRA=1`, переменные `INTEGRATION_*`) — берётся из `python-pytest-testing`. Здесь — CI-обвязка:

```yaml
unit-tests:
  stage: test
  script:
    - uv sync --no-default-groups --group test --no-install-project
    - >-
      uv run pytest src/tests/units
      --cov=src --cov-report=term-missing --cov-report=xml:coverage.xml --cov-report=html
      --junitxml=report.xml --durations=10 -q
  coverage: '/TOTAL.+?(\d+)%/'
  artifacts:
    when: always
    expire_in: 1 week
    paths:
      - htmlcov/
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

integration-tests:
  stage: test
  needs: ["unit-tests"]
  timeout: <согласовано>   # обычно длиннее default из-за services: и сети
  services:
    - name: postgres:18-alpine
      alias: postgres
      variables:
        POSTGRES_USER: <service>_test
        POSTGRES_DB: <service>_test
        POSTGRES_PASSWORD: <service>_test
    - name: nats:2.12-alpine
      alias: nats
      command: ["-js", "-m", "8222"]
  variables:
    INTEGRATION_USE_EXTERNAL_INFRA: "1"
    INTEGRATION_PG_HOST: postgres
    INTEGRATION_PG_PORT: "5432"
    INTEGRATION_PG_USER: <service>_test
    INTEGRATION_PG_DATABASE: <service>_test
    INTEGRATION_PG_PASSWORD: <service>_test
    INTEGRATION_NATS_HOST: nats
    INTEGRATION_NATS_PORT: "4222"
  script:
    - uv sync --no-default-groups --group test --no-install-project
    - >-
      uv run pytest src/tests/integration
      --cov=src --cov-report=term-missing --cov-report=xml:coverage.xml --junitxml=report.xml -q
  coverage: '/TOTAL.+?(\d+)%/'
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

- `--junitxml` + `reports:junit` → вкладка Tests в пайплайне и диф упавших тестов в MR (работает на GitLab CE).
- `--cov-report=xml:coverage.xml` + `reports:coverage_report` (cobertura) → подсветка покрытых строк в дифе MR (работает на CE).
- `coverage:`-regex вытаскивает процент в бейдж/виджет пайплайна.
- `--cov-report=html` + `paths: htmlcov/` → скачиваемый HTML-отчёт (опционально, но дёшево).
- `--durations=10` — топ-10 самых медленных тестов в логе.
- Порог покрытия (`--cov-fail-under`) **не** ставим по умолчанию — согласовано с `python-pytest-testing` (метрика-ориентир, не жёсткий гейт). Если команда хочет — добавляется в `pytest`-вызов, не в YAML.
- Если в проекте нет NATS — убери `nats` из `services:` и соответствующие переменные. Имена тестовых переменных подключения сверяй с тем, что реально читают фикстуры (`python-pytest-testing`).

## Джоба build-image

Сборка через **Kaniko** — без Docker-демона, без privileged-раннера, без монтирования docker-сокета в джобы. Почему Kaniko, а не docker-socket/dind, и что для этого нужно на стороне раннера/GitLab — в [build_and_registry.md](references/build_and_registry.md).

```yaml
build-image:
  stage: build
  timeout: <согласовано>   # учитывает pull базовых образов и push в registry
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  before_script: []
  cache: []
  variables:
    GIT_DEPTH: "1"
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"${CI_REGISTRY}\":{\"username\":\"${CI_REGISTRY_USER}\",\"password\":\"${CI_REGISTRY_PASSWORD}\"}}}" > /kaniko/.docker/config.json
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      ${KANIKO_REGISTRY_MIRROR:+--registry-mirror=$KANIKO_REGISTRY_MIRROR --insecure-pull}
      --destination "${CI_REGISTRY_IMAGE}:${IMAGE_TAG}"
      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      variables:
        IMAGE_TAG: dev
    - if: '$CI_COMMIT_BRANCH == "main"'
      variables:
        IMAGE_TAG: prod
```

- Тег по ветке задаётся через `variables:` внутри `rules:` — на любой другой ветке/в MR джоба не создаётся вообще.
- Дополнительный неизменяемый тег `$CI_COMMIT_SHORT_SHA` — чтобы откатываться на конкретный коммит и не гадать, что внутри `:dev`.
- `${KANIKO_REGISTRY_MIRROR:+...}` — если переменная (уровня группы/инстанса) задана, билд тянет базовые образы через pull-through кэш; если нет — напрямую. Имя сервиса кэша в репозиторий не зашиваем.
- `before_script: []` / `cache: []` — обязательны, иначе подтянется `default` с `pip install uv`.

## Джоба container-scan

Сканируем уже собранный образ Trivy'ем; gate на HIGH/CRITICAL. На GitLab CE виджета «N уязвимостей» в MR не будет (это Ultimate), но джоба роняет пайплайн и оставляет отчёт-артефакт. Конфигурация — в [reports_and_gates.md](references/reports_and_gates.md).

## Джоба deploy

Рабочий профиль: на сервере уже лежит compose-файл стека (его поддерживают руками или из infra-репо), CI должен обновлять образы в этом стеке после успешного билда. Свежий образ под тегом ветки (`:dev`/`:prod`) уже запушен `build-image`, осталось заставить swarm перерезолвить его digest. Swarm сам это не сделает: он запоминает digest в момент `service create/update` и в registry больше не смотрит. Триггерит обновление команда `docker stack deploy -c <file> --with-registry-auth <stack>` — она читает compose, видит тег, тянет новый digest и катит rolling update тем сервисам, чьи образы изменились.

### Подход: SSH + remote `docker stack deploy`

`docker stack deploy -c <file>` читает compose **на стороне клиента** (там, где запущен docker CLI), резолвит переменные, шлёт уже распарсенный спек в swarm. Это даёт два варианта подключения:

1. **`DOCKER_HOST=ssh://...`** — docker CLI на runner'е, compose **тоже на runner'е** (в репо). Удобно, если compose шаблонизируется и живёт в этом же репозитории.
2. **`ssh <host> bash -s` + `docker stack deploy` на сервере** — compose **на сервере**, docker CLI выполняется там же. Удобно, если compose хранится централизованно вне CI-репо (типичный сценарий «один сервер — много стеков, конфиг ведут DevOps»).

В рамках этого скила по умолчанию — **второй вариант**. Тогда от CI требуется только ssh-доступ deploy-юзера и знание пути к compose; runner'у не нужен ни docker, ни compose-шаблон. На runner'е достаточно `alpine + openssh-client`.

### Шаблон джобы

```yaml
.deploy:
  stage: deploy
  image: alpine:3.23
  interruptible: false       # см. раздел «Отмена устаревших pipeline'ов»
  before_script:
    - apk add --no-cache openssh-client
    - mkdir -p ~/.ssh
    - cp "$SSH_PRIVATE_KEY" ~/.ssh/id_ed25519
    - cp "$SSH_KNOWN_HOSTS" ~/.ssh/known_hosts
    - chmod 600 ~/.ssh/id_ed25519 ~/.ssh/known_hosts
  cache: []
  dependencies: []
  script:
    - |
      ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes \
          "${DEPLOY_USER}@${DEPLOY_HOST}" bash -s <<EOF
      set -e
      echo '${REGISTRY_DEPLOY_TOKEN}' | docker login -u '${REGISTRY_DEPLOY_USER}' --password-stdin '${CI_REGISTRY}'
      docker stack deploy -c '${STACK_COMPOSE_PATH}' --with-registry-auth '${SWARM_STACK_NAME}'
      docker logout '${CI_REGISTRY}'
      EOF

deploy:
  extends: .deploy
  environment:
    name: $DEPLOY_ENV
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      variables:
        DEPLOY_ENV: development
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
      variables:
        DEPLOY_ENV: production
```

Точки внимания:

- **Имя джобы** — одно (`deploy`), не два (`deploy-dev` / `deploy-prod`). Поведение различается только окружением и веткой; rules + scoped-переменные дают весь нужный split, дублирование джоб только захламляет pipeline.
- **`environment:` через переменную** из rules (`name: $DEPLOY_ENV`) — синтаксис GitLab поддерживает. Это позволяет одной джобе соответствовать `development` и `production` в зависимости от того, какое правило сработало.
- **`when: manual` на `main`** — деплой в prod не должен запускаться автоматически по merge: должен быть ручной клик. Опционально включить в Settings → CI/CD → Protected environments approval'ы.
- **`cache: []` и `dependencies: []`** — нет смысла подтягивать uv-кэш и артефакты предыдущих стадий, deploy в них не нуждается.
- **Тег образа** в compose обычно стоит буквально (`:dev`, `:prod`). При `docker stack deploy --with-registry-auth` swarm перечитает digest этого тега и сделает rolling update. Если хочется иммутабельность — пробрасывать `${CI_COMMIT_SHORT_SHA}` через env и шаблонизировать `image:` в compose; но это уже отдельный паттерн (variant 1 выше).
- **`docker login --password-stdin`** — не светим токен в `ps aux` на сервере.
- **`docker logout` в конце** — снимает локальные creds в `~/.docker/config.json` deploy-юзера. К этому моменту `--with-registry-auth` уже передал креды в swarm raft-store, и worker'ы смогут pull'ить образ при последующих рестартах без локального `config.json` на нодах.

### Переменные

Раскладка по уровням:

**Group-level** (общее для всех проектов в группе):

| Variable | Type | Содержимое | Flags | Environment scope |
|---|---|---|---|---|
| `SSH_PRIVATE_KEY` | File | приватный ключ deploy-юзера | Protected | * |
| `SSH_KNOWN_HOSTS` | File | вывод `ssh-keyscan -t ed25519,rsa <hosts>` | Protected | * |
| `DEPLOY_USER` | Variable | имя deploy-юзера на сервере | Protected | * |
| `DEPLOY_HOST` | Variable | IP/hostname dev-сервера | Protected | `development` |
| `DEPLOY_HOST` | Variable | IP/hostname prod-сервера | Protected | `production` |
| `REGISTRY_DEPLOY_USER` | Variable | username GitLab deploy token (scope `read_registry`) | Protected, Masked | * |
| `REGISTRY_DEPLOY_TOKEN` | Variable | сам токен | Protected, Masked | * |

**Project-level** (специфика конкретного сервиса):

| Variable | Type | Содержимое | Flags |
|---|---|---|---|
| `SWARM_STACK_NAME` | Variable | имя стека на сервере (`docker stack ls`) | Protected |
| `STACK_COMPOSE_PATH` | Variable | абсолютный путь к compose-файлу на сервере | Protected |

Если имя стека или путь различаются на dev и prod — дублировать ту же переменную дважды с разными `Environment scope` (`development`/`production`).

**Default-таблица для значений (применяется в quick-режиме, в interactive — как Recommended):**

| Variable | Default | Когда уточнить |
|---|---|---|
| `DEPLOY_USER` | `deployer` | На сервере уже есть другой deploy-юзер (например, `gitlab-deploy`, `ci`); либо корпоративный стандарт имён системных пользователей. |
| `STACK_COMPOSE_PATH` | `/srv/stacks/<service-name>/compose.yml` | Compose поддерживают DevOps-команда и держит его в своём пути (`/home/admin_devops/...`, `/opt/...`); либо в проекте используется не `compose.yml`, а `docker-stack.yml`. |
| `SWARM_STACK_NAME` | имя проекта/сервиса (lowercase, без префиксов) | Стек на сервере уже создан под другим именем — нужно совпадение; либо одно имя стека на dev и prod различаются (тогда два значения через environment scope). |
| `SSH_PRIVATE_KEY` | — | Всегда вводится пользователем (сгенерированный ключ). |
| `SSH_KNOWN_HOSTS` | — | Всегда вводится пользователем (`ssh-keyscan` обоих хостов). |
| `DEPLOY_HOST` | — | Всегда вводится пользователем (IP/hostname серверов). |
| `REGISTRY_DEPLOY_USER` / `REGISTRY_DEPLOY_TOKEN` | — | Всегда вводится пользователем (создаёт Deploy Token в UI). |

Имя сервиса для подстановки в default путь и `SWARM_STACK_NAME` агент берёт из `pyproject.toml` (`project.name`) либо из имени GitLab-проекта, в указанном порядке.

**Почему Protected обязателен:** GitLab выдаёт защищённые переменные только в pipeline'ах на protected ветках. Если флаг не выставлен, а ветка `develop`/`main` помечена protected — переменная **не доедет** в job, и `cp "$SSH_PRIVATE_KEY"` упадёт на пустой строке. Перед тем как писать job, убедиться, что `develop`/`main` помечены protected в Settings → Repository → Protected branches.

### Deploy token vs job-token для pull на нодах

`--with-registry-auth` шлёт в swarm те creds, под которыми CI-джоба сейчас логинилась. Если использовать встроенные `CI_REGISTRY_USER`/`CI_REGISTRY_PASSWORD`, это **job-token**: после завершения job он становится невалидным. Это значит, что при ребуте swarm-ноды через сутки worker не сможет re-pull образ.

Решение — отдельный **deploy token** (Group → Settings → Repository → Deploy tokens) со scope `read_registry`. Username и token — это и есть `REGISTRY_DEPLOY_USER` / `REGISTRY_DEPLOY_TOKEN` из таблицы. Они долгоживущие, retro-rotation делается через UI.

## Подводные камни деплоя

1. **SSH-ключ без завершающего `\n` → `Load key: error in libcrypto`.** GitLab File-variable не добавляет newline сам. В UI: открыть `SSH_PRIVATE_KEY` → встать в конец значения (после `-----END OPENSSH PRIVATE KEY-----`) → нажать Enter → Save. После libcrypto-ошибки ssh падает на password auth, в логе виден симптом `Permission denied (publickey,password)` — настоящая причина выше, не в правах сервера.
2. **Protected variables + non-protected branch.** Если ветка не protected, защищённая переменная просто не доедет в job. Симптом — пустая строка в `$SSH_*`, ошибки вида `cp: can't stat ''`. Перед дебагом ssh-логина сначала проверить, помечены ли `develop`/`main` как protected.
3. **Права на сервере для pubkey-auth.** sshd молча отказывает в авторизации, если `~/.ssh` не `700`, `~/.ssh/authorized_keys` не `600`, или сам `/home/<user>` writable группой/other. Диагностика — `journalctl -u sshd -f` на сервере во время попытки: там будет точная строка типа `Authentication refused: bad ownership or modes`.
4. **Deploy-юзер без пароля и без sudo.** На сервере лучше `passwd -l <user>` — pubkey-only, никаких password fallback'ов в логах CI. Юзер должен состоять в группе `docker` (для прав на сокет), и **только** — никакого `sudoers`, никакого root.
5. **Compose в чужом home.** Если compose лежит, например, в `/home/admin_devops/...`, deploy-юзеру нужны не только права на сам файл, но и `o+x` на каждом каталоге в пути. Чище — перенести compose в нейтральное место (`/srv/stacks/<service>/compose.yml`), owned by deploy-юзером, права `750`/`640`. Это убирает необходимость лазить в чужие home.
6. **`STACK_COMPOSE_PATH` указывает на директорию, а не на файл.** `docker stack deploy -c <path>` хочет именно файл; на директорию ругается `permission denied` или `is a directory`. Перед коммитом переменной — убедиться, что путь полный и заканчивается на `<compose>.yml`.
7. **Swarm не подхватывает новый push сам.** Если кто-то ожидает, что push в registry автоматически обновит сервис — это не так, swarm не поллит registry. Альтернативы (Watchtower / shepherd) конфликтуют с manual-gate для prod и держат registry-creds на хостах постоянно — лучше не использовать.
8. **`docker stack deploy` читает compose на стороне клиента.** В подходе с `ssh <host> bash -s` команда **выполняется на сервере**, и compose читается там же — то есть переменные окружения подставляются в shell сервера, а не runner'а. Если в compose есть `${VAR}` — передавать `VAR` через ssh (`SendEnv` + `AcceptEnv` в sshd_config, либо `env VAR=... bash -c '...'` в скрипте). Если переменных в compose нет (image-теги стоят буквально) — это всё неважно.

## Tier-замечание (GitLab CE)

На self-managed **CE** работают: вкладка Tests (JUnit), подсветка покрытия в дифе (cobertura), Code Quality-виджет. Security-репорты (`reports:sast`, `reports:dependency_scanning`, `reports:container_scanning`) с виджетами и Security Dashboard — это Ultimate; на CE соответствующие джобы (`bandit`, `pip-audit`, `container-scan`) всё равно полезны как **гейты** (падают на находках) и оставляют отчёт артефактом. В скиле security-джобы и сделаны так, чтобы ценность не зависела от тарифа.

## Definition of Done

- Стадии: `lint → test → build → scan`, имя — `lint` (не `check`).
- Есть `workflow.rules` — нет дублирующихся пайплайнов на branch+MR; есть `workflow.auto_cancel.on_new_commit: interruptible` — устаревшие pipeline'ы отменяются на новый push.
- `interruptible: true` в `default:`, `interruptible: false` явно на `.deploy` (или каждой deploy-джобе) — pipeline отменяется на новый push, но текущий `docker stack deploy` не прерывается на полпути.
- `default:` несёт установку `uv` и кэш по `uv.lock`; джобы на чужих образах сбрасывают его (`before_script: []`, `cache: []`).
- `lint`: `ruff check src` + `ruff format --check src` + codequality-отчёт; `bandit` и `pip-audit` как гейты.
- `test`: `unit-tests` и `integration-tests` отдают `reports:junit` и `reports:coverage_report` (cobertura); `integration-tests` содержит `needs: ["unit-tests"]` и работает в режиме `INTEGRATION_USE_EXTERNAL_INFRA=1`.
- `build-image`: Kaniko; тег `dev` строго на `develop`, `prod` строго на `main`, плюс `$CI_COMMIT_SHORT_SHA`; на прочих ветках джобы нет; поддержан `KANIKO_REGISTRY_MIRROR`.
- `container-scan`: Trivy по собранному образу, gate на HIGH/CRITICAL.
- `deploy`: одна джоба с rules для `develop` (авто, env `development`) и `main` (`when: manual`, env `production`); подключение через ssh к swarm-manager на образе alpine+openssh-client; `docker stack deploy -c <compose> --with-registry-auth <stack>` исполняется на сервере; `--password-stdin` + завершающий `docker logout`. Все переменные подключения раскиданы по group/project уровням с правильным environment scope, защищённые ветки настроены, deploy token со scope `read_registry` существует.
- Задан `timeout:` в `default:` и индивидуально на длинных джобах (`integration-tests`, `build-image`, `container-scan`, `deploy`); значения соответствуют Default-таблице (quick-режим) либо подтверждены пользователем (interactive-режим), и заданы в обоих случаях.
- В начале работы агент явно спросил у пользователя режим (quick / interactive) и придерживался его для трёх категорий: таймауты, базовые образы, имена/пути в deploy.
- Нет дублирования настроек по джобам; `.yml` валиден (`python -c "import yaml; yaml.safe_load(open('.gitlab-ci.yml'))"` или CI Lint в GitLab).
- В описании учтено tier-ограничение CE для security-виджетов.
