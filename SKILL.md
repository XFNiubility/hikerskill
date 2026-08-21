---
name: "haikuo-miniapp"
description: "帮助 AI 编写海阔视界/嗅觉浏览器（HikerView）的规则型小程序（rule.json）。适用场景：用户要求为海阔视界写规则/写小程序、给某个网站做解析或播放源、制作视频/音频/漫画/阅读等类型的小程序。"
version: "v17"
---

# 海阔小程序开发（HikerView Mini-program）

海阔视界（HikerView，含同类应用嗅觉浏览器）通过"规则"（一个小程序 = 一份 `rule.json`）来包揽几乎所有解析能力：首页列表、搜索、二级/选集页、子页面、播放。开发者并不写原生 App，而是写"解释型规则"（选择器规则或用内嵌 JS 解析引擎），并返回标准数据结构由 App 渲染。

## 何时使用本 skill

当用户要"写一个海阔/海阔视界的小程序 / 规则 / 源"，或要给某个网站做视频、音频、漫画、阅读等解析时使用。产出物是一份 `rule.json`（可独立分享，通常以 `.txt` 或 `.json` 保存），复杂规则可再打包成 `.hk小程序.zip`（内含 `rule.json` 及 `libs.zip`/`data.zip`/`require.json`）。

## ⚠️ 核心原则（最重要！每一份源都必须遵守）

1. **一切以 App 实测为准，模拟/语义校验只是参考**：本 skill 不内置 Python 模拟器，jsoup/jsoup 语义校验通过≠App 命中。**规则最终要在海阔视界 App 里真实跑一遍**（首页→分类→搜索→选集→播放），命中 0 就参考 `实战高频坑` 换写法（典型：`pdfa` 取属性失败改正则）。
2. **先抓真实 HTML/接口，再写选择器，禁止臆造**：写任何规则前，必须基于目标站点真实返回确认结构。没有真实 HTML 前不写选择器；不编造站内不存在的 class/接口/替换词。
3. **不凭记忆拼凑，优先复用已验证范式**：能套 `references/examples/advanced-source-template.md` 或 `production-patterns.md` 的骨架就整体复制再填充，不要每源从零手写。
4. **逐级验证才交付**：首页 → 分类/过滤 → 搜索 → 详情 → 选集 → 播放，每一级确认再进下一级，用 `log()`/`setError` 埋点定位。

> 自检入口：交付前逐项过一遍 `references/checklist.md`。

## 核心工作流（三阶段 + 闸门）

**阶段一 收集信息——只查、只抓、只分析，不许写规则。**（对应步骤 1–2）
**阶段二 严格审查——按范式编写 + 基于真实结构校验，不许直接交付。**（对应步骤 3–5）
**阶段三 交付——拼装输出 + 逐级调试 + 自检 + 给出导入口令/文件。**（对应步骤 6–7）
> 闸门：任何阶段都不能跳过"先抓真实数据/先校验"直接给最终 JSON；写完必须能说明每个选择器/字段对应的真实 HTML 位置。

1. 明确类型与需求：确认解析对象是 视频/音频/漫画/资讯/阅读/直播 中的哪一种，目标网站、分类、搜索、是否有分页、是否需要动态解析、是否需要登录/Cookie。**先实际抓取目标页面 HTML 确认结构，再写选择器**，禁止凭空臆造选择器。
2. 判断复杂程度并选起点：若目标需要 **聚合多源 / 网盘解析 / 信息型详情(签名API+缓存+过滤) / 登录加密**，则**默认整体复制 `references/examples/advanced-source-template.md` 的统一引擎骨架**到子页面再填充；普通列表/选集源按下方 Schema + `production-patterns.md` 的模块化单例搭。
3. 搭骨架：按下方 `rule.json` 结构先写清首页规则 `url` + `find_rule` + `class_name`/`class_url`（分类）、`search_url` + `searchFind`、`detail_find_rule`（选集/二级）、`sdetail_find_rule`（三级级）。
4. 优先用选择器语法写首页/搜索/选集；无法用选择器表达时（JSON 接口、加密、需要循环/IO），用 `js:` 前缀写 JS 解析规则，输出用 `setResult(d)`。
5. 处理链接：播放/详情链接按需加 header、多线路 JSON、动态解析 `@rule=`/`@lazyRule=`、页面标识 `#xxx#`。
6. 用 `hiker://debug?url=xx` 或直接在 App 中逐步调试：先能出首页 → 分类 → 搜索 → 选集 → 播放。逐级验证，不要一次写完就交付。
7. 交付时给出可直接导入的口令或文件，说明规则名、分组、作者。

## ⚠️ 关键补充：布局与渲染（多数新手源缺失的部分）

**只写"能跑"的规则远远不够。** 从 2070 个真实源分析看，成熟源的核心差异在于**界面布局**——你写的页面应该长什么样，而不是能不能解析数据。

**默认优先套可复用渲染模板 `UILib`（`references/ui-template.md`）：写任何源之前先决定是否用 UILib。** 若页面基本落在 UILib 覆盖范围（首页/详情/搜索、折叠筛选、轮播、快速搜索、TVBox 推送、线路/选集/排序、简介折叠），则先读 `ui-template.md`，按"数据层只填数据契约 → `$.require('uil').renderXxx(data)`"装配，**不要手写排版权且不要看 layout-design 照抄**；只有页面不在 UILib 范围时才回 `references/layout-design.md` 按需手写。两者分工：

- **`ui-template.md`** — 渲染层默认实现（UILib 引擎 + 数据契约 + 顶层字段固定配方 + 数据层适配规范）
- **`layout-design.md`** — 布局原理/概念参考（col_type 决策树、美化技巧、可选手写模板），供 UILib 未覆盖场景按需查阅

布局原理该文档覆盖（UILib 已封装常见项，其余场景仍可参考）：

- **首页布局**：分类导航栏 → 搜索框 → 过滤按钮 → 轮播 → 分区标题 → 内容列表，每一步选什么 col_type、怎么写
- **详情页布局**：海报卡（`movie_1_vertical_pic_blur`）→ 简介（`setDesc` 可展开/收起）→ 小菜单 → 线路切换 → 剧集列表（动态选 `text_2`~`text_5`）
- **col_type 选择决策树**：根据数据量、图片质量、内容类型选合适的样式
- **页面美化**：文字颜色/加粗/高亮、背景渐变、间距控制、分割线、选中态反馈
- **完整布局模板**：可直接复制的首页和详情页完整代码骨架
- **常见缺陷修复**：首页空洞、详情缺海报卡、集数列数固定、选中无反馈、简介不能收起、图片防盗链

**核心原则：先设计布局（或用 UILib 选好数据契约），再写解析代码。** 规划好每个页面的元素排列顺序，再填充数据逻辑。

## rule.json 顶层字段（Schema）

核心字段如下（参考官方模板与通用结构），"页数/分类/年代/地区/排序"替换词与页面规则密切相关：

