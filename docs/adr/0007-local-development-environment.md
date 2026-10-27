# ADR-0007. Локальная разработка: Docker Compose и переменные окружения

**Статус:** Принято
**Дата:** 2026-10-27
**Связано:** ТЗ п. 3.1, 3.5, 9.2.5, 9.3, 10, 11.2; CLAUDE.md «Команды», правило 7; backlog F-005, F-064, F-065, F-068, F-071, Q-07; решения заказчика В-03, В-07, В-09, В-10

## Контекст

- Сейчас ничего не хостим. Нужен локальный запуск через `docker compose up -d` (Q-07). Prod-конфигурацию готовим при выборе хостинга.
- **Установка у клиента (on-premise) в R1 не входит** (В-07). При этом сохраняем принцип: вся конфигурация через env, секреты только в `.env` (правило 7).
- Команды из CLAUDE.md рассчитаны на сервис `app`.

## Решение

### Сервисы `compose.yaml`

| Сервис | Образ | Назначение | Профиль |
| --- | --- | --- | --- |
| `nginx` | nginx stable-alpine | HTTP, статика, прокси в FPM; ротация логов — не дольше 30 дней | всегда |
| `app` | свой образ `docker/php/Dockerfile` (PHP 8.5-FPM + pdo_pgsql, redis, intl, pcntl, opcache) | PHP-FPM; там же `artisan`, тесты, Pint, PHPStan | всегда |
| `horizon` | тот же образ | `php artisan horizon` — воркеры `mail`, `analytics`, `ai`, `default` | всегда |
| `scheduler` | тот же образ | `php artisan schedule:work` | всегда |
| `postgres` | postgres:18-alpine | БД, том `pgdata`, healthcheck `pg_isready` | всегда |
| `redis` | redis:8-alpine | Очереди, кеш, сессии, счётчики лимитов; AOF, `noeviction`, том `redisdata` (В-10) | всегда |
| `mailpit` | axllent/mailpit | Перехват писем, веб-интерфейс | `dev` |
| `node` | node:24-alpine | Dev-сервер `vite` для кабинета и `/survey` | `dev` |

- `app`, `horizon` и `scheduler` используют один PHP-образ.
- Расширение разрабатывается на хосте командой `npm --prefix extension run dev`.
- Для `postgres` и `redis` задан `depends_on` с `condition: service_healthy`. `/health` проверяет БД, Redis и Horizon.
- Тестовая БД — отдельная база в том же `postgres` (`DB_DATABASE_TEST`).

### Переменные окружения (`.env.example`, без значений секретов)

| Группа | Переменные |
| --- | --- |
| Приложение | `APP_NAME`, `APP_ENV`, `APP_KEY`, `APP_URL`, `APP_LOCALE=ru`, `APP_FALLBACK_LOCALE=en` |
| БД | `DB_CONNECTION=pgsql`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` |
| Redis, очереди, кеш | `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `QUEUE_CONNECTION=redis`, `CACHE_STORE=redis`, `SESSION_DRIVER=redis` |
| Сессии (В-09) | `SESSION_LIFETIME=20160` (минуты = 14 дней) |
| Sanctum, CORS | `SANCTUM_STATEFUL_DOMAINS`, `SESSION_DOMAIN`, `CORS_ALLOWED_ORIGINS` (ID расширений) |
| Почта | `MAIL_MAILER`, `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM_ADDRESS`, `MAIL_FROM_NAME` (локально — Mailpit) |
| Ключи (В-03) | `LOGIN_CODE_HMAC_KEY` (коды входа), `INVITE_CODE_HMAC_KEY` (поиск кода приглашения), `INVITE_CODE_ENCRYPTION_KEY` (копия кода для показа). Все три независимы от `APP_KEY` |
| Доступ | `HORIZON_ADMIN_EMAILS` |
| Лимиты и флаги | `AUTH_CODE_TTL`, `AUTH_CODE_MAX_ATTEMPTS`, `INVITE_RATE_PER_MINUTE`, `PLAN_PILOT_*`; значения по умолчанию заданы в `config/*.php` |
| Мониторинг | `LOG_CHANNEL`, `LOG_LEVEL`, `ERROR_REPORTER` (`log`; позже `sentry`/`glitchtip`) + DSN |
| Хранилище | `FILESYSTEM_DISK=local` (экспорты) |
| AI (R2) | `AI_PROVIDER=fake`, ключи провайдеров |
| Расширение (сборка) | `WXT_API_BASE_URL` |

### Хостинг и установка у клиента

- **Хостинг заказчика (вне релизов, Q-07):** prod-override (`compose.prod.yaml`) без `mailpit` и `node`. Плюс TLS и HSTS, бэкапы (`pg_dump`, хранение 14 дней), отчёт об ошибках, SMTP.
- **On-premise:** вне R1 (В-07). Готовность обеспечивают конфигурация через env и единый compose.

## Рассмотренные альтернативы

1. **Laravel Sail.** Заточен под dev, не даёт связку Nginx + FPM и prod-схему. Отклонено.
2. **Supervisor в одном контейнере.** Хуже с логами и перезапусками. Отклонено.
3. **Cron на хосте.** Добавляет зависимость от сервера. Отклонено.

## Последствия

- (+) `docker compose up -d` поднимает всё, команды CLAUDE.md работают как есть.
- (+) Вся конфигурация в env.
- (−) Восемь сервисов в dev. `node` и `mailpit` вынесены в профиль `dev`, их можно не поднимать.
- (−) Три ключа HMAC и шифрования нужно сгенерировать при установке и сохранить.
- (−) Prod-hardening остаётся долгом до выбора хостинга.
