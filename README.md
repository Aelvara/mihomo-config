# mihomo-config

Clash Verge / Mihomo Party / Sparkle 的**分流 + DNS 防污染**配置集。
目标只有一个：**不泄 DNS、不挑错节点、不被假 IP 带沟里**。

> 注意区分格式：`archive/*.js` 是 Clash Verge 的**全局扩展脚本**，不是 YAML；`*.yaml` 才是覆写配置 / 内核配置。

---

## 目录

| 文件 | 类型 | 用途 |
| --- | --- | --- |
| `分流+DNS防泄露.yaml` | YAML 覆写 | **主配置（3.1）**。Clash Verge / Mihomo Party / Sparkle 通用覆写，DNS + 分流 + 节点组一把梭 |
| `Sparkle/mihomo.yaml` | 内核配置 | Sparkle 面板「内核配置」的持久化文件，含顶层 `ipv6` / `tun` / 完整 `dns` 段 |
| `Sparkle/config.yaml` | 应用设置 | Sparkle 应用层设置，关键项是 `controlDns`（决定 DNS 卡片能否管理内核 DNS） |
| `archive/` | 存档 | 3.0 旧版覆写、历代 DNS 配置、Clash Verge 扩展脚本、Mihomo Party DNS 片段，**只读留档**，详见 [`archive/README.md`](archive/README.md) |

> 本仓库由早期的 `clash-verge-script` 迁出并更名：覆盖面从"Clash Verge 脚本"扩到 Clash Verge / Mihomo Party / Sparkle 三端，旧文件全部沉到 `archive/`，根目录只保留当前在用的三份产物。

---

## 快速开始

### Clash Verge / Mihomo Party

1. 订阅 → 右键 → **编辑规则 / 全局扩展覆写配置**
2. 把 `分流+DNS防泄露.yaml` 全文粘进去，保存
3. 重载配置（或重启内核）

### Sparkle

1. **覆写** → 新建 YAML 覆写 → 粘贴 `分流+DNS防泄露.yaml` → 启用（global）
2. **设置 → DNS**：保持 `controlDns` **开启**（关闭的话 DNS 卡片会消失，且 Sparkle 会把 dns 段从持久化配置里删掉）
3. 面板 DNS 卡片填的是 `Sparkle/mihomo.yaml` 里 `dns` 段的内容；`ipv6: false` 必须在 **mihomo.yaml 顶层**也写一份（原因见下方"坑 5"）

---

## 三条核心设计

### 1. DNS：只用境外 DoH，并且强制走代理

```yaml
dns:
  respect-rules: true          # DNS 请求按规则走，而不是直连发出去
  nameserver:                  # 上游只留境外 DoH
    - https://1.1.1.1/dns-query
    - https://8.8.8.8/dns-query
    - https://dns.google/dns-query
  nameserver-policy:
    'geosite:cn,private,bytedance': [223.5.5.5, 119.29.29.29]   # 国内域名走国内 DNS
  proxy-server-nameserver: [223.5.5.5, 119.29.29.29]
  fallback: []                 # 关键：清空 fallback 链
```

为什么这么狠：

- **国内 DoH 直连解析境外域名会被投毒**。实测 `dns.alidns.com` 解析 `www.youtube.com` 返回 `185.45.5.35`（Twitter 的 IP 段），解析 `google.com` 返回 `69.171.235.22`（Facebook 段）。这类答案能 TCP 连通、能握手，但站点是错的 —— 表现出来就是"能上网但全是怪现象"。
- **订阅模板会塞 fallback 链**（`dns.alidns.com` + `doh.pub` + `cloudflare-dns.com`）。只要 fallback 生效，主 nameserver 拿到的干净答案会被污染答案顶掉。所以直接 `fallback: []`。
- `respect-rules: true` + 非空 `proxy-server-nameserver` = DoH 请求经代理出口发出，境内投毒到不了。

### 2. fake-ip：黑名单模式，白名单里绝不放 `*`

```yaml
enhanced-mode: fake-ip
fake-ip-range: 198.18.0.1/16
fake-ip-filter-mode: blacklist
fake-ip-filter:
  - '*.lan'
  - '*.local'
  - '+.push.apple.com'
  - '+.steamcontent.com'
  - '+.hltv.org'        # 需要真实 IP 的站点
  - '+.hltv.com'
```

> ⚠️ `blacklist` 模式下 `fake-ip-filter` 里出现 `*` 等于**把 fake-ip 全关掉**。订阅模板经常自带这一条，合并后会让整份 fake-ip 设计失效（判据：日志里目的地址是 `198.18.x.x` 的连接数为 0）。

