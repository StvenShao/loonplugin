# 掌上英雄联盟去开屏广告 —— 抓包分析与 LOON 插件说明

> 交付物：`掌上英雄联盟去开屏广告.plugin`（LOON 插件，响应改写 + 连接拒绝，需 MITM 证书）
> 日期：2026-10-28

---

## 一、抓包记录分析

### 1. 抓包概况

压缩包 `22_1793123369518.zip` 共 **140 个 HTTP(S) 会话**（抓包时长约 50 秒，覆盖 App 冷启动、登录态校验、首页加载）。根据 UA 确认目标 App：

- **App：掌上英雄联盟**（腾讯 LOL 掌上助手，UA `QTL/12.9.1`，版本 **12.9.1**）
- **设备：iPhone / iOS 27.0.1**

### 2. 主要流量域（按功能归类）

| 域 | 用途 | 是否广告 |
|---|---|---|
| `mlol.qt.qq.com` | 掌盟主 API（登录、资讯、推荐、聊天、活动、广告位接口） | 部分接口是广告位 |
| `119.29.29.98`（`/d?id=...&dn=...`） | 腾讯 DNSPod **明文 HTTP DNS**，用于绕开系统 DNS | 否（但会干扰规则匹配） |
| `k6.tgl.qq.com` / `mlol-75288.qpic.cn` / `zm-picture...myqcloud.com` / `qlogo.cn` | 图片 CDN（资讯封面、头像、话题图） | 否（内容图片，不可拦截） |
| `szmg.qq.com`（`/cgi-bin/log_data.fcg?offer_id=1450015760`） | 腾讯广告曝光/点击上报 | 是（仅上报，非下发） |
| `snowflake.qq.com` / `othstr.beacon.qq.com` / `sentry.qt.qq.com` | 腾讯数据上报 / Sentry 崩溃上报 | 否 |
| `cloud.tgpa.qq.com` / `hc.tdm.qq.com` / `receiver.tdm.qq.com` / `tracker-01.qvb.qcloud.com` | 腾讯安全防护、动态配置、公告 | 否 |

### 3. 广告接口识别（本插件核心依据）

| 接口（均在 `mlol.qt.qq.com`） | 职责 | 本次抓包中的关键证据 |
|---|---|---|
| **`/go/recommend/platflashbox`** | **开屏广告配置接口**（冷启动开屏 + 登录后开屏） | 响应含 `ams_status:1`（腾讯广告 AMS 总开关）、`start_*/login_*` 素材字段（素材 ID/图片 URL/跳转 URL）、`ams_cold_start_interval:120`（冷启动广告间隔）、`max_show_splash_times:2` |
| `/go/recommend/platstrongshell` | 开屏/启动强弹广告 | 请求参数含 `last_ad=11556`（上次广告 ID）；现有插件已将其改写为 `{}` |
| `/go/zone/bottomtab_tip` | 底部 Tab 提示广告 | 现有插件已将其改写为 `{}` |
| `/go/recommend/platbanner` | 发现页运营 Banner（`isAmsAd:false`，活动推广位） | 返回多条 `feedsInfo`（adid、图片、跳转活动页） |
| `/go/recommend/floatbox` | 悬浮窗广告 | 响应 `data:{}`，`debug` 含广告 ID=225 与跳转链接 |

### 4. 现有插件（kelee.one `LOL_Bible_remove_ads.lpx`）为什么失效

从抓包中的 `replace_response_*` 文件可还原现有插件的行为：

1. `/go/recommend/platstrongshell` → 改写为 `{}` ✓
2. `/go/zone/bottomtab_tip` → 改写为 `{}` ✓
3. `119.29.29.98` HTTP-DNS → 404（强制回落系统 DNS）✓

**但它漏掉了真正的开屏配置接口 `/go/recommend/platflashbox`。** 开屏广告的素材和开关正是由 platflashbox 下发（`ams_status=1` + `start_*` 素材字段）。只拦强弹接口、不关掉 platflashbox 里的广告开关，冷启动开屏广告依然会显示。另外，**开屏素材可能被 App 本地缓存**——即使接口已拦截，缓存中的广告图在过期前仍会展示一两次，这也是"装了插件还有广告"的常见原因。

> 附注：kelee.one 站点已被 Cloudflare 拦截，无法直接获取插件原文；以上行为还原基于抓包中实际出现的"替换响应"记录。

---

## 二、新插件设计

```
[Rewrite]                          # 作用
──────────────────────────────────────────────────────────
platflashbox
  response-body-json-jq            # 核心：ams_status=1→0（关闭腾讯广告总开关）
  '.data.ams_status=0|            #    清空 start_*/login_* 素材与跳转字段
   .data.start_item_id=""|...'    #    展示次数 max_show_splash_times=0
platstrongshell → reject-dict      # 开屏强弹 → 返回 {}（沿用旧插件思路）
bottomtab_tip    → reject-dict     # 底部提示 → 返回 {}
119.29.29.98/    → reject          # 腾讯 HTTP-DNS → 连接拒绝，回落系统 DNS

[Mitm]
hostname = mlol.qt.qq.com          # 仅 MITM 掌盟主域（响应改写必需）
```

要点：

- **只动广告接口**：不拦截 `mlol.qt.qq.com` 其余接口、不拦截图片 CDN（封面/头像/资讯图）、不影响登录、资讯、视频播放。
- **结构保持**：platflashbox 用 jq 仅修改字段值（开关归零、素材清空），保留原 JSON 结构，App 不会因解析失败而异常。
- **可选开关**：插件内注释了关闭发现页 banner、悬浮窗、腾讯广告上报的规则，需要时取消注释即可。

---

## 三、使用方法（LOON）

1. 导入 `掌上英雄联盟去开屏广告.plugin`（LOON → 配置 → 插件 → `+`）；
2. **必须安装并信任 MITM 证书**（LOON → 设置 → 证书管理 → 安装 CA 证书 → 系统设置里"已下载描述文件"→ 安装并"信任"），否则 HTTPS 响应改写不生效；
3. 启用插件后，**杀掉掌上英雄联盟再冷启动**验证；
4. 若仍显示 1~2 次广告：是**本地缓存**所致——请在掌盟内清除缓存，或等待缓存过期（最长一天），必要时卸载重装（会清空登录态与本地数据）。

## 四、验证方法

- 冷启动掌盟 → 应直接进入首页，不再出现广告开屏；
- 在 LOON 的「请求日志」中可看到 `platflashbox` 请求被改写（返回体 `ams_status:0`）或 `platstrongshell` 被置空；
- 资讯、视频、登录等功能应一切正常。

## 五、边界说明

- 插件仅拦截广告展示，**不绕过会员/VIP/付费内容**；
- 本抓包窗口内 platflashbox 恰好返回空素材（广告按活动期/概率下发，未命中不代表接口无职责），但接口的广告字段与开关已明确，改写方案不受影响；
- 若未来版本接口路径变化导致失效，用抓包工具在开屏广告出现的瞬间抓包，把新的广告接口路径发给我即可更新。

## 六、参考来源

- LOON 官方文档：插件格式 — https://nsloon.app/en/docs/Plugin/
- LOON 官方文档：Rewrite（响应改写 / reject / jq）— https://nsloon.app/en/docs/Rewrite/
- LOON 官方文档：Script — https://nsloon.app/en/docs/Script/
