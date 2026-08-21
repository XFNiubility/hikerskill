# 生态语料蒸馏报告（海阔 2070 源全量分析）

> 来源：一次对 **2070 份真实可运行海阔小程序源**（压缩包名为"海阔4904个小程序"，实际含 2070 条 rule.json）的全量元数据挖掘。本文不复制任何单一源、站点域名、密钥或成人内容，只沉淀**从语料统计出的工程范式、生态事实与高频写法**，供后续写源时直接套用。

---

## 一、生态事实速览（回答"真实源长什么样"）

| 维度 | 统计 | 意义 |
| --- | --- | --- |
| type 实际枚举 | `video`(1013) / 空(384) / `other`(327) / `tool`(91) / `all`(63) / `music`(47) / `picture`(40) / `read`(33) / `live`(31) / `news`(26) / `cartoon`(15) | type 只是渲染分组提示，**别被它局限**；`other`/`tool`/`all` 大量存在，很多入口 `url` 直接是 `hiker://empty`，引擎全在 `pages` 里 |
| find_rule 写法 | **`js:` 引擎占 88%**（1828/2070），纯选择器仅 ~11% | 实战里 JS 引擎是主流，优先模块化引擎而非选择器 |
| `pages` 覆盖率 | 92% 的源使用 `pages`，平均子页面数较高 | `pages` 是标配，不是高级选项 |
| `preRule` 覆盖率 | 35%（732） | **预处理的地位被低估**，大量源靠 preRule 加载公共能力/取 Cookie |
| `ua` | mobile 73% / pc 13% / auto 12% | auto 也是一种常见合法 UA |
| 加密关键字 | `base64`(1649) / `CryptoJS`(271) / **`Des`(243)** / `md5`(331) / `AES`(74) / `RSA`(7) / `xor`(2) | 除 AES 外 **DES 也很常见**，别只准备 AES |
| 请求形态 | `fetch(` 约 3314 处；POST 仅 76 源 | 绝大多数是同步 GET `fetch(url,{headers})`，POST 是少数派 |
| `getMyVar`(381) / `MY_PARAMS`(136) | 跨子页面共享状态高度依赖 | 状态在模块间传递用 `getMyVar/putMyVar`，参数用 `MY_PARAMS` |

## 二、核心范式（本次挖掘的增量）

### P1. preRule ← require(远程共享工具库)（400+ 源的共同底座）
大量源在 `preRule` 里 `require` **一个作者维护的远程 JS**，把公共能力注入当前源，规则正文只写业务逻辑：
```js
// preRule 字段值
js:
var ua = '';
require('https://<站点托管地址>/js/预处理.js');   // 通用预处理
```
- 远程库形态（出现频次）：`预处理.js`(193+61+32+...)、`资源网预处理.js`(135)、`lazy.js`(10)、`subscribe.js`(4)、`lsg.js`(3)、`cms.js`。
- 托管地：gitcode.net、coding.net、gitlab.com、gitea.com、codeberg.org、jsdelivr(cdn)。单域名一个库被数百源共享。
- **写源启示**：团队/个人可先维护一个「预处理.js」集中放 UA、Cookie、公共正则、公共加密、版本检查，各源 `preRule` 一个 `require` 即复用，规则体空前精简。这是生态里"一个库养几百个源"的真实工程范式。
- 注意：此类库可能与依赖版本 strongly 耦合，若 target 库已删/改，源会整体失效——**自维护的源应承担库的版本化**。

### P2. 跨程序复用"别人的引擎"作为公共能力
源不只 `$.require` 自己 `pages` 里的子页面，还会直接复用**其它规则/模板的子页面引擎**：
```js
require('hiker://page/globalParse?rule=道长仓库Pro')   // 拿全局通用解析
require('hiker://page/auto?rule=XXX')                  // 通用 C 站解析
require('hiker://page/par?rule=dm盒子')
```
- 被他源复用的公共子页面 path 高频出现：`auto`(32)、`pro`(29)、`par`+dm盒子、`dtfl`、`fc`+dm盒子、`github`/`utils`。
- **写源启示**：想给"一键聚合解析"能力，就把通用解析器做成一个源的子页面，`$.require('hiker://page/解析器?rule=源名').fn()` 供其它源复用。

