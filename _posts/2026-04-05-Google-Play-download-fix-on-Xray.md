---
layout:     post
title:      Google Play 商店能打开但下载一直转圈，我是怎么把 Xray 修好的
subtitle:   从 CDN 误伤到 xray-ui 启动时序排查
date:       2026-04-05
author:     Honcy Ye
header-img: img/post-bg-night.jpg
catalog: true
tags:
    - 运维
    - Xray
    - Android
    - 科学上网
---

# 这次的问题，比 Gemini 图标异常更像“分流没命中”

前几天刚处理完一个和 Gemini 有关的 Xray 问题：页面能打开，但图标会显示成大号字母，根因最后定位到字体和静态资源域名没有被正确处理。

没想到同一台服务器上又出现了另一个很像、但又更隐蔽的问题：

- 安卓端通过这台 Xray 服务器访问 Google Play 商店没有问题
- 商店页面可以正常打开
- 详情页也能看
- 但只要点下载，就会一直转圈
- 临时断开代理后，下载反而立刻开始

这个现象其实很有代表性。它说明 **Google Play 的页面访问链路和真正的下载链路不是一回事**。前者能通，不代表后者也一定能通。

# 为什么“能打开商店，但下载不动”通常不是普通网络故障

如果一个代理节点连 Google Play 首页都打不开，那通常是比较基础的连通性问题。但这次不是。

这次最关键的线索其实是最后一条：

> 断开代理后，下载立刻开始。

这说明：

1. 手机本身和 Play 商店 App 没坏
2. Google Play 服务端也没坏
3. 真正出问题的，是“下载流量经过代理”这一段链路

而 Google Play 的下载过程，本来就不会只打一个域名。你看到的商店页面、接口请求、实际下发 APK/资源包的 CDN，往往会分别走不同的 API 和边缘节点。

所以这类现象最应该怀疑的，不是“Google Play 整体不通”，而是：

**某些负责下载的 API/CDN 域名，在当前代理分流规则里被误伤了。**

# 一开始的判断并没有错：大概率就是 CDN 或下载域名没被正确放行

由于前面 Gemini 的问题已经验证过一次“Google 生态里页面和静态资源是两套链路”，这次 Google Play 一上来就很像同类问题。

于是先看当前服务器上的 Xray 配置，发现一个非常关键的事实：

- 之前为了修 Gemini，只对白名单里的字体域名做了 `direct`
- Google Play 相关域名基本没有被单独处理
- 后面仍然保留着 `geosite:cn`、`geosite:geolocation-cn` 和 `geoip:cn` 的黑洞规则

这意味着只要 Google Play 的某个下载域名、某个回源 IP、某个 CDN 边缘节点在这套规则里落到了不合适的位置，就很容易出现这样一种状态：

- 商店页面能打开
- 下载相关流量命中了错误的路由
- 下载一直卡住不动

这和“页面可见但资源拿不到”的问题，本质上是同一种架构问题。

# 第一轮修复为什么没有生效

这次排障里最容易误导人的地方，其实不是分流思路，而是 `xray-ui` 的启动链。

因为这台服务器上的 Xray 不是手写配置文件直接跑，而是由 `xray-ui` 动态生成 `config.json`。这意味着任何手动改出来的配置，只要服务重启，就可能被面板重新生成覆盖掉。

所以和上次处理 Gemini 一样，这次的修复仍然放在启动 wrapper 里：让 `xray-ui` 每次拉起 Xray 时，先补丁 `config.json`，再执行真正的二进制。

第一轮我做的事情本身并不离谱：

- 加入 Play 相关 API/CDN 域名
- 重启 `xray-ui`
- 看服务是 `active`

表面上看一切正常，但用户测试后发现问题依旧。

真正的问题是：

> wrapper 文件虽然已经更新，但补丁脚本其实没有成功执行。

后来用 `sh -x` 把执行过程打印出来，才发现一个很隐蔽的报错：

```text
TypeError: 'NoneType' object does not support item assignment
```

错误位置正好出现在我给入站补 `sniffing` 的逻辑上。

原因是有些入站的 `sniffing` 字段不是对象，而是 `null`。我最开始用的是 `setdefault("sniffing", {})`，以为可以无脑拿到一个字典继续写入，但当原始值已经存在且是 `null` 时，`setdefault` 返回的还是 `None`，后面再去写：

```python
sniffing["enabled"] = True
```

就直接炸了。

也就是说，那一轮并不是“规则没命中”，而是：

**补丁脚本在半路报错退出，导致 `config.json` 根本没有被改成新版本。**

服务之所以还活着，是因为 wrapper 后面照样 `exec` 了 Xray 真正的二进制，所以表面上看 `xray-ui` 和 `xray` 都正常，但实际上吃到的仍然是旧配置。

这个坑很典型，也很容易骗过排障者。

# 真正生效的修复，最后到底做了什么

这次真正把 Google Play 下载修好的，不是简单补两个域名，而是一组更偏兼容性的调整。

## 1. 扩大 Google Play 下载相关域名的直连范围

最终加入 `direct` 的，不再只是商店前端域名，而是把下载链路里更常见的 API 和 CDN 一起纳进来：

