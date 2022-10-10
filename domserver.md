# DOMjudge 中心服务

## 前言

### 版本

DOMjudge 8.1.3

### 环境

- **操作系统**

  一般选择 Ubuntu 18.04 LTS / 20.04 LTS / 21.04。
  
  不同的系统可能对应不同的 PHP 版本（但是要求 7.1 及以上，以上三者分别对应 7.2、7.3、7.4），需要在下方脚本中注意。

- **运行模式**

  一般而言，我们可以让 DOMserver 运行在 apache2 + mod_php 或者 nginx + php-fpm 上。

  如果你对 Linux 系统的运维不太熟悉的话，推荐使用全新安装且干净的操作系统，并使用 apache2 + mod_php。

  另外如果你已经安装了 mysql-server 而不是 mariadb-server，那么下方部分指令可能会出现区别，需要一定的经验。

## 准备工作

### 安装依赖包和功能

更新系统组件。

```shell
sudo apt-get upgrade
sudo apt-get update
```

如果你使用 apache2 + mod_php，请按以下安装。

```shell
sudo apt install acl zip unzip mariadb-server apache2 \
        php php-fpm php-gd php-cli php-intl php-mbstring php-mysql \
        php-curl php-json php-xml php-zip composer ntp
```

如果你使用 nginx + php-fpm，请 [TODO]。

为防止后续 `configure` 步骤出错，接下来安装 `judgehost` 所需的部分依赖

```shell
sudo apt install make gcc g++ unzip debootstrap libcgroup-dev lsof \
        procps libcurl4-gnutls-dev libjsoncpp-dev libmagic-dev 
```


### 编译 DOMserver

```shell
wget https://www.domjudge.org/releases/domjudge-8.1.3.tar.gz
tar -zxvf domjudge-8.1.3.tar.gz
cd domjudge-8.1.3
./configure
make domserver
sudo make install-domserver
```

### 配置数据库

```shell
cd /opt/domjudge/domserver
sudo bin/dj_setup_database -u root install
```

### 配置 MySQL

编辑 `/etc/mysql/conf.d/mysql.cnf`，追加以下内容：

```cnf
[mysqld]
max_connections = 1000
max_allowed_packet = 512MB
innodb_log_file_size = 512MB
```

其中 `max_allowed_packet` 数值改成两倍于题目测试数据文件的大小，`innodb_log_file_size` 数值改成十倍于题目测试数据文件的大小。 

若使用的是 mariadb，则 `/etc/mysql/mariadb.conf.d/50-server.cnf` 中 `max_allowed_packet` 一项也需要修改。

```shell
sudo systemctl restart mysql
```

### 配置 Web 服务器

#### apache2 + mod_php

执行以下命令。

```shell
sudo ln -s /opt/domjudge/domserver/etc/apache.conf /etc/apache2/conf-available/domjudge.conf
sudo ln -s /opt/domjudge/domserver/etc/domjudge-fpm.conf /etc/php/7.4/fpm/pool.d/domjudge.conf
sudo a2enmod proxy_fcgi setenvif rewrite
sudo a2enconf php7.4-fpm domjudge
sudo systemctl reload apache2 php7.4-fpm
```

注意根据实际情况替换此处php的版本

