---
layout:     post
title:      Gemini 图标变成大字母后，我是怎么把 Xray 修好的
subtitle:   从字体资源异常到 xray-ui 启动链修复
date:       2026-04-01
author:     Honcy Ye
header-img: img/post-bg-bridge.jpg
catalog: true
tags:
    - 运维
    - Xray
    - 网络
    - 科学上网
---

# 这不是前端问题，而是字体没有加载成功

最近遇到一个很有意思的代理故障：Gemini 页面能打开，正文能看，交互也还在，但图标几乎全坏了。

更诡异的是，它不是显示成空白块，而是直接显示成大号字母，甚至能看到像 `page_info` 这样的字形名字。第一眼看上去很像前端样式出错，或者浏览器缓存抽风，但只要稍微熟悉一下 Gemini 的页面资源组成，就会发现方向其实很明确：

**页面主资源是正常的，真正坏掉的是图标字体。**

Gemini 很多图标来自 Google 的字体资源。当字体文件没有成功加载时，浏览器就会把 ligature 文本直接渲染出来，于是你看到的就不是图标，而是 `page_info`、大号字母和各种错位的字符。

# 一开始怀疑服务端 DNS，没有错，但不够

用户是通过 `v2rayN` 连接我在 `usa.honcyye.top` 上搭建的 Xray 节点访问 Gemini。客户端切到 `Global` 模式之后问题依旧，所以第一反应自然是怀疑服务端 DNS。

这个怀疑并不离谱，因为服务器上确实有一个明显的危险信号：

- 机器没有公网 IPv6
- 系统仍然会解析到 AAAA 记录

这种环境很容易出现一种很烦人的状态：主页面能开，但某些静态资源时好时坏，尤其是依赖多个 CDN 域名的 Google 系页面。

不过，真正往下查以后会发现，只停留在“DNS 可能有问题”这个层面，远远不够解释整个现象。

# 为什么 Gemini 页面能开，图标却会坏

在服务器上直接测试时，下面这些域名都可以解析：

- `gemini.google.com`
- `fonts.googleapis.com`
- `fonts.gstatic.com`
- `www.gstatic.com`

Gemini 页面本体也能访问。这一步非常容易把人带偏，让人误以为服务端整体没问题。

但一旦把请求真正走一遍 Xray 节点，结果就完全不一样了：

- `gemini.google.com` 页面主体可以正常访问
- `fonts.googleapis.com` 和 `fonts.gstatic.com` 在节点路径上异常
- 部分资源会在 TLS 阶段直接断开

这就是问题最关键的地方。

这次故障并不是“Gemini 打不开”，而是 **Gemini 页面主资源正常，但图标字体相关的静态资源在 Xray 节点链路上没有被正确处理**。现代网页本来就是由 HTML、JS、CSS、图片、字体、CDN 等一整套资源装配起来的，只要其中某个环节失败，页面就会表现出一种“半正常、半异常”的诡异状态。

Gemini 图标变成大字母，就是一个非常典型的例子。

# 排查过程中，我是怎么一步步把问题收敛的

## 1. 先排除最基础的 DNS 和直连异常

先看系统 DNS，本机用的是 `systemd-resolved`，上游 DNS 指向 Cloudflare。对 `gemini.google.com`、`fonts.googleapis.com`、`fonts.gstatic.com` 的解析都是正常的，普通 `curl` 直连这些域名也能拿到响应。

这意味着一个结论：

**服务器不是完全不能访问 Google，也不是普通的域名解析失败。**

## 2. 确认这台机器的 IPv6 状态并不健康

继续看网络栈，发现这台服务器没有公网 IPv6，但系统又会拿到 AAAA 记录。对于这类环境，很多程序会优先尝试 IPv6，再视情况回退 IPv4。理论上这听起来没什么，但一旦涉及代理、TLS、静态资源和多个 CDN 域名，链路就会变得很不稳定。

这一步虽然还不能证明它是根因，但它已经足够说明：

**默认保留 IPv6 行为，会放大 Google 静态资源访问中的不确定性。**

## 3. 用 Xray 回环测试验证“是节点路径出了问题”

真正把问题钉死的是回环测试。

我在服务器内部临时拉起一个 Xray 客户端，让请求先进入节点，再从节点出站，等于在服务器上模拟客户端真实访问路径。结果非常干脆：

- 走节点访问 `https://gemini.google.com`，能通
- 走节点访问 `https://fonts.googleapis.com/...`，异常
- 走节点访问 `https://fonts.gstatic.com/...`，异常

这时问题就非常清楚了：

> 不是 Gemini 本体过不去，而是 Gemini 依赖的字体与静态资源域名，在当前 Xray 通用链路上处理异常。

这个结论和用户浏览器里看到的 `page_info` 等文本渲染结果也完全一致。因为只要 Material Symbols 这类字体没加载到，浏览器就只能把 ligature 文本直接显示出来。

# 最后真正修好它的，不是某一个单点，而是一套组合修复

这次问题最终能收敛并稳定修复，不是因为“碰巧改对了一个参数”，而是因为把整条链路上最可疑的几个变量都收敛掉了。

真正有效的方案，一共有五步。

## 第一步：关闭这台机器上无效的 IPv6

既然这台机器根本没有公网 IPv6，那最稳妥的做法就不是让程序自己去试，而是直接在系统层把无效 IPv6 关掉。

