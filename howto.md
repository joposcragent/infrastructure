# How to...

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