### P3. 依赖模板自检（源靠其他源跑，先判定存在）
```js
if (request('hiker://home@模板·Q') == 'null') {   // 判依赖源是否已导入
  confirm({ title:'依赖检测', content:'检测到缺少模板依赖,请导入!', ... });
}
```
- **写源启示**：当规则依赖另一个源（模板/公共子页面/数据订阅）时，入口先 `request('hiker://home@<依赖源名>')` 探测，为 `'null'` 就弹 `confirm` 引导用户导入依赖。参考生态里"模板·Q / 道长DR / dm盒子"这类被依赖源。

### P4. 公共订阅地址 + initConfig 分发
```js
initConfig({
  categoryLib: 'http://<host>/rulelist.json?id=2705',   // 远程分类/数据源
  categoryLocalLib: 'hiker://...',
});
```
还有用 `request('hiker://home@模板·Q')` 判依赖 + 通过 `http://hiker.nokia.press/hikerule/rulelist.json?id=xxxx` 外部订阅分发列表。
- **写源启示**：分类、榜单、规范数据从远程 JSON 订阅，用 `initConfig` 一次性读取分发；比硬编码在页面里更易更新。

### P5. pages 工程化的 path 命名约定（成熟源的标准拆分）
高频子页面 path 呈现清晰的模块化分工，可作为自写引擎的拆分蓝本：
- 高复用工具：`Config`(71)、`frameLoad`(64)、`dataLoad`(69)、`urlParse`(47)、`parse`(49)、`funcTools`(19)、`hijackEnv`(19)
- 功能子页：`ClassTab`(76 分类标签页)、`lazy`(73 懒加载)、`p`(73)、`u`(62)、`ysfx`(61 影视解析)、`x5ai`(49)、`dt`/`tool`/`xnxi`/`sm`
- 模块内大量 `getMyVar`(381)/`MY_PARAMS`(136) 传态。
- **写源启示**：多面源（解析+工具+设置+下载）按「工具层 + 解析层 + UI层」拆子页面，约定 path 语义，单文件可读性极好。

### P6. 入口 URL 可以为 `hiker://empty`，引擎全在 pages
`other`/`tool` 类型大量源 `url: 'hiker://empty'`，首页列表内容完全由 `find_rule` 的 `js:` 引擎在 `pages` 里产出（工具入口、导航、聚合页）。
- **写源启示**：不需要"站内列表"时不必造假 URL，用 `hiker://empty` 当入口，首页交给 JS 引擎自由构造任意卡片。

### P7. 高覆盖 col_type 参考
| col_type | 频次 | 用途 |
| --- | --- | --- |
| `movie_3` | 1087 | 三列图，视频列表默认 |
| `movie_3_marquee` | 446 | 三列跑马灯（含"XX更新"角标） |
| `text_1`/`text_2`/`text_3` | 100+/26/29 | 纯文字列表 |
| `movie_2` / `movie_1` / `movie_1_vertical_pic` / `movie_1_left_pic` | 各数十 | 大图/横图/竖图/左图右字信息流 |
| `icon_round_small_4`(60)/`icon_round_4`/`icon_2_round` | 图标宫格 | 工具/导航入口 |
| `card_pic_3`/`pic_2_card`/`pic_1` | 17/9/15 | 卡片图 |

## 三、给 skill 的建议（待落地）
1. 在 `SKILL.md` 的 write patterns 里补「preRule 远程工具库」与「跨程序引擎复用 + 依赖自检」两节。
2. `multi-type-patterns.md` 加密章节补 **DES** 一段（并列 AES，给出 CryptoJS `CryptoJS.DES` 对称写法骨架）。
3. 补充 type 枚举 `other/tool/all` 与入口 `hiker://empty` 的说明。
4. col_type 推荐表补 `movie_3_marquee`、`card_pic_3`、`icon_round_small_4`。

## 五、第二轮深挖（搜索/三级/本地资源/解析 API）

### 深层统计
- **搜索**：75% 源有 `search_url`，其中 **95% 用 `**` 占位**（1474/1542）；带 Header `{}` 89 源、GBK 13、POST 12。`searchFind` 63% 用 JS 引擎、11% 选择器、26% 空/继承。
- **三级 sdetail**：**97.6% 继承**（2020/2070），即点开详情直接复用详情逻辑或三级即详情；`sdetail_col_type` **默认 `movie_1`(1751)**（大图一行）。这说明多数源"详情即播放/阅读页"，三级独立引擎是少数。
- **选择器写法**：非 js 的 `find_rule` 以 5 段选择器为主（210）；真正的大量解析走 JS 引擎内 `parseDom` 三件套。
- **解析 API 覆盖率**：`parseDomForHtml`(822) > `parseDom`(763) > `parseDomForArray`(744) >> `pdfh`(468)/`pdfa`(403)。**parseDom 三件套是绝对主流**，详见 `multi-type-patterns.md` H 节。
- **分页**：`fypage` 出现于 1657 源（80%）；`[firstPage=` 特例 165；后端分页形态 `&page=`(232)、`/p/`(124)、`pageNum`(68)。
- **子页面 js 引擎内高频 API**：`setResult`(621)、`request`(297)、`getMyVar`(231)、`getParam`(223)、`MY_PARAMS`(221)、`fetch`(220)、`getResCode`(155)、`toast`(97)、`confirm`(63)。