~~确认 `/opt/domjudge/domserver/webapp/var/` 属 `www-data` 用户所有。
稍早版本的 DOMJudge 安装时没有设置对应权限，见[相关 issue](https://github.com/DOMjudge/domjudge/issues/650)。~~

~~如果权限不对，请执行下面的命令：~~

```shell
sudo chown www-data:www-data -R /opt/domjudge/domserver/webapp/var/*
```

编辑 `/opt/domjudge/domserver/etc/apache.conf`，取消以下几行内容前的注释（若没有，请在文件尾加上）：

```conf
<IfModule mod_php7.c>
php_value max_file_uploads      110
php_value upload_max_filesize   512M
php_value post_max_size         512M
php_value memory_limit          1024M
</IfModule>
```

编辑 `/etc/php/7.4/apache2/php.ini`，搜索 `date.timezone` 关键字，取消其行前注释，并将其值设为 `Asia/Shanghai`。搜索 `max_execution_time` 关键字，将其值由30改为300，防止生成队伍密码时 PHP 执行超时。

编辑 `/etc/apache2/apache2.conf`，搜索 `KeepAlive` 关键字，将其值设为 `Off`，并在其后新增一行内容：

```conf
MaxClients 1000
```

重启 apache 服务。

```shell
sudo systemctl restart apache2
```

#### nginx + php-fpm

[TODO]

## 配置 domserver

访问 `http://127.0.0.1/domjudge`，使用用户名 `admin` 与 `/opt/domjudge/domserver/etc/initial_admin_password.secret` 内生成的密码登录。

### 修改 admin 密码

不必须，你记得住就行。权限管理应该由用户自行控制。

访问 users 页面，点 `admin` 用户边上的铅笔图标，在 Password 栏中输入新密码，保存。

### 比赛功能性参数设置

访问 home 页面，点 Configuration settings 进行设置。

技术组同学需要注意：

- **Verification required**：是否需要手动验证后给选手返回提交结果。请注意比赛中途切换容易导致丢失 event-feed，会影响 icpctools 工具等的使用。

- **Compile penalty**：编译错误是否计入罚时。

- **Score in seconds**：如果为 no，那么在大量同罚时提交时有可能会出现 FB 不在榜首的情况，如果为 yes，那么排行榜会被样式挤得比较难看，请技术组自行抉择。一般不开。

- **Timelimit overshoot**：如果比赛中有交互题，则可能需要调大参数，请具体参考结果。

- **Show pending**：是否在评测没出结果时显示在排行榜上，也影响了封榜后能否看到提交次数。

- **Show flags**：是否显示国旗，需要给 Affiliation 指定国家。

- **Show affiliation logos**：是否显示学校 logo。

- **Show teams submissions**：是否可以看到其他队伍的提交列表（仅提交时间、封榜前的评测结果，无法看到代码）。

- **Show sample output**：是否给选手查看样例测试点的评测结果细节。

- **Show balloons postfreeze**：封榜后气球用户是否还收得到更新。

- **Show relative time**：是否以相对比赛开始的时间差显示。

- **Allow team submission download**：是否允许选手下载之前提交的代码。开启后可能有作弊隐患。

- **Print command**：输入内容后可以开启打印功能。例如 `enscript -a 0-10 -f Courier9 [file] 2>&1`。推荐使用 [printing-persist-for-domjudge](https://github.com/cn-xcpc-tools/printing-persist-for-domjudge) 项目以开启打印功能。

除此之外的设置，可以交由出题组来完成，例如各项限制数值的限制。~~记得不要把 Output limit 设置的非常大，例如 512MB。~~

### 添加学校

访问 home 页面，点 Team Affiliations，点左下角的 add 按钮进行添加。

### 添加队伍

访问 home 页面，点 Teams，点左下角的 add 按钮进行添加。

对于 Category 一栏，ICPC 2018 沈阳站的做法是常规队伍填 Participants，打星队填 Observers，并在 Team Categories 设置里将 Observers 的 Sortorder 填成 0。

### 生成队伍密码

访问 home 页面，点 Manage team passwords，选中 all teams 和 as userdata.tsv download（按需选中 All teams 或 Teams without password，点击生成，下载并保存好 `userdata.tsv` ）。**注意：** 生成过的密码除了 `userdata.tsv` 不能在其他地方再被看到。

### 添加题目

访问 home 页面，点 Problems，点左下角的 add 按钮进行添加。

#### 关于题目压缩包上传

如果你的题目是以 kattis 格式压缩包上传，并且其中有可以评测的 std，请提前创建比赛，并给当前用户创建一个队伍，然后上传题包后会自动加入所有 std。

#### 关于 Special Judge

访问 home 页面，点 Executables，在其中可以添加新的执行脚本。执行脚本必须包含一个 `build` 文件，可以包含或使用 `build` 生成一个 `run` 文件。执行脚本时，domjudge 会依次运行 `build` 和 `run`，若 `run` 的 exitcode 为 42，domjudge 会返回结果 AC，若为 43，domjudge 会返回 WA。

#### 关于 Polygon

如果你使用 Codeforces Polygon 系统出题，则可以考虑 [testlib-for-domjudge](https://github.com/cn-xcpc-tools/testlib-for-domjudge) 中的 `p2d` 工具来进行题包的转换。

### 添加比赛

访问 home 页面，点 Contests，点左下角的 add 按钮进行添加。

## Troubleshooting

- 关于本地上传文件生成队伍及用户的中文编码的问题

  如果你需要使用文件导入的方式来进行队伍以及用户的生成，请注意 `DOMjudge` 只支持 **UTF-8** 文件编码的文件（无需担心本地转换完后中文乱码的问题，上传完就能够显示正常）。

  关于转码问题，可以使用 Windows自带的文本编辑器，在另存为时选择 `UTF-8` 的文件编码保存，再将保存好的文件上传至 domserver（在Jury界面的 `Import / export` 中）即可。

  关于文件名及格式问题，请参考ICPC官方wiki：[teams.tsv](https://clics.ecs.baylor.edu/index.php?title=Contest_Control_System_Requirements#teams.tsv)、[accounts.tsv](https://clics.ecs.baylor.edu/index.php?title=Contest_Control_System_Requirements#accounts.tsv)。

- 上传到最后几道题目时频繁超时

  前述 MySQL 配置主要针对上传过大测试数据时的 5xx 报错问题，如果上传最后几道题目出现超时，两个可选操作（推荐操作2）：

  1. 将 innodb_log_file_size 修改为至少两倍于所有题目测试数据文件的大小，重启 mysql

  2. 关闭 mysql，将 innodb_log_files_in_group 修改为 3，开启 mysql

- mysqldump 失败，提示 Got packet bigger than 'max_allowed_packet' when dumping 'XXX' at row xxx

  修改 `/etc/mysql/conf.d/mysqldump.cnf` 里的 `max_allowed_packet` 的值到合适大小（最好和之前给服务器设置的一样大）后即可正常进行 `mysqldump`。
