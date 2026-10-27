# 红果短剧去开屏广告 —— 抓包分析与 LOON 插件说明

> 交付物：`红果短剧去开屏广告.plugin`（LOON 插件，纯规则拦截，无需 MITM 证书）
> 日期：2026-10-28

---

## 一、抓包记录分析

### 1. 抓包概况

压缩包 `21_1793121715016.zip` 共包含 **11 个 HTTP(S) 会话**，每个会话含请求/响应头与请求/响应体。根据 User-Agent 与请求特征可确认目标 App：

- **App：红果短剧**（字节跳动旗下免费短剧 App，内部代号 `Eggplant`）
- **版本：7.4.1**（UA：`Eggplant/7.4.1.32 CFNetwork/3896.100.1.2.1 Darwin/27.0.0`，`app_id=8662`）
- **设备：iPhone18,4 / iOS 27.0.1**

### 2. 会话分类（11 个）

| 类别 | 数量 | 目标域名 | 说明 |
|---|---|---|---|
| Gecko 配置下发 | 3 | `gecko.zijieapi.com/gkx/api/resource/v7/custom`（POST） | 字节跳动客户端配置系统（Gecko），本次返回的是本地生活 POI 配置清单（life_*/poi_*/canal_*，含 `bytegecko.com` 资源包下载地址），**与开屏广告无关** |
| 短剧视频播放 | 8 | `*.qznovelvod.com`（v26-reading-video / v11-reading-video / v5-ex-reading-video） | 短剧正片 MP4 的 Range 分片请求（`X-ReqType: preload/play`），响应体为 `ftypisom` 开头的 MP4 数据，**是核心功能流量，必须放行** |

### 3. 关键结论

**这份抓包记录中没有捕获到任何开屏广告请求。**

- 未出现穿山甲（`pangolin-sdk-toutiao.com` / `eoff6.com`）、字节广告下发（`ads*.zijieapi.com`）、广告素材（`*-ad-sign.byteimg.com` / `*-reading-ad.qznovelvod.com`）等任何广告域流量。
- 最可能的原因：
  1. 广告请求走了 **QUIC/HTTP/3（UDP 443）**，多数抓包工具（HTTP 代理模式）抓不到 UDP 流量；
  2. 本次冷启动时开屏广告命中了本地缓存，未重新拉取；
  3. 本次启动恰好未展示广告（广告展示是概率性的）。

因此，**插件无法从这份抓包中"反推"出唯一的开屏广告域名**，本插件采用的是社区中已被验证有效的红果短剧/番茄系广告域名拦截方案（见下文依据），并给出精调路径（见第四节）。

---

## 二、插件设计依据

### 1. 社区已验证的拦截清单（合并去重）

| 域名/模式 | 拦截目标 | 来源 |
|---|---|---|
| `ads*-normal-l*.zijieapi.com` | 字节广告下发/开屏接口 | LOWERTOP 红果模块、chxm1023 番茄去广告列表 |
| `pangolin-sdk-toutiao.com`（后缀） | 穿山甲广告 SDK | chxm1023 番茄去广告列表 |
| `eoff6.com`（后缀） | 穿山甲广告 API 族 | 穿山甲公共广告域 |
| `*-reading-ad.qznovelvod.com` | 红果短剧广告视频 | LOWERTOP 红果模块 |
| `*-ad-sign.byteimg.com` | 广告素材签名图 | LOWERTOP 红果模块、chxm1023 番茄列表 |
| `*-misc-lq.fqnovel.com` | 番茄系广告杂项资源 | LOWERTOP 红果模块 |
| `is.snssdk.com` / `i.snssdk.com` / `security-lq.snssdk.com` | 字节旧版开屏/回传接口 | chxm1023 番茄去广告列表 |

### 2. 放行清单（依据本抓包确认，不可拦截）

| 域名 | 用途 |
|---|---|
| `gecko.zijieapi.com` | Gecko 配置下发，App 正常运行必需 |
| `*.bytegecko.com` | Gecko 配置资源包 CDN |
| `*-reading-video.qznovelvod.com`（v26/v11/v5-ex 等） | **短剧正片视频**，拦截会导致无法播放 |

> 社区 LOWERTOP 模块曾拦截 `v11-reading-video.qznovelvod.com`，但本抓包证明该域名正在承载短剧正片视频，**故不启用**，仅放入默认关闭的"可选激进项"并加警告注释。

### 3. 拦截机制

纯 `[Rule]` + `REJECT`：广告接口被拒绝连接后，广告 SDK 拿不到素材，App 会跳过广告、在短暂品牌页后直接进入首页。无需安装 MITM 证书，不影响视频播放。

---

## 三、使用方法（LOON）

1. 将 `红果短剧去开屏广告.plugin` 传到 iPhone（AirDrop / 微信文件 / 文件 App 均可）；
2. 打开 LOON → 「配置」→「插件」→ 右上角 `+` → 选择该文件导入并**启用**；
3. **重启红果短剧**验证：冷启动应不再出现广告开屏（或短暂品牌页后直接进首页）。

> 若 LOON 主配置未开启 TUN 模式，请先开启「设置 → 流量 → TUN 模式」。

## 四、如果开屏广告仍然出现（精调路径）

1. **确认广告请求是否走了 QUIC**：LOON → 设置 → 流量 → 打开「拦截 QUIC/HTTP3」，再重启红果短剧测试；
2. **补抓包**：在抓包工具中确认开启 UDP 抓包，**杀掉红果短剧后冷启动**，在开屏广告出现的瞬间抓包，导出包含广告域名的抓包记录发我，我按实际域名补一行规则即可；
3. **激进模式**：取消插件内"可选激进项"的注释（`*-iosreadingvideo` / `v11-reading-video` / `novelapp.ixigua.com`）后重启 LOON 再试；若出现视频无法播放，恢复注释。

## 五、边界说明

- 本插件仅拦截广告流量，**不绕过会员/VIP、付费内容等任何付费功能**；
- 规则基于公开社区方案与本次抓包设计，仅供个人学习与研究使用；
- 若与其他字节系 App（抖音、头条等）冲突，请按插件内注释逐行关闭 snssdk 相关规则。

## 六、参考来源

- LOWERTOP/Shadowrocket-First：`Cleaner.module`（红果短剧去广告规则）— https://raw.githubusercontent.com/LOWERTOP/Shadowrocket-First/main/Cleaner.module
- chxm1023/Script_X：`fanqiead.list`（番茄小说去广告）— https://raw.githubusercontent.com/chxm1023/Script_X/main/filter/fanqiead.list
- LOON 官方文档：插件格式 — https://nsloon.app/en/docs/Plugin/
