# Enviroment

> 操作系统:  Linux yu 5.12.10-arch1-1 #1 SMP PREEMPT Thu, 10 Jun 2021 16:34:50 +0000 x86_64 GNU/Linux
>
> Version: Zookeeper-release 3.7.0
>
> 源码阅读工具:  Idea 2021.1
>
> jdk版本:  openjdk 11.0.11 2021-04-20

## Compile

1. download source form [Github](openjdk 11.0.11 2021-04-20)
2. compile the source with idea 
   1. compile the zookeeper-jute module
   2. compile the zookeeper-server module
   3. debug the source code (refer to the ZkServer.sh) with the args (config file path)

## Error

1.  log error 

   ```
   log4j:WARN No appenders could be found for logger (org.apache.zookeeper.server.quorum.QuorumPeerConfig).
   log4j:WARN Please initialize the log4j system properly.
   log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
   ```

   add log4j.properties to resource and mark the dir as resources root

2.  ClassNotFoundException Error

   zookeeper-server.pom.xml file

   comment the scope of the follow dependencies

   * snappy-java
   * metrics-core
   * jetty-*
   * jackson-databind

