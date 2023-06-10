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

### Выполняем настройку

Смотрим состояние. Если все запустилось успешно, то
- настриваем ClickHouse
- настриваем Grafana


## Настройка ClickHouse

Подключаемся к базе clickhouse

```shell
docker exec -it clickhouse bash
clickhouse-client
```

Настройка таблиц

```sql

--- Создаем базу
CREATE DATABASE IF NOT EXISTS ers

--- переключаемся на нее (не обязательно)
use ers

--- Создаем таблицу и механизм извлечения данных из kafka
CREATE TABLE ers.collector_consumer
(
  deviceType String,
  page String,
  first_contentful_paint Float32,
  largest_contentful_paint Float32,
  speed_index Float32,
  total_blocking_time Float32,
  cumulative_layout_shift Float32,
  server_response_time Float32,
  first_cpu_idle Float32,
  interactive Float32,
  mainthread_work_breakdown Float32,
  perfomance Float32,
  vhost String
)
ENGINE = Kafka('kafka:9092', 'collector', 'group1', 'JSONEachRow')
SETTINGS kafka_skip_broken_messages=100, kafka_num_consumers=1;

--- Создаем таблицу для хранения статистики
CREATE TABLE ers.collector
(
    `event_date` Date DEFAULT toDate(event_time),
    `event_time` DateTime DEFAULT now(),
    `deviceType` String,
    `page` String,
    `first_contentful_paint` Float32,
    `largest_contentful_paint` Float32,
    `speed_index` Float32,
    `total_blocking_time` Float32,
    `cumulative_layout_shift` Float32,
    `server_response_time` Float32,
    `first_cpu_idle` Float32,
    `interactive` Float32,
    `mainthread_work_breakdown` Float32,
    `perfomance` Float32,
    `vhost` String
)
ENGINE = MergeTree
ORDER BY event_date
SETTINGS index_granularity = 8192

--- MATERIALIZED VIEW для формирования данных в ers.collector
--- По непонятной мне причине, это не нужно на проде, но без него на тесте идет потеря данных
CREATE MATERIALIZED VIEW ers.consumer TO ers.collector AS
SELECT * FROM  ers.collector_consumer;


--- Извлечь структуру таблиц
SHOW create ers.collector
SHOW create ers.collector_consumer
SHOW create ers.consumer

--- Удаление табил выполняется команадами
--- drop table ers.collector
--- drop table ers.collector_consumer

--- Вывести список баз
SHOW DATABASES
--- Вывести список таблиц из базы ers
SHOW TABLES FROM ers

```

Выходим 
CTRL + D


# Локальное развертывание

Развертывание выполняется на машине, на которой предварительно установлены docker с плагином compose
Для запуска необходимо не менее 2 Гб ОЗУ.

```shell
# Добавляем запись в hosts
sudo nano /etc/hosts
192.168.56.74 kafka

# Создаем файл конфигурации для kafkactl
nano $HOME/.config/kafkactl
contexts:
  default:
    brokers:
    - 192.168.56.74:9092
