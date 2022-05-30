# ELFK_redis
### STEP 1 install mysql-redis on docker-compose
~~~
cd ./ELFK-redis/mysql-redis
docker-compose up -d
~~~
listening port：6379
*make sure remember the psw and host for a running redis.*
*default psw 123456*

### STEP 2 install ELK
~~~
# Notes:
# modify ELFK verison in ./ELFK-redis/.env
# modify redis host & psw in ./ELFK-redis/.env

cd ./ELFK-redis/
docker-compose up -d
~~~
### STEP 3 install Filebeat
~~~
# Notes:
# modify args in ./filebeat_compose/.env

cd filebeat_compose/
docker-compose up -d
~~~

### STEP 4 navigate to ES/Kibana to config
navigate to ElasticSearch to check indices received from Logstash </br>
http://yourhost:9200/_cat/indices </br>

navigate to Kibana to setup index and discover data </br>
http://yourhost:5601/ </br>



# Some Details on ELFK-redis

### *data pipeline: filebeat => redis => logstash => es*
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
# filebeat输入部分
- type: log
  enabled: true

# 这里主要设置log文件路径，前面已经做了本地logs文件夹到/var/log/的映射，所以这里只需要更改后缀的文件名。
  paths:

    - /var/log/sample.log
    - /var/log/account.log
    - /var/log/config.log
    - /var/log/gateway.log
    - /var/log/investment.log
    - /var/log/thirdparty.log

# filebeat的多行采集，取决于本地日志文件结构，由于日志文件开头都是以yyyy-mm-dd开头，因此将不是这个类型开头的行内容，都加到到前一行（例如stacktrack）
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
  multiline.negate: true
  multiline.match: after


# 输出到redis，设置一些redis参数，key为存入redis的键，db为默认数据库编号，
# 此处一般无特殊要求。
output.redis:
        hosts: ["${redishost_f}:6379"]
        password: ${redispsw_f}
        key: javalog
        db: 0
        filetype: "list"


~~~

#### logstash.conf
~~~
input {
  redis {
# 输入部分，这里是从redis中取数据，此处字段需要和filebeat中配置一致。

    host => "172.20.91.116"
    port => 6379
    password => "123456"
    key => "javalog"
    db => 0
    data_type => "list"
    codec => json{}
  }
}
filter {
# logstash自带的过滤器，这里首先给stacktrack类型日志加上标志字段。然后用grok表达式匹配日志结构，拆分日志信息输出到数据库。

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



# 加上时间戳字段。
  date {
    match => [ "timestamp" , "yyyy-MM-dd HH:mm:ss.SSS" ]
  }

}

output {
# 把数据输出到数据库。

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