| 字段 | 含义 |
| --- | --- |
| `title` | 小程序名称（决定分享/导入名称） |
| `author` | 作者 |
| `version` | 版本号（整数），更新时递增 |
| `type` | 类型：`video`/`music`/`cartoon`/`news`/`read`/`live` 等 |
| `group` | 分组，如 `AI生成`/`测试`（决定在哪个分组下显示；AI 写源默认 `AI生成`） |
| `ua` | 请求 UA：`mobile`/`pc`/自定义字符串 |
| `url` | 首页列表地址。**替换词**：`fyclass`=分类、`fyarea`=地区、`fyyear`=年代、`fysort`=排序、`fypage`=从 1 递增、`fyAll`=任意类型都替换它；分页如 `fypage@-1@*20@` 表示 0,20,40 递增（分页规则不要放 URL 最末尾，末尾需补无效串如 `?_t=0`）；第一页特殊可用 `[firstPage=...]` |
| `class_name` / `class_url` | 分类名称与替换词，多个用 `&` 分隔（名称和地址一一对应、用 `&` 配对） |
| `area_name`/`area_url`、`year_name`/`year_url`、`sort_name`/`sort_url` | 地区/年代/排序过滤器（可选） |
| `col_type` | 首页默认显示样式，如 `movie_3`（详见 col-type 参考） |
| `find_rule` | 首页解析规则（选择器或 `js:`） |
| `search_url` | 搜索地址，用 `**` 作为关键词占位符（与 `%%` 冲突时用 `%%`）；可带 方法/编码/header，见下方"链接增强" |
| `searchFind` | 搜索结果解析规则 |
| `detail_col_type` / `detail_find_rule` | 二级（点首页项后的选集/详情）样式与规则 |
| `sdetail_col_type` / `sdetail_find_rule` | 三级样式与规则（`*` 表示继承上级） |
| `detail_find_rule` / `sdetail_find_rule` | 均支持 `*` 继承 |
| `preRule` | JS 预处理，在首页/搜索/分类规则执行前运行一次。六种用途：远程库加载、initConfig 配置、Cookie/登录/验证码、eval(fetch) 远程执行、confirm/request 依赖检查、动态 UA 设置。详见"preRule 完整指南"节 |
| `pages` | 子页面数组（见下），JSON 字符串 |
| `last_chapter_rule` | 最新章节规则（收藏界面显示"最新章节"用） |
| `icon` | 图标（可填颜色如 `#ff6354ef`） |

### 子页面（`pages`）

`pages` 为字符串化的数组，子页面格式：`{ "name": "页面名", "path": "标识", "col_type": "movie_3", "rule": "js: ..." }`。跳转子页面用 `hiker://page/<path>`；跨程序用 `hiker://page/<path>?rule=<规则名>`。传参两种方式：URL 参数（子页面内用 `getParam('key')` 取），或按钮 `extra` 对象（子页面用 `MY_PARAMS.key` 取，一个参数也要用对象）。

### 链接增强（编码/方法/header）

地址后可按 `;` 增加配置：`网址;POST;GBK;{User-Agent@Windows&&Cookie@id}`。
- 分号后依次为：请求方式（`get`/`POST`）、编码（`GBK` 等）、大括号里的 header。要 GBK 就必须写 get/POST。
- header 值可用 `.js:` 处理，如 `Cookie@id.js:input`；URL 参数值也可 `.js:` 处理。
- POST 传参：body 用两个中文问号 `？？` 分隔原链接 query 与参数；JSON body 用参数 `JsonBody`。
- 图片自定义头：`http://a.jpg@headers={"User-Agent":"Windows","Cookie":"111"}`；`@Referer=@User-Agent=@Cookie=` 为旧用法。
- URL 本身也可 JS：`http://a.com.js:input+'/'?a=b.js:input+'a'`。

## 解析规则（选择器语法）

- 首页/详情格式：`列表;标题;图片;描述;链接`。
- 搜索格式：`列表;标题;链接;描述;详情;图片`（前 3 必选，后 3 可省，省略用 `*` 占位）。
- 每段用 `;` 分隔，元素与属性用 `&&` 分隔：`body&&.list&&li;a&&title;img&&src;.desc&&Text;a&&href`。
- `Text`=取文字，`Html`=取含标签文本，否则默认取属性；`&&`取子元素，`#`取 id，`.`取 class；`--`为排除，如 `body--a&&a&&href`；`&,索引`取非首个子元素，可用负数，如 `body&&a,-1&&href`；支持 jsoup 原生选择器。
- 值后缀 `.js:代码` 可再处理（`input` 为取值结果），多个选择器结果用 `+` 拼接，`.js:`处理最后一个用中文 `＋`。
- 直接用 `js:` 开头写整段 JS 解析更灵活（JSON 接口、加密等首选）。
- 详情规则详细说明见 `references/selector-syntax.md`。

## 显示样式（col_type）

常见：`movie_3`（三列影视，默认）、`movie_2`/`movie_1`/`movie_1_left_pic`、`text_1`~`text_5`、`long_text`、`rich_text`（富文本，阅读正文）、`pic_2`/`pic_3`（图片）、`icon_2`~`icon_5`（图标）、`input`（输入框）、`video`（内嵌视频）、`x5_webview_single`、`flex_button`/`scroll_button`、`card_pic_2` 等。完整清单与 extra 用法见 `references/col-types.md`。

## 播放与详情链接

- 直链、`.m3u8`、`video://`等均可被识别；需要带 header 用 `地址;{Cookie@xx&&Referer@host}`。
- 多线路/字幕/弹幕/歌词：用对象 JSON 字符串作为 url。
- 漫画/多图：`pics://地址&&地址`。
- 动态解析（在二级/首页点击时才取真正的播放地址）：url 后加 `@lazyRule=.js:代码`（`&&` 转义为中文 `＆＆＆＆`），不能用普通选择器嵌套时用 `.js:`。深层嵌套用多个 `@rule=`。
- 详见 `references/link-schemes.md` 和 `references/selector-syntax.md`。

## 内嵌 JS 解析引擎（重点）

以 `js:` 开头的解析规则运行在 App 内置 JS 引擎中。最常用：
- 结果输出：`setResult(d)`（`d` 为对象数组，对象字段 `title`/`url`/`pic_url` 或 `img`/`desc`/`col_type`/`extra`）；可用 `setPreResult()`+`setResult()` 分优先与追加两次返回。
- 请求：`fetch(url, {headers, body, method})`；`post()`；`fetchPC()/postPC()`用电脑 UA；`fetch(url,{withHeaders:true})` 取响应头、`{withStatusCode:true}` 取状态码、`{timeout:5000}`、`{inputStream:true}`；`batchFetch()` 批量；`request()` 同步支持本地文件。
- 解析 DOM：`parseDom(html, rule)`（缩写 `pd`）、`parseDomForHtml`（`pdfh`）、`parseDomForArray`（`pdfa`）、`getResCode()` 返回当前页面源码。
- 常用变量：`MY_URL` 当前地址、`MY_HOME` 主页、`MY_RULE` 当前规则、`MY_PAGE` 页数、`MY_KEYWORD` 搜索词、`MY_PARAMS` 上级传入参数、`MY_TYPE`(`home`/`search`)、`MOBILE_UA`/`PC_UA`、`MY_CLASS_NAME`/`MY_CLASS_URL` 等。
- 编辑码：`base64Encode/Decode`、`encodeStr/decodeStr(str,'UTF-8'|'GBK')`、`aesEncode/aesDecode`、`rsaEncrypt/rsaDecrypt`；加密增强先 `eval(getCryptoJS())`。
- 存储：`putVar/getVar/clearVar`（重启失效）、`setItem/getItem/clearItem`（私有持久化）、`putMyVar/getMyVar`（规则内）、`setPublicItem`（公开）；对象用 `storage0.putMyVar(...)`。
- 网络：`fetchCookie()`、`getCookie(url)`、`registerDNS()`。
- 交互/跳转：`toast('...')`、`setError('...')`（显示报错/调试）、`refreshPage()`、`back()`、`backToHome()`、`copy()`、`showSelectOptions()`、`showLoading()/hideLoading()`。
- 模块：`require()`/`requireCache()`/`requirejs()`（远程代码块）；`initConfig`/`config.xx` 配置管理。
- 完整 API 清单见 `references/js-api.md`。

