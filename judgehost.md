# DOMjudge 评测机

## 前言

### 版本

DOMjudge 7.3.3

### 环境

Ubuntu 18.04 LTS / 20.04 LTS / 21.04

## 准备工作

### 安装依赖包和功能

**请注意下文的 `Troubleshooting` 中第 1 节的问题！**

```shell
sudo apt upgrade
sudo apt update

sudo apt install make unzip debootstrap libcgroup-dev lsof \
      php-cli php-curl php-json php-xml php-zip procps \
      gcc g++ default-jre-headless default-jdk-headless \
      ghc fp-compiler
```

### 编译 judgehost

请注意，此步骤尽量在非 root 用户下运行。如果使用 root 用户，则也需要创建一个新的用户并修改 `./configure` 的参数。

```shell
cd Downloads
wget https://www.domjudge.org/releases/domjudge-7.3.3.tar.gz
tar -zxvf domjudge-7.3.3.tar.gz

cd domjudge-7.3.3
./configure
make judgehost
sudo make install-judgehost
```

这会将 judgehost 安装在 `/opt/domjudge/judgehost` 里。

`make install-judgehost` 可能会提示找不到 `etc/restapi.secret`，可忽略，会在下面进行配置。

## 配置 judgehost

### 添加用户

```shell
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run

# 如果 judgehost 拥有多个 CPU 核心，你可以添加额外的用户来支持绑定，即执行完上一条命令后执行以下useradd命令。
# 不同的 judgehost 进程绑定到不同的 CPU 核心上，如下：
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-0
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-1
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-2
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-3
# ... 如果有更多的 CPU 核心，请自行添加更多的用户
```

### 配置 sudoers

```shell
sudo cp /opt/domjudge/judgehost/etc/sudoers-domjudge /etc/sudoers.d/
```

### 修改 rest 密码

使用 `vim`、`nano` 等文本编辑器编辑 `/opt/domjudge/judgehost/etc/restapi.secret` 这个文件。文件形如：

```text
default http://example.edu/domjudge/api/  judgehosts  MzfJYWF5agSlUfmiGEy5mgkfqU
```

格式为 `endpoint api_url username password`，其中用 tab 进行缩进。`endpoint` 可以保持不变，`api_url` 根据 domserver 的地址进行修改，`username` 和 `password` 一般从 domserver 上的 `etc/restapi.secret` 读取，该文件会在安装 domserver 时预生成，其实也可以在后台创建自己的 judgehost 用户或者修改 judgehost 密码。可以用这种方式连接多台 domserver。

### 构建 chroot 环境

如果你的服务器在国内，访问国外速度较慢，则考虑更换 chroot 环境中 apt 的源为合适的镜像。

```shell
sudo sed -i 's,http://us.archive.ubuntu.com./ubuntu/,http://cn.archive.ubuntu.com/ubuntu,g' /opt/domjudge/judgehost/bin/dj_make_chroot
```

其中 `cn.archive.ubuntu.com` 也可以更换成其他源，例如 `mirrors.tuna.tsinghua.edu.cn`、`mirrors.aliyun.com`、`mirrors.cloud.aliyuncs.com`（在阿里云 ECS 上推荐）、`azure.archive.ubuntu.com`（在 Azure 上推荐）……

如果需要请求代理服务器，那么请编辑 `/opt/domjudge/judgehost/bin/dj_make_chroot` 文件并找到 proxy 相关设置进行修改。

完成上述修改后**保存并运行此脚本**。这一步会从源上下载必要的软件包，所以请耐心等待。

### 设置 cgroup

