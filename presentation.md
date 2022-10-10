# Presentation Client / Admin

### 介绍

presClient 用于大屏幕内容，presAdmin 用于管理不同大屏展示内容。展示内容包括各种统计。非常建议现场赛大屏幕使用。

在使用 ICPC Tools 工具链之前，需要配置一个 CDS 实例。具体用途可以参考 [CDS配置](./cds.md)。

### 版本

一般在 [Releases](https://github.com/icpctools/icpctools/releases) 中下载最新版的客户端 `presentations-2.x.xxx.zip` 和管理端 `presentationAdmin-2.x.xxx.zip`。建议不使用预览版。

### 环境

两者在安装了 JRE8 的 Ubuntu / Windows 机器上均可运行。记得先安装 CDS。

### 管理端 presentationAdmin

架设到方便操作的机器上。启动命令行为 `./presAdmin https://cds presAdmin 密码`。注意同时只能打开一个，否则会出现问题。[文档参考](https://tools.icpc.global/docs/PresentationAdmin.pdf)

### 客户端 presentations

架设到连接大屏幕的机器上，并接入 CDS 所在网络。

启动命令为 `./client https://cds presentation 密码 --name "Screen 1" --display 1 --display_name "{team.display_name} ({org.formal_name})"`。[文档参考](https://tools.icpc.global/docs/PresentationClient.pdf)

在较新的版本中，可能需要在链接后额外添加`/API/`，即启动命令变为：`./client https://cds/api/ presentation 密码 --name "Screen 1" --display 1 --display_name "{team.display_name} ({org.formal_name})"`。(例如：`./client https://192.168.1.10:8443/api/ presentation presentat1on --name "Screen 1" --display 1 --display_name "{team.display_name} ({org.formal_name})"`(cds部署在局域网下，服务器IP为`192.168.1.10`，CDS服务器监听端口为`8443`, 账户名为`presentation`, 密码为`presentat1on`)

此外，默认策略下，不允许使用管理员账户(`admin`)登录presentations client。

由于自带字体不支持中文，需要设置 `ICPC_FONT` 环境变量。一般设置 DengXian 或者 Noto Sans CJK。

请注意，如果你需要在一台电脑上开启多个 presClient，请将程序文件夹复制多个，并在不同的文件夹中启动，否则数据可能出现混乱。

### 使用

在 `presAdmin` 中选中需要控制的 presentation，并在右侧选择需要令其显示的内容，然后点击 `Apply` 按钮。

### Troubleshooting

* `presAdmin` 右侧不显示可选的内容

  排查 CDS 是否正常运行、CDS 配置是否正确、`presAdmin` 是否与 CDS 在网络上能够通信。

* `client` 启动后，`presAdmin` 中未显示该 presentation

  排查 `client` 是否与 CDS 在网络上能够通信。

* 沈阳站场馆大屏幕的比例为 32:9，仅显示所连接 16:9 机器画面的上半部分

  此时在 `presAdmin` 中菜单栏内设置 presentation 的显示区域为 `(0,0,1920,540)` 即能使其完全在大屏幕上显示出来。

  或者在 `presClient` 启动时设置参数 `--display 1a`，其中 `1` 表示 Windows 系统的显示器 ID，`a` 表示屏幕左上角，还可以使用 `b`、`c`、`d`。

* 网络拓扑设计

  由于赛场一般比较空旷，可以考虑搞一个无线路由器并能够与 CDS 通信，然后控制室可以连接 WiFi，放置一台电脑用于展示，管理端使用 WiFi 接入。其实裁判也可以用自己笔记本连入 WiFi 答疑等。
