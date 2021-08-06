# DOMjudge 的 domserver 和 judgehost 的 docker 安装



## 前置：若在国内使用，建议更换 docker-hub 源以加速镜像下载

- 若要在国内安装运行，在安装好 `docker.io` ，`docker` 后，建议执行以下命令更换使用国内访问速度更快的 docker-hub 源地址。

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": [
        "https://reg-mirror.qiniu.com",
        "https://mirror.ccs.tencentyun.com",
        "http://hub-mirror.c.163.com"
    ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```





## Mariadb 数据库

### 命令解释

- `MYSQL_ROOT_PASSWORD` 和 `MYSQL_PASSWORD` 一定要确保后续命令中的相关项与其所对应的值一致。具体的值可以随机生成一组强度足够的密码。容器名字为 `dj-mariadb` ，MySQL 创建用户 `domjudge` ，密码对应 `[passwd2]` ，root 密码对应 `[passwd1]` ，容器时区设置为 `Asia/Shanghai` ，创建数据库 `domjudge` ，容器外部数据库端口设置为 `13306` ，数据库最大同时连接数为 `1000` 。

- 特别注意，设置 MariaDB 密码不能有特殊字符（最好是大小写字母加数字，否则会搭建失败）。

```shell
sudo docker run -it --name dj-mariadb -e MYSQL_ROOT_PASSWORD=[passwd1] -e MYSQL_USER=domjudge -e CONTAINER_TIMEZONE=Asia/Shanghai -e MYSQL_PASSWORD=[passwd2] -e MYSQL_DATABASE=domjudge -p 13306:3306 mariadb --max-connections=1000
```





## DOMjudge 前端

### 命令解释

- `MYSQL_ROOT_PASSWORD` 和 `MYSQL_PASSWORD` 一定要第一条命令中的值一致。容器 link 操作 `dj-mariadb:mariadb` ，容器 MySQL 主机地址设置为 `mariadb` ，容器名为 `domserver` ，MySQL 用户为 `domjudge` ，密码对应 `[passwd2]` ，root 密码对应 `[passwd1]` ，容器时区设置为 `Asia/Shanghai` ，容器对外开放端口 `80` ，本次指令中容器从 docker-hub 上拉取的 DOMjudge 镜像版本为 `7.3.2` ，这个版本必须与下方 `judgehost` 创建指令中的一致。

- 关于性能，根据北京理工的经验，千兆网络，36 核 80G 平均 CPU 占用不超过 50%，内存也不超过 3G。
- 根据黑龙江大学的经验，千兆内网、`domserver` 与 `judgehost` 所在实体机分离的情况下，`domserver` 分配 6 核 16G、`judgehost` 分别在 2 台实体机上各运行 2 个，且 2 台实体机各为 4 核 6G，对于 150+ teams、 5H 大概 3K 的提交，没有问题（题目的测试数据点均在 30 个以内，题目内存限制均在 512MB 以内，题目时间限制均在 2s 以内）。

```shell
sudo docker run --link dj-mariadb:mariadb -it -e MYSQL_HOST=mariadb -e MYSQL_USER=domjudge -e MYSQL_DATABASE=domjudge -e CONTAINER_TIMEZONE=Asia/Shanghai -e MYSQL_PASSWORD=[passwd2] -e MYSQL_ROOT_PASSWORD=[passwd1] -p 80:80 --name domserver domjudge/domserver:7.3.2
```





## 获取 DOMjudge 的 admin 和 judgehost 的账号密码

- 在 `domserver` 容器开始运行后，会显示出两行密码，形式如下所示。其中，`[passwd3]` 是登录 `admin` 账号时所需要的密码，`[passwd4]` 是下面创建 `judgehost` 的命令中需要保证一致的、评测机账号的密码。

```shell
Initial admin password is [passwd3]
Initial judgehost password is [passwd4]
```





## 修改 grub

- 用 `nano` 打开 `/etc/default/grub` ，找到 `GRUB_CMDLINE_LINUX_DEFAULT` 并修改为 `GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory swapaccount=1"` ，保存修改后执行 `sudo update-grub` 然后重启。重启完成后运行 `/proc/cmdline` 检查操作是否已经生效。在某些云计算提供商的云服务器上，像 Google Cloud 或 DigitalOcean ，`GRUB_CMDLINE_LINUX_DEFAULT` 可能会被 `/etc/default/grub.d/` 里的某个文件的给覆盖了，需要修改这个目录下的文件并执行 `sudo update-grub` 以实现修改。





## Judgehost 评测机

### 命令解释

- 容器名为 `judgehost-0` ，容器 link 操作 `domserver:domserver` ，判题机命名为 `judgedaemon-0` ，对应ID为 `0` ，`JUDGEDAEMON_PASSWORD` 要确保与上面获得的 `jusgehost` 的账号密码 `[passwd4]` 一致，容器时区设置为 `Asia/Shanghai` ，容器拉取镜像版本要和 `domserver` 的确保一致，本命令使用的版本是 `7.3.2` 。

```shell
sudo docker run -it --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro --name judgehost-0 --link domserver:domserver --hostname judgedaemon-0 -e DAEMON_ID=0 -e JUDGEDAEMON_PASSWORD=[passwd4] -e CONTAINER_TIMEZONE=Asia/Shanghai domjudge/judgehost:7.3.2
```

- `Judgehost` 可以运行多个，但是要确保 `--name` 、`--hostname` 、`DAEMON_ID` 为不重复关键字，即可。

```shell
sudo docker run -it --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro --name judgehost-1 --link domserver:domserver --hostname judgedaemon-1 -e DAEMON_ID=1 -e JUDGEDAEMON_PASSWORD=[passwd4] -e CONTAINER_TIMEZONE=Asia/Shanghai domjudge/judgehost:7.3.2
```

- 若要使用分布式评测，建议 `domserver` 和 `dj-mariadb` 放在同一台机器上、同一系统内，`judgehost` 可以安装在别的机器上，以减轻前、后端的系统资源压力。`judgehost` 安装指令只需按照以下指令类比修改。其中，`DOMSERVER_BASEURL` 为 `DOMjudge` 首页的地址。

```shell
sudo docker run -it --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro --name judgehost-3 --hostname judgedaemon-3 -e DAEMON_ID=0 -e JUDGEDAEMON_PASSWORD=[passwd4] -e DOMSERVER_BASEURL=http://192.168.1.251/ -e CONTAINER_TIMEZONE=Asia/Shanghai domjudge/judgehost:7.3.2
```

- 基于实际使用发现，不宜创建过多的 `judgehost` ，因为 `judgehost` 也是通过轮询的方式获取 `submission` 进行判题的，过多的 `judgehost` 会导致前端访问压力过大，容易导致崩溃或带宽耗尽等情况，所以请结合实际情况和服务器性能，合理部署适当数量的 `judgehost` 。根据北京理工的经验，开 11 个左右就能挡住 5k 提交了（虽然是在手动管理测评机池的情况下）。

- 对于 Python 测评容易导致 Judgehost 崩溃的问题，这里给出一种解决方法（该方法也适用于需要添加新的评测语言的时候）： ① `docker exec -it [container_name] bash` 进入 Judgehost 容器内。 ② chroot 进入其中的 chroot 目录。 ③执行 `apt update/install` 进行安装即可。

- 如果 `jusgehost` 大面积挂掉的话请检查网络。

