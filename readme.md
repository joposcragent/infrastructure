# Развертывание Joposcragent

Пока развертывание из говна и палок, в дальнейшем все будет сведено только к bash + docker.

## Склонировать репозитории

```bash
$ git clone git@github.com:joposcragent/infrastructure.git
$ git clone git@github.com:joposcragent/settings-manager.git
$ git clone git@github.com:joposcragent/job-postings-crud.git
$ git clone git@github.com:joposcragent/sentence-transformer.git
$ git clone git@github.com:joposcragent/job-postings-evaluator.git
$ git clone git@github.com:joposcragent/crawler-headhunter.git
$ git clone git@github.com:joposcragent/celery-orchestrator.git

```

## Собрать docker-образы

1. Собрать kotlin-сервисы командой `./gradlew bootBuildImage` в репозиториях:
    - `settings-manager`
    - `job-postings-crud`
    - `job-postings-evaluator`
2. Собрать python-сервисы командой  `./scripts/build_image.py` в репозиториях:
    - `crawler-headhunter`
    - `sentence-transformer` - этот прямо дохрена долго собирается, это нормально
3. Собрать node.js-сервисы командой `./scripts/build-image.mjs` в репозиториях:
    - `celery-orchestrator`

## Запуск в `docker compose`

После того, как всё собралось, переходим в репу `infrastructure` и кастуем `docker compose up -d`.

## И чо дальше?

А дальше постманом ;-)

### 1. Установить референсный контекст (резюме свое)

```shell
curl --location 'http://localhost:8080/reference-context' \
--header 'Content-Type: text/plain' \
--header 'Accept: application/json' \
--data 'тут
просто
текст'
```

Ответ `200` — JSON только с полями `vector`, `createdAt`, `updatedAt` (текст контекста в ответе не возвращается).

### 2. Настроить поисковые запросы (минимум один)

```shell
curl --location 'http://localhost:8080/search-query/{{new_random_uuid_v4}}' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--data '{
  "name": "Мой поиск CTO",
  "query": "https://hh.ru/search/vacancy?area=1&items_on_page=20&no_magic=true&order_by=publication_time&ored_clusters=true&professional_role=156&professional_role=160&professional_role=165&professional_role=36&professional_role=73&professional_role=96&professional_role=164&professional_role=104&professional_role=157&professional_role=112&professional_role=113&professional_role=148&professional_role=114&professional_role=124&professional_role=125&professional_role=126&search_field=name&text=%28CTO+OR+%22%D1%80%D1%83%D0%BA%D0%BE%D0%B2%D0%BE%D0%B4%D0%B8%D1%82%D0%B5%D0%BB%D1%8C+%D1%80%D0%B0%D0%B7%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B8%22+OR+%22%D1%82%D0%B5%D1%85%D0%BD%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B8%D0%B9+%D0%B4%D0%B8%D1%80%D0%B5%D0%BA%D1%82%D0%BE%D1%80%22+OR+%22chef+technical+oficer%22+OR+%22%D1%82%D0%B5%D1%85*%D0%BB%D0%B8%D0%B4%22+OR+%22tech*lead%22%29+AND+NOT%28teamlead+OR+%D1%82%D0%B8%D0%BC%D0%BB%D0%B8%D0%B4+OR+%22%D1%80%D1%83%D0%BA%D0%BE%D0%B2%D0%BE%D0%B4%D0%B8%D1%82%D0%B5%D0%BB%D1%8C+%D0%BA%D0%BE%D0%BC%D0%B0%D0%BD%D0%B4%D1%8B%22%29&search_period=1"
}'
```

### 3. Настрить пороги релевантности

Значение полностью зависит от вашего референсного контекста и подбирается эмпирически.

`CONTENT` - порог, после которого вакансия будет оценена, как `RELEVANT`
`NOTIFICATION` - порог, после котого будет отправлено уведомление о вакансии.

```shell
curl --location 'http://localhost:8080/relevance-thresholds/CONTENT' \
--header 'Content-Type: application/json' \
--header 'Accept: text/plain' \
--data '{
  "value": 0.822
}'
```

### Ну и всё

Теперь раз в час celery будет запускать задачу collection-batch, которая запустит по каждому поисковому запросу сбор вакансий.
По мере загрузки контента вакансий они будут отправляться на оценку.
Результаты оценки можно посмотреть через crud, в БД или на фронте, когда он будет сделан.

## Всё сделано, но нихрена не работает

Мои сорясики, в общем-то.

Из того, что можно сделать:

1. Первым делолм убедиться, что все контейнеры запущены, ниже симптомы при падении каждого:
   - `sentence-transformer` - без него не работает эмбеддинг и оценка
   - `postgres` - без БД не будут работать settings и crud
   - `settings-manager` - без этого ничего не работает
   - `job-postings-crud` - crawler вроде что-то делает, но новых вакансий не появляется
   - `job-postings-evaluator` - вакансии создаются, но статус всегда NEW
   - `crawler-headhunter` - новых вакансий нет, задачи collection-query FAILED
   - `redis` - если этот лежит, то оркестратор не работает и flower не открывается
   - `celery-orchestrator-api` - во flower есть только задачи collection-* и нет ни одной progress, finish
   - `celery-orchestrator-worker` - все задачи STARTED и не двигаются, crawler c evaluator'ом запросов не получают
   - `celery-orchestrator-flower` - UI Celery не открывается
   - `celery-orchestrator-beat` - раз в час не создаются автоматом задачи collection-batch
2. Посмотреть UI Celery: <http://localhost:5555/tasks>
   - там будут задачки, которые порождаются процессом
   - в задачках kwargs и статус, оттуда что-то можно понять
   - работает поиск по uuid вакансии (поле поиска в правом верхнем углу)
   - из этого можно сделать какие-то выводы
3. Посмотреть логи контейнеров
   - если в celery нет задач, то - в контейнер `celery-orchestrator-worker`
   - если есть задачи только collection-*, то:
     - `crawler-headhunter` - посмотреть, что происходит при скрапинге
     - `job-postings-crud` - убедиться, что вакансии создаются
     - `job-postings-evaluator` - пытается ли он оценку выполнить
     - `sentence-transformer` - что происходит с ML-моделью
     - `celery-orchestrator-api` - приходят ли в оркестратор и как обрабатываются события от crawler, crud и evaluator
     - `celery-orchestrator-worker` - посмотреть, что воркер с задачами делает
   - шесть и более задач в `RUNNING`
     - все потоки воркера чем-то заняты, вероятнее всего ждут finish-задач, которые по какой-то причине не приходят
     - шесть - волшебное число, которое можно увеличить параметром `--concurrency` в docker-compose.yaml
     - там же `CELERY_ORCHESTRATOR_FINISH_WAIT_TIMEOUT_SECONDS` - таймаут, после которого задача будет прибита автоматически

