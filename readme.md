# Развертывание Joposcragent

Пока развертывание из говна и палок и требует, чтобы на хосте были docker, bash и java8+, в дальнейшем все будет как-то улучшено.

## Склонировать репозитории

```bash
$ git clone git@github.com:joposcragent/specifications.git
$ git clone git@github.com:joposcragent/infrastructure.git
$ git clone git@github.com:joposcragent/settings-manager.git
$ git clone git@github.com:joposcragent/job-postings-crud.git
$ git clone git@github.com:joposcragent/sentence-transformer.git
$ git clone git@github.com:joposcragent/job-postings-evaluator.git
$ git clone git@github.com:joposcragent/crawler-headhunter.git
$ git clone git@github.com:joposcragent/celery-orchestrator.git
$ git clone git@github.com:joposcragent/web-front.git

```

## Собрать docker-образы

1. Flyway
   ```bash
   $ cd specifications/database-schema
   $ ./build-image.sh
   ```
2. Собрать kotlin-сервисы командой `./gradlew bootBuildImage` в репозиториях:
    - `$ cd settings-manager && ./gradlew bootBuildImage`
    - `$ cd job-postings-crud && ./gradlew bootBuildImage`
    - `$ cd job-postings-evaluator && ./gradlew bootBuildImage`
3. Собрать остальные сервисы командой  `./scripts/build_image.sh` в репозиториях:
    - `$ cd crawler-headhunter && ./scripts/build_image.sh`
    - `$ cd sentence-transformer && ./scripts/build_image.sh` - этот прямо дохрена долго собирается, это нормально
    - `$ celery-orchestrator && ./scripts/build_image.sh`
    - `$ web-front && ./scripts/build_image.sh`

## Запуск в `docker compose`

После того, как всё собралось, переходим в репу `infrastructure` и кастуем `docker compose up -d`.

## И чо дальше?

1. открываем браузером <http://localhost/>
2. настраиваем референсный контекс (текст своего резюме)
3. пороги релевантности (это лучше эмпирически после первого сбора)
4. поисковые запросы (это не URL, а `https://hh.ru/search/vacancy?вот-всё-что-тут-это-поисковый-запрос` )
5. Идём в Дашборд, жмём "Собрать вакансии" и ждём минут сколько-то там в зависимости от запроса и удачи.
   1. В разделе "Вакансии" можно F5-тиь и наблюдать, как появляются новые вакансии
   2. Процесс будет завершен, когда в оркестраторе задачи `collection-query` получат статус `SUCCESS`

### Ну и всё

Теперь раз в час celery будет запускать задачу collection-batch, которая запустит по каждому поисковому запросу сбор вакансий.
По мере загрузки контента вакансий они будут отправляться на оценку.
Результаты оценки можно посмотреть на фронте.

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

