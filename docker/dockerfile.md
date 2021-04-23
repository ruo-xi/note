# 构建镜像

两种方式：

- 更新镜像：使用`docker commit`命令

- 构建镜像：使用`docker build`命令，需要创建Dockerfile文件

## 更新镜像

先使用基础镜像创建一个容器，然后对容器内容进行更改，然后使用`docker commit`命令提交为一个新的镜像（以tomcat为例）。

1.根据基础镜像，创建容器

```
docker run --name mytomcat -p 80:8080 -d tomcat
```

2.修改容器内容

```
docker exec -it mytomcat /bin/bash
cd webapps/ROOT
rm -f index.jsp
echo hello world > index.html
exit
```

3.提交为新镜像

```
docker commit -m="描述消息" -a="作者" 容器ID或容器名 镜像名:TAG
# 例:
# docker commit -m="修改了首页" -a="yu" mytomcat huaan/tomcat:v1.0
```

4.使用新镜像运行容器

```
docker run --name tom -p 8080:8080 -d huaan/tomcat:v1.0
```

## 使用Dockerfile构建镜像

### 什么是Dockerfile？

Dockerfile is nothing but the source code for building Docker images

- Docker can build images automatically by reading the instructions from a Dockerfile

- A Dockerfile is a **text document** that contains all the commands a user could call on the command line to assemble an image

- - Using **docker build** users can create an automated build that executes several command-line instructions in succession

![dockerfile.jpg](https://cdn.nlark.com/yuque/0/2020/jpeg/365147/1587017706307-8195b999-d90b-467a-a24f-15969ee89493.jpeg)

### Dockerfile格式

- Format：

- - \#Comment

- - INSTRUCTION arguments

- The instruction is not case-sensitive

- - However,convention is for them to be UPPERCASE to distinguish them from arguments more easily

- Docker runs instructions in a Dockerfile in order

- The first instruction must be 'FROM' in order to specify the Base Image from which you are building

### 使用Dockerfile构建SpringBoot应用镜像

一、准备

1.把你的springboot项目打包成可执行jar包

2.把jar包上传到Linux服务器

二、构建

1.在jar包路径下创建Dockerfile文件`vi Dockerfile`

```
# 指定基础镜像，本地没有会从dockerHub pull下来
FROM java:8
#作者
MAINTAINER huaan
# 把可执行jar包复制到基础镜像的根目录下
ADD luban.jar /luban.jar
# 镜像要暴露的端口，如要使用端口，在执行docker run命令时使用-p生效
EXPOSE 80
# 在镜像运行为容器后执行的命令
ENTRYPOINT ["java","-jar","/luban.jar"]
```



2.使用`docker build`命令构建镜像，基本语法

```
docker build -t huaan/mypro:v1 .
# -f指定Dockerfile文件的路径
# -t指定镜像名字和TAG
# .指当前目录，这里实际上需要一个上下文路径
```

三、运行

运行自己的SpringBoot镜像

```
docker run --name pro -p 80:80 -d 镜像名:TAG
```

### 指令

#### FROM

```dockerfile
FROM [--platform=<platform>] <image> [AS <name>]
```

```
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
```

```
FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```

example:

```dockerfile
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
```



#### RUN

```
RUN <command>
RUN ["executable", "param1", "param2"]
```

example:

```dockerfile
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'

RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'

RUN ["/bin/bash", "-c", "echo hello"]
```

#### CMD

cmd命令的三种形式

- `CMD ["executable","param1","param2"]` (*exec* form, this is the preferred form)
- `CMD ["param1","param2"]` (as *default parameters to ENTRYPOINT*)
- `CMD command param1 param2` (*shell* form)

docker file中只能有一个CMD指令,如果有多个,那只有最后一个才会有效.



**The main purpose of a `CMD` is to provide defaults for an executing container.** These defaults can include an executable, or they can omit the executable, in which case you must specify an `ENTRYPOINT` instruction as well.

If `CMD` is used to provide default arguments for the `ENTRYPOINT` instruction, both the `CMD` and `ENTRYPOINT` instructions should be specified with the JSON array format.

#### LABEL

#### MAINTAINER

#### EXPOSE

#### ENV

#### ADD

#### COPY

#### ENTRYPOINT

#### VOLUME

#### USER

#### WORKDIR

#### ARG

#### ONBUILD

#### STOPSIGNAL

#### HEALTHCHECK

#### SHELL