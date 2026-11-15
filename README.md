# mihomo-config

Clash Verge / Mihomo Party / Sparkle 的**分流 + DNS 防污染**配置集。
目标只有一个：**不泄 DNS、不挑错节点、不被假 IP 带沟里**。

> 注意区分格式：`archive/*.js` 是 Clash Verge 的**全局扩展脚本**，不是 YAML；`*.yaml` 才是覆写配置 / 内核配置。

---

## 目录

| 文件 | 类型 | 用途 |
| --- | --- | --- |
| `分流+DNS防泄露.yaml` | YAML 覆写 | **主配置（3.8）**。Clash Verge / Mihomo Party / Sparkle 通用覆写，DNS + 分流 + 节点组一把梭；规则已全部内联，零外部规则集依赖；境外视频、社交、音乐各一个组，视频默认走按延迟择优的测速组；国外测速站走代理、国内测速站直连；Windows 更新 / STUN / BT tracker 兜底直连 |
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

## 七条核心设计

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
- **DoH 上游的出口单独钉死**（3.6 起）：`rules` 前几条写 `IP-CIDR,1.1.1.1/32,♻️ 自动选择 [系统],no-resolve`、`IP-CIDR,8.8.8.8/32,…`、`DOMAIN-SUFFIX,dns.google,♻️ 自动选择 [系统]`。这样即使把 `🚀 默认代理` 切到别的组，DNS 这只脚也不会跟着翻 —— 之前就踩过「默认代理换到 DoH 不通的节点 → 整站 DNS 全崩」。

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

### 4. 规则：全部内联进 `rules`，不依赖任何远程规则集

早期版本挂了 9 个 `rule-providers`，全部指向 `cdn.jsdelivr.net/gh/n0de-sudo/Perfect-Rules@main/...`，`interval: 86400`。实测这 9 份规则的真实内容：

| provider | 实际内容 | 字节 |
| --- | --- | --- |
| `ads` / `ai` / `youtube` / `disney` / `apple` / `google` | 各只有一行 `GEOSITE,xxx` | 27–38 B |
| `crypto` | `GEOSITE,category-cryptocurrency` + 3 条域名 | 136 B |
| `software` | 4 条 `PROCESS-NAME-WILDCARD`（Discord / Steam / EpicGames / WhatsApp） | 207 B |
| `steam_download` | 19 条 steamcontent 域名 | 773 B |

也就是 6 个 provider 只是给本地 `geosite.dat` 套了一层 YAML 外壳，还要每天跨境下载一遍。3.2 起全部内联：

- 6 个 GEOSITE 类 → 直接写成 `GEOSITE,category-ads-all,🛑 广告拦截 [系统]` 这样的原生规则；
- `software` / `steam_download` → 展开成 `PROCESS-NAME-WILDCARD` 与 `DOMAIN-SUFFIX` 规则（`PROCESS-NAME-WILDCARD` 依赖 `find-process-mode: always`，订阅模板一般已带）；
- `rule-providers` 整段删除，`rules` 由 20 条（含 9 条 `RULE-SET`）变成 44 条直写规则。

收益是整份配置**零外部依赖**：上游删库、jsDelivr 缓存延迟、断网、换机场订阅，都不会让分流悄悄失效。代价是想加规则得改这个文件 —— 本来也是这么用的。

> 真要恢复远程规则：把 `rule-providers` 段加回来、`rules` 里改回 `RULE-SET,<provider>,<策略>`。但**别用 `@main`**，钉 commit：`https://cdn.jsdelivr.net/gh/<owner>/<repo>@<sha>/rules/xxx.yaml`。jsDelivr 对 `@main` 最长 12 小时缓存，改了不生效极容易被误判成「覆写没合并」。

### 5. 流媒体与社交平台：拆成两个组，31 + 18 个 GEOSITE，1643 条记录

演进过程：3.3 取消独立的「📺 油管专用」组 → 3.4 曾把社媒与视频站**合并**成 `📱 社媒与流媒体 [自选]` → **3.5 起又拆回两个组**，因为两者的出口诉求本来就不一样。

