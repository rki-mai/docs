# Decision Log: РКИ

Это журнал оперативных решений, принятых во время работы над проектом.
Сюда фиксируются любые решения: технические, продуктовые, организационные.

---

| Поле            | Значение   |
| --------------- | ---------- |
| **Проект**      | РКИ        |
| **Команда**     | РКИ МАИ    |
| **Обновлено**   | 2026-06-07 |

---

## Памятка для команды

### Как добавить запись

1. Добавь строку в таблицу ниже
2. Присвой ID в формате `DL-NNN` (DL-001, DL-002, ...)
3. Укажи дату, контекст, решение и кто принял
4. Если решение изменилось — не удаляй старую запись, добавь новую со ссылкой на старую

### Статусы решений

- ✅ Действует — решение принято и актуально
- 🔄 Пересмотрено — заменено другим решением (ссылка)
- ❌ Отменено — решение отозвано
- ❓ Открытый вопрос — ещё не решено

---

## Журнал решений

| ID     | Дата       | Решение | Контекст / почему | Кто принял | Статус |
| ------ | ---------- | ------- | ----------------- | ---------- | ------ |
| DL-001 | 2026-03-13 | Перенести проект в организацию `rki-mai` | Легче разрабатывать компоненты параллельно; у каждого члена команды равные права в org | @WhatTheMUCK | ✅ |
| DL-002 | 2026-04-02 | Разделить лендинг на публикацию и черновик | Разные жизненные циклы и хранилища; зафиксировано в ADR после обсуждения в [docs#9](https://github.com/rki-mai/docs/pull/9) | @WhatTheMUCK @NamerPRO @Nifacy | ✅ |
| DL-003 | 2026-04-20 | Формат отправки изменений черновика — плоский список мутаций | Нужен контракт между frontend и backend. @NamerPRO зафиксировал в [docs#15](https://github.com/rki-mai/docs/pull/15); @Nifacy одобрил ревью: «можем начинать реализовывать логику черновиков на бэкенде и пилить фронтенд». См. [ADR «Формат изменений черновика»](./ADRs/Формат%20Изменений%20Черновика.md) | @NamerPRO @Nifacy | ✅ |
| DL-004 | 2026-04-10 | Перейти с микросервисной схемы на монолитную | После встречи команда сошлась, что для учебного проекта монолит практичнее микросервисов. Задача на обновление ADR: [docs#12](https://github.com/rki-mai/docs/issues/12). При реализации «сервис» в документации читается как «компонент» одного приложения ([wb-landing-builder#28](https://github.com/rki-mai/wb-landing-builder/issues/28#issuecomment-4529502157), @Nifacy) | @Nifacy | ✅ |
| DL-005 | 2026-03-21 | Публичный трафик к опубликованным лендингам идёт через CDN, минуя backend | @NamerPRO в ревью [docs#9](https://github.com/rki-mai/docs/pull/9#issuecomment-4104069050): «CDN перед S3 критичен для производительности — публичный трафик должен идти через CDN». Подтверждено диаграммами в том же PR | @NamerPRO @WhatTheMUCK | ✅ |
| DL-006 | 2026-04-01 | Бизнес-права (авторизация) — зона Auth Service, Gateway остаётся «тупым» фильтром | В [docs#9](https://github.com/rki-mai/docs/pull/9#issuecomment-4170865855) @NamerPRO: Auth Service — единый источник истины для аутентификации и авторизации; Gateway не должен знать про `manage_draft:foo`. @Nifacy согласился: «Да, мне нравится. Перенесу тогда туда» | @NamerPRO @Nifacy | ✅ |
| DL-007 | 2026-05-10 | Storage-компонент (черновики) — часть монолита `wb-landing-builder` | Прототип draft-service в [wb-landing-builder#1](https://github.com/rki-mai/wb-landing-builder/pull/1). @WhatTheMUCK: «концептуальных претензий нет»; @NamerPRO смержил после запроса @Nifacy (нужен для интеграции редактора). Принимает мутации по формату из DL-003 | @NamerPRO @WhatTheMUCK @Nifacy | ✅ |
| DL-008 | 2026-05-11 | Auth-компонент интегрирован в тот же backend, защищает роуты storage | [wb-landing-builder#4](https://github.com/rki-mai/wb-landing-builder/pull/4): структура auth/handler, middleware, repository, service; единый конфиг в корне (отдельная задача [#5](https://github.com/rki-mai/wb-landing-builder/issues/5) закрыта в [#16](https://github.com/rki-mai/wb-landing-builder/pull/16)) | @mregor787 @Nifacy | ✅ |
| DL-009 | 2026-04-28 | HTML-рендер черновика вынести в отдельный репозиторий `landing-builder-cli` | В [wb-landing-builder#28](https://github.com/rki-mai/wb-landing-builder/issues/28#issuecomment-4529502171) @Nifacy предложил снизить порог входа в публикацию; @WhatTheMUCK: «отличная идея». @Nifacy в [landing-builder-cli#2](https://github.com/rki-mai/landing-builder-cli/issues/2#issuecomment-4522802415): отдельный репо упростит миграцию при смене реализации ([docs#19](https://github.com/rki-mai/docs/issues/19)). Смержено: [landing-builder-cli#1](https://github.com/rki-mai/landing-builder-cli/pull/1) (2026-05-24) | @Nifacy @WhatTheMUCK | ✅ |
| DL-010 | 2026-04-20 | Первая версия публикации — заглушка: bundle из `index.json` + `index.html` (CLI), не полная сборка CSS/JS | @Nifacy в [wb-landing-builder#28](https://github.com/rki-mai/wb-landing-builder/issues/28#issuecomment-4529502146): «реальную сборку статики — в другой задаче»; интерфейс `BlobStorage` должен поддерживать несколько файлов ([#28](https://github.com/rki-mai/wb-landing-builder/issues/28#issuecomment-4529502164)). Реализовано в [wb-landing-builder#27](https://github.com/rki-mai/wb-landing-builder/pull/27) | @Nifacy @WhatTheMUCK | ✅ |
| DL-011 | 2026-06-01 | Публикация черновиков — асинхронный pipeline: HTTP → PENDING → RabbitMQ → worker → MinIO → FINISHED | [wb-landing-builder#27](https://github.com/rki-mai/wb-landing-builder/pull/27) по [wb-landing-builder#28](https://github.com/rki-mai/wb-landing-builder/issues/28). Согласовано с sequence diagram из [docs#9](https://github.com/rki-mai/docs/pull/9); in-process вызовы storage вместо gRPC между сервисами (монолит). @Nifacy запрашивал правки по sync→async; финальный коммит `ae8cb81` | @WhatTheMUCK @Nifacy | ✅ |
| DL-012 | 2026-05-22 | Доступ к черновику только владельцу (`owner_id`) | [wb-landing-builder#21](https://github.com/rki-mai/wb-landing-builder/pull/21) по [wb-landing-builder#6](https://github.com/rki-mai/wb-landing-builder/issues/6). Сравнение `userID` из JWT с `ownerID` в service-слое; 403 при несовпадении | @mregor787 | ✅ |
| DL-013 | 2026-05-14 | OpenAPI-документация через Swagger UI на backend | [wb-landing-builder#17](https://github.com/rki-mai/wb-landing-builder/pull/17): @NamerPRO подключил swagger; @Nifacy в ревью [#1](https://github.com/rki-mai/wb-landing-builder/pull/1#issuecomment-4320404910) просил схему для фронтенда. Доступ: `http://localhost:8080/swagger/index.html` | @NamerPRO | ✅ |
| DL-014 | 2026-05-19 | Заменить swaggo на huma для API-документации | @NamerPRO в [wb-landing-builder#20](https://github.com/rki-mai/wb-landing-builder/pull/20): «swaggo не очень хорошо справляется»; UI — Stoplight Elements. PR открыт, на доске [Review](https://github.com/rki-mai/wb-landing-builder/issues/19) | @NamerPRO | ❓ |
| DL-015 | 2026-06-01 | AI code review по команде `/review` в PR | [wb-landing-builder#36](https://github.com/rki-mai/wb-landing-builder/pull/36) по [wb-landing-builder#35](https://github.com/rki-mai/wb-landing-builder/issues/35). @WhatTheMUCK после настройки: «работает, но плохо — используйте на свой страх и риск» | @WhatTheMUCK | ✅ |
| DL-016 | 2026-06-06 | Просмотр опубликованных лендингов — через локальный CDN (nginx на `:8080`), не через backend-прокси | [wb-landing-builder#46](https://github.com/rki-mai/wb-landing-builder/pull/46) по [wb-landing-builder#45](https://github.com/rki-mai/wb-landing-builder/issues/45). Публичный URL: `http://localhost:8080/publications/{id}/index.html`; кэш `X-Cache-Status: MISS/HIT`. PR открыт, статус доски Review | @WhatTheMUCK | ❓ |
| DL-017 | 2026-04-07 | Детальный JSON-контракт редактора — отдельно от верхнеуровневого ADR архитектуры | Спор в [docs#9](https://github.com/rki-mai/docs/pull/9): @NamerPRO настаивал на фиксации API-контракта в том же PR; @Nifacy предложил сначала смержить ADR, формат — в отдельном PR. Итог: [docs#15](https://github.com/rki-mai/docs/pull/15) → DL-003 | @Nifacy @NamerPRO | ✅ |

---

## Открытые вопросы

| ID  | Вопрос | Кто должен решить | Дедлайн |
| --- | ------ | ----------------- | ------- |
| OQ-001 | Обновить ADR архитектуры под монолит и описать причины выбора ([docs#12](https://github.com/rki-mai/docs/issues/12)) | команда | — |
| OQ-002 | Смержить переход swaggo → huma ([wb-landing-builder#20](https://github.com/rki-mai/wb-landing-builder/pull/20), DL-014) | @NamerPRO | — |
| OQ-003 | Смержить CDN для просмотра публикаций ([wb-landing-builder#46](https://github.com/rki-mai/wb-landing-builder/pull/46), DL-016) | @WhatTheMUCK | — |
| OQ-004 | Интеграция фронтенда в backend как единое приложение ([docs#31](https://github.com/rki-mai/docs/issues/31)) | команда | — |
| OQ-005 | Управление проектами пользователя: модель Project, список проектов, UI ([docs#32](https://github.com/rki-mai/docs/issues/32), [wb-landing-builder#44](https://github.com/rki-mai/wb-landing-builder/pull/44)) | @mregor787 @Nifacy | — |
