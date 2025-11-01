# 使用步骤（使用本地 Docker 镜像包运行 KingbaseES V9）

本文档面向已下载官方 Docker 镜像包（.tar）和对应 license 的用户，提供从“导入镜像 → 运行容器 → 验证/连接 → 常见问题”的完整流程。

## 环境准备

- 已安装 Docker Desktop（macOS）
- 在本项目目录内操作（路径位于 `/Users/...` 下，方便挂载）
- 目录中已存在：
	- `KingbaseES_V009R001C010B0004_x86_64_Docker.tar`
	- `KingbaseES_V009R001C010B0004_aarch64_Docker.tar`
	- 一个 V009R001C 对应的 license 文件（例如：`license_V009R001C-开发版.dat`，或标准版/专业版/企业版之一）

可选：清理可能占用端口或重名的旧容器

```bash
docker ps
docker rm -f kingbase kingbase9 2>/dev/null || true
```

## 一、导入本地镜像（docker load）

建议两个镜像都导入，后续按机器架构选择使用：

```bash
# x86_64（Intel/AMD，或 Apple Silicon 需要 x86 兼容时）
docker load -i KingbaseES_V009R001C010B0004_x86_64_Docker.tar

# aarch64（Apple Silicon 原生优先使用）
docker load -i KingbaseES_V009R001C010B0004_aarch64_Docker.tar

# 检查是否成功加载镜像（关注 kingbase 相关行）
docker images | grep -i kingbase
```

常见加载后的镜像名与标签（以你本机实际输出为准）：

- `kingbase_v009r001c010b0004_single_arm:v1`（aarch64）
- `kingbase_v009r001c010b0004_single_x86:v1`（x86_64）

## 二、准备数据目录与 license

```bash
# 使用项目目录作为数据持久化位置（避免 macOS mounts denied）
mkdir -p ./data9
```

说明：
- `./data9` 用于持久化数据库数据；
- 仅挂载一个对应版本的 license 文件到容器内 `/opt/kingbase/Server/bin/license.dat`；
- 建议以只读方式挂载 license（`:ro`）。

## 三、运行容器

根据你的机器架构选择对应镜像，端口冲突可调整左侧宿主机端口（示例用 56432）。

### 1) Apple Silicon (M1/M2/M3)：优先使用 aarch64 镜像

```bash
docker run -d \
	--name kingbase9 \
	-p 56432:54321 \
	-e SYSTEM_PWD=SYSTEM \
	-v "$PWD/data9":/opt/kingbase/data \
	-v "$PWD/license_V009R001C-开发版.dat":/opt/kingbase/Server/bin/license.dat:ro \
	kingbase_v009r001c010b0004_single_arm:v1
```

### 2) Intel/AMD x86_64 机器：使用 x86_64 镜像

```bash
docker run -d \
	--name kingbase9 \
	-p 54321:54321 \
	-e SYSTEM_PWD=SYSTEM \
	-v "$PWD/data9":/opt/kingbase/data \
	-v "$PWD/license_V009R001C-开发版.dat":/opt/kingbase/Server/bin/license.dat:ro \
	kingbase_v009r001c010b0004_single_x86:v1
```

### 3) Apple Silicon 但必须使用 x86_64 镜像（仿真运行）

```bash
docker run -d \
	--platform linux/amd64 \
	--name kingbase9 \
	-p 57432:54321 \
	-e SYSTEM_PWD=SYSTEM \
	-v "$PWD/data9":/opt/kingbase/data \
	-v "$PWD/license_V009R001C-开发版.dat":/opt/kingbase/Server/bin/license.dat:ro \
	kingbase_v009r001c010b0004_single_x86:v1
```

提示：
- 使用双引号包裹 `$PWD/...` 以兼容路径中存在空格（zsh/bash 通用）。
- macOS 仅允许挂载 `/Users`、`/Volumes`、`/private`、`/tmp` 等路径，推荐使用项目目录路径进行挂载。

## 四、验证启动

```bash
docker logs -n 100 -f kingbase9
```

预期：初始化完成后容器保持运行，日志显示数据库已启动并监听容器内端口 `54321`。

## 五、连接数据库

- 端口：使用映射后的宿主机端口（示例：56432 / 54321 / 57432）
- 用户名：`SYSTEM`
- 密码：由 `-e SYSTEM_PWD=...` 传入（示例中为 `SYSTEM`）
- 客户端：可使用 Kingbase 官方客户端或 JDBC 驱动

可进入容器查看可用工具：

```bash
docker exec -it kingbase9 bash
ls /opt/kingbase/Server/bin
```

## 六、停止/启动/删除容器（数据可复用）

```bash
# 停止
docker stop kingbase9

# 再次启动（复用同一数据目录）
docker start kingbase9

# 删除容器（不会删除 ./data9 数据）
docker rm -f kingbase9
```

## 常见问题排查（FAQ）

- mounts denied（挂载失败）
	- 使用项目目录（位于 `/Users` 下）进行挂载，例如 `-v "$PWD/data9":/opt/kingbase/data`。
- 端口被占用
	- 更换宿主机端口（例如 `-p 56432:54321`）。
- Apple Silicon 上出现 rosetta/ELF 错误
	- 优先使用 aarch64 镜像；若必须使用 x86 镜像，在 `docker run` 时添加 `--platform linux/amd64`。
- license 导致连接数限制（如提示 superuser_reserved_connections must be less than max_connections）
	- 降低数据目录下 `kingbase.conf` 中相关参数，或更换更高授权的 license 文件。
- 权限问题导致初始化失败
	- 确保宿主机对挂载目录有读写权限，或使用 Docker 卷替代本地目录。

## 参考链接

- [Docker安装人大金仓（电科金仓）KingbaseES](https://juejin.cn/post/7510277359401271332)