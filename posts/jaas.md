---
title: 一个技术人的小玩具项目——JaaS
description: 记录 JaaS（Joke-as-a-Service）从接口开发到 Docker 部署、Nginx 反向代理与 DNS 配置的完整实现过程。
pubDate: 2025-12-02 21:01:02
---
最近发现了一个好玩的项目 [No as a service](https://github.com/hotheadhacker/no-as-a-service)。这个项目开发了一个有趣的api，然后将其部署到了网站。

受这个项目启发，我想做一个「Joke as a service」的项目。以下是实现和部署项目的记录。

## 实现步骤

### 1. 开发项目
项目整体结构非常简单，由以下三部分组成：
* Express 服务端：负责创建 Web 服务、加载 Joke 数据并提供 API。
* Joke 数据存储（jokes.json）：使用本地 JSON 文件作为轻量级持久化方案。
* /joke 接口：向客户端返回随机 Joke 及其解释，是整个服务的核心 API。

具体代码见：https://github.com/Himoryzhang/joke-as-a-service

### 2. 部署镜像
部署方面采用docker容器化部署的方案。

选取docker是因为docker部署非常方便，不需要考虑服务器上是否安装相关依赖，只要有docker，就能部署。

#### 1. 构建镜像
这里让cursor帮我们生成镜像的描述文件dockerfile，内容如下
```dockerfile
# Use the official Node.js LTS Alpine image as the base image
FROM node:18-alpine

# Set the working directory
WORKDIR /app

# Copy package.json and yarn.lock files
COPY package.json yarn.lock ./

# Install production dependencies
RUN yarn install --production --frozen-lockfile

# Copy the project source code
COPY . .

# Expose the port the app runs on
EXPOSE 3000

# Start the application
CMD ["yarn", "start"]
```

运行命令构建镜像。由于我是在mac上构建，在debian上部署，因此需要指定 `--platform` 参数，避免构建得到的镜像无法在debian运行。
```bash
docker build --platform linux/amd64  -t jaas . 
```

#### 2. 上传镜像
我使用了比较传统的方式，手动将镜像传到云服务器。

##### 1. 将镜像打包为tar文件
```bash
docker save -o jaas.tar jaas:latest
```

##### 2. 利用工具上传镜像至服务器
借助Termius其内置的sftp能力，将镜像上传到云服务器。


#### 3. 加载镜像
在运行镜像之前，需要将上传的image读取到docker中。

##### 1. 加载镜像
```bash
docker load -i jaas.tar
```

##### 2. 检查是否加载成功
```bash
docker images
```
如果image列表中有我们的镜像，则代表加载成功。

#### 4. 启动容器
启动容器，从而运行jaas服务。

##### 1. 启动容器，并且设置容器名称。
```bash
docker run -d --name jaas -p 127.0.0.1:3000:3000 jaas:latest
```

执行完毕启动容器的命令后，会返回一串字符串。

##### 2. 检查是否启动成功
可以使用curl命令检查api是否生效
```bash
curl localhost:3000/joke

{"joke":"Why don't programmers like nature? It has too many bugs.","explanation":"In programming, 'bugs' are errors in code. In nature, bugs are insects. Programmers don't like code bugs, so they joke they don't like nature's bugs either."}
```

### 3. 设置反向代理和DNS解析
目前为止，服务运行在我们服务器本地，外界无法访问。我们可以通过运行一个web服务器，设置反向代理，使外界服务可以通过访问我们的web服务器，以访问接口。

#### 1. 安装Nginx
```bash
sudo apt install nginx
```

安装完毕后，检查是否安装成功
```bash
systemctl status nginx
```

#### 2. 配置Nginx
需要添加反向代理配置。

由于debian系统规定，`/etc/nginx/sites-available`中存放可用的站点信息，`/etc/nginx/sites-enabled`中存放启用的站点。站点无需编写两次，在`sites-available`下创建后，创建软链接至`sites-enabled`下即可。

##### 1. 创建站点配置
在`/etc/nginx/sites-available`下创建文件`<site_name>，内容如下
```conf
server {
        listen 80;
        server_name <site_name>;

        location = / {
                return 200 '<site_name> running';
                add_header Content-Type text/plain;
        }

        location = /joke {
                return 301 /joke/;
        }

        location /joke/ {
                proxy_pass http://127.0.0.1:3000;

                proxy_http_version 1.1;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }
}
```

##### 2. 重新启动服务以使配置生效
```bash
systemctl restart nginx
```

或者也可以仅重新读取配置
```bash
nginx -s reload
```
##### 3. 检查是否生效
在服务器上执行curl命令，检查是否生效
```
curl -H "Host: <site_name>" http://127.0.0.1/joke/
```
应返回接口信息。
```json
"joke":"Why was the math book sad? It had too many problems.","explanation":"A 'problem' in a math book refers to math exercises, but it also means difficulties or troubles in general."}
```
#### 3. 创建DNS记录
需要为域名创建A记录，解析至云服务器。至此部署完成，访问`<site_name>/joke`即可访问jaas服务。

## 未来可以提升的点
部署过于复杂，可以创建流水线以实现自动部署。

## Q & A
这里记录了部署时可能遇到一些问题和解决方案。
### 部署完毕后，通过浏览器无法正常访问服务器，甚至nginx部署成功的页面都没有
#### 定位问题
我们先检查日志，查看nginx是否接收到了请求。
日志位置：`/var/log/nginx/access.log`
如果日志中没有收到请求，说明请求被拦截了，接下来查看防火墙配置。

#### 检查防火墙配置
有两个可能引起问题的防火墙
##### 1. 检查云服务提供商中的服务器防火墙配置。
参见云服务商中服务器的管理台，是否有firewall等模块，检查是否存在规则阻止了80、443端口。

##### 2. 检查服务器系统中的防火墙配置
查看ufw防火墙配置
```bash
sudo ufw status
```

如果返回中存在`Status: active`，并且没有`80`、`443`等端口，说明目前不允许外部访问服务器的80、443端口，因此需要开放这些端口。
```
sudo ufw allow 80
sudo ufw allow 443
```

再次检查：
1. `sudo ufw status`中应存在80、443端口配置。
2. 使用浏览器访问服务器，可以看到nginx的index页面。

