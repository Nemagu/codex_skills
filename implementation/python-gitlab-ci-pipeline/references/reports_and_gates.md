# Reports & Gates

Отчёты, которые пайплайн отдаёт GitLab, и security-джобы, которые служат гейтами. Главное про тарифы: на self-managed **CE** работают вкладка Tests (JUnit), подсветка покрытия в дифе (cobertura), Code Quality-виджет. SAST / Dependency Scanning / Container Scanning **виджеты и Security Dashboard — это Ultimate**; на CE соответствующие джобы всё равно полезны как гейты (валят пайплайн на находках) и оставляют отчёт обычным артефактом.

## JUnit — результаты тестов

```yaml
  script:
    - uv run pytest ... --junitxml=report.xml ...
  artifacts:
    when: always           # отчёт нужен и при упавших тестах
    reports:
      junit: report.xml
```

Даёт вкладку **Tests** в пайплайне и диф «новые/исправленные/упавшие» в MR. Работает на CE. `when: always` обязателен — иначе при провале тестов артефакт не загрузится и отчёта не будет.

## Coverage — cobertura + процент

```yaml
  script:
    - uv run pytest ... --cov=src --cov-report=xml:coverage.xml --cov-report=term-missing ...
  coverage: '/TOTAL.+?(\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

- `reports:coverage_report` (cobertura) → подсветка покрытых/непокрытых строк прямо в дифе MR. Работает на CE.
- `coverage:`-regex вытаскивает число из лога `pytest-cov` → бейдж проекта и колонка в списке джоб. Регексу нужен `--cov-report=term` (или `term-missing`) в выводе.
- `--cov-report=html` + `artifacts:paths: htmlcov/` — скачиваемый HTML-отчёт, опционально.
- Порог: `--cov-fail-under=N` в `pytest`-вызове, **не** в YAML. По умолчанию не ставим (см. `python-pytest-testing` — покрытие это ориентир, не жёсткий CI-гейт). Если команда решила иначе — добавляется одним флагом.

## Code Quality — ruff

`ruff` умеет писать отчёт в формате GitLab Code Quality:

```yaml
ruff:
  stage: lint
  script:
    - uv sync --only-group lint --no-install-project
    - uv run ruff check src                                          # gate: роняет джобу
    - uv run ruff check src --output-format=gitlab > gl-code-quality-report.json   # только отчёт
  artifacts:
    when: always
    reports:
      codequality: gl-code-quality-report.json
```

Виджет «Code Quality: +N/-M» в MR с аннотациями в дифе. Гейтом служит первый прогон (`ruff check src`); второй ничего не валит, только пишет JSON. Два прогона дешевле, чем парсить — ruff быстрый.

`ruff format --check src` — отдельная джоба `ruff-format` (форматирование != линт; в UI видно, что именно сломано).

## SAST — bandit (gate-джоба)

`bandit` не умеет нативно формат GitLab SAST, поэтому делаем его **гейтом + артефактом**, без `reports:sast`:

```yaml
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
```

- `-ll` — только MEDIUM и выше (LOW обычно шум). `-r src` — рекурсивно по исходникам, `-x src/tests` — без тестов (там `assert` и т.п. дают шум).
- Сначала прогон с `|| true` (всегда пишет JSON-отчёт, даже когда есть находки), потом `-f screen` — он и есть gate. Если поставить gate-прогон первым и он упадёт, скрипт оборвётся и `gl-sast-report.json` не создастся (`Uploading artifacts ... no matching files`).
- Частый false-positive — `B108 hardcoded_tmp_directory` на дефолтных значениях путей `/tmp/...` в pydantic-settings. Не подавляй вслепую: секреты → `/run/secrets/...`, рантайм-файлы (healthcheck-маячки и т.п.) → писабельная папка под рабочей директорией приложения (например `/app/run/...`, созданная в Dockerfile). Только если перенос реально невозможен — `# nosec B108` на конкретной строке с комментарием.
- Альтернатива «по-взрослому» (для Ultimate-инстансов): `include: { template: Security/SAST.gitlab-ci.yml }` — GitLab сам определит Python, прогонит semgrep/bandit с правильным форматом отчёта и виджетом. На CE виджета не будет, так что для CE-инстанса проще держать свою джобу.
- Если группы `security` в `pyproject.toml` нет — ставь bandit через `uv tool run bandit ...` (не обязательно тянуть в проектные группы).

## Dependency audit — pip-audit (gate-джоба)

```yaml
deps-audit:
  stage: lint
  script:
    - uv export --frozen --no-emit-project --no-dev -o requirements.txt
    - uv tool run pip-audit -r requirements.txt --strict --desc
```

- `uv export` фиксирует разрешённые версии из `uv.lock` в `requirements.txt`; `pip-audit` сверяет их с базой уязвимостей (PyPI Advisory / OSV).
- `--strict` — джоба падает при любой найденной уязвимости. Временно «пропустить» известную — `--ignore-vuln GHSA-xxxx-...` (с комментарием почему), а не `allow_failure: true` на всю джобу.
- `--desc` — печатать описание уязвимостей в лог.
- `uv tool run` — pip-audit ставится изолированно, не в проектные группы.
- Для Ultimate-инстансов есть `include: { template: Security/Dependency-Scanning.gitlab-ci.yml }` — тогда будет виджет; на CE — нет, держим свою джобу.

## Container scanning — Trivy (gate-джоба)

После `build-image`, в стадии `scan`:

```yaml
container-scan:
  stage: scan
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

- Сканируем образ по тегу `$CI_COMMIT_SHORT_SHA` (он только что запушен `build-image`). `TRIVY_USERNAME/PASSWORD` — чтобы Trivy смог его вытянуть из приватного реестра.
- Первый прогон пишет отчёт в GitLab-формате (`reports:container_scanning` → виджет на Ultimate, на CE — просто артефакт). Второй прогон с `--exit-code 1` — гейт.
- `--ignore-unfixed` — не валиться на уязвимости без доступного фикса в апстриме базового образа (иначе пайплайн краснеет по независящим причинам). Хочешь жёстче — убери флаг и держи базовый образ свежим.
- `--severity HIGH,CRITICAL` — гейтим только серьёзное; LOW/MEDIUM можно смотреть в отчёте.
- `/contrib/gitlab.tpl` — шаблон лежит внутри образа `aquasec/trivy`; если конкретная версия его не содержит, возьми template из репозитория Trivy либо отдавай отчёт обычным артефактом без `reports:`.
- `rules:` повторяет условия `build-image` — без образа сканировать нечего.

## Сводка: что включаем всегда vs опционально

| Отчёт/гейт | Канон скила | Виджет на CE | Стоимость |
|---|---|---|---|
| JUnit (`reports:junit`) | да | да (Tests) | бесплатно |
| Coverage (cobertura + `coverage:`) | да | да (diff) | бесплатно |
| Code Quality (ruff `--output-format=gitlab`) | да | да | почти бесплатно |
| `bandit` (SAST gate) | да | нет (артефакт) | +1 джоба |
| `pip-audit` (deps gate) | да | нет (артефакт) | +1 джоба |
| Trivy (container scan gate) | да (раз собираем образ) | нет (артефакт) | +1 джоба |
| HTML-coverage артефакт | опц. | — | бесплатно |
| GitLab Security templates (`include:`) | альт. для Ultimate | да | — |
