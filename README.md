# SAZ-Lab — Security Analysis Lab

Учебный стенд для полного цикла анализа защищённости приложений:
SCA → SAST → CA → DAST → управление уязвимостями.

---

## Описание стенда

Локальный стенд развёрнут на Linux (Kali) с использованием Docker и Docker Compose.  

| Компонент | Версия |
|---|---|
| ОС | Kali GNU/Linux Rolling, Linux 'Kali' 6.18.12+kali-amd64 x86-64 |
| Docker | 28.5.2 |
| Docker Compose | 2.40.3-3 |
| Git | 2.51.0 |
| Python | 3.13.12 |
| Node.js | v24.15.0 |
| OpenJDK | 21.0.11 |

| Образ | Назначение |
|---|---|
| `dependencytrack/frontend` | Frontend Dependency-Track |
| `dependencytrack/apiserver` | API-сервер Dependency-Track |
| `postgres:17-alpine` | База данных Dependency-Track |
| `defectdojo/defectdojo-django:latest` | Backend DefectDojo |
| `defectdojo/defectdojo-nginx:latest` | Nginx для DefectDojo |
| `postgres:18.3-alpine` | База данных DefectDojo |
| `valkey/valkey:9.0.3-alpine` | Кэш и очереди DefectDojo |
| `ghcr.io/zaproxy/zaproxy:stable` | OWASP ZAP |

Назначенные порты:

| Сервис | Порт |
|---|---|
| Juice Shop | `127.0.0.1:3000:3000` |
| WebGoat | `127.0.0.1:8084:8080` |
| WebGoat WebWolf | `127.0.0.1:9090:9090` |
| Dependency-Track | `8090` |
| DefectDojo | `8080` |

## Мишени

| Приложение       | Docker-образ                        | Порт контейнера | Публикуемый порт / URL                 |
|-----------------|--------------------------------------|-----------------|----------------------------------------|
| OWASP Juice Shop| `bkimminich/juice-shop:latest`       | 3000            | http://localhost:3000                  |
| OWASP WebGoat   | `webgoat/webgoat:latest`             | 8080 (HTTP), 9090 (admin) | http://localhost:8080 / http://localhost:8888/WebGoat |

---

## Инструменты

| Инструмент                        | Версия       | Назначение                                      | URL / Порт              |
|----------------------------------|-------------|-------------------------------------------------|-------------------------|
| Dependency-Track (Frontend)      | *(запланировано)* | SCA / SBOM-платформа — UI                     | http://localhost:8080   |
| Dependency-Track (API Server)    | *(запланировано)* | SCA / SBOM-платформа — API                    | http://localhost:8081   |
| DefectDojo                       | *(запланировано)* | AppSec Portal — управление уязвимостями       | http://localhost:8082   |
| OWASP ZAP                        | stable      | DAST — динамический анализ                     | http://localhost:8090   |
| Semgrep                          | 1.165.0     | SAST — статический анализ исходного кода       | CLI                     |
| Trivy                            | *(запланировано)* | CA — анализ контейнерных образов             | CLI                     |
| cdxgen                           | *(запланировано)* | SBOM-генератор (npm)                          | CLI                     |
| cyclonedx-maven-plugin           | *(запланировано)* | SBOM-генератор (Maven)                        | Maven plugin            |

**Источники уязвимостей (план):**

- OSS Index (Sonatype) + GitHub Advisory Database в Dependency-Track.
- NVD будет отключён из-за сетевых ограничений (см. «Известные проблемы»).

---



Актуальное состояние контейнеров:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

```text
NAMES        STATUS                   PORTS
webgoat      Up 3 hours (unhealthy)   127.0.0.1:8080->8080/tcp, 127.0.0.1:9090->9090/tcp
juice-shop   Up 3 hours               0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
```

SHA-дайджесты образов:

