@[TOC](Dillinger)
### dillinger 说明文档
dillinger 是一款在线 markdown 编辑器，外观简洁，操作方便，但由于其服务器在国外，对国内用户可能不太友好，解决办法是安装本地服务器，有两种方法安装，具体可参考[官方网站](https://dillinger.io/)，其中一种比较简单的方法时利用 docker 进行安装，docker 的安装步骤请参考[ docker 部署文档](https://dillinger.io/)，dillinger 的具体安装步骤如下：

- 步骤一：从[官方镜像地址](https://hub.docker.com/r/joemccann/dillinger)拉取最新 dillinger 镜像。
    ```
    docker pull joemccann/dillinger
    ```
- 步骤二：启动 dillinger 容器。
    ```
    docker run -d -p 8000:8080 joemccann/dillinger
    ```
    启动 dillinger 服务器时可以指定映射端口，命令中的 8000 对应宿主机端口，8080 对应 dillinger 的端口，两者都可以改变。
- 步骤三：用浏览器访问 dillinger 服务器。
    可以直接本地访问：
    ```
    127.0.0.1:8000
    ```
    也可以远程访问：
    ```
    your_ip_address:port
    ```