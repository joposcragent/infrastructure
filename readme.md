# Развертывание Joposcragent

## Локальная сборка

### 1. Склонировать репозитории

```bash
git clone git@github.com:joposcragent/specifications.git
git clone git@github.com:joposcragent/infra.git
git clone git@github.com:joposcragent/settings-manager.git
git clone git@github.com:joposcragent/job-postings-crud.git
git clone git@github.com:joposcragent/sentence-transformer.git
git clone git@github.com:joposcragent/job-postings-evaluator.git
git clone git@github.com:joposcragent/crawler-headhunter.git
git clone git@github.com:joposcragent/orchestration-scheduler.git
git clone git@github.com:joposcragent/orchestration-conductor.git
git clone git@github.com:joposcragent/orchestration-async-jobs-crud.git
git clone git@github.com:joposcragent/web-front.git
```

### 2. Собрать docker-образы

1. Flyway

   ```bash
   cd specifications/database-schema
   ./build-image.sh
   docker compose up -d
   ```

   Без поднятого Postgres из compose Kotlin-сервисы не соберутся (нужна БД для генерации и тестов по вашему процессу).

2. Собрать Kotlin-сервисы командой `./gradlew bootBuildImage` в репозиториях (нужен JDK, совместимый с проектом):

   - `settings-manager`
   - `job-postings-crud`
   - `job-postings-evaluator`
   - `orchestration-scheduler`
   - `orchestration-conductor`
   - `orchestration-async-jobs-crud`

3. Собрать остальные сервисы командой `./scripts/build_image.sh` в репозиториях:

   - `crawler-headhunter`
   - `sentence-transformer` (долго собирается — нормально)
   - `web-front`

### 3. Запуск в `docker compose`

После сборки перейдите в репозиторий `infra` и выполните `docker compose up -d`.
