# Docker

## 先决条件

[请先安装docker](https://docs.docker.com/installation/)

运行 Graylog 服务将会创建3个 Container

```
$ docker run --name some-mongo -d mongo:3
$ docker run --name some-elasticsearch -d elasticsearch:2 elasticsearch -Des.cluster.name="graylog"
$ docker run --link some-mongo:mongo --link some-elasticsearch:elasticsearch -p 9000:9000 -e GRAYLOG_WEB_ENDPOINT_URI="http://127.0.0.1:9000/api" -d graylog2/server
```

## 设置

Graylog 的默认配置就能工作，我们只需要给admin用户设置一个密码，同时需要给 Graylog API设置一个地址。两个配置都能通过环境变量来设置。

```
-e GRAYLOG_PASSWORD_SECRET=somepasswordpepper
-e GRAYLOG_ROOT_PASSWORD_SHA2=8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
-e GRAYLOG_WEB_ENDPOINT_URI="http://127.0.0.1:9000/api"
```

这样设置以后，你能通过admin作为用户名和密码来登录Graylog。你也可以通过下面的命令来生成自己的密码

```
$ echo -n yourpassword | shasum -a 256
```

我们可以把这些都放在 `docker-compose.yml` 文件里

```
version: '2'
services:
  mongo:
    image: "mongo:3"
  elasticsearch:
    image: "elasticsearch:2"
    command: "elasticsearch -Des.cluster.name='graylog'"
  graylog:
    image: graylog2/server:2.2.1-1
    environment:
      GRAYLOG_PASSWORD_SECRET: somepasswordpepper
      GRAYLOG_ROOT_PASSWORD_SHA2: 8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
      GRAYLOG_WEB_ENDPOINT_URI: http://127.0.0.1:9000/api
    depends_on:
      - mongo
      - elasticsearch
    ports:
      - "9000:9000"
``` 

用 docker-compose 启动三个容器之后，可以用浏览器访问 `http://127.0.0.1:9000`, 用admin/admin 登录。

## 怎样导入日志数据

我们可以通过 `System -> Inputs` 创建各种各样的输入, 然后只能使用 Docke容器已经做了映射的那些端口，不然数据将无法通过。

## 持久化日志数据

要持久化日志数据和配置文件，我们需要使用docker的外部卷来存储所有的数据。当容器重新启动的时候，便能重用之前的容器实例留下的数据。
下面的命令创建配置目录并拷贝默认的配置文件:

```
mkdir /graylog/config
cd /graylog/config
wget https://raw.githubusercontent.com/Graylog2/graylog2-images/2.2/docker/config/graylog.conf
wget https://raw.githubusercontent.com/Graylog2/graylog2-images/2.2/docker/config/log4j2.xml
```

`docker-compose.yml` 文件:

```
version: '2'
services:
  mongo:
    image: "mongo:3"
    volumes:
      - /graylog/data/mongo:/data/db
  elasticsearch:
    image: "elasticsearch:2"
    command: "elasticsearch -Des.cluster.name='graylog'"
    volumes:
      - /graylog/data/elasticsearch:/usr/share/elasticsearch/data
  graylog:
    image: graylog2/server:2.2.1-1
    volumes:
      - /graylog/data/journal:/usr/share/graylog/data/journal
      - /graylog/config:/usr/share/graylog/data/config
    environment:
      GRAYLOG_PASSWORD_SECRET: somepasswordpepper
      GRAYLOG_ROOT_PASSWORD_SHA2: 8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
      GRAYLOG_WEB_ENDPOINT_URI: http://127.0.0.1:9000/api/
    depends_on:
      - mongo
      - elasticsearch
    ports:
      - "9000:9000"
      - "12201/udp:12201/udp"
      - "1514/udp:1514/udp"
```

启动所有的容器

```
$ docker-compose up
```
## 配置

所有的配置选项都能通过环境变量来设置。只需在参数名前面加上GRAYLOG_ 前缀，并使用大写


下面的例子设置了邮件服务的smtp，`docker-compose.yml` :

```
environment:
  GRAYLOG_TRANSPORT_EMAIL_ENABLED: "true"
  GRAYLOG_TRANSPORT_EMAIL_HOSTNAME: smtp
  GRAYLOG_TRANSPORT_EMAIL_PORT: 25
  GRAYLOG_TRANSPORT_EMAIL_USE_AUTH: "false"
  GRAYLOG_TRANSPORT_EMAIL_USE_TLS: "false"
  GRAYLOG_TRANSPORT_EMAIL_USE_SSL: "false"
```

## 插件

要添加插件，你可以以 graylog2/server 镜像为基础，添加所需的插件，并构建新的镜像。在一个空的目录添加一个 Dockerfile:

```
FROM graylog2/server:2.2.1-1
RUN wget -O /usr/share/graylog/plugin/graylog-plugin-beats-1.1.0.jar https://github.com/Graylog2/graylog-plugin-beats/releases/download/1.1.0/graylog-plugin-beats-1.1.0.jar
```

构建新的镜像:

```
$ docker build -t graylog-with-beats-plugin
```

在这个例子当中， 我们创建一个安装了Beats插件的新镜像。我们现在可以引用这个镜像，例如在 `docker-compose.yml` 中

```
version: '2'
services:
  mongo:
    image: "mongo:3"
    volumes:
      - /graylog/data/mongo:/data/db
  elasticsearch:
    image: "elasticsearch:2"
    command: "elasticsearch -Des.cluster.name='graylog'"
    volumes:
      - /graylog/data/elasticsearch:/usr/share/elasticsearch/data
  graylog:
    image: graylog-with-beats-plugin
...
```


