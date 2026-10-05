# Library API — домашнее задание №3
OpenAPI 3.1 контракт (contract-first) для книг, читателей и выдач.

## Структура

| Файл | Назначение |
|---|---|
| `openapi.yaml` | Сам контракт |
| `STYLE.md` | 10 правил проектирования API |
| `REVIEW.md` | Саморевью по чек-листу со слайдов 32–35 |
| `README.md` | Этот файл |

## Ресурсы

| Ресурс | Операции |
|---|---|
| Books | `GET/POST /books`, `GET/PATCH/DELETE /books/{id}` |
| Readers | `GET/POST /readers`, `GET/PATCH/DELETE /readers/{id}` |
| Loans | `GET/POST /loans`, `GET /loans/{id}`, `POST /loans/{id}/return` |

Базовый URL: `https://api.library.example.com/v1`.

## Бизнес-правила и где они в контракте

| Правило | Ответ |
|---|---|
| Выдача только при `available_copies > 0` | `409 BUSINESS_RULE_VIOLATION`, `issue: no_available_copies` |
| Не более 5 активных выдач на читателя | `409 BUSINESS_RULE_VIOLATION`, `issue: active_loans_limit` |
| Возврат — `POST /loans/{id}/return`, не `DELETE` | `200` + `Loan`; повторный возврат — `409 CONFLICT` |
| Просроченные выдачи | `GET /loans?status=overdue` |
| Нельзя удалить книгу или читателя с активными выдачами | `409 BUSINESS_RULE_VIOLATION`, `issue: has_active_loans` |
| `due_date` строго позже текущей даты | `422 VALIDATION_ERROR` |
| `available_copies` считает сервер | `readOnly`, в запросах нет, лишние поля — `422` |

## Idempotency-Key на `POST /loans`

| Сценарий | Результат |
|---|---|
| Новый ключ | обычный `201` + `Location`, ответ сохраняется |
| Тот же ключ, то же тело | полный replay: тот же статус, заголовки и тело, вторая выдача не создаётся |
| Тот же ключ, другое тело | `409 IDEMPOTENCY_MISMATCH` |
| Повтор, пока первый запрос выполняется | `409 CONFLICT` + `Retry-After`; после повтора — replay |
| Ключ старше 24 часов | считается новым |

## Проверка контракта

```bash
# линтер (нужен Node.js)
npx @redocly/cli lint openapi.yaml

# предпросмотр документации
npx @redocly/cli preview-docs openapi.yaml
```

Либо вставьте `openapi.yaml` в Swagger Editor (в РФ он заблокирован — локально или через VPN) или откройте файл в VS Code с расширением OpenAPI.

## Mock-сервер и примеры curl (бонус)

```bash
npx @stoplight/prism-cli mock -p 4010 openapi.yaml
```

Prism отдаёт ответы из `examples`. Нужный код и пример выбираются заголовком `Prefer`.

```bash
# список книг
curl -s http://localhost:4010/v1/books \
  -H "Authorization: Bearer test-token"

# создание выдачи
curl -i -X POST http://localhost:4010/v1/loans \
  -H "Authorization: Bearer test-token" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 3f6c1d5e-8a2b-4c7e-9f10-2b6d4e8a1c33" \
  -d '{"book_id": 1, "reader_id": 10, "due_date": "2026-10-19"}'

# ошибка лимита выдач (пример 409)
curl -i -X POST http://localhost:4010/v1/loans \
  -H "Authorization: Bearer test-token" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 3f6c1d5e-8a2b-4c7e-9f10-2b6d4e8a1c33" \
  -H "Prefer: code=409, example=loans_limit" \
  -d '{"book_id": 1, "reader_id": 10, "due_date": "2026-10-19"}'

# возврат книги
curl -i -X POST http://localhost:4010/v1/loans/42/return \
  -H "Authorization: Bearer test-token"
```

Mock возвращает статические примеры: он не считает `available_copies`, не хранит ключи и не проверяет лимиты. Бизнес-логику реализует настоящий backend.

## Авторизация (бонус)

`securitySchemes.bearerAuth` — `http` / `bearer` / `JWT`, применён глобально. Отсутствие или недействительность токена — `401 UNAUTHORIZED`.

## Переход к v2: что совместимо, что нет (бонус)

| Изменение | Тип | Почему |
|---|---|---|
| Новое необязательное поле в response (например, `Loan.status`) | совместимое | старый клиент его игнорирует |
| Новый необязательный query-параметр (например, `q`) | совместимое | старые запросы работают |
| Новый endpoint (`GET /books/{id}/loans`) | совместимое | существующее не меняется |
| Новое значение в `enum` ответа | осторожно | старый клиент может не знать значение |
| Новый обязательный параметр или `required`-поле в request | **несовместимое** | старый клиент начнёт получать `400`/`422` |
| Удаление или переименование поля (`full_name` → `name`) | **несовместимое** | ломается разбор ответа |
| Смена типа поля (`published_year`: integer → string) | **несовместимое** | ломается разбор ответа |
| Смена формата ошибки или смысла `code` | **несовместимое** | клиенты ветвятся по `error.code` |
| Смена кода ответа (`200` → `204` на возврат) | **несовместимое** | меняется контракт обработки |
| Переход с `offset/limit` на cursor-пагинацию | **несовместимое** | другие параметры и поля `pagination` |
| Ужесточение валидации (`title` ≤ 100 символов) | **несовместимое** | ранее валидные данные начнут отвергаться |

Устаревающее поле сначала помечают `deprecated: true` с датой удаления в `description`, а удаляют только в `/v2`.
