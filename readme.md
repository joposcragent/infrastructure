# Развертывание Joposcragent

Развернуть можно из образов с Docker Hub или из локально собранных.

В первом случае переходите в каталог репозитория `infra` и выполняйте `docker compose up -d`, затем раздел [И чо дальше?](#и-чо-дальше).

Свежесть образов на хабе, к сожалению, пока не гарантируется, но я обещаю очень стараться.

Во втором случае понадобится [локальная сборка](#локальная-сборка).

## Локальная сборка

### Склонировать репозитории

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

### Собрать docker-образы

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

### Запуск в `docker compose`

После сборки перейдите в репозиторий `infra` и выполните `docker compose up -d`.

## И чо дальше?

1. Откройте в браузере [http://localhost/](http://localhost/).
2. Настройте эталонный контекст (текст резюме).
3. Подберите пороги релевантности (лучше после первого сбора).
4. Настройте поисковые запросы (в БД хранится query-string после `?` для страницы поиска hh.ru).
5. На дашборде или в списке вакансий запустите сбор; дождитесь появления записей и смены статусов оценки.

Плановый пакетный сбор выполняет **`orchestration-scheduler`** (задание `COLLECTION_BATCH`): публикует сообщения в **Kafka**, дальше цепочку ведёт **`orchestration-conductor`** и воркеры (`crawler-headhunter`, `job-postings-crud`, `job-postings-evaluator`). Учёт async-job в PostgreSQL ведёт **`orchestration-async-jobs-crud`**.

Состояние топиков и сообщений можно смотреть в **Kafka UI** (порт по умолчанию см. `KAFKA_UI_PORT` в compose, обычно [http://localhost:8090](http://localhost:8090)).

## Всё сделано, но нихрена не работает

Из того, что можно сделать:

1. Убедиться, что контейнеры запущены. Ниже — ориентиры по симптомам:

   - `sentence-transformer` — без него не работает эмбеддинг и оценка.
   - `postgres` — без БД не работают настройки и CRUD.
   - `kafka` — без брокера не работает асинхронная оркестрация.
   - `settings-manager` — базовые настройки и поисковые запросы.
   - `job-postings-crud` — REST и потребление Kafka для создания вакансий.
   - `job-postings-evaluator` — оценка; без него статусы оценки не двигаются.
   - `crawler-headhunter` — сбор с hh.ru; при сбоях не появляются новые вакансии.
   - `orchestration-scheduler` — плановые тики и публикация `collection-batch` в Kafka.
   - `orchestration-conductor` — реакция на события Kafka и ручной enqueue с фронта.
   - `orchestration-async-jobs-crud` — строки `orchestration.async_jobs` и связи; при сбоях расходится учёт джобов.

2. Открыть **Kafka UI** и проверить топики `async-job.*`: есть ли сообщения `*-begin` / `*-result`, задержки, ошибки консьюмеров.

3. Посмотреть логи контейнеров в порядке сценария:

   - `orchestration-scheduler` — срабатывает ли плановое задание.
   - `orchestration-conductor` — публикует ли дочерние джобы, нет ли исключений при обработке.
   - `crawler-headhunter` — запросы к hh.ru и вызовы CRUD.
   - `job-postings-crud` — вставка вакансий и Kafka-исходящие.
   - `job-postings-evaluator` — обработка `job-posting-evaluate`.
   - `orchestration-async-jobs-crud` — обновление статусов джобов по Kafka.
   - `sentence-transformer` — здоровье и время ответа.
