# 小白也可以自己安装n8n！

## 缘起

  最近大火的一个AI工作流编排工具是德国人出的n8n，很多用过的博主坚称这是一款最懂程序员的AI工作流编排工具，很贴近程序员思维，非常适合程序员来使用等等巴拉巴拉一堆。忍不住心痒痒准备试用下，心动不如行动，马上开干。

## 行动前准备

   等等，在动手之前，先允许我介绍下我的安装方式和手上现有资源，以及可能遇到的问题。

### 首先，是安装方式

    n8n官网推荐的安装方式有3种，具体可以参考以下链接中 **Set up your n8n** 章节：

[Learning path | n8n Docs](https://docs.n8n.io/learning-path/)

1. 通过n8n cloud试用（确切的说，这种不算安装😂）。
2. **通过docker安装**。
3. 通过npm安装。

对于一个有保持原生环境尽可能“干净”的强迫症患者，我毫不犹豫的选择 **docker安装**，当然，这也是官方建议的安装方式。

### 其次，所需资源

   既然决定了用docker安装，那么接下来看下n8n docker安装的资源需求，官网上没有明确说明n8n安装最小配置的需求，问了ChatGPT，得出来的结论是：

### 最小可行配置（个人 / 低并发）

- **ECS/EC2**: 1 vCPU / 2GB / 20GB SSD
- **数据库**: 内置 SQLite
- **部署方式**: Docker（单容器）
- **代理**: Nginx（HTTP，可加 Certbot）

### 推荐生产配置（小团队）

- **ECS/EC2**: 2 vCPU / 4GB / 20GB SSD
- **数据库**: 独立 PostgreSQL（RDS 或自建容器）
- **部署方式**: Docker Compose（n8n + Postgres）
- **代理**: Nginx/Traefik + HTTPS

可以看到，对我这样的个人使用者，1C2G 20G存储足以，而恰好手边有一个阿里云的2C2G的服务器，虽然上面部署了其他服务器，但是抱着能省则省的理念，就用它了。

虽然是个人使用，但我还是想尽可能靠近生产环境，所以数据库改成了PostgreSQL，**最终配置**如下：

- **阿里云**: 2 vCPU / 2GB / 40GB SSD （99元包年）
- **数据库**: PostgreSQL（Docker 安装）
- **部署方式**: Docker（单容器）
- **代理**: Nginx（HTTP，可加 Certbot）
- **HTTPS**：域名需要备案，时间来不及，自签名（生产不建议这么做）

### 最后，可能遇到的问题

上面说了，这台服务器还在跑着其他docker容器，资源需求倒是可以满足，问题是其中一个docker容器跑着nginx，给其他服务反向代理，好在没有用到443端口，可以给n8n使用，但需要对现有nginx容器配置做一些改动。

还有就是因为众所周知的网络原因，n8n的docker镜像不容易下载，虽然阿里云提供了容器加速器，但是就我这次实际操作来说，n8n镜像通过阿里云的加速器无法下载。

另外一个需要解决的问题是n8n生产环境推荐是https访问，如果只是访问自搭的n8n服务器还好，但是n8n有个很好用的功能就是提供webhook，这个也需要https的支持，所以，https就变得不可或缺，但是手头上没有已经备案好的域名，只能通过IP地址来访问，所以只能通过自签名的方式先凑合着公网IP用下。

好了，总结下可能遇到的问题和解决方案：

| **问题** | **问题描述** | **解决方案** |
| --- | --- | --- |
| **Nginx容器配置适配问题** | 服务器中已有运行的nginx容器（用于反向代理其他服务），未使用443端口，需修改其配置以适配n8n | 调整现有nginx容器的配置，将443端口分配给n8n使用，并配置相应的反向代理规则 |
| **n8n的docker镜像下载困难** | 因网络原因，n8n的docker镜像难以下载，且阿里云容器加速器无法解决该问题 | 尝试其他可用的容器镜像源（如DaoCloud、网易云等）；或通过能正常访问的环境下载镜像后，导出再导入目标服务器；也可考虑使用代理工具辅助下载 |
| **无备案域名下的HTTPS配置需求** | 生产环境需https访问n8n（尤其webhook功能依赖），但无备案域名，只能通过公网IP访问 | 采用自签名SSL证书的方式，为n8n配置https访问，以满足webhook等功能的安全需求 |

### 行动

接下来，我们按照如下步骤在已有nginx的服务器上安装n8n。

1. **独立部署 n8n**：使用 Docker Compose 启动 n8n 和 PostgreSQL，它们会拥有自己的独立网络。
2. **探查现有 Nginx 容器**：我们需要了解您当前 Nginx 容器的配置路径和网络情况。
3. **生成自签名证书**：生成https所需的自签名证书。
4. **打通网络**：将您**正在运行的 Nginx 容器**连接到 n8n 的网络中，使其能够“看到”n8n 容器。
5. **添加新配置**：在 Nginx 的配置目录中**新建一个**专门用于 n8n 的配置文件。
6. **平滑重载 Nginx**：执行一个不会中断现有服务的命令，让 Nginx 加载新的配置。

- **第一步 独立部署n8n**
1. **安装n8n镜像**
通过科学访问，获取n8n镜像。如果不会科学访问的，可以自行下载共享出来的n8n docker tar文件 [n8n镜像打包文件](https://pan.quark.cn/s/c1c357be485a?pwd=HJBd)，通过docker load 命令导入到服务器中（该文件只做学习和研究使用，请自行负责安全和其他任何问题）。然后将下载好的文件导入到服务器中，参考如下命令：



      

```bash
sudo docker load -i /path/to/image.tar
```

上面的  /path/to 换成你自己上传tar文件所在的服务器目录，image.tar 如果是下载的共享文件，换成n8n.tar。

1. **安装 postgreSQL 镜像**

```bash
sudo docker pull postgres:14
```

之所以要先下载镜像到服务器，就是为了避免网络问题，无法拉取镜像，这里强调下，之所以使用阿里云服务器也是因为阿里云为自己的服务器提供了镜像加速器，在拉取镜像上有很大便利，我的另外一台火山引擎的服务器就没有加速器可以使用，在这里还是给阿里云点个赞（虽然n8n的镜像拉不到，但残血版总比没有好）。

1. **编写docker-compose.yml文件 并启动 n8n 和 postgreSQL** 

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14
    container_name: n8n_postgres
    restart: always
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=YOUR_STRONG_POSTGRES_PASSWORD  # 请替换为您的数据库密码
      - POSTGRES_DB=n8n
    volumes:
      - ./postgres_data:/var/lib/postgresql/data # 将数据库文件映射到宿主机文件，持久化保存
    networks:
      - n8n_internal_network # 使用独立的内部网络

  n8n:
    image: n8nio/n8n
    container_name: n8n_app
    restart: always
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=YOUR_STRONG_POSTGRES_PASSWORD # 请确保与上面的数据库密码一致
      - N8N_HOST=YOUR_PUBLIC_IP # 请替换为您的公网IP
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://YOUR_PUBLIC_IP/ # 请替换为您的公网IP
    ports:
      - "127.0.0.1:5678:5678"
    volumes:
      - ./n8n_data:/home/node/.n8n # n8n文件持久化存储到宿主机
    depends_on:
      - postgres
    networks:
      - n8n_internal_network

networks:
  n8n_internal_network:
    name: n8n_network # 明确指定网络名称
```

**请注意**：我们在这里明确指定了网络名称为 n8n_network，这会让接下来的步骤更简单。

在 docker-compose.yml 文件所在的目录中，启动 n8n 服务：

```bash
docker-compose up -d
```

- **第二步** **探查现有 Nginx 容器**

您需要获取两项信息：Nginx 的配置文件在宿主机上的映射路径，以及容器的名称或 ID。

执行以下命令，将 your_nginx_container_name 替换为您 Nginx 容器的实际名称或 ID：

```bash
docker inspect your_nginx_container_name
```

在输出的 JSON 信息中，找到 Mounts 部分。它看起来会像这样：

```json
"Mounts": [
    {
        "Type": "bind",
        "Source": "/path/on/your/host/nginx/conf.d", // <--- 这是您需要找的路径
        "Destination": "/etc/nginx/conf.d",
        "Mode": "",
        "RW": true,
        "Propagation": "rprivate"
    }
    // ... 可能还有其他挂载点
]
```

请记下 Source 字段的值，这就是您 Nginx 配置在服务器上的实际位置。在下面的步骤中，我们称之为 /path/to/nginx/conf.d。

- **第三步：生成自签名证书**

这一步与之前相同，但我们将证书放置在您 Nginx 配置可以访问到的地方。建议放在 Nginx 配置目录的上一层，便于管理。

```bash
# 假设您的配置目录是 /path/to/nginx/conf.d
# 我们在 /path/to/nginx/ 下创建一个 ssl 目录
mkdir -p /path/to/nginx/ssl

# 生成证书和私钥
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /path/to/nginx/ssl/n8n.key \
-out /path/to/nginx/ssl/n8n.crt
```

在提示 "Common Name" 时，依然输入您的公网 IP 地址。

**重要！**：如果您正在运行的nginx容器只映射了一个宿主机目录，比如映射了/path/to/nginx/conf.d，而没有映射/path/to/nginx/conf.d 的上层目录，那么，ssl 目录，可以创建到 /path/to/nginx/conf.d里，而非一定要放在上层目录。切记！

- **第四步：将 Nginx 容器连接到 n8n 网络**

这是关键的一步。执行以下命令，将您**正在运行的 Nginx 容器**加入到刚刚由 Docker Compose 创建的 n8n_network 中。

```bash
docker network connect n8n_network your_nginx_container_name
```

执行后，您的 Nginx 容器就同时存在于它原有的网络和 n8n_network 中了。这意味着它既可以继续为之前的服务工作，又能通过服务名 (n8n 或 n8n_app) 访问到 n8n 容器。

- **第五步：为 Nginx 添加 n8n 的配置文件**

现在，进入您在第一步中找到的宿主机路径 /path/to/nginx/conf.d。在该目录下**创建一个新文件**，例如 n8n.conf。

```bash
cd /path/to/nginx/conf.d
touch n8n.conf
```

然后编辑这个 n8n.conf 文件，填入以下内容：

```lua
server {
    listen 80;
    server_name YOUR_PUBLIC_IP; # 替换为您的公网IP

    # 仅针对 n8n 的访问重定向到 HTTPS
    # 如果您有其他域名或服务，可以添加更精确的 location 匹配
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name YOUR_PUBLIC_IP; # 替换为您的公网IP

    # 注意这里的证书路径是容器内的路径
    # 如果您在第一步中发现挂载点是 /etc/nginx/，则路径相应调整
    ssl_certificate /etc/nginx/ssl/n8n.crt;
    ssl_certificate_key /etc/nginx/ssl/n8n.key;

    location / {
        # 使用 n8n 服务的容器名作为上游地址
        # 在我们的 docker-compose.yml 中，服务名为 n8n，容器名为 n8n_app
        # 两者通常都可以，使用服务名 n8n 更符合规范
        proxy_pass http://n8n:5678;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        proxy_set_header host $host;
        proxy_set_header "Access-Control-Allow-Origin" "*";
        proxy_set_header "Access-Control-Allow-Headers" "Origin, X-Requested-With, Content-Type, Accept";
        proxy_set_header "Access-Control-Allow-Methods" "GET, POST, OPTIONS, PUT, DELETE";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}
```

**重要！**：ssl_certificate 的路径需要根据您 Nginx 容器的挂载情况来确定。如果您的 Nginx 容器挂载了 /path/to/nginx 到 /etc/nginx，那么您在宿主机上创建的 /path/to/nginx/ssl/n8n.crt 在容器内对应的路径就是 /etc/nginx/ssl/n8n.crt。

- **第六步：安全地应用 Nginx 配置**

在重启或重载之前，先在容器内测试一下新配置的语法是否正确，这是一个非常好的习惯：

```bash
docker exec your_nginx_container_name nginx -t
```

如果看到类似 syntax is ok 和 test is successful 的提示，说明配置文件没有语法错误。

然后，执行**平滑重载 (graceful reload)** 命令。这个命令会重新加载配置而**不会停止 Nginx 服务**，因此不会影响到您已经在运行的其他网站或应用。

```bash
docker exec your_nginx_container_name nginx -s reload
```

至此，全部操作完成。您现有的 Nginx 服务没有受到任何影响，并且成功地增加了对 n8n 的 HTTPS 反向代理支持。您现在可以通过 https://您的公网IP 访问 n8n 了（浏览器会提示证书不安全，需要手动信任）。

好了，开始愉快的玩耍吧！
