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

 *部分metadata字段如"_index", "id"等，无法在logstash前置环节去掉，因此需要到登陆kibana手动设置屏蔽字段，也可直接在kibana设置中直接屏蔽所有多余字段*
1、登陆kibana，左侧导航栏 Stack Management => Advanced Settings </br>
2、在上方的设置搜索框，搜索 "Meta",  定位到metaFields setting， 将输入框中的所有字段删除。 </br>
3、保存设置
4、左侧导航栏，选择Kibana-Index Patterns => 选择Fieldfilters, 依次输入四个关键词并Add， </br> 
log.offset </br>
message.keyword </br>
log.flags* </br>
log.file.path.keyword  </br>
5、上方选择Fields，点击log.file.path后方的编辑按钮，打开set format按钮，选择Static Lookup，输入映射如下图：
https://github.com/hellopahe/ELFK-redis/blob/main/sample/kibana_6.png </br>


*Elastic/Kibana*</br>
http://yourhost:9200/_cat/indices </br>
http://yourhost:5601/ </br>



