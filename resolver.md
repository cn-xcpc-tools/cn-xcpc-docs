# 滚榜工具

在 [Releases](https://github.com/icpctools/icpctools/releases) 中下载 `resolver-2.x.xxx.zip`。

操作系统：Windows 10 / Ubuntu 18.04

运行时：JRE 8

### 导出 event-feed

在 DOMjudge 的 Team category 中将打星队类型设置为隐藏，之后对比赛进行 Finalize 操作，再访问 `http://domjudge/api/contests/{id}/event-feed?stream=false` 下载 event-feed。

如果你的导出数据存在问题，可以尝试使用 [gen_eventfeed.py](https://github.com/cn-xcpc-tools/gen_eventfeed) 直接从数据库读取。

### 设置奖项

使用 `resolver` 包中的 `award` 工具：

```shell
award /path/of/event-feed --medals {gold} {silver} {bronze} --rank 3 --fts true true
```

### 手工修改生成后的 event-feed

* 删除不在榜单上出现的 team （尤其是带有容易引发 `resolver` 报错的空字段的）

* 如果想在滚榜时显示打星队伍却不颁发任何奖项，请在上述 award 程序导出的新 json 文件末尾，找到类似 `team update hidden true` 的一行删除即可。

* 如果想给打星队伍办法 First To Solve 奖项但是不颁发金银铜奖，那么先 `award /path/of/event-feed-original --medals {} {} {}`，再删除新文件的 `team update hidden true` 那行，再运行 `award /path/of/event-feed-new --fts true true` 即可。

### 滚榜

```shell
resolver /path/of/event-feed.award --display_name "{team.display_name}（{org.formal_name}）" --fast 0.15
```

其中 `--fast` 参数的值影响滚榜时进行两次动画的间隙时间。

参数具体可参考[官方文档](https://tools.icpc.global/docs/Resolver.pdf)。

## Troubleshooting

* 沈阳站场馆大屏幕的比例为 32:9，仅显示所连接 16:9 机器画面的上半部分

  此时在运行 resolver 时为其设置 `--display 1a` 能使其在屏幕左上 1/4 的区域显示滚榜。还可以是 `1b`、`1c`、`1d`，具体效果请提前调试。

* 无法显示中文字体

  设置 `ICPC_FONT` 环境变量为 `DengXian`（Windows 推荐）或者 `Noto Sans CJK`（Ubuntu 推荐）。环境变量可以通过系统全局参数、修改 `resolver.bat` 或 `resolver.sh` 达到。

* 运行时内存超限崩溃

  队伍数较多时且有照片则可能出现。请考虑修改 `resolver.bat` 或 `resolver.sh` 中的 Java 虚拟机参数。
