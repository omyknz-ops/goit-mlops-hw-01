# lesson-3-pm — PM-пакет: контейнеризація ML-моделі

**Курс:** MLOps CI/CD 2.0 (AI&PM), Тема 3 «Контейнеризація ML-моделей» · **Домашнє завдання №1**
**Автор:** Ольга Михайленко, AI Project Manager · **Дата:** 12.10.2026

## Мета завдання

Подивитися на контейнеризацію ML-моделі з позиції AI PM: поставити задачу команді, визначити перевірювані критерії приймання, перевірити delivery-артефакти (Dockerfile, залежності, образи), оцінити ризики і вирішити, чи готове рішення до staging. Код, Dockerfile і Bash-скрипти пише команда; цей пакет — інструмент постановки, контролю і комунікації.

## Короткий контекст

AI/ML-команда контейнеризує pretrained-модель класифікації зображень із `torchvision.models` (ImageNet-1K). Плануються два образи:

- `ml-infer-fat:1.0` — на `python:3.13`, простий, еталон коректності;
- `ml-infer-slim:1.0` — на `python:3.13-slim`, multi-stage, оптимізований для delivery.

Inference повертає top-3 prediction. Головний принцип: **оптимізація не змінює поведінку моделі** — fat і slim на однаковому input дають однаковий результат.

Технічні артефакти команди: `model/model.pt`, `app/inference.py`, `requirements.txt`, `Dockerfile.fat`, `Dockerfile.slim`, `.dockerignore`, `README.md`, `report.md`.

## PM-артефакти і як ними користуватися

Файли розташовані в порядку роботи — кожен наступний спирається на попередні.

| # | Файл | Що це | Як використовувати |
|---|---|---|---|
| 1 | [`project_brief.md`](project_brief.md) | Бізнесовий і технічний контекст | Читати першим: що, навіщо, чому fat і slim, де результат має працювати. Використовувати на kick-off. |
| 2 | [`ai_ml_team_task.md`](ai_ml_team_task.md) | Задача для команди у форматі Jira/GitLab Issue | Скопіювати в трекер. Scope, Out of Scope, Deliverables, Constraints. |
| 3 | [`acceptance_criteria.md`](acceptance_criteria.md) | 16 перевірюваних критеріїв + Definition of Done | Чекліст приймання: кожен критерій має команду перевірки і власника. |
| 4 | [`docker_image_review.md`](docker_image_review.md) | Технічний PM-review: Dockerfile, залежності, запуск, відтворюваність | Заповнювати разом з DevOps і QA, коли команда передасть артефакти. Містить правило «блокер / зауваження». |
| 5 | [`optimization_report_review.md`](optimization_report_review.md) | Таблиця fat vs slim з власниками і командами вимірювань | Вимоги до `report.md` команди + висновок AI PM і Go/No-Go на staging. |
| 6 | [`compose_readiness_checklist.md`](compose_readiness_checklist.md) | Готовність локального стенду (API + Redis + PostgreSQL + monitoring) | Оцінка готовності системи, а не окремого образу. Містить чернетку наступної задачі. |
| 7 | [`risk_register.md`](risk_register.md) | 13 ризиків з ймовірністю, впливом, mitigation і owner | Переглядати щотижня і перед рішенням про staging. |
| 8 | [`stakeholder_summary.md`](stakeholder_summary.md) | Звіт для нетехнічних стейкхолдерів (10 речень) | Надсилати бізнесу як статус. |
| 9 | `README.md` | Цей файл — навігація | Точка входу в пакет. |

## Поточний статус і ключові рішення

- **Задачу поставлено**, критерії і перевірки визначені; фактичних артефактів і метрик від команди ще немає — тому в review-документах статуси ⬜ / `TBD`.
- **Staging: No-Go** до отримання `report.md` з цифрами і підтвердження однакового inference.
- **Узгоджений критерій «inference збігається»:** однакові top-3 `class_id` у тому самому порядку, різниця ймовірностей ≤ 1e-4.
- **Виявлений розрив у scope:** образи — це CLI-inference, а Compose-стенд потребує HTTP API (`/health`, `/predict`). API і Compose винесені в окрему задачу `[ML-INFER-2]`; monitoring на локальному стенді — мінімальний, full monitoring — out of scope.

