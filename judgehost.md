# Domjudge judgehost

## 版本

Domjudge 7.1.1

## 环境

Ubuntu 18.04.3 LTS，全新安装的系统（Judge 机）。
关于 Ubuntu 的批量网络安装方式，可以参考[这篇博客](https://blog.cool2645.com/posts/2018-icpc-shenyang-env-config-log/)。

## 准备工作

### 安装依赖包和功能

```shell
sudo apt-get upgrade && sudo apt-get update
```

```shell
sudo apt install make sudo debootstrap libcgroup-dev lsof \
        php-cli php-curl php-json php-xml php-zip procps \
        gcc g++ openjdk-8-jre-headless \
        openjdk-8-jdk ghc fp-compiler \
        libcurl4-gnutls-dev libjsoncpp-dev libmagic-dev \
        supervisor
```

注：这里的 supervisor 用来管理 judgehost 的评测进程，如果你愿意，也可以换成现代系统自带的 systemd 来管理。

### 编译 Domjudge

```shell
cd Downloads
wget https://www.domjudge.org/releases/domjudge-7.1.1.tar.gz
```

```shell
tar -zxvf domjudge-7.1.1.tar.gz
```

```shell
cd domjudge-7.1.1
./configure --prefix=/opt/domjudge --with-baseurl=127.0.0.1
make judgehost && sudo make install-judgehost
```

这会将 judgehost 安装在 `/opt/domjudge/judgehost` 里。  
(make install-judgehost 会提示找不到 etc/restapi.secret ，可忽略，会在下面进行配置)

## 配置 judgehost

### 添加用户

```shell
useradd -d /nonexistent -U -M -s /bin/false domjudge-run
# 如果 judgehost 拥有多个 CPU 核心，你可以添加额外的用户来支持绑定
# 不同的 judgehost 进程到不同的 CPU 核心上，如下：
useradd -d /nonexistent -U -M -s /bin/false domjudge-run-0
useradd -d /nonexistent -U -M -s /bin/false domjudge-run-1
useradd -d /nonexistent -U -M -s /bin/false domjudge-run-2
useradd -d /nonexistent -U -M -s /bin/false domjudge-run-3
# ... 如果有更多的 CPU 核心，请自行添加更多的用户
```

### 配置 sudoers

将 /opt/domjudge/judgehost/etc/sudoers-domjudge 复制到 /etc/sudoers.d/ 目录下。

```shell
sudo cp /opt/domjudge/judgehost/etc/sudoers-domjudge /etc/sudoers.d/
```

### 修改 rest 密码

使用 vim 等文本编辑器编辑 /opt/domjudge/judgehost 目录下 etc/restapi.secret 这个文件。文件的格式为：

```text
default http://example.edu/domjudge/api/  judgehosts  MzfJYWF5agSlUfmiGEy5mgkfqU
```

格式为 endpoint api_url username password ，endpoint 可以保持不变，api_url 根据 judgeserver 的地址进行修改，username 和 password 要与 judgeserver 上的 etc/restapi.secret 保持一致。

### 构建 chroot 环境

使用 vim 等文本编辑器编辑 domjudge 目录下 bin/dj_make_chroot 脚本，搜索 mirror 这个关键字，并更改搜索到的 ubuntu 的 mirror 为国内源（例如清华源，mirrors.tuna.tsinghua.edu.cn/ubuntu）（注意，脚本中除了 ubuntu mirror 还有 debian mirror 的配置，不要改错了），紧跟着 mirror 配置的下面有 proxy 代理服务器的配置，因为这一步需要访问网络，若需要配置代理服务器请按需设置。  
修改之后保存并运行此脚本。这一步会从源上下载必要的软件包，所以请耐心等待。

### 设置 cgroup

使用 vim 等文本编辑器编辑 /etc/default/grub 这个文件，对其中的这一行做如下修改：

```shell
GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory swapaccount=1"
```

然后执行：

```shell
update-grub
```

之后**重启计算机**。

### 启动 judgehost

如果需要使用cgroup，则每次重启之后都要运行 `/opt/domjudge/judgehost/bin/create_cgroups`  
`/opt/domjudge/judgehost/bin/judgedaemon` 即可启动，若提示 `error: Call to undefined function curl_init()`，则可以安装 php-curl 解决  

### 配置多 judgehost 的 systemd 及 rsyslog

使用 vim 等文本编辑器在 /lib/systemd/system 下新建一个文本文件叫做 create-cgroups.service，写入下列内容：

```shell
[Unit]
Description=Make sure cgroups exist for DOMjudge judgedaemon

[Service]
Type=oneshot
ExecStart=/opt/domjudge/judgehost/bin/create_cgroups
RemainAfterExit=true
```

在 /lib/systemd/system 下再新建一个文本文件叫做 domjudge-judgehost@.service，写入下列内容：
注意 `User=<username>` 要用自己编译 judgehost 时的用户名，因为 /etc/sudoers.d/sudoers-domjudge 列表里是当时的用户

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

在 /etc/rsyslog.d/ 下新建一个文本文件叫做 judgehost.conf，写入下列内容：

```conf
:programname, isequal, "judgehost-0" /var/log/judgehost/judgehost-0.log
:programname, isequal, "judgehost-1" /var/log/judgehost/judgehost-1.log
:programname, isequal, "judgehost-2" /var/log/judgehost/judgehost-2.log
:programname, isequal, "judgehost-3" /var/log/judgehost/judgehost-3.log
```

重启日志服务，启动四个 judgehost：

```shell
sudo systemctl restart rsyslog
sudo systemctl start domjudge-judgehost@0
sudo systemctl start domjudge-judgehost@1
sudo systemctl start domjudge-judgehost@2
sudo systemctl start domjudge-judgehost@3
```

judgedaemon的日志会保存在 /var/log/judgehost 下

## Troubleshooting
