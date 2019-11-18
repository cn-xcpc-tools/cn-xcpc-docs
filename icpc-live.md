# icpc-live

## 版本

cn-xcpc-tools/icpc-live-v2
OBS Studio 24.0.3 + win-overlaymaster 插件

## 环境

Windows 10 1903
jdk8
Apache Maven 3.6.2
wlp.CDS-2.1.2100
Noto Sans CJK JP Normal（为所有用户安装）

## 安装与编译 icpc-live 及其依赖

```shell
git clone https://github.com/cn-xcpc-tools/icpc-live-v2
cd icpc-live-v2
mvn install
```

## 配置

首先从 `icpc-live/config/sample` 中将 `events.properties` 与 `mainscreen.properties` 复制到 `icpc-live/config`。

### icpc-live/config/events.properties

* login 与 password

设置为 CDS 的 admin 用户。

* url

`https://cds/api/contests/{id}`

* problemsNumber

设置为题目数量。

### icpc-live/src/main/resources/users.data

为 Web 导播台设置用户名与密码，格式为 `{username}:{password}`。

## 启动

运行 `webserver.bat`，在其开始读取 CDS API 后运行 `main-screen.bat`，在 `c:\work\image.bin` 生成后运行 OBS 并添加 `Overlay Master` 项。

## 导播台

由 `http://127.0.0.1:8080` 进入导播台。

* 在 Advertisment 中设置 `#Clock#` 即可在左下角显示比赛时间。

* 在 Creeping Line 中设置 `#Standings#` 即可在底部黑条中显示靠前名次的滚动排行榜。

* 在 Standing 中可控制完整排行榜的显示和隐藏。

## Troubleshooting

* icpc-live 不会读取 team 的隐藏设置

沈阳站时的临时方案为在源码中将系统自带 DOMjudge 用户的提交均设置为封榜后的“？”状态。
应考虑修改 EventLoader 部分源码以将该用户在 team 的读入时跳过，或在启动 CDS 前保证该用户不在该比赛中。

## 未解决的问题

* OBS 的“图像幻灯片放映”功能不能正确地循环所有选择的图像文件，而是在某几张图像内反复播放。
