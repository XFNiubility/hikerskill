# 《青豆剧场》聚合源架构剖析与写源范式速查

> 背景：`青豆剧场.rule.json`（作者"不知道叫啥好"，1.4MB，44 个子页面）是一个极其完善的视频聚合源，涵盖豆瓣聚合首页、详情、搜索、多源聚合播放、网盘解析、自定义规则、弹幕、登录、验证码等。此文按"写源时要用到的范式"而非"按页面"组织，便于迁移到任意新源。

## 0. 整体架构（怎么读一个大源）

```
rule.json (顶层)
├── find_rule      首页：js: 版本检查 → 首启伪验证 → $.require('function') 组装 → addFilterOptions 过滤按钮 → 轮播 → GetDatas(endpoint)
├── search_url     hiker://empty?word=**&page=fypage  (页面式搜索入口)
├── searchFind     转发 $.require('search').searchMovie(...)
├── detail_find_rule  ""（详情大多从首页/搜索经 extra 跳转 hiker://page/movieDetail）
└── pages[]        4 类子页面
    ├── 功能模块:  function(核心引擎252KB)、search、movieDetail、lunbo、getDouList、nowplaying
    ├── 网盘模块:  panLib(公共库)、baidu/quark/uc/xunlei/pan123/guangya/yidong/tianyi Pan、各自 Login
    ├── 自定义规则: customRule(194KB)、parse(青豆管理)、addrule、addparse、import、doc(写源说明)、SKILL
    └── 工具脚本:  gzip.js、hikerPop.js(UI)、quickJsAdapter(QuickJS)、danmuNodeService.js、zhiyuan(验证码)
```

关键架构决策：
- 顶层所有解析字段只做**一行转发**到 `pages` 里的模块（`$.require('module').fn()`），代码集中管理。
- 用 `index.js`/`gzip.js` 这类 `path: 'xxx.js'` 的子页面传纯工具代码，用 `$.require('xxx.js')` 引用。
- `$.require('page')` 按子页面 `path` 定位，返回该页 `$.exports` 的对象。

## 1. 豆瓣 API 调用封装（写"信息型"源最常用）

`function.getDoubanRes` 是所有豆瓣子页面的数据源，内置**签名**、**UA 伪装**、**失败重试**：

```js
// 调用（绝佳范式：带 apikey + HMAC-SHA1 签名 + 官方客户端 UA + 限流重试）
function getDoubanRes(url) {
  var cc = null;
  if ($.type(url) == 'object') { cc = url.cc; url = url.url; }   // 支持 {url,cc} 重试传参
  var _url = url;
  if (url.indexOf('apikey') === -1)
    _url += (url.indexOf('?') === -1 ? '?' : '&') + "apikey=0dad551ec0f84ed02907ff5c42e8ec70";
  var _ts = parseInt(Date.now() / 1000).toString();
  var m = 'GET&' + encodeURIComponent(url.split('douban.com')[1].split('?')[0]) + '&' + _ts;
  eval(getCryptoJS());
  var _sig = hexToBase64(CryptoJS.HmacSHA1(m, 'bf7dddc7c9cfe6f7').toString());
  _url += '&_sig=' + _sig + '&_ts=' + _ts;
  var s = fetch(_url, {
    headers: { "User-Agent": "Rexxar-Core/0.1.3 api-client/1 com.douban.frodo/7.9.0.beta2(215) ..." },
    method: 'GET'
  });
  var json = JSON.parse(s);
  if (json.localized_message) {                 // 限流/超时
    cc = cc == null ? 1 : cc + 1;
    if (cc > 2) throw new Error('超时');
    java.lang.Thread.sleep(2000);
    return getDoubanRes({ url: url, cc: cc });
  }
  return json;
}
```

要点：`eval(getCryptoJS())` 注入加密库、`java.lang.Thread.sleep` 同步延时、递归重试。任何需要**签名 + UA + 限额**爬官方 API 的源都可照搬此骨架。

## 2. 缓存封装（避免重复请求、抗流量）

三套缓存各有用处：

```js
// ① 简单内存缓存 cachedFetch —— 首页推荐这类"短频"数据
function cachedFetch(cacheKey, id, fetchFn) {
  var CACHE_EXPIRY = 30 * 60 * 1000;
  var cached = getMyVar(cacheKey, '');
  if (cached) {
    try { var data = JSON.parse(cached);
      if (data.id === id && data.ts && (Date.now() - data.ts < CACHE_EXPIRY)) return data.data; } catch(e){}
  }
  var result = fetchFn();
  if (result) putMyVar(cacheKey, JSON.stringify({ id: id, ts: Date.now(), data: result }));
  return result;
}

// ② 文件级分层缓存 getLayerCache/setLayerCache —— 长 TTL + GZIP 压缩（>512B 自动压缩）
function setLayerCache(layer, cacheKey, data) {
  // hiker://files/cache/{qd_search_agg_|qd_detail_|qd_parse_}*.json
  // JSON.stringify → len>512 ? gzipCompress : 原文 → saveFile
}
// ③ AggCacheManager —— 聚合结果缓存（pageId 化的结果 + 合并结果，双写 myVar 与 secondary cache）
```

