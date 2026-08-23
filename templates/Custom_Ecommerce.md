# 跨境电商防泄漏模板 · 订阅转换调用说明

> 本文只讲**这一个模板 `Custom_Ecommerce.ini` 怎么用**。
> 方案原理（防 DNS/IPv6/WebRTC 泄漏、防关联）见同目录 [`README.md`](./README.md)。

---

## 一、模板是什么

`Custom_Ecommerce.ini` 是一个 **subconverter（订阅转换）配置模板**，它本身**不是订阅链接**，不能直接填进 OpenClash。

它的作用：把「你的机场 / 专线订阅」里的裸节点，**自动套上**这套跨境电商结构 ——

- 防泄漏：Fake-IP DNS、关 IPv6、拦截 WebRTC(STUN)、FINAL 走代理
- 平台分组：30 个策略组，社媒 / 电商 / 支付 / 验证 / 物流 全覆盖（见下表）
- 非标端口：`非标端口` 组统一接管 80/443 之外的端口（部分节点不支持非标端口）
- 分流规则：**平台清单 + GEOSITE 双保险**，清单来自 [`abxian/stash-override`](https://github.com/abxian/stash-override)，随仓库自动更新
- 不含广告拦截，便于广告投流

> 说明：模板**不做地区分类**，所有节点直接平铺在各分组里，按需手动钉节点即可。

### 1.1 策略组与覆盖平台

| 策略组 | 覆盖平台 |
|--------|---------|
| **Facebook** | Facebook、Messenger |
| **Instagram** | Instagram、Threads |
| **WhatsApp** | WhatsApp |
| **Twitter** | Twitter / X |
| **TikTok** | TikTok |
| **Telegram** | Telegram |
| **社交媒体** | Snapchat、Reddit、Pinterest、LinkedIn、Discord、Line、Kakao、VK、Quora、Tumblr、Signal |
| **Amazon** | Amazon 商城、卖家后台、广告及相关生态域名 |
| **Temu** | Temu |
| **SHEIN** | SHEIN |
| **eBay** | eBay |
| **AliExpress** | AliExpress |
| **Shopify** | Shopify 商店与后台 |
| **其他国际电商** | Etsy、Lazada、Shopee、MercadoLibre、Walmart、Rakuten、BestBuy、Alibaba、Target、Costco、Wish、Coupang 等长尾平台 |
| **支付平台** | PayPal、Stripe、Payoneer、Airwallex、Wise |
| **验证服务** | Cloudflare Turnstile、hCaptcha、reCAPTCHA、Arkose/FunCaptcha、GeeTest、DataDome、PerimeterX、Imperva、ThreatMetrix、FingerprintJS、Sift、Kount、Riskified、Signifyd、Forter、Auth0、Okta、Twilio/Authy |
| **物流快递** | DHL、FedEx、UPS、USPS、TNT、Aramex、DPD、GLS、Evri、Purolator、OnTrac、LaserShip、各国邮政（英/加/澳/日/韩/法/德/荷/西/意/北欧）、17TRACK、AfterShip、TrackingMore、4PX、云途、燕文、顺友、万邑通、谷仓、J&T、Flexport、**海关税务**（US CBP、UK HMRC、Avalara、TaxJar） |
| **AI服务** | OpenAI/ChatGPT、Claude、Gemini、Copilot 等 |
| **流媒体** | YouTube、Netflix、Disney+、Spotify、Twitch |
| **谷歌服务 / GitHub** | Google、GitHub |
| 节点选择 / 自动选择 | 顶层入口；自动选择自动排除专线中转节点 |
| 中转选择 / 专线中转 / 专线自动选择 | 链式代理(dialer-proxy)专用，不出现在其它可选组 |
| WebRTC / 非标端口 / 全球直连 / 漏网之鱼 | 防泄漏与兜底 |

### 1.2 ⚠️ 关于「验证服务」组的重要说明

验证服务（人机验证、风控、短信验证码）**理应跟随当前操作账号的出口 IP** ——
验证 IP 与账号 IP 不一致，是触发平台风控最常见的原因之一。

模板的处理方式：

- **平台自有验证端点**（如 OpenAI 的 `openai-api.arkoselabs.com`）→ 仍跟随各自平台组；
- **全站共用验证端点**（如 `challenges.cloudflare.com`）→ 归入 `验证服务` 组。

**局限**：`验证服务` 是单一策略组，同时运营多平台/多账号时，一个组无法同时匹配所有账号的出口。

**严格运营的正解**：用 `SRC-IP-CIDR` 按设备分流，让一台养号机的**全部流量（含验证）**走同一个 IP，
再叠加 [断网保护](./KillSwitch-断网保护.md)。共享 `验证服务` 组只适合单账号 / 单平台场景。

### 1.3 规则顺序原则（自定义时必须遵守）

规则自上而下匹配、命中即停，**子品牌必须排在母公司之前**，否则会被抢先命中分错节点：

| 必须在前 | 必须在后 | 原因 |
|---------|---------|------|
| WhatsApp / Instagram / Threads / Messenger | Facebook | `geosite:facebook` 收录了这些子品牌域名 |
| YouTube | 谷歌服务 | `geosite:google` 收录了 YouTube 域名 |
| 支付平台 | AI服务 | OpenAI 清单内含 `stripe.com` 计费域名 |
| 验证服务 | 谷歌服务 | `Google.list` 内含 `recaptcha.net` |
| 各平台组 | 验证服务 | 让平台自有验证端点跟随各自平台 |
| Amazon / Temu / SHEIN / eBay / AliExpress / Shopify | 其他国际电商 | 聚合清单包含部分核心平台域名，独立规则必须优先命中 |

> 另注：模板**不引用 `Global.list`**（3.2 万条，与 `geosite:geolocation-!cn` 完全重复，
> 且会撑爆 subconverter 规则上限导致整个模板被拒、回退成默认配置）。

---

## 二、模板地址（config= 用它）

| 线路 | 地址 |
|------|------|
| jsdelivr CDN（国内快，推荐） | `https://testingcf.jsdelivr.net/gh/abxian/Custom_OpenClash_Rules@main/templates/Custom_Ecommerce.ini` |
| GitHub raw（改完即时生效） | `https://raw.githubusercontent.com/abxian/Custom_OpenClash_Rules/main/templates/Custom_Ecommerce.ini` |

---

## 三、订阅转换怎么调用这个模板

### 原理
`subconverter` 后端接收两样东西：你的**订阅链接**(`url=`) + 这个**模板**(`config=`)，
合成后输出一条全新的 Clash 订阅链接，这条链接才填进 OpenClash。

```
你的机场订阅  ┐
             ├──►  subconverter 后端  ──►  合成后的 Clash 订阅链接  ──►  OpenClash
本模板 .ini   ┘
```

### URL 拼接格式
```
https://<subconverter后端>/sub?target=clash&url=<订阅链接>&config=<模板地址>&emoji=true&list=false&udp=true&scv=true&sort=false
```

| 参数 | 必填 | 说明 |
|------|:---:|------|
| `target` | ✅ | 固定 `clash` |
| `url` | ✅ | **你的机场/专线订阅链接，必须 URL 编码** |
| `config` | ✅ | **本模板地址，必须 URL 编码** |
| `udp` | 建议 | `true`，开启 UDP（游戏/QUIC 需要） |
| `scv` | 建议 | `true`，跳过证书校验（部分专线需要） |
| `emoji` | 可选 | `true`，节点带国旗 emoji |
| `list` | 可选 | `false`，输出完整配置而非纯节点列表 |
| `sort` | 可选 | `false`，不打乱节点顺序 |

> ⚠️ `url=` 和 `config=` 的值里含 `:` `/` `?` `&`，**必须先做 URL 编码**，否则参数会被截断。

### 完整实例
假设你的专线订阅是 `https://jc.example.com/sub?token=abc123`：

```
https://api.dler.io/sub?target=clash&url=https%3A%2F%2Fjc.example.com%2Fsub%3Ftoken%3Dabc123&config=https%3A%2F%2Ftestingcf.jsdelivr.net%2Fgh%2Fabxian%2FCustom_OpenClash_Rules%40main%2Ftemplates%2FCustom_Ecommerce.ini&emoji=true&list=false&udp=true&scv=true&sort=false
```

把这一整条链接填进 OpenClash 的「配置订阅 → 订阅地址」即可。

### URL 编码小抄
只需替换这几个字符：

| 原字符 | 编码 |
|:--:|:--:|
| `:` | `%3A` |
| `/` | `%2F` |
| `?` | `%3F` |
| `&` | `%26` |
| `=` | `%3D` |
| `@` | `%40` |

不想手动编码：打开 <https://www.urlencoder.org>，把订阅链接和模板地址分别粘进去编码后再拼。

---

## 四、subconverter 后端怎么选

| 方式 | 地址 | 说明 |
|------|------|------|
| 图形界面（最简单） | <https://sub-web.netlify.app> | 表单里填订阅 + 选「远程配置」粘模板地址，点生成，自动帮你编码 |
| 公共 API | `https://api.dler.io/sub`、`https://sub.xeton.dev/sub` | 直接拼 URL 用 |
| **自建（最安全，推荐专线用）** | 自己部署 [tindy2013/subconverter](https://github.com/tindy2013/subconverter) | 专线密钥不经过第三方，避免泄漏 |

> 🔒 **安全提醒**：用公共后端时，你的订阅链接（含 token/密钥）会经过对方服务器。**专线/付费节点强烈建议自建后端。**

### 4.1 自建后端 + 防泄漏 base（关键：让订阅自带 dns / sniffer / ipv6）

模板只生成「策略组 + 规则」，顶层 `dns / sniffer / ipv6` 来自后端的 **`clash_rule_base`（基础配置）**。
公共后端改不了它，所以要让防泄漏写进订阅，必须**自建后端**并把 base 换成本项目的
[`Custom_Ecommerce_Base.yaml`](./Custom_Ecommerce_Base.yaml)。

**① 部署后端（Docker）**
```bash
docker run -d --name subserver -p 25501:25500 asdlokj1qpi23/subconverter:latest
```

**② 把 `clash_rule_base` 指向防泄漏 base**（配置文件是 `/base/pref.toml`）
```bash
docker exec subserver sh -c '
URL="https://raw.githubusercontent.com/abxian/Custom_OpenClash_Rules/main/templates/Custom_Ecommerce_Base.yaml"
cp /base/pref.toml /base/pref.toml.bak
sed -i "s#^clash_rule_base = .*#clash_rule_base = \"$URL\"#" /base/pref.toml
grep -n "^clash_rule_base" /base/pref.toml
'
docker restart subserver
```

**③ 验证**（输出顶部应出现 `dns:` / `sniffer:` / `ipv6: false`）
```bash
curl -s "http://127.0.0.1:25501/sub?target=clash&url=ss://YWVzLTI1Ni1nY206dGVzdA%3D%3D@1.1.1.1:8388%23test&config=https%3A%2F%2Fraw.githubusercontent.com%2Fabxian%2FCustom_OpenClash_Rules%2Fmain%2Ftemplates%2FCustom_Ecommerce.ini" | head -40
```

**④ 前端指向自建后端**：改 sub-web 的 `.env`：
```ini
VUE_APP_SUBCONVERTER_DEFAULT_BACKEND = "http://你的域名:25501"
```

**⑤ 持久化**：容器内改动在 `docker rm` 重建时会丢失，重建时挂载配置：
```bash
-v /你的路径/pref.toml:/base/pref.toml
```

> **关于 `dns.listen: 0.0.0.0:1053`**：这是 Clash 内置 DNS 的监听端口。
> - **OpenClash / 路由器场景必须用非 53 端口** —— 53 已被 dnsmasq 占用，改成 53 会冲突导致 DNS 崩溃；
> - fake-ip + TUN 模式下 DNS 被劫持，与该端口无关。
> - **结论：保持 1053 无任何危害**，仅当你要把某台设备的 DNS 直接指向 Clash:53 时才需要 53。

---

## 五、生成后在 OpenClash 的设置

1. `配置订阅 → 添加` → 填入合成后的链接 → 更新 → 启用。
2. `覆写设置 → 运行模式 = Fake-IP（增强模式）` ← 防 DNS 泄漏前提。
3. `覆写设置 → 开启 TUN 模式`（Meta/Mihomo 内核）← 兜住 UDP/QUIC，防真实 IP 泄漏。
4. `覆写设置 → 禁用 IPv6`。
5. 进面板，给每个电商/社媒账号在对应平台组里**钉一个固定节点**（防关联）。

> TUN 与 Fake-IP 不是二选一：Fake-IP 管 DNS 不泄漏，TUN 管把所有流量（含 UDP/QUIC）抓进代理，**两个一起开**才是最强防泄漏组合。

---

## 六、常见问题

- **⚠️ 生成的配置里没有 `Amazon`/`Temu`/`其他国际电商`/`支付平台` 这些组，反而是 `🔰 节点选择`/`🌍 国外媒体`？**
  说明**模板根本没生效**，subconverter 拒绝了它并回退成自带的默认配置——这也是「域名漏到其它节点」的典型症状。
  两个常见原因：
  1. **规则总量超上限**：某个引用的清单太大（例如 `Global.list` 有 3.2 万条），撑爆后端的 `max_allowed_rules`。
     解决：不要引用超大且重复的清单，用 `GEOSITE` 标签替代（一行由内核解析，不占规则量）。
  2. **ruleset 条目超上限**：`ruleset=` 行数超过 `max_allowed_rulesets`（默认 64）。
     解决：把同类平台合并成聚合清单（本模板即用 `Ecommerce_All` / `Social_All` 等），或在自建后端的
     `pref.toml` 里调大 `max_allowed_rulesets` / `max_allowed_rules`。

  **自查方法**：生成订阅后看策略组名——出现 `Amazon`、`Temu`、`其他国际电商`、`支付平台`、`验证服务` 才说明模板真正生效。当前模板有 62 条 `ruleset=`，低于 subconverter 默认的 64 条上限。
- **改了模板 / 清单，订阅却没变化？**
  三层缓存都要考虑：① 自建后端 → `docker restart subserver`；② GitHub raw 约 5 分钟；
  ③ jsdelivr 对分支引用（`@master`）可缓存数小时，可用 <https://www.jsdelivr.com/tools/purge> 刷新，
  或临时改用带 commit 哈希的不可变地址验证。
- **某些端口的服务连不上（如自建服务、游戏、PT 用了非标端口）？**
  这类流量走 `非标端口` 组，默认**直连**。若该服务必须走代理，进这个组手动切一个**支持全端口**的节点即可（很多节点只支持 80/443）。
- **节点怎么选？**
  模板不按地区分类，全部节点平铺在 `节点选择` 及各平台组里。跨境电商为每个账号在对应平台组里**钉一个固定节点**长期不变（防关联）。
- **想自己加平台 / 规则？**
  在模板里加 `ruleset=组名,[]GEOSITE,标签` 或 `ruleset=组名,clash-classic:清单URL,86400`，
  再加 `custom_proxy_group=组名\`select\`[]节点选择\`[]自动选择\`.*`。
  ⚠️ 新规则必须放在 `ruleset=节点选择,[]GEOSITE,geolocation-!cn`（境外兜底）**之前**，
  否则会被兜底规则抢先命中；并遵守 1.3 节的顺序原则。
- **规则不更新？**
  规则走 jsdelivr 缓存，最长约 24h。要立即生效可把模板里的 `testingcf.jsdelivr.net` 换成 `raw.githubusercontent.com`。
- **改了模板但订阅没变化？**
  jsdelivr 对 `@main` 有缓存；用 GitHub raw 地址，或到 <https://www.jsdelivr.com/tools/purge> 刷新缓存。
- **能直接把 .ini 填进 OpenClash 吗？**
  不能。`.ini` 是转换模板，必须经 subconverter 合成后的链接才是订阅。
