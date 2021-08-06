# Domjudge judgehost

## 版本

Domjudge 7.3.3

## 环境

Ubuntu 20.04 LTS / 21.04

## 准备工作

### 安装依赖包和功能

**请注意下文的 `Troubleshooting` 中第 1 节的问题！**

```shell
sudo apt-get upgrade && sudo apt-get update
```

```shell
sudo apt install make sudo debootstrap libcgroup-dev lsof \
      php-cli php-curl php-json php-xml php-zip procps \
      gcc g++ default-jre-headless default-jdk-headless \
      ghc fp-compiler
```

### 编译 Domjudge

```shell
cd Downloads
wget https://www.domjudge.org/releases/domjudge-7.3.3.tar.gz
```

```shell
tar -zxvf domjudge-7.3.3.tar.gz
```

```shell
cd domjudge-7.3.3
./configure --prefix=/opt/domjudge --with-baseurl=127.0.0.1
make judgehost
sudo make install-judgehost
```

这会将 judgehost 安装在 `/opt/domjudge/judgehost` 里。  
(make install-judgehost 会提示找不到 etc/restapi.secret ，可忽略，会在下面进行配置)

## 配置 judgehost

### 添加用户

```shell
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run
# 如果 judgehost 拥有多个 CPU 核心，你可以添加额外的用户来支持绑定，但是要记得修改 judgehost-static.php 等文件里的 `RUNUSER` 。
# 不同的 judgehost 进程到不同的 CPU 核心上，如下：
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-0
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-1
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-2
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-3
# ... 如果有更多的 CPU 核心，请自行添加更多的用户
```

### 配置 sudoers

将 /opt/domjudge/judgehost/etc/sudoers-domjudge 复制到 /etc/sudoers.d/ 目录下。

```shell
sudo cp /opt/domjudge/judgehost/etc/sudoers-domjudge /etc/sudoers.d/
```

### 修改 rest 密码

使用 `vim`、`nano` 等文本编辑器编辑 `/opt/domjudge/judgehost/etc/restapi.secret` 这个文件。文件的格式为：

```text
default http://example.edu/domjudge/api/  judgehosts  MzfJYWF5agSlUfmiGEy5mgkfqU
```

格式为 `endpoint api_url username password`，`endpoin` 可以保持不变，`api_url` 根据 `domserver` 的地址进行修改，`username` 和 `password` 要与 `domserver` 上的 `etc/restapi.secret`  保持一致。

### 构建 chroot 环境

使用 `vim`、`nano` 等文本编辑器编辑 `/opt/domjudge/judgehost/bin` 目录下的 `dj_make_chroot` 脚本，搜索 `mirror` 这个关键字，并更改搜索到的 ubuntu 的 mirror 更换为国内源（例如清华源， `http://mirrors.tuna.tsinghua.edu.cn/ubuntu/`）（注意，脚本中除了 ubuntu mirror 还有 debian mirror 的配置，不要改错了），紧跟着 mirror 配置的下面有 proxy 代理服务器的配置，因为这一步需要访问网络，若需要配置代理服务器请按需设置。  
修改之后**保存并运行此脚本**。这一步会从源上下载必要的软件包，所以请耐心等待。

### 设置 cgroup

使用 `vim`、`nano` 等文本编辑器编辑 `/etc/default/grub` 这个文件，对其中的这一行做如下修改：

```shell
GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory swapaccount=1"
```


**注意**：如果你在使用由 Google Cloud Platform 或 DigitalOcean 提供的虚拟主机，请注意官方文档中的 `On VM hosting providers such as Google Cloud or DigitalOcean, GRUB_CMDLINE_LINUX_DEFAULT may be overwritten by other files in /etc/default/grub.d/` 请 cd 进入该目录后，在里面的 `.conf` 文件里进行相关的更改。


然后执行：

```shell
sudo update-grub
```

之后**重启计算机**。

### 启动 judgehost

如果需要使用 cgroup，则每次重启之后都要运行 `/opt/domjudge/judgehost/bin/create_cgroups` 和 `/opt/domjudge/judgehost/bin/judgedaemon` 后即可启动，若提示 `error: Call to undefined function curl_init()`，则可以安装 php-curl 解决  


### 配置多 judgehost 的 systemd 及 rsyslog

使用 `vim`、`nano` 等文本编辑器在 `/lib/systemd/system` 下新建一个文本文件叫做 `create-cgroups.service`，写入下列内容：

```shell
[Unit]
Description=Make sure cgroups exist for DOMjudge judgedaemon

[Service]
Type=oneshot
ExecStart=/opt/domjudge/judgehost/bin/create_cgroups
RemainAfterExit=true
```

在 `/lib/systemd/system` 下再新建一个文本文件叫做 `domjudge-judgehost@.service`，写入下列内容：
注意 `User=<username>` 要用自己编译 `judgehost` 时的用户名，因为 `/etc/sudoers.d/sudoers-domjudge` 列表里是当时的用户

```shell
[Unit]
Description=DOMjudge JudgeDaemon
Requires=create-cgroups.service
After=create-cgroups.service
After=network.target

[Service]
Type=simple

ExecStart=/opt/domjudge/judgehost/bin/judgedaemon -n %i
User=<username>

Restart=always
RestartSec=3
PrivateTmp=yes

StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=judgehost-%i

[Install]
WantedBy=multi-user.target
```

在 `/etc/rsyslog.d/` 下新建一个文本文件叫做 `judgehost.conf`，写入下列内容：

```conf
:programname, isequal, "judgehost-0" /var/log/judgehost/judgehost-0.log
:programname, isequal, "judgehost-1" /var/log/judgehost/judgehost-1.log
:programname, isequal, "judgehost-2" /var/log/judgehost/judgehost-2.log
:programname, isequal, "judgehost-3" /var/log/judgehost/judgehost-3.log
```

重启日志服务，启动四个 `judgehost`：

```shell
sudo systemctl restart rsyslog
sudo systemctl start domjudge-judgehost@0
sudo systemctl start domjudge-judgehost@1
sudo systemctl start domjudge-judgehost@2
sudo systemctl start domjudge-judgehost@3
```

`judgedaemon`的日志会保存在 `/var/log/judgehost` 下

## Troubleshooting

### 1.关于 judgehost 评测语言所需编译器的安装问题

由于 `judgehost` 的运行机理，请先在宿主机上 **安装好需要用到的语言的编译器** ，再进行 `judgehost` 的安装。或者在安装好后， `chroot` 进入对应的 `chroot` 目录后，再进行新语言编译器的安装。

### 2.关于题目限制内存大小与 JAVA 报错的问题

由于 DOMjudge 的机制，其对于 JAVA 评测时的 **限制内存大小** 指的是 `JVM堆+栈+永久区` 的合计大小（其他的例如 HUSTOJ，其对于 JAVA 的评测的限制内存大小指的是 `JVM堆` 的大小），所以建议每题的 **限制内存大小** 最少从 **256MB** 起步，推荐 `512MB` 以上或者 `Default` 。

### 3.关于单个 judgehost 进程后台保活的问题

因为 judgehost 进程需要一直维持，如果你是运行在 VPS 等无法一直保持开启终端窗口的环境下时，建议您使用 `screen` 来保活 judgehost 进程。