## 3. 首页过滤按钮范式（交互最强的首页）

`addFilterOptions(d, category)` 展现了完整的状态化过滤 UI。核心三要素：

```js
// A. 静态单选按钮（scroll_button），lazyRule 写状态 + 刷新
d.push({
  title: isSelected ? '““””<b> <font color="#FF5D50">' + title + ' </font></b>' : title, // 选中高亮
  col_type: 'scroll_button',
  url: $('hiker://empty#noLoading##noRecordHistory#').lazyRule((val) => {
    putMyVar('filter_local', val);   // 或 clearMyVar(...) 表示"全部"
    refreshPage();
    return 'hiker://empty';
  }, locals[i])
});

// B. 自定义输入项（input），校验 + 写状态 + 刷新
url: $('hiker://empty#noRecordHistory##noHistory#').lazyRule(() => {
  return $(custom_local, '直接输入具体地区,例如中国大陆').input(() => {
    if (input.trim() !== "") { putMyVar('custom_local', input.trim()); putMyVar('filter_local', '自定义'); }
    else { clearMyVar('filter_local'); clearMyVar('custom_local'); }
    refreshPage();
    return 'hiker://empty';
  })
});

// C. 分类导航（icon_5 顶栏），切换时清空全部过滤
url: $('#noLoading##noRecordHistory#').lazyRule(function(param) {
  putMyVar('fyclass', param); putMyVar('fypage', '1');
  clearMyVar('filter_local'); clearMyVar('filter_type'); clearMyVar('filter_year');
  clearMyVar('filter_platform'); clearMyVar('filter_score'); clearMyVar('filter_rank');
  refreshPage(true);
  return 'hiker://empty';
}, btn.title)
```

## 4. 数据组装 GetDatas（home find_rule 的核心）

```js
function GetDatas(keyword) {
  var page = parseInt(MY_PAGE) || 1;
  // 读过滤变量，拼 API（含 start=(page-1)*20、tags、sort、score_range、playable）
  var url = buildDoubanUrl(keyword, [actual_type, actual_local, actual_year, filter_platform]);
  // 缓存 key 必含所有过滤维度，确保不同筛选不串缓存
  let cacheKey = 'home_' + keyword + '_' + page + '_' + actual_local + '_' + actual_type + '_' + actual_year + '_' + filter_platform + '_' + filter_score + '_' + filter_rank + '_' + filter_content;
  let response = cachedFetch(cacheKey, url, () => getDoubanRes(url));
  // 结果字段：title/img(+@Referer)/id/col_type/source(评分)/doubanId/target_type/year
  // 剧集需 filter(isAnimation) 排除动画
}
```

## 5. 二级/详情与参数传递范式

`extra` 是跨页传参主轴。首页/搜索结果项里写好 `extra`，子页面用 `MY_PARAMS.xxx` 读。

```js
// 首页项
{ url: 'hiker://page/movieDetail#noHistory##immersiveTheme#',
  extra: { id: item.subject.id, title: item.subject.title, target_type: item.subject.subtype } }
// 子页面读参
let json = cachedFetch('qqxq_datas', MY_PARAMS.id,
  () => getDoubanRes('https://frodo.douban.com/api/v2/' + MY_PARAMS.target_type + '/' + MY_PARAMS.id + '?event_source=search'));
// 头部条（用 movie_1_vertical_pic_blur；intro 用 setDesc 自动折行）
d.push({ title: json.title + '/' + dbpf + '上映时间：' + (json.pubdate||json.year),
  desc: json.card_subtitle,
  pic_url: (json.pic ? json.pic.large : '默认海报') + '@Referer=https://m.douban.com/',
  col_type: 'movie_1_vertical_pic_blur', url: 'hiker://empty' });
setDesc(d, json.intro.replace(/\r\n|\r|\n/g,'<br>'), 100);
```

详情小菜单跳转额外页面（演职员/剧照/剧评/预告）用 `hiker://page/xxx` + `extra` 携带 id/type/url。

## 6. 选集与播放解析范式（核心中的核心）

`bocaiDetail`（77KB）是最大的播放页。理解它 = 理解海阔播放体系。

