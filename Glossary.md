# Глоссарий: РКИ (конструктор лендингов)

> Словарь терминов проекта «Конструктор лендингов» (организация [rki-mai](https://github.com/rki-mai)).
> Помогает команде говорить на одном языке при работе с документацией, кодом и задачами на [Task tracker](https://github.com/orgs/rki-mai/projects/1/views/3).

---

| Поле | Значение |
| --- | --- |
| **Проект** | РКИ — конструктор лендингов для Wildberries |
| **Команда** | РКИ МАИ |
| **Обновлено** | 2026-06-07 |

---

## Бизнес-термины

| Термин | Определение | Синонимы / Примечания |
| --- | --- | --- |
| Лендинг | Одностраничный сайт, который создаёт и редактирует пользователь в конструкторе | Landing page; в коде часто привязан к `project_id` |
| Черновик | Рабочая версия лендинга в процессе редактирования; хранится как структурированные данные (JSON) и история изменений | Draft; см. [ADR «Разделение лендинга»](./ADRs/Разделение%20лендинга.md) |
| Публикация | Неизменяемый снимок черновика, готовый для просмотра целевой аудиторией; представлен статическими файлами (HTML/CSS/JS) | Publish, Publication; поля `id`, `assets_path`, статусы `PENDING` / `FINISHED` |
| Компонент лендинга | Атомарный или составной UI-элемент страницы (заголовок, кнопка, изображение и т.д.), который пользователь добавляет и настраивает | Widget, Block; минимальный набор — в [ADR «Компоненты лендинга»](./ADRs/Компоненты%20лендинга.md) |
| Редактор | Пользователь системы, который создаёт и изменяет лендинги через визуальный интерфейс | Editor; репозиторий [`landing-editor`](https://github.com/rki-mai/landing-editor) |
| Конечный пользователь | Посетитель опубликованного лендинга; получает статику через CDN, без доступа к API редактирования | End user, EndUsers — на диаграммах в [ADR архитектуры](./ADRs/Уточнения%20и%20детализация%20архитектуры.md) |
| Проект (`project_id`) | Логическая единица, к которой привязан черновик и публикации одного лендинга; идентификатор в API storage | В разработке: модель Project, создание и список проектов ([wb-landing-builder#29](https://github.com/rki-mai/wb-landing-builder/issues/29)) |
| Мутация | Одно атомарное изменение черновика (создание, обновление, удаление, перемещение элемента), отправляемое на backend | Операции `create` / `update` / `delete`; см. [ADR «Формат изменений черновика»](./ADRs/Формат%20Изменений%20Черновика.md) |
| Версия черновика | Порядковый номер состояния черновика после применения мутаций; позволяет синхронизировать frontend и backend | Поле `version` в ответе API; см. DL-012 в [Decision Log](./Decision-Log.md) |
| CMS | Система управления контентом — класс ПО для создания и редактирования веб-контента без глубокого программирования | Наш продукт позиционируется как CMS для лендингов ([ADR архитектуры §1.3](./ADRs/Уточнения%20и%20детализация%20архитектуры.md)) |
| Сборка лендинга | Преобразование данных черновика в готовые статические файлы для публикации | Полная сборка CSS/JS/медиа — отдельная задача ([docs#19](https://github.com/rki-mai/docs/issues/19)); сейчас заглушка: `index.json` + `index.html` (DL-010) |

---

## Технические термины

| Термин | Определение |
| --- | --- |
| ADR | Architecture Decision Record — документ с архитектурным решением, контекстом и альтернативами; каталог: [ADRs/](./ADRs/README.md) |
| Decision Log (DL) | Журнал оперативных решений команды (технических, продуктовых, организационных); см. [Decision-Log.md](./Decision-Log.md) |
| Монолит | Текущая целевая архитектура: один backend-процесс `wb-landing-builder` с несколькими **компонентами** внутри (auth, storage, publishing). Решение DL-004 |
| Компонент | Логически выделенная часть монолита с собственной зоной ответственности (handler / service / repository). В старых ADR может называться «сервис» |
| `wb-landing-builder` | Основной backend-репозиторий: Go, Gin, MongoDB, RabbitMQ, MinIO; хранит черновики и публикует лендинги |
| Storage-компонент | Хранение и изменение черновиков: приём мутаций, выдача snapshot по версии, контроль `owner_id`. Бывш. draft-service ([wb-landing-builder#1](https://github.com/rki-mai/wb-landing-builder/pull/1)) |
| Auth-компонент | Регистрация, аутентификация, JWT; единый источник истины для авторизации (DL-006). Репозиторий: `auth/` внутри монолита |
| Publishing-компонент | Асинхронная публикация: POST → `PENDING` → RabbitMQ → worker → MinIO → `FINISHED` (DL-011). Код: `publishing/` |
| `landing-editor` | Frontend SPA (React, TypeScript) — визуальный редактор лендинга; интегрирован с backend ([landing-editor#24](https://github.com/rki-mai/landing-editor/pull/24)) |
| `landing-builder-cli` | Отдельная Python CLI-утилита: JSON черновика → `index.html`. Репозиторий [`landing-builder-cli`](https://github.com/rki-mai/landing-builder-cli) (DL-009) |
| API Gateway | Единая точка входа для HTTP-запросов клиента; JWT, rate limiting. Локально — nginx на `:8080`; не проверяет бизнес-права (DL-006) |
| CDN | Слой доставки статики опубликованных лендингов; публичный трафик идёт через CDN → S3, минуя backend (DL-005). Локально — nginx с `proxy_cache` |
| S3 / MinIO | Object storage для артефактов публикации; bucket `publications`, ключ `publications/{id}/index.html` |
| RabbitMQ | Очередь сообщений для асинхронной обработки задач публикации между HTTP-handler и worker |
| Bundle | Набор файлов, формируемый при публикации; интерфейс `BlobStorage` рассчитан на несколько файлов в bundle |
| `owner_id` | Идентификатор пользователя-владельца черновика; проверка доступа в service-слое, иначе HTTP 403 (DL-012) |
| JWT | JSON Web Token — токен аутентификации, передаётся клиентом в заголовках запросов к API |
| OpenAPI / Swagger | Машиночитаемое описание HTTP API; UI на backend (`/swagger/index.html`, DL-013). Переход на huma — в review (DL-014) |
| Contract-First | Подход: сначала фиксируется контракт API/данных, затем реализация; см. [ADR §1.1](./ADRs/Уточнения%20и%20детализация%20архитектуры.md) |
| Элемент (`lb-N`) | Узел дерева лендинга в JSON; id в формате `lb-<номер>`, поля `element`, `styles`, `children`, `parentId` |
| `root` | Специальный `parentId` для элементов верхнего уровня лендинга |
| Smoke-тест | Shell-скрипт `scripts/smoke.sh` — сквозная проверка auth, storage, publishing в docker-compose |
| Нейроревью | AI code review по команде `/review` в PR; GitHub Action в `wb-landing-builder` (DL-015) |

---

## Акронимы и сокращения

| Акроним | Расшифровка |
| --- | --- |
| РКИ | Разработка компонентов информационных систем (курс / команда МАИ) |
| WB | Wildberries |
| ADR | Architecture Decision Record |
| DL | Decision Log — журнал решений |
| CMS | Content Management System |
| API | Application Programming Interface |
| SPA | Single Page Application |
| JWT | JSON Web Token |
| CDN | Content Delivery Network |
| S3 | Simple Storage Service (объектное хранилище; локально — MinIO) |
| CLI | Command Line Interface |
| PR | Pull Request |
| MVP | Minimum Viable Product |
| UI | User Interface |
| HTTP | HyperText Transfer Protocol |
| REST | Representational State Transfer |
| gRPC | gRPC Remote Procedure Calls (в ADR — для связи сервисов; в монолите заменено in-process вызовами) |
| CI/CD | Continuous Integration / Continuous Deployment |
| WIP | Work In Progress — черновики ADR в каталоге `(WIP)` |

---

## Противоречия и уточнения

> Раздел фиксирует термины, которые в разных документах или этапах проекта назывались по-разному.

| Термин 1 | Термин 2 | Как правильно сейчас | Контекст |
| --- | --- | --- | --- |
| Сервис (Service) | Компонент | **Компонент** — в коде и Decision Log после перехода на монолит (DL-004) | ADR «Уточнения архитектуры» ещё описывает микросервисы; при чтении «Draft Service» = `storage/` в монолите ([комментарий к #28](https://github.com/rki-mai/wb-landing-builder/issues/28#issuecomment-4529502157)) |
| Landing Service | Draft Service | **Draft Service / storage-компонент** — работа с черновиками | @NamerPRO в [docs#9](https://github.com/rki-mai/docs/pull/9#issuecomment-4104069050): «лучше назвать Draft Service» |
| Publish Service | Publishing Service | **Publishing-компонент** (`publishing/`) | В ADR и коде встречаются оба; сущность — [ADR «Сервис публикации»](./ADRs/Сервис%20публикации%20лендингов.md) |
| `published` (bool) | `publicationId` | **`publicationId`** — ссылка на опубликованную версию | Старый ADR публикации; в реализации — отдельная сущность Publication с `id` и `assets_path` |
| swaggo / Swagger UI | huma / Stoplight Elements | **huma** — целевое решение (DL-014, PR открыт) | Пока в main — Swagger ([wb-landing-builder#17](https://github.com/rki-mai/wb-landing-builder/pull/17)) |
| Backend-прокси статики | CDN (nginx) | **CDN на `:8080`** для `GET /publications/{id}/…` (DL-016) | Публичная раздача не через `/api/` и не через Go-handler |
| Черновик (draft) | Лендинг (landing) | **Черновик** — редактируемое состояние; **публикация** — снимок для просмотра | См. [ADR «Разделение лендинга»](./ADRs/Разделение%20лендинга.md); не смешивать в одной сущности |
| Микросервисная архитектура | Монолитная архитектура | **Монолит** — принятое решение (DL-004); ADR архитектуры — частично устарел | Задача на переписывание: [docs#12](https://github.com/rki-mai/docs/issues/12) |

---

## Связанные документы

- [Decision Log](./Decision-Log.md) — журнал решений (DL-001…)
- [ADRs](./ADRs/README.md) — архитектурные решения
