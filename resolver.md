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

或者也可以直接打开awards.bat，会打开一个带GUI的程序，可以选择event-feed文件，也可以在GUI中添加奖项，然后点击save，保存为event-feed.json文件即可。

### 手工修改生成后的 event-feed

* 删除不在榜单上出现的 team （尤其是带有容易引发 `resolver` 报错的空字段的）

* 如果想在滚榜时显示打星队伍却不颁发任何奖项，请在上述 award 程序导出的新 json 文件末尾，找到类似 `team update hidden true` 的一行删除即可。

* 如果想给打星队伍办法 First To Solve 奖项但是不颁发金银铜奖，那么先 `award /path/of/event-feed-original --medals {} {} {}`，再删除新文件的 `team update hidden true` 那行，再运行 `award /path/of/event-feed-new --fts true true` 即可。

### 滚榜

```shell
resolver /path/of/event-feed.award --display_name "{team.display_name}（{org.formal_name}）" --fast 0.15
```

其中 `--fast` 参数的值影响滚榜时进行两次动画的间隙时间，加入`--singleStep 999`可以从一开始就单步执行。如果不加会发现点到一个队伍的pending的题目的时候，点了鼠标没有反应，过一会儿会自动显示题目的情况。


参数具体可参考[官方文档](https://tools.icpc.global/docs/Resolver.pdf)。



## 关于加入队伍照片和学校logo

首先需要强调的是，不同版本的resolver可能有一些不同，以下内容对2.4.727版本适用。

学校logo命名为logo.jpg（jpeg可能会显示不出来，不知道为什么），保存在CDP/organizations/{school_id}/logo.jpg。其中school_id为在domjudge上的team affilications的id。

队伍照片命名为photo.jpg，保存在CDP/teams/{team_id}/photo.jpg。team_id就是在domjudge上的队伍id。

然后把event-feed.json放在CDP文件夹。运行`resolver ./CDP`即可，不要再加event-feed.json文件，不然会不显示图片。

## Troubleshooting

* 沈阳站场馆大屏幕的比例为 32:9，仅显示所连接 16:9 机器画面的上半部分

  此时在运行 resolver 时为其设置 `--display 1a` 能使其在屏幕左上 1/4 的区域显示滚榜。还可以是 `1b`、`1c`、`1d`，具体效果请提前调试。

* 无法显示中文字体

  设置 `ICPC_FONT` 环境变量为 `DengXian`（Windows 推荐）或者 `Noto Sans CJK`（Ubuntu 推荐）。环境变量可以通过系统全局参数、修改 `resolver.bat` 或 `resolver.sh` 达到。

* 运行时内存超限崩溃

  队伍数较多时且有照片则可能出现。请考虑修改 `resolver.bat` 或 `resolver.sh` 中的 Java 虚拟机参数，请注意，根据经验，最好把-xmx后面的大小改为16GB以上，2022年ecfinal使用的大小为20GB。
