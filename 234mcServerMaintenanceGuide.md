# 服务器维护指南

## 服务器概述
这个Minecraft服务器运行在Linux环境之上，其中服务器的操作系统是Debian 12

这个Minecraft服务器使用Fabric作为模组加载器 , 并且已经加载了10个模组,包括性能优化模组还有皮肤加载模组和基岩互通模组

### 服务器操作系统概述

此服务器使用Debian12作为服务器的操作系统。具体配置如下：
* CPU：`Intel Xeon Platinum 8272CL (3) @ 2.593GHz`
* 内存大小 : 10GB
* 软件包源 : 阿里源
* OS : `Debian GNU/Linux 12 (bookworm) x86_64`

目前服务器使用"赔钱云" , 在这里管理服务器实例(账号密码找我要)
[赔钱云](https://www.peiqianyun.com/clientarea)

>### 注意
>如果换用服务器商，请使用能够完全操作其linux环境或者Windows环境的服务器，否则按照当前解决方案只能运行“服务器主进程”这个服务。

这个服务器是NAT机(并且提供50个NAT端口) , 也就是如果要从互联网访问这个服务器内的任何服务需要到[赔钱云](https://www.peiqianyun.com/clientarea)进行配置端口转发,把你的服务监听的端口在[赔钱云](https://www.peiqianyun.com/clientarea)配置转发到外部

### 服务器主要服务概述
此Minecraft服务器使用`Screen`工具将进程放到后台运行,你也可以尝试使用`systemctl`来代替`screen` , 但是我懒得搞

以下是服务器中运行的会话列表,使用`screen -r [会话名称]`来切换到对应的会话 , 你可以尝试搜索`screen`工具的使用说明
* `234mc` : 游戏服务器的主进程所在会话
* `skinapi` : 由我自己编写的皮肤上传接口所在会话
* `bed` : 基岩互通服务所在会话

除了这些主要服务会话,还有**定时任务**与`nginx`服务

其中定时任务是用来**备份游戏存档**的 , 每天3次备份 , 时间分别是04:00 12:00 20:00

`nginx`服务是用来把`\root\mcserver\player.skin.d`文件夹里面的内容映射到互联网让其内容能被外部使用http来访问 ，也就是让里面的皮肤文件能够使用http直链访问

---

## 服务器主进程维护

### 概述
服务器主进程在路径`/root/mcserver/234mc`中，里面是标准的Java版Minecraft服务器（使用Fabric Mod加载器）目录结构

## 皮肤接口进程维护

## 基岩互通进程维护

## `nginx` 服务维护

## 备份服务维护