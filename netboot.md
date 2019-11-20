# netboot

## 前期准备

[ICPC World Final的ubuntu镜像](http://pc2.ecs.baylor.edu/wfImageBuilds/icpc2020/ImageBuildInstructions.html)，使用该镜像安装虚拟机后，发现其中环境相当完善，但使用iso镜像批量部署不太方便。  

查看该虚拟机时发现，其比赛环境主要通过自建软件源提供软件包的方式进行配置，可以移植到自定义的网络部署中，于是动手进行尝试，大约花了一周多时间，基本完善了该比赛环境的网络部署。  

## 网络部署

这部分主要参考的是Bittersweet学长去年整理的[博客](https://blog.cool2645.com/posts/2018-icpc-shenyang-env-config-log/)。  

假设网络启动的server有多个网卡，同时连接外网（eth0）和内网（eth1），内网静态IP为`192.168.19.251/24`，一共200台选手机。

### DHCP Server

* 安装isc-dhcp-server提供DHCP服务：
  `sudo apt install isc-dhcp-server -y`
* 修改isc-dhcp-server的配置文件：  
  `sudo vim /etc/default/isc-dhcp-server`  
  在最下面两行指定网卡为`INTERFACESv4="eth1"`和`INTERFACESv6="eth1"`  
* 修改DHCP服务的配置文件：  
  `sudo vim /etc/dhcp/dhcpd.conf`  
  在最下面添加：  

  ```conf
  allow booting;
  allow bootp;
  subnet 192.168.19.0 netmask 255.255.255.0 {
      range 192.168.19.1 192.168.19.230;
      option subnet-mask 255.255.255.0;
      option routers 192.168.19.251;
      option broadcast-address 192.168.19.255;
      filename "pxelinux.0";
      next-server 192.168.19.251;
  }
  ```

  * `allow booting;`和`allow bootp;`都是告诉DHCP服务对特定请求做出响应的
  * `range`中分配了`192.168.19.1-230`，是因为随着网络部署的进行，最后几台电脑很难分配到未被使用的IP，所以指定范围大于实际选手机数量
  * `filename "pxelinux.0";`指定启动文件的名称
  * `next-server 192.168.19.251;`指定启动文件在哪台主机上
* 重启服务：  
  `sudo systemctl restart isc-dhcp-server`

### TFTP Server

* 安装tftpd-hpa提供tftp服务：  
  `sudo apt install tftpd-hpa -y`
* 更改目录读写权限：  
  `sudo chmod -R 777 /var/lib/tftpboot/`
* 从国内的镜像源下载ubuntu网络启动文件：  
  `wget http://mirrors.tuna.tsinghua.edu.cn/ubuntu/dists/bionic-updates/main/installer-amd64/current/images/netboot/netboot.tar.gz`  
  解压到目录：  
  `mkdir netboot`  
  `tar -xvf netboot.tar.gz -C netboot`
* 修改`netboot/ubuntu-installer/amd64/boot-screens/txt.cfg`，指定要读取的preseed文件位置：

  ```cfg
  default install
  label install
      menu label ^Install
      menu default
      kernel ubuntu-installer/amd64/linux
      append auto=true priority=critical url=http://192.168.19.251/preseed.conf vga=788 initrd=ubuntu-installer/amd64/initrd.gz --- quiet
  label cli
      menu label ^Command-line install
      kernel ubuntu-installer/amd64/linux
      append tasks=standard pkgsel/language-pack-patterns= pkgsel/install-language-support=false vga=788 initrd=ubuntu-installer/amd64/initrd.gz --- quiet
  ```

* 将网络启动文件复制到 TFTP 根目录  
  `sudo cp -r netboot/* /var/lib/tftpboot/`
* 重启服务：  
  `sudo systemctl restart tftpd-hpa`

### Apache Server

* 安装apache提供基本文件的访问：  
  `sudo apt install apache2 -y`
* 解压ubuntu的安装镜像的iso文件，复制其中的casper文件夹到`/var/www/html`下，在浏览器中测试`http://192.168.19.251/casper/filesystem.squashfs`的可访问性。

### preseed.conf

preseed文件的主要作用就是代替手动安装时语言，键盘布局，时区，帐户等一步一步的选择，ubuntu官方文档中[Automating the installation using preseeding](https://help.ubuntu.com/lts/installation-guide/amd64/apb.html)对其每一部分都有相应的介绍，还在[这里](https://help.ubuntu.com/lts/installation-guide/example-preseed.txt)提供了一个示例。  

我的[preseed文件](https://gist.github.com/whoisnian/1510ce6b274409dd9e391d243d45bdff)是配置完比赛环境后的，里面有一些特殊设置会在下面进行介绍。

### Squid Server

* 安装squid提供缓存代理服务：  
  `sudo apt install squid -y`
* 修改squid配置文件：  
  `sudo cp squid.conf squid.conf.old`  
  `sudo sed -i "/^#/d;/^ *$/d" /etc/squid/squid.conf`  
  `sudo vim squid.conf`  
  在默认的配置文件基础上修改为：  

  ```conf
  acl SSL_ports port 443
  acl Safe_ports port 80		# http
  acl Safe_ports port 21		# ftp
  acl Safe_ports port 443		# https
  acl Safe_ports port 70		# gopher
  acl Safe_ports port 210		# wais
  acl Safe_ports port 1025-65535	# unregistered ports
  acl Safe_ports port 280		# http-mgmt
  acl Safe_ports port 488		# gss-http
  acl Safe_ports port 591		# filemaker
  acl Safe_ports port 777		# multiling http
  acl CONNECT method CONNECT
  http_access deny !Safe_ports
  http_access deny CONNECT !SSL_ports
  http_access allow localhost manager
  http_access deny manager
  http_access allow localhost
  http_access allow all                             # 修改：允许未被前面规则禁止的所有连接
  http_port 8888                                    # 修改：默认端口修改为8888
  coredump_dir /var/cache/squid                     # 修改：coredump 文件夹
  dns_v4_first on                                   # 添加：首选ipv4 dns
  cache_mem 10240 MB                                # 添加：限制缓存占用的内存
  maximum_object_size 512 MB                        # 添加：最大缓存对象
  cache_dir ufs /var/cache/squid 20000 16 256       # 添加：缓存文件存储路径
  refresh_pattern ^ftp:		1440	20%	10080
  refresh_pattern ^gopher:	1440	0%	1440
  refresh_pattern -i (/cgi-bin/|\?) 0	0%	0
  refresh_pattern (Release|Packages(.gz)*)$      0       20%     2880
  refresh_pattern .		0	20%	4320
  ```
  
* 创建相关目录，并调整权限：  
  `sudo systemctl stop squid`  
  `sudo mkdir /var/log/squid`  
  `sudo mkdir /var/cache/squid`  
  `sudo chown proxy:proxy /var/log/squid`  
  `sudo chown proxy:proxy /var/log/squid`  
  `sudo squid -z`
* 启动squid：  
  `sudo systemctl start squid`

### NTP Server

* 安装ntp提供时间同步服务：  
  `sudo apt install ntp -y`

## 开发环境

### 本地源

前期准备中提到，ICPC WF镜像中使用自己的源中软件包部署比赛环境，详细查看后是[https://pc2cancer.ecs.csus.edu/apt/](https://pc2cancer.ecs.csus.edu/apt/)中的`icpc2020`包依赖了所需内容：

```text
Package: icpc2020
Version: 0.1.9
Architecture: all
Pre-Depends: gdm3, xorg, a2ps, bash, compiz-plugins, coreutils, cups, evince, g++, gcc, html2ps, icpc-eclipse, iptables, openjdk-11-jdk, openjdk-11-doc, openssh-server, passwd, patch,  rpcbind, rsyslog, sed, snmpd, tar, tcsh, unzip, vim-doc, vim-gnome, ffmpeg, vlc, wget, dc, emacs, tmux, geany, libstdc++-7-doc, file-roller, git, light-themes, openjdk-11-source, locate, cppreference-doc-en-html, pypy-doc, kate, icpc-pycharm, valgrind, apcalc, indicator-applet-session, indicator-applet-complete, libmagic1, libjsoncpp1, libcurl3-gnutls, codeblocks, icpc-intellij-idea, icpc-kotlinc, icpc-clion, icpc2020-jetbrains, libvte9, compiz-gnome, gnome-session-flashback, make, xterm, icpc-logkeys
```

其中`gnome-session-flashback`是桌面环境，`icpc-eclipse`，`vim-gnome`，`emacs`，`geany`，`kate`，`icpc-pycharm`，`codeblocks`，`icpc-intellij-idea`，`icpc-clion`是编辑器和IDE，`openjdk-11-doc`，`cppreference-doc-en-html`，`pypy-doc`提供了一些编程语言的文档。  

所以在preseed.conf文件中加入此源，安装`icpc2020`包即可。对应我preseed文件中的

```shell
d-i apt-setup/local0/repository string http://192.168.19.251/apt bionic main
d-i apt-setup/local0/key string http://192.168.19.251/pc2.key
```

因为该源只能https访问，squid无法直接缓存，且下载速度较慢，所以在本地建立镜像更加合适。  

我在虚拟机中使用`apt-mirror`将该源同步到了本地，利用`apt-key`工具导出了对应的gpg key，相关文件都放到了网络部署服务器的`/var/www/html`目录下由apache提供访问。  

### vscode

安装完`icpc2020`包后`/etc/apt/sources.list.d/`目录下会多出几个额外的源，其中`microsoft.list`中包含[https://pc2cancer.ecs.csus.edu/vscode/](https://pc2cancer.ecs.csus.edu/vscode/)，想在比赛环境中额外安装vscode的话`sudo apt install code icpc-code-extensions -y`即可。  

但同样的问题，无法缓存且下载速度慢，使用`apt-mirror`同步时发现该源内旧包过多，完整同步需要几十G，于是手动下载ubuntu mirror所需的文件，然后每个包只下载最新版本，也搭建了本地源。  

安装完毕后发现vscode无插件，`apt-file`查看`icpc-code-extensions`包内容发现插件安装到了`/var/code/extensions`处，`/etc/skel/`给出了选手机home目录所需的一些文件，其中/etc/skel/.vscode中设置了指向`/var/code/extensions`的软链接，于是`rsync -lr /etc/skel/. /home/syclient`。  
再次运行vscode，发现存在插件权限问题，于是不再采用软链接的方式，删除了`/home/syclient/.vscode`中的软链接，直接`cp -r /var/code/extensions /home/syclient/.vscode`，最后调整了两个文件`/home/syclient/.vscode/extensions/ms-vscode.cpptools-0.25.1/bin/Microsoft.VSCode.CPP.Extension.linux`和`/home/syclient/.vscode/extensions/ms-vscode.cpptools-0.25.1/bin/Microsoft.VSCode.CPP.IntelliSense.Msvc.linux`的权限为744，vscode插件正常。  

但vscode插件默认全部开启，键位会比较奇特，一些插件也并不需要，于是尝试批量进行删除：  

```shell
#!/bin/bash
su syclient -c 'code --uninstall-extension dbaeumer.vscode-eslint'
su syclient -c 'code --uninstall-extension hiro-sun.vscode-emacs'
su syclient -c 'code --uninstall-extension ms-vscode.csharp'
su syclient -c 'code --uninstall-extension ms-vscode.sublime-keybindings'
su syclient -c 'code --uninstall-extension ms-vscode.vscode-typescript-tslint-plugin'
su syclient -c 'code --uninstall-extension vscodevm.vim'
```

## 其它

### gdm设置自动登录

* `sed -i "s/\[daemon\]/\[daemon\]\nAutomaticLoginEnable=true\nAutomaticLogin=syclient/" /etc/gdm3/custom.conf`

### 备份与恢复home

* `mkdir /var/home_back`  
  `rsync -lr /home/syclient/. /var/home_back`
* `rm -rf /home/syclient`  
  `mkdir /home/syclient`  
  `rsync -lr /var/home_back/. /home/syclient`  
  `chown syclient:syclient -R /home/syclient`

### nmap检查选手机网络连通性

* `nmap -v -sn 192.168.19.1-200`

### parallel-ssh并行执行命令

* `parallel-ssh -h host.list -x '-o StrictHostKeyChecking=no' -o txt -t 0 uptime`  
  其中host.list为主机列表，我使用`create_ip.sh`脚本生成：  

  ```shell
  #!/bin/bash
  for ((index = $1;index <= $2;index++))
  do
      echo "root@192.168.19.$index"
  done
  ```

  使用方法：`./create_ip.sh 1 100 > host.list`

### 其它主机连接外网

* `export http_proxy=http://192.168.19.251:8888`  
  `export https_proxy=http://192.168.19.251:8888`  
  若要使用`sudo`则可加上`-E`选项保留环境变量，如`sudo -E apt install nmap`

### squid无法缓存https流量

* `/etc/apt/source.list`中使用http源，如`http://mirrors.tuna.tsinghua.edu.cn`
