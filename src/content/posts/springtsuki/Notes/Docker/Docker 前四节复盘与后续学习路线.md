---
title: Docker - 前四节复盘与后续实战路线
published: 2026-11-06
draft: true
description: 复盘 Docker 基础、Ubuntu 安装、Dockerfile 和 Compose，并用 Minishop 练习容器排障与交付。
tags: [Docker, 容器, 自动化部署]
category: Docker
---

## 1. 先导：从源码到运行中的容器

前四节可以串成一条线：用 Dockerfile 把应用和运行依赖构建成镜像，按需推送到镜像仓库，再从镜像创建容器。Minishop 最终由 Django 和 MySQL 两个容器协作，网络负责通信，卷负责保存需要跨容器重建保留的数据。

- **镜像**提供文件系统和默认启动配置。同一个镜像可以启动多个容器。
- **容器**有自己的可写层和主进程。主进程退出，容器就停止。停止后重新启动同一个容器，可写层仍在。删除容器会删除其可写层，独立管理的卷可以继续保留。
- **Docker CLI** 把请求交给 Docker daemon。Linux 容器共享宿主机的 Linux 内核，主要通过 namespace 隔离视图，通过 cgroup 管理资源。容器之间可以按网络配置通信。
- **镜像标签**方便标记版本，标签可以重新指向别的镜像。需要锁定同一份镜像内容时，可以使用 digest。

## 2. Ubuntu 部署：先区分安装、服务和拉取

