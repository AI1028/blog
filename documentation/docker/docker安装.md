@[TOC](Docker)
### docker 部署文档
- 卸载旧版本 docker 
    ```
    sudo apt-get remove docker \
               docker-engine \
               docker.io
    ```
- 添加使用 HTTPS 传输的软件包以及 CA 证书
    ```
    sudo apt-get update
    sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
    ```
- 添加软件源的 GPG 密钥
    ```
    curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
    ```
- 向 source.list 中添加 Docker 软件源
    ```
    sudo add-apt-repository \
    "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
    ```
- 安装 Docker CE
    ```
    sudo apt-get update
    sudo apt-get install docker-ce
    ```
- 启动 Docker CE
    ```
    sudo systemctl enable docker
    sudo systemctl start docker
    ```
- 建立 docker 用户组
    ```
    sudo groupadd docker
    sudo usermod -aG docker $USER
    ```
- 测试 Docker 是否安装正确，显示 hello from docker ... 等信息，即为安装成功。
    ```
    docker run hello-world
    ```