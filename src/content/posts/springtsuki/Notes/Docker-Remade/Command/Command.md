---
title: Docker - 常用命令实训
published: 2026-11-06
description: 用 Nginx 练习镜像管理、容器生命周期、日志、端口映射和故障定位。
tags: [Docker, 容器]
category: Docker
---

接着安装篇，在 Ubuntu 虚拟机里用 Nginx 练一遍容器操作。实验使用单独的容器名 `nginx-cli-lab`，避免动到之前用 Compose 启动的服务。

## 一、拉取镜像并启动容器

```bash
sudo docker pull nginx:alpine
sudo docker image ls nginx
sudo docker run -d --name nginx-cli-lab -p 127.0.0.1:8081:80 nginx:alpine
sudo docker ps --filter name=nginx-cli-lab
curl -fsS http://127.0.0.1:8081/
```

`pull` 下载镜像，`run` 从镜像创建并启动一个新容器。`-d` 让容器在后台运行，`--name` 给容器取名。`127.0.0.1:8081:80` 把 Ubuntu 虚拟机的本地端口 8081 映射到容器端口 80，所以在虚拟机里执行 `curl`。 [docker run 命令参考](https://docs.docker.com/reference/cli/docker/container/run/)

如果沿用安装篇的 Compose 示例，后台启动的命令是 `sudo docker compose up -d`。其中 `compose` 是子命令，`-d` 是 `up` 的选项。把 `compose` 拼成 `compuse` 时，Docker 无法按 Compose 命令解析后面的参数。

## 二、看日志、端口和容器内配置

```bash
sudo docker logs --tail 20 nginx-cli-lab
sudo docker port nginx-cli-lab
sudo docker inspect --format '{{.State.Status}}' nginx-cli-lab
sudo docker exec nginx-cli-lab nginx -t
sudo docker exec -it nginx-cli-lab sh
```

进入容器后执行 `ls /etc/nginx` 查看配置目录，再输入 `exit` 回到 Ubuntu。`exec` 在运行中的容器里启动一个额外进程。退出这次 shell，Nginx 主进程会继续运行。Alpine 镜像里的交互练习用 `sh`。

## 三、练习停止和重新启动

```bash
sudo docker stop nginx-cli-lab
sudo docker ps -a --filter name=nginx-cli-lab
sudo docker start nginx-cli-lab
curl -fsS http://127.0.0.1:8081/
```

在 `ps -a` 里记下容器 ID。重新启动后再看一次，ID 应保持一致。`start` 启动的是之前创建的容器，再执行 `run` 则会尝试创建新容器。

## 四、制造一次启动失败并定位

创建一个会打印错误并主动退出的容器：

```bash
sudo docker run -d --name nginx-exit-lab --entrypoint sh nginx:alpine -c 'echo "lab: startup failed" >&2; exit 42'
sudo docker ps -a --filter name=nginx-exit-lab
sudo docker logs nginx-exit-lab
sudo docker inspect --format '{{.State.Status}} exit={{.State.ExitCode}}' nginx-exit-lab
```

等状态变为 `Exited` 后，日志应包含 `lab: startup failed`，退出码应为 `42`。这里用 `--entrypoint` 临时换掉镜像入口来模拟启动失败。排障时先用 `ps -a` 找到已退出的容器，再读日志和退出码。已退出的容器仍可以查看日志和配置。

练完后清理：

```bash
sudo docker stop nginx-cli-lab
sudo docker rm nginx-cli-lab nginx-exit-lab
```

**验收**：能独立启动 Nginx、用 `curl` 验证映射端口，能区分 `run` 与 `start`，并能从一次启动失败里找出日志和退出码。
