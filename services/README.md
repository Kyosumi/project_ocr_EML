# Сервис распознавания

Веб-сервис: пользователь загружает фото строки документа и получает распознанный текст.
Сейчас вместо модели работает **заглушка**: она возвращает фиксированный текст.
Каркас уже готов к тому, чтобы подключить обученную модель.

## Схема

```
браузер ──> frontend (nginx, :8080) ──/api/──> gateway (:8000) ──> recognizer (:8000)
```

| Сервис | Что делает | Технологии |
|---|---|---|
| `frontend` | Страница загрузки фото и показа текста; проксирует `/api/` в gateway | HTML/CSS/JS, nginx |
| `gateway` | Единая точка входа API: проверяет файл, вызывает recognizer | FastAPI, httpx |
| `recognizer` | Загружает модель и распознаёт текст (пока заглушка) | FastAPI, Pillow |
| `preprocessing` | Нарезка листа на строки (позже) | — |

Наружу открыт только `frontend`. Остальные сервисы доступны лишь внутри docker-сети.

## Запуск

Нужен [Docker Desktop](https://www.docker.com/products/docker-desktop/) (он должен быть запущен).
Из корня репозитория:

```bash
docker compose up --build
```

Затем откройте http://localhost:8080. Остановить: `Ctrl+C`, затем `docker compose down`.

## API

Интерактивная документация FastAPI (`/docs`) доступна изнутри контейнеров.
Основной запрос идёт через frontend:

```bash
curl -F "file=@line.png" http://localhost:8080/api/recognize
```

Ответ:

```json
{"text": "Милостивый государь, ...", "model": "stub", "stub": true}
```

| Код | Когда |
|---|---|
| `200` | Текст распознан |
| `400` | Пустой файл |
| `413` | Файл больше `MAX_UPLOAD_MB` (10 МБ) |
| `415` | Не картинка (поддерживаются JPEG, PNG, TIFF, BMP, WebP) |
| `422` | Картинку не удалось прочитать |
| `502` / `503` / `504` | recognizer ответил ошибкой / недоступен / не ответил вовремя |

## Настройки (переменные окружения)

Задаются в `docker-compose.yml`.

| Сервис | Переменная | По умолчанию | Смысл |
|---|---|---|---|
| recognizer | `RECOGNIZER_BACKEND` | `stub` | Какой распознаватель загрузить |
| recognizer | `STUB_TEXT` | текст-пример | Что отвечает заглушка |
| gateway | `RECOGNIZER_URL` | `http://recognizer:8000` | Адрес recognizer |
| gateway | `MAX_UPLOAD_MB` | `10` | Лимит размера файла (то же число стоит в `frontend/public/app.js`) |
| gateway | `RECOGNIZER_TIMEOUT` | `30` | Сколько секунд ждать recognizer |

## Разработка без Docker

У каждого Python-сервиса свои зависимости: тяжёлые ML-библиотеки из корневого
`requirements.txt` сервисам не нужны. Пример для recognizer (для gateway так же):

```bash
cd services/recognizer
python -m venv .venv
.venv\Scripts\activate            # Windows; на Linux/macOS: source .venv/bin/activate
pip install -r requirements-dev.txt
python -m pytest                  # тесты
uvicorn app.main:app --reload     # запуск на http://localhost:8000, документация на /docs
```

## Как подключить настоящую модель

1. В `recognizer/app/recognizers.py` добавить класс, унаследованный от `Recognizer`:
   в `__init__` загрузить `best.pt` (веса, алфавит, параметры), в `predict`
   подготовить картинку так же, как `Dataset`, прогнать модель и декодировать ответ.
2. Зарегистрировать его в `build_recognizer`, например под именем `crnn`.
3. Добавить в `recognizer/requirements.txt` CPU-версию torch.
4. В `docker-compose.yml` поставить `RECOGNIZER_BACKEND: crnn` и подключить папку `artifacts/` как volume.

Остальные сервисы менять не нужно: формат ответа остаётся тем же.
