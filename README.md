# 📱 FlClash 手机端配置 · 去广告优先

> 一个文件搞定：**广告拦截为主，分流为辅**。适用于 FlClash（Android / iOS）以及其他 Mihomo / Clash Meta 内核客户端。

---

## 🎯 这个配置的定位

| 优先级 | 目标 | 做法 |
|---|---|---|
| 1 | **去广告** | anti-AD + AWAvenue + geosite 广告分类，三层叠加，全部 REJECT |
| 2 | 不误杀 | 关键 App 白名单前置，广告拦截可一键旁路 |
| 3 | 分流 | 只保留常用组（AI / 流媒体 / Google / Telegram / 微软 / TikTok / 地区） |
| 4 | 省电省内存 | 规则集全部用 `.mrs` 二进制格式，懒测速，关闭进程匹配 |

**适合**：手机日常刷 App、主要想清掉开屏 / 信息流 / Banner 广告，顺带科学上网。

**不适合**：需要极限分流精细化（几百个策略组、按进程分流）、需要强力反审查伪装的人 —— 那类场景请用更重的脚本方案。

---

## 📁 文件说明

```text
.
├── config-rule.yaml   # 主配置，导入 FlClash 直接用（只需改 1 行订阅）
├── my-ad.yaml         # 自定义补充拦截清单（可选，默认未启用）
└── README.md          # 本文档
```

---

## 🚀 快速上手（4 步）

### 1. 改订阅

打开 `config-rule.yaml`，找到最上面的：

```yaml
proxy-providers:
  Airport1:
    url: "https://你的订阅链接1"     # ← 换成你的机场订阅（Clash / Mihomo 格式）
```

> 只接受 **Clash 格式** 订阅。如果机场只给 SS / V2Ray 链接，先用 subconverter 转换。
> 有第二个订阅时，把文件里注释掉的 `Airport2` 整段取消注释即可。

### 2. 导入 FlClash

**方式 A（推荐）：导入文件**

- 手机把 `config-rule.yaml` 下载到本地；
- FlClash → **配置** → 右下角 **+** → **导入文件** → 选择该文件；
- 导入后点一下这张配置卡，使其**激活**。

**方式 B：用链接导入**

```text
https://fastly.jsdelivr.net/gh/lhflowersbloom/Flclash-rule@main/config-rule.yaml
```

好处是以后仓库更新能同步到手机；代价是订阅链接要在 FlClash 里重新编辑一次，且每次同步后可能被覆盖。**只在自己会改配置的情况下用这个方式。**

### 3. 更新规则集（首次必须做）

- 配置页 → 该配置卡 **⋮** → **更新规则集**（等待 20~60 秒）；
- 再点 **更新订阅**，让节点列表加载出来。

首次成功后会缓存在本地，之后按 24 小时自动更新。

### 4. 开代理

- 主界面 → 模式选 **规则**；
- 打开开关，允许 VPN 连接（TUN 已默认开启）；
- 进入 **代理** 页，确认 `🚀 节点选择` 里有节点、`⚡ 自动选择` 不是空的。

---

## 🛡️ 广告拦截怎么控制

所有广告规则都指向同一个策略组：**`🛡️ 广告拦截`**，默认值是 `REJECT`。

| 你想做的事 | 操作 |
|---|---|
| 正常用（默认） | 什么都不用做，保持 `REJECT` |
| 某个 App 登录 / 支付异常，怀疑误杀 | 临时把 `🛡️ 广告拦截` 切成 `DIRECT`，重试一次 |
| 切 DIRECT 后恢复正常 | 说明是广告规则误杀 → 按下面「误杀排查」精准加白 |
| 想让某类广告走代理而不是拦截 | 切成 `🚀 节点选择`（一般不需要） |

> ⚠️ 不建议长期把广告拦截设成 `DIRECT`，那等于关掉了这个配置的主要功能。

### 拦截来源（三层叠加）