## preRule 完整指南（预处理规则）

`preRule` 是 `rule.json` 顶层字段，在首页/搜索/分类规则执行前运行一次。**2070 个真实源中 731 个（35%）使用了 preRule**，远不止"取 Cookie"这么简单。以下是基于语料统计的六种真实用途及写法模板。

### 1. 远程公共库加载（最主流，363源/50%）

```json
"preRule": "js:require('https://gitcode.net/xxx/dr/-/raw/master/js/预处理.js')"
```

纯 `require` 一个远程 JS 文件，注入公共能力（加密工具、签名函数、通用解析引擎）。一个库可养几百个源。**多个 require 可串联**：`require('a.js');require('b.js')`。

### 2. initConfig 配置（88源/12%）

注入依赖脚本、模板、主题色：

```js
"preRule": "js:initConfig({依赖: 'https://xxx/srcDyTmpl.js', 模板: 'http://xxx/template.json', 分类颜色: '#1aad19'})"
```

`initConfig` 的 key-value 后续可通过 `config.xx` 读取。常用于模板引擎源提前加载渲染骨架。

### 3. Cookie/登录/验证码（30源/4%）

**自动获取 Cookie 并持久化（标准写法）**：
```js
"preRule": "js:if(!getVar('cookie@xxx')){var res=JSON.parse(fetch('https://xxx.com',{withHeaders:true}));var c=JSON.stringify(res.headers);/(session=.*?);/.test(c);putVar('cookie@xxx',RegExp.$1);}"
```

**验证码 URL 声明**：
```js
"preRule": "js:var 验证码='https://xxx.com/verify/index.html?'"
```

**登录请求**（POST 提交用户名密码，获取 uid/token）：
```js
"preRule": "js:var r=JSON.parse(fetch('https://xxx.com/api/login',{method:'POST',body:'user=xxx&pass=xxx'}));if(r.token)setItem('token',r.token);"
```

**关键原则**：Cookie/token 用 `setItem/getItem` 持久化跨重启，纯会话态用 `putVar/getVar` 即可。

### 4. eval(fetch) 远程执行（27源/4%）

从远程或加密源拉取代码当场执行：

```js
"preRule": "js:eval(base64Decode('bGV0IGxvY2FsID1mZ注意强调前缀固定为 `‘‘’’`（`‘‘’’`，两个左引号+两个右引号），勿用RjaCgnaGlrZXI6Ly9maWxlcy9...'))"
```

```js
"preRule": "js:eval(fetch('hiker://files/rules/xxx/init.js',{}))"
```

```js
"preRule": "js:evalPrivateJS('AAAAY3K5f2...')"  // 内置加密存储的 JS
```

### 5. confirm/request 依赖检查（17源/2%）

检测依赖规则是否存在，缺失时弹窗引导安装：

```js
"preRule": "js:if(request('hiker://home@模板·Q')=='null'){confirm({title:'依赖检测',content:'缺少模板依赖,请导入!',confirm:()=>'海阔视界首页频道规则【模板·Q】￥home_rule_url￥http://xxx/...'})}"
```

```js
"preRule": "js:let r=fetch('hiker://home@DR模板');if(!r||r.length<=5){confirm({title:'提示',content:'本规则依赖DR模板规则',confirm:ru,cancel:ru})}"
```

### 6. 动态 UA 设置（7源/1%）

覆盖默认 UA（非空 `var ua=''` 的那种）：

```js
"preRule": "js:putVar('ua',';get;utf-8;{User-Agent@Mozilla/5.0 (Linux; Android 7.0; wv lite baiduboxapp) ...}')"
```

```js
"preRule": "js:putVar('key','&access_key=xxx');putVar('ua',';get;utf-8;{User-Agent@PC_UA}')"
```

### 其他常见用法（199源/27%）

- **收藏集标记**：`putMyVar('myCollection', MY_RULE.title)` + `putMyVar('remoteUrl', 'hiker://files/rules/xxx/data.js')`，让收藏集工具识别该源。
- **文件操作**：`fileExist('hiker://files/rules/xxx/token.js')` 判存在 → `writeFile()` 清理旧值。
- **模块导入**：`$.require('hiker://page/libs?rule=XXX')` 跨程序引用引擎。
- **全局变量预置**：`var 域名='https://xxx.com';var 分类映射={1:'电影',2:'电视剧'};`
- **数据预取**：`if(!getVar('xxx-datas')){let html=fetch('https://xxx.com/nav.html');let urls=parseDomForArray(html,'a&&href');putVar('xxx-datas',JSON.stringify(urls));}`

### preRule 最佳实践

1. **能用远程库就优先远程库**：`require('https://...')` 一行解决，比内嵌几十行加密工具函数更省事。
2. **需要登录态的源，先判缓存再取**：`if(!getVar('cookie@xxx'))` 避免每次首页都重复请求。
3. **依赖检查放最前面**：第三方模板缺失时弹窗引导，比直接报错友好。
4. **preRule 里不要写复杂 UI 逻辑**：只做数据准备（Cookie/变量/配置），UI 交互放 `find_rule`/`detail_find_rule`。
5. **多个 preRule 行为可串联**：`require('a.js');initConfig({...});if(!getVar('ck'))getCookie();` 一行写完。
6. **跨页面的公共变量用 `putMyVar`**（规则内共享），单次请求的临时变量用 `putVar`。

## 特殊协议链接与页面标识

- 协议：`hiker://home`、`hiker://search?s=xx&group=群号`、`toast://提示`、`javascript:...`（彩蛋模式）、`copy://文本`、`confirm://`、`input://{...}`、`select://{...}`、`editFile://`、`openFile://`、`download://`、`web://`（强制网页）、`x5://`、`x5Play://`、`pics://`、`fileSelect://`、`hiker://debug` 等。
- 动态解析/网页嗅探：`webRule://链接@JS代码`、`x5Rule://链接@JS代码`、`video://网页`（自动提取视频）。
- 页面标识（加在 url 上的 `#xxx#`，请求时会自动清除）：`#noLoading#`、`#noHistory#`、`#noRecordHistory#`、`#immersiveTheme#`、`#fullTheme#`、`#readTheme#`、`#autoPage#`（阅读自动翻页）、`#autoCache#`、`#background#`、`#isVideo=true#`/`#ignoreVideo=true#`、`#isMusic=true#`、`#isM3u8#`、`#noRefresh#`、`#originalSize#` 等。
- 完整参考见 `references/link-schemes.md`。

## 完善级工程模式（模块化单例）

"能跑"的规则把解析代码散落在 `find_rule`/`searchFind`/`detail_find_rule` 各字段里，代码一复杂就难维护、难同步。**成熟的小程序（尤其是达到上传示例那种"完善"程度）都采用"模块化单例"架构**：所有解析逻辑集中封装在 `pages` 里的一个子页面 JS 模块中，顶层各解析字段变成一行入口转发。**写任何逻辑超过一屏的来源，默认就采用这套架构，不要等写成巨型字段再重构。** 详情与完整模板见 `references/production-patterns.md`。

核心要点：