| 组 | 类型 | 收谁 | 默认出口 |
| --- | --- | --- | --- |
| `⚡ 流媒体测速 [系统]` | url-test | 当前订阅里名字含 高速专线 / 三网推荐 / AWS / 东京 / 新加坡 / 香港 / HK / SG / JP / TW / Taipei 的节点（实测 10 个） | 按 `gstatic.com/generate_204` 延迟择优 |
| `🎬 流媒体 [自选]` | select | **31 个视频 GEOSITE**（含 tiktok） | 指向 `⚡ 流媒体测速` |
| `📱 社交平台 [自选]` | select | **18 个社媒 GEOSITE** | `🇺🇸 美国节点 [系统]` |

`⚡ 流媒体测速 [系统]` 在 `proxy-groups` 里紧跟在 `♻️ 自动选择 [系统]` 之后，UI 上就排在它下面 —— 两者都是 "按延迟挑节点" 的 url-test 组，放一起便于对照和手动切换。

**为什么拆**：视频要的是带宽和延迟 —— 同一个订阅下实测 `♻️ 自动选择` 196ms（新加坡）、`🇯🇵 日本` 387ms、`🇺🇸 美国` **1820ms**，用美国落地拉 4K 会明显变慢；社交要的是**落地稳定**，账号地区不能来回漂，否则容易触发二次验证。3.4 的「一个组一个出口」只能二选一，3.5 让两者各走各的。

**为什么测速组单独建，不直接用 `♻️ 自动选择`**：`♻️ 自动选择` 的探测地址是 `https://1.1.1.1/dns-query`（3.0 起为了让 DNS 有可用上游而改的），量的是「DoH 通不通」，完全不代表带宽。测速组用 `gstatic.com/generate_204` 单独量延迟。代价是多一个组要维护。

#### 视频侧（31 个分类，794 条）

| 分类 | 记录 | 覆盖 |
| --- | --- | --- |
| `disney` | 225 | Disney+ / Hulu / Star-Hotstar / ESPN / Marvel / Pixar / Lucasfilm / NatGeo / FX / ABC / 20th Century |
| `youtube` | 178 | YouTube 全站（含 `googlevideo` 以外的 ytimg / yt3 等） |
| `hbo` | 65 | HBO Max（`max.com`、`hbomax.com`、`hbo.com`） |
| `hulu` | 48 | Hulu（与 disney 分类有重叠，主站/日本站） |
| `tiktok` | 37 | TikTok（`tiktokv` / `ttwstatic` / 各地 `tiktokcdn`）—— 3.5 从社媒侧移入 |
| `twitch` | 34 | Twitch（含 `ttvnw.net`、`live-video.net` 播放域） |
| `netflix` | 28 | Netflix（`nflxvideo` / `nflxso` / `fast.com` + aws apiproxy 正则） |
| `primevideo` | 23 | Prime Video（含 amazon 图片 CDN） |
| `abema` | 21 | ABEMA（日） |
| `hamivideo` | 19 | Hami Video（台） |
| `vimeo` | 17 | Vimeo |
| `tvb` | 14 | TVB（港） |
| `litv` | 11 | LiTV（台） |
| `niconico` | 10 | niconico（日） |
| `apple-tvplus` | 8 | Apple TV+（只截流媒体，不影响 `🍎 苹果服务`） |
| `dazn` | 8 | DAZN |
| `viu` | 7 | Viu（港/新/东南亚） |
| `roku` | 6 | Roku |
| `tver` / `sling` | 5 / 5 | TVer（日）/ Sling TV |
| `plex` | 4 | Plex |
| `dailymotion` / `kktv` / `ftv` | 3 / 3 / 3 | Dailymotion / KKTV（台）/ 民视（台） |
| `mubi` / `streamable` / `plutotv` / `rumble` / `7tv` | 各 2 | 小众点播与直播站 |
| `mytvsuper` / `kick` | 各 1 | myTV SUPER（港）/ Kick |

合计 **794 条记录**（`disney` 与 `hulu` 有重叠，实际域名数略少）。规则顺序在 `GEOSITE,apple` 之前，所以 `apple-tvplus` 不会被苹果组抢走。

> TikTok 归视频侧而不是社媒侧：它的主要负载是视频流，选最快落地比"地区稳定"更重要。

