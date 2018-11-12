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

```shell
sudo apt install libcurl4-gnutls-dev libjsoncpp-dev libmagic-dev
```

```shell
sudo phpenmod json
```

### 编译 Domjudge

```shell

wget https://www.domjudge.org/releases/domjudge-6.0.2.tar.gz
```

```shell
tar -zxvf domjudge-6.0.2.tar.gz
```