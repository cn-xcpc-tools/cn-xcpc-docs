# Contest Data Server

### 介绍

在使用 ICPC Tools 工具链之前，需要配置一个 CDS 实例。具体用途可以参考[官方文档](https://tools.icpc.global/docs/ContestDataServer.pdf)。

### 版本

一般在 [Releases](https://github.com/icpctools/icpctools/releases) 中下载最新版的 `wlp.CDS-2.x.xxx.zip`。可以下载预览版。

### 环境

建议在搭建 DOMserver 的服务器上使用，记得安装 `openjdk-8-jre-headless`。

### 在 DOMjudge 中创建 CDS 用户

在 DOMjudge 中创建一个拥有 `API reader`、`API writer` 与 `Source code reader` 权限的用户并设置其密码。或者可以直接使用 admin 账户。

### 配置 CDS

##### cds/usr/servers/cds/config/cdsConfig.xml

```xml
<cds>
    <!-- Set location= to a directory that the user running the CDS can write to -->
    <contest location="/some/path/to/save/contest/data" recordReactions="false">
        <!-- Set the url to the URL of your DOMjudge installation, followed by /api/contests/<cid>, where <cid> is your CID or external ID -->
        <!-- Set the user and password to the user you created in the previous step -->
        <ccs
            url="https://domjudge/api/contests/{id}"
            user="cds"
            password="password" />
    </contest>
</cds>
```

##### cds/usr/servers/cds/users.xml

自行修改该文件中的用户密码（注意不要修改用户名）

| 用户名       | 解释                                           |
| ------------ | ---------------------------------------------- |
| admin        | 超级管理员用户，可以访问和修改所有数据         |
| presAdmin    | presClient 管理员用户，用于 presAdmin 登录     |
| blue         | 可以访问所有数据，但是不能控制工具链行为       |
| balloon      | 用于气球发放的用户，一般用不到                 |
| public       | 公共用户，无法读取封榜后提交评测结果和选手代码 |
| presentation | 展示客户端用户                                 |
| myicpc       | myicpc 用户，一般用不到                        |
| live         | icpc-live 用户，用于提供直播数据               |
| team1        | 队伍账户，用于 coachView 来看选手屏幕和        |

### 启动 CDS

前台启动：`cds/bin/server run cds`

后台启动：`cds/bin/server start cds`

结束后台进程：`cds/bin/server stop cds`

Web：`https://127.0.0.1:8443`

### Troubleshooting

