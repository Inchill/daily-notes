| Dockerfile | 指令 |	指令简介 |
| --- | --- | --- |
| FROM |	Dockerfile | 除了注释第一行必须是 FROM ，FROM 后面跟镜像名称，代表我们要基于哪个基础镜像构建我们的容器。
RUN	RUN 后面跟一个具体的命令，类似于 Linux 命令行执行命令。
ADD	拷贝本机文件或者远程文件到镜像内
COPY	拷贝本机文件到镜像内
USER	指定容器启动的用户
ENTRYPOINT	容器的启动命令
CMD	CMD 为 ENTRYPOINT 指令提供默认参数，也可以单独使用 CMD 指定容器启动参数
ENV	指定容器运行时的环境变量，格式为 key=value
ARG	定义外部变量，构建镜像时可以使用 build-arg = 的格式传递参数用于构建
EXPOSE	指定容器监听的端口，格式为 [port]/tcp 或者 [port]/udp
WORKDIR	为 Dockerfile 中跟在其后的所有 RUN、CMD、ENTRYPOINT、COPY 和 ADD 命令设置工作目录。
