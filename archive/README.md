# archive · 历史配置存档

这里放的是本仓库更名/重构**之前**各代配置文件，**只读留档，不再维护**。
当前在用的产物只剩三份，在仓库根目录与 `Sparkle/`：`分流+DNS防泄露.yaml`（3.1）、`Sparkle/mihomo.yaml`、`Sparkle/config.yaml`。

为什么要归档：这批老文件是**给 Clash Verge 全局扩展脚本用的** —— 靠 JS 在运行时注入 `dns` 段和 `proxy-groups`。Sparkle / Mihomo Party 没有这个机制，而且即便在 Clash Verge 里，运行时注入的 `dns` 也会被客户端合成逻辑覆盖（见根目录 README「为什么覆写必须搭配 Sparkle/mihomo.yaml」）。3.1 起统一改成**纯 YAML 覆写 + 面板内核配置**，老脚本全部退役。

## 文件清单

| 文件 | 类型 | 内容 / 当时的思路 | 退役原因 |
| --- | --- | --- | --- |
| `DNS防泄漏` | Clash Verge 扩展脚本（无扩展名） | 最早的最小版：只注入 `dns` 段，国内用 `223.5.5.5` / `doh.pub` DoH | 只防 DNS，不防分流；国内 DoH 直连解析境外域名会被投毒 |
| `DNS防泄漏（完整版）` | 扩展脚本 | 在最小版基础上补齐国内外两组 DoH 与 `fallback` 链 | `fallback` 链会被订阅模板顶回来，且污染答案可能胜出 |
| `DNS防泄漏（系统代理）` | 扩展脚本 | 面向「系统代理」模式，带 `ruleProviderCommon` 规则集定义 | 只适合系统代理，TUN 场景不适用 |
| `DNS防泄漏（终极版）` | 扩展脚本 | 国外 DoH 加 `#RULES` 后缀，强制这部分解析走代理分流 | 依赖 Clash Verge 的 `#RULES` 语义；Sparkle 前端不接受带 `#` 的策略后缀写法 |
| `DNS防泄露（全能版）` | 扩展脚本 | `ruleProviderCommon` + 全套规则集注入 | 同上，注入式写法在新客户端失效 |
| `DNS防泄露（精简版）` | 扩展脚本 | 国内阿里/腾讯 DoH + 国外 OpenDNS 等，代码最短 | 精简到没有 `respect-rules`，DoH 仍直连 |
| `优化版DNS防泄漏` | 扩展脚本 | 精简版的可读性整理版 | 同上 |
| `扩展脚本（原始）` | 扩展脚本 | 最早一版完整脚本，含 DNS 注入 + 分组 | 已被后两版取代 |
| `扩展脚本-优化版` | 扩展脚本 | 在原始版上整理参数与注释 | 已被 3.1 的 YAML 覆写取代 |
| `扩展脚本-地区分组` | 扩展脚本 | 在优化版上按地区自动分组（港/日/新/美/台） | 分组逻辑已搬进 YAML 覆写的 `proxy-groups` |
| `mihomo party.yaml` | YAML 片段 | Mihomo Party 的 `dns` 段参考，含 `respect-rules: true`、`listen: 0.0.0.0:1053` | 片段式写法已被 3.1 的完整覆写吸收 |
| `分流+DNS防泄露-3.0.yaml` | YAML 覆写（439 行） | 3.1 之前的覆写版本：`nameserver` 仍含 `doh.pub` / `dns.alidns.com`，无 `respect-rules`，`♻️ 自动选择` 用 `generate_204` 测速 | 3.1 已修正 DNS 上游与测速 URL，见根目录 README「变更记录」 |

## 取用建议

- 想看「3.1 到底改了什么」：拿 `分流+DNS防泄露-3.0.yaml` 和根目录的同名 3.1 版直接 diff，差异集中在 `dns` 段与 `proxy-groups` 的测速 URL。
- 想在 Clash Verge 里继续用 JS 扩展脚本：上面的脚本能跑，但要接受 `doh.pub` / `dns.alidns.com` 直连被投毒的事实（判据：`youtube` 解析到 `69.171.x.x`、`google` 解析到 `185.45.x.x`）。
- 文件没有扩展名的那几个，内容是 JavaScript，粘贴到 Clash Verge 的「全局扩展脚本」编辑器里即可。
