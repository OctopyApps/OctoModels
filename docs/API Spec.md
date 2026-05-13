# API Spec

Бэкэнд написан на Node.js + Hono + TypeScript. Все эндпоинты возвращают JSON, стриминг — через SSE. Базовый URL: `https://api.multillm.app` (локально `http://localhost:3000`).

## Содержание

- [Общие соглашения](#общие-соглашения)
- [Авторизация](#авторизация)
- [Ключи провайдеров](#ключи-провайдеров)
- [Модели](#модели)
- [Чаты](#чаты)
- [Сообщения](#сообщения)
- [Проксирование LLM](#проксирование-llm)
- [Файлы](#файлы)
- [Коды ошибок](#коды-ошибок)

---

## Общие соглашения

**Аутентификация** — все эндпоинты кроме `/auth/*` требуют заголовок:
```
Authorization: Bearer <accessToken>
```

**Content-Type** — всегда `application/json`, кроме загрузки файлов (`multipart/form-data`) и SSE-стримов.

**Формат дат** — ISO 8601: `2026-05-10T12:00:00Z`

**Пагинация** — курсорная, не offset-based:
```json
{
  "data": [...],
  "nextCursor": "cursor_xyz",
  "hasMore": true
}
```

**Версионирование** — пока без версии в URL. Когда появятся breaking changes — добавим `/v2/`.

---

## Авторизация

### `POST /auth/register`

Регистрация нового пользователя.

**Body:**
```json
{
  "email": "user@example.com",
  "password": "minlength8"
}
```

**Response `201`:**
```json
{
  "user": {
    "id": "usr_01HXY...",
    "email": "user@example.com",
    "createdAt": "2026-05-10T12:00:00Z"
  },
  "accessToken": "eyJ...",
  "refreshToken": "rtok_..."
}
```

**Ошибки:** `400` невалидные данные, `409` email уже занят.

---

### `POST /auth/login`

**Body:**
```json
{
  "email": "user@example.com",
  "password": "minlength8"
}
```

**Response `200`:**
```json
{
  "user": {
    "id": "usr_01HXY...",
    "email": "user@example.com"
  },
  "accessToken": "eyJ...",
  "refreshToken": "rtok_..."
}
```

**Ошибки:** `401` неверный email или пароль.

---

### `POST /auth/refresh`

Получить новый access token. Старый refresh token инвалидируется, выдаётся новый (rotation).

**Body:**
```json
{
  "refreshToken": "rtok_..."
}
```

**Response `200`:**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "rtok_..."
}
```

**Ошибки:** `401` токен невалиден или отозван.

---

### `POST /auth/logout`

Отзывает refresh token текущей сессии.

**Body:**
```json
{
  "refreshToken": "rtok_..."
}
```

**Response `204`** — no content.

---

### `GET /auth/me`

Данные текущего пользователя.

**Response `200`:**
```json
{
  "id": "usr_01HXY...",
  "email": "user@example.com",
  "createdAt": "2026-05-10T12:00:00Z"
}
```

---

## Ключи провайдеров

Ключи хранятся зашифровано. Сам ключ никогда не возвращается в ответах — только `hint` (последние 4 символа).

### `GET /keys`

Список подключённых провайдеров.

**Response `200`:**
```json
{
  "keys": [
    {
      "id": "key_01ABC...",
      "provider": "openai",
      "hint": "ABCD",
      "customEndpoint": null,
      "createdAt": "2026-05-10T12:00:00Z"
    },
    {
      "id": "key_01DEF...",
      "provider": "ollama",
      "hint": null,
      "customEndpoint": "http://localhost:11434",
      "createdAt": "2026-05-10T12:00:00Z"
    }
  ]
}
```

**Значения `provider`:** `openai`, `anthropic`, `google`, `mistral`, `groq`, `ollama`, `custom`

---

### `POST /keys`

Добавить или обновить ключ для провайдера. Если ключ для этого провайдера уже есть — перезаписывается.

**Body:**
```json
{
  "provider": "anthropic",
  "apiKey": "sk-ant-...",
  "customEndpoint": null
}
```

Для Ollama и других self-hosted провайдеров `apiKey` может быть пустым, но нужен `customEndpoint`:
```json
{
  "provider": "ollama",
  "apiKey": "",
  "customEndpoint": "http://localhost:11434"
}
```

**Response `201`:**
```json
{
  "id": "key_01ABC...",
  "provider": "anthropic",
  "hint": "SC3K",
  "customEndpoint": null,
  "createdAt": "2026-05-10T12:00:00Z"
}
```

**Ошибки:** `400` невалидный провайдер, `422` ключ не прошёл проверку (бэкэнд делает тестовый запрос к провайдеру).

---

### `DELETE /keys/:provider`

Удалить ключ провайдера. Чаты с этим провайдером переходят в read-only.

**Response `204`** — no content.

---

## Модели

### `GET /models`

Список всех доступных моделей по подключённым провайдерам. Возвращает только те провайдеры, для которых у пользователя есть ключ.

**Response `200`:**
```json
{
  "providers": [
    {
      "id": "openai",
      "name": "OpenAI",
      "models": [
        {
          "id": "gpt-4o",
          "name": "GPT-4o",
          "contextWindow": 128000,
          "supportsStreaming": true,
          "supportsVision": true,
          "inputPricePer1M": 5.0,
          "outputPricePer1M": 15.0
        },
        {
          "id": "gpt-4o-mini",
          "name": "GPT-4o mini",
          "contextWindow": 128000,
          "supportsStreaming": true,
          "supportsVision": true,
          "inputPricePer1M": 0.15,
          "outputPricePer1M": 0.6
        }
      ]
    },
    {
      "id": "anthropic",
      "name": "Anthropic",
      "models": [
        {
          "id": "claude-sonnet-4-5",
          "name": "Claude Sonnet 4.5",
          "contextWindow": 200000,
          "supportsStreaming": true,
          "supportsVision": true,
          "inputPricePer1M": 3.0,
          "outputPricePer1M": 15.0
        }
      ]
    }
  ]
}
```

> Цены — информационные, берутся из захардкоженного конфига. Не используются для биллинга.

---

## Чаты

### `GET /chats`

Список чатов пользователя, отсортированных по дате последнего сообщения.

**Query params:**
| Param | Тип | Описание |
|---|---|---|
| `cursor` | string | Курсор пагинации |
| `limit` | number | Кол-во записей, default 20, max 100 |
| `folderId` | string | Фильтр по папке |
| `q` | string | Поиск по названию чата |

**Response `200`:**
```json
{
  "data": [
    {
      "id": "chat_01XYZ...",
      "title": "Рефакторинг auth модуля",
      "mode": "single",
      "folderId": null,
      "tags": ["код", "typescript"],
      "lastMessage": {
        "role": "assistant",
        "preview": "Вот рефакторинг с учётом...",
        "createdAt": "2026-05-10T15:30:00Z"
      },
      "createdAt": "2026-05-10T14:00:00Z",
      "updatedAt": "2026-05-10T15:30:00Z"
    }
  ],
  "nextCursor": "cursor_abc",
  "hasMore": true
}
```

**Поле `mode`:** `single` — обычный чат, `multi` — мультимодельная сессия.

---

### `POST /chats`

Создать новый чат.

**Body:**
```json
{
  "title": "Новый чат",
  "mode": "single",
  "folderId": null,
  "tags": [],
  "config": {
    "provider": "anthropic",
    "model": "claude-sonnet-4-5",
    "systemPrompt": "Ты опытный TypeScript разработчик.",
    "temperature": 0.7,
    "maxTokens": 4096
  }
}
```

Для мультимодельного режима `config` выглядит так:
```json
{
  "mode": "multi",
  "config": {
    "models": [
      { "provider": "openai", "model": "gpt-4o" },
      { "provider": "anthropic", "model": "claude-sonnet-4-5" },
      { "provider": "google", "model": "gemini-2.5-pro" }
    ],
    "systemPrompt": "...",
    "temperature": 0.7,
    "maxTokens": 4096
  }
}
```

**Response `201`:** — полный объект чата (см. ниже `GET /chats/:id`).

---

### `GET /chats/:id`

Данные чата с конфигом. Без сообщений — они грузятся отдельно.

**Response `200`:**
```json
{
  "id": "chat_01XYZ...",
  "title": "Рефакторинг auth модуля",
  "mode": "single",
  "folderId": null,
  "tags": ["код"],
  "config": {
    "provider": "anthropic",
    "model": "claude-sonnet-4-5",
    "systemPrompt": "Ты опытный TypeScript разработчик.",
    "temperature": 0.7,
    "maxTokens": 4096
  },
  "createdAt": "2026-05-10T14:00:00Z",
  "updatedAt": "2026-05-10T15:30:00Z"
}
```

---

### `PATCH /chats/:id`

Обновить метаданные или конфиг чата. Все поля опциональны.

**Body:**
```json
{
  "title": "Новое название",
  "folderId": "folder_01...",
  "tags": ["код", "рефакторинг"],
  "config": {
    "model": "gpt-4o",
    "temperature": 0.5
  }
}
```

**Response `200`:** — обновлённый объект чата.

---

### `DELETE /chats/:id`

Удалить чат со всеми сообщениями.

**Response `204`** — no content.

---

### `POST /chats/:id/fork`

Создать копию чата с тем же контекстом. Используется для передачи контекста между чатами.

**Body:**
```json
{
  "targetChatId": "chat_01ABC...",
  "messageCount": 10,
  "insertAs": "system"
}
```

| Поле | Описание |
|---|---|
| `targetChatId` | ID существующего чата. Если не передать — создаётся новый |
| `messageCount` | Сколько последних сообщений взять, default 10 |
| `insertAs` | `"system"` или `"user"` — как вставить контекст в целевой чат |

**Response `201`:**
```json
{
  "chatId": "chat_01ABC...",
  "insertedMessageId": "msg_01..."
}
```

---

### `GET /chats/:id/export`

Экспорт чата.

**Query params:** `format` — `markdown` (default) или `json`.

**Response `200`** — файл с заголовком `Content-Disposition: attachment; filename="chat.md"`.

---

### Папки

#### `GET /folders`

```json
{
  "folders": [
    { "id": "folder_01...", "name": "Работа", "chatCount": 12, "createdAt": "..." }
  ]
}
```

#### `POST /folders`
```json
{ "name": "Личное" }
```
**Response `201`:** объект папки.

#### `PATCH /folders/:id`
```json
{ "name": "Новое имя" }
```

#### `DELETE /folders/:id`

Чаты в папке не удаляются — у них `folderId` становится `null`.

---

## Сообщения

### `GET /chats/:id/messages`

История сообщений с пагинацией (от новых к старым).

**Query params:**
| Param | Тип | Описание |
|---|---|---|
| `cursor` | string | Курсор пагинации |
| `limit` | number | Default 50, max 100 |

**Response `200`:**
```json
{
  "data": [
    {
      "id": "msg_01...",
      "chatId": "chat_01XYZ...",
      "role": "user",
      "content": "Отрефактори эту функцию:",
      "attachments": [
        {
          "id": "file_01...",
          "name": "auth.ts",
          "size": 2048,
          "mimeType": "text/typescript"
        }
      ],
      "createdAt": "2026-05-10T15:00:00Z"
    },
    {
      "id": "msg_02...",
      "chatId": "chat_01XYZ...",
      "role": "assistant",
      "content": "Вот рефакторинг...",
      "model": "claude-sonnet-4-5",
      "provider": "anthropic",
      "usage": {
        "promptTokens": 512,
        "completionTokens": 348,
        "totalTokens": 860
      },
      "durationMs": 4200,
      "attachments": [],
      "createdAt": "2026-05-10T15:00:05Z"
    }
  ],
  "nextCursor": "cursor_xyz",
  "hasMore": true
}
```

Для мультимодельных сессий `role: "assistant"` сообщения группируются по `groupId`:
```json
{
  "id": "msg_03...",
  "role": "assistant",
  "groupId": "grp_01...",
  "model": "gpt-4o",
  "provider": "openai",
  "content": "...",
  "usage": { ... },
  "durationMs": 3100
}
```

---

## Проксирование LLM

Это главная часть. Бэкэнд получает запрос, достаёт ключ пользователя, проксирует к провайдеру.

### `POST /proxy/stream`

Отправить сообщение в чат и получить стримингом ответ.

**Body:**
```json
{
  "chatId": "chat_01XYZ...",
  "message": {
    "content": "Объясни разницу между Promise и Observable",
    "attachmentIds": []
  }
}
```

**Response** — SSE стрим (`Content-Type: text/event-stream`):

```
event: message_start
data: {"messageId": "msg_04...", "model": "claude-sonnet-4-5", "provider": "anthropic"}

event: chunk
data: {"delta": "Promise"}

event: chunk
data: {"delta": " и Observable — это два разных подхода"}

event: chunk
data: {"delta": " к асинхронности..."}

event: message_end
data: {"usage": {"promptTokens": 24, "completionTokens": 312, "totalTokens": 336}, "durationMs": 3800}
```

При ошибке во время стрима:
```
event: error
data: {"code": "provider_error", "message": "Rate limit exceeded"}
```

> Сообщение сохраняется в БД на стороне бэкэнда — клиенту не нужно делать отдельный POST для сохранения.

---

### `POST /proxy/multi`

Мультимодельный запрос — один промпт к нескольким моделям параллельно.

**Body:**
```json
{
  "chatId": "chat_01XYZ...",
  "message": {
    "content": "Напиши функцию бинарного поиска на TypeScript"
  }
}
```

Модели берутся из конфига чата (тот самый `config.models` из `POST /chats`).

**Response** — SSE стрим, чанки размечены `modelId`:

```
event: session_start
data: {"sessionId": "sess_01...", "models": ["gpt-4o", "claude-sonnet-4-5", "gemini-2.5-pro"]}

event: chunk
data: {"modelId": "gpt-4o", "provider": "openai", "delta": "function binarySearch"}

event: chunk
data: {"modelId": "claude-sonnet-4-5", "provider": "anthropic", "delta": "const binarySearch"}

event: chunk
data: {"modelId": "gpt-4o", "provider": "openai", "delta": "<T>(arr: T[]"}

event: model_end
data: {"modelId": "gpt-4o", "usage": {"promptTokens": 18, "completionTokens": 156}, "durationMs": 2900}

event: model_end
data: {"modelId": "claude-sonnet-4-5", "usage": {"promptTokens": 18, "completionTokens": 203}, "durationMs": 3400}

event: model_end
data: {"modelId": "gemini-2.5-pro", "usage": {"promptTokens": 18, "completionTokens": 178}, "durationMs": 3100}

event: session_end
data: {"sessionId": "sess_01..."}
```

Если одна модель упала — остальные продолжают, приходит `model_error`:
```
event: model_error
data: {"modelId": "gpt-4o", "code": "provider_error", "message": "Rate limit exceeded"}
```

---

### `POST /proxy/stream` с файлом

Если к сообщению прикреплён файл (предварительно загруженный через `POST /files`), бэкэнд сам решает как передать его провайдеру — inline в контексте или как attachment.

**Body:**
```json
{
  "chatId": "chat_01XYZ...",
  "message": {
    "content": "Найди баги в этом файле",
    "attachmentIds": ["file_01..."]
  }
}
```

---

## Файлы

### `POST /files`

Загрузить файл. Используется перед отправкой сообщения с вложением.

**Request:** `multipart/form-data`
```
file: <binary>
chatId: chat_01XYZ...  (опционально, привязывает файл к чату)
```

**Response `201`:**
```json
{
  "id": "file_01...",
  "name": "auth.ts",
  "size": 4096,
  "mimeType": "text/typescript",
  "content": "import { ...",
  "createdAt": "2026-05-10T15:00:00Z"
}
```

> `content` возвращается только для текстовых файлов (< 500 КБ). Бинарные файлы и большие тексты — по отдельному запросу.

**Лимиты:** максимальный размер файла — 10 МБ. Поддерживаемые типы: любые текстовые форматы + PDF.

---

### `GET /files/:id`

Получить файл и его содержимое.

**Response `200`:**
```json
{
  "id": "file_01...",
  "name": "auth.ts",
  "size": 4096,
  "mimeType": "text/typescript",
  "content": "import { ...",
  "versions": [
    {
      "id": "fver_01...",
      "messageId": "msg_04...",
      "content": "import { ... (изменённая версия)",
      "createdAt": "2026-05-10T15:05:00Z"
    }
  ]
}
```

`versions` — история версий файла в рамках чата. Каждый раз когда модель возвращает изменённый файл — создаётся новая версия.

---

### `POST /files/:id/diff`

Получить дифф между двумя версиями файла.

**Body:**
```json
{
  "fromVersionId": "fver_00...",
  "toVersionId": "fver_01..."
}
```

Если `fromVersionId` не передан — сравнивается с оригиналом.

**Response `200`:**
```json
{
  "diff": [
    {
      "type": "unchanged",
      "lineStart": 1,
      "lineEnd": 5,
      "content": "import { createClient } from ..."
    },
    {
      "type": "removed",
      "lineStart": 6,
      "lineEnd": 8,
      "content": "const oldAuth = () => { ... }"
    },
    {
      "type": "added",
      "lineStart": 6,
      "lineEnd": 12,
      "content": "const auth = async () => { ... }"
    }
  ],
  "stats": {
    "additions": 6,
    "deletions": 3,
    "unchanged": 45
  }
}
```

---

### `POST /files/:id/apply`

Применить изменения из версии — сохранить как новый файл для скачивания.

**Body:**
```json
{
  "versionId": "fver_01...",
  "acceptedChunks": [0, 2, 4]
}
```

`acceptedChunks` — индексы из массива `diff` которые принимаем. Если не передать — применяются все изменения.

**Response `200`:**
```json
{
  "downloadUrl": "/files/file_01.../download?token=tmp_..."
}
```

---

### `GET /files/:id/download`

Скачать файл. Принимает одноразовый `token` из `/apply`.

**Response `200`** — бинарный файл с `Content-Disposition: attachment`.

---

## Коды ошибок

Все ошибки в одном формате:

```json
{
  "error": {
    "code": "validation_error",
    "message": "Поле email обязательно",
    "details": { "field": "email" }
  }
}
```

| HTTP код | `code` | Когда |
|---|---|---|
| `400` | `validation_error` | Невалидные данные в запросе |
| `401` | `unauthorized` | Нет или невалидный токен |
| `401` | `token_expired` | Access token истёк — нужен refresh |
| `403` | `forbidden` | Токен валиден, но нет доступа к ресурсу |
| `404` | `not_found` | Ресурс не найден |
| `409` | `conflict` | Ресурс уже существует (например, email при регистрации) |
| `422` | `invalid_api_key` | Ключ провайдера не прошёл проверку |
| `429` | `rate_limited` | Превышен лимит запросов |
| `502` | `provider_error` | Ошибка от LLM провайдера (пробрасывается наружу) |
| `503` | `provider_unavailable` | Провайдер недоступен |
| `500` | `internal_error` | Что-то пошло не так на нашей стороне |

Для `provider_error` и `provider_unavailable` в `details` пробрасывается оригинальный ответ провайдера:
```json
{
  "error": {
    "code": "provider_error",
    "message": "Rate limit exceeded",
    "details": {
      "provider": "openai",
      "providerCode": 429,
      "providerMessage": "You exceeded your current quota"
    }
  }
}
```
