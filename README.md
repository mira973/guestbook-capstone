# Guestbook Capstone

Небольшой REST API гостевой книги на **FastAPI** и **PostgreSQL**. Сервис позволяет сохранять и читать сообщения, проверять доступность приложения и получать приветствие.

Проект демонстрирует упаковку Python-приложения в Docker, управление зависимостями через `uv`, автоматические проверки в GitHub Actions, публикацию версионированных образов в GHCR и откат между релизами.

## 1. Требования

Для запуска приложения:

- Git;
- Docker с поддержкой `docker compose` (например, Docker Desktop);
- свободный порт `8000` на компьютере.

Для локальных проверок без контейнера дополнительно используются **Python 3.12**, **uv**, **Ruff** и **pytest**. Версия Python указана в `.python-version`, зависимости — в `pyproject.toml` и `uv.lock`.

## 2. Запуск через Docker Compose

Склонируйте репозиторий:

```bash
git clone https://github.com/mira973/guestbook-capstone.git
cd guestbook-capstone
```

Создайте локальный `.env` на основе шаблона:

```powershell
Copy-Item .env.example .env
```
Проверьте состояние контейнеров:

```bash
docker compose ps
```

Проверка доступности API (PowerShell):

```powershell
curl.exe http://localhost:8000/health
```

Ожидаемый ответ:

```json
{"status":"ok"}
```

Приложение доступно по адресу **http://localhost:8000**, а интерактивная документация FastAPI — по адресу **http://localhost:8000/docs**.

Для остановки без удаления данных:

```bash
docker compose down
```

**Важно:** команда `docker compose down -v` удаляет том PostgreSQL вместе с сохранёнными сообщениями. Не используйте её при обычном перезапуске или откате.

## 3. Конфигурация

Конфигурация приложения читается через `pydantic-settings` в `config.py`. Пример переменных находится в [`.env.example`](.env.example).

| Переменная | Назначение | Как используется |
| --- | --- | --- |
| `POSTGRES_PASSWORD` | Пароль PostgreSQL | Читается Docker Compose из `.env`; обязателен для запуска стека |
| `DATABASE_URL` | Строка подключения приложения к PostgreSQL | В контейнере `app` задаётся в `compose.yaml` с хостом `db` и `POSTGRES_PASSWORD` |
| `GREETING` | Приветствие на `GET /` | В текущем `compose.yaml` явно задано в `environment` |

При запуске через Docker Compose значения `DATABASE_URL` и `GREETING` из `compose.yaml` имеют приоритет над одноимёнными значениями из `.env` приложения. Изменение одного лишь `GREETING` в `.env` не изменит приветствие контейнеризованного приложения.

Файл `.env` добавлен в `.gitignore` и не должен попадать в Git. Не размещайте реальные пароли или токены в исходниках, README и коммитах.

## 4. API

| Метод | Путь | Описание |
| --- | --- | --- |
| `GET` | `/` | Возвращает приветствие; в версии `v1.1.0` также возвращает `version` |
| `GET` | `/health` | Проверка доступности HTTP-сервиса: `{"status":"ok"}` |
| `GET` | `/messages` | Список сообщений, начиная с самых новых |
| `POST` | `/messages` | Сохранение сообщения в PostgreSQL |
| `GET` | `/docs` | Интерактивная документация Swagger UI от FastAPI |

Пример тела запроса `POST /messages`:

```json
{
  "author": "Alice",
  "text": "Привет, гостевая книга!"
}
```

Пример отправки из PowerShell:

```powershell
Invoke-RestMethod -Uri http://localhost:8000/messages -Method Post -ContentType 'application/json; charset=utf-8' -Body '{"author":"Alice","text":"Hello!"}'
Invoke-RestMethod -Uri http://localhost:8000/messages -Method Get
```

Успешный `POST /messages` возвращает `{"ok":true}`. Таблица `messages` (`id`, `author`, `text`) создаётся при запуске сервиса, если отсутствует. Данные сохраняются в Docker volume `pgdata`.

**Ограничение:** `/health` проверяет ответ приложения, но не выполняет отдельную проверку подключения к PostgreSQL. Эндпоинта `/db-check` в текущем проекте нет.

## 5. Разработка и проверки

При установленном `uv`:

```bash
uv sync
uv run ruff check .
uv run pytest
```

Перед запуском тестов создайте `.env` по инструкции выше, чтобы `config.py` получил обязательные `GREETING` и `DATABASE_URL`. Текущие тесты проверяют `/` и `/health`; подключение к реальной базе и запись сообщений ими не покрываются.

## 6. CI/CD и релизы

Workflow находится в [`.github/workflows/ci.yml`](.github/workflows/ci.yml):

- на `push` и `pull_request` запускаются Ruff и pytest;
- после успешных проверок в `main` публикуется образ `ghcr.io/mira973/guestbook-capstone:latest`;
- при отправке Git-тега `vX.Y.Z` публикуется образ с соответствующим тегом версии.

[История GitHub Actions](https://github.com/mira973/guestbook-capstone/actions) · [Изменения по версиям](CHANGELOG.md)

Опубликованные версии для демонстрации: `v1.0.0` и `v1.1.0`.

### Запуск релиза и откат

Для запуска **опубликованного** образа вместо локальной сборки замените у сервиса `app` в `compose.yaml` строку `build: .` на:

```yaml
image: ghcr.io/mira973/guestbook-capstone:v1.1.0
```

Остальные настройки `app` и сервис `db` оставьте без изменений. Затем выполните:

```bash
docker compose pull app
docker compose up -d
```

Для **отката** измените только тег образа на `v1.0.0`, снова выполните `docker compose pull app` (если образ ещё не скачан) и `docker compose up -d`. Проверьте `/health` и `/`: в `v1.0.0` поле `version` в ответе `/` отсутствует.

Данные PostgreSQL остаются в `pgdata`. Откат образа **не откатывает схему или содержимое базы данных**. Текущий пример обновления между `v1.0.0` и `v1.1.0` не меняет схему БД. Простое пересоздание одного контейнера через Compose также **не гарантирует обновление без простоя**.
