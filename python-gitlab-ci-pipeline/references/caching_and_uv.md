# Caching & uv

Как обращаться с `uv` в CI: установка, кэш, переменные окружения.

## Канон: per-job sync + кэш `.uv-cache/`

```yaml
default:
  image: python:3.14-slim
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

И в каждой джобе свой `uv sync` с нужной группой зависимостей:

```sh
uv sync --only-group lint --no-install-project          # lint-джобы
uv sync --no-default-groups --group test --no-install-project   # test-джобы
```

Идея: кэшируется только **скачанный пакет-кэш** (`.uv-cache/`), а виртуальное окружение каждая джоба собирает заново из этого кэша — быстро, потому что ничего не качается по сети. Разные джобы ставят разные наборы зависимостей и не тащат лишнего.

- `--no-install-project` — не устанавливать сам пакет проекта в venv (в CI он не нужен — тесты импортируют `src/` напрямую, линтеру тем более). Ускоряет и убирает зависимость от собираемости проекта.
- `--only-group <g>` — поставить только указанную группу (без основных зависимостей и без других групп). Для линтеров.
- `--no-default-groups --group <g>` — основные зависимости проекта **не** ставить (`--no-default-groups`), поставить только группу `<g>`. Для тестов, если тестам не нужны рантайм-зависимости приложения; если нужны — убери `--no-default-groups`.
- Ключ кэша `key.files: [uv.lock]` — общий кэш на все джобы и ветки, инвалидируется при изменении lock-файла. Можно добавить в ключ имя джобы (`key: { files: [uv.lock], prefix: $CI_JOB_NAME }`), если джобы сильно мешают друг другу — обычно не нужно.

## Альтернатива: `uv sync` в `default.before_script` + кэш `.venv/`

```yaml
default:
  image: python:3.14-slim
  cache:
    key:
      files: [uv.lock]
    paths:
      - .venv/
      - .cache/uv/
  before_script:
    - pip install --no-cache-dir uv
    - uv sync

variables:
  UV_CACHE_DIR: .cache/uv
  UV_LINK_MODE: copy
  UV_FROZEN: "1"
```

Проще (один `uv sync` на все джобы, не думать про группы), но каждая джоба тащит **полный** набор зависимостей, включая то, что ей не нужно (линтеру — pytest, и наоборот), и кэш `.venv/` крупнее и чувствительнее к версии Python/раннера. Подходит маленьким проектам с тонким набором зависимостей.

**Рекомендация:** для прикладного сервиса с разделением на группы (`lint`, `test`, `security`, …) — канон (per-job sync, кэш `.uv-cache/`). Альтернативу выбирать осознанно, ради простоты на мелком проекте.

## Переменные окружения uv

- `UV_CACHE_DIR` — куда `uv` кладёт пакет-кэш. Указываем внутрь `$CI_PROJECT_DIR`, чтобы попадало под `cache.paths`.
- `UV_LINK_MODE=copy` — `uv` по умолчанию hardlink'ает файлы из кэша в venv; если кэш и venv на разных файловых системах раннера — будет варнинг/ошибка. `copy` это снимает ценой небольшого замедления.
- `UV_FROZEN=1` — `uv sync` не пытается обновить `uv.lock`, а падает, если он рассинхронизирован с `pyproject.toml`. В CI lock-файл меняться молча не должен.
- `UV_NO_PROGRESS=1` — без прогресс-баров в логах CI.
- `PIP_DISABLE_PIP_VERSION_CHECK` / `PIP_NO_CACHE_DIR` — для единственного `pip install uv` в `before_script`: не проверять обновления pip, не кэшировать (кэш pip тут не нужен — кэшируем `uv`).

## Джобы на чужих образах

`build-image` (Kaniko) и `container-scan` (Trivy) запускаются не на `python:3.14-slim`, поэтому переопределяют `image:` и **обязаны** сбросить унаследованное:

```yaml
build-image:
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  before_script: []     # иначе подтянется `pip install uv` из default — упадёт, в образе нет pip
  cache: []             # кэш `.uv-cache/` им не нужен
```

Забыть `before_script: []` — самая частая ошибка при добавлении такой джобы.