原笔记走的是 Docker **Engine** 的 apt 安装流程：添加 Docker 官方仓库和签名密钥，安装 Engine、CLI、containerd、Buildx 和 Compose 插件。复建时按 Ubuntu 版本对应的官方步骤操作。 [Docker Engine 安装文档](https://docs.docker.com/engine/install/ubuntu/)

```bash
sudo systemctl status docker
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

- `docker version` 同时看到 Client 和 Server 信息，说明当前客户端能和 daemon 通信。如果只有 Client，先检查服务状态、Docker context 和 socket 权限。
- 本地没有 `hello-world` 镜像时，命令也会验证能否从仓库拉取。若连接 daemon 成功但拉取失败，根据报错分别检查宿主机 DNS、HTTPS 连通性、daemon 代理、镜像加速器和仓库鉴权。查 DNS 时传域名，例如 `getent hosts registry-1.docker.io`。
- 原笔记记录的镜像加速地址只代表当时的试验环境。若需要配置加速器，核实来源与可用性，再将 `registry-mirrors` 合并进 `/etc/docker/daemon.json`，重启 Docker 服务并重新拉取验证。 [镜像加速配置](https://docs.docker.com/docker-hub/image-library/mirror/)

## 3. Dockerfile：分清构建时和运行时

第三节用 MySQL 镜像练习 `FROM`、`ENV`、`EXPOSE` 和 `docker exec`。Minishop 的应用镜像值得自己构建。MySQL 若只需要初始化配置和 SQL，可以直接在 Compose 中引用已验证版本的官方镜像，并在运行时传入配置。

| 指令 | 用途 |
| --- | --- |
| `FROM` | 指定基础镜像。复建时记录实际验证过的 Python、MySQL 版本 |
| `WORKDIR`、`COPY` | 设置工作目录，将构建上下文中的文件放入镜像 |
| `RUN` | 构建时执行，例如安装 `requirements.txt` 中的依赖 |
| `ENV` | 给构建后续步骤或容器设置默认环境变量。密码放到运行时提供，避免写进镜像或提交到仓库 |
| `ENTRYPOINT`、`CMD` | 指定容器的入口程序和默认命令或参数。Compose 的 `command` 覆盖默认命令，`entrypoint` 用于覆盖入口 |
| `EXPOSE` | 记录容器服务端口。宿主机访问需要显式发布端口，例如 `docker run -p 8000:8000` |

构建命令写作 `docker build -t minishop:dev .`，其中 `.` 是构建上下文。先复制依赖清单并安装，再复制源码。只改业务代码后重建，可以观察依赖安装层是否命中缓存。用 `.dockerignore` 排除虚拟环境、Git 元数据、本地密码文件和缓存。 [Dockerfile 参考](https://docs.docker.com/reference/dockerfile/) · [构建缓存](https://docs.docker.com/learn/content/how-image-layers-and-the-build-cache-work/)

原例的 `chmod +x /initsh` 路径应与实际复制的 `/init.sh` 一致。如果保留入口脚本，让脚本在初始化失败时退出，并用 `exec "$@"` 启动传入的主进程。这样 Compose 的 `command` 才能按预期传给入口脚本，服务进程也能直接收到停止信号。

## 4. Compose：让 Django 和 MySQL 一起运行

第四节完成了 Minishop 的联调。复建时可沿用 `app` 和 `db` 两个服务，重点验证以下行为。

- **网络**：Django 在 Compose 网络内通过 `db:3306` 连接 MySQL。Django 配置键是 `PORT`。应用监听 `0.0.0.0:8000`，宿主机通过发布的端口访问。若只需要 Django 访问 MySQL，数据库无需向宿主机发布端口。 [Compose 网络](https://docs.docker.com/compose/how-tos/networking/)
- **启动顺序**：普通 `depends_on` 只管启动顺序。为数据库配置能反映就绪状态的 `healthcheck`，再用 `condition: service_healthy` 让 Django 等待。数据库运行中重启后，应用仍需要具备重连能力。 [Compose 启动顺序](https://docs.docker.com/compose/how-tos/startup-order/)
- **数据与初始化**：用命名卷挂载 MySQL 的 `/var/lib/mysql`。挂载到 `/docker-entrypoint-initdb.d/` 的脚本在数据库数据目录首次初始化时执行。修改 SQL 后重启已有数据卷，脚本通常不会重跑。Django 的迁移文件应随代码管理，部署时执行 `python manage.py migrate`。模型变更时再生成并提交迁移。
- **权限和配置**：给 Django 配置专用数据库账号和目标库，避免沿用远程 root。旧笔记的 `allowPublicKeyRetrieval` 属于 JDBC 连接参数，Django 应按所用的 Python MySQL 驱动配置连接选项。把整个源码目录挂到 `/app` 会覆盖镜像里该路径的内容，复现镜像交付时先去掉这种开发挂载。

```bash
docker compose config --quiet
docker compose up --build -d
docker compose ps -a
docker compose logs --tail=100 app db
docker compose exec db mysql -u <应用数据库账号> -p
```

最后一条命令中的账号占位符需换成实际值。检查持久化时，先用测试数据写入一条记录，重建数据库容器，再查询该记录。演练时保留命名卷。

## 后续学习路线

| 顺序 | 学什么 | 用 Minishop 验收 |
| --- | --- | --- |
| 5. 容器生命周期与排障 | `run --name -d --rm` 的用途，`ps -a`、`logs`、`inspect`、`exec`、`stats`、`stop`、`rm` | 故意写错启动命令或让进程退出。用 `ps -a` 找容器，结合日志和 `inspect` 确认退出码，再修复并验证 HTTP 请求 |
| 6. 网络与存储 | bridge、服务名解析、容器端口和宿主机端口、绑定挂载与命名卷 | 分别改错应用监听地址和数据库主机名。从宿主机验证 HTTP，从应用容器验证 DNS 和 MySQL 连接，最后验证数据库容器重建后数据还在 |
| 7. 镜像构建与发布 | 构建缓存、`.dockerignore`、版本标签、仓库 push/pull | 改一行源码后检查构建缓存。发布两个版本的应用镜像，用拉取的镜像部署，再切回上一版本并验证。只用兼容现有数据库结构的两个版本练镜像回滚 |
| 8. Compose 可复现部署 | 运行时配置、健康检查、迁移和日志 | 在一套新的测试环境里按 README 跑通启动、迁移、页面访问和一次数据库连接故障的恢复，记录现象、证据、根因和恢复验证 |
| 9. 迁到 K8s | Deployment、Service、ConfigMap、Secret、readiness probe、requests/limits、滚动更新和回滚 | 沿用同一个 Django 镜像，先连接实验集群可达的独立 MySQL。发布一个会导致 Pod 无法就绪的版本，通过事件和日志定位，再回滚验证服务恢复 |

第一轮可以从第 5 节开始。用原项目做三次故障演练：启动命令错误、容器运行但宿主机访问失败、Django 连不上数据库。能独立定位这三类问题，并从源码构建出可部署的镜像，就可以开始迁 K8s。面试材料优先留下可复现的 README、发版和回滚步骤，以及两三份有命令和验证结果的故障记录。
