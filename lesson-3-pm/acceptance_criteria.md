# Acceptance Criteria — fat і slim Docker images для inference

Кожен критерій відповідає на питання: **«Як ми зрозуміємо, що задача виконана правильно?»** Критерій вважається виконаним лише тоді, коли є evidence (вивід команди, файл, скриншот), а не усне «зроблено».

Позначення образів: `ml-infer-fat:1.0` (далі — fat), `ml-infer-slim:1.0` (далі — slim).

## 1. Base images і структура збірки

| ID | Критерій | Як перевіряємо | Evidence | Хто перевіряє |
|---|---|---|---|---|
| AC-01 | Fat image використовує `python:3.13` | `grep ^FROM Dockerfile.fat` → `FROM python:3.13`; `docker run --rm --entrypoint python ml-infer-fat:1.0 --version` → `Python 3.13.x` | Вивід команд у `report.md` | DevOps |
| AC-02 | Slim image використовує `python:3.13-slim` | `grep ^FROM Dockerfile.slim` → фінальний stage `FROM python:3.13-slim`; `--version` → `Python 3.13.x` | Вивід команд | DevOps |
| AC-03 | Slim зібраний через multi-stage build | У `Dockerfile.slim` ≥ 2 інструкції `FROM`, фінальний stage копіює лише результат (`COPY --from=...`); у фінальному образі немає `gcc`/build-tools | `grep -c ^FROM Dockerfile.slim` ≥ 2; `docker run --rm --entrypoint sh ml-infer-slim:1.0 -c "which gcc"` → порожньо | DevOps |
| AC-04 | У репозиторії є `.dockerignore` | Файл існує і виключає `.git`, `__pycache__`, `*.ipynb`, `data/`, `.env`, `venv/` | Вміст файлу | DevOps |

## 2. Збірка і запуск

| ID | Критерій | Як перевіряємо | Evidence | Хто перевіряє |
|---|---|---|---|---|
| AC-05 | Обидва образи збираються без помилок | `docker build -f Dockerfile.fat -t ml-infer-fat:1.0 .` і `docker build -f Dockerfile.slim -t ml-infer-slim:1.0 .` завершуються з exit code 0 на чистій машині (`--no-cache`) | Лог збірки | QA |
| AC-06 | Обидва образи запускаються через `docker run` без ручного встановлення залежностей | `docker run --rm -v "$PWD/tests:/data:ro" <image> /data/test_image.jpg` → exit code 0 | Вивід у консоль | QA |
| AC-07 | Залежності зафіксовані | Кожен рядок `requirements.txt` має `==`; обидва Dockerfile використовують той самий файл | Вміст `requirements.txt` | AI PM / DevOps |

## 3. Коректність inference

| ID | Критерій | Як перевіряємо | Evidence | Хто перевіряє |
|---|---|---|---|---|
| AC-08 | Для fat і slim використовується однаковий test input | Один файл `tests/test_image.jpg`, SHA-256 зафіксований у README | `sha256sum tests/test_image.jpg` | QA |
| AC-09 | Inference повертає top-3 prediction | JSON містить рівно 3 елементи `class_id`, `label`, `probability`, відсортовані за спаданням, + поле `model_version` | Вивід JSON | QA |
| AC-10 | Результат inference у fat і slim збігається | Однакові `class_id` у тому самому порядку; `|p_fat − p_slim| ≤ 1e-4` для кожного класу; обидва збігаються з `tests/expected_output.json` | `diff` / скрипт порівняння, вивід у `report.md` | QA |
| AC-11 | Модель завантажується з `model/model.pt`, не з інтернету | `docker run --network none ...` проходить успішно | Вивід команди | QA |

> Чому допуск 1e-4, а не «байт-у-байт»: різні base images можуть мати різні системні математичні бібліотеки, що дає розбіжність у останніх знаках float. Для бізнесу важливо, що top-3 класи і їхній порядок однакові. Якщо розбіжність > 1e-4 — це сигнал, що оптимізація змінила поведінку моделі, і slim не приймається.

## 4. Чистота образу

| ID | Критерій | Як перевіряємо | Evidence | Хто перевіряє |
|---|---|---|---|---|
| AC-12 | У slim image немає зайвих файлів | `docker run --rm --entrypoint sh ml-infer-slim:1.0 -c "ls -R /app"` → лише `inference.py`, `model/`, мінімально потрібне; немає `.git`, ноутбуків, датасетів, `tests/`, `.env`, pip-кешу | Вивід команди | DevOps |
| AC-13 | В образах немає секретів | Пошук у файловій системі образу й `docker history --no-trunc` не показує токенів/паролів | Вивід команд | DevOps |

## 5. Документація

| ID | Критерій | Як перевіряємо | Evidence | Хто перевіряє |
|---|---|---|---|---|
| AC-14 | README містить команди для build/run | Людина, яка не писала код, виконує команди з README по черзі й отримує prediction | Результат «холодного» прогону | QA або колега з іншої команди |
| AC-15 | README описує модель | Архітектура, джерело ваг, версія, SHA-256 `model.pt`, формат input/output | Розділ README | AI PM |
| AC-16 | `report.md` містить порівняння fat і slim | Таблиця: розмір, кількість шарів, час збірки, час запуску, inference result, security findings, зайві файли — **кожна цифра з командою, якою отримана** | `report.md` | AI PM |

---

## Definition of Done

Задача вважається виконаною, якщо **будь-який член команди може клонувати репозиторій, виконати команди з README, зібрати fat і slim image, запустити inference на однаковому input і побачити однаковий результат prediction.**

Додатково для закриття задачі:

- [ ] Усі критерії AC-01 … AC-16 мають evidence.
- [ ] Перевірку AC-14 виконала людина, яка **не** писала Dockerfile.
- [ ] `report.md` показує, що slim менший за fat, або письмово пояснює, чому ні.
- [ ] Відкриті ризики з `risk_register.md` зі статусом High мають власника і план.
- [ ] AI PM підтвердив готовність переходу до staging або зафіксував, чого бракує.

## Що не є критерієм приймання

- Accuracy моделі (out of scope — модель pretrained, не перенавчаємо).
- Робота в Kubernetes або production.
- Наявність HTTP API та Compose-стенду (окрема задача).
