
# Minecraft 服务器维护指南 (234mc)

## 目录
- [0. 服务器地址](#0-服务器地址)
- [1. 服务器基础环境](#1-服务器基础环境)
    - [1.1 硬件规格](#11-硬件规格)
    - [1.2 网络与 NAT 转发映射表](#12-网络与-nat-转发映射表)
    - [1.3 域名解析](#13-域名解析)
- [2. 进程管理工具：Screen](#2-进程管理工具screen)
    - [2.1 核心会话列表](#21-核心会话列表)
    - [2.2 常用操作命令](#22-常用操作命令)
- [3. 服务器主进程维护 (234mc)](#3-服务器主进程维护-234mc)
    - [3.1 概览](#31-概览)
    - [3.2 运行与更新](#32-运行与更新)
- [4. 基岩互通服务维护 (bedlink)](#4-基岩互通服务维护-bedlink)
    - [4.1 运行机制](#41-运行机制)
    - [4.2 维护操作](#42-维护操作)
- [5. 皮肤系统维护 (C# API + Nginx)](#5-皮肤系统维护-c-api--nginx)
    - [5.1 用户前端](#51-用户前端)
    - [5.2 C# 皮肤接口 (skinapi)](#52-c-皮肤接口-skinapi)
    - [5.3 Nginx 静态服务](#53-nginx-静态服务)
- [6. 备份服务维护 (restic)](#6-备份服务维护-restic)
    - [6.1 定时任务 (Systemd Timer)](#61-定时任务-systemd-timer)
    - [6.2 恢复数据流程](#62-恢复数据流程)
- [7. 简单语音聊天模组 (SVC) 维护](#7-简单语音聊天模组-svc-维护)
    - [7.1 运行机制与网络](#71-运行机制与网络)
    - [7.2 核心配置参数](#72-核心配置参数)
    - [7.3 常见问题排查](#73-常见问题排查)
- [8. 维护注意事项与故障排除](#8-维护注意事项与故障排除)

---

## 0. 服务器地址


| 客户端类型 | 线路类型 | 服务器地址 (IP/域名) | 端口 | 备注 |
| :--- | :--- | :--- | :--- | :--- |
| **[Java 版](#3-服务器主进程维护-234mc)** (>= 1.21.1) | 普通线路 | `classicbyte.asia:10056` | - | |
| **[Java 版](#3-服务器主进程维护-234mc)** (>= 1.21.1) | 🚀 [加速线路](#8-维护注意事项与故障排除) | `ea70d80d.peiqianyun.xyz:54255` | - | |
| **[基岩版](#4-基岩互通服务维护-bedlink)** (最新版) | 普通线路 | `classicbyte.asia` | 10165 | |
| **[基岩版](#4-基岩互通服务维护-bedlink)** (最新版) | 🚀 [加速线路](#8-维护注意事项与故障排除) | `d3a6ab92.peiqianyun.xyz` | 54250 | |
| **[语音聊天](#7-简单语音聊天模组-svc-维护)** | 游戏内通讯 | `classicbyte.asia` | 10222 | 协议为 UDP，进入游戏按 V 键配置 |


## 1. 服务器基础环境
本服务器运行在 **Debian 12 (bookworm) x86_64** 操作系统上，部署于[赔钱云](https://www.peiqianyun.com/clientarea) 的 [NAT 实例](#12-网络与-nat-转发映射表)。

### 1.1 硬件规格
*   **CPU**: Intel Xeon Platinum 8272CL (3核) @ 2.593GHz
*   **内存**: 10GB (物理上限，分配方案：[主进程](#3-服务器主进程维护-234mc) 8G + [基岩转发](#4-基岩互通服务维护-bedlink) 2G)
*   **软件包源**: 已配置为阿里云 (Aliyun) 源

### 1.2 网络与 NAT 转发映射表
由于服务器处于 NAT 内网环境，所有外部访问必须通过 [赔钱云](https://www.peiqianyun.com/clientarea) 管理后台配置“端口转发”。目前的映射关系如下：

| 服务名称 | 公网访问地址 (外部) | 内部监听端口 | 协议 | 详细用途说明 |
| :--- | :--- | :--- | :--- | :--- |
| **SSH** | `160.202.254.36:10136` | 22 | TCP | 远程控制台管理 |
| **[Java版主进程](#3-服务器主进程维护-234mc)** | `160.202.254.36:10056` | 25565 | TCP | Java 版玩家连接地址 |
| **[基岩互通 (bedlink)](#4-基岩互通服务维护-bedlink)** | `160.202.254.36:10165` | 19132 | **UDP** | 基岩版 (手机/Win10) 接入 |
| **[语音聊天 (SVC)](#7-简单语音聊天模组-svc-维护)**| `160.202.254.36:10222` | 24454 | **UDP** | 游戏内 Simple Voice Chat 语音通信 |
| **[皮肤上传 API](#52-c-皮肤接口-skinapi)** | `160.202.254.36:10161` | 54259 | TCP | C# 编写的皮肤上传接口进程 |
| **[皮肤直链获取](#53-nginx-静态服务)** | `160.202.254.36:10159` | 54258 | TCP | Nginx 提供的直链访问及前端页面 |

### 1.3 域名解析
*   公网 IP `160.202.254.36` 已在 Cloudflare 后台解析至域名：`classicbyte.asia`。
*   玩家可以通过 `classicbyte.asia:10056` 尝试连接（具体取决于解析生效情况）。

---

## 2. 进程管理工具：Screen
服务器使用 `Screen` 挂起进程。

### 2.1 核心会话列表
使用 `screen -ls` 查看，主要包含以下会话：
*   **[`234mc`](#3-服务器主进程维护-234mc)**: 运行 Minecraft Java 版主进程。
*   **[`skinapi`](#52-c-皮肤接口-skinapi)**: 运行 C# 编写的皮肤上传接口服务。
*   **[`bedlink`](#4-基岩互通服务维护-bedlink)**: 运行 Geyser-Standalone 基岩版转发服务。

### 2.2 常用操作命令
*   **进入会话**: `screen -r [会话名称]`
*   **退出并保持运行**: 按下 `Ctrl + A`，然后松开按 `D` (Detach)。
*   **彻底强制终止**: 在会话窗口内按下 `Ctrl + C`。

---

## 3. 服务器主进程维护 (`234mc`)
### 3.1 概览
*   **核心路径**: `/root/mcserver/234mc`
*   **加载器**: Fabric (已加载性能优化、皮肤加载、[基岩互通](#4-基岩互通服务维护-bedlink)插件共 10 个)。
*   **主程序**: `fabric-server-launch.jar`

### 3.2 运行与更新
*   **启动命令**: 
    ```bash
    java -Xmx8G -Xms2G -jar fabric-server-launch.jar nogui
    ```
*   **正常关闭**: 在 `234mc` 会话中输入 `stop` 并回车，待提示保存完成后会自动结束。或者在进入了对应的screen会话后直接Ctrl+C也能关闭服务器。
*   **模组维护**: 若要更新模组，直接将新的 `.jar` 文件放入 `/root/mcserver/234mc/mods`，并删除对应的旧版文件，随后重启服务器。

---

## 4. 基岩互通服务维护 (`bedlink`)
### 4.1 运行机制
该服务独立运行 **Geyser-Standalone**。它作为一个中间网关，监听内部 UDP `19132` 端口，并将流量转发至[主服务器](#3-服务器主进程维护-234mc)的 `127.0.0.1:25565`。

### 4.2 维护操作
*   **会话**: `screen -r bedlink`
*   **启动方式**: 
	```
	java -Xmx2G -jar Geyser-Standalone.jar
	```

---

## 5. 皮肤系统维护 (C# API + Nginx)
该系统是一个闭环：玩家通过网页上传 -> [C# API](#52-c-皮肤接口-skinapi) 保存并通知游戏 -> [Nginx](#53-nginx-静态服务) 提供 HTTP 直链。

### 5.1 用户前端
*   **访问地址（不要用域名代替IP，否则无法访问）**:[http://160.202.254.36:10159](http://160.202.254.36:10159)
*   **功能**: 支持输入玩家 ID、选择手臂模型 (Classic/Slim) 以及上传 `.png` 皮肤。

### 5.2 C# 皮肤接口 (`skinapi`)
*   **源码逻辑**: 见 `SkinController.cs`。它监听 `54259` 端口。
*   **核心动作**: 
    1. 接收图片并重命名保存至 `/root/mcserver/player.skin.d`。
    2. 构造皮肤直链地址：`http://160.202.254.36:10159/[文件名].png`。
    3. 通过 **RCON** 协议 (端口 25575) 向 MC 服务器执行指令：
       `skin set web [model] "[url]" [playerName]`
* 以下是主要控制器（Controller）代码，由我自己编写，仅供参考

```csharp
using Microsoft.AspNetCore.Mvc;
using CoreRCON; // 引入 CoreRCON
using System.Net;
using System.IO;

namespace UploadPlayerSkin.Controllers
{
	[ApiController]
	[Route("api/skin")]
	public class SkinController : ControllerBase
	{
		// 1. 配置信息
		private readonly string _targetFolder = "/root/mcserver/player.skin.d/";
		private const string McServerIp = "127.0.0.1"; // MC服务器IP
		private const ushort McRconPort = 25575;      // RCON端口
		private const string McRconPassword = "NA,f`1Ya%/ggI|[4V%%[";
		private const string BaseUrl = "http://160.202.254.14:10159"; // 外部访问的基础URL

		[HttpPost("upload")]
		public async Task<IActionResult> UploadSkin(
			IFormFile file,
			[FromForm] string playerName,
			[FromForm] string skinModel) // <-- 新增参数接收：classic 或 slim
		{
			// --- 基础检查 ---
			if (file == null || file.Length == 0)
				return BadRequest("请选择一个文件");

			if (string.IsNullOrWhiteSpace(playerName))
				return BadRequest("请输入玩家名称");

			// 如果前端没传，默认设为粗手臂 classic
			if (string.IsNullOrWhiteSpace(skinModel))
				skinModel = "classic";

			var extension = Path.GetExtension(file.FileName).ToLower();
			if (extension != ".png")
				return BadRequest("只允许上传 .png 格式的皮肤文件");

			try
			{
				// 1. 确保目录存在
				if (!Directory.Exists(_targetFolder))
				{
					Directory.CreateDirectory(_targetFolder);
				}

				// 2. 保存文件 (保留你的随机数逻辑)
				string fileName = $"{playerName}.{DateTime.Now:yyyy-MM-dd-HH-mm-ss-fffff}.png";
				string filePath = Path.Combine(_targetFolder, fileName);

				using (var stream = new FileStream(filePath, FileMode.Create))
				{
					await file.CopyToAsync(stream);
				}

				// 3. 构造皮肤 URL
				string skinUrl = $"{BaseUrl}/{fileName}";

				// 4. 通过 RCON 执行命令，传入 skinModel (classic/slim)
				string rconResult = await ExecuteSkinCommand(playerName, skinUrl, skinModel);

				return Ok(new
				{
					message = $"皮肤已上传并尝试更新",
					fileName = fileName,
					url = skinUrl,
					model = skinModel,
					serverFeedback = rconResult
				});
			}
			catch (Exception ex)
			{
				return StatusCode(500, $"操作失败: {ex.Message}");
			}
		}

		/// <summary>
		/// 封装执行 RCON 命令的方法
		/// </summary>
		private async Task<string> ExecuteSkinCommand(string playerName, string url, string skinModel)
		{
			try
			{
				using var rcon = new RCON(IPAddress.Parse(McServerIp), McRconPort, McRconPassword);
				await rcon.ConnectAsync();

				// 将原本硬编码的 "classic" 替换为变量 {skinModel}
				// 结果示例：skin set web slim "http://..." PlayerName
				string command = $"skin set web {skinModel} \"{url}\" {playerName}";

				string response = await rcon.SendCommandAsync(command);
				return response;
			}
			catch (Exception ex)
			{
				return $"RCON命令执行失败: {ex.Message}";
			}
		}
	}
}
```
### 5.3 Nginx 静态服务
*   **作用**: 将 `/root/mcserver/player.skin.d` 目录映射至外部。
*   **配置文件**: `/etc/nginx/conf.d/share.conf` 下的配置已开启 `autoindex`。
*   **重启服务**: `systemctl restart nginx`。

---

## 6. 备份服务维护 (`restic`)
服务器使用 `restic` 工具进行增量备份。

### 6.1 定时任务 (Systemd Timer)
*   **备份频率**: 每天三次 (04:00, 12:00, 20:00)。
*   **脚本路径**: `/usr/local/bin/restic_auto_backup.sh` (仅备份 `world` 文件夹)。
*   **手动触发**: `sudo systemctl start restic-backup.service`
*   **查看快照列表**:
    ```bash
    sudo restic -r /srv/my_backups --password-file /root/.restic_pw snapshots
    ```
    *注：仓库密码为 `123456`。*

### 6.2 恢复数据流程
1.  **停止服务器**: `screen -r 234mc` 后输入 `stop`或者直接CTRL+C。
2.  **选择快照**: 使用 `snapshots` 命令找到目标 ID。
3.  **恢复**:
    ```bash
    sudo restic -r /srv/my_backups -p /root/.restic_pw restore [快照ID] --target /
	```
4.  **启动进程**: 重新运行 Java [启动命令](#32-运行与更新)。

---

## 7. 简单语音聊天模组 (SVC) 维护
服务器安装了 **Simple Voice Chat (SVC)** 模组，为玩家提供游戏内基于距离的3D语音以及群组语音通讯功能。该模组依赖于主进程运行，但网络传输是独立的。

### 7.1 运行机制与网络
*   **传输协议**: SVC 的音频流传输**强依赖 UDP 协议**。
*   **内网监听**: 模组在服务器内部监听端口 `24454`。
*   **外网映射**: 由于是在赔钱云 NAT 环境，已在云控制台将外网的 `10222` (UDP) 端口转发至内网的 `24454`。

### 7.2 核心配置参数
配置文件位于 `config/voicechat/voicechat-server.properties`。日常维护中请注意以下核心参数，切勿随意更改：

*   `port=24454`
    **绝对不要**将其设置为与游戏服务器相同的端口（如 -1 或 25565），否则会导致服务器与 UDP Query 冲突并崩溃。
*   `voice_host=classicbyte.asia:10222`
    这是下发给玩家客户端去连接的真实外网地址。**如果未来更换了域名或赔钱云映射的外网端口，必须同步修改此处并重启服务器。**
*   `max_voice_distance=48.0` / `whisper_distance=24.0`
    分别控制普通说话和悄悄话的最远收听距离（格数）。

### 7.3 常见问题排查
*   **玩家界面显示“语音未连接 (Voice Chat Not Connected)” 图标断开**：
    1. **检查映射协议**：前往赔钱云控制台，确认外部端口 `10222` 到内部 `24454` 的转发规则是 **UDP**，如果是 TCP 则完全无法通讯。
    2. **检查域名解析**：如果 `classicbyte.asia` 解析异常，玩家将无法连接语音主机。可临时将配置文件中的 `voice_host` 修改为 `160.202.254.36:10222` 并重启服务器。
    3. **防火墙**：确认 Debian 内部的 `ufw` 或 `iptables` 没有屏蔽 24454/udp 端口。

---

## 8. 维护注意事项与故障排除
1. 赔钱云服务商服务器不稳定，应该实时关注官方群内消息（我也想换服务商，但是赔钱云是最便宜性能最好的一个了），特别晚上网络延迟会很高。
2. 如果遇到网络延迟高的问题，使用[赔钱云BGP](http://bgp.peiqianyun.com/dashboard)进行负载均衡，也就是服务器延迟高的话用以下地址进入，延迟会低一点。
3. 负载均衡： 

| 地址 | 备注 |
| :--- | :--- |
| ea70d80d.peiqianyun.xyz:54255 |[Java版MC地址](#3-服务器主进程维护-234mc)|
| d3a6ab92.peiqianyun.xyz:54250 |[基岩版MC地址](#4-基岩互通服务维护-bedlink)|

---
*指南最后更新：2026年3月*
*维护者：[twitter@HYumerin]*