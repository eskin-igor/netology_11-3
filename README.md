# netology_11-3
# Домашнее задание к занятию «ELK»

## Задание 1. Elasticsearch

Установите и запустите Elasticsearch, после чего поменяйте параметр cluster_name на случайный.  
Приведите скриншот команды 'curl -X GET 'localhost:9200/_cluster/health?pretty', сделанной на сервере с установленным Elasticsearch.  
Где будет виден нестандартный cluster_name.

## Решение 1. Elasticsearch

Установка elasticsearch.
```
sudo apt install default-jdk -y
sudo apt-get install apt-transport-https -y
sudo apt install ./elasticsearch-8.12.2-amd64.deb -y
sudo apt install –f
```
Редактирование конфигурационного файла elasticsearch.  
sudo nano /etc/elasticsearch/elasticsearch.yml
```
cluster.name: netology-ES-cluster 
```
Перечитываем  конфиги systemd
```
sudo systemctl daemon-reload
```
Запуск службы elasticsearch.
```
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
sudo systemctl status elasticsearch.service
```
Проверяем, что сервер запустился.
```
curl -X GET 'localhost:9200/_cluster/health?pretty'
```
[Результат запроса](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-1.JPG)

Если получаете ответ «curl: (52) Empty reply from server», замените значение true на false.  
sudo nano  /etc/elasticsearch/elasticsearch.yml
```
xpack.security.enabled: false
```
Перезапустите службу.  
sudo systemctl restart elasticsearch.service

## Задание 2. Kibana

Установите и запустите Kibana.  
Приведите скриншот интерфейса Kibana на странице http://<ip вашего сервера>:5601/app/dev_tools#/console, где будет выполнен запрос GET /_cluster/health?pretty.

## Решение 2. Kibana

Установка Kibana.  
sudo apt install ./kibana-8.12.2-amd64.deb

Настройка.  
sudo nano  /etc/kibana/kibana.yml
```
# =================== System: Kibana Server ===================
# Kibana is served by a back end server. This setting specifies the port to use.
server.port: 5601

# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
server.host: "0.0.0.0"

# =================== System: Elasticsearch ===================
# The URLs of the Elasticsearch instances to use for all your queries.
elasticsearch.hosts: ["http://localhost:9200"]
```
Перечитать  конфиги systemd и запустить службу.  
```
sudo systemctl daemon-reload  
sudo systemctl enable kibana.service  
sudo systemctl start kibana.service  
sudo systemctl status kibana.service  
```
Подлючение.  
```
http://localhost:5601/app/dev_tools#/console
```
[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-2-1.JPG) 

Выполнение запроса
```
GET /_cluster/health?pretty
```
[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-2-2.JPG)

## Задание 3. Logstash

Установите и запустите Logstash и Nginx.  
С помощью Logstash отправьте access-лог Nginx в Elasticsearch.  
Приведите скриншот интерфейса Kibana, на котором видны логи Nginx.

## Решение 3. Logstash

Установка ngnx.  
```
sudo apt install nginx
```
Установка logstash.  
```
sudo apt install ./logstash-8.12.2-amd64.deb 
```
Создать файл конфигурации по адресу /etc/logstash/conf.d/logstash.conf.

Далее настроить поставку access-лога nginx в elasticsearch:  
```
# Читаем файл
input {
  file {
    path => ["/var/log/nginx/access.log"]
    start_position => "beginning"
  }
}
# Извлекаем данные из событий
filter {
  grok {
    match => { "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\]\[%{DATA:severity}%{SPACE}\]\[%{DATA:source}%{SPACE}\]%{SPACE}%{GREEDYDATA:message}" }
    overwrite => [ "message" ]
  }
}
# Сохраняем все в Elasticsearch
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-logs-%{+YYYY.MM}"
  }
}
```
Запуск службу logstash.  
```
sudo systemctl daemon-reload  
sudo systemctl enable logstash.service   
sudo systemctl start logstash.service  
sudo systemctl status logstash.service
```
Создать данные для просмотра в kibana.  
Маршрут: Management > stack management	> kibana > data view > Create data view

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-3-0.PNG)

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-3-1.PNG)

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-3-2.PNG)

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-3-3.PNG)

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-3-4.PNG)