#### 社交侧（18 个分类，849 条）

| 分类 | 记录 | 覆盖 |
| --- | --- | --- |
| `meta` | 555 | Facebook / Instagram / Threads / WhatsApp 网页版（含 `cdninstagram`、`fbcdn` 静态域） |
| `kakao` | 57 | KakaoTalk / Daum |
| `pinterest` | 52 | Pinterest（含 `pinimg` 图片 CDN） |
| `vk` | 52 | VK（俄） |
| `discord` | 28 | Discord 网页版（`discord.gg` / `discord.media` / 附件上传桶） |
| `twitter` | 24 | X / Twitter（`x.com`、`t.co`、`twimg`、`pscp.tv`） |
| `line` | 20 | LINE |
| `reddit` | 12 | Reddit（`redd.it` / `redditmedia` / `redditstatic`） |
| `linkedin` | 12 | LinkedIn（含 `licdn` / `lnkd.in`） |
| `snap` | 9 | Snapchat（`sc-cdn` / `snapchat.com`） |
| `signal` | 8 | Signal |
| `quora` | 6 | Quora（其中 `poe.com` 已被前面的 AI 规则先命中，不会重复） |
| `messenger` | 4 | Facebook Messenger（`messenger.com` / `m.me`） |
| `imgur` / `4chan` / `bluesky` / `clubhouse` | 各 3 | Imgur / 4chan / Bluesky（`bsky.app`）/ Clubhouse |
| `tumblr` | 1 | Tumblr（本地 geosite 只给了 `tumblr.com` 一条） |

合计 **849 条记录**。3.8 起 `PROCESS-NAME-WILDCARD,*Discord*` / `*Steam*` / `*EpicGames*` / `*WhatsApp*` 已移到 `MATCH` 之前 —— 域名规则一律先判，这四个进程只兜住没有域名的裸 IP 流量（放在域名规则之前时，Discord / Steam 内嵌浏览器打开的 youtube、github 会被一并拽去 `🖥️ 软件分流`）。

#### 测速组实测成员与延迟

`⚡ 流媒体测速 [系统]`（`type: url-test`、`url: https://www.gstatic.com/generate_204`、`interval: 600`、`tolerance: 100`、`lazy: false`）在某三毛机场订阅下的实测结果：

| 节点 | 延迟 |
| --- | --- |
| 新加坡1\|高速下载\|移动优化 | **209 ms** ← 被选中 |
| 新加坡2\|高速下载\|移动优化 | 512 ms |
| 2x专线-香港-2 | 752 ms |
| 2x专线-新加坡-2 | 786 ms |
| 2x专线-香港-1 | 809 ms |
| 2x专线-香港-3 | 821 ms |
| 2x专线-新加坡-1 | 951 ms |
| 2x专线-新加坡-3 | 997 ms |
| 2x专线-新加坡-4 (IPv6) / 2x专线-香港-4 (IPv6) | 0（探测超时，不参与择优） |

`lazy: false` 是关键：默认的 `lazy: true` 只在有流量走进该组时才测速，第一次播放视频会先用一个随机初始节点。关掉后启动即测。

> 延迟 ≠ 带宽。测速组只做粗筛（挑出「能连通且延迟低」的 1 个），真正的带宽差异要靠实际播放感知。想要更准可以给组加 `expected: "200"`，或换成一个较大的测速文件 URL。

#### 故意没放进去的

- **BBC iPlayer**（`GEOSITE,bbc`，28 条）—— 需要英国落地 IP，本配置的组里没有英国节点，进来也播不了，还会把 `bbc.com` 新闻一起带走
- **Spotify / Tidal / Deezer / SoundCloud / Apple Music**（音频）—— 3.6 起独立成 `🎵 音乐 [自选]`，不再塞进视频组
- **Bilibili / 爱奇艺 / 优酷 / 腾讯视频 / CCTV** —— 国内站，直接命中 `GEOSITE,CN,DIRECT`
- **Netflix 之外的 Netflix 类**（`primevideo` 里的 `amazon.com` 主站购物域）—— 分类本身不含主站，购物不受影响
- **Telegram**（`GEOSITE,telegram`，21 条）—— 继续留在独立的 `💬 电报专用 [自选]` 组，它有自己的落地偏好（部分机房 IP 会被 Telegram 风控）
- **`bytedance`**（955 条）—— 是 TikTok + 抖音 + 今日头条 + 番茄小说等国内 App 的超集，并进来会把国内应用一起推出境
- **微信 / 微博 / 小红书 / 抖音** —— 国内站，命中 `GEOSITE,CN,DIRECT`，本就不出境
- **`mastodon` / `9gag`** —— 本地 `geosite.dat` 里没有这两个分类，写进去会 `Parse config error` 直接起不来（实测已确认不存在）

