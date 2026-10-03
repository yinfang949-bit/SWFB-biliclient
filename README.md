# SWFB-biliclient
notice.md（确信）
- 原项目 **BiliClient**（包名 `com.SWFB.biliclient`）及其全部原有代码、资源、图标、   文案的著作权，归 **RobinNotBad** 与各原始贡献者所有。 - 本仓库**只包含补丁（Patch）文件**，不包含原项目的完整源码，也不主张对原项目任何权利。（详细信息见notice.md）

- > 📌 **请先阅读 [NOTICE.md](NOTICE.md)**（著作权归属、责任范围与第三方许可说明）

# BiliClient 魔改补丁：播放加速 + 推荐精准化

纯 smali 层改造，**无删代码、可整段回退**。
适配 `minSdk 14`（Android 4.4.4 可用）。

---

## 一、改动概览

| 模块 | 文件 | 改动性质 |
|---|---|---|
| A 视频加速 | `PlayerApi.smali` + 新建 `PlayerApi$ProbeTask.smali` | 新增 4 个方法 + 1 个内部类 |
| B 推荐精准化 | `RecommendApi.smali` 的 `getRecommend` | 纯追加 7 个请求参数（+44 行） |

对应目录（放入工程时）：
`smali/com/RobinNotBad/BiliClient/api/`

---

## 二、功能与原理

### A｜多 CDN 并发测速择优（≈ BTR）
B 站 DASH 流的每个片段都带 `baseUrl` + 多个 `backupUrl`，原客户端只用第一个。
改造后：把全部候选 URL 交给线程池**并发探测**，各测 64KB 下载耗时，
`1500ms` 后取**延迟最小**的节点返回，自动绕开慢节点。

核心方法：
- `PlayerApi$ProbeTask`（implements Runnable）：单 URL 测速任务，结果写入 `map`。
- `pickFastUrl(视频/音频)`：把 `baseUrl` + `backupUrl` 合成候选 List。
- `pickFastUrlFromList`：`newCachedThreadPool()` 并发提交，`awaitTermination(1500ms)` 后取最小值。
- `probeUrl`：`Range: bytes=0-65535`、超时 1200ms、读 64×256B、`nanoTime` 测速，失败返回 `Long.MAX_VALUE`。

### B｜补齐官方个性化参数（WBI 推荐）
推荐走官方同款 WBI 接口：
`https://api.bilibili.com/x/web-interface/wbi/index/top/feed/rcmd`

原来只传 5 个参数，服务端“认不出正经客户端”，只按账号大标签推泛热门。
在 `signWBI` **之前**追加 7 个参数、越过服务端识别阈值后，放开**行为级个性化**，
于是推荐变为“最近看过视频的类似视频”，且走纯接口无商业位，比官方客户端更干净。

> ⚠️ 原有 `screen=1100-2056`（非本补丁添加）会向服务端暴露“大屏/平板”特征，
> 触发大屏推荐策略（偏高完播率的中长视频/番剧），**这正是不推竖屏短视频的原因，建议不要改。**

---

## 三、追加参数说明

| 参数 | 值 | 说明 |
|---|---|---|
| `fresh_type` | `3` | 刷新类型 |
| `login_event` | `0` | 登录事件 |
| `fresh_idx` | `1` | 刷新序号 |
| `plat` | `1` | 平台=安卓 |
| `build` | `913000` | **估算值**，非官方真实 build 号 |
| `last_showlist` | `""` | 上次展示列表（静态版留空） |
| `y_num` | `1` | 拉取轮数 |

> `build` 为按 B 站历史规律估算，服务端若不识别可能忽略；当前实测有效。

---

## 四、验证结果

- A：加载 / 拖拽更快，观看无劣化。
- B：实机反馈“推荐明显更符合账号标签”“甚至比客户端还少杂七杂八的内容”。

---

## 五、回退方法

- **A**：删除 `PlayerApi$ProbeTask.smali`；把两处 `pickFastUrl(...)` 调用改回原 `baseUrl`。
- **B**：删除 `getRecommend` 中 `fresh_type` 到 `y_num` 的 7 段 `put`。

---

## 六、免责声明

仅供个人学习 / 研究，请勿用于商业用途。