- `play.googleapis.com`
- `play-fe.googleapis.com`
- `android.clients.google.com`
- `android.googleapis.com`
- `dl.google.com`
- `dl.l.google.com`
- `redirector.gvt1.com`
- `domain:gvt1.com`
- `domain:gvt2.com`
- `domain:googleapis.com`
- `domain:googleusercontent.com`
- `domain:ggpht.com`
- `domain:googlevideo.com`

这里的思路很明确：

**不要只放过 Play 商店页面本身，而要把实际承担下载分发的 API/CDN 域名一起放过。**

## 2. 把 `routing.domainStrategy` 改成 `AsIs`

之前的配置是：

```json
"domainStrategy": "IPIfNonMatch"
```

这个策略在某些场景下没问题，但对 Android 这类依赖多域名、多 CDN、甚至可能先走 IP 再靠 SNI/Host 恢复信息的流量来说，不一定是最稳的选择。

最终改成：

```json
"domainStrategy": "AsIs"
```

这样做的目的是尽量保留域名级别的路由判断，减少“先变成 IP 再匹配规则”导致的意外分流。

## 3. 重新打开入站 `sniffing`

前面处理 Gemini 时，为了减少额外变量，我曾把生产 Reality 入站的 `sniffing` 关掉。

那次这么做是合理的，因为当时面对的是字体资源问题，而且问题更像浏览器显式发出的标准 HTTPS 请求。

但这次不一样。Google Play 的安卓端流量比浏览器复杂得多，很多流量如果只看目标 IP，不一定能稳定命中我们写好的域名规则。所以这次最终反而要把 `sniffing` 重新打开，并让它覆盖：

- `http`
- `tls`
- `quic`

只有这样，Xray 才更有机会从 Android 端流量中恢复出足够的目标信息，把请求正确送到我们写好的域名规则上。

而这一步，也正是前面那轮补丁第一次失败的根源所在。

## 4. 去掉会误伤 Google CDN 的 CN 黑洞规则

这次最重要的一步之一，是不再让这台代理中继服务器保留会误伤下载链路的 CN 黑洞规则。

原本配置里有两类阻断：

- `geosite:cn`
- `geosite:geolocation-cn`
- `geoip:cn`

这套思路在某些“本地分流客户端”场景下有它的用途，但对一台专门给移动设备做代理的中继服务器来说，它的副作用也非常明显：

只要 Google 的某个下载域名或者 CDN 边缘节点在匹配上不够稳定，就可能被这些规则提前拦掉。

所以最终做法不是继续赌“哪些域名要不要再补一个”，而是直接把这类容易误伤 CDN 的黑洞规则收掉，只保留：

- 广告类阻断
- `bittorrent` 阻断

这一步把整条下载链路里的误判空间大幅缩小了。

## 5. 验证 `xray-ui` 的启动时序，而不是只看 `systemctl active`

这次还有一个特别值得记住的点：`systemctl is-active xray-ui` 显示 `active`，并不代表补丁已经真正落到 `config.json`。

我最后专门做了一次短轮询，连续看 `xray-ui` 重启后的几秒配置变化，结果非常有意思：

- 第 1 秒还是旧配置
- 第 2 秒开始，才切到补丁后的配置
- 后面保持稳定

这说明 `xray-ui` 的启动时序本身就分成了两段：

1. 先生成原始 `config.json`
2. 再由 wrapper 补丁后执行 Xray

如果只看服务状态，很容易误以为“刚重启完就已经是新配置”，实际上并不一定。

这也是为什么这次排障里，单看服务是否在线远远不够，必须直接读最终落地的 `config.json`。

# 最终结果

在完成下面这组修复之后：

1. 扩大 Google Play 下载相关 API/CDN 域名的 `direct` 范围
2. 将 `routing.domainStrategy` 改为 `AsIs`
3. 重新打开入站 `sniffing`，并覆盖 `http / tls / quic`
4. 去掉会误伤 Google CDN 的 `geosite:cn / geosite:geolocation-cn / geoip:cn` 黑洞规则
5. 确认 `xray-ui` 启动后第二秒开始，补丁配置稳定生效

安卓端重新连接代理后，Google Play 商店下载恢复正常。

用户最终确认：**可以了。**

# 这次问题最值得记住的经验

- Google Play 能打开但下载转圈，优先怀疑下载 CDN/分发链路，而不是商店页面本身
- 断开代理后下载立刻开始，几乎可以直接说明“代理分流规则有问题”
- 在 `xray-ui` 这类动态生成配置的面板里，不能只改 `config.json`，必须盯住生成链
- 服务 `active` 不代表补丁已经真正落到最终生效配置
- `sniffing = null` 这类小细节，足以让整个补丁脚本静悄悄失败
- 对 Android/Play 这种更复杂的客户端，兼容性优先时，域名规则、`sniffing` 和黑洞规则要一起看

如果把这次问题压缩成一句话，那就是：

> Google Play 商店不是打不开，而是下载链路里的 API/CDN 域名被代理规则误伤了；真正修好它，不是补一个域名就结束，而是把分流、sniffing、黑洞规则和 `xray-ui` 启动时序一起理顺。
