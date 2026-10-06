<img src="icon.png" width="96" align="right" alt="QQ Lite">

# QQ 瘦身 · QQ Lite

[![validate](https://github.com/k6nfmm7dbr-commits/qq-lite/actions/workflows/validate.yml/badge.svg)](https://github.com/k6nfmm7dbr-commits/qq-lite/actions/workflows/validate.yml)
![clients](https://img.shields.io/badge/clients-Egern%20%7C%20Surge%20%7C%20Loon%20%7C%20Stash%20%7C%20Shadowrocket-blue)
![license](https://img.shields.io/badge/license-MIT-green)

一个去广告 / 降内存的代理模块：拦截 **QQ**（`com.tencent.mqq`）的
**腾讯广告域名与埋点上报**，减少开屏广告等待、QQ 空间 / 看点信息流广告加载和后台统计开销。

支持 **Egern**（原生 YAML 模块）、**Surge / Stash / Shadowrocket**（`.sgmodule`）、
**Loon**（`.plugin`）。**零 MITM、零证书、零脚本。**

---

## ⚠️ 先说结论

代理层（Egern / Surge / Loon 这类）**无法直接降低 QQ 的内存占用**。
QQ 的内存大头全在 App 内部：

| 内存大头 | 量级 | 代理层能否触及 |
|---|---|---|
| 聊天记录数据库 + 消息索引 | 大 | ❌ 不能 |
| 图片 / 视频解码缓存 | 大 | ⚠️ 只能减少"要解码的东西" |
| QQ 空间 / 看点 / 小世界的 WebView | 中 | ⚠️ 只能减少页面里的第三方资源 |
| 小程序 / 内置浏览器（TBS）运行时 | 中 | ❌ 不能 |
| 埋点 / 上报队列 | 小 | ✅ 能 |
| **开屏广告 + 信息流广告** | 中 | ✅ **能** |

所以：**这个模块是"组合拳"里的一环，不是银弹。**
真正立竿见影的另一半在 [App 内设置](#六app-内设置真正有效的那一半) 那一节。

---

## 一、QQ 的广告和抖音、红果不一样

这是个让人省心的差别：

| | 抖音 / 红果短剧 | **QQ** |
|---|---|---|
| 广告素材 | 和正片**共用同一个 CDN** | 走腾讯**自建的独立广告域** |
| 域名层能否区分 | ❌ 不能，必须 MITM 看路径 | ✅ **能**，`ad.qq.com` / `adsview.qq.com` 这些域只干广告 |
| 需要证书吗 | 进阶版需要 | **不需要** |

也就是说：**QQ 这边光靠域名规则就能把广告拦干净，不用解密任何流量。**
零 MITM 也意味着零额外 CPU 开销 —— 对本来就在卡顿的机器更友好。

---

## 二、安装

| 客户端 | 导入链接 |
|---|---|
| **Egern** | [qq-lite.yaml](https://raw.githubusercontent.com/k6nfmm7dbr-commits/qq-lite/main/egern/qq-lite.yaml) |
| **Surge / Stash / Shadowrocket** | [qq-lite.sgmodule](https://raw.githubusercontent.com/k6nfmm7dbr-commits/qq-lite/main/surge/qq-lite.sgmodule) |
| **Loon** | [qq-lite.plugin](https://raw.githubusercontent.com/k6nfmm7dbr-commits/qq-lite/main/loon/qq-lite.plugin) |

- **Egern**：工具 → 模块 → 右上角 `+` → 粘贴上面的链接
- **Surge**：首页 → 模块 → 安装新模块 → 粘贴链接
- **Loon**：配置 → 插件 → `+` → 粘贴链接
- **不想托管任何文件**：把 [`egern/snippet-profile.yaml`](egern/snippet-profile.yaml) 里的
  `rules` 直接粘进 Egern 主配置 `Profile.yaml` 即可

### 转成别的客户端

> 在 Surge / Egern 里打开 **`script.hub`**（或访问 <https://scripthub.vercel.app>）
> → 来源类型选「Surge 模块」，粘贴上面的 raw 链接 → 目标类型选你要的客户端。

---

## 三、拦截清单

共 **36 条规则**，分三组。

### 1. 腾讯广告体系（广点通 / 优量汇）— 5 条

QQ、QQ 空间、看点、腾讯新闻共用同一套广告投放系统：

| 域名 | 说明 |
|---|---|
| `gdt.qq.com` | 广告请求主域，一条后缀覆盖 `mi.` / `win.` / `c.` / `v.` / `c2.` / `canvas-cdn.` 等子域 |
| `pgdt.gtimg.cn` | 广告素材 CDN |
| `pgdt.ugdtimg.com` | 同上，另一条素材 CDN |
| `e.qq.com` | 曝光上报（`sdk.e.` / `tangram.e.`） |
| `l.qq.com` | 点击上报（`c.l.` / `c2.l.` / `cm.l.`） |

### 2. 腾讯自有广告投放 / 素材 / 监测 — 22 条

这一组是腾讯广告系统在 `*.qq.com` 下的一批**专用域**，名字都以 `ad` 开头：
`ad.qq.com`、`adnet`、`adping`、`adsclick`、`adsfile`、`adsgroup`、`adshmct`、
`adshmmsg`、`adslvfile`、`adslvseed`、`adsqqclick`、`adsrich`、`adstextview`、
`adsview`、`adsview2`、`ads.privacy`、`adsmind.*`（3 条）、
`ads-acc.qqbrowser`、`adfilter.imtt`、`advert-prod.hyrz`。

> 我核对过完整列表：**`*.qq.com` 下所有 `ad` 开头的域名都是广告，没有正常业务混在里面**，
> 所以这一组可以放心拦。最后三条来自 QQ 内置浏览器 / 腾讯浏览服务（TBS），
> QQ 里很多页面是 WebView 渲染的，所以一并拦掉。

### 3. 埋点 / 统计 / 崩溃上报 — 9 条

`beacon.qq.com`、`beaconcdn.qq.com`、`beacon.cdn.qq.com`、`pingjs.qq.com`、
`sngmta.qq.com`（腾讯移动分析）、`mtrace.qq.com`、`trace.qq.com`、
`statistic.qzone.qq.com`（QQ 空间统计）、`snowflake.qq.com`。

对内存帮助很小（省一点网络与后台线程），风险也低。**介意就整段注释掉。**

### 故意没做的事

| 想法 | 为什么不做 |
|---|---|
| 拦 `qzonestyle.gtimg.cn` | QQ 空间的样式和脚本，拦了空间**直接白屏** |
| 拦 `imgcache.qq.com` / `qpic.cn` | 空间和头像的图片 CDN，拦了**全是裂图** |
| 拦 `r.inews.qq.com` / `news.ssp.qq.com` | 看点的内容源。`news.ssp.qq.com` 确实带 SSP 字样，但它同时供内容，不做默认拦截 |
| 拦 `qq.com` 主域 | QQ 直接打不开 |
| 拦文件传输 / 语音视频 / QQ 钱包相关域 | 都是核心功能 |
| 用脚本改客户端配置去精简界面 | 那是 **UI 精简**不是省内存，而且依赖会随版本变的结构。想要界面精简请用社区那类模块，别和去广告混在一起 |

---

## 四、⚠️ 两个必须注意的点

### 1. 规则顺序决定生死

Egern / Surge / Loon 的规则都是**从上到下匹配第一条命中的**。
如果你主配置里有这类宽泛规则且排在前面：

```yaml
- domain_suffix:
    match: "qq.com"
    policy: DIRECT          # ← 广告域名也匹配 qq.com，会被它抢先放行
```

那么本模块的 `REJECT` **完全不会生效**。请确保本模块规则排在它之前。

**验证**：请求记录 → 打开 QQ → 看 `adsview.qq.com`、`gdt.qq.com` 是否显示 `REJECT`。

### 2. 腾讯的广告域不止服务 QQ

`gdt.qq.com`、`pgdt.gtimg.cn` 这些是**腾讯整个广告体系**的域名，
微信、腾讯视频、QQ 音乐、腾讯新闻都在用。拦掉之后：

- 好处：这些 App 的广告也一并少了；
- 代价：如果你自己就是**广告主**，`e.qq.com` / `l.qq.com` 这类**投放后台的曝光点击上报**也会被拦，
  可能影响你自己看投放数据。

真是广告主的话，把 `e.qq.com` 和 `l.qq.com` 那两行注释掉即可。

---

## 五、预期效果 & 怎么验证

**合理预期**：

- 开屏广告不再出现 → 点开 QQ 直接进主界面，这是最直观的体感提升；
- QQ 空间 / 看点的信息流广告减少 → 广告图片不再下载，也就不进图片解码缓存；
- 省流量；
- 但**QQ 的常驻内存不会因为这个模块下降**。那是聊天库和 WebView。

**怎么验证**：

1. 请求记录 → 打开 QQ → 确认 `adsview.qq.com`、`adsmind.*`、`gdt.qq.com` 显示 `REJECT`；
2. 主观：开屏广告是不是没了、空间刷起来是不是干净了；
3. 对比开启前后「设置 → 通用 → iPhone 储存空间」里 QQ 占用的增长速度。

---

## 六、App 内设置：真正有效的那一半

模块只能省广告那部分。**QQ 的内存大头在 App 内部**：

### 1. 清缓存

> 我 → 设置 → 通用 → 存储空间 → 清理缓存

QQ 的图片、视频、表情、空间缓存都在这里，占用经常是几个 GB。

### 2. 关掉不用的入口 —— 减少 WebView 和预加载

> 我 → 设置 → 辅助功能 → 主页底部导航栏设置

把不用的入口隐藏掉（看点、小世界、频道、直播等）。
每关一个，就少一个常驻的 WebView 和一份数据预热 —— **这对内存的帮助比拦广告大**。

（QQ 的设置路径随版本会变，找不到就在「设置」里翻一下。）

### 3. 系统层

- **iOS 设置 → 通用 → 后台 App 刷新 → 关闭 QQ**。
  QQ 的消息推送走 APNs，关掉不影响收消息，但能减少后台驻留和预加载。
- **iOS 设置 → QQ → 照片 → 选「无」或「选中的照片」**，别给「所有照片」。
- **定期彻底退出 QQ**（上滑卡片杀掉）再打开。
  这一步专门清掉 **WebView 进程和 TBS 内置浏览器的残留** ——
  模块完全碰不到这块，但用久了它往往是内存涨得最凶的部分。

---

## 七、风险清单

| 风险 | 说明 | 缓解 |
|---|---|---|
| 规则不生效 | 被更靠前的宽泛规则抢先命中 | 用请求记录确认，调整顺序 |
| 影响腾讯其他 App 的广告 | `gdt.qq.com` 等是腾讯广告体系公共域 | 这通常是你想要的；不想要就注释掉 |
| 影响自己的广告投放后台 | 广告主看数据可能受影响 | 注释掉 `e.qq.com` / `l.qq.com` |
| 个别页面异常 | 极少数页面若依赖广告域做初始化可能异常 | 关掉模块开关即可完全恢复 |
| 埋点拦截 | 个别上报失败重试可能略增后台活动 | 注释掉埋点那一段 |

**回滚**：关掉模块开关即可，没有改任何持久状态。

---

## 八、域名清单来源

| 来源 | 提供的条目 |
|---|---|
| [217heidai/adblockfilters · adblockdnslite](https://github.com/217heidai/adblockfilters) | 全部 22 条 `*.qq.com` 广告域、`gdt.qq.com`、`pgdt.gtimg.cn` |
| [217heidai/adblockfilters · AdGuard Mobile Ads filter](https://github.com/217heidai/adblockfilters) | `pgdt.gtimg.cn`、`e.qq.com`、`mi.gdt.qq.com` |
| [blackmatrix7 · EasyPrivacy](https://github.com/blackmatrix7/ios_rule_script) | `beacon*.qq.com`、`pingjs.qq.com`、`sngmta.qq.com`、`mtrace.qq.com`、`trace.qq.com`、`statistic.qzone.qq.com`、`snowflake.qq.com`、`gdt.qq.com` |
| [blackmatrix7 · Advertising](https://github.com/blackmatrix7/ios_rule_script) | 腾讯系广告域交叉核对 |

App 标识核对自 App Store（`QQ`，id `444934666`，bundle id `com.tencent.mqq`）。

> 补充说明：社区里**没有**专门给 QQ 主 App 做的代理模块（只有 QQ 音乐、QQ 浏览器、
> QQ 钱包的）。所以这份清单是我从上面几个长期维护的广告 / 追踪规则集里
> 逐条提取并交叉核对出来的，每条都能在来源里找到对应。

---

## 附：这个模块系列

同系列（结构、规范、CI 完全一致）：

- [wechat-lite](https://github.com/k6nfmm7dbr-commits/wechat-lite) —— 微信去广告 / 降内存
- [douyin-lite](https://github.com/k6nfmm7dbr-commits/douyin-lite) —— 抖音去广告 / 降内存
- [ios-lite](https://github.com/k6nfmm7dbr-commits/ios-lite) —— 全系统通用广告 / 追踪拦截
- [hongguo-lite](https://github.com/k6nfmm7dbr-commits/hongguo-lite) —— 红果短剧去广告
- **qq-lite** —— QQ 去广告（本仓库）

> 五个里只有 `wechat-lite-plus`、`douyin-lite-plus`、`hongguo-lite-plus` 需要 MITM。
> **不要同时开多个** —— 每条 HTTPS 连接解密都要算 CPU，对性能是实打实的负担。
> `ios-lite` 和 `qq-lite` 都是零 MITM，可以放心常开。

---

## License

[MIT](LICENSE)
