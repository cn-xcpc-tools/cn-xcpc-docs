# Domjudge 简易搭建文档（judgehost 部分）
## 版本
Domjudge 6.0.2

## 环境
Ubuntu 18.04，全新安装的系统（Judge 机）。
关于 Ubuntu 的批量网络安装方式，可以参考[这篇博客](https://blog.cool2645.com/281)。

## 准备工作
### 安装依赖包和功能
```shell
sudo apt-get upgrade && sudo apt-get update
```

```shell
sudo apt install make sudo debootstrap libcgroup-dev \
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
wget https://www.domjudge.org/releases/domjudge-6.0.2.tar.gz
```
```shell
tar -zxvf domjudge-6.0.2.tar.gz
```
```shell
cd domjudge-6.0.2
./configure --prefix=$HOME/domjudge --with-baseurl=127.0.0.1
make judgehost && sudo make install-judgehost
```
这会将 judgehost 安装在当前用户家目录下的 domjudge/judgehost 里。

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
将 $HOME/domjudge/judgehost/etc/sudoers-domjudge 复制到 /etc/sudoers.d/ 目录下。
```shell
sudo cp $HOME/domjudge/judgehost/etc/sudoers-domjudge /etc/sudoers.d/
```

### 修改 rest 密码
使用 vim 等文本编辑器编辑 domjudge 目录下 etc/restapi.secret 这个文件。文件的格式为：
```text
default http://example.edu/domjudge/api/  judgehosts  MzfJYWF5agSlUfmiGEy5mgkfqU
```
格式为 endpoint api_url username password ，endpoint 可以保持不变，api_url 根据 judgeserver 的地址进行修改，username 和 password 要与配置 judgeserver 的时候设置保持一致。

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

### 配置 supervisor
使用 vim 等文本编辑器在 /etc/supervisor/conf.d 下新建一个文本文件叫做 judgehost.conf，写入下列内容： 
```shell
command=/opt/domjudge/judgehost/bin/judgedaemon -n %(process_num)d
autorestart=true
process_name=%(program_name)s_%(process_num)02d
stdout_logfile=/var/log/judgehost/judgehost.log
stderr_logfile=/var/log/judgehost/judgehost.err.log
numprocs=4
numprocs_start=0
```
注：上面的 numprocs 应该根据实际的 CPU 核心数。另外请根据实际情况配置 stdout_logfile 和 stderr_logfile.
最后，使用如下命令重新加载 supervisor ：
```shell
sudo systemctl restart supervisord.service
```
DONE！可以使用如下命令查看 judgehost 进程的运行情况：
```shell
sudo supervisorctl status
```
