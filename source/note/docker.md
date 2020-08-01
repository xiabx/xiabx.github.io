# 第一章：docker入门

docker run 镜像   运行容器

创建、启动、删除容器：docker create start stop kill rm

在文件夹中创建名为Dockerfile的文件，然后执行docker build 就可以自己生成镜像，可以通过docker images查看

链接两个容器，使用--link,比如一个服务要连接mysql这样，` docker run --name wordpress --link mysqlwp:mysql -p 80:80 -d wordpress`

-p指定端口映射，-d后台运行，进入后台运行的容器docker exec 容器名 命令，-ti保持bash窗口

在容器和宿主机之间复制文件，docker cp

docker ps查看运行中的容器，docker ps -a查看所有容器，docker images 查看所有镜像

# 第二章：镜像

**通过交互式方式创建镜像**

可以通过docker run -it imageName /bin/bash 运行镜像，产生新的容器，然后对容器进行修改，之后可以通过`docker commit 容器id 新镜像名`产生新的镜像。

可以通过`docker diff 容器id`查看容器相对于镜像所作的更改。

**将镜像或容器导出到文件系统**

将容器或镜像导出，通过`docker export 容器id 保存的文件名`导出容器为一个压缩包，通过`docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]] `将导出的容器镜像导入docker。通过`docker save`将镜像文件导出到文件系统，`docker load`将导出的镜像文件重新导入到docker。

**通过Dockerfile创建镜像**

通过创建名为Dockerfile的文件，然后执行`docker build [OPTIONS] PATH | URL | -`根据dockerfile文件内容生成镜像，使用docker buld -t 来指定生成镜像的name与tag。

```dockerfile
FROM ubuntu:15.10

CMD ["/bin/echo"]
```

CMD与ENTRYPOINT指令都可以指定镜像运行时需要执行的命令。不同之处在于CMD可以通过docker run后面的指令来覆盖，而ENTRYPOINT只能通过--entrypoint 来覆盖。

```dockerfile
FROM ubuntu:14.04

#在构建时候执行的命令
RUN apt-get update
RUN apt-get install -y python
RUN apt-get install -y python-pip
RUN apt-get clean all
RUN pip install flask

#将本地文件系统中的hello.py文件复制到/tmp/hello.py
ADD hello.py /tmp/hello.py 

#暴露5000端口
EXPOSE 5000

#镜像运行时执行的命令
CMD ["python","/tmp/hello.py"]
```

WORKDIR：设置镜像工作根目录，在WORKDIR之后的命令都在此目录下执行。RUN：构建镜像时执行的指令。ADD、COPY：将本地文件系统中文件复制到镜像中，ADD功能更强大，比如复制压缩文件后自动解压。EXPOSE：暴露端口号。CMD：镜像运行时执行的命令。

docker run -P 会将容器暴露的端口映射到宿主机的一个随机端口上。可以通过docker ps查看。

在编写Dockerfile时一些最佳实践：

1. 一个容器最好只运行一个进程，为了更好的解耦
2. 不应该修改容器的内容，因为容器都是临时的。如果要修改东西应该从基础镜像修改。如果需要运行时配置或数据保存时，可以通过docker volume进行管理。
3. 在镜像构建过程，docker会将dockerfile所在文件夹下的文件复制到构建环境中。使用.dockerignore文件可以排除文件或文件夹。
4. 尽量减少RUN指令，这样可以减少镜像层。因为每运行一次RUN镜像就增加一层，减少层数可以减小镜像大小。

Dockerfile中多个FROM是为了环境分离，比如编译环境和运行环境，可以将编译环境中产生的文件复制到运行环境中：https://blog.csdn.net/Michaelwubo/article/details/91872076

**docker tag**

使用docker tag命令为镜像重新打标签与重命名

**docker hub**

docker hub是一个镜像仓库，将一个本地镜像推送到docker hub需要以下几步：

1. 使用docker login命令登录
2. 使用docker hub上的用户名为已有镜像打标签，格式为`docker hub用户名/镜像名：tag`，命令为docker tag
3. 将打好标签的镜像推送到docker hub，使用命令docker push

使用`dcoker search 镜像名`可以在docker hub中搜索镜像

**ONBUILD指令**

一个镜像的docker file中使用了ONBUILD指令，便是如果另一个镜像继承自该镜像，即在子镜像的dockerfile中FROM改镜像，则触发父镜像定义的ONBUILD指令，ONBUILD指令相当于定义一个触发器。

# 第三章：Docker 网络

**查看容器的ip地址**

第一种方式：使用inspect命令，`docker inspect --format '{ { .NetworkSettings.IPAddress } }' 容器id或容器名`。

第二种方式：使用docker exec命令进入容器，然后` cat /etc/hosts`

**容器端口映射**

可以将容器的端口映射到宿主机的端口上，如果Dockerfile文件中通过EXPOSE暴露了端口，则可以在docker run时使用-P来动态一个端口进行映射，如果没有EXPOSE端口，则使用`-p 端口号`来动态映射，也可以指定宿主机端口进行映射，如：`-p 8080:80`将容器的80端口映射到宿主机的8080端口。

# 第五章：k8s

编排系统帮助用户将一组主机（也称为节点）视为一个统一、可编程、可靠 的集群。这个集群可以当作一台大型计算机来使用。

Docker 非常适合在单台主机上运行容器，Kubernetes 则致力于解决传统的多节点通信和规 模扩展所带来的挑战。

# 第九章：监控容器

**docker inspect 查看容器运行详情**

你想获得关于某个容器的详细信息，比如，这个容器是什么时候创建的，运行的是什么命 令，都有哪些端口映射，容器的 IP 地址是什么，等等。用法：docker inspect 容器id/容器名。

可以通过-f参数指定输出的内容，如`docker inspect -f '{ {.NetworkSettings.IPAddress} }'  87`输出容器的ip。

**docker stats查看容器运行时信息**

docker stats命令可以查看容器运行时的cpu、内存等资源使用情况，用法`docker stats 容器id`

**docker logs 查看容器日志**

当一个容器以-d形式运行，可以通过docker logs 容器id 来查看日志。-f参数表示持续获取日志。





