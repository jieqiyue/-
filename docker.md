#### 什么是docker？

docker是用于运行application的隔离环境。使得进程间相互隔离，并且能够将硬件资源划分到不同的组中。

#### docker和虚拟机的区别？

虚拟机是在硬件层面上抽象的机器。所以我们可以在一台物理机上面运行多个虚拟机。我们通过Hypervisor（VMware，Hyper-v）来实现。

虚拟机可以实现在不同的隔离环境下运行程序。这个程序所依赖的版本是不同的。

![image-20210521105459925](C:/Users/Jieqiyue/AppData/Roaming/Typora/typora-user-images/image-20210521105459925.png)

那么，虚拟机有以下的缺点：

- Each VM needs a full-blown OS 
- Slow to start 
- Resource intensive

相比之下，docker有如下的特点：

- Allow running multiple apps in isolation 
- Are lightweight 
- Use OS of the host  
- Start quickly 
- Need less hardware resources 

#### docker的架构

docker 系统使用了 C/S 的架构，docker client 通过 REST API 请求 docker daemon（server） 来管理 docker 的镜像和容器等。

Docker client 是给用户和 Docker daemon 建立通信的客户端，安装了 docker 之后，二进制文件 `docker` 就是 Docker client，与 Docker daemon 交互，实现对 Docker image 和 container 的管理请求。

Docker client 与 docker daemon 建立请求的方式有三种，分别是：

- tcp://host:port
- unix://path/to/socket
- fd://socketfd

![img](https://raw.githubusercontent.com/James-jqy/Concurrency/master/imgss/1460000006448762)

