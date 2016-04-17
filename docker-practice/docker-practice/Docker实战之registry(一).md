本期课程: Docker实战之Registry以及持续集成

上一讲<视频>我们知道了Docker的基础知识，以及学会了Dockerfile，第二讲将结合一个实际的Java应用，演示如何通过Docker以及Registry

实现自动化的镜像构建、自动部署以及持续集成测试。

通过本次课程，你将会学会：


1. 如何通过Git仓库，自动生成Docker镜像
2. 如何自动将多个容器部署起来
3. 容器部署好后，如何利用Jenkins自动做集成测试

#构建私有Registry
因各位知道的原因（GWF）无法使用docker hub 官方方式pull获取registry.建议注册Daocloud账号使用Dao加速器关联私有docker方式管理，可以优雅的使用docker hub.

账号：huyipow

官方地址：https://www.daocloud.io/

登陆Daocloud

    [root@localhost ~]# docker login daocloud.io
    Username (huyipow): huyipow
    WARNING: login credentials saved in /root/.docker/config.json
    Login Succeeded

#安装Dao主机监控程序
下载并安装
 
    curl -L -o /tmp/daomonit.x86_64.rpm https://get.daocloud.io/daomonit/daomonit.x86_64.rpm 
    sudo rpm -Uvh /tmp/daomonit.x86_64.rpm

配置

    daomonit -token=ff84968710cb00bc4b54760fd1c29e44d72921fb save-config
获取 registry image

    #[root@localhost ~]dao pull registry
    d8fadd78ba55: Load layer complete  
    2871f9204c1a: Load layer complete  
    a732fc2f204f: Load layer complete  
    01f379e9985c: Load layer complete  
    Loading image to docker ...
    ** Pull library/registry success. **




查看 registry pull是否成功？

    [root@localhost centos7]# docker images 
    REPOSITORY  TAG IMAGE IDCREATED  VIRTUAL SIZE
    registry	latest  8e93bdf0fcadAbout a minute ago   370.3 MB
    csphere/wordpress   4.2 20865c5fc2ac26 hours ago 745.4 MB
    csphere/mysql   5.5 de25e804d33629 hours ago 659.9 MB
    csphere/php-fpm 5.4 f4635faaf1f830 hours ago 707.8 MB
    csphere/centos  7.2 464754cf288431 hours ago 526.5 MB
    csphere/centos  last464754cf288431 hours ago 526.5 MB
    <none>  <none>  d677694c61eb31 hours ago 591.2 MB
    csphere/centos  7.1 0bdef1d1c28a4 days ago   591.2 MB
    docker.io/centoscentos7.2.1511  83ee614b834e6 weeks ago  194.6 MB
    docker.io/centoscentos7.1.1503  fab4b1df8eb13 months ago 212.1 MB
运行registry 

    [root@localhost centos7]# docker run -d -p 5000:5000 --name registry registry:latest
检查registry 是否启动成功

    [root@localhost php-fpm]# docker ps -a
    CONTAINER IDIMAGE COMMAND CREATED STATUS PORTSNAMES
    ea8fefcbebf9csphere/php-fpm:5.4   "/bin/bash" 7 minutes ago   Exited (0) 7 minutes agonginx
    673a9a3fd8d7registry:latest   "docker-registry"   2 hours ago Up 2 hours 0.0.0.0:5000->5000/tcp   registry

给centos 镜像制作tag标签

> Usage:	docker tag [OPTIONS] IMAGE[:TAG] [REGISTRYHOST/][USERNAME/]NAME[:TAG]

    [root@localhost php-fpm]# docker tag centos:7.2.1511 192.168.40.134:5000/csphere/centos:7.2.1511
    [root@localhost php-fpm]# docker images 
    REPOSITORY  TAG IMAGE IDCREATED VIRTUAL SIZE
    csphere/php-fpm 5.4 a1b79bd76f728 minutes ago   325.7 MB
    registrylatest  01f379e9985c8 days ago  422.8 MB
    daocloud.io/daocloud/daocloud-toolset   latest  1e743a7453e44 weeks ago 145.8 MB
    centos  7.2.151183ee614b834e6 weeks ago 194.6 MB
    192.168.40.134:5000/csphere/centos  7.2.151183ee614b834e6 weeks ago 194.6 MB

#安装docker-compose 

    curl -L https://github.com/docker/compose/releases/download/1.5.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

修改文件权限

    chmod +x /usr/local/bin/docker-compose
查看安装版本

    docker-compose --version
    docker-compose version: 1.5.2
使用docker-compose 批量自动化启动docker镜像。注意启动的容器必须实现存在镜像

    [root@localhost second]# cat docker-compose.yml 
    mysql:
       image: csphere/mysql:5.5
       ports: 
     - "3306:3306"
       volumes:
     - /var/lib/docker/vfs/dir/dataxc:/var/lib/mysql
       hostname: mydb.server.com
    
    tomcat:
       image: csphere/tomcat:7.0.55
       ports:
      - "8080:8080"
       links:
      - mysql:db
       environment:
      - TOMCAT_USER=admin
      - TOMCAT_PASS=admin
启动镜像

    [root@localhost second]# docker-compose up -d
    Starting second_mysql_1
    Starting second_tomcat_1
停止镜像

    [root@localhost second]# docker-compose stop
    Stopping second_tomcat_1 ... done
    Stopping second_mysql_1 ... done
查看运行状态

    [root@localhost second]# docker-compose ps
     NameCommandState Ports 
    ---------------------------------------------------
    second_mysql_1/scripts/start   Exit 137 
    second_tomcat_1   /scripts/run Exit 143

删除docker-compose 创建的镜像      
    [root@localhost second]# docker-compose rm 
    Going to remove second_tomcat_1, second_mysql_1
    Are you sure? [yN] y
    Removing second_tomcat_1 ... done
    Removing second_mysql_1 ... done
    
##如何通过git仓库自动构建docker镜像
下图描述了Docker模式在软件部署方式上的流程
![](http://i.imgur.com/GZqkhnR.png)

下图描述了传统模式和Docker模式在软件部署方式上的区别
![](http://i.imgur.com/wzdP6hb.png)

#构建一个java程序自动化发布。
----------
- java编译器maven需要提前下载好或者使用自己软件仓库。

- 准备以下版本docker镜像

----------

    csphere/jenkins   1.642.1 
    csphere/jdk   1.7.0   
    csphere/tomcat 7.0.55  
    csphere/jre   1.7.0
运行jenkins 容器

    [root@localhost maven]# docker run -d -p 8080:8080 --name jenkins -v /usr/bin/docker:/usr/bin/docker -v /var/run/docker.sock:/var/run/docker.sock -v /root/maven-tar/:/root csphere/jenkins:1.642.1

登入jenkins容器验证maevn程序是否存在

    [root@localhost maven]# docker exec -it jenkins /bin/bash
    [jenkins@0dd2048a51f0 root]$ ls /root/
    apache-maven-3.3.9-bin.tar.gz
使用宿主机192.168.140.132:8080访问jenkins，首次登陆允许匿名用户登录。生产环境加强安全需要修改登录权限。

点击 系统管理：Configure Global Security
![](http://i.imgur.com/MwdQJTx.png)
    

docker build -t csphere/php-fpm:5.4 $WORKSPACE