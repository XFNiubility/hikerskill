# 完善级工程模式（Production Patterns）

本文件指导如何把一个小程序写成"完善级"——健壮、可维护、易扩展，达到成熟规则的标准。核心是**模块化单例**：全部解析逻辑集中封装在一个子页面 JS 模块里，顶层字段只做入口转发。

## 为什么用模块化单例

散写在多个字段的规则存在硬伤：

- 首页、搜索、详情各自维护一份"构造请求/解析列表"逻辑，改一个接口要改多处。
- 域名一改、接口一变，多处都要手动同步。
- 无法做配置化（用户改域名、每页数量）与降级兜底。

模块化单例把"数据源 + 解析 + 配置"收拢成一个对象，各场景入口统一分发，逻辑只写一遍。

## 架构总览

```
rule.json
├─ 顶层字段     仅放：title/author/version/type/group/ua/url/class_*/search_url
├─ pages
│   └─ path: 引擎名（如 "mdai"）
│        rule: "js:" + 一段 JS，定义模块对象 + $.exports = 模块
├─ find_rule     = "js:$.require(\"mdai\").home()"
├─ searchFind    = "js:putMyVar('keyword', getParam('kw')); $.require(\"mdai\").search()"
│                   （或把取词也封装进 search() 内部）
├─ detail_find_rule
│                = "js:$.require(\"mdai\").detail(getParam('id') 或 MY_PARAMS...)"
└─ （可选）更多子页面：设置页、关于页、其它辅助页
```

`$.require("mdai")` 会按子页面 path 加载模块并返回 `$.exports` 的值。这是跨场景复用入口的基石。

## 模块骨架模板（通用，按需删减）

以"带配置、分页、分类、搜索、详情、降级"的视频类为例：

```js
js:
// 1) 模块对象：集中所有状态与逻辑
const engine = {
    // --- 元信息 ---
    d: [],
    name: '示例源',
    // --- 配置（可被用户修改 / 持久化） ---
    defaultHost: 'https://api.example.com',
    defaultPageSize: 24,
    host: function () {
        let h = getItem('example_host', this.defaultHost) || this.defaultHost;
        return String(h).replace(/\/+$/, ''); // 去除尾部斜杠，避免拼接出错
    },
    pageSize: function () {
        let n = parseInt(getItem('example_page_size', String(this.defaultPageSize)), 10);
        if (!(n > 0)) n = this.defaultPageSize;
        return Math.min(n, 60); // 钳制上限
    },
    // --- 统一请求头 ---
    headers: function (extra) {
        const h = {
            'User-Agent': 'Mozilla/5.0 (Linux; Android 13) ... Mobile ...',
            'Referer': this.host() + '/',
            'Accept': 'application/json,text/plain,*/*'
        };
        return Object.assign(h, extra || {});
    },
    // --- 统一请求：包装 fetch，携带 headers、超时与错误抛出 ---
    requestJson: function (path, opts) {
        opts = opts || {};
        let url = /^https?:/i.test(path) ? path : this.host() + path; // 支持绝对或相对
        let text = fetch(url, Object.assign({
            headers: this.headers(),
            timeout: 10000
        }, opts));
        return JSON.parse(text);
    },
    // --- 分页偏移 ---
    offset: function () {
        return (MY_PAGE - 1) * this.pageSize();
    },
    // --- 列表项 → 标准结果对象（复用） ---
    toItem: function (it) {
        return {
            title: it.name,
            url: this.host() + '/detail/' + it.id + '.html',
            pic_url: it.cover,                 // 或用 img
            desc: it.note,
            col_type: ''                        // 留空则继承页面 col_type
        };
    },
    // --- 各类入口 ---
    home: function () {
        const self = this;
        let list;
        try {
            let data = this.requestJson('/api/vod/list?class=' + (MY_CLASS_URL || '') + '&offset=' + this.offset() + '&limit=' + this.pageSize());
            list = (data.list || []).map(it => this.toItem(it));
        } catch (e) {
            // 降级：主接口失败时给出内置兜底分类/首页，保证入口不空
            list = this.fallbackList();
        }
        setResult(list);
    },
    search: function () {
        // keyword 可在 searchFind 里 putMyVar 传入，也可在此用 MY_KEYWORD
        let kw = getMyVar('keyword') || MY_KEYWORD || '';
        let data = this.requestJson('/api/vod/search?wd=' + encodeURIComponent(kw) + '&offset=' + this.offset());
        let list = (data.list || []).map(it => this.toItem(it));
        setResult(list);
    },
    detail: function (id) {
        let data = this.requestJson('/api/vod/detail?id=' + id);
        // 选集数组；如需逐条做播放进度记忆，给每条 extra.id 设全局唯一值
        let plays = (data.plays || []).map(it => ({
            title: it.title,
            url: it.url,
            extra: { id: 'example_' + data.vid + '_' + it.index }
        }));
        setResult(plays);
    },
    fallbackList: function () {
        // 主接口不可用时的兜底，保证首页不死
        let self = this;
        return (this.fallbackCategories || []).map(c => ({
            title: c.name,
            url: self.host() + '/list/' + c.id + '.html'
        }));
    },
    fallbackCategories: [ { id: 1, name: '备选分类' } ]
};

// 2) 导出：让 $.require("mdai") 能拿到上面对象
$.exports = engine;
```

