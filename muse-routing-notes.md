# Muse 分流核查记录

核查日期：2026-09-25（北京时间）。目标为 Meta 的 Muse 个人 AI 助手。

本次只扩充 `proxy.list`。原有 `photomath.net`、`meta.ai` 保留；三个客户端 YAML、AI 和 Meta 的上游规则集均未改动。新增 30 条域名规则及 106 条实测单地址 IP 规则（65 个 IPv4、41 个 IPv6）。

## 来源与方法

- [Meta 官方发布公告](https://about.fb.com/news/2026/09/introducing-muse-personal-ai-agent/)确认产品入口为 `muse.ai`。
- 读取 [Muse 官网](https://muse.ai/)和[连接器平台页面](https://muse.ai/platform)的 HTML、Content-Security-Policy（CSP）响应头，以及页面引用的 53 个 JavaScript 文件和其中继续引用的 15 个文件，共 68 个文件。只读取公开资源，未登录或调用用户账户功能。
- [官网域名配置代码](https://muse.ai/_next/static/chunks/0wmry3cxshu6j.js)列出 Muse、MuseAI、Ecto1、Hatch 等相关入口。[内容域名代码](https://muse.ai/_next/static/chunks/1o7_gw26ikm2r.js)包含用户内容域名。[OHTTP 代码](https://muse.ai/_next/static/chunks/3egig7fqv9rjl.js)列出两个 Fastly 中继相关主机。[官网视频代码](https://muse.ai/_next/static/chunks/44a0104fct17j.js)直接请求 `api.muse.ai`。这些程序文件名可能随网站发布变化。
- 分页读取 [Cert Spotter 的 Muse 证书记录](https://api.certspotter.com/v1/issuances?domain=muse.ai&include_subdomains=true&expand=dns_names)，获得 279 条记录，含 `*.muse.ai` 和 `*.preview.muse.ai`。证书只证明名称曾出现在证书中，不证明服务当前可用；也不能列举通配符下所有主机。
- 使用 Google 和 Cloudflare 的 DNS-over-HTTPS 查询 A/AAAA，并记录 CNAME、TTL 和返回地址。生效范围对应 204 次查询；去重结果及各 IP 的来源域名见 `muse-ip-observations.json`。

## 纳入范围

| 范围 | 域名或规则 | 依据与用途 |
| --- | --- | --- |
| 当前入口 | `muse.ai` 后缀 | 覆盖根域及全部子域，包括 `auth`、`api`、`security`、`introducing`、`preview`；无需逐个列出 |
| 已保留的 AI 端点 | `meta.ai` 后缀 | 已有规则也覆盖 `agent`、`hatch`、`ecto1`、`gateway`、`edge-chat`、`edge-chat-latest`、`shortwave` 等端点；这些名称来自代码或 CSP，不代表每个都正在使用 |
| 关联旧入口/内部入口 | `museai.com`、`ecto1.ai`、`hatch.club` 后缀 | 官网代码列出的 origin；`production.museai.com`、`voice.museai.com`、`hatch.club`、`ecto1.ai` 实测转入 Facebook 员工 SSO；保留相关覆盖，不代表普通用户必须访问 |
| 用户内容及应用 | `metaaiusercontent.com`、`ecto1usercontent.com`、`meta-agents-apps.workers.dev` 后缀 | 内容处理代码或 CSP 明确列出；后缀覆盖用户动态子域。根域无 A/AAAA 不等于子域不可用 |
| OHTTP | `meta-ohttp-config-prod.fastly-edge.com`、`meta-ohttp-relay-prod.fastly-edge.com` | 程序中的配置与中继端点；配置文件可公开读取 |
| 登录、恢复、帮助 | `auth.meta.com` 后缀；`www.facebook.com`、`web.facebook.com`、`www.instagram.com`、`www.meta.com`、`fb.okta.com` | 代码、CSP、相关入口的重定向。主机与其他 Meta 产品共用 |
| 动态媒体 | `fbcdn.net`、`fbsbx.com`、`cdninstagram.com`、`cdn.whatsapp.net`、`oculuscdn.com` 后缀 | Muse CSP 允许的媒体/图片来源；属于条件依赖，不声称每次打开页面都请求。会覆盖其他 Meta 产品使用同类 CDN 的流量 |
| 付款 | `js.stripe.com`、`api.stripe.com`、`hooks.stripe.com`、`m.stripe.network`、`link.com` 后缀 | 代码或 CSP 中的可选付款依赖；为其他商户共享 |
| 字体 | `fonts.googleapis.com`、`fonts.gstatic.com` | CSP 允许 Google Fonts 样式；`fonts.gstatic.com` 作为其字体文件配套域名纳入，属于配套推断而非本次页面直接请求的证据 |
| 诊断及统计 | 专用 Sentry ingest 主机、`connect.facebook.net`、`va.vercel-scripts.com` | 官网代码/CSP 引用，属于可选或共享依赖 |

未因名称相似就添加旧 Muse 视频平台（现为 Skiv）、Microsoft Muse、My Muse、MuseDAM 等其他产品；也未把框架文档、示例域名、商店下载链接、任意外链，或者整个 Facebook/Instagram/Meta/Cloudflare/Fastly 域名集合复制进来。CSP 是允许访问的范围，不是实际请求日志；已区分条件依赖与代码中的明确端点。

## IP 规则的明确边界

用户已明确选择启用共享 IP 的单地址规则，并接受同 IP 的其他流量也可能进入 Parco Proxy。因此把本次返回的全球可路由 IPv4 写为 `/32`、IPv6 写为 `/128`，全部加 `no-resolve`；没有扩大为 ASN 或整个网段。

这些 IP 包括 Meta、Vercel、Fastly、Stripe、Google、Sentry 等共享地址，不能视为 Muse 专用地址。`no-resolve` 仅避免该规则额外触发 DNS；连接本身已有目的 IP 时仍然可能命中。Parco Proxy 规则排在客户端的 AI、Meta 等规则之前，因此共享域名或 IP 命中的流量会优先使用 Parco Proxy；AI/Meta 规则集文件本身保持原样。

IP 是当时两个解析器视角的快照，可能因地区、TTL、调度和服务更新变化；不代表全球所有 IP，不能穷尽未来 IP。更新规则集不会自动重新解析这些 IP，需要后续重新核查。后缀域名规则则能覆盖该后缀下未来新增的子域。

## 覆盖及验证边界

已覆盖本次公开网站、可达的程序文件引用、CSP 和证书查询中能够确认的相关域名范围。未获取登录后流量、iOS/Android 原生应用流量或地区/账户专属后端，因此不能宣称绝对穷尽所有 Muse 连接。

新增语法为 `DOMAIN`、`DOMAIN-SUFFIX`、`IP-CIDR`、`IP-CIDR6`，以 `classical` 文本规则集供现有三个客户端配置读取。语法依据：[Mihomo 路由规则](https://wiki.metacubex.one/config/rules/)及 [Stash 规则类型](https://stash.wiki/rules/rule-types)。这不是三种客户端上完整登录、聊天、语音的实机功能测试。

客户端刷新 `parco_proxy` 规则集即可读取本次更新，不需要为此刷新有十分钟时效的 E-IX 节点订阅。