新增：

```conf
/etc/sysctl.d/99-disable-ipv6-codex.conf
```

内容如下：

```conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

同时在：

```conf
/etc/gai.conf
```

加入：

```conf
precedence ::ffff:0:0/96  100
```

这一组操作的目的很明确：

- 不再优先尝试不可用的 IPv6
- 把访问链路稳定收敛到 IPv4
- 降低 Google 静态域名访问中的随机失败概率

## 第二步：升级 Xray Core

原环境使用的 Xray Core 版本偏旧，于是直接升级到 `26.3.23`，并同步更新：

- `geoip.dat`
- `geosite.dat`

这一步的价值，不是说升级以后问题一定立刻消失，而是先把整个环境抬到一个更可靠的基线，避免旧版本在出站处理、Reality 兼容性或者数据文件上的问题继续干扰判断。

## 第三步：关闭生产 Reality 入站 sniffing

继续排查时，我还把生产 Reality 入站的 `sniffing` 关闭了。

原因也不复杂。既然已经确认问题不是“节点完全不可用”，而是“少数资源域名在特定链路里行为异常”，那就应该尽量减少链路中的额外变量。sniffing 在很多场景下很有用，但在这种对静态资源链路做精细定位的排障里，它反而可能让结果更不稳定。

所以这一步的目的就是：

**先把不必要的干扰项拿掉。**

## 第四步：不要手改 `config.json`，要改 `xray-ui` 的启动生成链

这是这次修复里最关键的一步，也是最容易被忽略的一步。

因为这套环境不是手写 Xray 配置，而是由 `xray-ui` 动态生成 `config.json`。这意味着如果你只是把当前的 `config.json` 手工改好了，那服务一重启，面板就会重新生成一份，把你的修改全部覆盖掉。

所以真正正确的做法不是改“生成结果”，而是改“生成之后、执行之前”的那一小段链路。

最终的做法是把：

```text
/usr/local/xray-ui/bin/xray-linux-amd64
```

改成一个包装器脚本，让它在每次启动 Xray 时先修补新生成的 `config.json`，然后再执行真正的二进制：

```text
/usr/local/xray-ui/bin/xray-linux-amd64.real
```

这样做的好处是：

- 配置修复能自动生效
- 不怕 `xray-ui` 重写 `config.json`
- 后续重启服务也不会把修复丢掉

这一步，才真正把“临时热修”变成了“可持久化的线上修复”。

## 第五步：给 Gemini 依赖的字体与静态资源域名单独加 `direct` 规则

这一步是最后真正把问题打穿的动作。

在启动包装器里，我单独给下面这些域名加了 `direct` 规则：

- `fonts.googleapis.com`
- `fonts.gstatic.com`
- `ssl.gstatic.com`
- `www.gstatic.com`
- `lh3.googleusercontent.com`

同时还强制设置：

```json
"dns": {
  "servers": ["1.1.1.1", "1.0.0.1"],
  "queryStrategy": "UseIPv4"
}
```

以及：

```json
"domainStrategy": "UseIPv4"
```

这一步的本质，其实就是把 Gemini 所依赖的关键资源域名，从那条会出问题的通用链路里单独剥离出来，并明确要求它们稳定走 IPv4。

换句话说，**最后真正修好 Gemini 图标的，不是“调了一下 DNS”，而是把 Gemini 的字体/CDN 资源做了单独路由和持久化修复。**

# 中间还踩了一个运维里非常真实的坑

思路是对的，但第一次改启动包装器时，我把脚本引用处理写坏了。

结果就是：

- `xray-ui` 面板进程在
- Xray 子进程没成功起来
- 外部看起来像“服务器突然连不上了”

这其实也是线上运维里非常常见的一类问题：

> 修复逻辑本身没错，但承载修复的脚本把启动链改挂了。

后面把包装器恢复并修正之后，服务才重新稳定。这件事也反过来说明，像 `xray-ui` 这种面板托管环境，最危险的往往不是某个 JSON 参数，而是启动链本身。

# 最终结果

在完成下面这组修复之后：

1. 关闭无效 IPv6
2. 提升 IPv4 优先级
3. 升级 Xray Core
4. 关闭生产 Reality 入站 sniffing
5. 在 `xray-ui` 启动链中持久注入 Gemini 字体/CDN 域名的 `direct` 规则
6. 重启 `xray-ui` 与 Xray 子进程

用户重新连接 `v2rayN`，再强制刷新 Gemini 页面，图标恢复正常显示。

到这里，问题才算真正闭环。

# 这次排障最值得记住的几点

- 页面能打开，不代表静态资源也正常
- 图标变成大字母，优先怀疑字体文件加载失败
- Google 系页面异常时，优先检查 `fonts.googleapis.com`、`fonts.gstatic.com`、`www.gstatic.com`、`lh3.googleusercontent.com`
- `xray-ui` 这类面板会重写配置，真正可靠的修复必须落在配置生成链，而不是临时生成文件
- 没有公网 IPv6 的服务器，保留默认 AAAA 行为，迟早会制造这类边缘问题

如果要把这次故障压缩成一句话，那就是：

> Gemini 不是打不开，而是字体资源经过 Xray 节点时坏了；真正解决它的办法，是把关键资源域名从问题链路里拆出来，并把修复写进 `xray-ui` 的启动生成链。