使用 `vim`、`nano` 等文本编辑器编辑 `/etc/default/grub` 这个文件，对其中的 `GRUB_CMDLINE_LINUX_DEFAULT` 中添加 `cgroup_enable=memory swapaccount=1`，例如：

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory swapaccount=1"
```

**注意**：如果你在使用由 Azure、Google Cloud Platform 或 DigitalOcean 提供的虚拟服务器，请注意官方文档中的 `On VM hosting providers such as Google Cloud or DigitalOcean, GRUB_CMDLINE_LINUX_DEFAULT may be overwritten by other files in /etc/default/grub.d/` 请 cd 进入该目录后，在里面的 `.conf` 文件里进行相关的更改。

修改完成后执行：

```shell
sudo update-grub
```

之后**重启计算机**。重启计算机后可以通过 `cat /proc/cmdline` 中是否有 `cgroup_enable=memory swapaccount=1` 验证是否安装成功。

### 测试启动 judgehost

在每次开机后，需要运行以下脚本初始化 cgroups。

```shell
sudo /opt/domjudge/judgehost/bin/create_cgroups
```

然后可以通过

```shell
/opt/domjudge/judgehost/bin/judgedaemon
```

启动评测终端。在需要启动多个终端时应该使用 `-n X` 参数，其中 0 <= X < 计算机核数。

### 利用 systemd 配置开机自启动

```shell
sudo cp /opt/domjudge/lib/systemd/system/domjudge-judgehost.service /opt/domjudge/lib/systemd/system/domjudge-judgehost@.service
sudo sed -i 's/judgedaemon -n 0/judgedaemon -n %i/g' /opt/domjudge/lib/systemd/system/domjudge-judgehost@.service
sudo ln -s /opt/domjudge/lib/systemd/system/domjudge-judgehost@.service /lib/systemd/system/
sudo ln -s /opt/domjudge/lib/systemd/system/create-cgroups.service /lib/systemd/system/
sudo systemctl enable create-cgroups
```

启动四个 `judgehost`（按需开启，照葫芦画瓢即可）：

```shell
sudo systemctl enable domjudge-judgehost@0
sudo systemctl enable domjudge-judgehost@1
sudo systemctl enable domjudge-judgehost@2
sudo systemctl enable domjudge-judgehost@3
```

`judgedaemon`的日志会保存在 `/opt/domjudge/judgehost/log/` 下。可以通过 `sudo systemctl status domjudge-judgehost@0` 或者 `journalctl -u domjudge-judgehost@0` 查看日志。

## Troubleshooting

- 关于 judgehost 评测语言所需编译器的安装问题

  先在运行 judgedaemon 的这个机器（宿主机器）上**安装好需要用到的语言的编译器**，保证可以通过该语言的 compile 脚本调用，然后 `sudo chroot /chroot/domjudge` 后利用 apt 安装你需要的语言编译器和运行时的安装。

- 关于题目限制内存大小与 JAVA 报错的问题

  由于 DOMjudge 的机制，其对于 JAVA 评测时的内存限制大小指的是 *JVM堆+栈+永久区* 的合计大小（其他的例如 HUSTOJ，其对于 JAVA 的评测的内存限制大小指的是 *JVM堆* 的大小），所以如果需要开启 Java 语言判题，则必须每题的限制内存大小最少从 256MB 起步，推荐 512MB 以上，否则 Java 提交的判题在触发后被禁用。

- 关于单个 judgehost 进程后台保活的问题

  因为 judgehost 进程需要一直维持，如果你是运行在 VPS 等无法一直保持开启终端窗口的环境下时，建议使用 `screen` 来保活 judgehost 进程。

- 想要 judgehost 能够返回 MLE 的评测结果

  非常不推荐，因为有时无法区分是*预申请了几乎所有内存后以其他方式发生运行错误*还是*申请内存超过限制返回空指针却没有进行异常处理而导致错误*，此处与*程序无法感知内存限制时使用内存超过限制*的语义可能不符。

  如果你确定需要的话，请参考这个 [commit](https://github.com/namofun/judgehost/commit/cbf72656a4a7c481b9a11415b01bc9811de2cdcc#diff-5d7612b63b2d5e7da8f4277055da1bb1509ddf947bb1f797016d5a448bb8d909) 修改 `testcase_run.sh` 文件。

- Bonus：Namomo OJ 是如何处理的评测机？

  如果你想要部署更简单，可以参考 [namofun/judgehost](https://github.com/namofun/judgehost) 的部署脚本。

  如果你想要有其他程序表现评测机运行状态，可以试试 [namofun/jethub](https://github.com/namofun/jethub)。

