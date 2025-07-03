---
title: Docker-相关问题
draft: false
tags:
  - 中间件
date: 2023-10-13
---
 
# 什么是Docker？它解决了什么问题？
 
 `docker` 是一个容器化平台，打包应用及其依赖到一个标准化的单元（容器）。解决“在我机器上能跑”的环境一致性问题，提供轻量级、快速启动、资源隔离的运行时环境。
# Docker容器与虚拟机(VM)的主要区别是什么？各自的优缺点是什么？

- VM虚拟化硬件：运行完整OS（Guest OS），重量级，启动慢，资源占用高
- docker 容器化技术：容器虚拟化OS内核，共享宿主机内核，轻量级，启动快，资源利用率高。容器隔离性弱于VM。

# 解释一下Docker镜像(Image)、容器(Container)和仓库(Registry)之间的关系。

- registry： **仓库**用于存储和分发镜像（公共如Docker Hub，私有如Harbor）
- image：**镜像**是只读模板，包含创建容器所需的文件系统和元数据。
- container：**容器**是镜像的运行实例，包含一个可写层。

# `Dockerfile` 的作用是什么？它包含哪些关键指令？

`dockerfile` 用于定义**如何自动构建Docker镜像**的文本文件，有以下常用指令
- `FROM`：指定基础镜像
- `RUN` ：执行命令（安装软件、编译代码等）
- `COPY / ADD` ：复制文件/目录到镜像中
- `WORKDIR` ：设置工作目录。
- `EXPOSE`: 声明容器运行时监听的端口。（此处只是声明，只是标识作用，既不是容器真的运行端口也不是映射的宿主机端口）
-  `CMD` / `ENTRYPOINT`: 指定容器启动时运行的命令。
-  `ENV`: 设置环境变量。

# 描述一下从编写`Dockerfile`到运行一个Java应用容器的基本步骤

1. 首先准备好已经编译好的应用 `jar` 包
2. 编写`dockerfile`
	1. 设置基础镜像，`FROM`
	2. `COPY` jar 包文件
	3. `CMD` / `ENTRYPOINT`: 指定容器启动时运行的命令。
3. 在 dockerfile 目录下运行命令，构建镜像 `docker build -t my-java-app:latest .`
4. 运行容器，`docker run -d -p 8080:8080 --name my-app-container my-java-app:latest`

# `docker run` 命令中 `-p` 和 `-v` 参数的作用是什么

- `-p [宿主机端口]:[容器端口]` 将宿主机的端口映射到容器内部的端口，使外部可以访问容器服务。
- `-v [宿主机路径]:[容器路径]` 将宿主机的目录或文件挂载到容器内部路径，实现数据持久化和宿主机-容器间文件共享。

# 如何查看正在运行的容器？如何查看所有容器（包括停止的）？如何查看容器日志？如何进入一个正在运行的容器？

- `docker ps` (查看运行中的容器)    
- `docker ps -a` (查看所有容器)    
- `docker logs` (查看容器日志，`-f` 跟踪日志)    
- `docker exec -it /bin/bash` (进入容器交互式终端，常用 `/bin/bash` 或 `/bin/sh`)

# 如何停止、启动和删除一个容器？如何删除一个镜像?

- `docker stop` (停止容器)    
- `docker start` (启动已停止的容器)    
- `docker rm` (删除停止的容器，`-f` 强制删除运行中的)    
- `docker rmi` (删除镜像，需先删除依赖它的容器)


# 在容器中运行Java应用时，如何设置JVM参数（如堆内存大小）？

- 方式1：在 `Dockerfile` 的 `CMD` 或 `ENTRYPOINT` 中直接指定：`java -Xmx512m -jar app.jar`
- 方式2：通过环境变量传递（启动脚本控制）
	1. dockerfile 中设置启动脚本为
		```yaml
			CMD ["sh" "-c" "java $JAVA_OPTS -jar jar包路径"]
		```

	2. 启动容器时，指定`JAVA_OPTS`值
		```bash
			docker run -e "JAVA_OPTS=-Xmx512m" ...
		```