> 想加别的站：写 `GEOSITE,<小写分类名>,🎬 流媒体 [自选]` 或 `GEOSITE,<小写分类名>,📱 社交平台 [自选]`。分类名必须是本地 `geosite.dat` 里真实存在的（`strings work/geosite.dat | grep ...` 或解 protobuf 都行），写错会 `Parse config error` 直接起不来。


---


### 6. 音乐：视频、社交之外的第三个内容组

| 分类 | 记录 | 覆盖 |
| --- | --- | --- |
| `spotify` | 28 | Spotify（含 `audio-ak-spotify-com.akamaized.net` 这类 Akamai 音频域） |
| `apple-music` | 18 | Apple Music |
| `soundcloud` | 3 | SoundCloud |
| `tidal` | 3 | TIDAL |
| `deezer` | 2 | Deezer |

合计 5 个分类、54 条域名，全部指向 `🎵 音乐 [自选]`（默认 `♻️ 自动选择`，候选另有 日 / 美 / 新 / 全部节点 / DIRECT）。

**顺序很关键**：`GEOSITE,apple-music` 必须写在 `GEOSITE,apple` **之前**，否则 Apple Music 先被 `🍎 苹果服务` 吃掉。3.5 的 `apple-tvplus` 同理，也是放在 `apple` 之前的。

### 7. 兜底直连：测速站 / Windows 更新 / STUN / BT tracker / 杀软更新

3.6 之前这四类全部落到 `MATCH` → `🚀 默认代理` → 机场出口。代价不显眼，但一直在付：

| 分类 | 记录 | 走代理的后果 | 3.6 起 |
| --- | --- | --- | --- |
| `category-speedtest`（内含 `include:ookla-speedtest`） | 77 | 测出来的永远是机场带宽，家宽再快也测不出 | `DIRECT` |
| `win-update` + `win-extra` | 364 + 355 | Windows 更新一次几百 MB，全算机场流量 | `DIRECT` |
| `category-stun` | 359 | STUN 走代理会打断 NAT 穿透，语音 / 联机直接哑 | `DIRECT` |
| `category-public-tracker` | 437 | BT tracker 走机场，易被限速或标记 | `DIRECT` |
| `kaspersky`（3.8 新增） | 132 | 特征库 / 大版本更新动辄几百 MB，全算机场流量 | `DIRECT` |

这 6 条写在 Steam 下载域名之后。3.8 起进程规则已挪到 `MATCH` 之前，域名规则天然先命中，不再需要靠顺序抢跑。

#### 国内外测速站必须分开（3.7）

3.6 把 `category-speedtest` 整个塞进 `DIRECT`，等于默认「你要测的是家宽」。但 fast.com、speedtest.net 这类站点的测速端点全在境外：直连时测到的是「家宽 → 国际出口」，和日常浏览的「家宽 → 机场落地 → 目标」是两条完全不同的路径，数字对你调节点没有参考价值。实测直连 fast.com：首字节 3.5s，Ookla 还把测速服务器选到了越南节点。

| 用途 | 该走哪条 | 3.7 规则 |
| --- | --- | --- |
| 测家宽真实带宽 | 直连 | `speedtest.cn` / `cnspeedtest.cn` → `DIRECT` |
| 测机场（代理）带宽 | 代理 | `fast.com` / `speedtest.net` / `ooklaserver.net` / `librespeed.org` / `nperf.com` / `openspeedtest.com` / `speedof.me` / `testmy.net` / `speed.cloudflare.com` → `🚀 默认代理` |

