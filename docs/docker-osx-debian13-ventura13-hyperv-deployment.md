# Debian 13 Hyper-V 部署 Docker-OSX Ventura 13

本记录用于在 Windows Server 2022 的 Debian 13 Hyper-V VM 中运行 macOS Ventura 13。当前 VM IP 为 \`192.168.31.67\`，已确认嵌套虚拟化生效：CPU 暴露 \`vmx\`，且存在 \`/dev/kvm\`。

## 资源与持久化

- Docker 容器限制为 7 vCPU、7 GiB RAM，swap 上限 8 GiB。
- macOS 磁盘为仓库内 \`data/mac_hdd_ng.img\`，qcow2 格式，虚拟容量 78 GiB。
- \`data/\` 已加入 \`.gitignore\`，不会进入 Git；请单独备份该目录。
- Compose 配置为 \`docker-compose.ventura13.yml\`，只提交配置和文档，不提交磁盘。

## Docker 代理

代理配置在 \`/etc/docker/daemon.json\`，用于 Docker daemon 拉取镜像：

\`\`\`json
{
  "proxies": {
    "http-proxy": "http://192.168.31.233:7890",
    "https-proxy": "http://192.168.31.233:7890",
    "no-proxy": "localhost,127.0.0.1,::1,192.168.31.0/24,172.17.0.0/16"
  }
}
\`\`\`

容器环境也传入相同的 \`HTTP_PROXY\`/\`HTTPS_PROXY\`，供 Ventura 安装介质和启动脚本下载文件使用。验证：

\`\`\`bash
systemctl restart docker
docker info | grep -i proxy
docker run --rm hello-world
\`\`\`

## 启动 macOS 容器

首次启动会下载 Ventura 安装介质，耗时取决于代理速度。不要删除容器；同一个容器和 \`data/mac_hdd_ng.img\` 一起保留即可继续安装进度：

\`\`\`bash
cd /root/docker-osx
docker compose -f docker-compose.ventura13.yml up -d
docker compose -f docker-compose.ventura13.yml logs -f macos
\`\`\`

检查状态：

\`\`\`bash
docker ps --filter name=docker-osx-ventura13
docker inspect docker-osx-ventura13
ss -ltnp | grep -E ':5900|:50922|:8006'
\`\`\`

## noVNC WebUI

容器的原始 VNC 端口仅绑定到 Debian 本机 \`127.0.0.1:5900\`。Debian 主机上的 \`websockify\` 将它转换为浏览器 WebSocket，并提供 noVNC 静态页面：

\`\`\`bash
websockify --web=/usr/share/novnc 0.0.0.0:8006 127.0.0.1:5900
\`\`\`

局域网浏览器访问：

\`\`\`text
http://192.168.31.67:8006/vnc.html?host=192.168.31.67&port=8006
\`\`\`

50922 是 macOS 来宾 SSH 转发端口，不是 WebUI 端口。不要把 5900 原始 VNC 端口暴露到局域网。

## 停止与恢复

\`\`\`bash
cd /root/docker-osx
docker compose -f docker-compose.ventura13.yml stop
docker compose -f docker-compose.ventura13.yml start
\`\`\`

只有在确认不再需要该 macOS 实例时才执行 \`down\`；不要删除 \`data/mac_hdd_ng.img\`。

macOS 虚拟化请遵守 Apple 软件许可和所在环境的合规要求。

