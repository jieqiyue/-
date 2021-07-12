### 	docker 安装 zookeeper

1. docker pull zookeeper:3.5.8
2. docker run -d --name zookeeper -p 2181:2181 zookeeper:3.5.8

此时已经安装启动成功。

进入命令行操作

3. docker exec -it ContainerID /bin/bash 

即可像在命令行中操作zookeeper一样操作

cd bin                        ./zkCli.sh -server 127.0.0.1:2181