## Задание 4. Filebeat.

Установите и запустите Filebeat.  
Переключите поставку логов Nginx с Logstash на Filebeat.  
Приведите скриншот интерфейса Kibana, на котором видны логи Nginx, которые были отправлены через Filebeat.

##  Решение 4. Filebeat.

### Установка и настройка filebeat.  
```
sudo apt install ./filebeat-8.12.2-amd64.deb  
```
Разрешаем обработку логов Nginx:  
```
mv /etc/filebeat/modules.d/nginx.yml.disabled /etc/filebeat/modules.d/nginx.yml
```
Приводим содержимое файла /etc/filebeat/modules.d/nginx.yml к виду:
```
- module: nginx
  # Access logs
  access:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    var.paths: ["/var/log/nginx/access.log"]

  # Error logs
  error:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    var.paths: ["/var/log/nginx/error.log"]

  # Ingress-nginx controller logs. This is disabled by default. It could be used in Kubernetes environments to parse ingress-nginx logs
  ingress_controller:
    enabled: false
```
Файл /etc/filebeat/filebeat.yml приводим к виду:
[filebeat.yml](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-4_filebeat/filebeat.yml)

Убеждаемся, что в конфигурационном файле нет ошибок:
```  
filebeat test config -c /etc/filebeat/filebeat.yml
```
Получаем вывод 
```
Config OK
```
Запускаем службу filebeat
```
sudo systemctl enable filebeat
sudo systemctl start filebeat
```
### Установка и настройка Logstash

Для удобства отладки предлагаю разделить конфигурационный файл logstash на три части.  
Создаём файл /etc/logstash/conf.d/input-beats.conf.
```
input {
  beats {
    port => 5044
  }
}
```
Создаём файл /etc/logstash/conf.d/output-elasticsearch.conf.
```
output {
  elasticsearch {
    hosts => ["localhost:9200"]
#    manage-template => false
    index => "nginx-filebeat-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```
Создаём файл /etc/logstash/conf.d/filter-nginx.conf.
```
filter {
  if [event][dataset] == "nginx.access" {
    grok {
      match => [ "message" , "%{IPORHOST:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] \"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\" %{NUMBER:response} (?:%{NUMBER:bytes}|-) %{QS:referrer} %{QS:user_agent}"]
      overwrite => [ "message" ]
    }
    mutate {
      convert => ["response", "integer"]
      convert => ["bytes", "integer"]
      convert => ["responsetime", "float"]
    }
    date {
      match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
      remove_field => [ "timestamp" ]
    }
  }
}
```
Запускаем службу logstash.  
```
sudo systemctl enable logstash
sudo systemctl start logstash
```
Проверяем, что logstash слушает порт 5044.
```
sudo netstat -tulpn | grep 5044
```

Смотрим, что получает kibana.

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-4-1.PNG)

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-4-2.PNG)

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-4-3.PNG)

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-4-4.PNG)

[](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-4-5.PNG)

### Для отладки удобно использовать следующие журналы.

1. Журнал Filebeat на сервере с nginx:
```
sudo journalctl -ru filebeat.service
```
2. Журнал Logstash на сервере Logstash:
```
sudo journalctl -ru logstash.service
```
Также в конфигурационный файл Logstash можно добавить вывод в консоль:
```
stdout { codec => rubydebug }
```
[Пример вывода в журнал](https://github.com/eskin-igor/netology_11-3/blob/main/11-3/11-3-4-6.PNG)

3. Журналы Elasticearch на узлах:
```
sudo journalctl -ru elasticsearch.service
```
