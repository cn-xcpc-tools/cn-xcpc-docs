# resolver（ICPC Tools）

## 版本

resolver-2.1.2100

## 环境

Windows 10 1903
Domjudge 7.1.1
jre8

## 导出 event-feed

在 Domjudge 中设置打星队为隐藏，之后对比赛进行 Finalize 操作，再访问 `http://domjudge/api/contests/{id}/event-feed?stream=false` 下载 event-feed。

## 设置奖项

使用 `resolver` 包中的 `award` 工具：

```shell
award /path/of/event-feed --medals {gold} {silver} {bronze} --rank 3 --fts true true
```

## 手工修改生成后的 event-feed

* 删除不在榜单上出现的 team （尤其是带有容易引发 `resolver` 报错的空字段的）

* 设置打星队伍（见 `https://github.com/yang-er/dj_helper/blob/master/Resolver/README.md`）

## 滚榜

```shell
resolver /path/of/event-feed.award --fast 0.15
```

其中 `--fast` 参数的值影响滚榜时进行两次动画的间隙时间。

## Troubleshooting

* 沈阳站场馆大屏幕的比例为 32:9，仅显示所连接 16:9 机器画面的上半部分

此时在运行 `resolver` 时为其设置 `--display 2` 能使其在屏幕左上 1/4 的区域显示滚榜。
