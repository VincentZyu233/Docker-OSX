# Debian 13 Hyper-V 部署 Docker-OSX Ventura 13

本记录用于在 Windows Server 2022 的 Debian 13 Hyper-V VM 中运行 macOS Ventura 13。当前 VM IP 为 `192.168.31.67`，已确认嵌套虚拟化生效：CPU 暴露 `vmx`，且存在 `/dev/kvm`。

## 资源与持久化

- Docker 容器限制为 7 vCPU、7 GiB RAM，swap 上限 8 GiB。
- macOS 磁盘为仓库内 `data/mac_hdd_ng.img`，qcow2 格式，虚拟容量 78 GiB。
- `data/` 已加入 `.gitignore`，不会进入 Git；请单独备份该目录。
- Compose 配置为 `docker-compose.ventura13.yml`，只提交配置和文档，不提交磁盘。

## Docker 代理

代理配置在 `/etc/docker/daemon.json`，用于 Docker daemon 拉取镜像：

```json
{
  "proxies": {
    "http-proxy": "http://192.168.31.233:7890",
    "https-proxy": "http://192.168.31.233:7890",
    "no-proxy": "localhost,127.0.0.1,::1,192.168.31.0/24,172.17.0.0/16"
  }
}
```

容器环境也传入相同的 `HTTP_PROXY`/`HTTPS_PROXY`，供 Ventura 安装介质和启动脚本下载文件使用。验证：

```bash
systemctl restart docker
docker info | grep -i proxy
docker run --rm hello-world
```

## 启动 macOS 容器

首次启动会下载 Ventura 安装介质，耗时取决于代理速度。不要删除容器；同一个容器和 `data/mac_hdd_ng.img` 一起保留即可继续安装进度：

```bash
cd /root/docker-osx
docker compose -f docker-compose.ventura13.yml up -d
docker compose -f docker-compose.ventura13.yml logs -f macos
```

检查状态：

```bash
docker ps --filter name=docker-osx-ventura13
docker inspect docker-osx-ventura13
ss -ltnp | grep -E ':5900|:50922|:8006'
```

## noVNC WebUI

容器的原始 VNC 端口仅绑定到 Debian 本机 `127.0.0.1:5900`。Debian 主机上的 `websockify` 将它转换为浏览器 WebSocket，并提供 noVNC 静态页面：

```bash
websockify --web=/usr/share/novnc 0.0.0.0:8006 127.0.0.1:5900
```

局域网浏览器访问：

```text
http://192.168.31.67:8006/vnc.html?host=192.168.31.67&port=8006
```

本机 noVNC 页面已加入 CSS 覆盖，让 Windows 浏览器光标保持可见；如果浏览器使用旧缓存，请执行强制刷新（`Ctrl+F5`）。

50922 是 macOS 来宾 SSH 转发端口，不是 WebUI 端口。不要把 5900 原始 VNC 端口暴露到局域网。

## Debian SSH 中文显示

如果 SSH 终端把中文文件名显示成 `\\345\\270...` 形式，重新登录或执行下面命令加载 UTF-8 locale：

```bash
export LANG=C.UTF-8
export LANGUAGE=C
locale
```

本机 `/root/.profile` 已设置 `LANG=C.UTF-8`。Windows Terminal、PowerShell 或 PuTTY 也应选择 UTF-8 编码；修改后重新建立 SSH 连接。

## macOS 首次安装

打开 WebUI 后会进入 macOS Ventura Recovery：

1. 选择 `Reinstall macOS Ventura`，点击 `Continue`。
2. 如果没有可用目标磁盘，打开 `Disk Utility`，选择 `View -> Show All Devices`。
3. 选择约 78 GiB 的 QEMU 内部磁盘，不要选择 Recovery 安装介质，点击 `Erase`。
4. `Format` 选择 `APFS`，`Scheme` 选择 `GUID Partition Map`，名称可填 `Macintosh HD`。
5. 退出磁盘工具，重新选择 `Reinstall macOS Ventura`，目标选刚创建的 `Macintosh HD`。

安装期间保持容器运行。安装完成后如果重回 Recovery，重启并从 `Macintosh HD` 启动；安装进度和系统文件都保存在 `data/mac_hdd_ng.img`。

安装器显示的剩余时间是动态估算，在 QEMU 中可能长时间停留在 `1 hour` 以上，并不等于卡死。只要 WebUI 还能刷新画面、容器状态是 `running`，且 Debian 上 `data/mac_hdd_ng.img` 的实际占用或 QEMU 写入量继续增长，就应耐心等待，不要点击 `Cancel`、重启容器或删除磁盘。若连续较长时间完全没有磁盘写入，再提供最新截图和下面的输出：

```bash
docker inspect docker-osx-ventura13 --format 'status={{.State.Status}} restarts={{.RestartCount}}'
ls -lh /root/docker-osx/data/mac_hdd_ng.img
```

## macOS 内安装 Homebrew、Fish 和 Fastfetch

以下命令在 macOS Ventura 桌面内的 Terminal 执行。当前虚拟机是 Intel x86_64，Homebrew 默认路径为 `/usr/local`。首次安装若提示 Command Line Tools，按提示安装并重新运行 Homebrew 安装命令。

```bash
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

安装完成后，为 zsh 配置 Homebrew 环境（Intel）：

```bash
echo 'eval "$(/usr/local/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/usr/local/bin/brew shellenv)"
brew --version
```

### 切换 USTC 镜像

USTC 文档推荐通过环境变量切换 Brew 源、API 和 bottle 下载源。写入 `~/.zprofile` 后新开的终端也会生效：

```bash
cat >> ~/.zprofile <<'EOF'
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.ustc.edu.cn/brew.git"
export HOMEBREW_BOTTLE_DOMAIN="https://mirrors.ustc.edu.cn/homebrew-bottles"
export HOMEBREW_API_DOMAIN="https://mirrors.ustc.edu.cn/homebrew-bottles/api"
EOF
source ~/.zprofile
brew update
```

如果后续要恢复 Homebrew 官方源，先执行 `unset HOMEBREW_BREW_GIT_REMOTE HOMEBREW_BOTTLE_DOMAIN HOMEBREW_API_DOMAIN`，再按 USTC 文档恢复对应 Git remote。

### 安装并设置 Fish

```bash
brew install fish
fish --version
echo "$(brew --prefix)/bin/fish" | sudo tee -a /etc/shells
chsh -s "$(brew --prefix)/bin/fish"
```

重新打开 Terminal 或执行 `exec fish` 后即可进入 Fish。需要返回 zsh 时执行 `chsh -s /bin/zsh`。

### 安装 Fastfetch

```bash
brew install fastfetch
fastfetch
```

验收时应看到 `macOS Ventura`、Intel CPU、内存和磁盘信息。Homebrew、Fish 配置和安装的软件都写入持久化 macOS 磁盘，不会随容器重启丢失。

## 停止与恢复

```bash
cd /root/docker-osx
docker compose -f docker-compose.ventura13.yml stop
docker compose -f docker-compose.ventura13.yml start
```

只有在确认不再需要该 macOS 实例时才执行 `down`；不要删除 `data/mac_hdd_ng.img`。

macOS 虚拟化请遵守 Apple 软件许可和所在环境的合规要求。
