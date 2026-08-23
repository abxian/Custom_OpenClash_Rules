# 跨境电商 OpenClash 防泄漏 / 防关联 配置方案

一套面向 **跨境电商 / 多平台多账号运营** 的 OpenClash 完整方案。核心解决两件事：

1. **防泄漏** —— 杜绝 DNS、IPv6、WebRTC 三大泄漏渠道，保证出口 IP 与归属地不被真实网络暴露。
2. **防关联** —— 各社媒 / 电商 / AI 平台独立分组，可为每个账号钉死固定节点，避免多账号因共用 IP / 指纹被平台风控判定为“关联账号”。

配套两个规则仓库：

| 仓库 | 作用 |
|------|------|
| [abxian/Custom_OpenClash_Rules](https://github.com/abxian/Custom_OpenClash_Rules) | 订阅转换模板（`.ini`）与专线订阅规则，基于 Aethersailor 全分组防泄漏模板魔改 |
| [abxian/stash-override](https://github.com/abxian/stash-override) | 各平台分流规则清单 `rule/*.list`（Facebook / TikTok / Amazon / OpenAI 等），blackmatrix7 格式 |

---

## 一、文件清单

| 文件 | 说明 |
|------|------|
| `config.yaml` | 可直接导入 OpenClash 的完整配置：含节点、全部策略组、防泄漏 DNS、Sniffer、平台分流规则 |
| `templates/Custom_Ecommerce.ini` | **订阅转换模板**：把任意机场/专线订阅，一键转换成本方案的“防泄漏 + 平台分组”结构，规则引用 `stash-override` |
| `config.yaml.bak.*` | 修改前的原始配置备份 |

> 两种用法二选一：
> - **A. 直接用 `config.yaml`** —— 节点写死在文件里，适合节点固定、要求最高可控性的场景。
> - **B. 用 `Custom_Ecommerce.ini` 模板** —— 只填订阅链接，节点由订阅自动更新，适合专线/机场订阅会变的场景。
> 两者的分组结构与防泄漏原理完全一致。

---

## 二、防泄漏原理（重点）

“翻墙能上网”不代表“没泄漏”。跨境电商真正致命的是**归属地/真实 IP 被平台探测到**。泄漏有三条主要通道，本方案逐一封堵：

### 1. DNS 泄漏 —— 用 Fake-IP + 加密上游解决

**问题**：如果域名在本地用运营商 DNS 解析，ISP 就知道你在访问 facebook.com；解析结果也可能被污染或返回国内 CDN，暴露“人在中国”。

**方案**（见 `config.yaml` 的 `dns:` 段）：
```yaml
dns:
  enable: true
  enhanced-mode: fake-ip          # 关键：代理域名不在本地做真实解析
  fake-ip-range: 198.18.0.1/16
  nameserver:  [https://223.5.5.5/dns-query, https://doh.pub/dns-query]   # 国内域名走国内加密 DNS
  fallback:    [https://1.1.1.1/dns-query, https://8.8.8.8/dns-query]     # 国外域名走境外加密 DNS
  fallback-filter: { geoip: true, geoip-code: CN }                        # 非 CN 结果一律用 fallback
```
- **Fake-IP**：访问 `facebook.com` 时，Clash 直接返回一个假 IP（198.18.x.x），**本地根本不发真实 DNS 查询**；真实解析在你连上代理服务器后，由**服务器端远程完成**。ISP 全程只看到一堆加密的代理流量 → DNS 无从泄漏。
- **加密上游（DoH/DoT）**：即便是需要本地解析的域名，也走 `https://` / `tls://` 加密通道，ISP 无法看到/篡改。
- **fallback-filter geoip CN**：海外域名强制采用境外 DNS 结果，避免被 CDN 定位回国内。

### 2. IPv6 泄漏 —— 直接全局关闭

**问题**：很多代理只代理 IPv4，系统却优先用 IPv6 直连出去，于是 IPv6 请求**绕过代理**，暴露真实 IPv6 地址（能精确定位到宽带账号）。

**方案**：
```yaml
ipv6: false          # 全局关闭
dns:
  ipv6: false        # DNS 不返回 AAAA 记录，从源头断掉 IPv6
```
> 若在路由器上跑 OpenClash，还需在 **OpenClash → 覆写设置 → 常规 → 禁用 IPv6** 一并关闭，并在浏览器 `about:config` / 系统里确认无 IPv6 出口。

### 3. WebRTC 泄漏 —— 拦截 STUN

**问题**：浏览器的 WebRTC 会通过 **STUN 服务器**直接探测本机公网 IP，**完全绕过代理**——这是跨境电商最常被忽视、也最致命的泄漏点。

**方案**：新增 `🎥 WebRTC` 策略组（默认 **REJECT**），并把 STUN 流量导入其中：
```yaml
- DOMAIN-KEYWORD,stun,🎥 WebRTC
- DOMAIN-SUFFIX,stun.l.google.com,🎥 WebRTC
```
> Clash 只能拦截 STUN 信令。**浏览器层面仍强烈建议**安装 “WebRTC Control / WebRTC Leak Prevent” 插件，或 Firefox 里设 `media.peerconnection.enabled=false`，双保险。

### 4. IP 直连绕过 —— Sniffer 兜底

**问题**：部分 App 不做 DNS 解析，直接用硬编码 IP 连接，导致基于域名的分流规则失效，可能真实 IP 直连或走错节点。

**方案**：开启域名嗅探，从 TLS SNI / HTTP Host / QUIC 中**还原域名**，让规则重新生效：
```yaml
sniffer:
  enable: true
  force-dns-mapping: true
  parse-pure-ip: true
```

### 5. FINAL 走代理，不走直连

模板与配置的最后一条都是 `MATCH → 🐟 漏网之鱼`，而漏网之鱼**指向代理**而非 DIRECT。
> 原因：任何没被规则命中的“漏网”境外流量若直连，就会在 DNS 泄漏检测中暴露。宁可多走代理，绝不漏网直连。

---

## 三、防关联原理（跨境电商核心）

平台判定“关联账号”的两大维度：**出口 IP** 和 **浏览器指纹**。本方案负责**出口 IP 侧**：

- 核心平台都是独立的 `select` 策略组：`Facebook`、`Instagram`、`Twitter`、`TikTok`、`Amazon`、`Temu`、`SHEIN`、`eBay`、`AliExpress`、`Shopify`、`AI服务`……长尾电商由 `其他国际电商` 统一兜底。
- `profile.store-selected: true` —— **记住每个组手动选定的节点，重启不丢**。
- 组内优先列出**固定 IP 的直连节点**（如 `美国01直连`、`阿曼01直连`），这类节点 IP 稳定，最适合“一账号一 IP”。

**实操建议**：
1. 一个账号 = 一个固定出口节点，长期不变（切 IP 本身就是风控信号）。
2. 不要给电商/社媒账号用 `♻️ 自动选择`（会自动切换 IP → 触发关联/风控）。
3. 多账号请扩充节点池，做到 IP 互不重叠；配合独立浏览器指纹环境（如多开浏览器/指纹浏览器）效果最佳。
4. 模板不内置广告拦截，避免影响广告投放、归因和平台后台；需要拦截时应另建规则并充分验证业务链路。

---

## 四、OpenClash 一般设置步骤

> 以 OpenWrt / iStoreOS 上的 OpenClash 为例。

1. **导入配置**
   - 方式 A：`配置管理 → 上传` 直接上传 `config.yaml`。
   - 方式 B：`配置订阅 → 添加` 填入“订阅转换”生成的链接（见第五节），选用本项目模板。
2. **运行模式**：`覆写设置 → 常规 → 运行模式 = Fake-IP（增强模式）`。**这是防 DNS 泄漏的前提**，务必启用。
3. **DNS 设置**：
   - 勾选 `启用自定义上游 DNS 服务器`，让 `config.yaml` 里的 `dns:` 段生效；
   - 若开启了 OpenClash 自带的“DNS 劫持/重定向”，确保不与自定义 DNS 冲突（推荐：DNS 劫持开，上游用自定义）。
4. **关闭 IPv6**：`覆写设置 → 常规 → 禁用 IPv6`。
5. **绑定网卡/接口**：按需绑定 LAN，确保内网设备全部经 OpenClash。
6. **General → 其他**：开启 `TCP 并发`、`统一延迟`（配置里已含 `tcp-concurrent / unified-delay`）。
7. 启动后到 **面板（Dashboard）** 里，进入各平台策略组，为每个账号手动指定固定节点。

---

## 五、订阅转换模板用法（`Custom_Ecommerce.ini`）

模板用于把**你的专线/机场订阅**自动转换成本方案结构，无需手写节点。

**在线/自建 subconverter 拼接格式**：
```
https://<你的subconverter后端>/sub?target=clash
  &url=<URL编码后的机场订阅链接>
  &config=https://testingcf.jsdelivr.net/gh/abxian/Custom_OpenClash_Rules@main/templates/Custom_Ecommerce.ini
  &emoji=true&list=false&udp=true&scv=true&sort=false
```
> - `url=` 填你的专线订阅（记得 URL 编码）。
> - `config=` 指向本模板（把它推到你的 `Custom_OpenClash_Rules` 仓库 `templates/` 后即可用 jsdelivr 引用）。
> - 生成的链接填进 OpenClash 的“配置订阅”。

**模板做了什么**（`templates/Custom_Ecommerce.ini`）：
- 规则全部引用 `stash-override/rule/*.list`（Facebook / TikTok / Amazon / OpenAI / Claude / Gemini / PayPal / Stripe …），随仓库更新自动生效。
- 生成核心社媒与电商独立组、`其他国际电商` 长尾兜底组，以及 `AI服务`、`流媒体`、`WebRTC(默认REJECT)`、自动测速和专线中转等分组。
- 规则顺序：直连兜底 → 广告拦截 → WebRTC拦截 → 各平台 → 境外兜底(GFW) → 国内兜底(ChinaMax/GEOSITE:cn) → `FINAL 走代理`。

---

## 六、策略组一览

| 组名 | 类型 | 用途 |
|------|------|------|
| 🚀 手动选择 / 节点选择 | select | 总入口，聚合全部节点与地区组 |
| ♻️ 自动选择 | url-test | 自动选延迟最低（**不建议给电商账号用**） |
| 🔒 专线中转 / 专线中转 | select/url-test | 专线/中转节点 |
| 📘 Facebook / 📷 Instagram / 🐦 Twitter / 🎵 TikTok / 📲 Telegram | select | 社媒各自独立，账号级固定 IP |
| 🤖 AI服务 | select | ChatGPT / Claude / Gemini / Copilot 等 |
| Amazon / Temu / SHEIN / eBay / AliExpress / Shopify | select | 核心电商平台分别固定节点 |
| 其他国际电商 | select | Etsy / Lazada / Shopee / Walmart 等长尾平台统一兜底 |
| 🎥 流媒体 | select | YouTube / Netflix / Disney / Spotify |
| 🎥 WebRTC | select | 默认 **REJECT**，防真实 IP 探测 |
| 🎯 全球直连 / 🛑 全球拦截 / 🐟 漏网之鱼 | select | 直连 / 广告拦截 / 兜底(走代理) |

---

## 七、验证是否泄漏

部署后依次自测：

1. **DNS 泄漏**：<https://dnsleaktest.com>（Extended Test）→ 结果里**不应出现中国的 DNS 服务器**。
2. **IPv6**：<https://test-ipv6.com> → 应显示无 IPv6，或 IPv6 与代理出口一致。
3. **WebRTC**：<https://browserleaks.com/webrtc> → **Public IP 不应显示你的真实公网 IP**。
4. **出口 IP / 归属地**：<https://whoer.net> 或 <https://ipinfo.io> → 应为所选节点的国家，`whoer` 匿名度尽量高。
5. **时区/语言**：浏览器时区、系统语言要与出口国匹配（这属于指纹侧，需自行调整）。

---

## 八、可继续完善的方向

- **规则更全**：`config.yaml` 目前用域名清单，可改用 GEOSITE 规则集（`category-social-media-!cn`、`category-ecommerce`、`category-ai-!cn`）覆盖更全，需开启 OpenClash 的 GeoData/Meta 内核。模板 `.ini` 已用 GEOSITE 兜底。
- **按国家站点细分**：如 `amazon.com` / `amazon.co.uk` / `amazon.de` 各自钉不同国家节点，进一步贴合站点归属。
- **流媒体解锁分组**：Netflix / Disney 拆出独立解锁组，选支持对应区解锁的节点。
- **每账号独立出口**：节点数量不足时，多账号会共享 IP，存在关联风险；建议扩充节点或采购住宅 IP。

---

## 九、致谢

- 防泄漏模板思路：[Aethersailor/Custom_OpenClash_Rules](https://github.com/Aethersailor/Custom_OpenClash_Rules)
- 分流规则来源：[blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script) 及本项目 `stash-override`
