# Lighthouse
Автоматизированный запуск, сбор статистики, визуализация данных анализатора производительность сайта
## Lighthouse

Более менее полноценная документация доступна по ссылке:
<https://github.com/GoogleChrome/lighthouse/blob/main/docs/readme.md>

Образа доступны по ссылке:
<https://hub.docker.com/r/femtopixel/google-lighthouse>


## ClickHouse

ClickHouse Server Docker Image
https://github.com/ClickHouse/ClickHouse/blob/master/docker/server/README.md#clickhouse-server-docker-image


## kafkactl

kafkactl from snap or:
https://github.com/deviceinsight/kafkactl
https://github.com/jbvmio/kafkactl


## Развертывание

### Собираем образ

```shell
cd ~/lighthouse/lh
docker build -t collector .
```

> Для тестового запуска можно выполнить команду
> docker run -it --rm --cap-add SYS_ADMIN  collector:latest

### Запускаем сервисы

```shell
cd ..
chmod +x ./kafka_cfg/update_run.sh
docker network create collector
docker-compose up -d
docker-compose ps
```
