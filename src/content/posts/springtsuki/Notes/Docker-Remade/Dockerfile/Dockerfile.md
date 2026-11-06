---
title: Docker - Dockerfile 编写实训
published: 2026-11-06
description: 从源码构建 Python HTTP 服务镜像，练习构建上下文、缓存、启动参数和端口排障。
tags: [Docker, 容器]
category: Docker
---

这一篇先用一个只依赖 Python 标准库的 HTTP 服务练手。构建、启动和排障跑通后，再把同样的步骤用到 Minishop。

## 一、准备构建目录

```bash
mkdir -p ~/Docker/DockerfileLab
cd ~/Docker/DockerfileLab
```

创建 `index.html`：

```html
<!doctype html>
<html lang="zh-CN">
  <meta charset="utf-8">
  <title>Dockerfile 实训</title>
  <h1>Dockerfile lab v1</h1>
</html>
```

创建 `.dockerignore`：

```text
.git
.venv
.env
__pycache__/
```

构建命令末尾的 `.` 指当前目录，它是 Docker 构建时可访问的文件范围。`.dockerignore` 用于把匹配的文件排除在构建上下文之外。 [Dockerfile 参考](https://docs.docker.com/reference/dockerfile/)

## 二、编写 Dockerfile

创建名为 `Dockerfile` 的文件：

```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN mkdir public
COPY index.html public/
EXPOSE 8000
CMD ["python", "-m", "http.server", "8000", "--bind", "0.0.0.0", "--directory", "public"]
```

- `FROM` 选择基础镜像，`WORKDIR` 设置后续指令和容器启动时的工作目录。
- `RUN` 在构建时执行，`COPY` 将构建上下文中的文件放进镜像。
- `EXPOSE` 记录服务端口。要从 Ubuntu 虚拟机访问，启动时仍要显式用 `-p` 发布端口。
- `CMD` 指定容器的默认启动命令。这里用 JSON 数组形式直接启动 Python 进程。Python 的 `http.server` 用于本次本地实验。

## 三、构建镜像并验证页面

```bash
sudo docker build --progress=plain -t dockerfile-lab:v1 .
sudo docker run -d --name dockerfile-lab-v1 -p 127.0.0.1:8082:8000 dockerfile-lab:v1
curl -fsS http://127.0.0.1:8082/
```

看到页面里的 `Dockerfile lab v1`，说明镜像里的文件已由容器内的服务对外提供。这里把虚拟机的 8082 端口映射到了容器内的 8000 端口。

## 四、改一行源码，观察构建缓存

把 `index.html` 的标题正文改成 `Dockerfile lab v2`，再构建：

```bash
sudo docker build --progress=plain -t dockerfile-lab:v2 .
sudo docker rm -f dockerfile-lab-v1
sudo docker run -d --name dockerfile-lab-v2 -p 127.0.0.1:8082:8000 dockerfile-lab:v2
curl -fsS http://127.0.0.1:8082/
```

对比两次构建日志。基础镜像和前置步骤一致时，`RUN mkdir public` 可以复用缓存，`COPY` 会因 HTML 内容变化而重新执行。迁到 Minishop 时，把 `requirements.txt` 的复制和依赖安装放在复制业务源码之前，就能用同样的方法观察缓存。 [Docker 构建缓存](https://docs.docker.com/build/cache/)

## 五、让容器在运行但访问失败

先删除本次实验的 v2 容器，再在镜像名后传入完整命令，覆盖默认 `CMD`：

```bash
sudo docker rm -f dockerfile-lab-v2
sudo docker run -d --name dockerfile-bind-lab -p 127.0.0.1:8082:8000 dockerfile-lab:v2 \
  python -m http.server 8000 --bind 127.0.0.1 --directory public

sudo docker ps --filter name=dockerfile-bind-lab
sudo docker port dockerfile-bind-lab
curl -fsS http://127.0.0.1:8082/
sudo docker exec dockerfile-bind-lab python -c 'import urllib.request; print(urllib.request.urlopen("http://127.0.0.1:8000/").status)'
```

预期容器仍在运行，宿主侧的 `curl` 失败，而容器内的请求返回 `200`。端口映射指向容器的网络接口，服务只监听容器自己的回环地址时，映射进来的流量到不了服务。

恢复默认启动命令并验证：

```bash
sudo docker rm -f dockerfile-bind-lab
sudo docker run -d --name dockerfile-lab-v2 -p 127.0.0.1:8082:8000 dockerfile-lab:v2
curl -fsS http://127.0.0.1:8082/
sudo docker stop dockerfile-lab-v2
sudo docker rm dockerfile-lab-v2
```

**验收**：能从头写出 Dockerfile，说明哪条指令在构建时执行、哪条决定容器启动方式，能从构建日志确认缓存命中，并能用容器内外的请求定位监听地址错误。