```text
Juice Shop: bkimminich/juice-shop@sha256:5d572029141b32e065edefc9b09ca88567296ff7134ebe3181ab7ac9e9ed4056
WebGoat: webgoat/webgoat@sha256:3101bd9e7bcfe122d7ef91e690ef3720de36cc4e86b3d06763a1ddf2e2751a4b
```

---

## Структура проекта

```text
saz-lab/
├── tools/
│   ├── dependency-track/                 # SCA-платформа (compose-файлы, конфигурация)
│   ├── defectdojo/                       # AppSec Portal (compose-файлы, импорт-скрипты)
│   ├── zap/                              # ZAP, Automation Framework, контексты
│   └── semgrep-custom/
│       └── rules.yaml                    # Кастомные правила Semgrep (SAST)
├── targets/
│   ├── juice-shop/                       # Исходники OWASP Juice Shop
│   └── webgoat/                          # Исходники OWASP WebGoat
├── reports/
│   ├── sca/                              # SBOM (CycloneDX) и SCA-отчёты
│   ├── sast/                             # Semgrep SARIF / JSON
│   ├── ca/                               # Trivy JSON (образ + файловая система)
│   ├── iac/                              # Trivy config-scan JSON
│   ├── dast/                             # ZAP HTML / JSON / XML
│   └── asoc/                             # Итоговый PDF / экспорт из DefectDojo
└── triage/
    ├── triage_report.md                  # Аналитический отчёт (SAST/SCA/DAST/CA)
    └── screenshots/                      # Скриншоты UI всех инструментов
```

---

## Пошаговый журнал выполнения

### Шаг 0 — Проверка окружения

```bash
docker --version
docker compose version
git --version
python3 --version
node --version
```

Результат (зафиксировано для отчётности):

```text
Docker Compose version 2.40.3-3
Docker version 28.5.2+dfsg4, build 9cc6dea35e9a963f281434761c656fba4ac43aed
git version 2.51.0
Python 3.13.12
Node.js v24.15.0
```

Скриншоты:

- `triage/screenshots/juice-shop-running_000.png`
- `triage/screenshots/juice-shop-running_001.png`
- `triage/screenshots/webgoat-running_000.png`

---

### Шаг 1 — Развёртывание мишеней (Juice Shop и WebGoat)

*(Фактически уже запущены; команды приведены для воспроизводимости.)*

```bash
# Juice Shop
docker run -d --name juice-shop -p 3000:3000 bkimminich/juice-shop:latest

# WebGoat
docker run -d --name webgoat -p 8080:8080 -p 9090:9090 webgoat/webgoat:latest

# Проверить состояние контейнеров
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Ожидаемый результат:

- Juice Shop доступен по `http://localhost:3000`.
- WebGoat доступен по `http://localhost:8080` (или по `http://localhost:8888/WebGoat` при использовании другого проксирования).

Скриншоты:

- `triage/screenshots/juice-shop-running_000.png`
- `triage/screenshots/webgoat-running_000.png`

---

### Шаг 2 — SAST: Semgrep (установлено и выполнено)

#### 2.1. Установка Semgrep

```bash
python3 -m pip install --user semgrep
semgrep --version
```

Результат:

```text
1.165.0
```

Скриншот:

- `triage/screenshots/semgrep-version.png`

#### 2.2. Semgrep SAST — Juice Shop

Рабочая директория: `targets/juice-shop`

Команды:

```bash
cd targets/juice-shop

# SARIF
semgrep scan \
  --config p/owasp-top-ten \
  --config p/security-audit \
  --config p/javascript \
  --metrics=off \
  --sarif --output ../../reports/sast/juice-shop.sarif

# JSON
semgrep scan \
  --config p/owasp-top-ten \
  --config p/security-audit \
  --config p/javascript \
  --metrics=off \
  --json --output ../../reports/sast/juice-shop.json
```

Результат:

- `reports/sast/juice-shop.sarif`
- `reports/sast/juice-shop.json`
- Findings: 28 (28 blocking), Rules run: 143, Targets scanned: 1114.