### 3. 节点组：只认当前订阅真实存在的节点

- `🌐 全部节点` / `♻️ 自动选择`：`include-all-proxies: true` + `exclude-filter` 过滤掉机场塞进来的伪节点（"剩余流量：1.68 TB"、"套餐到期：长期有效"）—— 它们会被当成真节点参与测速并可能被选中，然后必然失败。
- 地区组（港/日/新/美/台）：只写 `filter`，**不写 `proxies`、不写 `use`**。
  - 订阅里有港股就自动出来，没有就是空组。
  - 空组 mihomo 会自动塞进 `COMPATIBLE`（= DIRECT 别名），UI 显示 1 个绿色节点但其实是直连 —— 这是内核行为，不是配置写错。
  - **不要用 `proxy-providers` 把别的订阅硬挂进来**：provider 的 url 只能是静态字符串，等于把订阅地址和 token 明文写死在配置文件里，换订阅就废。

---

## 为什么覆写必须搭配 `Sparkle/mihomo.yaml` 一起用

Sparkle 不是把覆写文件直接丢给内核。它在启动时把三份来源**按固定顺序叠**成一份 `work/config.yaml`，内核只认这一个文件：

```
订阅(profile) + 覆写  ──deepMerge──┐
                                   ├─→ cleanBooleanConfigs / cleanDnsConfig ──→ work/config.yaml（内核真正加载）
面板 mihomo.yaml      ──deepMerge──┘        同名键覆盖前者
```

一句话：**同名键永远是面板赢**。覆写里写得再对，合完也可能被顶掉。

| 键 | 覆写里写 | 面板里是 | 合成后的实际结果 |
| --- | --- | --- | --- |
| `ipv6` | `false` | `true` | 被顶成 `true`，再被 `cleanBooleanConfigs` 的 `if (profile.ipv6 !== false) delete profile.ipv6` 整键删除 → 内核回落默认 `true`，日志刷 `unreachable network` |
| `tun` | `stack: mixed` / `auto-route` | `stack: system` / `strict-route` / `mtu: 1500` | 整段以面板为准，覆写里的 tun 基本只是心理安慰 |
| `dns.fake-ip-filter` | 白名单，无 `*` | 若仍带 `*` | blacklist 模式下 `*` = **关掉全部 fake-ip**（判据：日志里没有目的地址为 `198.18.x.x` 的连接） |
| `dns.nameserver` | 境外 DoH | 若仍是 `doh.pub` / `dns.alidns.com` | DoH 直连被投毒，google/youtube 解析到错误 IP 段 |

还有一层：`controlDns: true` 时 Sparkle 会自动 `delete dns.fallback` 与 `dns["fallback-filter"]`（这步其实是好事，等于替你清掉订阅模板的污染链），**但前提是面板 dns 段本身干净**。面板留着旧值，覆写写得再对也会被覆盖。

所以正确姿势是**两份一起改、一起备份**：

1. **覆写**管分流 —— 规则顺序、节点组结构、fake-ip 白名单
2. **`Sparkle/mihomo.yaml`** 管内核参数 —— 顶层 `ipv6`、`tun`、`dns` 上游
3. 改完**别信任何一份源文件**，去看合成产物 `work/config.yaml` 和内核的 `GET /configs`，那才是真正生效的东西

两种典型翻车方式：

- **只拷覆写、不改 mihomo.yaml**：重启后 `ipv6: false` 消失、IPv6 直连告警刷几百条；面板里旧的 fake-ip-filter（含 `*`）或国内 DoH 继续生效，污染照旧
- **只改 mihomo.yaml、不拷覆写**：DNS 链路干净了，但 hltv 回到 403、伪节点重新混进自动选择、苹果服务指向空港股组变直连

## 最小可用配置示例（标出必须对齐的键）

图例：**✅ 必须两边一致**（不一致就出事） · **⚙️ 面板独占**（覆写里写也无效） · **🔧 可按需改**

### 1. `Sparkle/mihomo.yaml`（面板，必填）