- **一个子页面承载整个引擎**：在 `pages` 中放一个子页面（如 `path: "mdai"`），其 `rule` 是 `js:` 开头的一段代码，定义一个模块对象（如 `const mdai = {...}`）并在末尾 `$.exports = mdai`。工具纯代码可用 `path: "xxx.js"` 的子页面，`$.require('xxx.js')` 引用。
- **入口只留一行转发**：
  - `find_rule`: `js:$.require("mdai").home()`
  - `searchFind`: `js:putMyVar('keyword', getParam('kw'));$.require("mdai").search()`
  - `detail_find_rule` / 子页面：`js:$.require("mdai").detail(...)` 等
  - `$.require("模块名")` 按子页面 path 找到模块并返回 `$.exports` 导出的对象；模块内用 `this` 或 `$.exports` 引用。
- **配置化而非硬编码**：
  - `host()`：用 `getItem('xxx_host', 默认值)` 让用户可改域名，读取时清洗/去尾部斜杠。
  - `pageSize()`：读用户配置并做数值校验与上限钳制。
  - `headers()`：统一返回 `{User-Agent, Referer, Accept}`，所有请求复用，保持一致请求头、降低被封风险。
- **统一的分页逻辑**：用 `MY_PAGE` 计算 offset（`(MY_PAGE-1)*pageSize`），翻页时按需请求并返回下一页结果。
- **健壮性与降级**：主接口解析失败时提供内置 fallback 分类/首页兜底；单条解析失败用 try/catch 隔离而非整页崩掉。
- **复用与可读性**：把"解析列表 HTML/JSON → 结果对象数组"拆成模块内的 `parse` 方法供 home/search/detail 复用；页面统一篇幅、函数职责单一。

## 高阶源能力（聚合 / 网盘 / 信息型源 —— 达到"完善级"的进阶规范）

下面这些范式是《青豆剧场》这类顶级源实际验证过的能力。**当一个源需要聚合多源、网盘解析、信息型详情、登录/加密时，不要从零写——直接整体复制 `references/examples/advanced-source-template.md` 的统一引擎骨架(签名客户端+三层缓存+过滤按钮+详情头+播放分流+网盘openShare一体)到子页面里，再只替换占位符。** 理解每个范式为何这样写看本章；想看在真实大型源里的完整样子看 `references/examples/qingdou-architecture.md`；动手写源时以骨架为默认起点。

### 分级请求封装（默认都要做）
- **统一 API 客户端**：需要签名/UA/限额的官方接口（如豆瓣），封装 `getXxxRes(url)`：内联 apikey + HMAC 签名 + 官方客户端 UA + 失败重试（递归 `{url,cc}` 计数，超限抛错）。支持 `$.type(url)=='object'` 传 `{url,cc}`。
- **三层缓存**：
  - 内存缓存 `cachedFetch(cacheKey,id,fetchFn)`：短频数据，30 分钟过期，`putMyVar` 存 `{id,ts,data}`。
  - 文件分层缓存 `getLayerCache/setLayerCache(layer,key,data)`：长 TTL，>512B 自动 gzip 压缩。
  - 聚合缓存 `AggCacheManager`：聚合结果按 pageId 存过，双写 myVar 与二级缓存。
- **缓存 key 必须含全部过滤维度**（分类/页/地区/类型/年代/平台/排序/评分），否则不同筛选互相串缓存。

### 首页过滤按钮（交互式过滤）
用 `scroll_button` 做选项、`$().lazyRule(...)` 写状态并 `refreshPage()`、`$().input()` 做自定义输入并校验。选中项高亮写法：`title: '““””<b> <font color="#FF5D50">' + v + ' </font></b>'`。分类切换时清空全部 filter 变量。

### 详情页（返回信息卡头，不只选集）
**必须遵循 `references/layout-design.md` 第 3 节的"详情页硬性实施规范"**：返回顺序固定为 `[海报卡(movie_1_vertical_pic_blur，url=封面'#.jpg#')] + [剧情简介(必须用 setDesc：rich_text 内嵌 `<a href='#noLoading#@lazyRule=.js:…'>展开:</a>` 原地展开/收起，见 layout-design 3.3 方式1)] + [小菜单(演职员/剧照/剧评/预告，跳 hiker://page/xxx)] + [线路切换(scroll_button)] + [剧集列表(text_2~text_5 动态)]`。**禁止**用 `long_text` 假装折叠、**禁止**手写无 `href` 的裸 `<a>展开:</a>`；`<a>` 的 href 必须单引号包裹。所有数据经 `MY_PARAMS`/`getParam` 收参 → 封装的 API 客户端(含缓存) → 结构化组装。

### 选集与播放分流（统一判断链）
多源结果返回 `{urls, names, danmu, headers}` 的 JSON 字符串（多线路播放）。真实地址用 `lazyRule` 点击时才解析。统一分流顺序：
1. `pan://` → `resolvePanJump`
2. JSON 对象/数组 `{urls:[...]}` → 拆开
3. `video://`/`#isVideo=true#` → 直接
4. `.m3u8|.mp4|.flv` → 加 `#isVideo=true#`
5. 其它 `http(s)://` → 包 `video://`
剧集保存 `name$url` 用 `#` 分集、`||` 分源；聚合分 `buildCategories`：官方源 / 各规则源(按名) / 网盘源分组。多线程聚合加 15s `TimerTask` 兜底，`onClose` 清理线程。

### 网盘源骨架（多种网盘统一）
明文规则：`openShare(shareUrl,pwd)` → `parseShareUrl`(正则提取 shareKey+sharePwd) → `ensureAuth()`(未登录给提示) → `listShare` → 点击播放。提取码缺失时弹 `$().input()` 让用户补。公共能力抽到 `panLib` 模块（`CookieManage`、`httpRequest`、`ORIGIN_KEYS` 登录态 key 映射、`get/set/removeCookie`），各网盘 `$.require("panLib")` 复用。

### 信息/详情/第三方数据源
TMDB 类第三方：多候选 API 域名数组 + Bearer token + TTL 缓存 + 图片多代理 fallback。剧集绑定可调 AI（仅返回 JSON，temperature 低）。

### 状态与登录态管理
- 页面状态/过滤：`putMyVar/getMyVar/clearMyVar`（会话内）。
- 持久配置/登录态：`setItem/getItem/clearItem`（跨重启）；网盘/账号 token 必须用它，别放仅会话的 myVar。
- 文件配置：`hiker://files/rules/{规则名}/xxx.json`，`readFile/writeFile/saveFile/fileExist`。

## 多类型源速查（视频签名 / 网盘 / 阅读漫画 / 直播 / UI与工具）

视频、音频之外，源还要覆盖**网盘、阅读、漫画、直播、UI 组件、模板引擎、工具**等类型，每类有成套的可复用范式。**完整目录见 `references/examples/multi-type-patterns.md`**，此处给最关键的取舍：