### 6.1 剧集/线路数据结构
- 每集保存为 `name$url`，多集用 `#` 分隔：`epUrls.join('#')`；多源用 `||` 分隔。
- 播放链接常是 `剧名/集数$播放链接` 组合。

### 6.2 播放地址分流（成熟源的标准判断链）

```js
// pan:// 网盘跳转
if (typeof playUrl === 'string' && playUrl.indexOf('pan://') === 0) return fn.resolvePanJump(playUrl);
// JSON 对象/数组
if (playUrl && typeof playUrl === 'object') {
  if (playUrl.url !== undefined) playUrl = playUrl.url;
  else if (Array.isArray(playUrl.urls)) { for (u of playUrl.urls) vUrls.push(u); continue; }
}
// 各协议/格式
if (/^video:\/\//i.test(p) || /#isVideo=true#/.test(p)) vUrls.push(p);
else if (/\.m3u8|\.mp4|\.flv/i.test(p)) vUrls.push(p + '#isVideo=true#');
else if (/^https?:\/\//i.test(p)) vUrls.push('video://' + p + '#isVideo=true#');
else vUrls.push(p);
```

### 6.3 多源合并结果

```js
// 按类型分组：官方源 / 各规则源(按 ruleName) / 菠菜源
buildCategories(results);   // 返回 [{name, results}]
// 最终输出播放对象（支持 urls+names+danmu+headers）
var result = { urls: vUrls, names: sNames };
if (dmPath) { result.danmu = dmPath; putVar('dm_share', dmPath); }
return JSON.stringify(result);   // 多线路播放
```

### 6.4 懒解析 lazyRule（点击才解析真地址）

```js
url: $(playLink).lazyRule(function(dId, sTitle, eIdx, eTitle, pLink, movieTitle) {
  fn.setProgress(dId, { sourceTitle: sTitle, episodeIndex: eIdx, episodeTitle: eTitle, playLink: pLink });
  return fn.playBocaiEpisode(dId, sTitle, eIdx, eTitle, pLink, movieTitle);
}, obj.id, activeLine.name, originalIndex, epName, playLink, obj.title)
```

### 6.5 聚合多线程 + 15s 兜底

```js
var _bcAggThreads = [], _bcAggTimer=null, GLOBAL_TIMEOUT = 15000;
function onAllComplete(){ if (completedCount.incrementAndGet() >= totalCount) doMerge(); }
var safetyTimer = _bcAggTimer.schedule(new java.util.TimerTask({ run:function(){ doMerge(); } }), GLOBAL_TIMEOUT);
addListener('onClose', $.toString(()=>{ /* cancel timer; interrupt threads */ }));
```

## 7. 网盘源范式（panLib + 各 Pan 模块）

几乎所有网盘源遵循同一骨架，写新网盘源照填：

### 7.1 公共库 panLib 提供
`CookieManage`、`httpRequest`(链式)、`generateRandomUUID`、`bytesToSize`、`formatTimestamp`、`sortSx`、`matchBaiduLink`、各 `panPlayModes`/`sortOptions`、`ORIGIN_KEYS`(登录态 key 映射)、`getCookie/setCookie/removeCookie`、`validateAll`。

### 7.2 标准 openShare 骨架（所有网盘共用）

```js
function openShare(shareUrl, pwd) {
  var parsed = parseShareUrl(decodeURIComponent(shareUrl || ''));
  if (pwd && !parsed.sharePwd) parsed.sharePwd = pwd;
  if (!parsed.shareKey) { setResult([{title:'未识别到XX分享链接', url:'hiker://empty', col_type:'text_center_1'}]); return; }
  if (!ensureAuth()) { setResult([{title:'XX未登录', desc:'请在设置页登录', col_type:'text_center_1'}]); return; }
  listShare(parsed.shareKey, parsed.sharePwd || pwd, 0);
}
```

### 7.3 分享链接解析 parseShareUrl（通用正则模板）

```js
function parseShareUrl(url) {
  var shareKey='', sharePwd='';
  var keyPatterns = [
    /\/\/[^\/]*域名(?:pan|865)?\.(?:com|cn)\/s\/([a-zA-Z0-9_-]+)/i,
    /域名\.(?:com|cn)\/share\/(?:view\/)?([a-zA-Z0-9_-]+)/i,
    /[?&](?:sharekey|key)=([a-zA-Z0-9_-]+)/i
  ];
  // ...遍历取 key (含 '.',截断到 '.')
  var pwdPatterns = [ /[?&](?:密码|code|pwd|key)=([a-zA-Z0-9]{4})/i, /提取码[^\w]*([a-zA-Z0-9]{4})/,
                       /提取码/访问码[^\w]*([a-zA-Z0-9]{4})/ ];
  return { shareKey, sharePwd };
}
```

