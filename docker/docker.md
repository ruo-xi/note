# docker

## 架构

![2.png](https://cdn.nlark.com/yuque/0/2020/png/365147/1587017629001-91afd66d-0b2b-4836-92f9-8614fcbc691b.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_14%2Ctext_6bKB54-t5a2m6Zmi5Ye65ZOB%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

### Host(Docker 宿主机)

安装了Docker程序，并运行了Docker daemon的主机。

#### Docker daemon(Docker 守护进程)：

运行在宿主机上，Docker守护进程，用户通过Docker client(Docker命令)与Docker daemon交互。

#### Images(镜像)：

将软件环境打包好的模板，用来创建容器的，一个镜像可以创建多个容器。

镜像分层结构：



![docker image.jpg](https://cdn.nlark.com/yuque/0/2020/jpeg/365147/1587017642482-36494fbb-d25f-426c-b76c-2a9b27c415f9.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_14%2Ctext_6bKB54-t5a2m6Zmi5Ye65ZOB%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



位于下层的镜像称为父镜像(Parent Image)，最底层的称为基础镜像(Base Image)。

最上层为“可读写”层，其下的均为“只读”层。

AUFS:

- advanced multi-layered unification filesystem：高级多层统一文件系统

- 用于为Linux文件系统实现“联合挂载”

- AUFS是之前的UnionFS的重新实现

- Docker最初使用AUFS作为容器文件系统层

- AUFS的竞争产品是overlayFS，从3.18开始被合并入Linux内核

- Docker的分层镜像，除了AUFS，Docker还支持btrfs，devicemapper和vfs等

#### Containers(容器)：

Docker的运行组件，启动一个镜像就是一个容器，容器与容器之间相互隔离，并且互不影响。

### Docker Client(Docker 客户端)

Docker命令行工具，用户是用Docker Client与Docker daemon进行通信并返回结果给用户。也可以使用其他工具通过[Docker Api ](https://docs.docker.com/develop/sdk/)与Docker daemon通信。

### Registry(仓库服务注册)

经常会和仓库(Repository)混为一谈，实际上Registry上可以有多个仓库，每个仓库可以看成是一个用户，一个用户的仓库放了多个镜像。仓库分为了公开仓库(Public Repository)和私有仓库(Private Repository)，最大的公开仓库是官方的[Docker Hub](https://hub.docker.com/)，国内也有如阿里云、时速云等，可以给国内用户提供稳定快速的服务。用户也可以在本地网络内创建一个私有仓库。当用户创建了自己的镜像之后就可以使用 push 命令将它上传到公有或者私有仓库，这样下次在另外一台机器上使用这个镜像时候，只需要从仓库上 pull 下来就可以了。

## Docker常用操作

### 镜像常用操作



查找镜像：



```
docker search 关键词
#搜索docker hub网站镜像的详细信息
```



下载镜像：



```
docker pull 镜像名:TAG
# Tag表示版本，有些镜像的版本显示latest，为最新版本
```



查看镜像：



```
docker images
# 查看本地所有镜像
```



删除镜像：



```
docker rmi -f 镜像ID或者镜像名:TAG
# 删除指定本地镜像
# -f 表示强制删除
```



获取元信息：



```
docker inspect 镜像ID或者镜像名:TAG
# 获取镜像的元信息，详细信息
```



### 容器常用操作



运行：

```
docker run --name 容器名 -i -t -p 主机端口:容器端口 -d -v 主机目录:容器目录:ro 镜像ID或镜像名:TAG
# --name 指定容器名，可自定义，不指定自动命名
# -i 以交互模式运行容器
# -t 分配一个伪终端，即命令行，通常-it组合来使用
# -p 指定映射端口，讲主机端口映射到容器内的端口
# -d 后台运行容器
# -v 指定挂载主机目录到容器目录，默认为rw读写模式，ro表示只读
```



容器列表：

```
docker ps -a -q
# docker ps查看正在运行的容器
# -a 查看所有容器（运行中、未运行）
# -q 只查看容器的ID
```



启动容器：

```
docker start 容器ID或容器名
```



停止容器：

```
docker stop 容器ID或容器名
```



删除容器：

```
docker rm -f 容器ID或容器名
# -f 表示强制删除
```



查看日志：

```
docker logs 容器ID或容器名
```



进入正在运行容器：

```
docker exec -it 容器ID或者容器名 /bin/bash
# 进入正在运行的容器并且开启交互模式终端
# /bin/bash是固有写法，作用是因为docker后台必须运行一个进程，否则容器就会退出，在这里表示启动容器后启动bash。
# 也可以用docker exec在运行中的容器执行命令
```



拷贝文件：

```
docker cp 主机文件路径 容器ID或容器名:容器路径 #主机中文件拷贝到容器中
docker cp 容器ID或容器名:容器路径 主机文件路径 #容器中文件拷贝到主机中
```



获取容器元信息：

```
docker inspect 容器ID或容器名
```