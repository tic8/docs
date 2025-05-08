# Docker + Go + AWS SDK 构建与部署兼容性分析文档

## 背景

在项目使用 Docker 构建 Go 应用时，通过 `--platform=linux/amd64` 设置构建平台，在绝大多数功能中运行正常。但在部署到 AWS EC2 并启用默认 IAM Role 时，遇到如下错误：

```text
failed to refresh cached credentials, no EC2 IMDS role found
```

该问题与构建出的二进制兼容性有关。

---

## 问题分析

### 1. `--platform=linux/amd64` 的作用范围

Docker 的 `--platform=linux/amd64` 主要作用于以下场景：

* 指定 **镜像拉取平台**（避免拉取 ARM 镜像）
* 指定 **容器运行平台**（适配宿主机 QEMU）

❗️它**不会自动影响 Go build 的目标平台**。

### 2. Go 的交叉编译行为

Go 默认使用当前容器或主机的架构进行编译。

在基于 ARM 主机（如 Apple Silicon）或跨平台 Docker 构建时，若未设置：

```bash
GOOS=linux GOARCH=amd64
```

编译出的二进制仍可能是 **arm64 架构**，即使镜像是 x86 的。

### 3. AWS SDK 的特殊性

AWS SDK，尤其是默认凭证提供器链（如 EC2 IAM Role），会尝试访问：

```http
http://169.254.169.254/latest/meta-data/iam/info
```

该过程依赖：

* 系统 DNS 解析
* `/proc` 文件系统
* Go 的底层 `net/http`, `net` 库
* glibc / CGO 的兼容性

如果使用了非目标架构（如 ARM64）的二进制运行在 x86 上，或 CGO 环境异常，就可能导致 metadata 获取失败。

---

## 兼容性矩阵

| 场景                             | `--platform` 设置 | `GOARCH` 设置 | 是否兼容 AWS SDK | 备注                     |
| ------------------------------ | --------------- | ----------- | ------------ | ---------------------- |
| Docker ARM 主机，不设 `GOARCH`      | ✅               | ❌           | ❌            | 构建出的为 arm64，可运行但部分功能失效 |
| Docker x86 主机，不设 `GOARCH`      | ✅               | ✅（隐式）       | ✅            | 默认编译为 amd64，兼容性好       |
| Docker 任意主机，显式设 `GOARCH=amd64` | ✅               | ✅           | ✅            | 推荐做法                   |

---

## 最佳实践（Dockerfile 示例）

```dockerfile
FROM --platform=linux/amd64 public.ecr.aws/o5b3f9w7/gdex-builder:latest AS builder

WORKDIR /app

COPY . .

# 显式设置目标平台，确保兼容性
ENV GOOS=linux
ENV GOARCH=amd64

RUN CGO_ENABLED=1 \
    CGO_LDFLAGS="-L/usr/local/lib -lrocksdb -lstdc++ -lm -lz -lsnappy -llz4 -lzstd -lbz2" \
    go build -tags "ec2 netgo" -ldflags "-s -w" -o /app/bin/myapp main.go

FROM --platform=linux/amd64 ubuntu:22.04
RUN apt-get update && apt-get install -y libsnappy-dev ca-certificates

COPY --from=builder /app/bin/myapp /app/myapp

ENTRYPOINT ["/app/myapp"]
```

---

## 检查方法

### 检查构建出的二进制架构：

```bash
file ./myapp
```

应输出：

```text
ELF 64-bit LSB executable, x86-64, ...
```

### 检查运行容器平台：

```bash
uname -m
```

应输出：

```text
x86_64
```

---

## 总结

在基于 Docker 的构建链中：

* **`--platform` 仅影响镜像拉取与容器运行，不影响 Go 构建目标架构**
* **必须显式设置 `GOOS=linux GOARCH=amd64` 保证构建的二进制在 EC2 等环境上完全兼容**
* **AWS SDK 比其他功能更容易暴露底层架构或 libc 问题**

正确设置之后，服务可无障碍访问 EC2 Metadata Service 并使用 IAM Role 凭证进行上传等操作。