`api.fast.com` 被 `DOMAIN-SUFFIX,fast.com` 覆盖；`ooklaserver.net` 是 Ookla 各节点测速服务器的统一后缀，走代理后服务器列表会按代理落地就近选（实测落到新加坡 / 印尼节点），测出来的就是该节点的带宽。想横向比节点，把 `🚀 默认代理` 临时切到目标组再测一次即可。

`category-speedtest`（77 条，源头本身已 `include:ookla-speedtest`，3.8 删掉了重复的 `GEOSITE,speedtest`）仍整体 `DIRECT`，作用是兜住国内长尾测速站（各省运营商、宽带厂商自建测速页）；代价是其中少量国外域名也会直连，想彻底干净就删掉这条 `GEOSITE,category-speedtest`，只留上面 11 条域名级规则（9 条国外走代理 + 2 条国内直连）。


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
| 改过组名后，某个组突然不走原来的节点（例如流媒体回落 `♻️ 自动选择`、甚至直连） | Sparkle 按**组名**存选择状态，组名一变旧选择就失效，组回落到第一个候选；空组更会填 `COMPATIBLE`（=DIRECT） | 重载后手动点一次该组出口；改名前先记下当前节点 |
| 测出来的速度总贴着机场峰值，家宽明明更快 | 测速站自己也走了代理 | 3.6 起 `category-speedtest` / `speedtest` 已 `DIRECT`；自己加的测速域也记得放 `DIRECT` |
| 语音 / 联机 / BT 全都不通或极慢 | STUN、tracker 走代理，NAT 穿透与 UDP 回包被打断 | 3.6 起 `category-stun` / `category-public-tracker` 已 `DIRECT` |
| 热加载 `PUT /configs` 后某个 select 组自己换了节点 | select 的选中值不保证保留，会回落到第一个候选 | 想让出口固定，就把它写在该组 `proxies` 的第一位，别靠 UI 点选 |
| `url-test` 组第一次用总是走一个奇怪的节点 | 默认 `lazy: true`（有流量才测速） | 加 `lazy: false`，启动即测 |
| 规则集改了 24 小时还不生效 / 某类站点忽然不再分流 | ① jsDelivr 对 `@main` 最长 12h 缓存；② provider 上游删库或改名后，内核拉取失败会继续用 `work/rules/<md5>` 旧缓存，日志刷 `initial provider xxx error`，界面没有任何提示 | 规则内联（3.2 已全部内联）；必须用远程规则就钉 commit，并盯内核日志 |

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

- **2026-11-15 · 3.8**
  - 进程规则后移：4 条 `PROCESS-NAME-WILDCARD`（Discord / Steam / EpicGames / WhatsApp）从域名规则之前移到 `MATCH` 之前 —— 原位置会截走这四个进程访问的**所有**域名（含内嵌浏览器打开的 youtube / github），只兜裸 IP 流量才是本意
  - 删除冗余 `GEOSITE,speedtest`（14 条）：源头只有 `include:ookla-speedtest` 一行，已被 `category-speedtest` 完全包含
  - 新增 `GEOSITE,geolocation-!cn`（27431 条）→ `🚀 默认代理`，位置在各类内容组之后、`GEOSITE,CN` 之前，语义显式化
  - 新增 `GEOSITE,kaspersky`（132 条）→ `DIRECT`（实测 `p13.upd3.kaspersky.com` 原先落 `MATCH` 走机场）
  - 新增 `DOMAIN-SUFFIX,cline.bot` → `🤖 AI服务`（实测 `api.cline.bot` 原先落 `MATCH`）
  - `rules` 116 条 → **118 条**；离线 `mihomo -t` 通过；热加载后日志实测：`p13.upd3.kaspersky.com` → `GeoSite(kaspersky) DIRECT`、`api.cline.bot` → `DomainSuffix/cline.bot 🤖 AI服务`、`linux.do` / `www.yahoo.com` → `GeoSite(geolocation-!cn) 🚀 默认代理`
  - `GEOIP,CN,DIRECT,no-resolve` → **`GEOIP,CN,DIRECT`**：fake-ip 下 `no-resolve` 使域名目的地址永远匹配不上（只对裸 IP 生效），未收录进 `geosite:CN` 的国内站会被送进隧道；去掉后用 `hosts` 定点验证——域名指向 `114.114.114.114` 时日志为 `match GeoIP/cn … using DIRECT`，改前同类请求全部落 `MATCH`；代价是走到该规则的尾部连接各多一次 DNS 解析（有缓存）
