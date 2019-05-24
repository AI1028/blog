@[TOC](Kong)
### kong 的部署及使用
#####  安装数据库
1. 添加源
	```bash
	sudo vim /etc/apt/sources.list.d/pgdg.list
	deb http://apt.postgresql.org/pub/repos/apt/ trusty-pgdg main
	```
2. 导入key
	```bash
	wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
	sudo apt-get update
	```
3. 安装 PostgreSql
	```bash
	sudo apt-get install postgresql-9.5
	```
4. 修改配置文件
	- 修改 postgresql.conf 文件
		```bash
		sudo vim /etc/postgresql/9.5/main/postgresql.conf 
		```
		修改如下参数
		```bash
		listen_addresses = '*'
		password_encryption = on
		```
	- 修改 pg_hba.conf 文件
		```bash
		sudo vim /etc/postgresql/9.5/main/pg_hba.conf
		```
		添加如下内容
		```bash
		host all all 0.0.0.0 0.0.0.0 md5
		```
	- 重启 PostgreSql ，修改数据库密码并切换到默认创建的 postgres 用户
		```bash
		sudo service postgresql restart
		sudo passwd postgres
		su - postgres
		```
	- 为 Kong 建立数据库
		```bash
		psql postgres psql 
		CREATE USER kong_user WITH PASSWORD 'kong_pass'; 
		create database "kong_db"; GRANT ALL PRIVILEGES ON DATABASE "kong_db" to kong_user;
		```

##### 部署 Kong
1. 安装依赖
	```bash
	sudo apt-get install netcat openssl libpcre3 dnsmasq procps
	```
2. 下载并安装 Kong，下载地址：https://getkong.org/install/ubuntu/ 
	```bash
	sudo dpkg -i kong-0.8.3.*.deb
	```
3. 修改 Kong 配置文件
	```bash
	sudo vim /etc/kong/kong.conf 
	```
	修改内容如下
	```bash
	database = 'postgres'
	pg_host = 127.0.0.1
	pg_port = 5432
	pg_user = kong_user
	pg_password = kong_pass
	pg_database = kong_db
	pg_ssl = off
	pg_ssl_verify = off
	```
4. 初始化数据库表
	```bash
	kong migrations up
	```
5. 启动与停止 Kong
	```bash
	kong start
	kong stop
	```
#####  部署 kong-dashboard
1. 安装 nodejs-legacy
    ```
    sudo apt install nodejs-legacy
    ```
2. 安装 npm
    ```
    sudo apt install npm
    ```
3. 安装模块 n
    ```
    sudo npm install -g n
    ```
4. 安装稳定版本 nodejs
    ```
    sudo n stable
    ```
5. 安装 kong dashboard
    ```
    npm install -g kong-dashboard
    ```
6. 运行 kong dashboard
    ```
    kong-dashboard start --kong-url http://localhost:8001 --port 8088
    ```
7. 在浏览器中访问 kong dashboard
    ```
    http://localhost:8088
    ```

#####  添加 API
简单演示百度开源 OCR-API 添加过程
1. 添加服务
	```bash
	curl -i -X POST \
  	--url http://localhost:8001/services/ \
  	--data 'name=ocr-service' \
  	--data 'url=https://aip.baidubce.com/rest/2.0/ocr/v1/general'
	```
2. 为服务添加路由
	```bash
	curl -i -X POST \
 	 --url http://localhost:8001/services/ocr-service/routes \
 	 --data 'hosts[]=ocr.com'
	```
3. 创建用户
	```bash
	curl -i -X POST \
 	--url http://localhost:8001/consumers/ \
  	--data "username=user_name"
	```
4. 为用户创建密码
	```bash
	curl -i -X POST \
  	--url http://localhost:8001/consumers/HuiChen/key-auth/ \
  	--data 'key=123456'
	```
##### 调用 API
通过 Kong 调用百度 OCR-API，完整调用过程如下
1. 启动 PostgreSql 数据库
	```bash
	service postgresql start
	```
2. 启动 Kong
	```bash
	sudo kong start
	```
3. 启动 Kong-dashboard（[Kong-dashboard 部署过程](https://blog.csdn.net/weixin_43377750/article/details/88020268)）
	```bash
	kong-dashboard start --kong-url http://localhost:8001 --port 8088
	```
4. 使用 postman 模拟访问 OCR-API
	  param  |   value
	---|---
	  POST  |  http://locaohost:8000?access_token=24.be78b6236896fa66c118d28bfcb1733f.2592000.1550629910.282335-15062501
	  Host  |  ocr.com
	  Content-Type     |  application/x-www-form-urlencoded
	  apikey  |  123456
	  url  |  https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=2730107330,1426106144&fm=15&gp=0.jpg
	  Host、Content-Type、apikey 放在 Header 中，url 放在 Body 中。
6. 结果示例：
	- 待识别图片：
	![在这里插入图片描述](https://img-blog.csdnimg.cn/2019021821045619.PNG)
	- 返回结果：
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20190218210255806.PNG)