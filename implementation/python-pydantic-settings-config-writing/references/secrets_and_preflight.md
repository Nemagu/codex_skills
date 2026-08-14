# Секреты и предварительная проверка

## Модель

Settings хранит только путь:

```python
from pathlib import Path
from typing import Annotated

from pydantic import Field


ExternalPath = Annotated[Path, Field(strict=False)]


class PostgresCredentialsSettings(ConfigModel):
    password_file: ExternalPath
```

Не читать файл в validator/property и не создавать serializable `password`.

## Поставщик секрета

Отдельный provider читает секрет при создании клиента:

```python
from pathlib import Path

from pydantic import SecretStr


class FileSecretProvider:
    def read(self, secret_file: Path) -> SecretStr:
        value = secret_file.read_text(encoding="utf-8").strip()
        if not value:
            raise ValueError("файл секрета пуст")
        return SecretStr(value)
```

Не логировать `get_secret_value()`. Передавать секрет клиенту на минимальной
границе и не собирать DSN с паролем в обычную строку.

## Предварительная проверка при запуске

Preflight после structural validation проверяет:

- абсолютность пути;
- существование обычного читаемого secret-файла;
- существование writable-родителя runtime-файла;
- другие внешние артефакты, заданные worker-ом.

Он не создаёт каталоги и не выполняет сетевые запросы. Permissions/owner
проверять только по требованиям окружения.

## Ротация

Для ротации перечитать secret и пересоздать зависимый client/pool явным
lifecycle-компонентом. Повторное чтение property не меняет credentials уже
открытого соединения.

## Безопасный вывод

- Не логировать settings целиком.
- Не включать secret path без необходимости.
- Не помещать secret в exceptions, metrics и tracing.
- Использовать явный allowlist для startup-summary.