要点：

- `MY_PAGE` 由 App 注入，首页/搜索解析规则每次执行都能读到当前页数，`home()`/`search()` 据此算 offset。
- `MY_CLASS_URL` 为顶部/页面传入的分类替换词；无分类时可忽略。
- 单个 list 项解析也可包 try/catch：`map` 中捕获异常的行跳过，避免一项坏数据废掉整页。
- `col_type: ''` 表示继承页面级样式，页面级用 `col_type: "movie_2"` 等统一。

## 顶层字段接法（示例）

```json
{
  "title": "示例源",
  "author": "开发者",
  "version": 1,
  "type": "video",
  "group": "AI生成",
  "ua": "mobile",
  "url": "hiker://empty",
  "class_name": "分类A&分类B",
  "class_url": "ca&cb",
  "col_type": "movie_2",
  "find_rule": "js:$.require(\"mdai\").home()",
  "search_url": "hiker://empty?kw=**",
  "searchFind": "js:putMyVar('keyword', getParam('kw'));$.require(\"mdai\").search()",
  "detail_col_type": "text_3",
  "detail_find_rule": "js:$.require(\"mdai\").detail(getParam('id'))",
  "sdetail_col_type": "text_3",
  "sdetail_find_rule": "*",
  "preRule": "",
  "pages": "[{\"name\":\"引擎\",\"path\":\"mdai\",\"col_type\":\"movie_2\",\"rule\":\"js:...\"}]",
  "icon": "#ff6354ef"
}
```

说明：

- `url: "hiker://empty"` 表示首页不自动请求网络，真正逻辑都在 `find_rule`（js）里用 `requestJson` 发起。
- `search_url` 里 `kw=**` 承载关键词；`searchFind` 先用 `getParam('kw')` 取出再 `putMyVar` 暂存，供 `search()` 内使用，也可直接依赖 `MY_KEYWORD`。
- `detail_find_rule` 需要资源 id 等参数时，首页 `toItem` 里把 id 放进 url（如上 `/detail/<id>.html`），子页/详情用 `getParam('id')` 读出再交给 `detail()`。

## 配置化管理（让用户可调）

- 读配置：`getItem('example_host', 默认)` / `getItem('example_page_size', '24')`。
- 写配置（供"设置"子页面执行）：`setItem('example_host', input)`。
- 入口逻辑统一走 `this.host()`/`this.pageSize()`，不要直接写死域名。
- 若想提供"设置"子页面，在 `pages` 增加一个 `path:"设置"` 的子页面，用 `input`/`select://` 或 `setItem` 交互改配置后返回子页面/highlight 按钮即可。

## 健壮性与降级 Checklist

- 主请求包 try/catch，失败调用 `fallbackList()` 等兜底，避免空白页。
- `JSON.parse` 的响应统一封装，格式异常抛错被上层捕获。
- 单条列表项解析失败不拖垮整页：用 `map` + try/catch 或先过滤坏数据。
- 所有"可能随改版变化"的选择器/接口，集中在模块内一处（如 `selectors`/`apiPaths` 常量），便于改版一键更新。
- 请求头统一走 `headers()`，Referer/UA 一致可降低被拦截概率；必要时按资源加 Referer。
- 播放地址、图片可按需加 `@Referer=`/`@headers=`。
- 全集/章节每条 `extra.id` 保证全局唯一，播放进度才不会串。
- 用 `setError(msg)`/`log()` 标记易错点，方便用户反馈定位。

