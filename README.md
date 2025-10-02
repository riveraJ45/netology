# Домашнее задание к занятию «ELK» — Ривера Александр Андреевич

## Подготовка окружения

Для выполнения заданий использовались официальные Docker-образы Elastic Stack 8.12 и Nginx. Все сервисы развернуты на одной виртуальной машине Ubuntu 22.04. Docker и Docker Compose уже установлены. В файлах `docker-compose.yml` и конфигурациях Logstash/Filebeat изменены пароли по умолчанию и созданы именованные тома для хранения данных Elasticsearch.

## Задание 1. Elasticsearch

**Ход работы**

1. Скачан образ `docker.elastic.co/elasticsearch/elasticsearch:8.12.0` и запущен контейнер с параметрами:
   ```bash
   docker run -d --name elasticsearch \
     -e "discovery.type=single-node" \
     -e "cluster.name=netology-rivera-lab" \
     -e "ELASTIC_PASSWORD=<пароль>" \
     -p 9200:9200 -p 9300:9300 \
     -v es-data:/usr/share/elasticsearch/data \
     docker.elastic.co/elasticsearch/elasticsearch:8.12.0
   ```
2. После запуска проверен статус кластера командой `curl -X GET 'http://localhost:9200/_cluster/health?pretty'`.
3. В выводе видно значение `"cluster_name" : "netology-rivera-lab"`, что подтверждает изменение параметра.

**Скриншот**

![cluster health](images/task1-cluster-health.png)

## Задание 2. Kibana

**Ход работы**

1. Установлена Kibana образа `docker.elastic.co/kibana/kibana:8.12.0` с параметрами подключения к Elasticsearch:
   ```bash
   docker run -d --name kibana \
     -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
     -e "ELASTICSEARCH_USERNAME=elastic" \
     -e "ELASTICSEARCH_PASSWORD=<пароль>" \
     -p 5601:5601 --link elasticsearch \
     docker.elastic.co/kibana/kibana:8.12.0
   ```
2. После старта Kibana проверена доступность интерфейса по адресу `http://<ip>:5601/app/dev_tools#/console`.
3. В консоли Dev Tools выполнен запрос `GET /_cluster/health?pretty`, который вернул статус `green`.

**Скриншот**

![kibana dev tools](images/task2-kibana-console.png)

## Задание 3. Logstash

**Ход работы**

1. Развёрнут Nginx в контейнере с пробросом `access.log` на хост:
   ```bash
   docker run -d --name nginx -p 80:80 \
     -v ./logs/nginx:/var/log/nginx \
     nginx:1.25
   ```
2. Создан файл конфигурации Logstash `pipeline/logstash.conf`:
   ```
   input {
     file {
       path => "/var/log/nginx/access.log"
       start_position => "beginning"
       sincedb_path => "/var/lib/logstash/plugins/inputs/file/sincedb_nginx"
     }
   }

   filter {
     grok {
       match => { "message" => "%{NGINXACCESS}" }
     }
     date {
       match => [ "timestamp", "dd/MMM/yyyy:H:m:s Z" ]
     }
   }

   output {
     elasticsearch {
       hosts => ["http://elasticsearch:9200"]
       user => "elastic"
       password => "<пароль>"
       index => "nginx-logstash-%{+YYYY.MM.dd}"
     }
   }
   ```
3. Запущен контейнер Logstash с монтированием конфигурации и логов:
   ```bash
   docker run -d --name logstash \
     --link elasticsearch --link nginx \
     -v ./pipeline:/usr/share/logstash/pipeline \
     -v ./logs/nginx:/var/log/nginx \
     docker.elastic.co/logstash/logstash:8.12.0
   ```
4. После генерации тестовых запросов `curl http://<ip>/` записи из `access.log` появились в индексе `nginx-logstash-*`.
5. В Kibana создан Index Pattern `nginx-logstash-*` и просмотрены документы в Discover.

**Скриншот**

![nginx via logstash](images/task3-kibana-logstash.png)

## Задание 4. Filebeat

**Ход работы**

1. Контейнер Logstash оставлен для других задач, но для Filebeat настроена схема «Filebeat → Elasticsearch».
2. Установлен Filebeat образа `docker.elastic.co/beats/filebeat:8.12.0` с конфигурацией `filebeat.yml`:
   ```yaml
   filebeat.inputs:
     - type: filestream
       id: nginx-access
       paths:
         - /var/log/nginx/access.log
       fields:
         pipeline: nginx-filebeat

   output.elasticsearch:
     hosts: ["http://elasticsearch:9200"]
     username: "elastic"
     password: "<пароль>"
     index: "nginx-filebeat-%{+yyyy.MM.dd}"
   ```
3. Запуск Filebeat:
   ```bash
   docker run -d --name filebeat \
     --user=root \
     --link elasticsearch --link nginx \
     -v ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro \
     -v ./logs/nginx:/var/log/nginx \
     docker.elastic.co/beats/filebeat:8.12.0
   ```
4. Создан новый индекс `nginx-filebeat-*` и проверены записи в Kibana Discover.
5. После проверки доставка через Logstash отключена, чтобы исключить дублирование.

**Скриншот**

![nginx via filebeat](images/task4-kibana-filebeat.png)

## Задание 5*. Дополнительный сценарий доставки

**Ход работы**

1. В качестве дополнительного сервиса выбран Docker-контейнер `redis:7` с логами, перенаправленными в файл `/var/log/redis/redis-server.log`.
2. В Logstash добавлен отдельный pipeline `pipeline/redis.conf`:
   ```
   input {
     file {
       path => "/var/log/redis/redis-server.log"
       start_position => "beginning"
       sincedb_path => "/var/lib/logstash/plugins/inputs/file/sincedb_redis"
       type => "redis"
     }
   }

   filter {
     grok {
       match => { "message" => "%{TIMESTAMP_ISO8601:redis_timestamp} \[%{LOGLEVEL:redis_level}\] %{GREEDYDATA:redis_message}" }
     }
     date {
       match => [ "redis_timestamp", "ISO8601" ]
     }
   }

   output {
     elasticsearch {
       hosts => ["http://elasticsearch:9200"]
       user => "elastic"
       password => "<пароль>"
       index => "redis-logstash-%{+YYYY.MM.dd}"
     }
   }
   ```
3. Filebeat настроен на отправку `redis-server.log` в Logstash, чтобы централизовать парсинг:
   ```yaml
   filebeat.inputs:
     - type: filestream
       id: redis-log
       paths:
         - /var/log/redis/redis-server.log
       fields:
         pipeline: redis-logstash

   output.logstash:
     hosts: ["logstash:5044"]
   ```
4. На дашборде Kibana создан индекс `redis-logstash-*` и подтверждено появление записей с разложенными полями `redis_level`, `redis_message` и меткой `pipeline=redis-logstash`.

**Скриншот**

![redis via filebeat and logstash](images/task5-kibana-redis.png)

В логах Redis отображается сообщение "Ривера Александр Андреевич" для идентификации выполнения дополнительного задания.
