# presentation 工具链（ICPC Tools）

## 版本

presentations-2.1.2100
presentationAdmin-2.1.2100

## 环境

（presAdmin机）Ubuntu 18.04.3 LTS
（client机）Windows 10 1903
jre8
wlp.CDS-2.1.2100

## presentationAdmin（架设到方便操作的机器上）

`presAdmin https://cds presAdmin {password}`

## presentations（架设到连接大屏幕的机器上，并接入 CDS 所在网络）

`client {presentation-id，自选，多开时不重复即可} https://cds presentation {password}`

## 使用

在 `presAdmin` 中选中需要控制的 presentation，并在右侧选择需要令其显示的内容，然后点击 `Apply` 按钮。

## Troubleshooting

* `presAdmin` 右侧不显示可选的内容

排查 CDS 是否正常运行、CDS 配置是否正确、`presAdmin` 是否与 CDS 在网络上能够通信。

* `client` 启动后，`presAdmin` 中未显示该 presentation

排查 `client` 是否与 CDS 在网络上能够通信。

* 沈阳站场馆大屏幕的比例为 32:9，仅显示所连接 16:9 机器画面的上半部分

此时在 `presAdmin` 中菜单栏内设置 presentation 的显示区域为 `(0,0,1920,540)` 即能使其完全在大屏幕上显示出来。
