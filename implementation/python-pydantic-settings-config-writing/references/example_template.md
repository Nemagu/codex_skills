# Шаблон новой технологии

## Сбор требований

До кода согласовать:

- точный client/package и установленную версию;
- обязательные connection-поля;
- authentication и file-based secrets;
- timeout/pool/retry с единицами;
- TLS и сертификаты;
- lifecycle и preflight;
- различия окружений;
- source и приоритет каждого значения.

## Модель

```python
from pathlib import Path
from typing import Annotated

from pydantic import Field


PositiveSeconds = Annotated[float, Field(gt=0)]
ExternalPath = Annotated[Path, Field(strict=False)]


class TechnologySettings(ConfigModel):
    endpoint: str
    credential_file: ExternalPath
    timeout_seconds: PositiveSeconds
```

Это форма, а не обязательный набор. Не копировать поля без требований.

## Материал технологии

Новый reference должен содержать:

1. Immutable-модели.
2. Явный mapping в client kwargs.
3. YAML-фрагмент без plaintext secrets.
4. Cross-field validation.
5. Preflight.
6. Тестовые случаи.
7. Ссылку из `SKILL.md`.

Не добавлять новый пример в общий монолитный файл.
