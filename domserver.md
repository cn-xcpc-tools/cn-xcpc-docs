# Domjudge server

## 版本

Domjudge 7.1.1

## 环境

Ubuntu 18.04.3 LTS，全新安装的系统

## 准备工作

### 安装依赖包和功能

```shell
sudo apt-get upgrade && sudo apt-get update
```

```shell
sudo apt install gcc g++ make zip unzip mariadb-server \
        apache2 php php-cli libapache2-mod-php php-zip \
        php-gd php-curl php-mysql php-json php-xml php-intl php-mbstring \
        acl bsdmainutils ntp phpmyadmin python-pygments \
        libcgroup-dev linuxdoc-tools linuxdoc-tools-text \
        groff texlive-latex-recommended texlive-latex-extra \
        texlive-fonts-recommended texlive-lang-european composer
```

安装时选择 `apache2`

```shell
sudo apt install libcurl4-gnutls-dev libjsoncpp-dev libmagic-dev
```

```shell
sudo phpenmod json
```

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
make domserver && sudo make install-domserver
make docs && sudo make install-docs
```

### 配置数据库

```shell
cd /opt/domjudge/domserver
sudo bin/dj_setup_database -u root install
```

### 配置 Web 服务器

```shell
cd /opt/domjudge/domserver
sudo ln -s /opt/domjudge/domserver/etc/apache.conf /etc/apache2/conf-available/domjudge.conf
sudo a2enmod rewrite
sudo a2enconf domjudge
sudo systemctl reload apache2
sudo chown www-data:www-data -R /opt/domjudge/domserver/webapp/var/*
```

现在你应该可以访问 `http://127.0.0.1/domjudge` 并使用用户名 `admin` 与 `/opt/domjudge/domserver/etc/initial_admin_password.secret` 内生成的密码登录 domjudge 后台了。

### 配置 MySQL

编辑 `/etc/mysql/conf.d/mysql.cnf`，追加以下内容：

```cnf
[mysqld]
max_connections = 1000
max_allowed_packet = 16MB
innodb_log_file_size = 48MB
```

其中 `max_allowed_packet` 数值改成两倍于题目测试数据文件的大小，`innodb_log_file_size` 数值改成十倍于题目测试数据文件的大小。  
若使用的是 mariadb，则 `/etc/mysql/mariadb.conf.d/50-server.cnf` 中 `max_allowed_packet` 一项也需要修改。

```shell
sudo systemctl restart mysql
```

### 配置 PHP

编辑 `/opt/domjudge/domserver/etc/apache.conf`，取消以下几行内容前的注释：

```conf
<IfModule mod_php7.c>
php_value max_file_uploads      100
php_value upload_max_filesize   128M
php_value post_max_size         128M
php_value memory_limit          512M
</IfModule>
```

编辑 `/etc/php/7.2/apache2/php.ini`，搜索 `date.timezone` 关键字，取消其行前注释，并将其值设为 `Asia/Shanghai`。搜索 `max_execution_time` 关键字，将其值由30改为300，防止生成队伍密码时 PHP 执行超时。

```shell
sudo systemctl restart apache2
```

### 配置 Apache

编辑 `/etc/apache2/apache2.conf`，搜索 `KeepAlive` 关键字，将其值设为 `Off`，并在其后新增一行内容：

```conf
MaxClients 1000
```

```shell
sudo systemctl restart apache2
```

## 配置 domserver

访问 `http://127.0.0.1/domjudge`，使用用户名 `admin` 与 `/opt/domjudge/domserver/etc/initial_admin_password.secret` 内生成的密码登录。

### 修改 admin 密码

访问 users 页面，点 `admin` 用户边上的铅笔图标，在 Password 栏中输入新密码，保存。

### 设置 domserver

访问 home 页面，点 Configuration settings 进行设置。技术组配置时主要注意修改 Score in seconds（如果为 no，那么在大量同罚时提交时有可能会出现 FB 不在榜首的情况，如果为 yes，那么排行榜会被样式挤得比较难看，请技术组自行抉择）、Show flags（显示国旗）、Show pending（是否在评测没出结果时显示在排行榜上，也影响了封榜后能否看到提交次数）、Show balloons postfreeze（封榜后气球用户是否还收得到更新）、Compile penalty（编译错误是否计入罚时）、Enable printing（是否开启 domjudge 自带的打印功能）、Show relative time（是否以剩余时间显示比赛情况）等。

除此之外的设置，可以交由出题组来完成，例如各项数值的限制。

### 添加学校

访问 home 页面，点 Team Affiliations，点左下角的 add 按钮进行添加。

### 添加队伍

访问 home 页面，点 Teams，点左下角的 add 按钮进行添加。

对于 Category 一栏，ICPC 2018 沈阳站的做法是常规队伍填 Participants，打星队填 Observers，并在 Team Categories 设置里将 Observers 的 Sortorder 填成 0。

### 生成队伍密码

访问 home 页面，点 Manage team passwords，选中 all teams 和 as userdata.tsv download。注意生成过的密码除了 userdata.tsv 不能在其他地方再被看到。

### 添加题目

访问 home 页面，点 Problems，点左下角的 add 按钮进行添加。

#### 关于 Special Judge

访问 home 页面，点 Executables，在其中可以添加新的执行脚本。执行脚本必须包含一个 `build` 文件，可以包含或使用 `build` 生成一个 `run` 文件。执行脚本时，domjudge 会依次运行 `build` 和 `run`，若 `run` 的 exitcode 为 42，domjudge 会返回结果 AC，若为 43，domjudge 会返回 WA。

### 添加比赛

访问 home 页面，点 Contests，点左下角的 add 按钮进行添加。

## Troubleshooting
