# Build & Registry

Как собирать docker-образ в CI, как тегировать, и что для этого нужно за пределами `.gitlab-ci.yml`.

## Чем собирать: Kaniko vs docker-socket vs dind

| | docker socket binding | docker-in-docker (dind) | **Kaniko** |
|---|---|---|---|
| правка `config.toml` раннера | да (`volumes += /var/run/docker.sock`) | да (`privileged = true`) | **нет** |
| privileged-раннер | не нужен | **нужен** | не нужен |
| `DOCKER_HOST` / CI-переменные | надо чинить | надо чинить | не нужно |
| правка джобы в `.gitlab-ci.yml` | нет | да (`services: [docker:dind]`) | да (это и есть джоба) |
| root на хосте из любой джобы | **да** | **да** | нет |
| кэш-зеркало хостового daemon работает само | да | да | нет (нужен `--registry-mirror`) |

**Канон скила — Kaniko.** Он не требует ничего на стороне раннера (кроме сетевой достижимости реестра — см. ниже), не даёт джобам root на хосте, и вся «кастомизация» живёт в одной самодостаточной джобе — это нормальный портируемый паттерн, а не хак под конкретную инфру.

docker-socket разумен, только если уже так настроено и важно «зеркало из коробки». dind — только если раннер уже privileged и есть причина; иначе не вводить privileged ради сборки.

## Аутентификация в реестре

Kaniko читает `~/.docker/config.json` (точнее `/kaniko/.docker/config.json`):

```sh
mkdir -p /kaniko/.docker
echo "{\"auths\":{\"${CI_REGISTRY}\":{\"username\":\"${CI_REGISTRY_USER}\",\"password\":\"${CI_REGISTRY_PASSWORD}\"}}}" > /kaniko/.docker/config.json
```

`$CI_REGISTRY`, `$CI_REGISTRY_USER` (`gitlab-ci-token`), `$CI_REGISTRY_PASSWORD` (`$CI_JOB_TOKEN`), `$CI_REGISTRY_IMAGE` — предопределённые переменные GitLab, ничего настраивать не надо. Условие — в проекте включён **Container Registry** (Settings → General → Visibility → Container registry).

> Если используется внешний реестр (не GitLab) — креды класть в CI/CD-переменные (masked) и подставлять в тот же `config.json`; `--destination` указывать на внешний путь.

## Стратегия тегов

- `dev` — на ветке `develop`, `prod` — на ветке `main`. Скользящие теги: всегда «последний из этой ветки».
- `$CI_COMMIT_SHORT_SHA` — неизменяемый тег на каждый коммит. Нужен для отката («задеплой `:a1b2c3d`») и чтобы понимать, что внутри `:dev`/`:prod` сейчас.
- На прочих ветках и в MR образ не собираем — `rules:` с `IMAGE_TAG` внутри них; нет ветки `develop`/`main` → джобы нет вовсе.

Тег задаётся через `variables:` в `rules:`, а не через `if/else` в скрипте — декларативнее и видно в UI, какой тег будет.

```yaml
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      variables: { IMAGE_TAG: dev }
    - if: '$CI_COMMIT_BRANCH == "main"'
      variables: { IMAGE_TAG: prod }
```

Реестр копит образ на каждый коммит (`:<sha>`) — настрой **cleanup policy** (Settings → Packages and registries → Clean up image tags): хранить `dev`/`prod` + N последних SHA-тегов, остальное удалять.

## Pull-through кэш базовых образов

Kaniko не использует `registry-mirrors` хостового daemon — он тянет базовые образы сам. Подключается флагом:

```
--registry-mirror=<host:port> --insecure-pull   # --insecure-pull, если зеркало по HTTP
```

Имя сервиса кэша **не зашиваем в репозиторий** — выносим в CI/CD-переменную уровня группы/инстанса (`KANIKO_REGISTRY_MIRROR = registry-cache-dockerhub:5000`) и в джобе разворачиваем условно:

```sh
${KANIKO_REGISTRY_MIRROR:+--registry-mirror=$KANIKO_REGISTRY_MIRROR --insecure-pull}
```

`${VAR:+...}` — если переменная задана и непустая, подставляется текст; иначе пусто, и проект просто билдит без зеркала. Так один и тот же `.gitlab-ci.yml` работает в любой группе. Переменная должна быть обычной (не protected), иначе пайплайны не-protected веток её не увидят.

> Из контейнера джобы `127.0.0.1` — это сам контейнер, не хост. Кэш должен быть доступен по имени сервиса в общей docker-сети раннера и джоб, отсюда `registry-cache-dockerhub:5000`, а не `127.0.0.1:5000`.

## Что нужно на стороне раннера / GitLab (вне репозитория)

Это не про `.gitlab-ci.yml`, но без этого `build-image`/`container-scan` упадут — упоминай при подключении пайплайна:

1. **Container Registry включён** в проекте — иначе `$CI_REGISTRY_IMAGE` пустой.
2. **Реестр и GitLab достижимы из джобы по HTTPS.** Частый кейс self-managed: TLS снимает внешний reverse-proxy, а контейнер GitLab внутри сети слушает только :80 — тогда `docker login`/Kaniko ловят `connection refused` на :443. Лечится на раннере: `extra_hosts` в `[runners.docker]`, указывающие `gitlab.<domain>` и `registry.<domain>` на IP прокси, либо подключением прокси в общую docker-сеть. На прокси (nginx) должен быть `server`-блок и для `registry.<domain>` (тот же upstream, что и `gitlab.<domain>` — Omnibus различает их по `Host`-заголовку, прокидывай оригинальный `Host`).
3. **`KANIKO_REGISTRY_MIRROR`** — если хотите pull-through кэш: переменная уровня группы/инстанса.
4. **Доступ к `gcr.io`** (образ Kaniko) и `docker.io`/`ghcr.io` (`aquasec/trivy`) — раннер тянет их при старте джоб.
5. **Cleanup policy** для реестра — чтобы SHA-теги не копились бесконечно.
