# Docker

此文章学习于[Docker快速入门](https://docker.easydoc.net/)，以及[🐳Docker概念，工作流和实践 - 入门必懂]( https://www.bilibili.com/video/BV1MR4y1Q738/?share_source=copy_web&vd_source=69ff2ea2830c328716375b4cad874563)

## 简介

产生背景：一般情况下，我们会有很多项目。当需要部署上线时，需要在我们的服务器上安装相应的环境，比如Java项目需要jdk，Python项目可能需要下载一些第三方库。相当于一个大房间没有分区，发现没有床就得买床，缺少桌子了就要再买桌子。而这样每次都在服务器上安装既难以管理，又浪费存储资源。因此Docker诞生了，它相当于将每个项目独立，只需要安装这个项目需要的环境。如果房间里只需要床而不需要桌子，那么就安装床，这样保证了每个项目的独立与轻便。同时，还保证了不同系统环境下运行环境的一致性。

### 小知识

- 打包：将项目以及对应的第三方库打包成一个安装包
- 分发：将安装包放到网上供人下载
- 部署：使用安装包运行项目



## docker三大概念

1. 容器：docker的核心概念就是将不同项目所用的环境与依赖装到不同容器中，实现资源隔离。
2. 镜像文件：将特定操作系统、应用版本、依赖写入镜像文件
3. 镜像：相当于为镜像文件定型，实际上还没有运行

![image-20250630100309794](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\image-20250630100309794.png)



## 实际上手

1. [写镜像文件]根据项目所用的系统，软件版本等写镜像文件。系统建议在docker-hub仓库中获取，大部分情况下，我们还需要将项目文件路径也写进镜像中。

2. [将镜像文件构造为镜像]使用build构造镜像（注意后面`.`不要缺少，是相对路径）
   
   ```bash
   docker bulid -t [project_name] .
   ```
   
3. [将镜像运行，分配容器]运行（`-d`为后台，`-p`为设置端口，使用端口映射）
   
   ```bash
   docker run -d -p [主机端口]:[容器端口] --name [project_name] [镜像名]
   ```
   
4. 检查状态（此处能看到正在运行的docker服务）
   ```
   docker ps
   ```

5. 停止某个docker服务
   ```bash
   docker stop id
   ```

其他一些常用命令
```
docker ps -a		# 将所有服务都呈现(包含停止的)

docker rm -f [镜像名]	# 删除镜像
docker rmi [容器名]	# 删除容器
```

这样我们就可以完成基础的环境搭建啦！但很快会发现若代码文件发生变化，已经运行的docker无法进行热加载（即无法实时更新）。同时，如果我们在容器中放了数据，容器一旦停止就会丢失。所以又有一个解决方案：目录挂载。

## 目录挂载

```bash
# 创建并挂载具名卷(my-volume指存在容器中的卷名称，/container/path指容器路径)
docker run -v my-volume:/container/path my-image
```

使用volume方式进行挂载[适合生产环境]：`-d`表示后台运行，`-v` 表示volume，能够将本地文件夹和容器文件夹绑定。
（`volume` 由容器创建和管理，将数据放在宿主机上，所以删除容器不会丢失，官方推荐，更高效，Linux 文件系统，适合存储数据库数据。可挂到多个容器上）
删除容器中的数据卷：

```bash
docker rm -fv [容器卷名]
```



其余的挂载方式还有：

- `bind mount`：[适合开发环境]宿主机目录映射到容器内，适合挂代码目录与配置文件，以便于挂到多个容器

  ```bash
  # 将主机的 /host/path 挂载到容器的 /container/path
  docker run -v /host/path:/container/path my-image
  ```

- `tmpfs mount`：将临时文件存在主机内存中，适合存储临时文件

然而挂载之后后端还是无法进行热加载的，我们需要根据不同的编程语言进行设置，以下用Java举例：（下面配置完`docker-compose.yaml`则可以每次运行时写大量docker命令，直接启动服务即可）

**Java (Spring Boot DevTools)**

1. 确保 `spring-boot-devtools` 在 `pom.xml` 中

   ```
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-devtools</artifactId>
     <scope>runtime</scope>
   </dependency>
   ```

2. 启动容器时挂载代码

   ```
   docker run -d \
     -v $(pwd)/src:/app/src \  # 挂载源码目录
     -p 8080:8080 \
     your-java-image
   ```

3. 使用 `docker-compose` 简化配置

  1. 创建 `docker-compose.yaml` 文件：
     ```yaml
     version: '3'
     services:
       app:
         image: your-image
         volumes:
           - ./app:/app  # 挂载代码目录
         ports:
           - "3000:3000"
         command: npm run dev  # 直接运行热加载命令
     ```

  2. 启动服务
     ```
     docker-compose up
     ```

     
