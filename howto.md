# How to...

## Пересоздать только один контейнер

Сервис в compose называется **`job-postings-crud`**, образ — `joposcragent/job-postings-crud` с тегом по умолчанию `latest`, у сервиса стоит **`pull_policy: never`**, то есть обновление только из **локально** собранного образа с тем же тегом.

Чтобы пересоздать **только** этот контейнер и не трогать остальные сервисы:

```bash
# в общем случае
docker compose up -d <имя_сервиса>
# в конкретном случае
docker compose -f infra/docker-compose.yaml up -d --no-deps --force-recreate job-postings-crud
```

Запускайте из каталога, где удобно указать путь к файлу (или перейдите в `infra` и уберите `-f ...`).

- **`--no-deps`** — не поднимать заново postgres, flyway и т.д.
- **`--force-recreate`** — новый контейнер даже при том же теге `latest`, чтобы подтянулся **новый** локальный образ с этим тегом.

Если compose всё же не подхватит новый слой образа (редко, но бывает), можно перед этим остановить и удалить контейнер сервиса, затем снова `up`:

```bash
docker compose -f infra/docker-compose.yaml stop job-postings-crud
docker compose -f infra/docker-compose.yaml rm -f job-postings-crud
docker compose -f infra/docker-compose.yaml up -d --no-deps job-postings-crud
```

Итого для типичного случая после `docker build ... -t joposcragent/job-postings-crud:latest` достаточно первой команды с **`--force-recreate`**.

## Посмотреть что пошло не так при запуске

```bash
$ docker inspect infra-settings-manager-1 --format '{{json .State.Health}}'
```

## Sentence-transformer

```bash
$ cd app/sentence-transformer/
$ docker build -t sentence-transformer:latest .
```

## settings-manager

```bash
$ cd app/settings-manager/
$ IMAGE_NAME=joposcragent-settings-manager IMAGE_TAG=0.0.2-SNAPSHOT ./gradlew bootBuildImage
```
