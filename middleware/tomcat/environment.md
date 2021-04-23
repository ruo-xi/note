## environment


> OS: 	Linux yu 5.10.9-arch1-1 #1 SMP PREEMPT Tue, 19 Jan 2021 22:06:06 +0000 x86_64 GNU/Linux
>
> jdk:	openjdk 11.0.10-internal 2021-01-19
>
> tomcat-version: 	   tomcat-10.0.4
>
> ide:		idea
>
> ant: 	   Apache Ant(TM) version 1.10.9 compiled on December 22 1969

## comiple steps

### ant

`yay -S ant`

### tomcat

```bash
wget https://github.com/apache/tomcat/archive/refs/tags/10.0.4.zip
unzip 10.0.4.zip
cd tomcat-10.0.4
```

### import into the idea

```bash
# if you want to use idea debug or run the tomcat maybe you can set the enviroment bellow
#   ANT_HOME
#   TOMCAT_BUILD_LIBS default = ~/tomcat-build-libs
# 再idea中生成.project .classpath文件 并下载依赖到TOMCAT_BUILD_LIBS
ant -buildfile build.xml ide-intellij

# build the project
ant 
```

### Debug

#### run tomcat

1. open the dir with idea
2. **check the project structure -> module -> dependencies if is right** 

#### create web app 

1. see the tomcat official website and check the javaee version
2. create project 
3. resolve enpendencies
4. enjoy your coding
5. build the war package
6. push the war package into the webapps directory