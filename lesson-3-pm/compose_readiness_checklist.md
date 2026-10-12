# Docker Compose Readiness Checklist — локальний стенд ML-системи

**Питання, на яке відповідає документ:** чи можна підняти всю локальну ML-систему **однією командою** і виконати test prediction без ручних дій?

## Важливе застереження: розрив між поточним scope і вимогами до стенду

Поточна задача команди (`ai_ml_team_task.md`) — це **CLI-inference** через `docker run`. Для Compose-стенду потрібен **HTTP-сервіс** з `/health` і `/predict`, а також Redis, PostgreSQL і monitoring. Цього в scope немає, а full monitoring прямо винесено в Out of Scope.

Висновок: робочий fat/slim image — **необхідна, але не достатня** умова для стенду. Compose-стенд оформлюється окремою задачею наступного етапу (чернетка — в кінці документа). Monitoring на локальному стенді — це **мінімальний** варіант (метрики сервісу і логи), а не full monitoring з алертами і drift detection.

## Цільова склад системи

| Сервіс | Роль | Залежить від |
|---|---|---|
| `model-api` | HTTP API: приймає запит, повертає prediction; endpoints `/health`, `/predict` | `model-runtime` (або модель всередині API), Redis, PostgreSQL |
| `model-runtime` | Inference на базі `ml-infer-slim:1.0` (окремий сервіс або вбудований у `model-api` — рішення команди) | Файл моделі |
| `redis` | Кеш результатів / черга запитів | — |
| `postgres` | Лог запитів і prediction (аудит, аналітика) | — |
| `monitoring` | Мінімальний: збір метрик `model-api` (наприклад, Prometheus) | `model-api` |

## Checklist

Статуси: ✅ OK · ⚠️ Зауваження · ❌ Не виконано · ⬜ Не перевірено.
**Поточний стан (12.10.2026):** Compose-файлу немає, статус усіх пунктів — ⬜ / ❌ (поза поточним scope).

| # | Перевірка | Як перевірити | Що вважаємо OK | Статус |
|---|---|---|---|---|
| C-01 | Є `compose.yaml` | `ls compose.yaml` (або `docker-compose.yml`) | Файл у корені репозиторію | ❌ немає |
| C-02 | Описаний `model-api` | `docker compose config --services` | Є сервіс `model-api` з `build` або `image: ml-infer-slim:1.0`-based | ⬜ |
| C-03 | Описані всі залежні сервіси | `docker compose config --services` | `model-api`, `model-runtime` (або обґрунтовано вбудований), `redis`, `postgres`, `monitoring` | ⬜ |
| C-04 | Сервіси не потребують ручного запуску | `docker compose up --build` на чистій машині | Жодного `docker run` чи ручної ініціалізації БД; міграції/init-скрипти запускаються автоматично; `depends_on` з `condition: service_healthy` | ⬜ |
| C-05 | Правильно налаштований порт | `docker compose ps` | `model-api` доступний на погодженому порту (наприклад, `8000:8000`); порти Redis/PostgreSQL не відкриті назовні без потреби | ⬜ |
| C-06 | Є health endpoint | `curl -f http://localhost:8000/health` | HTTP 200 **лише коли модель завантажена** (а не просто «процес запущений»); healthcheck описаний у Compose | ⬜ |
| C-07 | Є predict endpoint | `curl -F "file=@tests/test_image.jpg" http://localhost:8000/predict` | Повертає top-3 JSON, ідентичний `tests/expected_output.json` (у межах `1e-4`) | ⬜ |
| C-08 | Конфігурації винесені в `.env` | `cat .env.example`; `grep -n "environment\|env_file" compose.yaml` | Порти, `MODEL_PATH`, `MODEL_VERSION`, адреси Redis/PostgreSQL — через змінні; у репо лише `.env.example`, `.env` — у `.gitignore` | ⬜ |
| C-09 | Немає production-секретів | Перегляд `compose.yaml`, `.env.example`, `git log -p` | Лише локальні тестові значення (`POSTGRES_PASSWORD=local_dev_only`); жодних production-ключів, токенів, URL; `chmod 600 .env` локально | ⬜ |
| C-10 | Стенд не підключений до production DB / Redis / queue | `grep -n "host\|url" compose.yaml .env.example` | Усі адреси — імена сервісів усередині Compose-мережі (`postgres`, `redis`); жодних зовнішніх хостів | ⬜ |
| C-11 | Команда `docker compose up --build` описана в README | README → розділ «Локальний стенд» | Є команда запуску, перевірки (`curl`) і зупинки (`docker compose down -v`) | ⬜ |
| C-12 | Модель монтується read-only | `grep -n "model" compose.yaml` | `./model:/app/model:ro` або модель всередині образу | ⬜ |
| C-13 | Зв'язок зі staging описаний | README / коментар команди | Команда пояснює, що з цього стенду переходить у staging і що змінюється (секрети → secret manager, БД → керована) | ⬜ |

## Фінальний висновок

**Локальний стенд можна вважати готовим, якщо всі сервіси піднімаються однією командою, а test prediction виконується без ручних дій.**

На 12.10.2026 стенд **не готовий** — і це очікувано: API і Compose не входили в scope першої задачі. Готовність окремого image (fat/slim) не означає готовність системи до інтеграційного середовища. Перед staging потрібна окрема задача.

## Чернетка наступної задачі для команди

**[ML-INFER-2] Docker Compose стенд для локального запуску ML inference system**

- Обгорнути inference у HTTP API (`/health`, `/predict`) на базі `ml-infer-slim:1.0`.
- Описати в `compose.yaml`: `model-api`, `redis`, `postgres`, мінімальний `monitoring`.
- Конфігурація — через `.env` (у репо лише `.env.example`), без production-секретів і production-адрес.
- **Acceptance:** `docker compose up --build` піднімає все; `/health` = 200 після завантаження моделі; `/predict` на `tests/test_image.jpg` повертає той самий top-3, що й CLI; README описує запуск, перевірку і зупинку.
- **Out of Scope:** алерти, drift detection, production Kubernetes.
- **Owner:** AI/ML Engineer + DevOps; review — AI PM, QA.