```yaml
ipv6: false                     # ✅ 与覆写一致

tun:
  enable: true                  # ⚙️ 面板独占
  stack: system
  strict-route: true

dns:
  enable: true                  # ⚙️ 面板独占（controlDns: true 时由 DNS 卡片管理）
  ipv6: false
  respect-rules: true           # ✅ false 则 DoH 直连 → 境内被劫持
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16  # ✅ 改了它，旧假 IP 全部失配
  fake-ip-filter-mode: blacklist
  fake-ip-filter:               # ✅ 绝不能含 "*"
    - '*.lan'
    - '*.local'
    - '+.hltv.org'
  default-nameserver: [223.5.5.5, 119.29.29.29]
  nameserver:                   # ✅ 只放境外 DoH
    - https://1.1.1.1/dns-query
    - https://dns.google/dns-query
  nameserver-policy:            # ✅
    'geosite:cn,private': [223.5.5.5, 119.29.29.29]
  proxy-server-nameserver: [223.5.5.5, 119.29.29.29]   # ✅ 必须非空
  proxy-server-nameserver-policy: {}                   # ✅ 必须存在，可留空
  direct-nameserver: [223.5.5.5]
  direct-nameserver-follow-policy: false
  fallback: []                  # ✅ 清空，掐掉订阅模板的污染链

profile:
  store-selected: true
  store-fake-ip: true
```

### 2. 覆写（Clash Verge / Mihomo Party / Sparkle 通用）

```yaml
ipv6: false                     # ✅ 与面板一致（只写这里会被面板顶掉）

dns:
  enable: true
  respect-rules: true           # ✅
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16  # ✅
  fake-ip-filter-mode: blacklist
  fake-ip-filter:               # ✅
    - '*.lan'
    - '*.local'
    - '+.hltv.org'
  default-nameserver: [223.5.5.5, 119.29.29.29]
  nameserver:                   # ✅
    - https://1.1.1.1/dns-query
    - https://dns.google/dns-query
  nameserver-policy:            # ✅
    'geosite:cn,private': [223.5.5.5, 119.29.29.29]
  proxy-server-nameserver: [223.5.5.5, 119.29.29.29]   # ✅
  direct-nameserver: [223.5.5.5]
  direct-nameserver-follow-policy: false
  fallback: []                  # ✅

proxy-groups:
  - name: "🚀 默认代理"
    type: select
    proxies: ["♻️ 自动选择", DIRECT]

  - name: "♻️ 自动选择"
    type: url-test
    include-all-proxies: true
    url: "https://1.1.1.1/dns-query"   # 🔧 用 DoH 测速，筛掉"能开网页但通不了 DoH"的节点
    interval: 300
    tolerance: 50
    lazy: false

rules:
  - "DOMAIN-SUFFIX,hltv.org,DIRECT"   # 🔧 按需
  - "GEOIP,CN,DIRECT"
  - "MATCH,🚀 默认代理"
```

### 必须对齐的键，以及不对齐会怎样

| 键 | 谁说了算 | 不一致的后果 |
| --- | --- | --- |
| `ipv6` | 面板 | 面板 `true` 会把覆写 `false` 顶掉，再被整键删除 → 内核默认 `true`，IPv6 直连告警刷屏 |
| `dns.respect-rules` | 面板 | 面板缺这一项 / 为 `false` 时，DoH 请求直连发出 → 1.1.1.1、8.8.8.8 在境内被劫持，全站解析失败 |
| `dns.nameserver` | 面板 | 面板仍是 `doh.pub` / `dns.alidns.com` → 解析境外域名被投毒（youtube → Facebook 段 IP） |
| `dns.fake-ip-filter` | 面板 | 面板里留着 `*` → blacklist 模式下等于关掉全部 fake-ip |
| `dns.fake-ip-range` | 面板 | 与覆写不同 → 切换后同一域名的旧假 IP 对不上，应用需重连 |
| `dns.nameserver-policy` | 面板 | 面板缺国内分流 → 国内域名也走境外 DoH，慢且可能拿到 CDN 远端节点 |
| `dns.proxy-server-nameserver` | 面板 | 为空时 `respect-rules` 直接失效（内核会警告） |
| `tun.*` | 面板（独占） | 覆写里的 `stack` / `auto-route` / `mtu` 全被覆盖，写不写都一样 |

> 两个片段都做过离线校验：`mihomo -t` 分别对「覆写单独」与「面板片段 + 覆写组/规则合并后」返回 `test is successful`，可直接抄。

## 已知坑速查