## Questions to AI/ML Team

**Модель**
1. Який фінальний model artifact використовується (архітектура, ваги — наприклад, ResNet-18 `IMAGENET1K_V1`)?
2. Яка його версія і SHA-256 `model/model.pt`?
3. Модель завантажується лише з `model.pt`, без звернення в інтернет під час запуску?
4. Чи дозволяють умови ліцензії ваг комерційне використання?

**Input / Output**
5. Який input format очікується (шлях до файлу, JPEG/PNG, обмеження розміру)?
6. Який output format: структура JSON top-3, поле `model_version`, поведінка при помилці і exit code?

**Середовище і залежності**
7. Чи підтверджена сумісність із `python:3.13`?
8. Чи підтверджена сумісність із `python:3.13-slim` (системні бібліотеки, потрібні pillow)?
9. Чи сумісні `torch`, `torchvision`, `pillow` з обраною версією Python — які саме версії зафіксовані?
10. Чи однакова збірка torch (CPU-only) у fat і slim?
11. Чи є `.dockerignore` і що саме він виключає?

**Результат**
12. Чи збігається inference для fat і slim на `tests/test_image.jpg` — і чим це підтверджено?
13. Чи є `report.md` з порівнянням і командами, якими отримані цифри?
14. Чи README містить команди build/run, і чи пройшла їх людина, яка не писала код?

**Наступний етап**
15. Чи потрібен Docker Compose для локального стенду вже зараз, і хто робить HTTP API?
16. Чи немає секретів у Compose / `.env` і чи не підключається стенд до production-ресурсів?

## Команди для перевірки

Команди відповідають тегам, зафіксованим у задачі. Шлях до test input — `tests/test_image.jpg`.

```bash
# Збірка
docker build --no-cache -f Dockerfile.fat  -t ml-infer-fat:1.0  .
docker build --no-cache -f Dockerfile.slim -t ml-infer-slim:1.0 .

# Запуск inference на однаковому input
docker run --rm -v "$PWD/tests:/data:ro" ml-infer-fat:1.0  /data/test_image.jpg > fat.json
docker run --rm -v "$PWD/tests:/data:ro" ml-infer-slim:1.0 /data/test_image.jpg > slim.json
diff fat.json slim.json        # для точного збігу; інакше — скрипт порівняння з допуском 1e-4

# Робота без мережі
docker run --rm --network none -v "$PWD/tests:/data:ro" ml-infer-slim:1.0 /data/test_image.jpg

# Розмір і шари
docker images | grep ml-infer
docker history ml-infer-fat:1.0
docker history ml-infer-slim:1.0
docker image inspect ml-infer-slim:1.0 --format '{{len .RootFS.Layers}}'

# Base image, multi-stage, вміст
grep ^FROM Dockerfile.fat Dockerfile.slim
docker run --rm --entrypoint sh ml-infer-slim:1.0 -c "ls -R /app"
docker run --rm --entrypoint pip ml-infer-slim:1.0 list

# Той самий model artifact
docker run --rm --entrypoint sha256sum ml-infer-fat:1.0  /app/model/model.pt
docker run --rm --entrypoint sha256sum ml-infer-slim:1.0 /app/model/model.pt

# Security
trivy image ml-infer-slim:1.0      # або: docker scout cves ml-infer-slim:1.0

# Локальний стенд (після задачі ML-INFER-2)
docker compose up --build
curl -f http://localhost:8000/health
curl -F "file=@tests/test_image.jpg" http://localhost:8000/predict
docker compose down -v
```

## Здача

Репозиторій: https://github.com/omyknz-ops/goit-mlops-hw-01 — PM-пакет у папці `lesson-3-pm/`, гілка `lesson-3-pm`.

```bash
git clone https://github.com/omyknz-ops/goit-mlops-hw-01.git
cd goit-mlops-hw-01
git checkout lesson-3-pm
zip -r ДЗ3_PM_Михайленко_Ольга.zip lesson-3-pm/
```

У LMS: архів + посилання на гілку `lesson-3-pm`: https://github.com/omyknz-ops/goit-mlops-hw-01/tree/lesson-3-pm