## 常见空白页根因与调试（重要）

现象"只有搜索能用、首页/分类/详情全空白"，几乎总是这个原因：**入口规则用了 `getResCode()`，但 App 在该场景没把 url 源码注入进来**。搜索能用只是因为搜索走 `search_url` 接口返回了 JSON 能被拿到。

诊断与修复：

- 判断依据：首页/详情的 `find_rule`/`detail_find_rule` 若是 `js:`，**不要依赖 `getResCode()`**，改在规则内显式 `var html = fetch(MY_URL)` 主动请求 url 再解析。
- 搜索规则同样可 `var raw = fetch(MY_URL)`（MY_URL 即替换关键词后的 search_url）。
- 命名规范化的解析：把"取列表/取详情"各自拆成一个方法（如 `parseList(html)` / `detail()`），入口只做「fetch + 调用 + setResult」，统一 try/catch 并 `setError` 打好调试标记（如 `home-fetch:`、`home-parse:`），定位到具体失败点。
- 选择器多线路去重：详情若有多个播放线路 tab，很多站所有 tab 里的剧集 `<a href>` 完全相同（线路在播放器内部切换）。此时**不要重复列，只保留一套去重后的剧集**：按 `href.replace(/\/igokj\/(\d+)-\d+-(\d+)\.html$/,'$1-$2')`（资源号+集号）作去重键；仅当各 tab 链接确实不同且用户需要显式切线路时才按 line 分组输出。每集 `extra.id` 设全局唯一值防进度串集。
- **`pdfa`/`pdfh` 对纯 HTML 属性取值不稳定**：目标站把列表数据塞进单引号属性（`<li data-json='{...}'>`）或详情 `<a>` 需取时，选择器可能命中 0。做法：**jsoup/jsoup 语义校验通过但 App 里命中 0 时，直接改用正则提取** `<li ... data-json='([^']+)'` / `<a href="([^"]+)"[^>]*>\s*([^<]*?)\s*</a>`，必要时 `JSON.parse`。正则方案不依赖 App 选择器引擎，最稳。
- 详情页完善度：除剧集列表外，推荐补 `[海报卡]`（`col_type:'movie_1_vertical_pic_blur'`，取 `h1` 标题+评分+封面图）与 `[剧情简介]`（`col_type:'long_text'`，取详情页第一个非"常见问题"的内容块），两者 `url:'javascript:;'`。参考该站信息卡结构后再实现。
- 逐级验证顺序：首页 URL（能否直接 curl 到含列表结构的 HTML）→ find_rule 解析条目 → 点开详情 → 选集 → 播放。任何一级空白先停，确认该级 fetch/选择器是否命中，再往下。

## 生态高频范式（2070 源语料蒸馏）

来自对 2070 份真实可运行海阔源的全量挖掘，完整报告见 `examples/eco-corpus-distillation.md`。写源时优先套这些被验证的主流做法：

- **88% 源用 `js:` 引擎而非选择器**（1828/2070），`pages` 覆盖 92%。凡能模块化就用引擎，别死磕选择器。
- **preRule ← require(远程共享工具库)**：大量源在 `preRule` 里一个 `require('https://.../预处理.js')` 注入公共能力（UA/Cookie/正则/加密），一个作者库养几百个源。自建源可先维护一个公共 `预处理.js` 再各源复用。注意库与依赖版本强耦合，需自行版本化。
- **跨程序复用「别人的引擎」**：`require('hiker://page/<解析器>?rule=<源名>').fn()`。把通用解析器做成某源的子页面，其它源 `$.require('hiker://page/auto?rule=XXX')` 直接调用。
- **依赖模板自检**：源依赖其它源（模板/公共子页/订阅）时，入口先 `if (request('hiker://home@<依赖源名>') == 'null') { confirm({ title:'依赖检测', content:'缺少依赖,请导入!', ...}) }`，引导用户补依赖。
- **入口 URL 可为 `hiker://empty`**：纯工具/聚合/导航源无需站点列表，`url:'hiker://empty'`，首页完全交给 `find_rule` 的 JS 引擎自由构造卡片（配合 `other`/`tool`/`all` 类型）。
- **加密以 AES 为主，但 DES 也不少见**（语料 243 处）；`base64`/`md5` 是常客。参见 `multi-type-patterns.md` 加密三档，DES 写法并列补入。
- **col_type 高频参考**：`movie_3`(1087)、`movie_3_marquee`(446，带角标三列)、`text_1/2/3`、`icon_round_small_4`(60，工具图标宫格)、`movie_1_*` 系列大图。工具/导航多用图标型宫格。