- **视频接口签名三分档**：低档=可信任 `init/login/request` 三件套（POST 表单+按需 token）；中档=RSA 解密对称密钥 + AES-CBC 请求体 + MD5 拼盐签名；高档=Wire 协议（AES-GCM 加密体 + HMAC-SHA256 按行签名 `METHOD\npath\nts\nappId\ndevice\nnonce\nbodySha256` + `X-App-*` 头 + 设备指纹）。**先判断目标接口是哪一档再封装，别一上来就写最高档。**
- **多线路用双数组**：`tabs[]`（线路名）与 `lists[][]`（每线路 `name$url`）索引一一对应；点线路 `putMyVar(线路index)` + `refreshPage(false)`；播放点击时才 `lazyRule` 解析，失败兜底 `toast://解析失败`。
- **网盘三件事：登录态、加解密、分享转存**。登录态四方案（Cookie+WebView / 短信+拼图 / 二维码轮询 / 手动输 token）；加解密四形（CryptoJS-AES / Java-AES随机IV前置 / AES-GCM前12字节IV / Wire直通）；分享链接正则提 key+提取码、缺码弹 `$().input()`、转存两段式（createSaveTask→queryTask轮询）。公共能力抽 `panLib`（`CookieManage`/`httpRequest`/登录态key映射）。失效/需密码/过期必须兜底。
- **阅读(type=read)**：目录页用 `MY_PARAMS.Toc` 模板定位章节，URL 加 `#autoPage##readTheme#`，GBK 时追加 `;get;编码`；正文分段转 `rich_text`，段首补全角缩进。
- **漫画(type=cartoon)**：图片相对路径统一补全（`http`/`//`/`/` 三分支）；章节列表附"下载"入口跳 `hiker://page/download.view`（`chapterList`+`parseCode` 塞 `extra.info`）；图片加密用 JavaImporter `AES/CBC` 解字节流。
- **直播(type=live)**：数据存 `hiker://files/rules/live/config.json`；M3U(`#EXTM3U`) 与 `#genre#` 两套正则 switch；同组同名多条地址/headers 用数组 `concat` 合并。
- **hikerPop / 模板·Q / 工具源**：hikerPop `$.require('hikerPop.js')` 反射 `com.xpopup/ToastMgr/PlayerChooser` 原生 UI；模板·Q 按 HTML 片段关键字匹配模板再 eval（`$.require('hiker://page/auto?rule=XX').autoPage`）；工具/资讯源做带盐 MD5 登录 + `getMyVar('Page')` 做"加载更多"。

口诀：`视频分档封装，网盘三件套，阅读讲编码，漫画补相对路径，直播双正则，工具源带盐登录`。

## 编写规范与良好实践

- **先抓页面再写选择器**：网络变化快，选择器必须基于真实 HTML 校验；建议用浏览器/抓包确认 Class、数据接口。
- JSON 接口优先用 `js:` 解析（`eval('json='+getResCode())` 或 `JSON.parse`），结构清晰不易随页面改版失效。
- 分页、筛选替换词写进 `url`，保持规则简短；能用选择器就别写重 JS。
- 播放记忆：多选集/章节给每项 `extra` 的 `id` 字段设全局唯一值（通常用详情页链接或"规则名+资源ID"），否则多集进度会串。
- 图片防盗链：按需给图加 `@Referer=`/`@headers=`，或对缩略图用目标站 UA。
- 兼容 UA：移动端站用 `ua: mobile`，Web 接口用 `fetchPC`。
- Cookie 类规则把取 Cookie 逻辑放 `preRule`。
- 用 `setError`/`log` 调试；逐级验证（首页→分类→搜索→选集→播放）后再交付。
- 打包与分享：单个规则以口令/文件交付；多文件小程序打好 `.hk小程序.zip`（内含 `rule.json`，可选 `libs.zip`、`data.zip`/`data2.zip`、`require.json`）。

## 通用示例（视频类骨架）

```json
{
  "title": "示例影视源",
  "author": "开发者",
  "version": 1,
  "type": "video",
  "group": "AI生成",
  "ua": "mobile",
  "url": "https://example.com/index.php/vod/show/area/**/class/fyclass/page/fypage.html",
  "class_name": "电影&电视剧&综艺",
  "class_url": "1&2&3",
  "col_type": "movie_3",
  "find_rule": ".hl-list-item&&ul&&li;.hl-item-title&&Text;.hl-item-thumb&&a&&img&&data-original;.hl-item-score&&Text;a&&href",
  "search_url": "https://example.com/index.php/ajax/suggest?mid=1&wd=**;POST",
  "searchFind": "js:var d=[];var j=JSON.parse(getResCode());for(var it of j.list){d.push({title:it.name,url:'https://example.com/index.php/vod/detail/id/'+it.id+'.html'})}setResult(d);",
  "detail_col_type": "text_3",
  "detail_find_rule": "body&&.playslist;;a&&Text;*;*;a&&href",
  "sdetail_col_type": "text_3",
  "sdetail_find_rule": "*",
  "preRule": "",
  "pages": "[]",
  "icon": "#ff6354ef"
}
```

（示例仅演示字段用法，选择器/接口需按目标网站实际情况重写。）

## 实战高频坑（从真实站点调试总结）

以下是写多个实战源（含影视类）时验证过的高频问题与解法，写规则时优先照此规避。

