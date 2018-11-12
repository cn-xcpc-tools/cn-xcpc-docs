# Domjudge 简易搭建文档（domserver 部分）

## 版本

Domjudge 6.0.2

## 环境

Ubuntu 18.04，全新安装的系统

## 准备工作

### 安装依赖包和功能

```shell
sudo apt-get upgrade && sudo apt-get update
```

```shell
sudo apt install gcc g++ make zip unzip mariadb-server \
        apache2 php php-cli libapache2-mod-php php-zip \
        php-gd php-curl php-mysql php-json php-xml php-mbstring \
        acl bsdmainutils ntp phpmyadmin python-pygments \
        libcgroup-dev linuxdoc-tools linuxdoc-tools-text \
        groff texlive-latex-recommended texlive-latex-extra \
        texlive-fonts-recommended texlive-lang-european
```

安装时选择 apache2

```shell
sudo apt install libcurl4-gnutls-dev libjsoncpp-dev libmagic-dev
```

```shell
sudo phpenmod json
```

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
make domserver && sudo make install-domserver
make judgehost && sudo make install-judgehost
make docs && sudo make install-docs
```

### 配置数据库

```shell
cd ~/domjudge/domserver
sudo bin/dj_setup_database -u root install
```

### 配置 Web 服务器

```shell
cd ~/domjudge/domserver
sudo ln -s /home/username/domjudge/domserver/etc/apache.conf /etc/apache2/conf-available/domjudge.conf
sudo a2enmod rewrite
sudo a2enconf domjudge
sudo systemctl reload apache2
```

注意，ln 命令中的 username 需要替换成你的实际用户名。

现在你应该可以访问 http://127.0.0.1/domjudge 并使用用户名 admin 密码 admin 登录 domjudge 后台了。