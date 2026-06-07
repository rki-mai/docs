# ADR — Architecture Decision Records

Каталог архитектурных решений проекта «Конструктор лендингов» ([rki-mai](https://github.com/rki-mai)).

> Формат записей ADR-0001…0004 — по [шаблону преподавателя](https://github.com/user-attachments/files/25998244/ADR-template.md) ([docs#1](https://github.com/rki-mai/docs/issues/1)).  
> Старые документы сохранены как reference; часть помечена **Superseded** новыми ADR.

**Обновлено:** 2026-06-07

---

## Реестр решений (формат шаблона)

| ID | Название | Статус | Дата | Авторы |
| --- | --- | --- | --- | --- |
| [ADR-0001](./ADR-0001-Монолитная-архитектура-MVP.md) | Монолитная архитектура backend MVP | Approved | 2026-05-11 | @WhatTheMUCK, @NamerPRO, @mregor787 |
| [ADR-0002](./ADR-0002-Разделение-черновика-и-публикации.md) | Разделение черновика и публикации | Approved | 2026-04-02 | @WhatTheMUCK, @NamerPRO, @Nifacy |
| [ADR-0003](./ADR-0003-Формат-мутаций-черновика.md) | Формат мутаций черновика | Approved | 2026-04-20 | @NamerPRO, @Nifacy |
| [ADR-0004](./ADR-0004-Асинхронная-публикация-и-CDN.md) | Асинхронная публикация и CDN | Approved | 2026-06-01 | @WhatTheMUCK |

---

## Детальные документы (legacy / reference)

| Документ | Статус | Связь с ADR |
| --- | --- | --- |
| [Уточнения и детализация архитектуры](./Уточнения%20и%20детализация%20архитектуры.md) | Superseded (частично) | Микросервисная модель → см. **ADR-0001**; переписывание: [docs#12](https://github.com/rki-mai/docs/issues/12) |
| [Разделение лендинга](./Разделение%20лендинга.md) | Superseded | Краткая версия → **ADR-0002** |
| [Формат Изменений Черновика](./Формат%20Изменений%20Черновика.md) | Reference | Полная спецификация → **ADR-0003** |
| [Сервис публикации лендингов](./Сервис%20публикации%20лендингов.md) | Superseded (частично) | Sync/отдельный сервис → **ADR-0004** |
| [Компоненты лендинга](./Компоненты%20лендинга.md) | Approved (reference) | UI-модель компонентов MVP |
| [(WIP) Auth Service](./(WIP)%20Auth%20Service.md) | Draft | Реализовано in-process: `auth/` (**ADR-0001**) |
| [(WIP) Draft Service](./(WIP)%20Draft%20Service.md) | Draft | Реализовано: `storage/` (**ADR-0001**) |
| [(WIP) Frontend Client](./(WIP)%20Frontend%20Client.md) | Draft | [`landing-editor`](https://github.com/rki-mai/landing-editor) |

---

## Хронология (PR → ADR)

| Дата | Событие | ADR |
| --- | --- | --- |
| 2026-03-20 | [docs PR #9](https://github.com/rki-mai/docs/pull/9) — архитектура (микросервисы) | legacy doc |
| 2026-04-02 | Разделение лендинга | ADR-0002 |
| 2026-04-20 | [docs PR #15](https://github.com/rki-mai/docs/pull/15) — формат мутаций | ADR-0003 |
| 2026-05-10 | [wb-landing-builder PR #1](https://github.com/rki-mai/wb-landing-builder/pull/1) — storage | ADR-0001 |
| 2026-05-11 | [PR #4](https://github.com/rki-mai/wb-landing-builder/pull/4) — auth | ADR-0001 |
| 2026-06-01 | [PR #27](https://github.com/rki-mai/wb-landing-builder/pull/27) — publishing | ADR-0004 |

---

## Как добавить ADR

1. Скопировать [шаблон](https://github.com/user-attachments/files/25998244/ADR-template.md)
2. Именовать файл `ADR-NNNN-Краткое-название.md`
3. Присвоить статус `Draft` → `Proposed` → `Approved`
4. Добавить строку в таблицу реестра выше
5. При замене старого решения — указать **Supersedes** / пометить legacy-документ
