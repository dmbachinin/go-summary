# Standard Go Project Layout

`Standard Go Project Layout` — это неофициальный набор рекомендаций по организации структуры проекта на Go.

Он помогает поддерживать читаемость, масштабируемость и единообразие кода. Этот подход не навязан компилятором, но признан де-факто стандартом в сообществе.

`ДОКУМЕНТАЦИЯ:` <https://github.com/golang-standards/project-layout/blob/master/README_ru.md>

## Основные особенности

- cmd/ — директория с исполняемыми приложениями.
  - Каждый подкаталог соответствует отдельной программе.
  - Внутри — только main.go, логика выносится в internal/ или pkg/.
- internal/ — приватный код, недоступный за пределами модуля.
  - Используется для бизнес-логики, сервисов, реализации API.
  - Аналог private-пакетов в Java.
- pkg/ — публичные пакеты, которые можно переиспользовать в других проектах.
  - Аналог библиотек (JAR) в Java.
- api/ — схемы API (Swagger/OpenAPI/Protobuf/GraphQL).
- configs/ — конфигурации приложений в формате YAML/JSON/TOML.
- scripts/ — вспомогательные скрипты для CI/CD, генерации кода, деплоя.
- build/ — артефакты сборки (Dockerfile, CI pipeline, бинарники).
- deployments/ — Kubernetes манифесты, Ansible, Terraform и т.п.
- test/ — дополнительные интеграционные тесты.
- docs/ — документация проекта.
- tools/ — утилиты, завязанные на проект.

## Пример структуры проекта

```plantext
├── cmd/
│   ├── app1/
│   │   └── main.go       // точка входа первого приложения
│   └── app2/
│       └── main.go       // точка входа второго приложения
├── internal/
│   └── service/
│       └── logic.go      // приватная бизнес-логика
├── pkg/
│   └── utils/
│       └── strings.go    // публичные вспомогательные функции
├── api/
│   └── openapi.yaml      // описание API
├── configs/
│   └── config.yaml
├── scripts/
│   └── build.sh
├── build/
│   └── docker/
│       └── Dockerfile
├── deployments/
│   └── k8s.yaml
├── test/
│   └── integration_test.go
├── docs/
│   └── README.md
```
