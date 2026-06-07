# API Reference: WB Landing Builder

> Контракт HTTP API backend-сервиса [`wb-landing-builder`](https://github.com/rki-mai/wb-landing-builder).
> Документ отвечает на вопрос: «Как вызвать API конструктора лендингов?»
> Актуально для монолита с компонентами **auth**, **storage** и **publishing** (см. [Glossary](./Glossary.md)).

---

| Поле | Значение |
| --- | --- |
| **Версия API** | `v1` (префикс `/api/v1`) |
| **Base URL (local)** | `http://localhost:8080` |
| **Base URL (prod)** | _не развёрнут_ |
| **Swagger UI** | [`http://localhost:8080/swagger/index.html`](http://localhost:8080/swagger/index.html) |
| **OpenAPI (исходник)** | [`wb-landing-builder/docs/swagger.yaml`](https://github.com/rki-mai/wb-landing-builder/blob/main/wb-landing-builder/docs/swagger.yaml) |
| **Обновлено** | 2026-06-07 |

---

## Обзор

Backend предоставляет REST API для:

1. **Auth** — регистрация, вход, обновление JWT.
2. **Storage** — черновики лендингов: мутации и чтение snapshot по версии.
3. **Publications** — асинхронная публикация черновика в статику (MinIO + CDN).

Публичный просмотр опубликованных лендингов идёт **не через `/api`**, а через CDN: `GET /publications/{id}/index.html` (без авторизации).

```
Клиент (editor) ──► nginx :8080 ──► /api/* ──► Go backend
Посетитель      ──► nginx :8080 ──► /publications/* ──► MinIO (кэш nginx)
```

---

## Аутентификация

**Тип:** Bearer Token (JWT)

```http
Authorization: Bearer <access_token>
```

| Где получить токен | Эндпоинт |
| --- | --- |
| Регистрация + первый вход | `POST /api/v1/auth/register`, затем `POST /api/v1/auth/login` |
| Обновление пары токенов | `POST /api/v1/auth/refresh` |

**Параметры токенов (по умолчанию):**

| Токен | TTL |
| --- | --- |
| Access token | 15 минут (`expires_in`: 900 сек) |
| Refresh token | 168 часов (7 суток); ротация при refresh |

**Защищённые маршруты:** все `/api/v1/storage/*` и `GET /api/v1/auth/me`.

**Проверка владения проектом:** `project_id` привязан к пользователю при первой мутации. Чужой проект → `403 Forbidden`.

---

## Общие правила

| Параметр | Значение |
| --- | --- |
| Формат запроса | `application/json` |
| Формат ответа | `application/json` (кроме `DELETE` → `204 No Content`) |
| Кодировка | UTF-8 |
| Идентификаторы элементов | `lb-<номер>` (regex: `^lb-.*$`) |
| Корень дерева | `parentId: "root"` |

### Коды ответов

| Код | Значение |
| --- | --- |
| 200 OK | Успех |
| 201 Created | Ресурс создан |
| 204 No Content | Успешное удаление без тела |
| 400 Bad Request | Ошибка валидации или формата |
| 401 Unauthorized | Нет или невалидный JWT |
| 403 Forbidden | Нет доступа к проекту |
| 404 Not Found | Ресурс не найден |
| 409 Conflict | Конфликт (например, email уже занят) |
| 413 Payload Too Large | Тело запроса > 1 MB (мутации) |
| 429 Too Many Requests | Превышен rate limit |
| 500 Internal Server Error | Ошибка сервера |

### Формат ошибки

```json
{
  "error": "human-readable message"
}
```

Примеры: `"missing Authorization header"`, `"access denied"`, `"invalid credentials"`, `"rate limit exceeded. Limit: 100 requests per 1m0s. Retry after: …"`.

---

## Auth

### POST /api/v1/auth/register

Регистрация нового пользователя.

**Auth:** не требуется

**Тело запроса:**

```json
{
  "email": "user@example.com",
  "password": "SuperSecretPass123"
}
```

**Ответ `201 Created`:**

```json
{
  "id": "507f191e810c19729de860ea",
  "email": "user@example.com"
}
```

| Код | Когда |
| --- | --- |
| 400 | Пустые поля, невалидный JSON |
| 409 | Пользователь с таким email уже существует |

---

### POST /api/v1/auth/login

Вход и получение пары токенов.

**Auth:** не требуется

**Тело запроса:**

```json
{
  "email": "user@example.com",
  "password": "SuperSecretPass123"
}
```

**Ответ `200 OK`:**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "550e8400-e29b-41d4-a716-446655440000",
  "expires_in": 900
}
```

| Код | Когда |
| --- | --- |
| 401 | Неверный email или пароль |

---

### POST /api/v1/auth/refresh

Обновление access/refresh токенов. Старый refresh token инвалидируется.

**Auth:** не требуется

**Тело запроса:**

```json
{
  "refresh_token": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Ответ `200 OK`:** тот же формат, что у login (`TokenResponse`).

| Код | Когда |
| --- | --- |
| 400 | Отсутствует `refresh_token` |
| 401 | Невалидный или просроченный refresh token |

---

### GET /api/v1/auth/me

Профиль текущего пользователя.

**Auth:** Bearer JWT

**Ответ `200 OK`:**

```json
{
  "id": "507f191e810c19729de860ea",
  "email": "user@example.com"
}
```

| Код | Когда |
| --- | --- |
| 401 | Нет или невалидный токен |
| 404 | Пользователь удалён |

---

## Storage (черновики)

`project_id` — строковый идентификатор лендинга/проекта на стороне клиента. Отдельного эндпоинта «создать проект» нет: владение фиксируется при первой мутации.

Формат мутаций и элементов — [ADR «Формат изменений черновика»](./ADRs/Формат%20Изменений%20Черновика.md).

### POST /api/v1/storage/{project_id}/mutations

Применить одну мутацию к черновику.

**Auth:** Bearer JWT

**Path:**

| Параметр | Тип | Описание |
| --- | --- | --- |
| `project_id` | string | ID проекта |

**Тело запроса:**

```json
{
  "operation": "create",
  "data": {
    "element": "container",
    "id": "lb-1",
    "parentId": "root",
    "index": 0,
    "styles": {
      "display": "flex",
      "flexDirection": "column",
      "padding": "20px"
    }
  }
}
```

**Операции:**

| `operation` | Обязательные поля в `data` |
| --- | --- |
| `create` | `element`, `id`, `parentId`, `index` |
| `update` | `id`, `fields` (частичное обновление: `value`, `styles`, `parentId`, `index`, `src`, `alt`) |
| `delete` | `id` (каскадное удаление потомков) |

**Типы элементов:** `text`, `container`, `link`, `image`, `button`.

**Ответ `200 OK`:**

```json
{
  "status": "ok",
  "version": "6"
}
```

| Код | Когда |
| --- | --- |
| 400 | Невалидная схема, неизвестная операция, элемент не найден (update) |
| 403 | Проект принадлежит другому пользователю |
| 404 | Черновик/элемент не найден |
| 413 | Размер тела > 1 MB |
| 429 | Rate limit (см. раздел ниже) |

---

### GET /api/v1/storage/{project_id}

Получить актуальный snapshot черновика.

**Auth:** Bearer JWT

**Ответ `200 OK`:**

```json
{
  "version": 6,
  "elements": [
    {
      "id": "lb-1",
      "element": "container",
      "parentId": "root",
      "index": 0,
      "styles": { "display": "flex", "padding": "20px" },
      "version": 1,
      "deleted": false
    }
  ]
}
```

| Код | Когда |
| --- | --- |
| 403 | Чужой проект |
| 404 | Черновик не найден |

---

### GET /api/v1/storage/{project_id}/versions/{version}

Получить snapshot черновика на конкретной версии.

**Auth:** Bearer JWT

**Path:**

| Параметр | Тип | Описание |
| --- | --- | --- |
| `project_id` | string | ID проекта |
| `version` | integer | Номер версии |

**Ответ `200 OK`:** тот же формат, что у `GET …/storage/{project_id}`.

---

## Publications (управление публикациями)

Публикация **асинхронная**: `POST` создаёт запись со статусом `PENDING`, worker обрабатывает очередь RabbitMQ и складывает артефакты в MinIO.

**Статусы:** `PENDING` → `PROCESSING` → `FINISHED` | `FAILED`

### GET /api/v1/storage/{project_id}/publications

Список ID публикаций проекта (от новых к старым).

**Auth:** Bearer JWT

**Ответ `200 OK`:**

```json
{
  "ids": ["550e8400-e29b-41d4-a716-446655440000"]
}
```

Пустой список: `{ "ids": [] }`.

---

### POST /api/v1/storage/{project_id}/publications

Создать публикацию из **последнего** snapshot черновика.

**Auth:** Bearer JWT

**Тело:** не обязательно (можно `{}` или пустое).

**Ответ `201 Created`:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "project_id": "demo-project",
  "version": 0,
  "status": "PENDING",
  "created_at": "2026-06-07T12:00:00Z",
  "public_url": "http://localhost:8080/publications/550e8400-e29b-41d4-a716-446655440000/index.html"
}
```

После завершения (`FINISHED`):

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "project_id": "demo-project",
  "version": 6,
  "assets_path": "s3://publications/publications/550e8400-e29b-41d4-a716-446655440000/",
  "status": "FINISHED",
  "public_url": "http://localhost:8080/publications/550e8400-e29b-41d4-a716-446655440000/index.html",
  "created_at": "2026-06-07T12:00:00Z"
}
```

При ошибке (`FAILED`): дополнительно поле `error_message`.

| Код | Когда |
| --- | --- |
| 404 | Черновик не найден |

---

### GET /api/v1/storage/{project_id}/publications/{id}

Метаданные и текущий статус публикации.

**Auth:** Bearer JWT

**Ответ `200 OK`:** объект `Publication` (см. выше).

---

### DELETE /api/v1/storage/{project_id}/publications/{id}

Удалить публикацию и файлы в object storage.

**Auth:** Bearer JWT

**Ответ:** `204 No Content`

---

## Public CDN (без авторизации)

Раздаётся **nginx CDN**, не Go-backend. См. DL-005, DL-016 в [Decision Log](./Decision-Log.md).

### GET /publications/{publication_id}/index.html

HTML опубликованного лендинга.

**Пример:**

```http
GET /publications/550e8400-e29b-41d4-a716-446655440000/index.html
```

**Заголовки ответа (CDN):**

| Заголовок | Значение |
| --- | --- |
| `Cache-Control` | `public, max-age=300` |
| `X-Cache-Status` | `HIT` / `MISS` (nginx cache) |

### GET /publications/{publication_id}/{path}

Любой файл bundle (CSS, JS, медиа — по мере добавления в сборку).

Поле `public_url` в API строится как `{PUBLIC_BASE_URL}/publications/{id}/index.html` (env `PUBLIC_BASE_URL`, по умолчанию `http://localhost:8080`).

---

## Rate Limiting

| Область | Лимит | Эндпоинт |
| --- | --- | --- |
| Мутации черновика | 100 запросов / проект / минута | `POST …/mutations` |

При `429`:

- Тело: `{"error":"rate limit exceeded…"}`
- Заголовки: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`

Настраивается env `RATE_LIMIT` (default: `100`).

---

## Локальная разработка

### docker-compose (полный стек)

| Сервис | URL / порт |
| --- | --- |
| API Gateway + CDN + Swagger | `http://localhost:8080` |
| MongoDB | `localhost:27017` |
| Mongo Express | `http://localhost:8081` |
| MinIO S3 | `http://localhost:9000` |
| MinIO Console | `http://localhost:9001` |
| RabbitMQ Management | `http://localhost:15672` |

Маршрутизация nginx:

- `/api/*`, `/swagger/*` → backend `wb-landing-builder:8080`
- `/publications/*` → MinIO bucket `publications`

### docker-compose.dev.yml

Backend напрямую на `http://localhost:8080` (без CDN/nginx).

### Smoke-тест

Скрипт `scripts/smoke.sh` в репозитории `wb-landing-builder` — сквозная проверка auth → storage → publishing.

---

## Сводная таблица эндпоинтов

| # | Method | Path | Auth | Компонент |
| --- | --- | --- | --- | --- |
| 1 | POST | `/api/v1/auth/register` | — | auth |
| 2 | POST | `/api/v1/auth/login` | — | auth |
| 3 | POST | `/api/v1/auth/refresh` | — | auth |
| 4 | GET | `/api/v1/auth/me` | JWT | auth |
| 5 | POST | `/api/v1/storage/{project_id}/mutations` | JWT | storage |
| 6 | GET | `/api/v1/storage/{project_id}` | JWT | storage |
| 7 | GET | `/api/v1/storage/{project_id}/versions/{version}` | JWT | storage |
| 8 | GET | `/api/v1/storage/{project_id}/publications` | JWT | publishing |
| 9 | POST | `/api/v1/storage/{project_id}/publications` | JWT | publishing |
| 10 | GET | `/api/v1/storage/{project_id}/publications/{id}` | JWT | publishing |
| 11 | DELETE | `/api/v1/storage/{project_id}/publications/{id}` | JWT | publishing |
| — | GET | `/publications/{id}/index.html` | — | CDN |
| — | GET | `/swagger/index.html` | — | docs |
| — | GET | `/swagger/doc.json` | — | OpenAPI JSON |

---

## Changelog

| Версия | Дата | Изменения |
| --- | --- | --- |
| v1.0 | 2026-05-11 | Auth-компонент: register, login, refresh, me ([wb-landing-builder#4](https://github.com/rki-mai/wb-landing-builder/pull/4)) |
| v1.0 | 2026-05-18 | Storage: мутации и snapshot ([wb-landing-builder#1](https://github.com/rki-mai/wb-landing-builder/pull/1)) |
| v1.0 | 2026-05-25 | Publications API: CRUD + async pipeline ([wb-landing-builder#27](https://github.com/rki-mai/wb-landing-builder/pull/27)) |
| v1.0 | 2026-06-07 | Документ API Reference в репозитории docs ([docs#4](https://github.com/rki-mai/docs/issues/4)) |

---

## Связанные документы

- [Glossary](./Glossary.md) — термины (мутация, публикация, компонент)
- [Decision Log](./Decision-Log.md) — DL-005 (CDN), DL-006 (auth), DL-011 (async publishing)
- [ADR «Сервис публикации»](./ADRs/Сервис%20публикации%20лендингов.md)
- [ADR «Формат изменений черновика»](./ADRs/Формат%20Изменений%20Черновика.md)