- **2026-11-15 · 3.7**
  - 测速站按地理拆分：9 条国外测速站（`fast.com` / `speedtest.net` / `ooklaserver.net` / `librespeed.org` / `nperf.com` / `openspeedtest.com` / `speedof.me` / `testmy.net` / `speed.cloudflare.com`）→ `🚀 默认代理`，2 条国内测速站（`speedtest.cn` / `cnspeedtest.cn`）→ `DIRECT`
  - 动因：直连 fast.com 实测首字节 3.5s、Ookla 选到越南节点——直连测的是家宽国际出口，与日常代理路径无关
  - `rules` 105 条 → **116 条**；`proxy-groups` 仍 21 组；离线 `mihomo -t` 通过；热加载后实测 fast.com / speedtest.net / librespeed.org → `🚀 默认代理`，`www.speedtest.cn` → `DIRECT`；浏览器实测 fast.com 正常出数（24 Mbps）
- **2026-11-15 · 3.6**
  - 新增 `🎵 音乐 [自选]`（select，默认 `♻️ 自动选择`），收 `spotify`(28) / `apple-music`(18) / `soundcloud`(3) / `tidal`(3) / `deezer`(2)，共 54 条；`apple-music` 排在 `GEOSITE,apple` 之前
  - 新增 6 条兜底直连：`category-speedtest`(77) / `speedtest`(14) / `win-update`(364) / `win-extra`(355) / `category-stun`(359) / `category-public-tracker`(437)，共 1074 条，位置在 Steam 域名之后、`PROCESS-NAME-WILDCARD` 之前
  - `GEOSITE,github` 显式化（原先靠 `microsoft` 分类顺带命中，日志里显示的是 `GeoSite(microsoft)`）
  - DoH 上游出口钉死：`IP-CIDR,1.1.1.1/32` / `IP-CIDR,8.8.8.8/32` / `DOMAIN-SUFFIX,dns.google` → `♻️ 自动选择 [系统]`，避免默认代理换组时 DNS 跟着崩
  - `🚀 默认代理` 候选列表新增 `⚡ 流媒体测速 [系统]`（排在 `♻️ 自动选择` 之后，一键可切；默认仍留 `♻️ 自动选择`，它 300s 复测比测速组的 600s 恢复更快）
  - `rules` 90 条 → **105 条**，组数 20 → **21**；离线 `mihomo -t` 通过；热加载后实测：spotify / apple-music → `🎵 音乐 [自选]`，speedtest / Windows 更新 → `DIRECT`，github → `Ⓜ️ 微软服务`，DoH 连接 → `♻️ 自动选择`

- **2026-11-15 · 3.6 微调**
  - `🎵 音乐 [自选]` 整块移到 `💰 币圈专用 [自选]` 之后（`proxy-groups` 顺序即 UI 顺序）；组内候选与 105 条规则不变
- **2026-11-15 · 3.5**
  - 又拆回两个组：`📱 社媒与流媒体 [自选]` → `🎬 流媒体 [自选]`（31 个视频 GEOSITE）与 `📱 社交平台 [自选]`（18 个社媒 GEOSITE）；`rules` 仍 90 条，组数 19 → 20
  - 新增 `⚡ 流媒体测速 [系统]`（`type: url-test`、`url: https://www.gstatic.com/generate_204`、`interval: 600`、`tolerance: 100`、`lazy: false`），filter 收 高速专线 / 三网推荐 / AWS / 东京 / 新加坡 / 香港 / HK / SG / JP / TW / Taipei，实测 10 个候选，选中 `新加坡1|高速下载|移动优化`（209 ms，同组最慢的 2x专线-新加坡-3 为 997 ms）
  - `🎬 流媒体` 默认指向 `⚡ 流媒体测速`；`📱 社交平台` 默认 `🇺🇸 美国节点 [系统]`（美国-HY2-55，201 ms）
  - `GEOSITE,tiktok` 从社媒侧移到视频侧（TikTok 主要负载是视频流）；Pinterest / Imgur 留在社媒侧不动
