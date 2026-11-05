# 小米 HyperOS / MIUI 数据收集域名拦截规则 · 说明文档

> 版本 1.0.0 · 整理日期 2026-11-05 · 静态整理，非自动更新订阅
> 本文解释规则里每个域名**收集什么、依据来自哪里、屏蔽后会有什么影响**。规则本身见同目录的规则文件。

---

## 0. 交付内容一览

| 文件 | 格式 / 适用工具 | 说明 |
| --- | --- | --- |
| `xiaomi-hyperos-telemetry.adguard.txt` | AdGuard / Adblock Plus / AdGuard Home / uBlock Origin | **主规则**（推荐）。标准 AdGuard 屏蔽语法（双竖线 + 域名 + `^`），按 7 类分组并带注释 |
| `xiaomi-hyperos-telemetry.hosts.txt` | hosts 文件（Android root / PC / 路由器） | 主规则的 hosts 版，`0.0.0.0 域名` |
| `xiaomi-hyperos-telemetry.dnsmasq.conf` | DNSMasq（OpenWrt / 梅林 / AdGuard Home 导入） | 主规则的 `address=/域名/` 版 |
| `xiaomi-hyperos-telemetry.clash.yaml` | Clash.Meta / Mihomo `rule-providers` payload | `DOMAIN-SUFFIX` 列表 |
| `xiaomi-hyperos-telemetry.surge.list` | Surge / Shadowrocket / Loon / Stash | `[Rule]` + `DOMAIN-SUFFIX,域名,REJECT` |
| `xiaomi-hyperos-optional-caution.adguard.txt` | AdGuard | **谨慎使用**：账号/推送/更新等功能链路域名，屏蔽会影响功能 |
| `xiaomi-hyperos-all-domains.csv` | CSV | **全量对照清单**：3798 条合并域名，逐条标注分类、是否小米自有域、来源 |

**数字**：4 个开源列表 + 权威资料点名域名 → 清洗去重后得到 **3798** 条小米相关域名；其中 **995** 条判定为「数据收集类」进入主规则，**87** 条为「功能类但携带设备/使用数据」单独列出，**2716** 条为电商、CDN、论坛、云服务等功能/内容域名，未纳入拦截。全部 3798 条都在 CSV 里可核对。

---

## 1. 这份规则是怎么做出来的

1. 拉取 4 个专门针对小米/HyperOS 的开源拦截列表（hosts / AdGuard / Pi-hole 格式）与 1 个行为遥测规则集，见下表 S4–S8。
2. 解析出域名，统一小写、去掉 IP 与残缺/拼接损坏的条目、校验域名合法性、去重。
3. 剔除其中与小米无关的通用广告域名（如快手、知乎、Boss 直聘等第三方应用自己的埋点域名），只保留**小米自有域名**、名称中带 `xiaomi`/`miui` 的云托管与 CDN 别名，以及**被资料明确点名**的第三方 SDK 接收端。
4. 用「广告 / 统计遥测 / 崩溃诊断 / 安全遥测 / 浏览器与内容 / 第三方接收端 / 测试节点」关键词把收集类域名分成 7 组；其余归入对照清单。
5. 按 anti-AD 合并规则项目的多格式订阅风格 [S10] 输出 5 种格式。

> **重要**：域名清单来自社区抓包整理，不是小米官方公布的清单。不同 ROM 版本、区域、已卸载的系统应用，实际请求的域名差别很大。带 `staging`/`test`/区域前缀的长尾域名，多数是同一服务在不同环境或地区的别名，其收集内容与主域名一致，但**没有逐条抓包验证**。

---

## 2. 信息来源清单