Скриншоты:

- `triage/screenshots/semgrep-juice-shop.sarif.png`
- `triage/screenshots/semgrep-juice-shop.JSON.png`

#### 2.3. Semgrep SAST — WebGoat

Рабочая директория: `targets/webgoat`

Команды:

```bash
cd targets/webgoat

# SARIF
semgrep scan \
  --config p/owasp-top-ten \
  --config p/security-audit \
  --config p/java \
  --metrics=off \
  --sarif --output ../../reports/sast/webgoat.sarif

# JSON
semgrep scan \
  --config p/owasp-top-ten \
  --config p/security-audit \
  --config p/java \
  --metrics=off \
  --json --output ../../reports/sast/webgoat.json
```

Результат:

- `reports/sast/webgoat.sarif`
- `reports/sast/webgoat.json`
- Findings: 47 (47 blocking), Rules run: 208, Targets scanned: 1097.

Скриншоты:

- `triage/screenshots/semgrep-webgoat.sarif.png`
- `triage/screenshots/semgrep-webgoat.json.png`

#### 2.4. Собственное правило Semgrep

Файл: `tools/semgrep-custom/rules.yaml`

```yaml
rules:
  - id: custom-no-eval
    pattern: eval(...)
    message: "Использование eval() небезопасно — потенциал RCE"
    languages: [javascript, typescript]
    severity: ERROR
    metadata:
      category: security
      cwe: "CWE-95"
      owasp: "A03:2021 - Injection"
```

Запуск:

```bash
cd saz-lab
semgrep scan \
  --config tools/semgrep-custom/rules.yaml \
  targets/juice-shop \
  --metrics=off \
  --json --output reports/sast/juice-shop-custom-no-eval.json
```

Результат:

- `reports/sast/juice-shop-custom-no-eval.json`
- Findings: 2 (2 blocking), Rules run: 1.

Скриншот:

- `triage/screenshots/semgrep-custom-no-eval.png`

---

### Шаг 3 — SCA: Dependency-Track (план / шаблон шагов)

*(На момент текущего шага лабораторной фактически ещё не поднималось, ниже — план развёртывания.)*

```bash
cd tools/dependency-track
docker compose up -d

# Подождать ~60 сек до инициализации
curl -s http://localhost:8081/api/version | jq .version

# Открыть UI: http://localhost:8080
# Login: admin / admin -> сменить пароль при первом входе
```

Планируемые действия:

- Генерация SBOM для Juice Shop и WebGoat (cdxgen / cyclonedx-maven-plugin).
- Загрузка SBOM в Dependency-Track через API.
- Анализ зависимостей и уязвимостей.

---

### Шаг 4 — CA: Trivy (план / шаблон шагов)

*(Будет выполнено на следующем этапе лабораторной.)*

Пример команд:

```bash
# Juice Shop image
trivy image \
  --format json \
  --output reports/ca/juice-shop-image.json \
  bkimminich/juice-shop:latest

# WebGoat image
trivy image \
  --format json \
  --output reports/ca/webgoat-image.json \
  webgoat/webgoat:latest
```

План:

- Сканирование образов на уязвимости (HIGH/CRITICAL).
- При необходимости — filesystem scan (`trivy fs`) и IaC (`trivy config`).

---

### Шаг 5 — DAST: OWASP ZAP (план / шаблон шагов)

*(ZAP ещё не поднимался; ниже — план развёртывания и использования.)*

```bash
cd tools/zap
docker compose up -d

# Проверить готовность ZAP API
curl http://localhost:8090/JSON/core/view/version/
```

Планируемые сценарии:

- Baseline-сканирование Juice Shop и WebGoat (пассивный анализ).
- Full scan для Juice Shop (активные атаки).
- Загрузка отчётов в `reports/dast/` в форматах HTML/JSON/XML.

---

