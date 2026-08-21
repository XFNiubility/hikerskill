# 已验证实战坑（多站点实测 · 直接规避）

> 来源：山楂影视（签名+多线路+RSA 解密）、ko58、iysdq、NIUMO 等真实源在 App 内逐级调试中验证过的坑。均为**实测有效**的结论，写源时默认套用，避免反复踩。
>
> 与 `checklist.md`（结构性自检）不同，本文件专讲"改哪里、怎么改才能不炸"的落地细节。

## 一、链接级能力接管 `@rule=`（最高优先，万能兜底）

**问题**：列表/搜索卡片点进去有时跳到原网页，或详情空、落空。

**根因**：依赖顶层 `detail_find_rule`/`searchFind` 让引擎路由到详情/搜索，一旦路由上下文丢失或返回为空，海阔就 fallback 把结果当成原网页打开。

**解法**：在**每个卡片/条目自己的 url** 上显式追加 `@rule=` 做强接管，让它和顶层配置无关：

```js
// 列表/搜索卡片 → 接本源 detail
url: g.abs(href) + '#immersiveTheme#@rule=js:$.require("{engine}").detail()'
```

原理：`@rule=` 是海阔的**能力强制接管**。卡片带上它，无论顶层 `detail_find_rule`/`sdetail_find_rule` 怎么配，点击都执行指定 JS，绝不 fallback 原网页。

**约定**：
- 凡 js 引擎统一产出的卡片（首页分区/分类/搜索共用 `parseItems`），url 一律追加 `@rule` 接 `detail`。
- 剧集/播放项用 `play` 接管，**不要**用 `detail`（否则点集数会进详情而非播放）。

## 二、自绘搜索框确认跳转，必须 `@rule` 接本源 search

**问题**：首页用 `col_type:'input'` 自绘搜索框，点搜索后跳去海阔自带搜索、丢本源。

**解法**：input 回调里把关键词写入 myVar，返回 `@rule` 接本源 search：

```js
// 正确
return 'hiker://empty#noLoading##noHistory#@rule=js:$.require("{engine}").search()';
// 错误（调海阔系统级搜索，丢本源 / 搜不到）
// return 'hiker://search?s=' + encodeURIComponent(kw);
```

配套：回调里 `putMyVar('kw', 输入词)`，`search()` 里读 `getMyVar('kw') || (typeof MY_KEYWORD!=='undefined'?MY_KEYWORD:'')`。

## 三、多线路播放：区分"直链"与"需解析"线路

**问题**：一个影片返回多条线路（如 `SZYS / duanju / rose / YYNB / youku / co_egg`），但多数要么是内部编码、要么解析服务到期，只有少数能直链播。盲目逐条请求浪费大量时间、最后只出 1 条还被误以为 bug。

**实测结论（山楂影视）**：
- 直链线路：url 字段直接是 `https://...m3u8` 或 `.mp4`，点播即 `#isVideo=true#`，**无需解析**。
- 内部编码线路：url 是 `AR8A...`/`YYNB-...`/`co_...` 一类的私有编码串，无可用域名前缀可还原 → 直接跳过。
- 第三方解析线路：url 指向别的解析服务（`api.php` 等），需二次请求解析，且可能"已到期"。

**约定**：`play()` 逐线路判断——直链 http 开头直接进播放列表；`duanju/YYNB/co_egg` 类无域名内部编码跳过；第三方解析失败给出明确提示，不要静默失败。命中多条才返回 `{urls,names}` 供线路切换。

## 四、总结：为什么 prompt 长篇不如预置参数能奏效

一句话：**"会跑"是基础分，"不跳网页 / 能播 / 不白等"是满分。** `@rule` 接管解决"跳网页与详情落空"，多线路判断解决"能播与白等"，本文件就是这两个满分点的落地清单。