### 7.4 需要提取码时弹输入框

```js
url: $(m.code || '').input(function(m){ $.require('guangyaPan').openShare(m.url + ' 提取码:' + input, input); }, m)
```

## 8. 自定义规则系统（customRule + parse + import）

- `customRule`：规则 CRUD（`getRuleList/saveRule/removeRule`）、状态管理（`getSourceState/updateSourceState`）、规则校验、TVBox 配置导入、QuickJS/drpy/CatVod 适配（`quickJsAdapter.js` 的 `DrpyAdapter`）。
- 文件存储路径常量：`RULE_DIR = hiker://files/rules/青豆剧场/custom_rules/`、`RULE_INDEX = index.json`。
- 每种解析类型：`X`=WEB嗅探、`Z`=JSON直链、`J`=JS嗅探。
- `import`：识别 `var Rule={}` / `var parse={}` / TVBox `{"sites":...}` / `青豆(解析|采集|规则|接口)￥` 管道 / URL 多重格式 → 分发到对应导入函数。
- 导入确认 UI：`qdGetImportPlan` 对比 name/host/api 判定 add/overwrite → `qdShowRuleImportConfirm` 展示确认卡（confirm 按钮）→ `qdDoImport`。

## 9. 工具脚本（gzip / hikerPop / QuickJS / 验证码）

| 模块 | 作用 | 关键 API |
| --- | --- | --- |
| `gzip.js` | 纯 Java GZIP 压缩/解压 | `zip`/`unzip`（通过 `$.require('gzip.js')`） |
| `hikerPop.js` | Android 原生 UI（弹窗/输入/选择/弹幕/滑块） | `selectBottom`/`inputConfirm`/`showTooltip`/`toastMeg`/`confirmSync` |
| `quickJsAdapter.js` | QuickJS 引擎封装，兼容 drpy2.js / CatVod | `DrpyAdapter`、`setWrapMode(1)`、依赖注入 `libDirs` |
| `danmuNodeService.js` | 后台 Node 项目启停（跑弹幕服务） | `available/start/ensureRunning/stop/isRunning` |
| `zhiyuan` | 伪验证码（GitHub 拉题 + x5 网页选图） | `getChallengeHtml()` |
| `lunbo` | 首页轮播横幅（`registerTask` 定时切换） | `banner(title,start,arr,data,cfg)` |

## 10. TMDB 详情（第三方数据源接入范式）

```js
// 多候选 API 域名 + token 鉴权 + 缓存
var TMDB_API_CANDIDATES = ["https://api.themoviedb.org","https://api.tmdb.org","https://tmdb.nastool.org"];
var detailUrl = tmdbProxy + mediaType + '/' + tmdbId + '?language=zh-CN&include_image_language=zh,en,null&append_to_response=images';
var detailRes = fetch(detailUrl, { headers: {'Authorization':'Bearer '+tmdbToken, 'Content-Type':'application/json'}, timeout: 8000 });
// 图片多代理 fallback：TMDB_IMG_PROXIES 数组 + onerror 轮换脚本
// 播放/剧集：runIndex = 依据 tmdb season/episode，或调 AI(Spark) 做剧集绑定（temperature 0.1, 只返回 JSON）
```

## 11. 首页轮播 lunbo 的 registerTask 定时切换

```js
registerTask(id, time, $.toString(function(obj, id){ updateItem(id+'bar',{title,img,url}); }, obj, id));
```

## 12. 页面标识与 longClick 常见组合

```js
extra: { longClick: [{ title:'🛠️ 管理', js: $.toString(()=> "hiker://page/parse#noHistory##noRecordHistory#") },
                      { title:'⭐ 收藏', js: $.toString(()=> "hiker://collection?rule=青豆剧场") },
                      { title:'🔍 搜索', js: $.toString(()=> "hiker://page/bocaiSearch#noHistory##noRecordHistory#") }] }
```

## 一句话总结迁移清单

写一个新源时，从本文取这份骨架：
1. `find_rule` → 版本检查 → `$.require('function')` → 过滤按钮(addFilterOptions) → 轮播 → `GetDatas`。
2. `search_url` = `hiker://empty?word=**&page=fypage`；`searchFind` 转发 `search.searchMovie`。
3. 结果项 `extra` 带 `{id,title,type}` → `hiker://page/xxx`。
4. 详情页收 `MY_PARAMS` → 封装的 API 客户端(签名/UA/重试) + 缓存 → 海报卡+setDesc简介+小菜单 → 选集懒解析。
5. 播放链接统一走 `video://`/`#isVideo=true#` 分流，多源返回 `{urls,names,danmu}`。
6. 网盘源：openShare → parseShareUrl → listShare → lazyRule 播放。