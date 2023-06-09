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

# Устанавливаем kafkactl
sudo snap install kafkactl

# Создаем каталог и скачиваем docker-compose файл
mkdir kafka
cd kafka/
wget https://raw.githubusercontent.com/bitnami/containers/main/bitnami/kafka/docker-compose.yml

# Добавляем переменную для доступа к экземпляру в докере
nano docker-compose.yml
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092

# Запускаем и смотрим логи
docker compose up -d
docker compose logs -f

# Проверяем состояние брокера и данных топика collector
kafkactl list brokers
kafkactl get topics
kafkactl describe topic collector
kafkactl consume collector --from-beginning


```


## Настройка Grafana

1. ClickHouse 

⚙️-> Data Sources -> ClickHouse

Server address: clickhouse
Server port:    9000

Save & test

Dashboards -> Import

2. Altinity  

URL:    http://192.168.56.74:8123
Access: Server (default)

Save & test






---

### Дополнение

```shell
# Инспектирование точек монтирования
docker inspect broker|jq '.[].Mounts[] | .Type ,.Destination'
docker inspect broker|jq '.[].Mounts '
```

Для каталога kafka возможно появление ошибки из-за проблем с правами

volumes:
  - ./zk-single-kafka-single/kafka1/data:/var/lib/kafka/data

Решением може стать измнение прав каталога
Please make sure container has write permission on the host directory:
chown -R 1000:1000  ./zk-single-kafka-single/kafka1/data 
