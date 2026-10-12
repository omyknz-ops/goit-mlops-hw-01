# Title

**[ML-INFER-1] Контейнеризувати модель класифікації зображень: fat і slim Docker images з ідентичним inference**

| Поле | Значення |
|---|---|
| Тип | Story |
| Пріоритет | High |
| Assignee | AI/ML Engineer |
| Reviewers | DevOps (Dockerfile, registry), QA (перевірка inference) |
| Reporter / Accountable | AI PM — Ольга Михайленко |
| Labels | `mlops`, `docker`, `inference`, `delivery` |
| Estimate | 3 story points (уточнити на планінгу) |

---

## Background

У нас є pretrained-модель класифікації зображень із `torchvision.models` (ImageNet-1K). Зараз inference запускається лише в середовищі ML-інженера, тому його неможливо відтворити в CI/CD чи на staging. Потрібно запакувати модель у Docker image — це буде одиниця delivery для наступних етапів (registry → CI/CD → staging).

Робимо два образи:
- `ml-infer-fat:1.0` — простий варіант, еталон коректності;
- `ml-infer-slim:1.0` — оптимізований варіант для delivery.

> As an AI Project Manager,
> I want the ML model to be packaged into fat and slim Docker images with identical inference behavior,
> so that the team can run and verify the model consistently across local, CI/CD and staging environments.

## Scope

1. Зберегти ваги pretrained-моделі у `model/model.pt`; зафіксувати архітектуру, версію ваг і SHA-256 у README.
2. Написати `app/inference.py`: приймає шлях до зображення, виконує той самий preprocessing, повертає **top-3 prediction** у JSON (`class_id`, `label`, `probability`) + `model_version`. Модель переводиться в режим `eval()`.
3. Створити `Dockerfile.fat` на базі `python:3.13`.
4. Створити `Dockerfile.slim` на базі `python:3.13-slim` з **multi-stage build** (build-етап встановлює залежності, у фінальний образ потрапляють лише runtime-артефакти).
5. Створити `.dockerignore` (виключити `.git`, `__pycache__`, ноутбуки, датасети, `.env`, віртуальні середовища, локальні артефакти).
6. Зафіксувати версії всіх залежностей у `requirements.txt` (`==`, без діапазонів); обидва образи використовують **той самий** файл.
7. Додати тестовий input `tests/test_image.jpg` та еталонний вихід `tests/expected_output.json`, отриманий із fat image.
8. Перевірити, що fat і slim дають однаковий inference на test input.
9. Оновити `README.md` командами build / run / перевірки.
10. Підготувати `report.md` з порівнянням fat vs slim (метрики — див. Expected Deliverables).

## Out of Scope

- Model retraining.
- Покращення accuracy або зміна архітектури моделі.
- Production Kubernetes (manifests, Helm, autoscaling).
- Full monitoring (Prometheus/Grafana, алерти, drift detection).
- HTTP API (`/health`, `/predict`) і Docker Compose-стенд — окрема задача наступного етапу (див. `compose_readiness_checklist.md`).

## Expected Deliverables

| Артефакт | Опис |
|---|---|
| `Dockerfile.fat` | База `python:3.13` |
| `Dockerfile.slim` | База `python:3.13-slim`, multi-stage |
| `.dockerignore` | Виключає все, що не потрібне для inference |
| `requirements.txt` | Усі версії зафіксовані |
| `app/inference.py` | Inference з top-3 у JSON |
| `model/model.pt` | Артефакт моделі + версія і SHA-256 у README |
| `tests/test_image.jpg`, `tests/expected_output.json` | Однаковий test input і еталон |
| `README.md` | Build / run / compare команди |
| `report.md` | Таблиця fat vs slim: розмір, шари, час збірки, час запуску, inference result, security findings, зайві файли — **з командами, якими отримано кожну цифру** |
| Зібрані образи | `ml-infer-fat:1.0`, `ml-infer-slim:1.0` |

## Technical Constraints

- Fat: `FROM python:3.13`. Slim: `FROM python:3.13-slim` у фінальному stage.
- Slim: multi-stage build обов'язковий; у фінальному образі немає компіляторів, build-tools, pip-кешу, dev-залежностей.
- Python 3.13: версії `torch`, `torchvision`, `pillow`, `numpy` мають бути сумісні з 3.13 — команда підтверджує в README.
- Обидва образи використовують **один і той самий** `requirements.txt`, той самий `model.pt`, той самий preprocessing і **ту саму збірку torch** (CPU-only wheels для обох або однакова для обох) — інакше порівняння inference некоректне.
- Ваги не завантажуються з інтернету під час `docker run` — лише з `model/model.pt`.
- Жодних секретів, токенів чи `.env` в образі.
- Рекомендовано: slim запускається від non-root користувача (`USER`).
- Теги образів — саме `ml-infer-fat:1.0` і `ml-infer-slim:1.0`.

## Acceptance Criteria Summary

Повний перелік — у `acceptance_criteria.md`. Коротко:

- [ ] Fat на `python:3.13`, slim на `python:3.13-slim` з multi-stage build.
- [ ] Обидва образи збираються без помилок і запускаються через `docker run`.
- [ ] На однаковому test input обидва повертають top-3; класи й порядок збігаються, ймовірності — з точністю до 1e-4.
- [ ] `requirements.txt` зафіксований, `.dockerignore` є, зайвих файлів у slim немає.
- [ ] README містить build/run команди; `report.md` містить порівняння з цифрами.

**Definition of Done:** будь-який член команди клонує репозиторій, виконує команди з README, збирає fat і slim, запускає inference на однаковому input і бачить однаковий prediction.
