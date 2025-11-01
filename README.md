# 电科金仓Kingbase数据库

如果您想自己构建镜像可参照以下操作：

```bash
git clone https://github.com/renfei/kingbase-es-v8-r3-docker.git
cd kingbase-es-v8-r3-docker
docker build -t kingbase:v8r3 .
```

## 运行

```bash
docker run -d --name kingbase -p 54321:54321 -e SYSTEM_PWD=SYSTEM -v /opt/kingbase/data:/opt/kingbase/data -v /opt/kingbase/license.dat:/opt/kingbase/Server/bin/license.dat kingbase:v8r3
```

- --name: 容器名称
- -p: 端口映射
- -e: 默认用户SYSTEM,通过环境变量SYSTEM_PWD指定初始化数据库时的默认用户密码
- -v: 挂载宿主机的一个目录，这里挂载了数据目录和license文件

### macOS 注意事项（挂载路径）

在 macOS 上通过 Docker Desktop 运行容器时，默认仅允许挂载位于 `/Users`、`/Volumes`、`/private`、`/tmp` 等目录下的路径。如果直接挂载宿主机的 `/opt/...` 路径会出现 mounts denied 错误。

有两种解决方式，推荐第 1 种：

1) 使用当前项目目录下的路径进行挂载（推荐）

```bash
# 在项目根目录下创建数据目录
mkdir -p ./data

# 使用当前目录（$PWD）进行挂载，确保路径位于 /Users 下
docker run -d \
	--name kingbase \
	-p 54321:54321 \
	-e SYSTEM_PWD=SYSTEM \
	-v "$PWD/data":/opt/kingbase/data \
	-v "$PWD/license.dat":/opt/kingbase/Server/bin/license.dat:ro \
	kingbase:v8r3
```

说明：
- `./data` 会持久化数据库数据；
- 仓库根目录已包含 `license.dat`，以只读方式挂载到容器中的 `/opt/kingbase/Server/bin/license.dat`；
- 使用双引号包裹 `$PWD/...` 以兼容路径中包含空格的情况（zsh/bash 皆适用）。

2) 在 Docker Desktop 中共享 `/opt` 路径（可选）

- 打开 Docker Desktop → Settings → Resources → File sharing；
- 添加 `/opt/kingbase`（或 `/opt`）到共享列表并应用；
- 然后可以继续使用原始命令：

```bash
docker run -d --name kingbase -p 54321:54321 -e SYSTEM_PWD=SYSTEM -v /opt/kingbase/data:/opt/kingbase/data -v /opt/kingbase/license.dat:/opt/kingbase/Server/bin/license.dat kingbase:v8r3
```

如果仍遇到权限问题，建议优先采用方案 1，或将挂载的宿主机目录的属主/权限进行适配。

### Apple Silicon (M1/M2/M3) 架构说明（必须指定 amd64）

Kingbase 提供的二进制为 x86_64 架构，Apple Silicon 默认拉取 arm64 基础镜像与运行架构，直接运行会出现如下错误：

```
rosetta error: failed to open elf at /lib64/ld-linux-x86-64.so.2
Trace/breakpoint trap ./initdb
```

解决：构建与运行均指定 `linux/amd64`：

```bash
# 方式一：使用 buildx 构建并加载到本地
docker buildx build --platform linux/amd64 -t kingbase:v8r3-amd64 . --load

# 运行时同样指定平台（结合上面的挂载示例）
docker run -d \
	--platform linux/amd64 \
	--name kingbase \
	-p 54321:54321 \
	-e SYSTEM_PWD=SYSTEM \
	-v "$PWD/data":/opt/kingbase/data \
	-v "$PWD/license.dat":/opt/kingbase/Server/bin/license.dat:ro \
	kingbase:v8r3-amd64

# 方式二：临时设定默认平台后用经典 build
export DOCKER_DEFAULT_PLATFORM=linux/amd64
docker build -t kingbase:v8r3-amd64 .
```

此外，本仓库的 `Dockerfile` 已固定为 `FROM --platform=linux/amd64 centos:7`，确保在 Apple Silicon 上也会基于 x86_64 基础镜像构建。

## 常见问题
### 启动失败
- 启动失败，日志报 kingbase: superuser_reserved_connections must be less than max_connections
- 原因：本仓库中的 license.dat 文件是开发测试版，限制最大连接数为10，而人大金仓配置文件默认连接数为100，导致启动失败。
- 解决：修改数据目录下的 kingbase.conf 配置文件

 ```bash
 max_connect = 10
 superuser_reserved_connections = 5 #小于max_connect
 super_manager_reserved_connections = 3  #小于superuser_reserved_connections
 ```
### FATAL: lock file kingbase.pid already exists
- 提示：FATAL: lock file kingbase.pid already exists。是因为 docker 容器被关闭了数据库还没来得及停机，我们去数据目录下把 kingbase.pid 文件删除掉即可，数据目录就是上面映射本机目录的，我的教程里是在 /opt/kingbase/data/。


## 参考链接

- [安装文件](https://www.kingbase.com.cn/download.html#database)
- [授权文件](https://www.kingbase.com.cn/download.html#authorization?authorcurrV=V9R1C10)
