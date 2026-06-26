# 1 拉取镜像

~~~shell
docker pull mysql:latest
~~~

# 2 打包镜像到服务器指定目录

~~~shell
docker save mysql > /root/test/mysql.tar
~~~

mysql 是镜像名字

# 3 下载打包的mysql.tar文件，并上传到新服务器

# 4 加载镜像

~~~shell 
docker load < /usr/local/docker/mysql.tar
~~~

![clipboard](./assets/05-Docker打包离线镜像到本地，上传解压到服务器.assets/1707968343167.png)

# 5 查看加载的镜像

~~~shell
docker image ls
#或
docker images 
~~~

![clipboard](./assets/05-Docker打包离线镜像到本地，上传解压到服务器.assets/1707968404360.png)

# 6 创建容器

~~~shell
docker run -itd --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql 
~~~

启动成功，docker ps也能查到，成功

![img](./assets/05-Docker打包离线镜像到本地，上传解压到服务器.assets/1707968343165.png)



---



<p align="center">
    <a href="https://blog.csdn.net/dkbnull/article/details/136159798" target="_blank">
       <img src="https://img.shields.io/badge/CSDN-访问地址-red?logo=csdn">
    </a>
    <a href="https://mp.weixin.qq.com/s/8WcogkO0hCcooLOquC1JuA" target="_blank">
       <img src="https://img.shields.io/badge/微信公众号-访问地址-brightgreen?logo=wechat">
    </a>
</p>

