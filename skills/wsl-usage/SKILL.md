---
name: wsl-usage
description: >
  掌握并指导使用 WSL（Windows Subsystem for Linux）。涵盖 WSL2 安装/升级、发行版管理、与 Windows 文件互访（/mnt/c）、在 WSL 里跑 Linux 工具链、从 Windows 调用 WSL（wsl 命令）以及反向从 WSL 启动 Windows exe、端口/网络、systemd、默认发行版与配置等。
  当用户在使用 Windows + WSL、说"WSL 里装什么/跑什么""用 Linux 命令但是 Windows 机器""怎么访问 Windows 文件""wsl --install" 时触发。
license: MIT
compatibility: WSL2 需 Windows 10 2004+ / Win11；运行是 Windows 上的 Linux 子系统，本 skill 给操作与配置，不需本地 Linux
---

# WSL (Windows Subsystem for Linux) 使用

指导用户在 **Windows 里高效使用 WSL2** 跑 Linux。核心：装好 → 管理发行版 → 打通 Windows/Linux 文件与命令 → 按需装工具。

## When to use

- 用户在装有 WSL 的 Windows 上，要用 Linux 命令/工具/环境
- 装或升级 WSL、新增发行版（Ubuntu 等）、改默认
- 需要 Linux 与 Windows 互相访问文件、互相调起程序、共享端口/网络
- WSL 里跑 agent、脚本、Python/Node/容器等开发工具

## 安装 / 升级（管理员 PowerShell）

```powershell
# 全新安装（重启后按提示设 Linux 用户名密码）
wsl --install

# 装特定发行版 / 列可用发行版
wsl --install -d Ubuntu-24.04
wsl --list --online        # 简写 wsl -l -o

# 升级内核到 WSL2 并设默认（Win10 老版需要手动开关功能）
wsl --set-default-version 2
wsl --version              # 查看 WSL 自身版本
```

版本务必 **WSL2**（虚拟化、性能好、可跑 Docker）：用 `wsl -l -v` 看，1 版则 `wsl --set-version Ubuntu 2`（需开启 BIOS 虚拟化）。

## 发行版管理

```powershell
wsl -l -v                  # 列出发行版 + 版本(1/2) + 状态
wsl -s Ubuntu-24.04        # 设为默认发行版
wsl -d Ubuntu-24.04        # 指定发行版进入
wsl --terminate Ubuntu     # 停该发行版（释放内存）
wsl --unregister Ubuntu    # ⚠️ 删除该发行版及其全部数据！
wsl --update               # 更新 WSL 内核
```

进入：
```powershell
wsl                       # 进默认发行版
wsl -d Ubuntu-24.04       # 指定发行版
wsl -u root               # 以 root 进
```

## ⚠️ 文件放哪（性能关键）

- **Linux 文件** 放在发行版自己系统里（`~/`）→ 原生 ext4 快，跑开发/数据库用它
- 跨系统 `\\wsl$\...` 或 `/mnt/c/...` 是跨文件系统，**I/O 慢**（尤其 node_modules/数据库/大量小文件）
- **路径建议**：源码如果主要给 WSL 用就放 WSL 内；如果要在 Windows 编辑器(Linux 导出的文件)两侧用，才放 /mnt/c 并接受/避免大文件目录

```bash
# 从 WSL 看 Windows 盘（装在常规盘时）
ls /mnt/c/Users/你的名字/Desktop

# Windows 里访问 WSL 文件
# 资源管理器地址栏输入:  \\wsl$\Ubuntu-24.04\home\xxx
# 或命令:  explorer.exe \.
# 从 WSL 临时取桌面文件常用 /mnt/c/...
```

## Linux ↔ Windows 命令互调

```bash
# 在 WSL 里启动 Windows 程序（用 .exe）
notepad.exe
explorer.exe .                 # 打开当前目录资源管理器
code .                         # Windows VS Code 打开当前 WSL 目录（最好装 WSL 扩展）
powershell.exe -c "Get-Date"
```

```powershell
# 从 Windows 调 WSL 里跑一条命令（取结果）
wsl ls -l /etc/os-release
wsl python3 /home/me/script.py
# 交互式跑 WSL 脚本并回传：
wsl -e bash -lc "echo from-linux"
```

- `cmd.exe /c dir` 从 WSL 取 Windows 输出也行。注意**路径互换算**：`C:\x` → WSL 里 `/mnt/c/x`。
- 中文路径在 WSL 里出现乱码，改用 `\\wsl$\`Explorer 或给路径加引号。

## 网络 / 端口

WSL2 用 NAT，Windows 访问 WSL 服务大多 `localhost` 直达（自动转发）；外部设备访问要规则：

```powershell
# 查 WSL 局域网 IP（两个途径）
wsl hostname -I
# WSL2 到 Windows host: 常见用 $(ip route | awk '/default/{print $3}') 或 localhost(.exe)
```

- WSL2 里跑 `python3 -m http.server 8000`，Windows 浏览器直接 `http://localhost:8000`
- 需局域网/公网访问，用端口转发 `netsh interface portproxy` 或把 WSL 设镜像网络（`.wslconfig` 开 `networkingMode=mirrored`，Win11 走巧）

## systemd + .wslconfig 配置

Win11 / 老版本 `.wslconfig`（在 Windows 用户主目录，控制内存 CPU 网络 systemd）：

```
# C:\Users\你\.wslconfig
[wsl2]
memory=6GB
processors=4
swap=2GB
# networkingMode=mirrored      # Win11 镜像网络，端口直通省 portproxy
```

在发行版内开 systemd（服务管理）：
```bash
# 编辑 /etc/wsl.conf
[boot]
systemd=true
# 然后 > wsl --terminate Ubuntu 后重进，systemctl 就能用了
```

## 在 WSL 装常用开发工具

WSL 就是个完整的 Ubuntu → `apt` 装一切：
```bash
sudo apt update && sudo apt install -y python3 python3-pip git curl unzip
# Docker（WSL2 集成最佳，装 Docker Desktop 用 WSL2 backend 或 apt 装 docker.io）
# VS Code 集成：安装 "WSL" 扩展后在 WSL 内 `code .`
```

## 常见坑 / 排障

- **缺 WSL 更新**：老 Windows 装不上 docker / 报 WSL2 内核错 → 管理员 `wsl --update`
- **wsl --install 只装默认**：要更多发行版 `wsl --install -d <name>`；没有列出的用 tar 导入
- **磁盘占满**：wsl --unregister 会清光（先备份）；调小 .wslconfig swap；虚拟磁盘文件不要塞到 OneDrive 目录
- **DNS 难连**：`/mnt/c` 文件跑 node 卡 → 移进 `~/`；`sudo nano /etc/resolv.conf` 改成中国 DNS 8.8.8.8 + 设 `nameserver=8.8.8.8`（有时需在 .wslconfig generation 里关自动覆盖）
- **端口占用/连不上外部**：WSL2 默认不对外开，外部服务要镜像网络或 portproxy；确认 Windows 防火墙放行
- **GUI 应用**：用 `wsl --install` + WSLg（Win11/H2 自动）直接跑 Linux GUI app
- **别乱敲 unregister**：等价删库+数据归零，给这个命令前确认或先提示备份

## 原则

**先问清"跑 Linux 工具在哪端需要"——纯 Linux 工作放发行版内、与 Windows 互访走 /mnt + \\wsl$、进程互调 .exe & wsl 命令。** 文件路径归属和 Networking 是两大致败点，永远把 .wslconfig 与 WSL2(不是1)一起想。