### 新增范式 P8–P10
**P8. 本地文件仓库承载逻辑**（`hiker://files/` + `eval(fetch(...))`）
```js
// find_rule / searchFind
eval(fetch('hiker://files/rules/xyq/zywcj.js')); zywhm();  // 或 ZYWCJ.txt 数据
```
规则把整套解析逻辑（甚至分类/搜索/线路解析）放到**自定义子页面 `hiker://files/rules/<作者>/<名>.js`**，入口 `eval(fetch(hiker://files/...))` 后调暴露函数。作者可把几十个源共享的解析骨架收敛到一个本地文件仓库文件，秒级批量更新。
- **写源启示**：多源同框架（同 CMS）时，把 `home/search/detail/parse` 函数集中到一个 `files/rules/` JS，各源 `find_rule = js: eval(fetch('hiker://files/...')) ; fn();` 复用。

**P9. 聚合搜索协议 `$$$`**
```js
"search_url": "hiker://$$$?wd=**&pg=1&ac=list$$$fypage", "searchFind": "js: ...zywsea()"
```
`$$$` 作分隔符把**协议名 + 参数模板 + 分页替换词**组合成自定义聚合链路。适合"搜多个上游插件的合集"场景（utools/WIAC 式）。

**P10. 入口 `hiker://empty#<真实base>` + `MY_URL.split('#')[1]`**
```js
"url": "hiker://empty#http://base.com/fyAll/index{fypage}.html[firstPage=hiker://empty#http://base.com/fyAll/index.html]"
// find_rule 内
var cnam = MY_URL.split('#')[1];   // 取 # 后真实 base
var html = cnam.includes('首页') ? request(base,{}) : request(MY_URL.split('#')[1],{});
```
`hiker://empty` 只是不自动请求的空壳，真实站点地址放 `#` 后；分类/翻页/firstPage 都围绕它拼。首页/分类分支用 `MY_URL.split('#')[1]` + `includes('首页')` 区分。

### 第二轮沉淀位置
- parseDom 三件套真实用法 & 顶层 `hiker://empty#<base>` 骨架 → `multi-type-patterns.md` **H 节**（已落）。
- P8/P9/P10 与深层统计 → 本文 P8–P10。

## 六、第三轮深挖（非视频类型 / preRule 分类 / 三级非继承）

### P11. 非视频类型的独特模式

#### live（直播，31源）— 流直出，不需详情
- **detail_find_rule 空置率 97%**：直播源不需要二级详情页，流直接由 find_rule 提供。
- **协议差异**：`rtmp`(19%)、`flv`(23%)、`m3u8`(23%)，video 源 rtmp/flv 为 0。`m3u8` 在直播源也常见。
- **解析手段**：大量使用 `split`(61%)、`regex`(45%)、`has_auth_token`(26%)，远高于 video。
- **布局**：`text_3`(29%) 纯文字列表为主，不需缩略图。
- **写源启示**：直播源 = `find_rule` 直接输出流地址（rtmp:// / flv:// / m3u8://），无需 detail/sdetail；结构以 split/regex 解析 JSON 流列表为主。

#### news（新闻，26源）— 远程 API 取数，带加密
- **搜索**：`searchFind` 和 `search_url` 仅 23% 填写，多数新闻源不提供搜索。
- **加密频率极高**：`base64`(12%)、`crypto_sign`(12%)、`encode_decode`(12%)，均比 video 高 6-12 倍。
- **API 调用**：`remote_api_fetch`(62%) 远高于 video(23%)，新闻源多用 JSON API 接口。
- **去重**：`has_dedup` 1 源，video 为 0。
- **写源启示**：新闻源 = `remote_api_fetch` 拉 JSON + `base64/md5` 签名 → `setResult` 出列表，首页常是带盐签名请求。