- **列表/资料放 HTML 属性时，`pdfa` 常取不到值，改用正则**。典型：目标站首页列表项形如 `<li class="dx-vod" data-json='{...}'>`（属性值用**单引号**包裹）。此时 `pdfa(html,'li.dx-vod&&data-json')` 极易命中 0。**优先直接用正则提取**：`/<li class="dx-vod"[^>]*data-json='([^']+)'/g` 匹配整个属性值再 `JSON.parse`（先 `replace(/&quot;/g,'"')`）。同理详情页 `<a href="/igokj/xx.html">第01集</a>` 长链选择器也可能全 0，用正则 `<a href="(\/[^"]+)"[^>]*>\s*([^<]*?)\s*<\/a>` 更稳。**判断标准**：jqoup/jsoup 语义校验通过但 App 命中 0 时，就改用正则。
- **`hiker://empty...@rule=js:$.require("模块")` vs 内联 lazyRule，两种"点击后解析"的取舍**：海阔有两种让项被点击后做进一步处理的写法，别混用、别选错。(a) **内联 lazyRule**：`url: $().lazyRule(function(pu){ /* 独立作用域，内联完整逻辑 */ return 真地址+'#isVideo=true#'; }, param)` —— 自包含、直白，但**回调在独立作用域，用不了模块闭包变量**，旧版箭头函数/`$.require('模块').fn` 在回调里经常拿不到 `_H`/`_UA`。(b) **`@rule` 跳模块**：`url: 'hiker://empty' + id + '#tag#@rule=js:$.require("erji")'` —— 在模块作用域执行，**能复用模块函数/加密工具/配置**，适合解析复杂、逻辑被多入口复用的源。取舍口诀：`简单源内联，复杂复用@rule跳模块`。两者产物一致（最终都返回可解析链接/字符串），但架构和适用范围不同，写之前先想清楚这一集/一项到底要不要复用复杂逻辑。
- **`$.toString`/`registerTask` 轮播定时回调同样在独立作用域执行，闭包变量会报 `ReferenceError: "xx" 未定义`**：UILib 模板的轮播 `registerTask(id, time, $.toString(function(obj){ updateItem(id+'bar',...); unRegisterTask(id); }, {data:items}, id))` 曾因回调内引用闭包 `id` 在 App 里直接报「JS引擎错误 ReferenceError: "id" 未定义」。**所有 `$.toString`/`lazyRule` 序列化回调都不能引用函数闭包变量（`id`/`key`/`onPick`/`foldKey` 等），必须把变量作为参数传入、回调内从形参取**：`registerTask(id,time,$.toString(function(obj){ var id=obj.id; ... }, {id:id,data:items}))`；`lazyRule` 多参数按序绑定形参 `lazyRule(function(k){...}, key)`、`lazyRule(function(fk,fold){...}, foldKey, fold)`。回调内只能用引擎全局函数（`putMyVar`/`refreshPage`/`$.require` 等）和形参。口诀：`序列化回调禁闭包，变量走参数传进去`。
- **搜索项点击进原网页/报"未找到播放数据"，查 `sdetail_find_rule` 是否与选集架构匹配**：`sdetail_find_rule` 必须跟详情选集用的播放方式一致。两种已验证架构：(a) 选集是**单独 `play()` 三级解析**（低端影视），`sdetail_find_rule: js:$.require('模块').play()`；(b) 选集用**内联 lazyRule 直接返回 m3u8**（看剧么/iysdq），`sdetail_find_rule: "*"`（继承 detail）。若把 (b) 配成 `play()`，点搜索项会被误路由进无参的 play()，直接报「未找到播放数据」。搜索项跳原网页则多为 `detail()` 抛异常/取空——必须 `try/catch` + `_fetch(MY_URL)||''`（`getResCode()` 在搜索场景拿错上下文也会跳原网页）。口诀：`sdetail 按选集架构配，内联就继承，三级才指 play`。
- **卡片 url 直接追加 `@rule=js:$.require(模块).detail()`，链接级强接管详情，彻底杜绝"搜索/列表项点进原网页"**：这是比配顶层 `detail_find_rule` 更根本、更稳的兜底。解析引擎写卡片时（`parseItems` 等统一列表入口）直接把详情接管写进**每一条** `url`，如 `url: g.abs(href) + '#immersiveTheme#@rule=js:$.require("niumo").detail()'`。原理：`@rule=` 是海阔的**能力强制接管**，一旦链接带上它，无论顶层 `detail_find_rule`/`sdetail_find_rule` 配没配、配得对不对，点击该卡片都直接执行指定 JS 生成结果页，绝不 fallback 成"原网页"。它比"指望规则引擎正确路由到该条详情页"更可靠，正好绕开 ko58/iysdq 那种"搜索场景 `getResCode()` 取不到上下文"的边界。**写源约定：凡引擎以 `js:` 统一产出卡片（首页/分类/搜索共用 `parseItems`），卡片 `url` 一律追加 `#immersiveTheme#@rule=js:$.require("引擎名").detail()`**；剧集/播放项则用 `play` 接管（不是 detail），工作方式仍可按 `$().lazyRule(...)` 内联或 `@rule` 跳模块。口诀：`列表卡片 url 加 @rule 接 detail，点击进详情不再跳网页`。
- **自绘搜索框的确认跳转，必须 `@rule` 接回本源 `search()`，别用 `hiker://search?s=...` 调海阔自带搜索**：源常在首页用 `col_type:'input'` 自绘搜索框。点搜索后的跳转 url 有两种写法，效果天差地别：(a) **错** `return 'hiker://search?s=' + encodeURIComponent(kw);` —— 这是海阔**系统级搜索协议**，会丢弃本源、打开 App 自带的搜索页/套用系统搜索规则，不是你的 `search()`，经常"搜不到/结果不对"。(b) **对** `return 'hiker://empty#noLoading##noHistory#@rule=js:$.require("niumo").search()';` —— `@rule` 强制接管、回到本源引擎的 `search()`，关键字已在 input 回调里 `putMyVar('kw', 输入词)`，`search()` 读 `getMyVar('kw')`/`MY_KEYWORD` 即可。**写源约定：自绘搜索框确认后一律返回 `hiker://empty#noLoading##noHistory#@rule=js:$.require("引擎名").search()`，并把关键词写进 myVar；不要 return 任何 `hiker://search`/`#http://...@rule=` 之外的海阔系统跳转。**需特别区分：**若搜索采用"数据层/渲染层分离"（`search()` 只取数、`renderSearch` 渲染），则首页 input 必须先开 `hiker://empty?page=fypage` 子页再 `search()`，禁止内联直接执行**，否则 fetch 返回的 `"error"` 会被原生 `ArticleListModel` 当 URL 发请求报 `Expected URL scheme 'http' or 'https' but was 'error'`（完整规范见 `references/search-implementation.md`）。口诀：`自绘搜索 return @rule 接本源 search，别用 hiker://search`。
- **详情页要补信息卡头，不只回剧集列表**。"完善"的详情页返回顺序通常是：`[海报卡] + [剧情简介] + [剧集列表]`。海报/标题/评分从详情页 `h1`、封面的 `<img>`/背景图、评分标签正则提取；海报卡用 `col_type:'movie_1_vertical_pic_blur'`（海报点击可看图用 `url: 图'#.jpg#'`）；简介用 `setDesc` 折叠（`rich_text` + 内嵌 `<a href='#noLoading#@lazyRule=.js:…'>展开:</a>`，用 `updateItem` 原地切换，完整模板见 `layout-design.md` 3.3），两者 `url:'javascript:;'` 不触发跳转。**`<a>` 的 href 必须单引号包裹**（lazyRule 代码全双引号），否则点"展开"无响应。**别默认简介在 `detail-content`**：不同苹果CMS 模板的简介容器 class 常是 `<p class="desc hidden-xs">`、`<span class="left text-muted">简介：</span>` 后接正文，信息块标题/主演/导演/类型/地区/年份在 `<p class="data">` 里——写正则前先打印真实片段定位，别凭经验硬套固定 class。
- **多线路：先判断要不要全列+切换，避免只显示一条被反馈"线路少了"**。苹果CMS(stui) 详情页常有多个"线路 tab"（`<h3 class=title>` 分隔，如"五号线路/三号线路"），每个 tab 对应一个 `<ul class="stui-content__playlist">`，各 tab 的剧集 `sid` 不同（`/58ysplay/{id}-{sid}-{nid}`）、内容重复（真实 m3u8 在播放页 `player_aaaa` 里，`{"encrypt":0,"url":"..."}`）。两种取向：(a) **精简**——线路在播放器内部切换时只取第一个有内容的 `ul` 回一套即可；(b) **多线路切换**——把全部线路都解析出来并加 `scroll_button` 线路栏（存 `myVar` 当前index + `refreshPage(false)` 重渲染），让用户可切。**若用户反馈"只有一条线路"，说明该站线路≠播放器内切、用户想要多线路，就应改成 (b)**。解析线路要点：线路名在 `ul` **前驱**最近的 `.h3.title`，不能用 `pdfa` 硬绑 class，要按出现顺序把「线路名 h3 与后续 ul」配对（脚本扫描所有线路名 h3 位置 + 所有 playlist ul 位置，为每个 ul 找其最近前置线路名）；`sid` 与线路名无固定映射，绝不能硬编码"线路1=一号"。选集切换仍在**内联 lazyRule** 里 `fetch(播放页)`→抠 `player_aaaa`；播放页通常只需 UA+Referer 即可（搜索/列表才可能过验证码）。
- **调试别带坏 cookie，444 可能假阴性**：用 `curl -b 旧cookie文件` 时若 cookie 文件损坏/过期，请求可能反复 444 或"系统安全验证"，误判为"该功能需要登录"。"先删掉 cookie 文件、只带 UA+Referer 重试一次"再下结论。苹果CMS 的"系统安全验证"（`ajajax/verify_check?type=search` + 图片验证码）多在**搜索**接口触发、每次必现，无法在规则内自动绕过——源能出内容就算可用，搜索如实提示即可。
- **`fetch` 能拿到页面但重复写 `?_t=` 时间戳会导致分页 URL 变形**。若用 JS 手动构造分页（由 `MY_PAGE` 计算），务必自己拼连字符与参数，不要依赖可能被重写的 `url` 字段；构造完先 `log('目标URL='+var)` 验证。
- **用 `log()` 精确埋点定位**：首页/搜索/详情各自 try/catch，关键环节打印 `目标URL`、`html长度`、`正则命中数`、`返回条数`、`首条JSON`. App 里日志入口一般在 设置→开发者选项/调试，或按 skill 文件提到的 `setError()` 弹窗形式（`setError` 会把信息直接显示在页面）。
- **逐级验证**：首页 → 分类 → 搜索 → 选集 → 播放，每一级确认再进下一级；播放页真实源在页面 JS（变量/JSON）里时，用 `video://` 交给 App 嗅探，比裸 `fetch` 解析更省事且抗改版。
- **接口签名别一上来就写最高档**：先抓接口判加密档位（可信头 / RSA+AES+MD5 / Wire AES-GCM+HMAC），低档别用高档写法，白写一堆加解密还跑不通。
- **`hiker://empty?...&...` 这类占位 URL 的参数传递链在海阔里不可靠**：当列表项/搜索项以 `js:` 引擎解析并返回 `url: 'hiker://empty?id=x&type=y'`，随后 `detail_find_rule`/`searchFind` 用 `getParam('id')`/`getParam('kw')` 去取，实测经常取不到（`hiker://empty` 是空指令页，query 不一定原样保留），导致详情"未获取到影片信息"、搜索"总是搜不到"。**在 js 引擎里，最稳的做法是：(a) 项 URL 上带参数，`detail()` 用 `MY_URL` 字符串解析 query（`String(MY_URL).split('#')[0].split('?')[1]` 后再正则取 id/type），`getParam` 只做兜底；(b) 搜索词直接读海阔内置变量 `MY_KEYWORD`，不要依赖 `getParam('kw')`；三层结构收尾用 `setResult` 前再 `log('id='+id)` 埋点验证。** 口诀：`参数信 MY_URL/MY_KEYWORD，别信占位URL的getParam`。
- **首页系统分类栏点不动，往往是因为分类逻辑没读系统传入的分类**：顶层配了 `class_name/class_url` 会渲染"系统双排/网格分类栏"，但若 `find_rule` 是 `js:$.require(引擎).home()` 且 home() 只读 `getMyVar('cls')` 而不读 `MY_CLASS_URL`/系统分类，则用户点系统分类栏没反应（看起来"第一排没作用"）。要么 home() 兼容系统分类（读 `MY_CLASS_URL`），要么干脆**删掉 class_name/class_url，只留引擎内自绘 `scroll_button` 分类**，避免双套分类互相打架。口诀：`分类两套必打架，自绘导航就删系统栏`。
- **线路与选集必须'索引一一对应'**：`tabs[]` 与 `lists[][]` 一旦错位，切线路就串剧；点线路后务必 `putMyVar` + `refreshPage(false)`。
- **网盘/账号 token 必须持久化**：用 `setItem/getItem`（跨重启），放 `putMyVar` 会话一关就失效；失效/需提取码/过期三类异常都要兜底（弹输入框 / `toast` / 跳登录页）。
- **阅读&漫画先管编码与相对路径**：GBK 站不声明 `charset` 必乱码；漫画图片不补 `httpHandle`（`//`、`/` 前缀）必裂图。

