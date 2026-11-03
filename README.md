# FlClash 手机端配置 · 去广告优先

> 一份 YAML，解决四件事：**拦截广告、少误杀、常用服务分流、手机端省电。**
> 适配 FlClash（Android / iOS）以及任意 Mihomo / Clash Meta 内核客户端。

| 内核要求 | 规则集格式 | 校验状态 |
|---|---|---|
| Mihomo ≥ 1.19（FlClash ≥ 0.8.85） | `mrs` 二进制 | Mihomo v1.19.14 实测通过 |

## 目录

1. [定位与取舍](#定位与取舍)
2. [文件说明](#文件说明)
3. [快速上手](#快速上手)
4. [广告拦截怎么控制](#广告拦截怎么控制)
5. [分流逻辑](#分流逻辑)
6. [自定义](#自定义)
7. [误杀排查](#误杀排查)
8. [性能设计](#性能设计)
9. [与 js 覆写脚本的区别](#与-js-覆写脚本的区别)
10. [常见问题](#常见问题)
11. [致谢与免责](#致谢与免责)

---

## 定位与取舍

这套配置围绕手机端的真实使用习惯设计，优先级从高到低：

| 优先级 | 目标 | 实现方式 |
|---|---|---|
| 1 | **去广告** | anti-AD + AWAvenue + geosite 广告分类三层叠加，统一送进 `REJECT` |
| 2 | **不误杀** | 关键 App 白名单前置；广告拦截可一键旁路，无需改文件 |
| 3 | **能分流** | 只保留高频组：AI、流媒体、Google、Telegram、微软、TikTok、地区池 |
| 4 | **省电省内存** | 规则集全部 `mrs` 格式、懒测速、关闭进程匹配 |

**适合**：日常刷 App，主要想清掉开屏、信息流、Banner 广告，顺带科学上网。

**不适合**：需要几百个策略组做极限精细化分流、或需要按进程精确控制每一个 App 的场景 —— 那类需求请选更重的脚本方案。

---

## 文件说明

```text
.
├── config-rule.yaml   # 主配置：导入 FlClash 即可用，只需改 1 行订阅
├── my-ad.yaml         # 自定义补充拦截清单（可选，默认未启用）
└── README.md          # 本文档
```

---

## 快速上手

共 4 步，第 1 步和第 3 步最容易漏。

### 第 1 步 · 改订阅

打开 `config-rule.yaml`，最上面就是唯一需要动的地方：

```yaml
proxy-providers:
  Airport1:
    url: "https://你的订阅链接1"     # ← 换成你的机场订阅（Clash / Mihomo 格式）
```

- 只接受 **Clash 格式** 订阅；机场只给 SS / V2Ray 链接时，先用 subconverter 转换。
- 有第二个订阅时，把文件里注释掉的 `Airport2` 整段取消注释即可。

### 第 2 步 · 导入 FlClash

**方式 A（推荐）：导入文件**

1. 把 `config-rule.yaml` 下载到手机；
2. FlClash → **配置** → 右下角 **+** → **导入文件** → 选择该文件；
3. 点一下这张配置卡，使其**激活**。

**方式 B：用链接导入**

```text
https://fastly.jsdelivr.net/gh/lhflowersbloom/Flclash-rule@main/config-rule.yaml
```

好处是仓库更新能同步到手机；代价是订阅链接需要重新编辑一次，且同步后可能被覆盖。**只在自己会改配置的前提下用这个方式。**

### 第 3 步 · 更新规则集（首次必须做）

1. 配置页 → 该配置卡 **⋮** → **更新规则集**，等待 20~60 秒；
2. 再点 **更新订阅**，让节点列表加载出来。

首次成功后规则集会缓存在本地，之后按 24 小时自动更新，不需要再手动点。

### 第 4 步 · 开代理

1. 主界面 → 模式选 **规则**；
2. 打开开关，允许 VPN 连接（TUN 已默认开启）；
3. 进 **代理** 页确认：`🚀 节点选择` 里有节点，`⚡ 自动选择` 不是空的。

---

## 广告拦截怎么控制

所有广告规则都指向同一个策略组：**`🛡️ 广告拦截`**，默认值 `REJECT`。这就是拦截总开关。

| 场景 | 操作 |
|---|---|
| 日常使用 | 什么都不用做，保持 `REJECT` |
| 某 App 登录 / 支付异常，怀疑误杀 | 临时切成 `DIRECT`，重试一次 |
| 切 `DIRECT` 后恢复正常 | 确认是广告规则误杀 → 按[误杀排查](#误杀排查)精准加白 |
| 想让某类流量走代理而非拦截 | 切成 `🚀 节点选择`（一般不需要） |

> ⚠️ 不要长期停在 `DIRECT`，那等于关掉了这套配置的主要功能。

### 拦截来源

| 规则集 | 上游 | 覆盖重点 | 规则数 |
|---|---|---|---|
| `ads-anti` | [anti-AD](https://github.com/privacy-protection-tools/anti-AD)（DustinWin 的 mrs 镜像） | **中文站点 / 国内 App 广告**（主力） | 约 10.6 万 |
| `ads-awa` | [AWAvenue-Ads-Rule](https://github.com/TG-Twilight/AWAvenue-Ads-Rule) | App 开屏、信息流、广告 SDK 补充 | 约 970 |
| `ads-meta` | [MetaCubeX geosite](https://github.com/MetaCubeX/meta-rules-dat) `category-ads-all` | 国际广告 / 追踪域名兜底 | 约 900 |

实测命中记录（Mihomo 日志）：

```text
--> mi.gdt.qq.com:443                 match RuleSet(ads-anti) using 🛡️ 广告拦截[REJECT]   # 广点通
--> pagead2.googlesyndication.com:443 match RuleSet(ads-anti) using 🛡️ 广告拦截[REJECT]   # Google 广告
--> res.wx.qq.com:443                 match DomainSuffix(res.wx.qq.com) using DIRECT      # 微信资源未被误杀
```

---

## 分流逻辑

规则**从上到下**匹配，位置越靠前优先级越高：

```text
0. 兼容白名单（微信 / 支付宝 / 12306 / 风控验证码…）
1. 私有地址与局域网
2. 广告拦截（三层）
3. 服务分流：AI / Google / YouTube·Netflix·Spotify / Telegram / TikTok / 微软 / OneDrive / GitHub / Apple
4. 地域兜底：非中国域名 → 代理；中国域名与 IP → 直连
5. 剩余流量 → 🐟 漏网之鱼
```

### 策略组

| 组 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `🚀 节点选择` | 手动 | `⚡ 自动选择` | 全局出口 |
| `⚡ 自动选择` | 自动测速 | — | 全部节点中择优，懒测速省电 |
| `🛡️ 广告拦截` | 手动 | `REJECT` | 拦截总开关 |
| `🤖 AI 服务` | 手动 | `🚀 节点选择` | ChatGPT / Claude / Gemini 等，可手动指定美国 |
| `📺 国际流媒体` | 手动 | `🚀 节点选择` | YouTube / Netflix / Spotify，可手动指定香港 |
| `🍀 Google` | 手动 | `🚀 节点选择` | 搜索、Play 服务、登录链路 |
| `📲 Telegram` | 手动 | `🚀 节点选择` | 想固定地区时手动选香港 / 新加坡 |
| `🎵 TikTok` | 手动 | `🚀 节点选择` | 需手动切到 US / SG 节点 |
| `Ⓜ️ 微软服务` | 手动 | 直连 | 微软、OneDrive、Xbox |
| `🐟 漏网之鱼` | 手动 | 节点选择 | 未命中任何规则的流量 |
| `🇭🇰 🇯🇵 🇸🇬 🇺🇸` 节点池 | 手动 | 自动归类 | 按节点名关键词自动分组 |

> **服务组默认都指向 `🚀 节点选择`**，而不是某个地区组。原因是：机场如果没有该地区节点，地区组会是空的，指向它的服务就会直接断掉。需要固定地区时手动切一下即可。
>
> 地区分组依赖**节点名称关键词**（香港 / HK、日本 / JP、新加坡 / SG、美国 / US…）。
> 机场命名不含这些词时，节点进不了对应分组 —— 按[自定义](#自定义)改成你的关键词即可。

---

## 自定义

### 添加分流规则

写在 `config-rule.yaml` 的 `rules:` 段，**越靠上优先级越高**：

```yaml
  - DOMAIN-SUFFIX,example.com,🚀 节点选择   # 该域名走代理
  - DOMAIN-SUFFIX,example.cn,DIRECT         # 该域名直连
  - IP-CIDR,1.2.3.0/24,DIRECT,no-resolve    # 该网段直连
```

### 添加自己的广告域名

1. 把域名写入 `my-ad.yaml`：

```yaml
payload:
  - "ads.example.com"
  - "tracker.example.net"
```

2. 在 `config-rule.yaml` 的 `rule-providers` 中取消这行注释：

```yaml
  ads-custom: {<<: *domain, format: yaml, url: "https://fastly.jsdelivr.net/gh/lhflowersbloom/Flclash-rule@main/my-ad.yaml", path: ./ruleset/ads-custom.yaml}
```

3. 在 `rules:` 的广告段取消这行注释：

```yaml
  - RULE-SET,ads-custom,🛡️ 广告拦截
```

4. 保存 → 更新规则集 → 生效。

### 添加白名单

写入 `rules:` **最顶部**的「兼容白名单」段，位置比拦截规则靠前就一定生效：

```yaml
  - DOMAIN-SUFFIX,某个App的域名.com,DIRECT
```

---

## 误杀排查

手机上最容易被广告规则误杀的是五类功能：**登录注册、支付、人机验证、消息推送、应用更新**。处理顺序如下：

1. FlClash → **日志**，把级别调到 `info`（或临时把配置改成 `log-level: info`）；
2. 复现问题，例如点一次登录；
3. 在日志里搜 `REJECT`：

```text
--> api.xxx.com:443 match RuleSet(ads-anti) using 🛡️ 广告拦截[REJECT]
```

4. 命中的域名加进白名单（`DOMAIN-SUFFIX,xxx.com,DIRECT`），保存并更新规则集；
5. 日志里**没有** `REJECT` 时，问题通常在节点、DNS 或 App 自身，不要乱删规则。

已预置的白名单，覆盖最常见的几处误杀：

```text
微信 / 微信支付：weixin.qq.com、res.wx.qq.com、wechat.com、tenpay.com、wechatpay.com、mmbiz.qpic.cn
支付宝：alipay.com、alipayobjects.com
其他：mybank.cn、12306.cn
风控与人机验证：online-metrix.net、hcaptcha.com、challenges.cloudflare.com
```

---

## 性能设计

手机端的性能瓶颈通常不在内核，而在规则的组织方式与更新频率。这套配置做了几件事：

| 设计 | 原因 |
|---|---|
| 规则集全部 `mrs`（二进制） | 相比 `.yaml` / `.list` 文本规则，加载更快、内存占用低一个量级 |
| `find-process-mode: "off"` | 手机端按进程匹配收益有限，却要常驻扫描，关掉更省电 |
| `lazy: true` 懒测速 | 只有真正用到某个组时才测速，避免后台反复打点 |
| `interval: 86400` | 规则集每天更新一次足够，减少启动与切换时的等待 |
| 规则集下载 `proxy: DIRECT` | 下载不依赖节点：节点全挂时规则集仍能更新，不会连带规则失效 |
| 只保留必要策略组 | 组越少，启动越快，切换越省内存 |

> 想进一步减负：把 `ads-anti` 换成更小的源，或在 `rules` 里注释掉 `ads-awa` / `ads-meta`。
>
> 若你的网络访问不了 jsDelivr 直连，把 `rule-anchor` 里的 `proxy: DIRECT` 改成 `proxy: 🚀 节点选择`。

---

## 与 js 覆写脚本的区别

同一个生态里有两条路，本仓库选的是前者：

| | 本仓库 `.yaml`（完整配置） | 社区 `.js`（覆写脚本） |
|---|---|---|
| 本质 | 订阅、策略组、规则全在一份文件里 | 一段补丁，运行时改写你的订阅配置 |
| 换机场 | 改文件里的订阅链接 | 不用动脚本，刷新订阅即可 |
| 生效方式 | 导入文件后整份替换 | 配置卡「覆写」里挂载，叠加在订阅之上 |
| 出错表现 | 有 schema 校验，定位明确 | 语法错误整份失效，需看日志 |
| 启动开销 | 直接解析，毫秒级 | 需先执行脚本，轻量脚本几十至几百毫秒 |
| 依赖风险 | 标准格式，内核升级基本不受影响 | 依赖客户端脚本 API 版本，可能失效 |

两者**运行时的转发性能没有差别** —— 脚本只负责生成配置，最终跑的还是同一个 Go 内核。真正让手机变卡的是「超大脚本 + 大量内联规则」，而不是 `.js` 这个后缀。

---

## 常见问题

**Q：某个服务连不上，但别的服务正常（比如 Telegram 不通、Play 商店正常）？**
A：这是"服务组指向了空的地区组"的典型表现。早期版本把 `📲 Telegram` 默认指向 `🇸🇬 新加坡节点`，如果机场没有新加坡节点，那个组就是空的，选中它等于直连，于是只有 Telegram 断掉。现在所有服务组默认指向 `🚀 节点选择` 已修掉这个问题。若你用的是旧文件，手动把该组切成 `🚀 节点选择` 即可。

**Q：怎么确认到底是哪一步出问题？**
A：看日志，按下面顺序对号：

```text
dial 📲 Telegram (match RuleSet/telegram-domain)  ... refused / timeout   → 规则命中没问题，是这个组的节点不可用
dial 🐟 漏网之鱼 (match Match/)  --> t.me                                 → 规则没命中，多半是规则集没下载成功
proxy not found                                                          → 规则指向的策略组名字写错了
DNS failed / context deadline exceeded                                   → 节点或 DNS 有问题，与规则无关
```

**Q：导入后节点是空的？**
A：订阅没改成自己的链接，或者订阅不是 Clash 格式。改 `Airport1.url` 后重新「更新订阅」。

**Q：规则集一直更新失败？**
A：全部规则集走 `fastly.jsdelivr.net`，国内一般可直连。若所在网络访问不了，把 URL 里的 `fastly.jsdelivr.net` 换成 `cdn.jsdelivr.net`；仍失败就把 `rule-anchor` 里的 `proxy: DIRECT` 改成 `proxy: 🚀 节点选择`，让规则集走代理下载。

**Q：广告还在？**
A：依次确认三件事：① `🛡️ 广告拦截` 是 `REJECT`；② 规则集已成功更新；③ 该 App 的广告是否走 IP 硬编码或自带 DoH —— 这类 DNS 层拦不住，任何 Clash 配置都一样。看日志能确认域名有没有命中。

**Q：拦截太狠，某个 App 打不开？**
A：按[误杀排查](#误杀排查)走一遍。临时解法是把 `🛡️ 广告拦截` 切成 `DIRECT` 验证。

**Q：能按 App 分流吗？**
A：默认关闭了进程匹配（`find-process-mode: "off"`）以省电。需要时改成 `strict`，再写 `PROCESS-NAME,包名,策略组`。注意 Android 上部分应用读不到进程名，效果因系统而异。

**Q：iPhone 能用吗？**
A：可以。FlClash / Clash Meta 系客户端导入同一份配置；客户端不支持 TUN 时改由系统 VPN 接管，规则不变。

**Q：这份配置校验过吗？**
A：是。用 **Mihomo Meta v1.19.14** 实测：`mihomo -t` 校验通过，23 个规则集全部下载解析成功（含 10.6 万条广告域名），实际请求在 `REJECT`、`DIRECT` 与各服务组的命中均符合预期。

---

## 致谢与免责

**上游项目**

- 原配置：[yudong820/Flclash-rule](https://github.com/yudong820/Flclash-rule)，其模板参考 [qichiyuhub](https://github.com/qichiyuhub)
- 广告规则：[anti-AD](https://github.com/privacy-protection-tools/anti-AD)、[AWAvenue-Ads-Rule](https://github.com/TG-Twilight/AWAvenue-Ads-Rule)
- 分流规则：[MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)、[DustinWin/ruleset_geodata](https://github.com/DustinWin/ruleset_geodata)、[wwqgtxx/clash-rules](https://github.com/wwqgtxx/clash-rules)
- 脚本思路参考：[Wuzhenshi8874/flclash-v-series](https://github.com/Wuzhenshi8874/flclash-v-series)

**工具与协作**

- [WorkBuddy](https://www.workbuddy.cn)：本仓库的配置重构、规则源核验与文档整理均在 WorkBuddy 中完成
- [DeepSeek](https://www.deepseek.com)：提供推理模型支持（DeepSeek-V4.1-Flash）

**免责**

本仓库只提供客户端分流配置，不提供任何代理服务或节点。请遵守所在地区的法律法规，仅用于合法的网络访问需求。
