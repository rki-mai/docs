# Требования: Конструктор лендингов (РКИ MVP 1.0)

> Функциональные (FR) и нефункциональные (NFR) требования к CMS для создания одностраничных лендингов.
> Основано на ADR и фактической реализации (2026-06-07).

---

| Поле | Значение |
| --- | --- |
| **Проект** | РКИ — конструктор лендингов (Wildberries / учебный MVP) |
| **Версия** | MVP 1.0 |
| **Автор** | @WhatTheMUCK |
| **Согласовано** | Команда РКИ МАИ |
| **Дата** | 2026-06-07 |
| **Трекер задач** | [Task tracker](https://github.com/orgs/rki-mai/projects/1/views/3) |

**Легенда статуса:** ✅ реализовано · 🔄 в работе · 📋 запланировано

---

## Контекст

Команда разрабатывает **CMS для лендингов**: редактор создаёт одностраничный сайт в визуальном интерфейсе, изменения сохраняются как структурированный черновик (JSON + мутации), после публикации посетитель получает статическую HTML-страницу через CDN.

**Пользователи:**

- **Редактор** — сотрудник/мерчант, создаёт и редактирует лендинг ([`landing-editor`](https://github.com/rki-mai/landing-editor)).
- **Конечный пользователь** — посетитель опубликованной страницы (только просмотр, без API).

**Архитектурный контекст:** backend — монолит [`wb-landing-builder`](https://github.com/rki-mai/wb-landing-builder) (компоненты auth, storage, publishing). Черновик и публикация — **разные сущности** ([ADR «Разделение лендинга»](./ADRs/Разделение%20лендинга.md)). Production-окружение **не развёрнуто**; NFR ниже ориентированы на MVP и локальный docker-compose.

---

## Функциональные требования (FR)

### Обязательные (Must Have)

| ID | Требование | Критерий приёмки | Статус |
| --- | --- | --- | --- |
| FR-01 | Система должна регистрировать пользователя по email и паролю | `POST /api/v1/auth/register` → `201`, возвращает `id` и `email`; дубликат email → `409` | ✅ [PR #4](https://github.com/rki-mai/wb-landing-builder/pull/4) |
| FR-02 | Система должна аутентифицировать пользователя и выдавать JWT | `POST /api/v1/auth/login` → `access_token`, `refresh_token`, `expires_in` | ✅ |
| FR-03 | Система должна обновлять пару токенов по refresh token | `POST /api/v1/auth/refresh` → новая пара; старый refresh инвалидируется | ✅ |
| FR-04 | Защищённые API должны требовать Bearer JWT | Запрос без/с невалидным токеном → `401` | ✅ |
| FR-05 | Система должна хранить черновик лендинга как дерево элементов | Snapshot: `{ version, elements[] }`; id элементов `lb-*`, `parentId: root` | ✅ [PR #1](https://github.com/rki-mai/wb-landing-builder/pull/1) |
| FR-06 | Редактор должен применять атомарные мутации к черновику | `POST …/mutations` с `create` / `update` / `delete` → `{ status: ok, version }` | ✅ [docs#15](https://github.com/rki-mai/docs/pull/15) |
| FR-07 | Система должна валидировать мутации по JSON Schema | Невалидная мутация → `400` с `{"error":"…"}` | ✅ |
| FR-08 | Система должна возвращать актуальный и исторический snapshot | `GET …/storage/{project_id}` и `…/versions/{version}` → JSON черновика | ✅ [PR #25](https://github.com/rki-mai/wb-landing-builder/pull/25) |
| FR-09 | Доступ к черновику только у владельца (`owner_id`) | Чужой `project_id` → `403 Forbidden` | ✅ [PR #21](https://github.com/rki-mai/wb-landing-builder/pull/21) |
| FR-10 | Редактор должен визуально создавать, изменять, перемещать и удалять элементы | UI landing-editor: create / properties / move / delete без raw JSON | ✅ [landing-editor PRs](https://github.com/rki-mai/landing-editor/pulls?q=is%3Apr+is%3Amerged) |
| FR-11 | Frontend должен сохранять состояние локально и синхронизировать diff с backend | LocalStorage + отправка мутаций при изменениях | ✅ [docs#28](https://github.com/rki-mai/docs/issues/28), [PR #24](https://github.com/rki-mai/landing-editor/pull/24) |
| FR-12 | Система должна создавать публикацию из последнего черновика асинхронно | `POST …/publications` → `201`, `status: PENDING`; worker → `FINISHED` | ✅ [PR #27](https://github.com/rki-mai/wb-landing-builder/pull/27) |
| FR-13 | Система должна отдавать метаданные и список публикаций проекта | `GET …/publications`, `GET …/publications/{id}`; удаление → `204` | ✅ |
| FR-14 | Конечный пользователь должен просматривать опубликованный лендинг без авторизации | `GET /publications/{id}/index.html` через CDN → HTML `200` | ✅ / 🔄 CDN [PR #46](https://github.com/rki-mai/wb-landing-builder/pull/46) |
| FR-15 | Система должна документировать HTTP API | Swagger UI `/swagger/index.html`, OpenAPI `swagger.yaml` | ✅ [PR #17](https://github.com/rki-mai/wb-landing-builder/pull/17) |
| FR-16 | Черновик должен поддерживать базовые типы элементов MVP | `text`, `container`, `link`, `image`, `button` | ✅ [ADR формат черновика](./ADRs/Формат%20Изменений%20Черновика.md) |
| FR-17 | Публикация должна генерировать HTML из JSON черновика | Worker вызывает CLI `generate.py`; артефакт в MinIO | ✅ [landing-builder-cli#1](https://github.com/rki-mai/landing-builder-cli/pull/1) |

### Желательные (Should Have)

| ID | Требование | Критерий приёмки | Статус |
| --- | --- | --- | --- |
| FR-20 | Редактор должен иметь страницы регистрации и входа | UI login/register, JWT в клиенте | 🔄 [landing-editor#27](https://github.com/rki-mai/landing-editor/issues/27), [#28](https://github.com/rki-mai/landing-editor/issues/28) |
| FR-21 | Frontend должен автоматически обновлять access token | Refresh до истечения 15 мин без разлогина | 📋 [landing-editor#31](https://github.com/rki-mai/landing-editor/issues/31) |
| FR-22 | Пользователь должен управлять списком проектов (лендингов) | CRUD проектов, имена, список по user | 📋 [docs#32](https://github.com/rki-mai/docs/issues/32), [wb-landing-builder#29](https://github.com/rki-mai/wb-landing-builder/issues/29) |
| FR-23 | Редактор должен показывать preview до публикации | Preview-режим в UI без POST publications | 📋 ADR Frontend Client |
| FR-24 | Полная сборка лендинга (CSS, JS, медиа в bundle) | Publication содержит не только `index.html`, но полный bundle | 📋 [docs#19](https://github.com/rki-mai/docs/issues/19) |
| FR-25 | OpenAPI через huma + Stoplight Elements | Замена swaggo, авторизация в UI | 🔄 [PR #20](https://github.com/rki-mai/wb-landing-builder/pull/20) |
| FR-26 | Сквозная проверка системы одной командой | `make test-smoke` проходит auth → storage → publish → CDN | ✅ |
| FR-27 | AI code review по запросу в PR | GitHub Action `/review` | ✅ [PR #36](https://github.com/rki-mai/wb-landing-builder/pull/36) |

### Не в этом релизе (Out of Scope)

- **Production-деплой** в инфраструктуру WB — нет SLA, k8s, Grafana
- **Полный набор компонентов из ADR** (Callout, заголовки H1–H3 как отдельные виджеты) — частично покрыто `text`/`container`
- **Шаблоны лендингов (Templates)** — ADR Frontend, не реализовано
- **Откат черновика через UI** — версии в API есть, UI отката нет
- **Интеграция с внешними WB-сервисами** (каталог, заказы, аналитика)
- **Мультиязычность** интерфейса и лендингов
- **WYSIWYG-редактор текста** (bold/italic в callout) — явно out of scope в ADR компонентов
- **Публичный self-service регистрация мерчантов WB** — учебный MVP, локальные пользователи

---

## Нефункциональные требования (NFR)

> Для MVP зафиксированы **измеримые** цели локальной разработки. Production-метрики — целевые на будущее.

### Производительность

| Метрика | Требование (MVP local) | Как измеряем | Статус |
| --- | --- | --- | --- |
| Latency API (p99) | ≤ 500 ms на мутацию и GET snapshot | `curl -w '%{time_total}'`, smoke-тест | 📋 нет formal load test |
| Latency CDN (p99) | ≤ 200 ms для `index.html` (cache HIT) | `curl` + заголовок `X-Cache-Status` | 🔄 |
| Rate limit мутаций | ≤ 100 запросов / project / мин | `429` + `X-RateLimit-*`, env `RATE_LIMIT` | ✅ |
| Размер тела мутации | ≤ 1 MB | `413 Payload Too Large` | ✅ |
| Время публикации (E2E) | ≤ 60 s от POST до `FINISHED` (smoke) | `scripts/smoke.sh` | ✅ |

### Надёжность

| Метрика | Требование (MVP) | Статус |
| --- | --- | --- |
| Доступность local stack | Compose healthchecks: mongo, minio, rabbitmq, API, CDN — `healthy` | ✅ |
| Семантика HTTP-ошибок | Не все ошибки → 500; 400/401/403/404/409/429 где применимо | ✅ [PR #24](https://github.com/rki-mai/wb-landing-builder/pull/24) |
| Повторная доставка задач публикации | Failed worker → статус `FAILED` + `error_message`; сообщение в RabbitMQ nack/requeue | ✅ |
| RPO (локально) | Допустима потеря данных при `docker compose down -v` | — |
| MTTR (локально) | Восстановление `make restart` ≤ 5 мин | ✅ Runbook |

**Целевые production-метрики (не в MVP):** availability ≥ 99.9%, MTTR ≤ 30 min, RPO = 0 для черновиков.

### Масштабируемость

- Backend упакован в Docker; зависимости (MongoDB, MinIO, RabbitMQ) — отдельные контainers
- Публикация **асинхронная** (очередь `publish.requests`) — не блокирует HTTP-handler ([PR #27](https://github.com/rki-mai/wb-landing-builder/pull/27))
- Горизонтальное масштабирование API/worker — **не реализовано** (один процесс worker in-process)
- Объём: MVP рассчитан на десятки проектов и пользователей команды, не на production-нагрузку WB

### Безопасность

| Требование | Реализация | Статус |
| --- | --- | --- |
| Аутентификация API | JWT Bearer, access 15 min, refresh 7 d | ✅ |
| Изоляция черновиков | `owner_id` на уровне service | ✅ |
| Публичный доступ только к статике | `/publications/*` без JWT; `/api/*` с JWT | ✅ |
| Секреты не в репозитории | `JWT_SECRET`, credentials через env / compose | ✅ (dev defaults) |
| HTTPS | Локально HTTP; prod — TLS 1.2+ | 📋 prod |
| ПДн | Email пользователя в MongoDB; политика хранения — не формализована | 📋 |

### Совместимость

| Требование | Детали | Статус |
| --- | --- | --- |
| API version prefix | `/api/v1` | ✅ |
| Contract-First | OpenAPI (Swagger 2.0); формат мутаций в ADR + `schema.json` | ✅ |
| Frontend | React + TypeScript, Vite; Chrome/Firefox актуальных версий | ✅ |
| Backend | Go 1.21+, Linux container | ✅ |
| Обратная совместимость API | Breaking changes только с новой major-версией prefix | 📋 политика не формализована |

---

## Ограничения

| Ограничение | Описание |
| --- | --- |
| **Стек backend** | Go, Gin, MongoDB, RabbitMQ, MinIO/S3 — зафиксирован реализацией |
| **Стек frontend** | React, TypeScript, Vite — [`landing-editor`](https://github.com/rki-mai/landing-editor) |
| **Архитектура** | Монолит с логическими компонентами (решение команды; ADR микросервисов частично устарел — [docs#12](https://github.com/rki-mai/docs/issues/12)) |
| **Команда** | Студенческая команда РКИ МАИ, 4 репозитория |
| **Срок** | MVP 1.0 — учебный семестр 2026 |
| **Инфраструктура** | Только docker-compose локально; нет k8s/Helm |
| **Рендер** | Python CLI внутри образа; полная сборка CSS/JS — отложена |
| **project_id** | Задаётся клиентом; централизованного API «создать проект» пока нет |

---

## Сценарии использования (Use Cases)

### UC-01: Регистрация и вход редактора

**Актор:** Редактор

**Предусловие:** Backend доступен на `:8080`

**Основной поток:**

1. Редактор регистрируется (`POST /auth/register`) или входит (`POST /auth/login`)
2. Получает JWT
3. Открывает landing-editor, выполняет API-запросы с `Authorization: Bearer …`

**Альтернативный поток:**

1a. Неверный пароль → `401 invalid credentials`  
2a. Email занят → `409 user already exists`

**Постусловие:** Пользователь аутентифицирован, `owner_id` привязан при первой мутации

**Статус:** ✅ API · 🔄 UI auth pages

---

### UC-02: Редактирование черновика лендинга

**Актор:** Редактор

**Предусловие:** JWT valid, выбран `project_id`

**Основной поток:**

1. Редактор добавляет элемент на canvas (create mutation)
2. Меняет свойства (update mutation)
3. Перемещает или удаляет элемент (update/delete)
4. Backend применяет мутацию, увеличивает `version`
5. Frontend сохраняет состояние локально и синхронизирует diff

**Альтернативный поток:**

3a. Rate limit → `429` + `Retry-After`  
4a. Чужой project → `403`

**Постусловие:** Черновик сохранён в MongoDB, версия согласована с клиентом

**Статус:** ✅

---

### UC-03: Публикация лендинга

**Актор:** Редактор

**Предусловие:** Черновик содержит хотя бы один элемент; пользователь — owner

**Основной поток:**

1. Редактор вызывает `POST …/publications`
2. Система создаёт запись `PENDING`, ставит задачу в RabbitMQ
3. Worker рендерит HTML через CLI, загружает в MinIO
4. Статус → `FINISHED`, API возвращает `public_url`
5. Редактор открывает ссылку или передаёт её аудитории

**Альтернативный поток:**

3a. Ошибка рендера/MinIO → `FAILED` + `error_message`  
1a. Нет черновика → `404`

**Постусловие:** Статический HTML доступен по `/publications/{id}/index.html`

**Статус:** ✅

---

### UC-04: Просмотр опубликованного лендинга

**Актор:** Конечный пользователь

**Предусловие:** Публикация в статусе `FINISHED`

**Основной поток:**

1. Пользователь переходит по URL публикации
2. CDN отдаёт `index.html` из MinIO (с кэшем nginx)

**Альтернативный поток:**

2a. Публикация не готова → `404`  
2b. Cache MISS — первый запрос медленнее, далее HIT

**Постусловие:** Пользователь видит лендинг без доступа к API редактирования

**Статус:** ✅ / 🔄 CDN hardening

---

### UC-05: Удаление публикации

**Актор:** Редактор (owner)

**Предусловие:** JWT valid, publication существует

**Основной поток:**

1. `DELETE …/publications/{id}`
2. Система удаляет метаданные и файлы в object storage
3. Ответ `204 No Content`

**Постусловие:** URL публикации больше не отдаёт контент

**Статус:** ✅

---

## Трассировка: требования → артефакты

| Область | Документ / код |
| --- | --- |
| Архитектура | [ADRs/](./ADRs/README.md) |
| Формат черновика | [ADR «Формат изменений»](./ADRs/Формат%20Изменений%20Черновика.md) |
| HTTP API | [API-Reference.md](./API-Reference.md) (PR отдельно) |
| Операции | [Runbook.md](./Runbook.md) (PR отдельно) |
| Решения команды | [Decision-Log.md](./Decision-Log.md) |

---

## Changelog

| Версия | Дата | Изменения |
| --- | --- | --- |
| MVP 1.0 | 2026-06-07 | Первая версия для [docs#2](https://github.com/rki-mai/docs/issues/2): FR/NFR по факту реализации и backlog |
