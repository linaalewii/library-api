# REVIEW.md — саморевью по чек-листу (слайды 32–35)

Статусы: ✅ выполнено · ⚠️ выполнено с оговоркой · ❌ не выполнено.

Проверка структуры: YAML парсится, все `$ref` разрешаются, `operationId` уникальны, у всех 14 операций есть `tags`, `summary` и `description`, имена путей в kebab-case, свойства схем в snake_case, все примеры соответствуют своим схемам.
Результат `npx @redocly/cli lint openapi.yaml`: ____ (впишите после запуска: «0 errors» и число warnings).

## Слайд 32. Ресурсы и URL

| Пункт | Статус | Где / комментарий |
|---|---|---|
| Path — существительные, множественное число, kebab-case | ✅ | `/books`, `/readers`, `/loans` |
| Нет глагольных URL, кроме осознанных действий | ✅ | единственное действие — `POST /loans/{id}/return`, обоснование в `description` и в STYLE.md п. 2 |
| `id` ресурса — в path, фильтры — в query | ✅ | `/loans/{id}`; `status`, `reader_id`, `book_id` — query |
| Версия API указана в `servers` | ✅ | `https://api.library.example.com/v1` |

## Слайд 33. Методы, коды, схемы

| Пункт | Статус | Где / комментарий |
|---|---|---|
| POST создания возвращает `201` + `Location` | ✅ | `createBook`, `createReader`, `createLoan` |
| DELETE возвращает `204` без тела | ✅ | `deleteBook`, `deleteReader` |
| Пустой список — `200` + пустой `data`, не `404` | ✅ | написано в `description` списков |
| Бизнес-конфликт — `409`, ошибка валидации — `422` | ✅ | нет копий, лимит 5 выдач, удаление с выдачами — `409`; `due_date` в прошлом, плохой email, неизвестное поле — `422` |
| У каждого request body отдельная схема | ✅ | `Create*Request`, `Update*Request` |
| У каждого успешного response есть схема | ✅ | `Book`, `BookList`, `Reader`, `ReaderList`, `Loan`, `LoanList`; `204` без тела по определению |
| `required`, `enum`, `format`, `nullable`, ограничения явно | ✅ | `returned_at: [string, 'null']`, `format: email/date/date-time/uuid`, `minLength`, `minimum` |
| Список и карточка — разные схемы | ⚠️ | у книг есть `BookListItem`; у читателей и выдач список использует полную схему: она и так компактна |
| `available_copies` только в response | ✅ | `readOnly: true`, нет в `CreateBookRequest` и `UpdateBookRequest`, `additionalProperties: false` |

## Слайд 34. Ошибки и списки

| Пункт | Статус | Где / комментарий |
|---|---|---|
| Все ошибки используют один `ErrorResponse` | ✅ | все `4xx/5xx` ссылаются на `#/components/schemas/ErrorResponse` |
| Все коды из требований присутствуют | ✅ | `INVALID_REQUEST`, `NOT_FOUND`, `VALIDATION_ERROR`, `CONFLICT`, `BUSINESS_RULE_VIOLATION`, `IDEMPOTENCY_MISMATCH` (плюс `UNAUTHORIZED`, `INTERNAL_ERROR`) |
| Для 422 есть example с `details` | ✅ | `ValidationEmptyTitle`, `ValidationDueDate` |
| На коллекциях есть фильтры, сортировка, пагинация | ✅ | `/books`, `/readers`, `/loans` |
| `GET /loans?status=overdue` | ✅ | `status: [active, returned, overdue]` |
| `limit` имеет default и maximum | ✅ | default 20, maximum 100 |

## Слайд 35. Документация

| Пункт | Статус | Где / комментарий |
|---|---|---|
| Operations имеют `summary`, `description`, `tags` | ✅ | у всех 14 операций |
| Есть examples успеха и типичной ошибки | ✅ | 31 пример в `components/examples`: успех, `400`, `404`, `422`, шесть видов `409` |
| Общие схемы через `$ref`, не копируются руками | ✅ | поля вынесены в `Id`, `BookTitle`, `Isbn`, `Email` и др.; `ErrorResponse`, параметры и общие ответы подключены через `$ref` |
| Файл OpenAPI валидируется без ошибок | ⚠️ | структурные проверки пройдены; результат `redocly lint` указан в начале файла |

## Требования ДЗ (сверка)

| Требование | Статус |
|---|---|
| OpenAPI 3.1, `/v1/` в `servers` | ✅ |
| CRUD Books и Readers | ✅ |
| Loans: создание, список, получение, возврат | ✅ |
| `Idempotency-Key` на `POST /loans`: replay, mismatch, конкурентные повторы, TTL 24 часа | ✅ (описано в `description` операции и параметра) |
| Бизнес-правила 1–7 | ✅ (в `info.description` и в операциях) |
| kebab-case в URL, snake_case в JSON | ✅ |
| tags Books / Readers / Loans | ✅ |
| Минимум 3 examples | ✅ |
| Бонус: Bearer JWT (`securitySchemes`) | ✅ |
| Бонус: mock + curl | ✅ (команды в README; в среде подготовки не запускались) |
| Бонус: таблица breaking / non-breaking для v2 | ✅ (README) |

## Известные компромиссы

- Конкурентный повтор отвечает `409 CONFLICT` + `Retry-After`: это явное поведение, которое проще описать в контракте, чем ожидание на сервере.
- Несуществующий `book_id` или `reader_id` в теле `POST /loans` — `422`, потому что идентификаторы переданы в теле, а не в path.
- Повторный возврат уже закрытой выдачи — `409 CONFLICT`.
- Авторизация описана минимально (схема и `401`); роли и `403` не вводились, они вне рамок урока.