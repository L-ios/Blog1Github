title: 初涉Docker

# tags:

# 记录docker的学习

## Docker的安装

首先学习的环境Ubuntu 16.04， 确认内核版本号：

```
$ uname -r
4.4.0-36-generic
```

这里的内核版本号是 4.4.0-36-generic

添加gpg文件

```
    $ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
```

如果这里无法执行，则需要安装一些包

```
sudo apt-get install apt-transport-https ca-certificates
```

安装docker

```
$ sudo apt-get install docker-engine
```

这里可能要卸载以前的 "lxr-docker"来进行安装（此处未解）

等待安装完成后，需要测试是否能正常运行:

- 首先启动docker服务

    ```
    $ sudo service docker start
    ```

- 运行docker的hello-world

    ```
    $ sudo docker hello-world
    Hello from Docker!
    This message shows that your installation appears to be working correctly.

    To generate this message, Docker took the following steps:
     1. The Docker client contacted the Docker daemon.
     2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
     3. The Docker daemon created a new container from that image which runs the
        executable that produces the output you are currently reading.
     4. The Docker daemon streamed that output to the Docker client, which sent it
        to your terminal.

    To try something more ambitious, you can run an Ubuntu container with:
     $ docker run -it ubuntu bash

    Share images, automate workflows, and more with a free Docker Hub account:
     https://hub.docker.com

    For more examples and ideas, visit:
     https://docs.docker.com/engine/userguide/

    ```
通过运行hello-world的docker，我们可以看到提示——> “docker run -it ubuntu bash”,这是启动一个基于
ubuntu的docker，且运行ubuntu中的bash命令

#### 一些docker的配置
#####创建一个docker用户组
