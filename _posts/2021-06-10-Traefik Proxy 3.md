---
layout: post
title:  "项目基础知识培训三"
date:   2021-06-10
tags:
    - 项目
    - 培训
---

如果，您已经按照[项目基础知识培训二](/2021/06/06/Docker-Swarm-2/)中的描述设置了一个**Docker Swarm**集群。

现在你可以添加一个[**Traefik**](https://traefik.io/)用来：

*   处理**连接**。
*   根据域名**公开**特定服务和应用程序。
*   处理**多个域**（如果需要）。它类似于“虚拟主机”。
*   处理**HTTPS**。
*   使用[Let's Encrypt](https://letsencrypt.org/)**自动**获取（生成）**HTTPS 证书**（包括续订）。
*   为您需要保护且没有自身安全性的任何服务添加 HTTP**基本**身份**验证**等。
*   从堆栈中设置的**Docker 标签中**自动获取其所有配置（您不需要更新配置文件）。

这些办法、技术和工具也适用于其他集群编排，如 Kubernetes 或 Mesos，以添加具有 HTTPS 支持、证书生成等功能的负载均衡。但本文主要关注 Docker Swarm 模式。

介绍
---

这个想法是，用一个负载均衡/代理来覆盖所有的 Docker Swarm 集群，并处理每个域的 HTTPS 证书和请求。

但是这样做的方式允许您在每个堆栈中拥有其他 Traefik 服务而不会相互干扰，基于同一堆栈中的路径进行重定向（例如，一个`/`用于 Web 前端的容器句柄和`/api`用于同一 API 下的另一个句柄），或者有选择地从 HTTP 重定向到 HTTPS。

准备
---

*   通过 SSH 连接到集群中用来安装 Traefik 服务的管理器节点（您可能只有一个节点）。
*   创建一个 Traefik 专用网络，使它可以从外部网络访问容器共享的网络，如下：

```s
docker network create  --driver=overlay traefik-public
```

*   获取此节点的 Swarm 节点 ID 并将其存储在环境变量中：

```s
export NODE_ID=$( docker info -f '{{.Swarm.NodeID}}' )
```

*   在这个节点中创建一个标签，让 Traefik 始终部署到同一个节点并使用同一个卷：

```s
docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID
```

*   使用您的电子邮件创建一个环境变量，用于生成 Let's Encrypt 证书，例如：

```s
export EMAIL=admin@example.com
```

*   使用要用于 Traefik UI（用户界面）的域名创建一个环境变量，例如：

```s
export DOMAIN=traefik.sys.example.com
```

*   您将在此域中访问 Traefik 仪表板，例如`traefik.sys.example.com`. 因此，请确保您的 DNS 记录将域指向集群的 IP 之一。如果它是 Traefik 服务运行的 IP（您当前连接到的管理器节点），则更好。
    
*   创建一个用户名环境变量（您将在 Traefik 和 Consul UI 的 HTTP 基本身份验证中使用它），例如：
    
```s
export USERNAME=admin
```
*   创建一个密码环境变量，例如：

```s
export PASSWORD=changethis
```

*   使用`openssl`生成密码的“散列”版本，并将其存储在一个环境变量：

```s
export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
```

**（可选）**：或者，如果您不想将密码放在环境变量中，您想交互输入，例如：

```s
export HASHED_PASSWORD=$(openssl passwd -apr1)
```

*   您可以通过以下方式检查以上变量的内容：

它看起来像：

```s
echo $HASHED_PASSWORD
```

创建
---

*   下载文件`traefik.yml`：

```s
curl -L dockerswarm.rocks/traefik.yml -o traefik.yml
```

*   ...或手动创建它，例如，使用`nano`：

```s
nano traefik.yml
```

*   并复制下面的内容：

```yml
version: '3.3'

services:

  traefik:
    # Use the latest v2.2.x Traefik image available
    image: traefik:v2.2
    ports:
      # Listen on port 80, default for HTTP, necessary to redirect to HTTPS
      - 80:80
      # Listen on port 443, default for HTTPS
      - 443:443
    deploy:
      placement:
        constraints:
          # Make the traefik service run only on the node with this label
          # as the node with it has the volume for the certificates
          - node.labels.traefik-public.traefik-public-certificates == true
      labels:
        # Enable Traefik for this service, to make it available in the public network
        - traefik.enable=true
        # Use the traefik-public network (declared below)
        - traefik.docker.network=traefik-public
        # Use the custom label "traefik.constraint-label=traefik-public"
        # This public Traefik will only use services with this label
        # That way you can add other internal Traefik instances per stack if needed
        - traefik.constraint-label=traefik-public
        # admin-auth middleware with HTTP Basic auth
        # Using the environment variables USERNAME and HASHED_PASSWORD
        - traefik.http.middlewares.admin-auth.basicauth.users=${USERNAME?Variable not set}:${HASHED_PASSWORD?Variable not set}
        # https-redirect middleware to redirect HTTP to HTTPS
        # It can be re-used by other stacks in other Docker Compose files
        - traefik.http.middlewares.https-redirect.redirectscheme.scheme=https
        - traefik.http.middlewares.https-redirect.redirectscheme.permanent=true
        # traefik-http set up only to use the middleware to redirect to https
        # Uses the environment variable DOMAIN
        - traefik.http.routers.traefik-public-http.rule=Host(`${DOMAIN?Variable not set}`)
        - traefik.http.routers.traefik-public-http.entrypoints=http
        - traefik.http.routers.traefik-public-http.middlewares=https-redirect
        # traefik-https the actual router using HTTPS
        # Uses the environment variable DOMAIN
        - traefik.http.routers.traefik-public-https.rule=Host(`${DOMAIN?Variable not set}`)
        - traefik.http.routers.traefik-public-https.entrypoints=https
        - traefik.http.routers.traefik-public-https.tls=true
        # Use the special Traefik service api@internal with the web UI/Dashboard
        - traefik.http.routers.traefik-public-https.service=api@internal
        # Use the "le" (Let's Encrypt) resolver created below
        - traefik.http.routers.traefik-public-https.tls.certresolver=le
        # Enable HTTP Basic auth, using the middleware created above
        - traefik.http.routers.traefik-public-https.middlewares=admin-auth
        # Define the port inside of the Docker service to use
        - traefik.http.services.traefik-public.loadbalancer.server.port=8080
    volumes:
      # Add Docker as a mounted volume, so that Traefik can read the labels of other services
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Mount the volume to store the certificates
      - traefik-public-certificates:/certificates
    command:
      # Enable Docker in Traefik, so that it reads labels from Docker services
      - --providers.docker
      # Add a constraint to only use services with the label "traefik.constraint-label=traefik-public"
      - --providers.docker.constraints=Label(`traefik.constraint-label`, `traefik-public`)
      # Do not expose all Docker services, only the ones explicitly exposed
      - --providers.docker.exposedbydefault=false
      # Enable Docker Swarm mode
      - --providers.docker.swarmmode
      # Create an entrypoint "http" listening on port 80
      - --entrypoints.http.address=:80
      # Create an entrypoint "https" listening on port 443
      - --entrypoints.https.address=:443
      # Create the certificate resolver "le" for Let's Encrypt, uses the environment variable EMAIL
      - --certificatesresolvers.le.acme.email=${EMAIL?Variable not set}
      # Store the Let's Encrypt certificates in the mounted volume
      - --certificatesresolvers.le.acme.storage=/certificates/acme.json
      # Use the TLS Challenge for Let's Encrypt
      - --certificatesresolvers.le.acme.tlschallenge=true
      # Enable the access log, with HTTP requests
      - --accesslog
      # Enable the Traefik log, for configurations and errors
      - --log
      # Enable the Dashboard and API
      - --api
    networks:
      # Use the public network created to be shared between Traefik and
      # any other service that needs to be publicly available with HTTPS
      - traefik-public

volumes:
  # Create a volume to store the certificates, there is a constraint to make sure
  # Traefik is always deployed to the same Docker node with the same volume containing
  # the HTTPS certificates
  traefik-public-certificates:

networks:
  # Use the previously created public network "traefik-public", shared with other
  # services that need to be publicly available via this Traefik
  traefik-public:
    external: true
```

提示：
```t
这只是一个标准的 Docker Compose 文件。

可使用常见的命名文件`docker-compose.yml`或`docker-compose.traefik.yml`。

这里命名为`traefik.yml`是为了简洁。
```
部署
---

使用以下命令部署堆栈：

```s
docker stack deploy -c traefik.yml traefik
```

它将使用您在上面创建的环境变量。

检查
---

*   检查堆栈是否已部署：

```s
docker stack ps traefik
```
*  它会输出如下内容：

```t
ID             NAME                IMAGE          NODE              DESIRED STATE   CURRENT STATE          ERROR   PORTS
w5o6fmmln8ni   traefik_traefik.1   traefik:v2.2   dog.example.com   Running         Running 1 minute ago
```

*   您可以使用以下命令检查 Traefik 日志：

```s
docker service logs traefik_traefik
```

界面
---

运行完成后，Traefik 将获取 Web 用户界面 (UI) 的 HTTPS 证书。

您将能够直接访问`https://traefik.<your domain>`，并使用上面创建的用户名和密码安全地访问 Web UI 。

部署堆栈后，您将看到它，并了解不同的主机和路径如何映射到不同的 Docker 服务/容器。

获取客户端IP
---

如果您需要使用Traefik 提供的`X-Forwarded-For`或`X-Real-IP`标头读取应用程序/堆栈中的客户端 IP ，则需要让 Traefik 直接侦听，而不是通过 Docker Swarm 模式，即使在使用 Docker Swarm 模式部署时也是如此。

为此，您需要使用“host”模式发布端口。

因此，Docker Compose 文件内需要把：

```yaml
    ports:
      - 80:80
      - 443:443
```

修改成：

```yaml
    ports:
      - target: 80
        published: 80
        mode: host
      - target: 443
        published: 443
        mode: host
```

您可以直接下载主机模式文件：

```s
curl -L dockerswarm.rocks/traefik-host.yml -o traefik-host.yml
```

或者，复制它：

```yml
version: '3.3'

services:

  traefik:
    # Use the latest Traefik image
    image: traefik:v2.2
    ports:
      # Listen on port 80, default for HTTP, necessary to redirect to HTTPS
      - target: 80
        published: 80
        mode: host
      # Listen on port 443, default for HTTPS
      - target: 443
        published: 443
        mode: host
    deploy:
      placement:
        constraints:
          # Make the traefik service run only on the node with this label
          # as the node with it has the volume for the certificates
          - node.labels.traefik-public.traefik-public-certificates == true
      labels:
        # Enable Traefik for this service, to make it available in the public network
        - traefik.enable=true
        # Use the traefik-public network (declared below)
        - traefik.docker.network=traefik-public
        # Use the custom label "traefik.constraint-label=traefik-public"
        # This public Traefik will only use services with this label
        # That way you can add other internal Traefik instances per stack if needed
        - traefik.constraint-label=traefik-public
        # admin-auth middleware with HTTP Basic auth
        # Using the environment variables USERNAME and HASHED_PASSWORD
        - traefik.http.middlewares.admin-auth.basicauth.users=${USERNAME?Variable not set}:${HASHED_PASSWORD?Variable not set}
        # https-redirect middleware to redirect HTTP to HTTPS
        # It can be re-used by other stacks in other Docker Compose files
        - traefik.http.middlewares.https-redirect.redirectscheme.scheme=https
        - traefik.http.middlewares.https-redirect.redirectscheme.permanent=true
        # traefik-http set up only to use the middleware to redirect to https
        # Uses the environment variable DOMAIN
        - traefik.http.routers.traefik-public-http.rule=Host(`${DOMAIN?Variable not set}`)
        - traefik.http.routers.traefik-public-http.entrypoints=http
        - traefik.http.routers.traefik-public-http.middlewares=https-redirect
        # traefik-https the actual router using HTTPS
        # Uses the environment variable DOMAIN
        - traefik.http.routers.traefik-public-https.rule=Host(`${DOMAIN?Variable not set}`)
        - traefik.http.routers.traefik-public-https.entrypoints=https
        - traefik.http.routers.traefik-public-https.tls=true
        # Use the special Traefik service api@internal with the web UI/Dashboard
        - traefik.http.routers.traefik-public-https.service=api@internal
        # Use the "le" (Let's Encrypt) resolver created below
        - traefik.http.routers.traefik-public-https.tls.certresolver=le
        # Enable HTTP Basic auth, using the middleware created above
        - traefik.http.routers.traefik-public-https.middlewares=admin-auth
        # Define the port inside of the Docker service to use
        - traefik.http.services.traefik-public.loadbalancer.server.port=8080
    volumes:
      # Add Docker as a mounted volume, so that Traefik can read the labels of other services
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Mount the volume to store the certificates
      - traefik-public-certificates:/certificates
    command:
      # Enable Docker in Traefik, so that it reads labels from Docker services
      - --providers.docker
      # Add a constraint to only use services with the label "traefik.constraint-label=traefik-public"
      - --providers.docker.constraints=Label(`traefik.constraint-label`, `traefik-public`)
      # Do not expose all Docker services, only the ones explicitly exposed
      - --providers.docker.exposedbydefault=false
      # Enable Docker Swarm mode
      - --providers.docker.swarmmode
      # Create an entrypoint "http" listening on address 80
      - --entrypoints.http.address=:80
      # Create an entrypoint "https" listening on address 443
      - --entrypoints.https.address=:443
      # Create the certificate resolver "le" for Let's Encrypt, uses the environment variable EMAIL
      - --certificatesresolvers.le.acme.email=${EMAIL?Variable not set}
      # Store the Let's Encrypt certificates in the mounted volume
      - --certificatesresolvers.le.acme.storage=/certificates/acme.json
      # Use the TLS Challenge for Let's Encrypt
      - --certificatesresolvers.le.acme.tlschallenge=true
      # Enable the access log, with HTTP requests
      - --accesslog
      # Enable the Traefik log, for configurations and errors
      - --log
      # Enable the Dashboard and API
      - --api
    networks:
      # Use the public network created to be shared between Traefik and
      # any other service that needs to be publicly available with HTTPS
      - traefik-public

volumes:
  # Create a volume to store the certificates, there is a constraint to make sure
  # Traefik is always deployed to the same Docker node with the same volume containing
  # the HTTPS certificates
  traefik-public-certificates:

networks:
  # Use the previously created public network "traefik-public", shared with other
  # services that need to be publicly available via this Traefik
  traefik-public:
    external: true
```

然后进行部署：

```s
docker stack deploy -c traefik-host.yml traefik
```

后面的课程介绍
---

接下来是使用这个 Docker Swarm 模式集群部署一个堆栈（一个完整的 Web 应用程序，具有后端、前端、数据库等）。

其实很简单，你可以使用Docker Compose进行本地开发，然后在Docker Swarm模式集群中使用相同的文件进行部署。

如果您想立即尝试，可以使用 Vue.js [https://github.com/tiangolo/full-stack-fastapi-postgresql](https://github.com/tiangolo/full-stack-fastapi-postgresql)和 FastAPI。