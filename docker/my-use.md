# User



```bash
# nginx
docker run --name=nginx --network=my-net -v ~/docker/conf/nginx/nginx.conf:/etc/nginx/nginx.conf  -p 80:80 -d nginx:latest

# mongo
docker run --name=mongo --network=my-net -e MONGO_INITDB_ROOT_USERNAME=yu -e MONGO_INITDB_ROOT_PASSWORD=cao19981128 -v ~/docker/data/mongo:/data/db -p 27017:27017 -d mongo:latest

# mysql
docker run --name=mysql -e MYSQL_ROOT_PASSWORD=cao19981128 -v ~/docker/data/mysql:/var/lib/mysql -v ~/docker/conf/mysql:/etc/mysql  -d  --network my-net  -p 3306:3306  mysql:latest

# redis
docker run  --name=redis --network=my-net -p 6379:6379  -v ~/docker/data/redis:/data -v ~/docker/conf/redis:/usr/local/etc/redis -d   redis redis-server /usr/local/etc/redis/redis.conf

#nacos
docker run --name=nacos1 --network=my-net -ePREFER_HOST_MODE=hostname -eSPRING_DATASOURCE_PLATFORM=mysql -e NACOS_SERVERS=[nacos1:8848,nacos2:8848,nacos3:8848] -e MYSQL_MASTER_SERVICE_HOST=mysql -e MYSQL_MASTER_SERVICE_PORT=3306 -e MYSQL_MASTER_SERVICE_DB_NAME=nacos-test -e MYSQL_MASTER_SERVICE_USER=root -e MYSQL_MASTER_SERVICE_PASSWORD=cao19981128 -e JVM_XMS=480M -e JVM_XMX=480M -e JVM_XMN=240M nacos/nacos-server:latest

#JVM_MS:   #-XX:MetaspaceSize default :128m
#JVM_MMS:  #-XX:MaxMetaspaceSize default :320m
#NACOS_DEBUG: n #是否开启远程debug，y/n，默认n
#TOMCAT_ACCESSLOG_ENABLED: true #是否开始tomcat访问日志的记录，默认false
# mysql NACOS_SERVER_IP: 192.168.87.133 
# MYSQL_SLAVE_SERVICE_HOST: mysql
# MYSQL_SLAVE_SERVICE_PORT: 
```

