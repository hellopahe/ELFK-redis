# ELFK_redis
### 1 install mysql-redis on docker-compose
~~~
cd ./mysql-redis
docker-compose up -d
~~~
listening port：6379

### 2 docker-compose ELFK
~~~
chmod -R 777 ./ELFK-redis
~~~
~~~
# modify args in ./.env & filebeat.Dockerfile to change ELFK version.

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
.<img src="https://github.com/hellopahe/ELFK_redis/blob/main/containers.png" width="220" height="85" />
~~~
# 运行期进入redis控制台查询不到条目，关闭logstash，redis中出现条目.
docker exec -it redis /bin/bash
redis-cli
keys *
~~~
.<img src="https://github.com/hellopahe/ELFK_redis/blob/main/redis1.png" width="200" height="45" />

#### logstash.conf
~~~
input {}
filter {
  # If log line contains tab character followed by 'at' then we will tag that entry as stacktrace
  if [message] =~ "\tat" {
    grok {
      match => ["message", "^(\tat)"]
      add_tag => ["stacktrace"]
    }
  }
  grok {
    match => ["message", "%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:severity} \] %{NOTSPACE:threadName} %{NOTSPACE:service} %{JAVACLASS:class} %{GREEDYDATA:msg}"]
  }
  #Parsing out timestamps which are in timestamp field thanks to previous grok section
  date {
    match => [ "timestamp" , "yyyy-MM-dd HH:mm:ss.SSS" ]
  }
}
output {}
~~~
--- [grok debugger](http://grokdebug.herokuapp.com/?#) </br>
--- [logstash files](https://elkguide.elasticsearch.cn/logstash/)
### es/kibana
~~~
# search by Class, Level, threadName, Stacktrace..
# demo
Discover search: [tags.keyword :"stacktrace"]
Discover search: [severity.keyword :"WARN"]
Discover search: [class.keyword :]
~~~
<img src="https://github.com/hellopahe/ELFK_redis/blob/main/kibana_1.png" width="500" height="280" />
<img src="https://github.com/hellopahe/ELFK_redis/blob/main/kibana_2.png" width="500" height="280" />


