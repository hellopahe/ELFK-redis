# ELFK_redis
### 1 install mysql-redis on docker-compose
~~~
cd ./ELFK-redis/mysql-redis
docker-compose up -d
~~~
listening port：6379
*make sure remember the psw and host for a running redis.*

### 2 docker-compose ELFK
~~~
chmod -R 777 ./ELFK-redis
~~~
~~~
# modify ELFK verison in ./ELFK-redis/.env
# modify redis host & psw in ./ELFK-redis/.env

docker-compose up -d
~~~
~~~
# config your filebeat.yml & logstash.conf to make sure they are connected to [redishost:6379]
vim ./filebeat/filebeat.yml
vim ./logstash/logstash.conf

# change redis pwd for root
vim ./mysql-redis/docker-compose.yml
~~~
### 3 *data pipeline: filebeat => redis => logstash => es*
~~~
docker ps -a
~~~
.<img src="https://github.com/hellopahe/ELFK-redis/blob/main/sample/containers.png" width="220" height="85" />
~~~
# 运行期进入redis控制台查询不到条目，关闭logstash，redis中出现条目.
docker exec -it redis /bin/bash
redis-cli
keys *
~~~
.<img src="https://github.com/hellopahe/ELFK-redis/blob/main/sample/redis1.png" width="200" height="45" />
#### filebeat.yml
~~~
filebeat.inputs:
- type: log
  enabled: true

  paths:
    - /var/log/sample.log
    
  # springboot logs are started with yyyy-MM-dd, use multiline pattern to publish a integral stacktrace.
  
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
  multiline.negate: true
  multiline.match: after

output.redis:
        hosts: ["172.20.91.116:6379"]
        password: 123456
        key: javalog
        db: 0
        filetype: "list"

~~~

#### logstash.conf
~~~
input { codec => json{} }

filter {
  #If log line contains tab character followed by 'at' then we will tag that entry as stacktrace
  if [message] =~ "\tat" {
    grok {
      match => ["message", "^(\tat)"]
      add_tag => ["stacktrace"]
    }
  }

  # log format
  grok {
    match => ["message",
              "%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:severity} \] %{NOTSPACE:threadName} %{NOTSPACE:service} %{JAVACLASS:class} %{GREEDYDATA:msg}",
              "message",
              "%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:severity}\] %{NOTSPACE:threadName} %{NOTSPACE:service} %{JAVACLASS:class} %{GREEDYDATA:msg}"]
  }

  #Parsing out timestamps
  date {
    match => [ "timestamp" , "yyyy-MM-dd HH:mm:ss.SSS" ]
  }
}

output {
  elasticsearch {
    hosts => "elasticsearch:9200"
    index => "v3.0-%{+YYYY.MM.dd}"
  }
}
~~~
~~~
--- ElasticSearch Config http://host:9200/_cat/
~~~

--- [grok debugger](http://grokdebug.herokuapp.com/?#) </br>
--- [logstash files](https://elkguide.elasticsearch.cn/logstash/)

)

### 4 es/kibana
~~~
# search by Class, Level, threadName, Stacktrace..
# demo
Discover search: [tags.keyword :"stacktrace"]
Discover search: [severity.keyword :"WARN"]
Discover search: [class.keyword :]
~~~
#### searching by Class
<img src="https://github.com/hellopahe/ELFK-redis/blob/main/sample/kibana_1.png" width="500" height="280" />

#### stacktrace
<img src="https://github.com/hellopahe/ELFK-redis/blob/main/sample/kibana_3.png" width="1000" height="600" />

#### visualize
<img src="https://github.com/hellopahe/ELFK-redis/blob/main/sample/kibana_4.png" width="500" height="280" />
<img src="https://github.com/hellopahe/ELFK-redis/blob/main/sample/kibana_5.png" width="500" height="280" />