## preRule 六种真实用途（语料统计）

preRule 不仅仅是"JS 预处理"。2070 源中 731 个有 preRule，用途分为：

1. **远程公共库加载（最主流，363源）**：仅 `require('https://.../预处理.js')`，无其他逻辑。一个库养几百个源。
2. **initConfig 配置（88源）**：`initConfig({依赖: '...', 模板: '...', 分类颜色: '#...'})`，注入依赖脚本和主题。
3. **Cookie/登录/验证码（30源）**：`fetch` 带 `withHeaders:true` 取响应头 → regex 提取 session/token → `putVar` 存。验证码 URL 声明。
4. **eval(fetch) 远程执行（27源）**：从远程/加密源拉取并执行代码，`eval(base64Decode('...'))` 或 `eval(fetch('hiker://files/...'))`。
5. **confirm/request 依赖检查（17源）**：`request('hiker://home@模板·Q') == 'null'` 判依赖 → `confirm({title:'依赖检测', content:'缺少依赖', confirm:()=>导入口令})`。
6. **动态 UA 设置（7源）**：`putVar('ua',';get;utf-8;{User-Agent@...}')` 覆盖默认 UA。

**写源启示**：需要预处理的场景优先走 `initConfig` 或 `require(远程库)`，需要登录态的先 `fetch` + `withHeaders:true` 取 Cookie 再用 `putVar` 存，需要依赖检测的用 `request('hiker://home@依赖源') == 'null'` 弹窗引导。

## 非视频类型速查（live/news/read/cartoon）

| 类型 | 核心模式 | 与 video 最大差异 |
|------|---------|-----------------|
| live | `find_rule` 直接输出流地址（rtmp/flv/m3u8），detail 空置率 97% | 无详情页，文本列表为主，split/regex 解析流 JSON |
| news | 远程 API fetch + base64/md5 签名，search 仅 23% 填充 | 加密频率高 6-12 倍，首页带盐签名请求 |
| read | 纯选择器规则或 `parseDom` 取正文，preRule 空置率 94% | 61% 不需要二级详情，目录/txt/chapter 关键词 |
| cartoon | `last_chapter_rule` 追更 + `cover` 封面 + 防盗链，`pic_2_card`/`pic_1_full` 渲染 | 图片处理 60%，防盗链 header 47%，`pics://` 多图阅读 |

## 播放/详情链路高频坑（实战沉淀）

以下坑均来自真实源排障，属"语义校验通过但一进 App 就翻车"的典型：

- **`lazyRule` 回调在独立作用域执行，拿不到模块闭包变量**。模块顶层 `var _H='...'`、`var _UA='...'` 这类常量，在 `$().lazyRule(function(){ fetch(_H...) })` 回调里**不可用**，一点选集就报 `ReferenceError: _H 未定义`。修复：把回调体内用到的域名/UA 直接内联为字面量，不要引用闭包变量。判断哪些场景是独立作用域：模块方法（`.home()`/`.detail()` 由 `$.require` 调用）在模块作用域内，闭包变量可用；`@lazyRule=`/`$.lazyRule`/`$().lazyRule` 的回调体则独立求值。
- **选中态高亮的正确写法：全角引号 `““””` 包裹 HTML，不能用反引号 ````（实战踩坑）**。反引号开头的 `<font>` 富文本在部分 App 版本会**原样把标签文本渲染出来**（显示成 `<font color="#FF5D50">✓</font>` 而不是红字）。正确写法是 `““””` 全角引号作为前缀再拼 HTML（青豆剧场等成熟源即此写法，实测可渲染）：
  ```js
  title: isActive
      ? '““””<b><font color="#FF5D50">' + ch.name + '</font></b>'
      : ch.name,
  ```
  若某 App 连 `““””` 前缀的 `<font>` 也不支持，才退化到 `extra` 属性做背景/文字色：
  ```js
  var it = { title: ch.name, url: ..., col_type: 'scroll_button' };
  if (idx === cur) it.extra = { 'backgroundColor': '#FF5D50', 'textColor': '#FFFFFF' };
  ```
  **判别口诀：高亮前缀用全角 `““””`，反引号 ```` 会现形**。