以上经验的完整调试版与模板见 `references/production-patterns.md`。

## 禁区与口诀（写规则时直接套）

把高频坑浓缩成便于记忆的口诀，**宁可先套口诀再写，不要踩了才想起**。

- **属性值取不到 → 改正则**：`pdfa(x,'&&data-xxx')` 命中 0（尤其属性用单引号时）就改正则 + `JSON.parse`。口诀：`取属性，正则稳，pdfa 命中零就换手`。
- **详情页必须带信息头**：返回顺序 `海报卡 + 简介 + 选集`，简介用 `setDesc` 原地展开/收起（见 layout-design 3.3 方式1）。口诀：`简介折叠用 setDesc，海报要能点大图`。
- **`lazyRule` 回调在「独立作用域」执行，不能引用模块闭包变量，且应返回可解析链接**：`$.require('模块').fn(...)`、`_H`/`_UA` 等在回调里取不到，App 会报"未知链接"。正确做法（看剧么已验证）：回调内**内联完整解析逻辑**，UA/Referer 等直接写成 JS 字符串字面量，`fetch` 目标链接后 `return 真地址+'#isVideo=true#'`，失败 `return 'toast://xxx'`。用 `$()`（或 `$('#noLoading#')`）作为 base，参数按序绑定形参。口诀：`lazyRule 回调解析要内联，闭包变量一引就未知链接`。
- **播放地址默认不加 `@headers=`，除非明确防盗链**：很多 CDN 域名（如 `vip.dytt-network.com` 等）对额外 header 敏感，播放地址后附 `@headers={...}` 反而会导致解析/播放异常。直接在 lazyRule 末尾 `return 真地址+'#isVideo=true#'` 即可；抓取播放**页面**时可带 headers（那只影响抓取），但**返回的播放地址**不要套 header。口诀：`播放地址默认裸，防盗链才带头`。
- **详情页顶部留白 → 详情跳转 URL 加 `#immersiveTheme#`**：海报卡写法对但顶部仍留白，往往是列表/搜索项跳详情没加 `#immersiveTheme#`（沉浸式顶格，海报背景贯通）。口诀：`详情跳转加沉浸式，顶部留白即消失`。注意强调前缀固定为 `‘‘’’`（`\u2018\u2018\u2019\u2019`，两个左引号+两个右引号），勿用 `“”` 或交替引号。
- **列表卡片 url 加 `@rule` 接详情**：所有 js 引擎产出的列表/搜索卡片，`url` 一律追加 `#immersiveTheme#@rule=js:$.require("引擎名").detail()`，点击进详情绝不跳原网页。口诀：`卡片 url 加 @rule 接 detail，点击进详情不再跳网页`。
- **自绘搜索框确认跳转要 `@rule` 接本源 search**：`return 'hiker://empty#noLoading##noHistory#@rule=js:$.require("引擎名").search()'`，千万别 `hiker://search?s=...`（那会调海阔自带搜索、丢本源）。口诀：`自绘搜索 return @rule 接本源 search，别用 hiker://search`。
- **首页搜索框内联直接执行 `search()`，会把 fetch 失败返回的 `"error"` 当 URL 发请求，报 `Expected URL scheme 'http' or 'https' but was 'error'`**：如果搜索用"数据层/渲染层分离"（`search()` 只取数、`renderSearch()` 渲染，此时 `search()` 末尾不再 `setResult`，而是 `$.require('uil').renderSearch(...)`），首页自绘 input 的确认跳转**必须先用 `$('hiker://empty?page=fypage')` 开一个专属搜索子页，再在子页内 `$.require('引擎').search()`**，绝不能 `url:"hiker://empty...@rule=js:putMyVar('keyword',input);$.require('引擎').search()"` 在当前（非搜索页）上下文里内联执行。内联执行会让原生 `ArticleListModel` 用 DATA 层 fetch 失败时返回的字符串字面量 `"error"` 去 construct 请求 URL 而抛异常。口诀：`搜索分离后，首页 input 要开 fypage 子页再搜，别内联直接跑 search`。
- **多线路别重复列剧集**：线路在播放器内部切换时，详情只回一套去重剧集。口诀：`线路在播放器，看板只给一套集`。
- **手动分页别依赖被重写的 url 字段**：用 `MY_PAGE` 自己拼参数，构造后 `log()` 验证。口诀：`分页自己拼，写完先 log`。
- **每次交付先逐级验证 + 自检清单**：不要一次写完就交付。口诀：`写完走六关：首页 分类 搜 详情 选集 播放`。
- **接口加密分档**：先判档再封装，别一上来就写 Wire。口诀：`签名先看档，高档未必是王道`。
- **线路选集要配对**：`tabs[]` 和 `lists[][]` 一一对应。口诀：`线路选集双数组，错位就串剧`。
- **账号 token 要持久**：会话关了就失效。口诀：`token 用 setItem，别睡 myVar`。
- **编码相对路径先管**：阅读/漫画的乱码裂图多源于此。口诀：`阅读管编码，漫画补路径`。