| 编号 | 来源 | 性质 | 本文用到了什么 |
| --- | --- | --- | --- |
| S1 | [《Android Mobile OS Snooping By Samsung, Xiaomi, Huawei and Realme Handsets》](https://www.scss.tcd.ie/doug.leith/Android_privacy_report.pdf)，Doug Leith 等，Trinity College Dublin，2021 | **学术论文 / 实测抓包 + 解密** | 明确点名 `tracking.intl.miui.com`、`api.ad.intl.xiaomi.com`、`data.mistat.intl.xiaomi.com`；给出解密后的事件报文与标识符清单 |
| S2 | [MIUI 14 Privacy White Paper](https://trust.mi.com/docs/miui-privacy-white-paper-global/)（[4.12 App Usage Data](https://trust.mi.com/docs/miui-privacy-white-paper-global/4/12)、[第 5 章 International Data Transfer](https://trust.mi.com/docs/miui-privacy-white-paper-global/5)），小米官方，2023-05 | **官方文件** | 官方列明的应用使用数据字段、用途（分析 / 个性化 / 广告）、180 天保留期、数据存储国家与云服务商 |
| S3 | [小米隐私政策](https://privacy.mi.com/all/zh_CN/) / [小米隐私中心](https://privacy.miui.com/en/) | **官方文件** | 设备与 SIM 信息、应用列表、日志信息、位置、浏览与搜索记录等收集类别 |
| S4 | [Kittyskj/FreeFromMi](https://github.com/Kittyskj/FreeFromMi)（`hosts`、`hosts_adguard`、`Xiaomi_ata.md`、`README.md`） | **开源拦截列表 + 抓包示例** | 全量小米服务域名；`Xiaomi_ata.md` 的广告/遥测/崩溃分类说明；README 中 `api.sec.miui.com` 与 `sdkconfig.ad.xiaomi.com` 的真实 POST 报文 |
| S5 | [smokedzn/xiaomi-dns-filter](https://github.com/smokedzn/xiaomi-dns-filter)（`xiaomi_pihole_block.lst`，合并了 HaGeZi Multi PRO++ 与 unknownFalleN 的小米列表） | **开源 DNS 拦截列表** | 1271 条人工核对的小米遥测/广告域名，含 `bigdata.*.neo.xiaomi.com`、`pandora` 等长尾节点 |
| S6 | [moodyKroko — Hosts for blocking MIUI from calling back home](https://gist.github.com/moodyKroko/343f288d81b005742949d113b65a407a) | **开源 hosts 列表** | 245 条 MIUI 回连域名，用于交叉验证 |
| S7 | [AWAvenue-Ads-Rule Issue #197](https://github.com/TG-Twilight/AWAvenue-Ads-Rule/issues/197) | **中文规则社区注释** | 对 `data.mistat.xiaomi.com`、`tracking.miui.com`、`api.ad.xiaomi.com`、`tracker.ai.xiaomi.com`、浏览器广告位的中文用途说明与放行建议 |
| S8 | [Aegis / BehaviorDomain.list](https://github.com/Thoseyearsbrian/Aegis/blob/main/rules/BehaviorDomain.list) | **开源行为遥测识别规则** | 交叉验证 `tracking.miui.com`、`sdkconfig.xiaomi.com` 等遥测节点 |
| S9 | [XDA：Xiaomi 浏览器数据收集研究报道](https://xda-developers.com/xiaomi-mi-web-browser-pro-mint-collecting-browsing-data-incognito-mode)（2020，Forbes / G. Cirlig / A. Tierney）、[蓝点网中文报道](https://landian.vip/archives/73487.html) | **安全研究 / 媒体报道** | Mi Browser / Mint Browser 上报 URL、搜索词、信息流浏览与设备元数据；小米的回应；神策数据（Sensors Analytics）SDK |
| S10 | [217heidai/adblockfilters（anti-AD 合并规则）](https://github.com/217heidai/adblockfilters) | **格式参考** | 仅用于对齐多格式订阅的语法与文件头风格，未使用其域名数据 |

**证据等级**：S1/S2/S3 为可直接核查的实测或官方口径 > S4/S5/S6/S8 为社区抓包与 DNS 观测整理的域名清单 > S7/S9 为社区标注与媒体报道。文中「收集内容」若来自官方字段清单或解密报文会明确写出；仅由域名命名与上游分类推断的部分，统一表述为「通常用于 / 一般对应」。

---

## 3. 这些域名具体收集什么

### 3.1 汇总：跨类别都会出现的字段

综合 [S1][S2][S3]，一次上报里常见的字段可分为五组：

| 类别 | 具体字段 |
| --- | --- |
| 设备与网络标识 | IMEI / MEID、Android ID、OAID / VAID / AAID、GAID（Google 广告 ID）、硬件序列号、MAC / Wi-Fi MAC 的 MD5、Secure DeviceID、ECV（由小米账号 ID + 设备 ID + 已连接 Wi-Fi ID + 位置生成的环境特征值）、IP |
| 设备与系统状态 | 机型、MIUI/HyperOS 与 Android 版本、屏幕分辨率、CPU / 存储 / 电池 / 温度、设备激活时间、销售渠道、语言 / 国家地区 / 时区、字体、运营商与网络类型、SIM 位置区 |
| 应用数据 | 已安装应用清单（包名 + 版本 + 安装来源）、应用下载 / 安装 / 更新 / 卸载 / 首次启动事件、应用进出前后台的时间、SDK 版本、系统更新设置 |
| 行为事件 | 每个 Activity/窗口的打开与关闭时间戳及停留时长、通话开始与结束时间、点击 / 曝光 / 滑动、文件夹与设置菜单打开频次、会话时长、页面访问时长 |
| 内容与日志 | 浏览历史与 URL、搜索词、信息流阅读 / 订阅、音乐播放记录、主题与壁纸的浏览与搜索、崩溃堆栈、系统日志、网络请求日志、Cookie 与标识符 |

[S1] 的解密报文给出了最直观的例子：向 `tracking.intl.miui.com/track/v4` POST 的事件序列里包含 `pkg`（包名）、`class`（Activity 名）、`ts`（时间戳）、`stat`（`app_start` / `app_end`）、`duration`、`app_duration`。论文据此指出，拨号界面 `InCallActivity` 的开关时间会被上报，从而可推断通话的时间与时长。论文同时观测到：即使在开机引导时关闭「使用情况与诊断」，这类事件遥测仍然存在；并记录了小米设备把数据从欧洲发往**新加坡**服务器（`tracking.intl.miui.com`、`api.ad.intl.xiaomi.com`、`data.mistat.intl.xiaomi.com`）[S1]。

[S2] 的 4.12 节则给出官方口径：应用使用数据包含 GAID、设备与网络信息、应用使用序列 / 状态 / 已安装应用快照 / 设备上报数据，用途为**分析改进、个性化与广告**，传输用 HTTPS、保存 **180 天**，并明确「关闭应用使用数据不会阻止设备发送系统更新与安全等基础服务所需的遥测」。第 5 章列出数据存储地：中国大陆（阿里云、金山云、世纪互联）、新加坡（阿里云、AWS 美国）、EEA（AWS 德国、Azure 荷兰）、印度（AWS / Azure）、俄罗斯（金山云）、英国等。

### 3.2 按分组说明

| 分组 | 条数 | 收集内容 | 代表域名 | 主要依据 |
| --- | --- | --- | --- | --- |
| 01 广告投放与个性化推荐 | 116 | 广告请求与素材下发、开屏/信息流/横幅广告位、曝光/点击/转化回传、定向与用户画像、广告 SDK 远程配置；携带 OAID/GAID、机型、区域、应用列表 | `api.ad.xiaomi.com`、`ad.mi.com`、`sdkconfig.ad.xiaomi.com`、`zeus.ad.xiaomi.com`、`ssp.ad.xiaomi.com` | [S1][S2][S3][S4][S7] |
| 02 使用统计与行为遥测 | 389 | 核心遥测：事件序列、应用使用与安装清单、设备标识符、A/B 实验与采集配置下发、大数据/BI/ETL 聚合 | `tracking.miui.com`、`data.mistat.xiaomi.com`、`bigdata.xiaomi.com`、`stat.pandora.xiaomi.com` | [S1][S2][S4][S5][S6][S8] |
| 03 崩溃日志与诊断上报 | 45 | 崩溃堆栈、错误日志、性能监控（APM/RUM）、网络请求日志与设备信息 | `apm-rum.inf.miui.com`、`sentry.sec.miui.com`、`log.pandora.xiaomi.com` | [S2][S4][S5] |
| 04 安全组件遥测 | 73 | 病毒库/引擎更新请求、安全扫描结果、设备指纹（IMEI-MD5、机型、MIUI 版本、区域、运营商）、集成的 Avast 安全遥测 | `api.sec.miui.com`、`virus-engine.sec.miui.com`、`etl.xlmc.sec.miui.com` | [S4] 抓包示例与 ata 分类 |
| 05 浏览器与内容推荐 | 64 | 浏览与搜索记录、网址安全检测提交的 URL、信息流与横幅推荐位、主题/壁纸/负一屏的浏览点击曝光与订阅 | `api.browser.miui.com`、`r.browser.miui.com`、`hot.browser.miui.com` | [S3][S7][S9] |
| 06 第三方 SDK 接收端 | 31 | 系统应用内嵌的第三方统计/广告/推送 SDK 的数据接收端；以及海外用户数据的云托管与 CDN 别名 | `sensorsdata.10046.xiaomimobile.com`、`adview.cn`、`astat.bugly.qq.com`、`*.elb.amazonaws.com` | [S1][S4][S9] |
| 07 测试与预发布节点 | 277 | 上述服务在 `staging` / `test` / `preview` / 灰度环境的别名；行为与对应正式节点一致 | `staging.*`、`test.*`、`*.test.xiaomi.com` | [S4][S5] |

---

## 4. 关键域名逐条说明

下表覆盖主要节点；**完整 995 条**在规则文件中按分组全部列出，`xiaomi-hyperos-all-domains.csv` 里还额外标了每条的来源编号。

### 4.1 广告与个性化

| 域名 | 用途 / 收集内容 | 依据 |
| --- | --- | --- |
| `api.ad.xiaomi.com` | 系统广告 API 主接口，开屏广告与系统内推广位的素材加载 | [S7] |
| `api.ad.intl.xiaomi.com` | 国际版广告接口；论文观测到系统应用 `com.miui.msa.global` 向其发送加密数据，与广告管理相关 | [S1] |
| `ad.xiaomi.com` / `ad.intl.xiaomi.com` / `ad.mi.com` / `ad1.xiaomi.com` | 小米广告平台的投放与定向端点，回传曝光、点击与定向参数 | [S4] ata |
| `sdkconfig.ad.xiaomi.com` | 广告 SDK 远程配置下发。README 抓包示例为 `POST /api/v4/detail/config_common/`，表单含 `re`(区域)、`ml`(机型)、`ov`(小米 OS)、`sv`/`av`/`appVer`(SDK 与应用版本)、`platform`、`sign` | [S4] README 抓包 |
| `sdkconfig.ad.intl.xiaomi.com` / `sdkconfig.ad.india.xiaomi.com` / `sdkconfig.xiaomi.com` | 上述配置的国际/印度/通用节点 | [S4][S5][S8] |
| `zeus.ad.xiaomi.com` / `zeus.ad.intl.xiaomi.com` / `ssp.ad.xiaomi.com` | 广告平台与 SSP/竞价端点，处理竞价、曝光与投放指标 | [S4] ata |
| `log.ad.xiaomi.com` / `diagnosis.ad.xiaomi.com` / `m.track.ad.xiaomi.com` | 广告日志、广告链路诊断、点击/曝光追踪 | [S4] |
| `adv.sec.miui.com` | 手机管家/安全中心内的推广位（不影响杀毒功能） | [S7] |
| `tracker.ai.xiaomi.com` | 小爱同学与系统个性化推荐的训练/追踪节点 | [S7] |
| `marketing.ai.xiaomi.com` / `marketing.ad.xiaomi.com` / `matrix.api.ad.xiaomi.com` | 营销分群、行为信号与定向 API | [S4] ata |
| `databi.ad.xiaomi.com` / `bi.migc.xiaomi.com` | 广告数据的 BI / ETL 聚合 | [S4] ata |
| `ad.jr.mi.com` / `adx.tv.mi.com` / `ad.quickapp.hybrid.xiaomi.com` / `ad.eu.xiaomi.com` / `ad.india.xiaomi.com` | 金融、OTT 电视、快应用及各区域广告节点 | [S4][S5] |
| `maizhenapi.ad.xiaomi.com` / `g.cn.miaozhen.com` | 第三方广告监测（秒针）曝光与转化测量 | [S4] ata |
| `ad.serving-sys.com` / `adview.cn` / `adups.com` / `adsunflower.com` / `adsmind.gdtimg.com` | 第三方广告网络与测量 SDK 的接收端，回传设备与行为数据 | [S4] ata |

### 4.2 统计与行为遥测

| 域名 | 用途 / 收集内容 | 依据 |
| --- | --- | --- |
| `tracking.miui.com` | 国内版 MIUI 用户行为追踪，记录操作习惯用于产品优化 | [S7] |
| `tracking.intl.miui.com` | **论文实测的核心遥测端点**。`com.miui.analytics` 用 AES/ECB 加密后经 HTTPS 上报，报文含带时间戳的窗口打开/关闭事件、包名、Activity、停留时长；密钥经 `/track/key_get` 交换 | [S1] |
| `default.tracking.miui.com` / `om.tracking.miui.com` / `miuiboot.tracking.miui.com` / `mitv.tracking.*.miui.com` | 默认、运维监控、开机与电视/IoT 的遥测节点 | [S4] ata、[S7] |
| `data.mistat.xiaomi.com` | 小米统计服务，收集系统使用数据、错误日志 | [S7] |
| `data.mistat.intl.xiaomi.com` | 论文观测：文件管理器、设置、安全中心等系统应用向其发送加密数据（同套 AES 密钥交换机制） | [S1] |
| `mistat.xiaomi.com` / `stat.xiaomi.com` / `stats.xiaomi.com` / `a.stat.xiaomi.com` | 通用统计聚合端点 | [S4] ata |
| `abtest.mistat.xiaomi.com` / `abtest.mistat.intl.xiaomi.com` | A/B 实验、功能开关与实验指标上报 | [S4] ata、[S6] |
| `storeconfig.mistat.*.xiaomi.com` | 采集规则/开关的远程下发（决定采什么、采多频繁） | [S4] ata |
| `ident.mistat.xiaomi.com` / `o.data.mistat.xiaomi.com` | 标识符映射与统计数据中转 | [S4] ata |
| `api.collect.data.intl.miui.com` / `apm-rum.inf.miui.com` / `apm-rum-{eu,ind,ru,sgp}.inf.miui.com` | 应用遥测收集 API 与各区域性能监控（RUM） | [S4] ata、[S5] |
| `bigdata.xiaomi.com` / `o.bigdata.xiaomi.com` / `*.bigdata.*.neo.xiaomi.com` | 大数据平台与 BI 数据接入节点（S5 列表中数量最多的一类） | [S4] ata、[S5] |
| `stat.pandora.xiaomi.com` / `log.pandora.xiaomi.com` / `monitor.pandora.xiaomi.com` | Pandora 数据平台的统计、日志与监控接入 | [S4]、[S6] |
| `jupiter.*.sys.miui.com` / `client.sms.miui.com` | 系统遥测、短信送达状态日志 | [S4] ata |
| `api.installer.xiaomi.com` / `api.hybrid.xiaomi.com` / `o2o.api.xiaomi.com` | 应用安装/更新数据来源、混合应用与 O2O 服务接口（涉及应用清单与安装事件） | [S4] |
| `sensorsdata.10046.xiaomimobile.com` / `moka.datasink.sensorsdata.cn` | 神策数据（Sensors Analytics）行为分析 SDK 接收端；2020 年研究在小米浏览器与系统应用中发现 `SensorDataAPI` | [S4] ata、[S9] |
| `piwik.account.mi.com` | Matomo/Piwik 站点分析端点 | [S4] ata |

### 4.3 崩溃、日志与安全遥测

| 域名 | 用途 / 收集内容 | 依据 |
| --- | --- | --- |
| `sentry.sec.miui.com` / `sentry.pt.xiaomi.com` / `astat.bugly.qq.com` | Sentry / 腾讯 Bugly 崩溃上报：堆栈、机型、系统版本、设备标识 | [S2][S4] |
| `h.trace.qq.com` / `othstr.beacon.qq.com` / `sdkreport.e.qq.com` / `tdid.m.qq.com` / `tmfsdk.m.qq.com` | 腾讯灯塔/Beacon 链路追踪、SDK 遥测、设备 ID 服务 | [S4] ata |
| `api.sec.miui.com` | **README 抓包示例**：POST 表单含 `appVersion`、`carrier`、`device`、`miuiVersion`、`region`、`versionType`、`sign`，`param` 为 base64，解码后为 `{"imei":"md5hashed","dataVersion":0,"isDiff":false}` | [S4] README |
| `api.sec.intl.miui.com` / `srv.sec.miui.com` / `virus-engine.sec.miui.com` / `flash.sec.miui.com` / `mazu.sec.miui.com` | 安全服务的国际节点、引擎接口与扫描组件 | [S4] |
| `update.avlyun.sec.miui.com` / `logupdate.avlyun.sec.miui.com` / `*.avlyun.com` | 安全组件/病毒库更新与更新日志回传。**注意**：[S7] 把 `update.avlyun.sec.miui.com` 列为「安全组件更新必需，建议放行」，与拦截列表的处理相反，见第 6 节 | [S4][S7] |
| `etl.xlmc.sec.miui.com` / `api.ra2.xlmc.sec.miui.com` / `adinfo.ra1.xlmc.sec.miui.com` | 安全组件的数据聚合（ETL）与广告/安全元数据接口 | [S4] ata |
| `analytics.ff.avast.sec.miui.com` / `ip-info.ff.avast.sec.miui.com` / `apkrep.ff.avast.com` | MIUI 集成的 Avast 安全遥测、IP 信息与 APK 检测上报 | [S4] ata、[S1] 提及第三方 Avast |

### 4.4 浏览器、内容与第三方接收端

| 域名 | 用途 / 收集内容 | 依据 |
| --- | --- | --- |
| `api.browser.miui.com` / `r.browser.miui.com` | 浏览器主接口与首页个性化推荐算法接口 | [S7] |
| `hot.browser.miui.com` / `hd.browser.miui.com` / `hd1.*` / `hot1.*` | 热门推荐信息流、开屏与横幅广告位 | [S7] |
| `security.browser.miui.com` / `security1.browser.miui.com` | 网址安全检测（会把待检测 URL 送出）；如不需要可放行 | [S7] |
| `reader.browser.miui.com` / `c3-cache.browser.miui.com` | 阅读模式与浏览器缓存/资源节点 | [S4][S6] |
| `api.zhuti.xiaomi.com` / `api.zhuti.intl.xiaomi.com` 等主题、壁纸节点 | 主题与壁纸的浏览、搜索、点赞、收藏、分享、曝光与订阅记录 | [S3][S4] |
| `stats.music.xiaomi.com` / `api.tencentmusic.com` / `tmead.y.qq.com` | 音乐播放记录、搜索词与音乐服务分析/广告 | [S3][S4] ata |
| `fclick.baidu.com` / `nsclick.baidu.com` / `mini.cpc.sogou.com` / `api.tuisong.baidu.com` | 百度/搜狗的点击追踪与推送 SDK 回执 | [S4] ata |
| `beacon-api.aliyuncs.com` / `pssn.alicdn.com` / `p.alimama.com` | 阿里云 Beacon / CDN 埋点与阿里妈妈广告归因 | [S4] ata |
| `data-mistat-india-xiaomi-com-*.ap-south-1.elb.amazonaws.com`、`tracking-india-miui-com-*.elb.amazonaws.com` | 印度区域小米遥测服务的 AWS 负载均衡别名，对应 [S2] 第 5 章「印度 AWS」数据节点 | [S5][S2] |
| `*.edgesuite.net` / `*.ksyna.net`（含 `ad.intl.xiaomi.com` 前缀） | 小米国际广告域名使用的第三方 CDN 别名 | [S4] |

> 关于浏览器数据：2020 年安全研究指出 Mi Browser / Mint Browser / MIUI 预装浏览器会上报访问过的 URL、搜索词、信息流浏览记录与设备元数据，隐身模式下亦有争议 [S9]。小米当时回应称数据为匿名聚合、经用户同意，并随后为隐身模式增加了关闭数据收集的开关。本文只记录技术事实，不对双方主张作判定。

---

## 5. 怎么用这些规则

| 场景 | 用哪个文件 | 操作 |
| --- | --- | --- |
| AdGuard Home / AdGuard / uBlock Origin | `*.adguard.txt` | 「过滤器 → 自定义 → 从文件导入」；格式与 anti-AD 规则 1 相同 [S10] |
| 手机(root) / PC / 路由器 hosts | `*.hosts.txt` | 追加到 `/etc/hosts`（Android 为 `/system/etc/hosts`），注意保留 localhost 行 |
| OpenWrt / 梅林 / dnsmasq | `*.dnsmasq.conf` | 放入 `/etc/dnsmasq.d/` 后重启 dnsmasq |
| Clash.Meta / Mihomo | `*.clash.yaml` | 作为 `rule-providers` 的 `payload` 文件，策略组选 `REJECT` |
| Surge / Shadowrocket / Loon / Stash | `*.surge.list` | 复制 `[Rule]` 段内容到配置；Quantumult X 去掉 `[Rule]` 行 |

**误拦截放行**（AdGuard 语法）：`@@||被误拦的域名^$important`，与 anti-AD 的做法一致 [S10]。

**建议的启用顺序**：先只启用 01（广告）、02（统计）、05（浏览器/内容）三组，观察 1–2 天；再按需要加 03（崩溃日志）、04（安全遥测）、06（第三方）、07（测试节点）。07 组普通设备一般不会请求，可整段注释掉。

**配合系统内开关**（比 DNS 拦截更根本）：
- 设置 → 系统安全 → **使用情况与诊断**：关闭（[S2] 4.12 明确给出路径）
- 设置 → 隐私保护 → **广告个性化 / 系统广告（MSA）**：关闭
- 各系统应用（浏览器、应用商店、主题、壁纸、负一屏、小爱同学）内部的「个性化推荐」逐项关闭

---

## 6. 谨慎/可选文件说明

`xiaomi-hyperos-optional-caution.adguard.txt` 的 87 条域名属于**功能链路**，但请求中同样会带设备与使用数据。它们没有进入主规则，因为屏蔽会直接破坏功能：

| 域名（族） | 屏蔽后果 |
| --- | --- |
| `account.xiaomi.com`、`ruac.account.xiaomi.com`、`sgpac.account.xiaomi.com` | 小米账号登录、注册、找回、同步失败 |
| `a0.app.xiaomi.com` | 新机首次开机/恢复后的激活流程异常（[S7] 明确列为必须放行） |
| `update.miui.com`、`update.intl.miui.com`、`api.io.mi.com`、`upgrade.*` | 系统 OTA 检查与升级、IoT 固件更新失败 |
| `*.xmpush.xiaomi.com`、`register.xmpush.*`、`resolver.msg.xiaomi.net` | 小米推送（Mi Push）收不到通知，影响大量 App 的消息送达 |
| `find.api.micloud.xiaomi.net`、`us.find.api.micloud.xiaomi.net`、`api.g.micloud.xiaomi.net` | 「查找设备」、云服务状态异常 |
| `global.market.xiaomi.com`、`thm.market.xiaomi.com` | 应用商店系统应用更新不可用 |
| `mcc.intl.inf.miui.com` | 小米手环/手表固件与表盘兼容性受影响（[S5] 白名单注明） |
| `statusapi.*`、`noteapi.*`、`sfsapi.*`、`relocationapi.*` | 云笔记、云盘等小米云服务的接口异常 |

**一处需要你自行取舍的冲突**：[S4] 把 `update.avlyun.sec.miui.com` 纳入拦截（安全组件遥测/日志），而 [S7] 把它标为「病毒库、安全扫描引擎更新必需，必须放行」。本规则按「数据收集」口径把它放进了 04 组；如果你依赖手机管家的病毒库更新，建议单独放行：

```
@@||update.avlyun.sec.miui.com^$important
@@||a0.app.xiaomi.com^$important
```

另一处边界情况：`userid.xiaomi.com` 因含「标识符」语义被归入主规则的 02 组，但它同时承担账号 ID 服务。若启用后出现账号相关异常，建议放行 `@@||userid.xiaomi.com^$important`。

---

## 7. 局限与免责

1. **不是官方清单**：所有域名来自社区抓包与 DNS 观测整理，未经小米确认；部分域名可能已停用、改作他用或只在特定区域/版本出现。
2. **长尾域名未逐条验证**：尤其是 `staging.*`、`test.*`、区域前缀与 `*.bigdata.*.neo.xiaomi.com` 一类节点，分类依据是命名规律和上游列表的归类，不是逐条抓包结论。清单里保留它们是为了「宁全不漏」。
3. **DNS 层拦截不是万能的**：硬编码 IP、DoH/DoT、VPN 内直连、系统应用内的重试与旁路都可能绕过；被拦截后某些系统进程会频繁重试，反而增加耗电。[S4] 也提示部分应用（如 GetApps）需要配合 ADB 卸载才能彻底断开。
4. **功能影响**：屏蔽遥测与广告接口会让系统广告位、个性化推荐、诊断上报失效；个别情况下可能影响系统应用更新校验。请勿在主力机上一次性全量启用，先小范围验证。
5. **合规提示**：本文件只做公开信息的整理与技术说明，不构成对厂商行为的定性结论；厂商的官方口径见 [S2][S3]。规则仅供个人在自有设备上管理网络请求使用。
6. **静态文件**：不会自动同步上游。建议每季度重新核对 [S4][S5] 的更新；CSV 里带来源编号，方便自己复核。

## 8. 自己复核的方法

- **DNS 查询日志**：在 AdGuard Home / Pi-hole / NextDNS 里查看拦截命中，确认哪些域名在你的机型上真的被请求过。
- **设备抓包**：用 PCAPdroid（免 root 的本地 VPN 抓包）或 root 环境下配合证书抓取 HTTPS，观察 `tracking.*`、`*.mistat.*`、`*.ad.xiaomi.com` 的请求频率。
- **对照官方字段**：把观察到的上报字段与 [S2] 4.12 的数据清单、[S3] 隐私政策的「日志信息 / 应用使用信息」条目逐项比对。
- **逐条查来源**：打开 `xiaomi-hyperos-all-domains.csv`，`sources` 列标明该域名来自哪个开源列表（S4/S5/S6/S8）或是否由权威资料点名。

---

[S1]: https://www.scss.tcd.ie/doug.leith/Android_privacy_report.pdf
[S2]: https://trust.mi.com/docs/miui-privacy-white-paper-global/
[S3]: https://privacy.mi.com/all/zh_CN/
[S4]: https://github.com/Kittyskj/FreeFromMi
[S5]: https://github.com/smokedzn/xiaomi-dns-filter
[S6]: https://gist.github.com/moodyKroko/343f288d81b005742949d113b65a407a
[S7]: https://github.com/TG-Twilight/AWAvenue-Ads-Rule/issues/197
[S8]: https://github.com/Thoseyearsbrian/Aegis/blob/main/rules/BehaviorDomain.list
[S9]: https://xda-developers.com/xiaomi-mi-web-browser-pro-mint-collecting-browsing-data-incognito-mode
[S10]: https://github.com/217heidai/adblockfilters
