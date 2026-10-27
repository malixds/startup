# ADR-0007. Локальная разработка: Docker Compose и переменные окружения

**Статус:** Принято
**Дата:** 2026-10-27
**Связано:** ТЗ п. 3.1, 3.5, 9.2.5, 9.3, 10, 11.2; CLAUDE.md «Команды», правило 7; backlog F-064, F-065, F-068, F-071, Q-07; решения заказчика В-03, В-05, В-07, В-09, В-10

## Контекст

- Сейчас ничего не хостим; нужен локальный запуск через `docker compose up -d` (Q-07). Prod-конфигурация готовится при выборе хостинга.
- **Установка у клиента (on-premise) в R1 не входит** (В-07). Принцип «вся конфигурация через env, секреты только в `.env`» сохраняем, чтобы добавить её позже без переделки (CLAUDE.md, правило 7).
- Команды из CLAUDE.md ориентированы на сервис `app`.

## Решение

### Сервисы `compose.yaml`

| Сервис | Образ | Назначение | Профиль |
| --- | --- | --- | --- |
| `nginx` | nginx stable-alpine | HTTP, статика, прокси в FPM, политика логов: без access-лога для анонимных комментариев, ротация ≤ 30 дней | всегда |
| `app` | свой образ `docker/php/Dockerfile` (PHP 8.5-FPM + pdo_pgsql, redis, intl, pcntl, opcache) | PHP-FPM; здесь же `artisan`, тесты, Pint, PHPStan | всегда |
| `horizon` | тот же образ | `php artisan horizon` — воркеры `mail`, `analytics`, `ai`, `default` | всегда |
| `scheduler` | тот же образ | `php artisan schedule:work` | всегда |
| `postgres` | postgres:18-alpine | БД, том `pgdata`, healthcheck `pg_isready` | всегда |
| `redis` | redis:8-alpine | Очереди, кеш, сессии, счётчики лимитов; AOF, `noeviction`, том `redisdata` (В-10) | всегда |
| `mailpit` | axllent/mailpit | Перехват писем, веб-интерфейс | `dev` |
| `node` | node:24-alpine | `vite` dev-сервер для кабинета и `/survey` | `dev` |

- Один PHP-образ для `app`, `horizon` и `scheduler`, чтобы версии не расходились. В dev код монтируется томом; prod-сборка (многоэтапная, код внутри образа) делается при выборе хостинга.
- Расширение разрабатывается на хосте (`npm --prefix extension run dev`): WXT открывает браузер с расширением, в контейнере это неудобно. `WXT_API_BASE_URL` для dev — `http://localhost`, только для dev-сборки.
- `depends_on` с `condition: service_healthy` для `postgres` и `redis`. `/health` проверяет БД, Redis и наличие живых супервизоров Horizon (п. 9.3.2).
- Тестовая БД — отдельная база в том же `postgres` (`DB_DATABASE_TEST`), создаётся init-скриптом.
- Команды ручной очистки мягко удалённых данных (`privacy:purge-user`, `privacy:purge-project`, ADR-0004) запускаются через `docker compose exec app php artisan ...` и в Scheduler не добавляются (В-05).

### Переменные окружения (`.env.example`, без значений секретов)

| Группа | Переменные |
| --- | --- |
| Приложение | `APP_NAME`, `APP_ENV`, `APP_KEY`, `APP_URL`, `APP_LOCALE=ru`, `APP_FALLBACK_LOCALE=en` |
| БД | `DB_CONNECTION=pgsql`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` |
| Redis, очереди, кеш | `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `QUEUE_CONNECTION=redis`, `CACHE_STORE=redis`, `SESSION_DRIVER=redis` |
| Сессии (В-09) | `SESSION_IDLE_TIMEOUT=1440` (минуты, 24 ч бездействия; передаётся в `SESSION_LIFETIME` Laravel), `SESSION_ABSOLUTE_LIFETIME=20160` (минуты, 14 дней) |
| Sanctum, CORS | `SANCTUM_STATEFUL_DOMAINS`, `SESSION_DOMAIN`, `CORS_ALLOWED_ORIGINS` (ID расширений) |
| Почта | `MAIL_MAILER`, `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM_ADDRESS`, `MAIL_FROM_NAME` (локально — Mailpit) |
| Ключи (В-03) | `LOGIN_CODE_HMAC_KEY` — HMAC кодов входа; `INVITE_CODE_HMAC_KEY` — HMAC кодов приглашения для поиска; `INVITE_CODE_ENCRYPTION_KEY` — шифрование копии кода для показа. Все три независимы от `APP_KEY`, чтобы ротация одного ключа не ломала остальное |
| Доступ | `HORIZON_ADMIN_EMAILS` |
| Лимиты и флаги | `AUTH_CODE_TTL`, `AUTH_CODE_MAX_ATTEMPTS`, `INVITE_RATE_PER_MINUTE`, `ANONYMITY_MIN_RESPONDENTS=3`, `ANONYMOUS_COMMENTS_PER_DAY=1`, `PLAN_PILOT_*`; значения по умолчанию лежат в `config/*.php` |
| Мониторинг | `LOG_CHANNEL`, `LOG_LEVEL`, `ERROR_REPORTER` (`log`; позже `sentry`/`glitchtip`) + DSN |
| Хранилище | `FILESYSTEM_DISK=local` (экспорты; S3-совместимое хранилище — при выборе хостинга) |
| AI (R2) | `AI_PROVIDER=fake`, ключи провайдеров |
| Расширение (сборка) | `WXT_API_BASE_URL` |

### Хостинг и установка у клиента

- **Хостинг заказчика (вне релизов, Q-07):** тот же набор сервисов в prod-override (`compose.prod.yaml`) без `mailpit` и `node`, TLS на Nginx или внешнем балансировщике (HTTPS + HSTS, п. 9.1.1), бэкапы (`pg_dump`, хранение 14 дней), отчёт об ошибках, SMTP.
- **Установка у клиента (on-premise):** вне R1 (В-07). Готовность обеспечивается env-конфигурацией и единым compose. Для неё понадобятся поле «адрес сервера» в расширении (ADR-0003) и проверка лицензии Redis для поставки (ADR-0008, Р-10).

## Рассмотренные альтернативы

1. **Laravel Sail.** Быстрый старт, но заточен под dev: нет связки Nginx + FPM и prod-схемы, а ТЗ требует одинаковую схему для dev и prod. Отклонено.
2. **Supervisor внутри одного контейнера (FPM + Horizon + cron).** Меньше сервисов, но хуже с логами, перезапусками и масштабированием. Отклонено.
3. **Cron на хосте вместо `schedule:work`.** Появляется зависимость от настроек сервера. Отклонено.

## Последствия

- (+) `docker compose up -d` поднимает всё; команды CLAUDE.md (`docker compose exec app ...`) работают без изменений.
- (+) Вся конфигурация в env, поэтому on-premise добавляется позже без изменения кода.
- (−) В dev восемь сервисов; на слабых машинах `node` и `mailpit` можно не поднимать (профиль `dev`).
- (−) Три отдельных ключа (HMAC и шифрование) нужно сгенерировать при установке и нельзя терять: потеря ключа шифрования делает коды приглашения непоказываемыми (лечится перевыпуском), а потеря HMAC-ключа — ненаходимыми.
- (−) Prod-hardening (секреты, TLS, бэкапы, лимиты ресурсов) остаётся долгом до выбора хостинга.
