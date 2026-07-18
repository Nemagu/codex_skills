# Pipeline Skeleton

Эталонный `.gitlab-ci.yml` для Python-сервиса на `uv` со всеми джобами и отчётами. Бери его целиком как стартовую точку и адаптируй: подставь `<service>` (имя БД/пользователя для тестового Postgres), убери `nats` из `services:`, если в проекте нет NATS, поправь пути тестовых директорий под раскладку проекта, согласуй с пользователем значения `timeout:` (помечены как `<согласовано>` — см. раздел «Таймауты» в SKILL.md).

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS == null
    - if: $CI_COMMIT_TAG

stages:
  - lint
  - test
  - build
  - scan

default:
  image: python:3.14-slim
  timeout: <согласовано>   # короткие джобы (lint, unit-tests) живут с этим значением
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

# ---------------------------------------------------------------- lint

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

bandit:
  stage: lint
  script:
    - uv sync --only-group security --no-install-project
    - uv run bandit -r src -ll -x src/tests -f json -o gl-sast-report.json || true   # отчёт пишем всегда
    - uv run bandit -r src -ll -x src/tests -f screen                                 # gate: роняет на findings >= MEDIUM
  artifacts:
    when: always
    paths:
      - gl-sast-report.json

deps-audit:
  stage: lint
  script:
    - uv export --frozen --no-emit-project --no-dev -o requirements.txt
    - uv tool run pip-audit -r requirements.txt --strict --desc
  allow_failure: false

# ---------------------------------------------------------------- test

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
  timeout: <согласовано>   # длиннее default — поднимаются services: и идёт сетевое взаимодействие
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

# --------------------------------------------------------------- build

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

# ---------------------------------------------------------------- scan

container-scan:
  stage: scan
  timeout: <согласовано>   # скачивание БД уязвимостей + скан образа
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  before_script: []
  cache: []
  variables:
    TRIVY_USERNAME: $CI_REGISTRY_USER
    TRIVY_PASSWORD: $CI_REGISTRY_PASSWORD
    TRIVY_NO_PROGRESS: "true"
  script:
    - trivy image --severity HIGH,CRITICAL --ignore-unfixed
      --format template --template "@/contrib/gitlab.tpl"
      -o gl-container-scanning-report.json
      "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
    - trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1
      "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
  artifacts:
    when: always
    reports:
      container_scanning: gl-container-scanning-report.json
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

## Заметки по адаптации

- **Группы зависимостей.** В `pyproject.toml` ожидаются группы `lint` (ruff), `test` (pytest, pytest-cov), `security` (bandit). Если группы `security` нет — либо заведи её, либо в джобе `bandit` ставь его через `uv tool run bandit ...` (как `pip-audit`). `pip-audit` через `uv tool run` ставить в проектные группы не обязательно.
- **`pip-audit` и `--strict`.** `--strict` валит джобу при любой найденной уязвимости. Если нужна возможность временно «пропустить» известную — используй `--ignore-vuln <GHSA/CVE>` или сними `--strict` и оставь `--exit-code` дефолтным. Не превращай в `allow_failure: true` бездумно — гейт теряет смысл.
- **`bandit`: `-ll`** — репортить только findings уровня MEDIUM и выше (LOW обычно шум). Сначала прогон `-f json ... || true` (всегда пишет отчёт-артефакт, даже при находках), потом `-f screen` — он и есть gate (роняет джобу). Порядок важен: если gate-прогон идёт первым и падает, скрипт обрывается и отчёт не создаётся. Частый false-positive — `B108` на дефолтных значениях путей вида `/tmp/...` в pydantic-settings; правильнее не подавлять, а вынести дефолт из `/tmp` (секреты — в `/run/secrets/...`, рантайм-файлы — в писабельную папку под рабочей директорией приложения).
- **DAG через `needs:`.** Если хочется, чтобы `build-image` стартовал, не дожидаясь всей стадии `test` (а только нужных джоб), добавь `needs: ["unit-tests", "integration-tests", "ruff", "ruff-format"]`. По умолчанию оставлено на стадиях — проще и предсказуемее.
- **`GIT_DEPTH: "1"`** в `build-image` — Kaniko история git не нужна, shallow-клон быстрее. Для тестовых джоб не трогаем (некоторые плагины/инструменты смотрят историю).
- **Trivy `--ignore-unfixed`** — не валиться на уязвимости, для которых ещё нет фикса в базовом образе (иначе пайплайн краснеет по независящим причинам). Хочешь жёстче — убери флаг.
- **`/contrib/gitlab.tpl`** — шаблон GitLab-отчёта лежит внутри образа `aquasec/trivy`. Если версия образа его не содержит — взять template из репозитория Trivy или отдавать отчёт обычным артефактом без `reports:container_scanning`.
- Проверяй итог: `python -c "import yaml; yaml.safe_load(open('.gitlab-ci.yml'))"` локально и **CI Lint** в GitLab (Project → CI/CD → Editor / Lint) — он валидирует не только синтаксис, но и ключи.