#### read（阅读，33源）— 选择器为主，无预处理
- **preRule 空置率 94%**：阅读源几乎不需要预处理（video 48%）。
- **detail_find_rule 仅 39% 填充**：61% 阅读源不需要二级详情，直接三级出内容。
- **关键词**：`目录`(12%)、`txt`(12%)、`chapter`(6%)、`content`(18%)。
- **格式**：`format:other`(27%) 大量使用纯 CSS 选择器（非 js:）。
- **写源启示**：阅读源 = 纯选择器规则（`ul&&li;.title&&Text;...;a&&href`）或 `js:` 引擎内 `getResCode()` + `parseDom` 取正文 → `setResult` 分页。

#### cartoon（漫画，15源）— 追更+图片处理
- **last_chapter_rule 40%**：远高于 video(7%)，漫画"追更"需求。
- **cover 关键词**：`cover` 在 detail 中 60%（video 2%），漫画详情页封面提取是刚需。
- **图片处理**：`has_image_rules`(60%)、`sets_headers`(47% 防盗链)、`uses_split`(47%)。
- **col_type**：`pic_2_card`(20%) 为漫画特有。
- **写源启示**：漫画源 = `last_chapter_rule` 追更 + `cover` 封面提取 + 图片防盗链 header + `pic_2_card`/`pic_1_full` 渲染 + `pics://` 多图阅读。

### P12. preRule 的六种真实用途（排除纯远程 require 后）
对 731 个有 preRule 的源，排除 363 个仅 `require(远程库)` 后，剩余 368 个分类：

| 分类 | 数量 | 占比 | 说明 |
|------|------|------|------|
| initConfig 配置 | 88 | 23.9% | 配置依赖脚本/模板/主题色 |
| 其他杂项 | 199 | 54.1% | 收藏标记、变量存储、文件操作、模块导入等 |
| Cookie/登录/验证码 | 30 | 8.2% | 自动获取 Cookie、验证码 URL、登录请求 |
| eval(fetch) 远程执行 | 27 | 7.3% | 从远程/加密源拉取并执行代码 |
| confirm/request 依赖检查 | 17 | 4.6% | 检测依赖规则是否存在，弹窗引导安装 |
| 动态 UA 设置 | 7 | 1.9% | 主动设置非默认 User-Agent |

**Cookie 获取典型写法**：
```js
if(!getVar("cookie@xxx")){
    var res=JSON.parse(fetch("https://xxx.com",{withHeaders:true}));
    var cookie=JSON.stringify(res.headers);
    /(session=.*?);/.test(cookie);
    putVar("cookie@xxx", RegExp.$1);
}
```

**eval(fetch) 远程执行典型写法**：
```js
eval(base64Decode('bGV0IGxvY2FsID1mZXRjaCgnaGlrZXI6Ly9maWxlcy9ydWxlcy9...'));
```

**依赖检查典型写法**：
```js
if (request('hiker://home@模板·Q') == 'null') {
    confirm({ title:'依赖检测', content:'缺少模板依赖,请导入!', confirm:()=>'...'});
}
```

### P13. 三级非继承源（48 个，占 2.3%）
- **规则类型**：JS 引擎 40(83%)、选择器 5(10%)、内联 2(4%)。
- **col_type**：`movie_1`(58%)、`text_center_1`(13%)、`text_1`(8%)。
- **外部引擎引用**：12 个引用外部文件，36 个内联。唯一多源共享的是 `zywcj.js`(3 源)。
- **与 detail 关系**：89.6% 的 sdetail 规则与 detail 不同，说明 sdetail 是作为 detail 的 fallback/override 机制存在，而非独立引擎。
- **写源启示**：97.6% 源继承三级，不用纠结。非继承场景主要是：直播/音乐/广播直接播放（绕开 detail）、阿里云盘 x5_webview_single 渲染、图片/漫画独立阅读器。

### 第三轮沉淀位置
- P11 非视频类型模式 → 本文 P11 节。
- P12 preRule 六种用途 → 本文 P12 节 + production-patterns.md 补充。
- P13 三级非继承 → 本文 P13 节。

## 四、方法学备注（本次蒸馏管线可复用）
- 批量脚本过滤站私属（域名/密钥/成人），只留结构、频次、范式名称。
- `require(远程库)`、`hiker://page/xxx?rule=`、`hiker://empty`、加密关键字、col_type 用正则星号统计即可发现生态主干。
- 单个源因 gitcode WAF(418) 无法拉取原始库，但不影响范式提炼（管线第 2 步：抽象超越单站点）。