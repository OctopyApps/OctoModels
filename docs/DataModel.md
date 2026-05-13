# Data Model

PostgreSQL + Drizzle ORM. Здесь описана схема БД: таблицы, поля, связи, индексы и решения по хранению данных.

## Содержание

- [Обзор схемы](#обзор-схемы)
- [Таблицы](#таблицы)
  - [users](#users)
  - [refresh_tokens](#refresh_tokens)
  - [api_keys](#api_keys)
  - [folders](#folders)
  - [chats](#chats)
  - [messages](#messages)
  - [message_groups](#message_groups)
  - [files](#files)
  - [file_versions](#file_versions)
- [Связи](#связи)
- [Индексы](#индексы)
- [Решения и комментарии](#решения-и-комментарии)

---

## Обзор схемы

```
users
  ├── refresh_tokens
  ├── api_keys
  ├── folders
  │     └── chats
  └── chats
        ├── messages
        │     ├── message_groups (для multi-режима)
        │     └── file_versions  (вложения / результаты)
        └── files
              └── file_versions
```

---

## Таблицы

### users

Базовая таблица пользователей. Пароль хранится как bcrypt-хэш.

```sql
CREATE TABLE users (
  id           TEXT        PRIMARY KEY,        -- формат: usr_<ulid>
  email        TEXT        NOT NULL UNIQUE,
  password     TEXT        NOT NULL,           -- bcrypt hash, rounds=12
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
// Drizzle schema
export const users = pgTable('users', {
  id:        text('id').primaryKey(),
  email:     text('email').notNull().unique(),
  password:  text('password').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Заметки:**
- `id` — ULID вместо UUID. Лексикографически сортируемый, удобнее в логах.
- Email приводится к нижнему регистру перед сохранением.
- `updated_at` обновляется триггером или в коде при каждом UPDATE.

---

### refresh_tokens

Хранит активные refresh-токены. Один пользователь может иметь несколько сессий (Web + iOS).

```sql
CREATE TABLE refresh_tokens (
  id          TEXT        PRIMARY KEY,         -- формат: rtok_<ulid>
  user_id     TEXT        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  TEXT        NOT NULL UNIQUE,     -- SHA-256 от токена, не сам токен
  expires_at  TIMESTAMPTZ NOT NULL,
  revoked_at  TIMESTAMPTZ,                     -- NULL = активен
  user_agent  TEXT,                            -- для отображения активных сессий
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
export const refreshTokens = pgTable('refresh_tokens', {
  id:        text('id').primaryKey(),
  userId:    text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  tokenHash: text('token_hash').notNull().unique(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  revokedAt: timestamp('revoked_at', { withTimezone: true }),
  userAgent: text('user_agent'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Заметки:**
- Хранится хэш токена (SHA-256), не сам токен — если БД утечёт, токены нельзя использовать.
- `revoked_at NOT NULL` = сессия завершена. Не удаляем сразу — нужно для детектирования reuse атак.
- Старые истёкшие записи чистятся крон-джобом раз в сутки.

---

### api_keys

Зашифрованные API-ключи пользователей к LLM-провайдерам.

```sql
CREATE TABLE api_keys (
  id               TEXT        PRIMARY KEY,    -- формат: key_<ulid>
  user_id          TEXT        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider         TEXT        NOT NULL,       -- 'openai' | 'anthropic' | 'google' | ...
  encrypted_key    TEXT        NOT NULL,       -- base64(iv + authTag + ciphertext)
  key_hint         TEXT,                       -- последние 4 символа ключа, для UI
  custom_endpoint  TEXT,                       -- для Ollama / Azure / self-hosted
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

  UNIQUE (user_id, provider)                   -- один ключ на провайдера
);
```

```typescript
export const apiKeys = pgTable('api_keys', {
  id:             text('id').primaryKey(),
  userId:         text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  provider:       text('provider').notNull(),
  encryptedKey:   text('encrypted_key').notNull(),
  keyHint:        text('key_hint'),
  customEndpoint: text('custom_endpoint'),
  createdAt:      timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:      timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  uniqueUserProvider: unique().on(t.userId, t.provider),
}))
```

**Формат `encrypted_key`:**
```
base64( IV(12 bytes) + AuthTag(16 bytes) + Ciphertext(N bytes) )
```
Алгоритм: AES-256-GCM. Подробности шифрования — в ADR-007.

**Заметки:**
- `UNIQUE (user_id, provider)` — один ключ на провайдера. Обновление = upsert.
- `key_hint` — например `"ABCD"` от ключа `sk-...ABCD`. Только для отображения в UI.
- `custom_endpoint` — для Ollama, LM Studio, Azure OpenAI с кастомным URL.

---

### folders

Папки для организации чатов. Простая плоская структура, без вложенности.

```sql
CREATE TABLE folders (
  id         TEXT        PRIMARY KEY,          -- формат: folder_<ulid>
  user_id    TEXT        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name       TEXT        NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
export const folders = pgTable('folders', {
  id:        text('id').primaryKey(),
  userId:    text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  name:      text('name').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

---

### chats

Чаты пользователей. Хранит метаданные и конфигурацию модели.

```sql
CREATE TABLE chats (
  id          TEXT        PRIMARY KEY,         -- формат: chat_<ulid>
  user_id     TEXT        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  folder_id   TEXT        REFERENCES folders(id) ON DELETE SET NULL,
  title       TEXT        NOT NULL DEFAULT 'Новый чат',
  mode        TEXT        NOT NULL DEFAULT 'single', -- 'single' | 'multi'
  tags        TEXT[]      NOT NULL DEFAULT '{}',
  config      JSONB       NOT NULL DEFAULT '{}',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
export const chats = pgTable('chats', {
  id:        text('id').primaryKey(),
  userId:    text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  folderId:  text('folder_id').references(() => folders.id, { onDelete: 'set null' }),
  title:     text('title').notNull().default('Новый чат'),
  mode:      text('mode').notNull().default('single'),
  tags:      text('tags').array().notNull().default([]),
  config:    jsonb('config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Структура `config` для `mode: 'single'`:**
```json
{
  "provider": "anthropic",
  "model": "claude-sonnet-4-5",
  "systemPrompt": "Ты опытный TypeScript разработчик.",
  "temperature": 0.7,
  "maxTokens": 4096,
  "topP": 1.0
}
```

**Структура `config` для `mode: 'multi'`:**
```json
{
  "models": [
    { "provider": "openai",     "model": "gpt-4o" },
    { "provider": "anthropic",  "model": "claude-sonnet-4-5" },
    { "provider": "google",     "model": "gemini-2.5-pro" }
  ],
  "systemPrompt": "...",
  "temperature": 0.7,
  "maxTokens": 4096
}
```

**Заметки:**
- `config` — JSONB, чтобы не плодить колонки под каждый параметр модели. Схема валидируется на уровне приложения (zod).
- `tags` — PostgreSQL массив текста. Для сложной фильтрации по тегам можно перейти на отдельную таблицу, но для v1 массива достаточно.
- `updated_at` обновляется при каждом новом сообщении — для сортировки списка чатов.

---

### message_groups

Группа ответов в мультимодельном режиме. Один пользовательский промпт → одна группа → N ответов от разных моделей.

```sql
CREATE TABLE message_groups (
  id         TEXT        PRIMARY KEY,          -- формат: grp_<ulid>
  chat_id    TEXT        NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
export const messageGroups = pgTable('message_groups', {
  id:        text('id').primaryKey(),
  chatId:    text('chat_id').notNull().references(() => chats.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Заметки:**
- Таблица намеренно минималистичная. Вся полезная информация — в `messages`.
- В `single` режиме `group_id` у сообщений всегда `NULL`.
- Группа создаётся в момент отправки промпта, до того как придут ответы.

---

### messages

Самая нагруженная таблица. Все сообщения всех чатов.

```sql
CREATE TABLE messages (
  id            TEXT        PRIMARY KEY,       -- формат: msg_<ulid>
  chat_id       TEXT        NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
  group_id      TEXT        REFERENCES message_groups(id) ON DELETE SET NULL,
  role          TEXT        NOT NULL,          -- 'user' | 'assistant' | 'system'
  content       TEXT        NOT NULL,
  provider      TEXT,                          -- NULL для role='user'
  model         TEXT,                          -- NULL для role='user'
  usage         JSONB,                         -- { promptTokens, completionTokens, totalTokens }
  duration_ms   INTEGER,                       -- время генерации в мс
  finish_reason TEXT,                          -- 'stop' | 'length' | 'error'
  meta          JSONB       NOT NULL DEFAULT '{}',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
export const messages = pgTable('messages', {
  id:           text('id').primaryKey(),
  chatId:       text('chat_id').notNull().references(() => chats.id, { onDelete: 'cascade' }),
  groupId:      text('group_id').references(() => messageGroups.id, { onDelete: 'set null' }),
  role:         text('role').notNull(),
  content:      text('content').notNull(),
  provider:     text('provider'),
  model:        text('model'),
  usage:        jsonb('usage'),
  durationMs:   integer('duration_ms'),
  finishReason: text('finish_reason'),
  meta:         jsonb('meta').notNull().default({}),
  createdAt:    timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Структура `usage`:**
```json
{
  "promptTokens": 512,
  "completionTokens": 348,
  "totalTokens": 860
}
```

**Поле `meta`** — для будущих расширений без миграций. Сейчас используется для:
```json
{
  "contextSourceChatId": "chat_01...",   // если сообщение — вставленный контекст из другого чата
  "isContextInsert": true
}
```

**Заметки:**
- Сообщения не обновляются после создания — append-only. Это упрощает кэширование и репликацию.
- `content` хранит финальный текст. Во время стриминга бэкэнд накапливает чанки в памяти, записывает в БД после `message_end`.
- При большом объёме данных (10M+ сообщений) — партиционирование по `chat_id` или диапазону дат.

---

### files

Файлы, загруженные пользователем в чат.

```sql
CREATE TABLE files (
  id          TEXT        PRIMARY KEY,         -- формат: file_<ulid>
  user_id     TEXT        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  chat_id     TEXT        REFERENCES chats(id) ON DELETE SET NULL,
  name        TEXT        NOT NULL,
  size        INTEGER     NOT NULL,            -- байты
  mime_type   TEXT        NOT NULL,
  storage_key TEXT        NOT NULL,            -- путь в S3 / object storage
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
export const files = pgTable('files', {
  id:         text('id').primaryKey(),
  userId:     text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  chatId:     text('chat_id').references(() => chats.id, { onDelete: 'set null' }),
  name:       text('name').notNull(),
  size:       integer('size').notNull(),
  mimeType:   text('mime_type').notNull(),
  storageKey: text('storage_key').notNull(),
  createdAt:  timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Заметки:**
- Содержимое файла не хранится в PostgreSQL — только метаданные. Сам файл — в S3-совместимом хранилище (R2, MinIO, AWS S3). `storage_key` — путь внутри бакета.
- Для небольших текстовых файлов (< 100 КБ) можно кэшировать содержимое в памяти при запросе, не идти в S3 каждый раз.
- `chat_id` — опциональный. Файл может быть загружен до привязки к чату.

---

### file_versions

Версии файла — оригинал и каждое изменение от модели.

```sql
CREATE TABLE file_versions (
  id          TEXT        PRIMARY KEY,         -- формат: fver_<ulid>
  file_id     TEXT        NOT NULL REFERENCES files(id) ON DELETE CASCADE,
  message_id  TEXT        REFERENCES messages(id) ON DELETE SET NULL,
  storage_key TEXT        NOT NULL,            -- путь к версии в S3
  is_original BOOLEAN     NOT NULL DEFAULT false,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
export const fileVersions = pgTable('file_versions', {
  id:         text('id').primaryKey(),
  fileId:     text('file_id').notNull().references(() => files.id, { onDelete: 'cascade' }),
  messageId:  text('message_id').references(() => messages.id, { onDelete: 'set null' }),
  storageKey: text('storage_key').notNull(),
  isOriginal: boolean('is_original').notNull().default(false),
  createdAt:  timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Заметки:**
- При загрузке файла создаётся первая версия с `is_original = true`.
- Каждый раз когда модель возвращает изменённый файл — бэкэнд сохраняет новую версию, `message_id` указывает на сообщение-ответ.
- Дифф вычисляется на лету при запросе (не хранится) — сравниваем содержимое двух версий из S3.

---

## Связи

```
users 1──N refresh_tokens
users 1──N api_keys
users 1──N folders
users 1──N chats
users 1──N files

folders 1──N chats          (folder_id nullable, SET NULL при удалении папки)

chats 1──N messages
chats 1──N message_groups
chats 1──N files            (chat_id nullable)

message_groups 1──N messages  (group_id nullable, только для mode='multi')

messages 1──N file_versions   (через message_id, nullable)

files 1──N file_versions
```

---

## Индексы

```sql
-- refresh_tokens: поиск по хэшу токена (при каждом запросе с refresh)
CREATE INDEX idx_refresh_tokens_token_hash ON refresh_tokens(token_hash);
CREATE INDEX idx_refresh_tokens_user_id    ON refresh_tokens(user_id);

-- api_keys: быстрый поиск ключей пользователя
CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);

-- chats: список чатов пользователя, отсортированный по дате (самый частый запрос)
CREATE INDEX idx_chats_user_id_updated_at ON chats(user_id, updated_at DESC);
CREATE INDEX idx_chats_folder_id          ON chats(folder_id) WHERE folder_id IS NOT NULL;

-- chats: поиск по названию
CREATE INDEX idx_chats_title_search ON chats USING gin(to_tsvector('russian', title));

-- messages: история чата (самый частый запрос в приложении)
CREATE INDEX idx_messages_chat_id_created_at ON messages(chat_id, created_at DESC);
CREATE INDEX idx_messages_group_id           ON messages(group_id) WHERE group_id IS NOT NULL;

-- files: файлы чата
CREATE INDEX idx_files_chat_id  ON files(chat_id) WHERE chat_id IS NOT NULL;
CREATE INDEX idx_files_user_id  ON files(user_id);

-- file_versions: версии файла
CREATE INDEX idx_file_versions_file_id ON file_versions(file_id);
```

---

## Решения и комментарии

**Почему ULID, а не UUID v4**

ULID лексикографически сортируемый — новые записи идут в конец индекса B-tree, меньше фрагментации. Плюс читается удобнее в логах: `msg_01HXY...` сразу понятно что это сообщение.

**Почему JSONB для `config` в чатах**

Параметры модели (temperature, top_p, presence_penalty и т.д.) различаются у каждого провайдера. Хранить их как отдельные колонки — или куча NULL-колонок, или по таблице на провайдера. JSONB проще, схема валидируется на уровне приложения через zod. Поиск по полям конфига не нужен, так что потери от JSONB минимальны.

**Почему файлы в S3, а не в PostgreSQL**

Хранить бинарные данные в БД — плохая практика: раздувает размер базы, усложняет бэкапы, медленнее отдача. S3-совместимое хранилище (Cloudflare R2 — дешевле, без egress fees) специально создано для этого. В БД только `storage_key`.

**Почему дифф не хранится**

Дифф между двумя текстовыми файлами вычисляется мгновенно (Myers diff algorithm). Хранить его — избыточно и создаёт проблему синхронизации (что если версия обновилась). Считаем на лету при каждом запросе.

**Append-only сообщения**

Сообщения создаются один раз и никогда не обновляются. Это упрощает кэширование (можно кэшировать по `id` бесконечно), репликацию и аудит. Единственное исключение — бэкэнд дописывает `usage` и `duration_ms` после завершения стрима, но это в рамках одной транзакции сразу после генерации.

**Мягкое удаление**

Soft delete (`deleted_at`) не используем в v1 — усложняет все запросы без реальной пользы. Если понадобится корзина — добавим в v2.
