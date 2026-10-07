готов

1) Учебный сервис предварительной оценки заявки на заём под ПТС: принимает заявку, считает LTV (сумма / оценочная стоимость) и возвращает решение approve / review / reject. Все данные синтетические (по AGENTS.md).
2) Makefile: `make up` (docker compose up -d --build, сервис на http://localhost:8080), `make down`, `make test` (PHPUnit), `make lint` (php -l по backend/ и tests/), `make install` (composer install), `make seed` (залить db/seed.sql в уже поднятую базу), `make logs`, `make ps`, `make help`. docker-compose.yml: сервисы backend (php -S на 8080, тома backend/frontend/tests, ENV DB_DSN/DB_USER/DB_PASSWORD/APP_DEBUG, порт ${APP_PORT:-8080}:8080) и db (mysql:8.0, том db-data, initdb schema.sql и seed.sql, healthcheck mysqladmin ping, порт ${DB_PORT:-3307}:3306).
3) Решение считается в `backend/src/Domain/DecisionEngine.php` (пороги approve_max/review_max из `backend/config/rules.php`); сборка входа/выхода — `backend/src/Domain/AssessmentService.php`; запись решения — `backend/src/Repository/ApplicationRepository.php`.

модель: training-2026-09-minimax-m3