| 规则集 | 来源 | 覆盖重点 | 规则数 |
|---|---|---|---|
| `ads-anti` | [anti-AD](https://github.com/privacy-protection-tools/anti-AD)（DustinWin 的 mrs 镜像） | **中文站点 / 国内 App 广告**（主力） | ~10.6 万 |
| `ads-awa` | [AWAvenue-Ads-Rule](https://github.com/TG-Twilight/AWAvenue-Ads-Rule) | App 开屏、信息流、广告 SDK 补充 | ~970 |
| `ads-meta` | [MetaCubeX geosite](https://github.com/MetaCubeX/meta-rules-dat) `category-ads-all` | 国际广告 / 追踪域名兜底 | ~900 |

实测命中示例（mihomo 日志）：

```text
--> mi.gdt.qq.com:443        match RuleSet(ads-anti) using 🛡️ 广告拦截[REJECT]   # 广点通
--> pagead2.googlesyndication.com match RuleSet(ads-anti) using 🛡️ 广告拦截[REJECT]
--> res.wx.qq.com:443        match DomainSuffix(res.wx.qq.com) using DIRECT     # 微信资源未被误杀
```

---

## 🧩 分流逻辑

规则**从上到下**匹配，顺序即优先级：

```text
0. 兼容白名单（微信 / 支付宝 / 12306 / 风控验证码…）
1. 私有地址 & 局域网
2. 广告拦截（三层）
3. 服务分流：AI / Google / YouTube·Netflix·Spotify / Telegram / TikTok / 微软 / OneDrive / GitHub / Apple
4. 地域兜底：非中国域名 → 代理；中国域名 / IP → 直连
5. 剩余流量 → 🐟 漏网之鱼
```

**策略组一览**

| 组 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `🚀 节点选择` | 手动 | `⚡ 自动选择` | 全局出口 |
| `⚡ 自动选择` | 自动测速 | — | 全部节点里挑最快，懒测速省电 |
| `🛡️ 广告拦截` | 手动 | `REJECT` | 拦截总开关 |
| `🤖 AI 服务` | 手动 | 美国 | ChatGPT / Claude / Gemini 等 |
| `📺 国际流媒体` | 手动 | 香港 | YouTube / Netflix / Spotify |
| `🍀 Google` | 手动 | 香港 | 搜索、Play 服务、登录 |
| `📲 Telegram` | 手动 | 新加坡 | — |
| `🎵 TikTok` | 手动 | 美国 | 需要 US/SG 节点 |
| `Ⓜ️ 微软服务` | 手动 | 直连 | 微软 / OneDrive / Xbox |
| `🐟 漏网之鱼` | 手动 | 节点选择 | 未命中规则的流量 |
| `🇭🇰 🇯🇵 🇸🇬 🇺🇸` 节点池 | 手动 | 自动归类 | 按节点名关键词自动分组 |

> 地区分组靠**节点名称关键词**识别（香港/HK、日本/JP、新加坡/SG、美国/US…）。
> 如果你的机场节点名不含这些关键词，节点会进不了对应分组 —— 改成自己的关键词即可，见下方「自定义」。

---

## 🔧 自定义

### 加自己的分流规则

在 `config-rule.yaml` 的 `rules:` 段里加，**越靠上优先级越高**：

```yaml
  - DOMAIN-SUFFIX,example.com,🚀 节点选择   # 该域名走代理
  - DOMAIN-SUFFIX,example.cn,DIRECT         # 该域名直连
  - IP-CIDR,1.2.3.0/24,DIRECT,no-resolve    # 该网段直连
```

### 加自己的广告域名

1. 域名写进 `my-ad.yaml` 的 `payload:` 下：

```yaml
payload:
  - "ads.example.com"
  - "tracker.example.net"
```

2. 打开 `config-rule.yaml`，在 `rule-providers` 里取消这一行的注释：

```yaml
  ads-custom: {<<: *domain, format: yaml, url: "https://fastly.jsdelivr.net/gh/lhflowersbloom/Flclash-rule@main/my-ad.yaml", path: ./ruleset/ads-custom.yaml}
```

3. 在 `rules:` 的广告段里取消这一行的注释：

```yaml
  - RULE-SET,ads-custom,🛡️ 广告拦截
```

4. 保存 → 更新规则集 → 生效。

### 加白名单（处理误杀）

把域名加到 `rules:` **最顶部**的「兼容白名单」段，比拦截规则靠前就一定会赢：

```yaml
  - DOMAIN-SUFFIX,某个App的域名.com,DIRECT
```

---

## 🕵️ 误杀排查（重要）

手机上广告规则最容易误杀的几类：**登录 / 注册、支付、验证码人机验证、推送、更新下载**。

排查流程：

1. FlClash → **日志** → 把日志级别调到 `info`（或临时改配置 `log-level: info`）；
2. 复现问题（比如点登录）；
3. 在日志里找 `REJECT`：

```text
--> api.xxx.com:443 match RuleSet(ads-anti) using 🛡️ 广告拦截[REJECT]
```

4. 把这个域名加成白名单（`DOMAIN-SUFFIX,xxx.com,DIRECT`），保存并更新；
5. 若日志里没有 REJECT，问题多半在节点 / DNS / App 自身，不要乱删规则。

**已经预置的白名单**（防止常见误杀）：

```text
微信/微信支付：weixin.qq.com、res.wx.qq.com、wechat.com、tenpay.com、wechatpay.com、mmbiz.qpic.cn
支付宝：alipay.com、alipayobjects.com
其他：mybank.cn、12306.cn
风控/人机验证：online-metrix.net、hcaptcha.com、challenges.cloudflare.com
```

---

## ⚡ 性能说明（为什么在手机上更省）

| 设计 | 原因 |
|---|---|
| 规则集全部 `.mrs`（二进制） | 相比 `.yaml` / `.list` 文本规则，加载快、内存占用低很多 |
| `find-process-mode: "off"` | 手机端按进程匹配收益低，还会常驻扫描，关掉更省电 |
| `lazy: true` 懒测速 | 只有真正用到某个组时才测速，避免后台反复打点 |
| `interval: 86400` | 规则集每天更新一次即可，减少启动时的下载等待 |
| 只保留必要策略组 | 组越少，启动越快，切换越省内存 |

> 想进一步减负：把 `ads-anti` 换小一号的源，或在 `rules` 里注释掉 `ads-awa` / `ads-meta`。

---

## ❓ 常见问题

**Q：导入后没有节点 / 节点是空的？**
A：订阅没改成自己的链接，或者订阅不是 Clash 格式。改 `Airport1.url` 后重新「更新订阅」。

**Q：规则集一直更新失败？**
A：配置里全部走 `fastly.jsdelivr.net`（国内一般可直连）。若所在网络访问不了，把 URL 里的 `fastly.jsdelivr.net` 换成 `cdn.jsdelivr.net`；仍不行就开代理后重试。

**Q：广告还在？**
A：先确认 ① `🛡️ 广告拦截` 是 `REJECT`；② 规则集已更新成功；③ 该 App 的广告是不是走 IP 硬编码或自带 DoH（这类 DNS 层拦不住，任何 Clash 配置都一样）。可以看日志确认域名有没有命中。

**Q：拦截太狠，某 App 打不开？**
A：按「误杀排查」走一遍。临时解法是把 `🛡️ 广告拦截` 切 `DIRECT` 验证。

**Q：能按 App 分流吗？**
A：本配置关闭了进程匹配（`find-process-mode: "off"`）以省电。需要时改成 `strict`，再写 `PROCESS-NAME,包名,策略组`。注意 Android 上部分应用无法读取进程名，实际效果因系统而异。

**Q：iPhone 能用吗？**
A：可以，FlClash / Clash Meta 系客户端导入同一份配置；如果客户端不支持 TUN，改用系统 VPN 接管，其余规则不变。

**Q：配置校验过了吗？**
A：是。用 **Mihomo Meta v1.19.14** 实测通过：`mihomo -t` 校验成功，23 个规则集全部下载解析成功（含 10.6 万条广告域名），实际请求命中 `REJECT` / `DIRECT` / 各服务组均符合预期。

---

## 🔄 回滚

升级前把旧的 `config-rule.yaml` 另存一份。出问题就重新导入旧文件、重新激活即可。

---

## 📜 致谢

- 原配置：[yudong820/Flclash-rule](https://github.com/yudong820/Flclash-rule)，参考模板来自 [qichiyuhub](https://github.com/qichiyuhub)
- 广告规则：[anti-AD](https://github.com/privacy-protection-tools/anti-AD)、[AWAvenue-Ads-Rule](https://github.com/TG-Twilight/AWAvenue-Ads-Rule)
- 分流规则：[MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)、[DustinWin/ruleset_geodata](https://github.com/DustinWin/ruleset_geodata)、[wwqgtxx/clash-rules](https://github.com/wwqgtxx/clash-rules)
- 脚本思路参考：[Wuzhenshi8874/flclash-v-series](https://github.com/Wuzhenshi8874/flclash-v-series)

## ⚠️ 免责

本仓库只提供客户端分流配置，不提供任何代理服务或节点。请遵守所在地区的法律法规，仅用于合法的网络访问需求。