### Шаг 6 — Импорт отчётов в DefectDojo (план / шаблон шагов)

*(DefectDojo пока не развёрнут; здесь — только описание будущих шагов.)*

План:

1. Поднять DefectDojo:

```bash
cd tools/defectdojo
docker compose up -d

# Дождаться окончания инициализации
docker compose logs -f dojo-initializer
```

2. Получить API Token через UI (admin / admin, смена пароля).
3. Импортировать отчёты SAST/SCA/CA/DAST в единый Engagement:

- `reports/sast/juice-shop.sarif` — SAST.
- `reports/sast/webgoat.sarif` — SAST.
- `reports/sca/*.json` — SCA.
- `reports/ca/*.json` — CA.
- `reports/dast/*.xml` — DAST.

4. Включить дедупликацию и политики.

---

## Триаж (кратко, ссылка на triage_report.md)

Подробный триаж SAST (Semgrep) оформляется в `triage/triage_report.md` и включает:

- Минимум 5 находок.
- Классификацию: True Positive / False Positive / Risk Accepted.
- Привязку к CWE / OWASP.
- Remediation (план исправления).

Скриншоты, подтверждающие наличие и анализ находок:

- `triage/screenshots/semgrep-version.png`
- `triage/screenshots/semgrep-juice-shop.sarif.png`
- `triage/screenshots/semgrep-juice-shop.JSON.png`
- `triage/screenshots/semgrep-webgoat.sarif.png`
- `triage/screenshots/semgrep-webgoat.json.png`
- `triage/screenshots/semgrep-custom-no-eval.png`

---

## Известные проблемы (планируемые / ожидаемые)

### NVD недоступен (SCA / Dependency-Track)

Ожидаемая проблема: `api.nvd.nist.gov` может быть недоступен из учебной сети.  
Решение (будет применено при настройке DT):

```yaml
# tools/dependency-track/docker-compose.yml
- ALPINE_NVD_ENABLED=false
- ALPINE_OSSINDEX_ENABLED=true
- ALPINE_GITHUB_ADVISORIES_ENABLED=true
```

### Долгий старт DefectDojo

Инициализация (миграции PostgreSQL) занимает 2–3 минуты. При отсутствии ответа Nginx:

```bash
docker compose -f tools/defectdojo/docker-compose.yml logs -f dojo-initializer
```

### Ложные срабатывания ZAP на WebGoat

WebGoat намеренно ломает сессии и реализует уязвимости для обучения. ZAP Full Scan может давать ложные срабатывания и устаревшие URL — потребуется аккуратный триаж и использование контекстов / аутентификации.

---

## Учётные данные по умолчанию (только для учебной среды)

*(Будут использоваться при настройке стенда; в production такие учётки недопустимы.)*

| Сервис          | URL                     | Логин               | Пароль    |
|-----------------|------------------------|---------------------|-----------|
| Dependency-Track| http://localhost:8080  | admin               | admin     |
| DefectDojo      | http://localhost:8082  | admin               | admin     |
| Juice Shop      | http://localhost:3000  | admin@juice-sh.op   | admin123  |
| WebGoat         | http://localhost:8080 / http://localhost:8888/WebGoat | webgoat | webgoat |

> Внимание: Не использовать эти учётные данные вне лабораторной среды.

---

## Как воспроизвести текущий прогресс

Чтобы воспроизвести сделанную часть лабораторной:

1. Поднять мишени (Juice Shop и WebGoat).
2. Установить Semgrep (`python3 -m pip install --user semgrep`).
3. Запустить SAST-сканы для `targets/juice-shop` и `targets/webgoat` по указанным командам.
4. Сохранить SARIF/JSON в `reports/sast/`.
5. Запустить кастомное правило `custom-no-eval` и убедиться в двух срабатываниях.

Дальнейшие шаги (SCA, CA, DAST, DefectDojo) описаны в виде пошаговых рецептов и будут выполнены на следующих этапах.
