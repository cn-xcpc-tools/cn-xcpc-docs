# Contest Data Server（ICPC Tools）

## 版本

wlp.CDS-2.1.2100

## 在 Domjudge 中创建 cds 用户

在 Domjudge 中创建一个拥有 `API reader`、`API writer` 与 `Source code reader` 权限的用户并设置其密码。

## 配置 CDS

### cds/usr/servers/cds/config/cdsConfig.xml

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

### cds/usr/servers/cds/users.xml

自行修改该文件中的用户密码（注意不要修改用户名）

## 启动 CDS

前台启动：`cds/bin/server run cds`

后台启动：`cds/bin/server start cds`

结束后台进程：`cds/bin/server stop cds`

Web：`https://127.0.0.1:8443`

## Troubleshooting
