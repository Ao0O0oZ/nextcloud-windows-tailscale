# Windows 上的私有 Nextcloud 云盘

这套方案把一台 Windows 11 电脑变成私有云盘。授权设备可跨网络访问网页并双向同步文件，路由器无需端口转发。服务器本机也可以通过独立的同步目录编辑文件。

## 工作原理

- **Nextcloud 管理文件和账号。** Nextcloud AIO 作为 Linux 容器在 Docker Desktop 的 WSL 2 后端运行。Nextcloud 管理用户、文件权限、版本和回收站。Nextcloud Desktop 把每台电脑的同步目录与服务器双向同步。
- **Tailscale 提供私有网络入口。** 客户端接入同一个 Tailscale 网络（tailnet），访问策略决定谁可连到服务器。Serve 在服务器的 `*.ts.net` 域名上提供 HTTPS，把请求转发给本机的 Nextcloud。访问文件仍需通过 Nextcloud 登录和权限校验。 [Tailscale Serve](https://tailscale.com/docs/reference/tailscale-cli/serve)

```text
已授权的设备 → Tailscale Serve :443 → 127.0.0.1:11000 → Nextcloud AIO
服务器本机   → https://127.0.0.1:8080 → AIO 管理页
```

如果希望由 Codex 协助实际部署，在 Codex 中打开本仓库并调用 `$deploy-nextcloud-windows`，再告诉它要在哪台 Windows 电脑上部署。 [部署技能](.agents/skills/deploy-nextcloud-windows/SKILL.md)

## 依赖与安装顺序

准备一台符合 Docker Desktop 系统要求的 Windows 11 电脑，启用硬件虚拟化，至少有 8 GB 系统内存。用有管理员权限的 Windows 账户完成安装。为 Windows 上的云盘数据、Docker 镜像与数据库，以及每台电脑上的同步副本分别留出空间。 [Docker Desktop 系统要求](https://docs.docker.com/desktop/setup/install/windows-install/#system-requirements)

| 软件 | 安装位置 | 用途与安装顺序 |
| --- | --- | --- |
| [WSL 2](https://learn.microsoft.com/windows/wsl/install) | Windows 服务器 | 先安装或更新 WSL，为 Docker Desktop 提供 Linux 内核。Docker 当前要求 WSL 2.1.5 或更新版本。 |
| [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/) | Windows 服务器 | WSL 就绪后安装。选择 WSL 2 引擎和 Linux 容器模式，由 Docker 下载和运行 Nextcloud AIO。 |
| [Tailscale](https://tailscale.com/download) | 服务器和每台远程设备 | 安装后登录同一个 tailnet。服务器通过内置的 Serve 功能提供私有 HTTPS。需要有权限管理该 tailnet 的 DNS 与访问策略。 |
| [Nextcloud Desktop](https://docs.nextcloud.com/server/stable/user_manual/en/desktop/installation.html) | 每台需要文件同步的电脑 | 在服务端可访问后安装并连接。如果服务器也需要一份日常编辑用的同步副本，也在服务器上安装。 |

Windows 自带的 PowerShell 可以执行本指南的命令。Docker Desktop 自带运行 Linux 容器所需的环境，额外安装 Ubuntu 等发行版是可选的。 [Docker Desktop 的 WSL 2 后端](https://docs.docker.com/desktop/features/wsl/)

## 1. 准备 Windows、WSL 2 和目录

如果这台电脑还没有安装 WSL，在管理员 PowerShell 中运行以下命令，并按提示重启。

```powershell
wsl --install --no-distribution
```

重启后打开管理员 PowerShell，更新 WSL 并检查版本。已有 WSL 时可以从这里开始。

```powershell
wsl --update
wsl --version
wsl --status
```

安装并启动 Docker Desktop，在 **Settings → General** 中确认使用 WSL 2 引擎，并开启登录 Windows 时启动。等 Docker Desktop 启动后检查引擎。

```powershell
docker --context desktop-linux info --format '{{.OSType}}'
wsl --list --verbose
```

预期第一条命令输出 `linux`，第二条命令显示 `docker-desktop` 使用版本 `2`。如果 `docker` 无法识别，重新打开 PowerShell。如果找不到 `desktop-linux` 或引擎未启动，回到 Docker Desktop 检查启动状态和 Linux 容器模式。 [Docker Desktop 设置](https://docs.docker.com/desktop/settings-and-maintenance/settings/)

选择空间充足的内部 NTFS 磁盘保存服务端文件，并为每台桌面客户端选择独立的本地同步目录。下文用 `D:\CloudServer\Data` 和 `C:\CloudSync` 举例。如果只有 C 盘，可用 `C:\CloudServer\Data` 和 `C:\CloudSync`。已有合适的卷可以直接用。

```powershell
Get-Disk | Select-Object Number, PartitionStyle, Size
Get-Volume | Select-Object DriveLetter, FileSystem, Size, SizeRemaining
```

## 2. 创建私有 HTTPS 入口

在服务器和需要同步的设备上安装 Tailscale，并登录同一个 tailnet。在管理后台启用 MagicDNS 和 HTTPS Certificates。证书签发后，完整设备域名会进入公开证书透明度日志，建议先把设备名改为通用名称。检查访问策略，只允许预期的用户或设备访问服务器的 TCP 443。Tailscale 的允许规则会叠加，已有的全量放行规则也需要一起检查。 [Tailscale HTTPS 证书说明](https://tailscale.com/docs/how-to/set-up-https-certificates)

在服务器的管理员 PowerShell 中运行以下命令。记录 `serve status` 显示的主机名，形如 `<设备名>.<tailnet 名>.ts.net`。该地址要等 Nextcloud AIO 启动后才能返回应用页面。路由器无需端口转发，保持 Serve 输出显示 `tailnet only`。

```powershell
tailscale status
tailscale serve status
tailscale serve --bg --https=443 http://127.0.0.1:11000
tailscale serve status
```

如果 TCP 443 上已经有配置，先确认该端口用于本项目，再运行上面的 Serve 命令。 [Tailscale Serve](https://tailscale.com/docs/reference/tailscale-cli/serve) · [MagicDNS](https://tailscale.com/docs/features/magicdns) · [Tailscale 访问规则](https://tailscale.com/docs/features/access-control/grants)

## 3. 启动 Nextcloud AIO

以下命令适用于全新部署。在管理员 PowerShell 中运行，并把第一行替换成你选定的路径。先确定服务端路径，AIO 首次初始化后应按官方迁移流程移动数据。命令检查 Docker 上下文、同名 AIO 容器或卷，以及服务端目录是否为空。遇到旧实例时保留其容器、卷和文件，并通过 AIO 的恢复流程处理。

```powershell
$dataDir = 'D:\CloudServer\Data'
$context = 'desktop-linux'
$engine = docker --context $context info --format '{{.OSType}}'
if ($LASTEXITCODE -ne 0 -or $engine -ne 'linux') {
    throw 'Start Docker Desktop with the WSL 2 Linux engine first.'
}
$containers = @(docker --context $context ps -a --format '{{.Names}}')
if ($LASTEXITCODE -ne 0) { throw 'Cannot inspect Docker containers.' }
$volumes = @(docker --context $context volume ls --format '{{.Name}}')
if ($LASTEXITCODE -ne 0) { throw 'Cannot inspect Docker volumes.' }
if (($containers -like 'nextcloud-aio-*') -or ($volumes -like 'nextcloud_aio_*')) {
    throw 'Existing Nextcloud AIO state found. Review it before proceeding.'
}
if ($dataDir -notmatch '^[A-Za-z]:\\[^\\]') {
    throw 'Choose an absolute data directory on an internal Windows drive.'
}
$volume = Get-Volume -DriveLetter $dataDir.Substring(0, 1) -ErrorAction Stop
if ($volume.FileSystem -ne 'NTFS' -or $volume.DriveType -ne 'Fixed') {
    throw 'Choose a directory on an internal NTFS drive.'
}
if (Get-NetTCPConnection -State Listen -LocalPort 8080,11000 -ErrorAction SilentlyContinue) {
    throw 'Ports 8080 and 11000 must be available before installing AIO.'
}
if (Test-Path -LiteralPath $dataDir) {
    $item = Get-Item -LiteralPath $dataDir -Force -ErrorAction Stop
    if (-not $item.PSIsContainer -or ($item.Attributes -band [IO.FileAttributes]::ReparsePoint) -or
        (Get-ChildItem -LiteralPath $dataDir -Force -ErrorAction Stop | Select-Object -First 1)) {
        throw 'The server data directory must be an empty local directory.'
    }
} else {
    New-Item -ItemType Directory -Path $dataDir -ErrorAction Stop | Out-Null
}
$dataDir = (Get-Item -LiteralPath $dataDir -ErrorAction Stop).FullName
$drive = $dataDir.Substring(0, 1).ToLowerInvariant()
$relativePath = $dataDir.Substring(3).Replace('\', '/')
$dockerDataDir = "/run/desktop/mnt/host/$drive/$relativePath"

$aioArgs = @(
    'run', '--detach', '--init', '--sig-proxy=false',
    '--name', 'nextcloud-aio-mastercontainer',
    '--restart', 'always',
    '--publish', '127.0.0.1:8080:8080',
    '--env', 'APACHE_PORT=11000',
    '--env', 'APACHE_IP_BINDING=127.0.0.1',
    '--env', 'SKIP_DOMAIN_VALIDATION=true',
    '--env', "NEXTCLOUD_DATADIR=$dockerDataDir",
    '--volume', 'nextcloud_aio_mastercontainer:/mnt/docker-aio-config',
    '--volume', '//var/run/docker.sock:/var/run/docker.sock:ro',
    'ghcr.io/nextcloud-releases/all-in-one:latest'
)
docker --context $context @aioArgs
if ($LASTEXITCODE -ne 0) { throw 'Nextcloud AIO mastercontainer did not start.' }
```

`SKIP_DOMAIN_VALIDATION=true` 跳过 AIO 的域名预检，适用于此处的私有入口。仍需使用准确的 Serve 主机名，并通过正常的 HTTPS 证书校验来验收。上述 `NEXTCLOUD_DATADIR` 使用 Docker Desktop 的 Linux 路径。Nextcloud 数据文件放在选定的 Windows 目录，数据库和 AIO 配置由 Docker 持久卷保存。 [AIO Windows 安装与数据目录](https://github.com/nextcloud/all-in-one#how-to-run-aio-on-windows) · [AIO 反向代理配置](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)

在服务器本机打开 `https://127.0.0.1:8080`。这是使用自签名证书的 AIO 管理地址。把管理口令保存在自己的密码管理器中。在初始化界面输入 Serve 显示的完整主机名，不带 `https://`，然后启动 AIO 组件。等待健康检查通过，使用安装界面提供的 Nextcloud 管理员凭据登录，再创建日常同步使用的普通账户。 [Nextcloud AIO 管理界面](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md#4-open-the-aio-interface)

启动完成后，在已连接 Tailscale 的电脑上访问 `https://<设备名>.<tailnet 名>.ts.net/status.php`，将占位符换成自己的主机名。浏览器应接受 HTTPS 证书，页面应返回 `"installed":true` 和 `"maintenance":false`。随后用同一主机名打开网页登录。

## 4. 配置双向同步和 Windows 回收站

1. 在客户端登录 Tailscale，先用浏览器打开私有 HTTPS 地址。安装 Nextcloud Desktop，添加此地址，通过浏览器为客户端授权。
2. 选择一个独立的空同步目录，例如 `C:\CloudSync`。服务端目录由 Nextcloud 管理，日常编辑应通过客户端目录、网页或 WebDAV 完成。开启 Nextcloud Desktop 登录时启动。
3. 如果需要 Windows 回收站保护，在客户端同步目录行的 **⋯** 菜单里启用虚拟文件支持，然后将整个目录设为 **始终在本地可用**。等文件完整下载。
4. 从托盘退出 Nextcloud Desktop。在 `%APPDATA%\Nextcloud\nextcloud.cfg` 现有的 `[General]` 段加入或更新 `moveToTrash=true`，保存后重新启动客户端。保留文件里的其他设置。

Windows 客户端对本机回收站的处理取决于版本、虚拟文件模式和文件状态。以实际的删除与还原测试为准。Nextcloud 网页端的「已删除文件」可单独用于恢复。 [桌面同步菜单](https://docs.nextcloud.com/server/stable/user_manual/en/desktop/usage.html) · [客户端配置文件](https://docs.nextcloud.com/server/stable/user_manual/en/desktop/configfile.html) · [Nextcloud 已删除文件](https://docs.nextcloud.com/server/stable/user_manual/en/files/deleted_file_management.html)

- **仅同步需要的内容。** 在桌面客户端选择要同步的目录，并按需配置忽略规则。把 Git 工作区和自动化生成的临时目录放在同步范围外。需要限制某台设备对目录的访问时，使用未获该目录授权的普通账号。 [Nextcloud Desktop 使用说明](https://docs.nextcloud.com/server/stable/user_manual/en/desktop/usage.html)
- **按需上传 Android 照片。** 手机连接 Tailscale 后，在 Nextcloud 的「自动上传」中添加自定义本地目录，指定云端目的地，再放入几张照片验证。这个流程用于手机向云端上传。 [Nextcloud Android 自动上传功能](https://github.com/nextcloud/android/blob/master/CHANGELOG.md)

## 5. 验收与恢复

- **服务健康。** 在服务器运行 `docker --context desktop-linux ps --filter name=nextcloud-aio`，确认运行中的组件健康。用浏览器打开私有 HTTPS 地址，确认登录页可用。服务器需要保持开机并避免进入睡眠。重启 Windows 并登录后再检查一次。
- **跨网双向同步。** 从另一个网络上的已授权设备上传一个小文本文件，在 Windows 同步目录打开并修改它，确认网页和另一台设备都收到更新。断网期间修改测试文件，恢复网络后确认同步继续。
- **删除恢复。** 在确认测试文件完整保存在 Windows 后，从网页端删除。等待同步，检查 Windows 回收站，再还原文件并确认重新上传。另用测试文件验证 Nextcloud 网页回收站。
- **实例备份。** 需要重装、迁移或保留原账号与版本历史时，通过 AIO 管理页创建 Borg 备份。备份仓库放在独立存储，加密密钥另存，并在维护计划中安排恢复演练。 [AIO 备份和恢复](https://github.com/nextcloud/all-in-one#backup)
- **常见连接问题。** 网页连不上时先核对两端 Tailscale 是否上线、域名是否正确、访问规则是否允许 TCP 443，然后检查 `tailscale serve status` 和 Docker 组件。若设备启用了系统代理，可让私有域名直连后再测试。 [Tailscale Serve 排障](https://tailscale.com/docs/features/tailscale-serve)

## 6. 多账号隔离与迁移

- **个人与共享目录。** 为每个人创建普通账号，管理账号单独使用。个人目录放在各自账户下。在「账户」中创建用户组并添加成员，再在「管理设置 → 团队文件夹」给该组授权。授权后的目录出现在成员的「文件」页面。Nextcloud 的「Teams（团队）」页面列出另行创建并加入的团队。给成员授予删除权限后，他们可以删除共同目录里的文件。 [用户组管理](https://docs.nextcloud.com/server/stable/admin_manual/configuration_user/user_configuration.html#managing-groups) · [团队文件夹](https://github.com/nextcloud/groupfolders#configuring-group-folders)
- **迁移旧账号的数据。** 先暂停相关客户端同步，并通过 AIO 备份。通过 Nextcloud 的文件所有权转移功能迁移指定目录，再用两个新账号分别验证个人文件和共享文件的访问范围。迁移出错时，先对照源与目标的文件路径、大小、索引和共享记录，再决定如何恢复。若确认文件完整落在目标且只是未登记，可在保存旧版本后用 `occ files:scan --path=...` 扫描该目录。文件扫描只更新索引，旧版本和共享需要另行核对。 [所有权转移和文件扫描](https://docs.nextcloud.com/server/stable/admin_manual/occ_files.html)
- **停用与清理。** 确认仍有可用的管理员账号，将各设备上的旧同步连接解除并保留本地文件。核对旧账号的回收站、历史版本及共享，先取回仍需保留的数据。可先停用旧账号，确认迁移验收后再删除。迁移时临时归档的版本若是最后一份，确认可以舍弃后再清理。 [停用与删除账号](https://docs.nextcloud.com/server/stable/admin_manual/configuration_user/user_configuration.html#disable-and-enable-users)