| 症状 | 根因 | 修法 |
| --- | --- | --- |
| 整站 Cloudflare「Sorry, you have been blocked」/ 某站持续 403 | 机场共享落地 IP 被 CF 打了低信任分，触发 managed challenge | 该站改 `DIRECT`（家宽出口反而干净）：`DOMAIN-SUFFIX,example.com,DIRECT` 放 rules 最前。注意：**curl 永远过不去**（libcurl 的 JA3/JA4 指纹在挑战名单里），只能用真浏览器验证 |
| 所有域名同时解析失败 `all DNS requests failed ... EOF` | `♻️ 自动选择` 选到"能开网页但通不了 DoH"的节点 | 组测速 URL 改成 DoH 端点 `https://1.1.1.1/dns-query`，`lazy: false`，让这类节点直接出局 |
| google/youtube 解析到明显错的 IP 段 | 国内 DoH 直连被投毒 | `nameserver` 只留境外 DoH + `respect-rules: true` + `fallback: []` |
| Sparkle 面板 DNS 卡片三条输入框报红：参数不合法 | Sparkle 前端校验器只允许字母/数字/中文/空格/`-`/`_`，不允许 emoji 和方括号，所以 `https://1.1.1.1/dns-query#🚀 默认代理 [自选]` 必红 | 去掉 `#策略名` 后缀，改成靠 `respect-rules` 让 DNS 走规则（效果等价，日志里能看到 DoH 连接走代理组） |
| 每次重启后 `ipv6: false` 丢失、日志刷 `unreachable network` | Sparkle 合成配置时 `deepMerge(订阅, 面板 mihomo.yaml)`，面板里是 `ipv6: true` 就把覆写的 `false` 顶掉，随后 `cleanBooleanConfigs` 因 `true !== false` 把整个顶层 `ipv6` 删掉，内核回落默认 true | 改 `Sparkle/mihomo.yaml` 顶层的 `ipv6: false`（治本），而不是每次手动往合成配置里补 |
| fake-ip 完全不生效 | `fake-ip-filter` 里有 `*` | 删掉 |

---

## 验证方法

启动后确认这三条，基本就能判定 DNS 链路是对的：

```bash
# 1. 看内核实际生效的配置（不是看文件）
curl -s http://127.0.0.1:9090/configs | grep -E '"ipv6"|"stack"'

# 2. 看 DNS 答案是否是"真"答案：google/youtube 不应落到 Twitter/Facebook 段
curl -s "http://127.0.0.1:9090/dns/query?name=www.youtube.com&type=A"

# 3. 确认 fake-ip 在工作（应返回 198.18.x.x），白名单站点（如 hltv）应返回真实 IP
nslookup www.google.com
nslookup www.hltv.org
```

Sparkle 的 mihomo 走命名管道而不是 TCP，端点同名：`\\.\pipe\Sparkle\mihomo`，用 `curl` 不通，需用 `NamedPipeClientStream` 发 HTTP 请求。

判定"是否污染"的正确姿势是**看 IP 归属段**，不是看能不能 ping 通：
- google → `142.251.x.x` ✅ / `185.45.x.x` ❌
- youtube → `142.251.x.x` ✅ / `69.171.x.x`、`157.240.x.x` ❌
- 注意 GitHub 的 A 记录本身就是池化轮转的（`20.6.7.17`、`20.247.166.64`、`57.155.223.240` 都是真的），别把轮转当污染。

---

## 变更记录

- **2026-11-15 · 仓库迁移**
  - 仓库由 `clash-verge-script` 迁出并更名为 `mihomo-config`，覆盖面统一写成 Clash Verge / Mihomo Party / Sparkle 三端
  - 3.0 旧版覆写、历代 DNS 配置、Clash Verge 扩展脚本、Mihomo Party DNS 片段全部移入 [`archive/`](archive/README.md)，根目录只保留当前在用产物
- **2026-11-15 · 3.1**
  - 新增 `dns.respect-rules: true`，DNS 上游统一走代理出口
  - `nameserver` 精简为 3 条境外 DoH，删除 `doh.pub` / `dns.alidns.com`（直连被投毒）
  - `fallback: []`，掐掉订阅模板塞进来的混合解析链
  - `♻️ 自动选择` 测速 URL 由 `generate_204` 改为 DoH 端点，`lazy: false`
  - `🌐 全部节点` 补 `exclude-filter`，过滤"剩余流量/套餐到期"伪节点
  - `🍎 苹果服务` 移除空港股组候选，避免指向空组变直连
  - `fake-ip-filter` 补 `+.hltv.org` / `+.hltv.com` / `+.steamcontent.com` / `csgo.wmsj.cn`
  - hltv 全站改直连（CF 对机场落地 IP 持续挑战）
  - 新增 `Sparkle/` 目录：面板内核配置与应用设置留档
- 2025-10-20 移除延迟选优、故障转移分组（优化版）
- 2025-09-26 添加 TikTok 分流规则
- 2025-08-22 B 站规则默认直连，添加 `config["proxies"]` 判断
- 2025-08-19 修复 ChatGPT 规则，合并为新的 AI 代理分组
- 2025-07-21 新增规则：哔哩哔哩港澳台
- 2025-07-02 新增 Mihomo Party 覆写脚本（YAML 格式）

相关视频：<https://www.youtube.com/watch?v=sp6Iafweo7I>
