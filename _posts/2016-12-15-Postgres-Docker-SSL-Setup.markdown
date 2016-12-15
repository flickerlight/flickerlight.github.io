---
layout: post
title:  "Docker PostgresSQL SSL Setup During Initialization"
date:   2016-12-15
author: Flickerlight
category: Other
---


基础是Docker官方提供的[postgres official image](https://hub.docker.com/_/postgres/),我用的是当前最新版本9.6.1。默认是SSL off的，而项目安全审计要求database必须使用SSL连接，花了半下午弄好了，在此记录一下。

PostgreSQL本身服务器端的SSL enable很简单，只需要生成好server certificate 放到data目录并改动几处配置下即可（可参考[官方文档](https://www.postgresql.org/docs/9.1/static/ssl-tcp.html)）。时间主要花在尝试把SSL的配置自动apply到docker image上。

具体步骤如下：

#### 1. 生成证书

我用的是自签名的证书，openssl命令生成。

```
openssl genrsa -out server.key 4096

openssl req -new -key server.key -out server.csr

openssl x509 -req -days 3650 -in server.csr -singkey server.key -out server.crt

```

#### 2. 把证书文件和初始化DB用的脚本文件一起在Dockerfile里ADD到/docker-entrypoint-initdb.d/。

根据Docker postgres image的官方文档，放到/docker-entrypoint-initdb.d/下的.sql和.sh会被entrypoint执行，一般对于DB的初始化操作（建表，建用户，导入metadata之类）都建议放在这里自动完成。

>After the entrypoint calls initdb to create the default postgres user and database, it will run any *.sql files and source any *.sh scripts found in that directory to do further initialization before starting the service.

重点在于：

**把证书文件也放进/docker-entrypoint-initdb.d/，这样当entrypoint执行配置的sh脚本就能访问到证书。**

Dockerfile很简单就两行：

```
FROM library/postgres
ADD init/* /docker-entrypoint-initdb.d/
```

我的init文件夹中的内容如下，两个sql分别用于创建schema和导入初始数据。setupSSLConnection.sh用来配置服务器使用SSL远程连接。

```
init_schema.sql
init_metadata.sql
server.key
server.crt
setupSSLConnection.sh
...

```


#### 3. 在bash配置脚本里，把证书文件放到Postgres的data目录下，给予正确权限，并修改postgres.conf 和pg_hba.conf

setupSSLConnection.sh：

```bash
#!/bin/bash

cp /docker-entrypoint-initdb.d/server.crt /var/lib/postgresql/data/
cp /docker-entrypoint-initdb.d/server.key /var/lib/postgresql/data/
chmod 600 /var/lib/postgresql/data/server.key
sed -i 's/^#ssl = off/ssl = on/' /var/lib/postgresql/data/postgresql.conf
sed -i 's/^host all all/hostssl all all/' /var/lib/postgresql/data/pg_hba.conf
```


#### 4. 在docker-compose.yml里完成其他配置工作，如监听端口、连接密码、持久化数据路径等。

docker-compose.yml:

```
version: '2'
services:
  postgres-test:
    build: .
    image: postgres-test:latest
    restart: always
    container_name: postgres-test
    ports:
      - 5432:5432
    environment:
      - POSTGRES_PASSWORD=test123
    volumes:
      - /mnt/data:/var/lib/postgresql/data
```

### 5. Build和运行

```bash
root$:docker-compose build
Building postgres-test
Step 1 : FROM library/postgres
 ---> 0267f82ab721
Step 2 : ADD init/* /docker-entrypoint-initdb.d/
 ---> Using cache
 ---> 38f1ae10a1ef
Successfully built 38f1ae10a1ef
root$:docker-compose up
```

### 6. 用psql连一下吧，应该已经提示是SSL连接了。

```
root$ psql -h localhost -U postgres
Password for user postgres: 
psql (9.6.1)
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# 

```

搞定，收工！