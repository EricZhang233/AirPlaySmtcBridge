# AirPlaySmtcBridge

AirPlaySmtcBridge 是一个 Windows 后台桥接程序：它被动读取本机网络适配器上的 AirPlay/RAOP 流量，并把歌曲信息、封面、播放状态和播放进度发布到 Windows 的系统媒体传输控件（SMTC）。程序本身不接收、播放或转发音频。

> **运行前提：** 需另行运行 AirPlay 接收软件；本程序只读取流量，不接收或播放音频。

## 功能

- 从 AirPlay RTSP 流中读取歌曲信息、封面、播放进度和会话状态。
- 在 Windows 媒体控件中显示歌曲名、艺人、专辑、封面和进度。
- 将媒体控件中的播放、暂停、上一首、下一首、快退和快进操作通过 AirPlay DACP 发回发送端。

## 运行要求

- 必须在运行桥接程序的这台电脑上另行安装并运行 AirPlay 接收客户端（如[AirServer](https://www.airserver.com/download "建议下载64-Bit版本而非MSIX版本，64位版本支持无界面运行及开机自启，这对音频播放和即用性很友好。")和[爱思投屏](https://www.i4.cn/pro_screen.html)等），由它接收发送设备推送的 AirPlay 流。本程序不提供接收服务；没有活动会话时无法获取 AirPlay 数据。
- Windows 10 或 Windows 11（x64）。
- 本机网络适配器和 Npcap 抓包驱动。程序通过 Npcap 的 `Packet.dll` 被动读取所有可打开的网络适配器上的网络帧。
- 首次运行如未检测到 Npcap，程序会提示并尝试从 Npcap 官网下载官方安装程序；安装需要在 Windows UAC 提示中由用户确认。程序自身不提权。
- AirPlay 发送设备必须能通过网络发现并连接这台电脑上的本桥接程序和接收客户端；通常需要处于同一局域网，并允许 mDNS 和 AirPlay 流量通过。

## 构建

本项目不提供预编译的 Release 包。以下两种构建方式分别说明各自需要的源码获取步骤；下载命令适用于公开仓库。

### 普通 Windows 环境（推荐，无需 Git 和其他额外安装，只需要完整的 Windows ）

先下载源码 ZIP 并解压：

在 PowerShell 中下载 `main` 分支并解压到当前目录：

```powershell
Invoke-WebRequest -Uri "https://github.com/EricZhang233/AirPlaySmtcBridge/archive/refs/heads/main.zip" -OutFile AirPlaySmtcBridge-main.zip
Expand-Archive .\AirPlaySmtcBridge-main.zip -DestinationPath .
Set-Location .\AirPlaySmtcBridge-main
```

依赖系统自带的 .NET Framework C# 编译器和 `%WINDIR%\System32\WinMetadata` 元数据，无需安装 .NET SDK：

```powershell
.\build-dotnet-framework-csharp.ps1
```

生成文件：`AirPlaySmtcBridge.exe`（项目目录）。

### Visual Studio 或 .NET CLI

先用 Git 克隆仓库：

```powershell
git clone --branch main https://github.com/EricZhang233/AirPlaySmtcBridge.git
```

在 Visual Studio 中打开 `AirPlaySmtcBridge.csproj`，选择 `Release` 和 `x64` 后生成；也可以运行：

```powershell
dotnet build .\AirPlaySmtcBridge.csproj -c Release -p:Platform=x64
```

生成文件：`bin\Release\AirPlaySmtcBridge.exe`。

此方式需要 .NET SDK、MSBuild 和 .NET Framework 4.8 targeting pack。

## 启动与参数

双击 `AirPlaySmtcBridge.exe` 即可启动。程序没有主窗口；运行日志写入：

```text
%TEMP%\eric\AirPlayBridge\AirPlayBridge.log
```

支持以下命令行参数：

| 参数 | 作用 |
| --- | --- |
| `--no-control` | 不向发送端发送播放控制命令，只镜像媒体信息。 |
| `--no-timeline` | 不发布播放进度时间轴。 |
| `--disconnect-sec=N` | 设置判断 AirPlay 会话结束的等待秒数，默认 5 秒。 |
| `--verbose` | 在日志文件之外向标准错误输出额外诊断信息。 |
| `--no-dialogs` | 禁用提示对话框，诊断和部署流程改写入日志。 |
| `--stop` | 通知正在运行的实例正常退出并清理 SMTC 媒体会话。 |
| `--fetch-npcap` | 仅下载并验证 Npcap 安装程序，不安装。 |
| `--test-dialog` | 显示 Npcap 安装提示对话框，用于检查提示内容。 |

例如，关闭回传控制并开启详细日志：

```powershell
.\AirPlaySmtcBridge.exe --no-control --verbose
```

## 开机自启设置

将程序快捷方式放入当前 Windows 用户的“启动”文件夹即可：

1. 按 `Win + R`，输入 `shell:startup` 并按回车，打开当前用户的启动文件夹。
2. 将 `AirPlaySmtcBridge.exe` 的快捷方式（`.lnk`）复制到该文件夹。可以复制开始菜单中的 `AirPlaySmtcBridge` 快捷方式；如果没有该快捷方式，请右键程序文件，选择“创建快捷方式”，再将创建的快捷方式复制进去。
3. 下次登录 Windows 时，程序会自动启动。此设置只对当前用户生效；请勿把程序本体移动到启动文件夹。

## 工作方式

- **发现与抓包：** 通过 mDNS 查询 `_airplay._tcp` 和 `_raop._tcp` 服务，动态发现接收端公布的 RTSP 端口；也监听本机 mDNS 广播。未发现服务时回退到 TCP 7000。程序会抓取所有可打开的 Npcap 网络适配器，并从每个有 IPv4 地址的接口发出查询。
- **媒体信息与控制：** 从接收端发送的 AirPlay RTSP 流中读取媒体信息，并通过 DACP 将媒体控件操作发回发送设备。
- **播放状态：** 依据发往本机的大包 UDP 音频流量推断；检测到音频包时显示“正在播放”，采样期间没有音频包时显示“已暂停”。
- **播放进度：** 使用 AirPlay 上报的时间轴；两次上报之间由 Windows 继续推进显示。时间轴跳转范围设为零宽，以保留进度显示并关闭可拖动的跳转范围。
- **会话与文件：** 使用 `AirPlaySmtcBridge` 作为进程互斥标识，避免多个实例同时运行。运行日志和临时封面保存在 `%TEMP%\eric\AirPlayBridge`。

## 限制

- 媒体元数据依赖接收软件发送本程序支持的明文 RTSP/DMAP 格式。
- 暂停状态由 UDP 流量推断；网络丢包、Npcap 状态或抓包情况可能导致判断不准。
- 播放进度取决于设备上报。上报间隔较长或时间轴与实际声音不同步时，显示可能滞后或偏离听感；程序没有音频输出时间戳，无法保证消除偏差。
- 不同 Windows 组件对零宽跳转范围的进度条交互表现可能不同。快退和快进按钮保持启用，但效果取决于发送设备及 AirPlay/DACP 会话是否支持相应命令。

## 项目文件

- `AirPlayBridge.cs`：程序实现。
- `build-dotnet-framework-csharp.ps1`：使用 .NET Framework C# 编译器的简易构建脚本。
- `AirPlaySmtcBridge.csproj`：Visual Studio 和 .NET CLI 共用的 MSBuild 项目。
- `app.manifest`：Windows 应用清单。
- `airplay.ico`、`airplay.png`：应用图标资源。
