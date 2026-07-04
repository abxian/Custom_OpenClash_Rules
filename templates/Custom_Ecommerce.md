# 跨境电商防泄漏模板 · 订阅转换调用说明

> 本文只讲**这一个模板 `Custom_Ecommerce.ini` 怎么用**。
> 方案原理（防 DNS/IPv6/WebRTC 泄漏、防关联）见同目录 [`README.md`](./README.md)。

---

## 一、模板是什么

`Custom_Ecommerce.ini` 是一个 **subconverter（订阅转换）配置模板**，它本身**不是订阅链接**，不能直接填进 OpenClash。

它的作用：把「你的机场 / 专线订阅」里的裸节点，**自动套上**这套跨境电商结构 ——

- 防泄漏：Fake-IP DNS、关 IPv6、拦截 WebRTC(STUN)、FINAL 走代理
- 平台分组：📘Facebook / 📷Instagram / 🐦Twitter / 🎵TikTok / 📲Telegram / 🤖AI服务 / 🛒国际电商 / 🎥流媒体
- 分流规则：全部引用 [`abxian/stash-override`](https://github.com/abxian/stash-override) 的 `rule/*.list`，随仓库自动更新

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

- **地区组是空的 / 没节点？**
  模板用节点名里的关键词（HK、日本、SG…）自动归类。若你的机场节点命名不含这些关键词，地区组会为空——直接用 `🚀 手动选择` 里的全部节点即可，不影响使用。
- **规则不更新？**
  规则走 jsdelivr 缓存，最长约 24h。要立即生效可把模板里的 `testingcf.jsdelivr.net` 换成 `raw.githubusercontent.com`。
- **改了模板但订阅没变化？**
  jsdelivr 对 `@main` 有缓存；用 GitHub raw 地址，或到 <https://www.jsdelivr.com/tools/purge> 刷新缓存。
- **能直接把 .ini 填进 OpenClash 吗？**
  不能。`.ini` 是转换模板，必须经 subconverter 合成后的链接才是订阅。
