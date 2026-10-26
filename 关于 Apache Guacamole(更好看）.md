下面是基于 abhinavasr/Guacamole 项目的 Windows 一键脚本部署完整步骤。这个方案使用 PostgreSQL 作为后端，并会自动创建一个 VNC 连接（一个内置的 Chrome 浏览器）供你测试。

📋 前置条件

在开始之前，请确保 Windows 上已准备好：

1. Docker Desktop：已安装并处于运行状态。如果没有安装，Windows 11 通常需要先启用 WSL 2。
2. Git：用于克隆仓库。

🚀 第一步：克隆仓库

打开 PowerShell（或 CMD），在你希望存放项目的目录下执行：

```powershell
git clone https://github.com/abhinavasr/Guacamole.git
cd Guacamole
```

⚡ 第二步：运行一键脚本

在 Guacamole 目录下，直接运行：

```powershell
setup.bat
```

这个脚本会自动完成镜像拉取、容器启动和数据库初始化等一系列操作。

🌐 第三步：访问并登录

脚本执行完成后，打开浏览器访问：
http://localhost:5000/
使用默认凭据登录：

· 用户名：guacadmin
· 密码：guacadmin

✅ 第四步：验证安装

登录后，你应该能在连接列表中看到一个自动创建的 Chrome 浏览器连接。点击它，如果能看到一个运行着 Chrome 的虚拟桌面，就说明 Guacamole 的核心服务（guacd、数据库、Web 前端）都已正常工作。

---

⚠️ 关键后续操作与避坑提醒

1. 立即修改默认密码
guacadmin/guacadmin 仅用于初始访问，登录后请务必在右上角的用户设置中立即修改密码。

2. 添加 RDP 连接来远控“本机 Windows”
一键脚本创建的 Chrome 连接只是个测试。要远控你的 Windows 主机，需要手动新建连接：

· 在 Guacamole 设置中，新建一个连接。
· 协议选择 RDP。
· Hostname 必须填这台 Windows 电脑的局域网 IP 地址（如 192.168.1.100），绝对不能填 localhost 或 127.0.0.1。因为 guacd 运行在独立的容器网络中，它看到的 localhost 是容器自己，而不是你的宿主机。
· 填写你的 Windows 用户名和密码，保存后即可连接。

3. 关于 Windows 下的脚本权限
如果 setup.bat 执行中因脚本权限问题报错，官方 README 提供了一个 PowerShell 的补救措施，可以移除 init 目录下脚本的只读属性：

```powershell
attrib -R .\init\*.sh
```

4. 结合你的 Cloudflare Tunnel
Guacamole 服务目前在本地运行于 http://localhost:5000。你只需在 Cloudflare Tunnel 的配置中，将公共主机名指向本地的 http://localhost:5000，即可在外网安全访问，无需额外开放路由器端口。

5. 日常管理
如需停止或重启服务，在 Guacamole 目录下可以使用标准的 Docker Compose 命令：

```powershell
docker-compose down   # 停止并移除容器
docker-compose up -d  # 重新在后台启动
```

方案一里的两种后端，切换步骤都不复杂，但它们的关键区别在于系统版本门槛和资源要求。下面分别说明。

🖥️ 情况一：切换到 Hyper-V 后端

前提：Hyper-V 仅支持 Windows 专业版、企业版或教育版，家庭版无法使用。如果你的系统符合条件，按以下步骤操作：

1. 启用 Hyper-V 功能（如果尚未启用）
以管理员身份打开 PowerShell，执行：

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

或者，在“控制面板 → 程序 → 启用或关闭 Windows 功能”中，勾选 Hyper-V 和 Containers。完成后重启电脑。

2. 在 Docker Desktop 中切换后端
重启后，打开 Docker Desktop → Settings → General，在“Use the WSL 2 based engine”或相关后端选项中，取消勾选 WSL 2，Docker 会自动回退到 Hyper-V 后端。点击 Apply & restart 即可。

重要提醒：启用 Hyper-V 后，VirtualBox、VMware Workstation（15以下版本）等第三方虚拟机可能无法正常使用，因为它们与 Hyper-V 的虚拟化层存在冲突。如果你同时需要用这些虚拟机（如安卓模拟器），需要谨慎考虑。

---

🐳 情况二：切换到 Docker VMM（Beta）后端

这是 Docker 自研的容器优化虚拟机管理器，不限制 Windows 版本（家庭版也可用），是 WSL 2 的稳定替代方案。

1. 检查版本与内存

· Docker Desktop 版本：必须为 4.86 或更高版本。低版本设置中不会出现 Docker VMM 选项。
· 内存要求：分配给 Docker Linux VM 的内存最低 4 GB。

2. 调整内存分配（如不足 4 GB）
打开 Docker Desktop → Settings → Resources，将 Memory 滑块调整为 4 GB 以上，点击 Apply & restart。

3. 切换后端
内存确认充足后，进入 Settings → General → Virtual Machine Manager，选择 Docker VMM，然后点击 Apply & restart。重启后即生效。

4. 已知小问题
Docker VMM 不支持自动的文件共享（bind mount auto-shares）。如果后续在 Guacamole 部署中遇到“file is not shared from the host”报错，需要手动去 Settings → Resources → File sharing 中，把需要共享的目录添加进去。

---

💎 如何选择

· 系统是 Windows 专业版/企业版：两种都可用。Hyper-V 更成熟，但会挤占其他虚拟化软件；Docker VMM 更轻量、对容器优化更好，但仍是 Beta。
· 系统是 Windows 家庭版：只能选 Docker VMM，Hyper-V 不可用。

切换到任一后端后，就可以回到 Guacamole 的部署流程，继续执行 git clone 和 setup.bat 了。