- **不要凭线路名盲过滤"蓝光/超清"**。有的站详情接口会给多条线路，命名"蓝光××线""高清××线"；若按 `/蓝光|超清/` 过滤，遇到**整个详情只有一条"蓝光CO线"**的类型（如纪录片、短剧单线路），会把唯一线路也滤掉 → 剧集列表全空。判断某线路该不该留，标准是**该 code 能不能 parse 出真实地址**，而不是名称。拿不准就全部保留。
- **简介不要塞进信息卡 `desc`**：信息卡 `url:'javascript:;'`，把长篇简介拼进 desc 后用户点简介会"跳转空白页"。正确做法是简介单独一条：信息卡只放评分/地区/年份/主创/状态等短元数据，简介独立一行展示（折叠用 `setDesc`，见 `layout-design.md` 3.3）。`url:'javascript:;'`。返回顺序：`[信息卡] + [剧情简介] + [线路] + [选集]`。
- **简介折叠用 `setDesc`（河马范式），别用 `long_text` 伪折叠，也别手写无 href 的 `<a>展开</a>`（实战踩坑）**：
  - `long_text` 是"全屏阅读富文本"样式，不是内容区折叠，折叠场景会展示不一致。
  - 手写裸 `<small><a>展开:</a></small>`（无 `href`）→ 只显示文字、点了没反应。
  - 正确做法：`rich_text` 富文本内，`<a href='#noLoading#@lazyRule=.js:...'>展开:</a>`，代码里 `storage0` 记态 + `updateItem(m,{title:…})` 原地替换"展开:"↔"收起:"及正文。**完整可用代码抄 `layout-design.md` 3.3 方式1**。
  - **`<a>` 的 `href` 必须用单引号包裹** `href='...'`，因为 `@lazyRule=.js:` 代码内部全是双引号；用 `href="..."` 会把属性在第一个内层 `"` 处截断 → 出现"展开二字在但点了没功能"，甚至在个别 App 报 JS 语法错误。
  - `extra.id`、`findItem(m)`、`updateItem(m,…)`、`storage0` 记态的 key 必须统一同一字符串（如上例 `desc`）。
- **漫画源阅读页渲染常见坑（本次实战沉淀）**：
  - `atob` 需要前缀 `window0.` → `window0.atob` 裸用 `atob` 会报 `ReferenceError: "atob" not defined`。文档 js-api.md 第 42 行已说明，实战还是容易忘。
  - 相对路径 URL → App 报"未知连接"；完整 URL → App 直接跳网页不走规则。标准做法：把完整链接塞进 `extra`，跳转用 `hiker://page/path#noHistory#`，子页面内用 `MY_PARAMS.url` 取完整 URL 再 `fetch`。
  - `pics://` 不要放数组，`col_type` 别用不存在的 `pic` → 文档说漫画支持 `pics://`，但要符合渲染要求必须：**每张图片作为独立项，`col_type='pic_1_full'`**，App 才会正确分页渲染。用单个 `url:'pics://a&&b&&c'` 加 `col_type='pic'` 会空白，因为 `pic` 不是合法 `col_type`。
- **`fetch` 返回同步字符串**（对齐 App 内同步 fetch），不要按 Promise 处理 `res.json()`；直接 `JSON.parse(fetch(...))`。真实接口常返回带注释的响应，`JSON.parse` 前按需用正则剥掉。

## 交付与维护

- 交付时给出可直接导入的口令或 `.json`/`.txt` 文件，注明规则名、分组、作者、版本。
- 多文件/带依赖时打包为 `.hk小程序.zip`（内含 `rule.json`，可选 `libs.zip`、`data.zip`/`data2.zip`、`require.json`）。
- 版本号在更新时递增；模块内记录 `version` 便于线上排查。
- 避免在输出中写入任何用户私有信息或敏感数据，示例统一用占位域名与占位字段。

## 学习来源

以上工程模式提炼自成熟海阔小程序的一般写法与官方开发者生态通用实践；具体接口、选择器必须基于目标站点的真实页面/接口逐项校验（用 `hiker://debug?url=` 查看源码、抓包确认 JSON 结构），禁止凭印象臆造路径。