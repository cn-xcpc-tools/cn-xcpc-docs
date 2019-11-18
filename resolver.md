# resolver（ICPC Tools）

## 版本

resolver-2.1.2100

## 导出 event-feed

在 Domjudge 中设置打星队为隐藏，之后对比赛进行 Finalize 操作，再访问 `http://domjudge/api/contests/{id}/event-feed?stream=false` 下载 event-feed。

## 设置奖项

使用 `resolver` 包中的 `award` 工具：

```shell
award /path/of/event-feed --medals {gold} {silver} {bronze} --rank 3 --fts true true
```

## 手工修改生成后的 event-feed

* 删除不在榜单上出现的 team（尤其是带有容易引发 `resolver` 报错的空字段的）

* 设置打星队伍（见 `https://github.com/yang-er/dj_helper/blob/master/Resolver/README.md`）

## 滚榜

```shell
resolver /path/of/event-feed.award --fast 0.15
```

## Troubleshooting
