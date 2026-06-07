# Runbook: WB Landing Builder (конструктор лендингов)

> Операционное руководство для дежурного разработчика команды РКИ.
> Отвечает на вопрос: **«Что делать, когда что-то сломалось?»**
> Стек: Go-монолит + MongoDB + RabbitMQ + MinIO + nginx CDN. Production **не развёрнут** — документ ориентирован на локальный docker-compose и отладку инцидентов команды.

---

| Поле | Значение |
| --- | --- |
| **Сервис** | `wb-landing-builder` — backend конструктора лендингов |
| **Команда** | РКИ МАИ ([rki-mai](https://github.com/rki-mai)) |
| **Владелец** | @WhatTheMUCK |
| **On-call контакт** | @WhatTheMUCK / задачи на [Task tracker](https://github.com/orgs/rki-mai/projects/1/views/3) |
| **Обновлено** | 2026-06-07 |

---

## Быстрый старт

### Что делает сервис

Backend принимает HTTP-запросы от [`landing-editor`](https://github.com/rki-mai/landing-editor): регистрация/авторизация (JWT), хранение черновиков лендингов (мутации), асинхронная публикация в статику (HTML в MinIO). Конечные пользователи получают опубликованные страницы через **CDN** (`GET /publications/{id}/index.html`), минуя API.

### Где запущен

| Окружение | Адрес / ссылка |
| --- | --- |
| Production | _не развёрнут_ |
| Local (единая точка входа) | `http://localhost:8080` — nginx CDN |
| Swagger UI | `http://localhost:8080/swagger/index.html` |
| OpenAPI JSON (healthcheck) | `http://localhost:8080/swagger/doc.json` |
| Mongo Express | `http://localhost:8081` |
| MinIO Console | `http://localhost:9001` (minioadmin / minioadmin) |
| RabbitMQ Management | `http://localhost:15672` (guest / guest) |
| Grafana / Kibana | _нет (локальная разработка)_ |

### Как запустить локально

```bash
# 1. Клонировать репозиторий
git clone https://github.com/rki-mai/wb-landing-builder.git
cd wb-landing-builder/wb-landing-builder

# 2. Поднять полный стек (mongo, minio, rabbitmq, API, CDN)
make up
# эквивалент: docker compose up -d --build

# 3. Проверить статус
make ps

# 4. Сквозной smoke-тест (auth → storage → publishing → CDN)
make test-smoke
```

**Важно:** с хоста API доступен только через **CDN на `:8080`**. Контейнер `wb-landing-builder` не публикует порт наружу.

### Запуск тестов

```bash
make test-go        # go build ./...
make test-unit      # go test ./...
make test-smoke     # scripts/smoke.sh (нужен поднятый compose)
make test           # все три
```

---

## Архитектура (кратко)

```
landing-editor ──► nginx CDN :8080 ──► /api/* ──► wb-landing-builder (Go)
                                      │
                                      ├──► MongoDB (черновики, публикации, users)
                                      ├──► RabbitMQ (очередь publish.requests)
                                      └──► MinIO (bucket publications)

Посетитель ──► nginx CDN :8080 ──► /publications/* ──► MinIO (кэш nginx)
```

**Компоненты внутри монолита:** `auth/`, `storage/`, `publishing/` (+ in-process worker). Решение о монолите — DL-004 ([Decision Log](./Decision-Log.md)).

**Хронология ключевых изменений:** storage ([PR #1](https://github.com/rki-mai/wb-landing-builder/pull/1)), auth ([#4](https://github.com/rki-mai/wb-landing-builder/pull/4)), swagger ([#17](https://github.com/rki-mai/wb-landing-builder/pull/17)), owner_id ([#21](https://github.com/rki-mai/wb-landing-builder/pull/21)), publishing ([#27](https://github.com/rki-mai/wb-landing-builder/pull/27)), CDN ([#46](https://github.com/rki-mai/wb-landing-builder/pull/46) — в review). Полный реестр: [Project-Registry.md](../plans/project-history/Project-Registry.md).

---

## Конфигурация

Переменные читаются из env и `config/.env`. Источник defaults: `wb-landing-builder/config/config.go`.

### Сервер и лимиты

| Переменная | Описание | Default |
| --- | --- | --- |
| `PORT` | Порт HTTP backend | `8080` |
| `ENVIRONMENT` | Окружение | `production` |
| `LOG_LEVEL` | Уровень логов | `info` |
| `RATE_LIMIT` | Мутаций / project / мин | `100` |
| `API_SECRET` | Внутренний секрет | `stub` |

### Auth / JWT

| Переменная | Default |
| --- | --- |
| `JWT_SECRET` | `dev-secret` |
| `JWT_EXPIRATION` | `15m` |
| `REFRESH_TOKEN_EXPIRATION` | `168h` |

### MongoDB

| Переменная | Default (compose) |
| --- | --- |
| `MONGO_HOST` | `mongo` |
| `MONGO_PORT` | `27017` |
| `MONGO_USER` / `MONGO_PASSWORD` | `admin` / `admin` |
| `MONGO_DATABASE` | `storage` |

### S3 / MinIO

| Переменная | Default |
| --- | --- |
| `S3_ENDPOINT` | `http://minio:9000` |
| `S3_ACCESS_KEY` / `S3_SECRET_KEY` | `minioadmin` / `minioadmin` |
| `S3_BUCKET` | `publications` |
| `S3_USE_PATH_STYLE` | `true` |

### Publishing

| Переменная | Default |
| --- | --- |
| `RABBITMQ_URL` | `amqp://guest:guest@rabbitmq:5672/` |
| `RABBITMQ_PUBLISH_QUEUE` | `publish.requests` |
| `PUBLISHING_CLI_PATH` | `/app/cli/generate.py` |
| `PUBLIC_BASE_URL` | `http://localhost:8080` |

> Локально конфиги в `docker-compose.yml`. Для staging/prod — _не настроено_.

---

## Эксплуатация

### Рестарт сервиса

```bash
cd wb-landing-builder/wb-landing-builder

# Полный перезапуск стека
make restart          # down + up --build

# Только API + worker (in-process)
docker compose restart wb-landing-builder

# CDN
docker compose restart cdn

# Инфраструктура
docker compose restart mongo minio rabbitmq
```

После изменения кода или env — пересборка образа:

```bash
docker compose up -d --build wb-landing-builder
# или
make rebuild    # swag + up --build
```

### Проверить статус

```bash
# Healthcheck API через CDN (используется Docker healthcheck)
curl -fsS http://localhost:8080/swagger/doc.json | head -c 100

# Статус контейнеров
docker compose ps

# Smoke-тест (рекомендуется после изменений)
make test-smoke

# Verbose smoke
SMOKE_VERBOSE=1 make test-smoke
```

**Отдельного `/health` нет.** Критерий «жив» — `200` на `/swagger/doc.json`.

### Посмотреть логи

```bash
make logs                                    # только API
docker compose logs -f wb-landing-builder    # landing-builder
docker compose logs -f cdn                   # landing-cdn
docker compose logs -f mongo rabbitmq minio  # зависимости
docker compose logs -f --tail 100            # все сервисы
```

**Publishing — что искать в логах API:**

- `Publication worker started`
- `Connected to RabbitMQ (queue=publish.requests)`
- `Failed to process publish task` / `Failed to init rabbitmq` / `Failed to init blob storage`

---

## Алерты и инциденты

> Production-алертов нет. Ниже — типовые сбои из истории проекта (issues и PR).

### Сводка инцидентов

| Симптом | Вероятная причина | Первое действие |
| --- | --- | --- |
| API не отвечает на `:8080` | CDN или backend down | `docker compose ps`, `curl /swagger/doc.json` |
| `Authentication failed` при старте | Mongo ещё не инициализирован | Дождаться healthy mongo, restart API ([#9](https://github.com/rki-mai/wb-landing-builder/issues/9)) |
| `schema.json: no such file` | Образ без schema | `docker compose up -d --build wb-landing-builder` ([#10](https://github.com/rki-mai/wb-landing-builder/issues/10)) |
| Все ошибки → HTTP 500 | Старый образ | Rebuild; ожидай 400/401/403/404/429 ([#22](https://github.com/rki-mai/wb-landing-builder/issues/22), [PR #24](https://github.com/rki-mai/wb-landing-builder/pull/24)) |
| Публикация застряла в `PENDING` | Worker / RabbitMQ / CLI | См. [Публикация не завершается](#публикация-не-завершается-pendingprocessing) |
| CDN 404 на `/publications/...` | Объект не в MinIO или статус не FINISHED | Проверить MinIO console, статус publication |
| `403 access denied` на чужой project | Ожидаемое поведение | owner_id ([#21](https://github.com/rki-mai/wb-landing-builder/pull/21)) |
| `429 rate limit exceeded` | >100 мутаций/мин на project | Подождать `Retry-After` или снизить частоту |

---

### API недоступен (healthcheck failed)

1. `docker compose ps` — все сервисы `healthy`?
2. Если `landing-builder` в `Restarting` / `unhealthy`:
   ```bash
   docker logs landing-builder --tail 100
   ```
3. Проверить зависимости:
   ```bash
   docker compose ps mongo minio rabbitmq
   ```
4. Рестарт API: `docker compose restart wb-landing-builder`
5. Если CDN healthy, а API нет — проблема в backend; если оба down — `docker compose up -d --build`
6. Не помогло за 10 минут — issue в [wb-landing-builder](https://github.com/rki-mai/wb-landing-builder/issues), тег @WhatTheMUCK

---

### MongoDB: Authentication failed

**Контекст:** [wb-landing-builder#9](https://github.com/rki-mai/wb-landing-builder/issues/9) — гонка при первом cold start.

**Симптом в логах API:**

```
Failed to init draft repository: mongo ping error: ... Authentication failed.
```

**Шаги:**

1. `docker compose logs mongo --tail 50` — replica set и user `admin` созданы?
2. Дождаться `mongo` → `(healthy)`
3. `docker compose restart wb-landing-builder`
4. Если повторяется на **чистом volume:**
   ```bash
   docker compose down -v
   docker compose up -d --build
   ```
   ⚠️ Удалит все данные Mongo и MinIO.

---

### schema.json not found в контейнере

**Контекст:** [wb-landing-builder#10](https://github.com/rki-mai/wb-landing-builder/issues/10), fix в [PR #12](https://github.com/rki-mai/wb-landing-builder/pull/12).

**Симптом:**

```
failed to read schema file: open /app/storage/schema.json: no such file or directory
```

**Шаги:**

1. Пересобрать образ:
   ```bash
   docker compose up -d --build wb-landing-builder
   ```
2. Проверить файл в контейнере:
   ```bash
   docker exec landing-builder ls -la /app/storage/schema.json
   ```

---

### Публикация не завершается (PENDING/PROCESSING)

**Нормальный flow:** `PENDING` → `PROCESSING` → `FINISHED` | `FAILED`

1. **Статус и ошибка:**
   ```bash
   curl -sS -H "Authorization: Bearer $TOKEN" \
     "http://localhost:8080/api/v1/storage/$PROJECT_ID/publications/$PUB_ID" | jq .
   ```
2. **Логи worker** (in-process в `landing-builder`):
   ```bash
   docker logs landing-builder --tail 100 -f
   ```
3. **RabbitMQ** — http://localhost:15672, очередь `publish.requests`: сообщения застряли?
4. **MinIO** — http://localhost:9001, bucket `publications`, путь `publications/{id}/index.html`
5. **CLI рендера:**
   ```bash
   docker exec landing-builder python3 /app/cli/generate.py --help
   docker exec landing-builder ls -la /app/cli/generate.py
   ```
6. **Черновик существует?** Без мутаций POST publications → `404`
7. Рестарт: `docker compose restart wb-landing-builder`

Подробнее: [`publishing/README.md`](https://github.com/rki-mai/wb-landing-builder/blob/main/wb-landing-builder/publishing/README.md)

---

### CDN: 404 на опубликованный лендинг

1. Publication в статусе `FINISHED`? (не `PENDING`)
2. Объект в MinIO:
   ```bash
   curl -sSI "http://localhost:9000/publications/publications/$PUB_ID/index.html"
   ```
3. Через CDN:
   ```bash
   curl -sSI "http://localhost:8080/publications/$PUB_ID/index.html"
   # X-Cache-Status: HIT|MISS
   ```
4. `PUBLIC_BASE_URL` = `http://localhost:8080` (иначе `public_url` в API неверный)
5. Конфиг nginx: `docker/cdn/nginx.conf` — proxy на `minio:9000/publications/publications/`
6. `docker compose restart cdn minio`

---

### RabbitMQ / MinIO недоступны при старте

**RabbitMQ:**

```bash
docker compose logs rabbitmq --tail 50
curl -u guest:guest http://localhost:15672/api/overview
docker compose restart rabbitmq wb-landing-builder
```

**MinIO:**

```bash
docker compose logs minio --tail 50
# Console :9001 — bucket "publications" существует?
docker compose restart minio wb-landing-builder
```

---

### Frontend (landing-editor) не видит backend

1. Backend: `curl -fsS http://localhost:8080/swagger/doc.json`
2. Editor локально: `npm run dev` → `:5173`
3. CORS/proxy: [landing-editor#25](https://github.com/rki-mai/landing-editor/issues/25) — vite proxy может быть не настроен; тестировать через Swagger или прямые curl
4. JWT: [landing-editor#30](https://github.com/rki-mai/landing-editor/issues/30), [#31](https://github.com/rki-mai/landing-editor/issues/31) — refresh token на фронте в работе

---

## Откат (Rollback)

> Kubernetes/Helm не используется. Откат = предыдущий образ или чистый volume.

### Откат кода (docker)

```bash
git checkout main   # или нужный tag/commit
docker compose up -d --build wb-landing-builder
make test-smoke
```

### Откат данных (полный сброс)

```bash
docker compose down -v
docker compose up -d --build
make test-smoke
```

⚠️ Удаляет MongoDB, MinIO, RabbitMQ volumes. Использовать только локально.

### Откат миграций БД

Явных миграций (goose/flyway) **нет**. Схема Mongo создаётся приложением. При сбросе — `down -v`.

---

## Порядок запуска (compose)

```
mongo ──┬──► wb-landing-builder ──► cdn (:8080)
minio ──┤              ▲
rabbitmq┘              └── worker in-process

mongo ──► mongo-express (:8081)
```

Все три инфра-сервиса должны быть `healthy` до старта API. CDN ждёт API + MinIO.

---

## Контакты и эскалация

| Кому | Когда | Контакт |
| --- | --- | --- |
| @WhatTheMUCK | Publishing, CDN, CI/neuroreview, инфра compose | GitHub / Task tracker |
| @NamerPRO | Storage, Swagger/huma, API контракт | GitHub |
| @mregor787 | Auth, owner_id, projects model | GitHub |
| @Nifacy | landing-editor, docker refactor | GitHub |

**Эскалация:** issue в соответствующем репозитории → комментарий с логами (`docker logs …`) → mention владельца компонента.

---

## Changelog Runbook

| Дата | Изменения |
| --- | --- |
| 2026-06-07 | Первая версия для [docs#3](https://github.com/rki-mai/docs/issues/3): compose, smoke, инциденты #9/#10/#22, publishing, CDN |

---

## Связанные документы

- [API Reference](./API-Reference.md) — HTTP-контракт
- [Glossary](./Glossary.md) — термины
- [Decision Log](./Decision-Log.md) — DL-004 (монолит), DL-005 (CDN), DL-011 (async publishing)
- [publishing/README.md](https://github.com/rki-mai/wb-landing-builder/blob/main/wb-landing-builder/publishing/README.md) — детали pipeline
