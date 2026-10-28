# Changelog

## [1.1.0] - 2026-10-28

### Changed
- В ответ GET / добавлено поле version.
- Версия приложения обновлена до v1.1.0.



## [1.0.0] - 2026-10-27

### Added
- API гостевой книги на FastAPI.
- Эндпоинты GET /, GET /health, GET /messages.
- Эндпоинт POST /messages.
- Хранение сообщений в PostgreSQL.
- Dockerfile и Docker Compose.
- Управление зависимостями через uv.
- Автоматические проверки Ruff и pytest.
- Публикация Docker-образа в GHCR.