## 自我进化（吸收新规律，让 skill 越用越强）

本 skill 默认就是一种"自我进化"型 skill：每当从**真实新源 / 用户提供的规则 / 一次成功的实战调试**里发现可复用范式，都要主动把它沉淀进知识库，而不是用完即弃。沉淀入口：`references/examples/distillation-pipeline.md`。

触发时机：
1. 用户提供了一个"偏门/高级/能跑"的完整规则（如聚合、网盘、自定义规则、信息型源）要求分析——提炼可复用范式进 `qingdou-architecture.md` 同级的剖析文档，并把稳定通用范式提升进 `SKILL.md` 正文。
2. 一次调试发现新坑或新解法——追加进 `实战高频坑` 与 `禁区与口诀`。
3. 用户纠正了本 skill 里的错误表述——立即修正，并记录修正。

吸收标准：只提炼**通用工程范式**（结构、分流顺序、骨架、口诀），**不复制**具体站点、私有密钥、登录态、成人内容。

## 参考文件

- `references/layout-design.md` — **界面布局设计指南（概念参考）**：从真实源提炼的首页/详情/搜索/播放页布局体系、col_type 选择决策树、页面美化技巧（颜色/背景/渐变/分割线/字体）、完整布局模板、常见缺陷修复。**默认情况下先读 `ui-template.md` 套 UILib 即可；仅当目标页不在 UILib 覆盖范围（播放页/纯文字列表/自绘搜索建议/特殊 col_type/自定义动效）时，才回到本文件按需手写布局。**
- `references/selector-syntax.md` — 选择器与解析规则完整说明、高级语法、动态/深层嵌套
- `references/js-api.md` — 内嵌 JS 引擎完整 API 速查
- `references/col-types.md` — 显示样式 col_type 完整清单与 extra 选项
- `references/link-schemes.md` — 特殊协议链接、链接增强写法、页面标识 #tag#
- `references/production-patterns.md` — 完善级工程模式：模块化单例骨架模板、入口分发、配置化、分页、降级与交付清单
- `references/examples/advanced-source-template.md` — **高阶源统一引擎骨架（首选模板）**：聚合/网盘/信息型源直接整段复制、只改占位符即用的模块代码，含填坑自查清单
- `references/examples/qingdou-architecture.md` — 《青豆剧场》巨型聚合源剖析：豆瓣签名 API、缓存、首页过滤按钮、详情/选集/播放解析、多源聚合、网盘 openShare/parseShareUrl 范式、自定义规则、工具脚本与 TMDB 接入等，按"写源范式"组织的高阶速查
- `references/examples/multi-type-patterns.md` — **多类型源范式速查（新增）**：视频接口签名三分档（init/login/request→RSA+AES+MD5→Wire HMAC）、多线路双数组、网盘登录态四方案+加解密四形+分享转存、阅读(type=read)目录/正文、漫画(type=cartoon)分P与图片解密、直播(type=live)双格式、hikerPop/模板·Q/工具源的各自写法与高频坑
- `references/ui-template.md` — **可复用 UI 渲染模板 UILib（初版）**：把整套页面渲染（折叠筛选、轮播开关、随机色、快速搜索、TVBox 推送、线路/选集/排序、简介折叠、搜索 loading+空态）封装成渲染模块，AI 写源只填"数据对象契约"即可渲染首页/详情/搜索。含完整可复制引擎代码、AI 数据契约、三步接源流程。**搜索框已内置"开 `hiker://empty?page=fypage` 子页再搜"的正确写法（`UI.engine` 需填引擎子页 path），并含完整 `UI.renderSearch`（首屏 loading + 空态），杜绝 `Expected URL scheme...'error'` 报错。**样式统一、动效继承、一处修处处生效。
- `references/checklist.md` — 交付前自检清单：按源类型逐项核对必填字段、特殊情况处理（懒加载/无封面/GBK/Cloudflare 类验证/搜索无封面）、最终六关验证
- `references/examples/distillation-pipeline.md` — **提炼管线（自我进化入口）**：把用户提供的真实规则/新源按标准流程蒸馏成可复用范式，规范吸收什么样的知识、怎么吸收、沉淀到哪里
- `references/examples/verified-pitfalls.md` — **已验证实战坑（新增）**：多站点实测有效的落地细节——`@rule=` 链接级强接管（详情不再跳原网页）、自绘搜索框 `@rule` 接本源 search、多线路播放区分直链/需解析、RSA 分块解密等；写源前后必看，避开即可满分
- `references/search-implementation.md` — **搜索实现规范（数据层/渲染层分离，实测验证）**：顶层 `search_url`+`searchFind` 只转发、引擎 `search()` 只取数据、UILib `renderSearch(data)` 接管 loading+空态+渲染的三层拆解，含各层完整代码、数据契约字段名、以及"首页自绘搜索框必须开 `hiker://empty?page=fypage` 子页再搜（否则 `fetch` 返回的 `"error"` 被原生 ArticleListModel 当 URL，报 `Expected URL scheme...`）"的关键坑。写带搜索功能的视频/漫画类源时，搜索部分直接套这套三层规范。
- `references/examples/eco-corpus-distillation.md` / `references/examples/backtest-report.md` — **历史分析存档（非写源模板）**：2070 源全量语料蒸馏与回测报告，所含范式均已并入 `SKILL.md` 正文及各参考文件，保留仅供追溯，请勿作为写源模板直接套用。

## 实战案例分析（写作范式来源）

本 skill 的正式内容来源于对多份真实可执行小程序的逆向提炼：

- **HaierToken.docx / 海阔help文档**：官方 help，确定了顶层字段、选择器、col_type、JS 引擎、链接协议、页面标识、web_bridge、demo 模板等基础规范。
- **《青豆剧场》rule.json（1.4MB/44 子页面）**：一个到"完善"级别的视频聚合源，贡献了豆瓣签名 API、分层缓存、过滤按钮交互、多源聚合播放、网盘 openShare/parseShareUrl、自定义规则、QuickJS 适配、弹幕/验证码等**高阶可复用范式**（详见 `references/examples/qingdou-architecture.md`）。
- **两批 19 份多类型规则（用户提供）**：河马影视/瓜子影视/豆花电影/jciyuan（视频，含 RSA+AES+MD5 与 Wire 协议缓存）、搜云盘/移动云盘/Quark.简/云帧享（网盘登录态与加解密）、悦读/阅漫/开心看漫画/佬丝图（阅读漫画）、聚直播/期刊杂志demo（直播/阅读）、hikerPop/模板·Q/葫芦侠（UI库/模板引擎/工具资讯）。贡献了**多类型源范式速查**（详见 `references/examples/multi-type-patterns.md`）。

> 提示：这些源都包含运行所需的私有数据（如网盘登录态、第三方 API key）。作为 skill 只提炼**通用工程范式**，不复制其具体站点/密钥/内容。

> 持续沉淀机制见 `references/examples/distillation-pipeline.md`——收到任何新的真实规则，都会按该管线提炼后回写本文件与参考库。