- **2026-11-15 · 3.5 微调**
  - `⚡ 流媒体测速` 的 `interval` 由 `300` 改为 `600`（减少每 5 分钟 × 10 个成员的测速开销）
  - `⚡ 流媒体测速` 整块移到 `♻️ 自动选择 [系统]` 之后（`proxy-groups` 顺序即 UI 顺序）
  - 实测：浏览器打开 youtube / netflix / tiktok / twitch 全部走 `🎬 流媒体 [自选][新加坡1|高速下载|移动优化]`，instagram / x.com / pinterest 全部走 `📱 社交平台 [自选][美国-HY2-55]`，重载后落回 `MATCH` 的媒体/社媒域名 0 条
- **2026-11-15 · 3.4**
  - 组名 `🎬 流媒体 [自选]` → **`📱 社媒与流媒体 [自选]`**，境外主流社交平台并入同一组；`rules` 71 条 → **90 条**（组数仍 19）
  - 并入 19 个社交分类：`meta`(555) / `kakao`(57) / `pinterest`(52) / `vk`(52) / `tiktok`(37) / `discord`(28) / `twitter`(24) / `line`(20) / `reddit`(12) / `linkedin`(12) / `snap`(9) / `signal`(8) / `quora`(6) / `messenger`(4) / `imgur`·`4chan`·`bluesky`·`clubhouse`(各 3) / `tumblr`(1)，共 886 条记录
  - 保留独立组：`GEOSITE,telegram` → `💬 电报专用 [自选]`；排除 `bytedance`（955 条，含抖音/今日头条等国内 App）
  - 改名副作用：Sparkle 按组名存选择状态，组名一变就回落到首个候选 `♻️ 自动选择`，重载后要再点一次出口（本机已设回 `🇺🇸 美国节点 [系统]`）
  - 实测：浏览器打开 x.com / instagram.com / facebook.com / reddit.com / discord.com / tiktok.com / pinterest.com 全部正常，日志统一 `match GeoSite(meta|tiktok|discord|reddit|pinterest) using 📱 社媒与流媒体 [自选]`；重载后社媒域名落回 `MATCH` 的行数 = 0
- **2026-11-15 · 3.3**
  - `rules` 44 条 → 71 条：`🎬 流媒体` 由 netflix/disney 两条扩为 **30 个境外视频 GEOSITE**（hbo / hulu / primevideo / youtube / twitch / apple-tvplus / dazn / viu / abema / niconico / tver / tvb / mytvsuper / hamivideo / kktv / litv / ftv / vimeo / dailymotion / plex / mubi / streamable / roku / sling / plutotv / kick / rumble / 7tv），共 757 条记录
  - 删除独立的 `📺 油管专用 [自选]` 组，YouTube 并入流媒体组（组数 20 → 19）
  - 媒体规则位于 `GEOSITE,apple` 之前，避免 `apple-tvplus` 被苹果组截走
  - 实测日志：`twitch` / `youtube` / `hbo`（max.com）/ `primevideo` / `viu` / `abema` 全部命中 `🎬 流媒体 [自选]`，无一条落回 `MATCH`
- **2026-11-15 · 3.2**
  - 删除全部 9 个 `rule-providers`，规则内联进 `rules`（20 条 → 44 条直写规则，0 条 `RULE-SET`）
  - 6 个 GEOSITE 包装型 provider 还原为原生 `GEOSITE,xxx,<策略>`（`category-ads-all` / `category-ai-!cn` / `youtube` / `category-cryptocurrency` / `disney` / `apple` / `google`）
  - `software` 的 4 条进程规则、`steam_download` 的 19 条域名规则展开内联
  - 新增 `DOMAIN-SUFFIX,ssh.github.com,DIRECT`（机场共享出口对 22/443 常 RST）
  - 离线 `mihomo -t` 通过，geosite 记录数校验一致（ads 911 / apple 1792 / google 1075 / disney 225 / youtube 178 / crypto 235 / ai 182）
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
