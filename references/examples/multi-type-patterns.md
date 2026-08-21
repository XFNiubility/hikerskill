# 多类型源范式速查（视频签名 / 网盘 / 阅读漫画 / 直播 / UI与工具）

> 来源：对 19 份真实可运行源（两批）逆向提炼——含视频加密协议源、网盘源、阅读/漫画/直播/UI 库/模板引擎/工具源。此处只沉淀**通用工程范式**（结构、骨架、顺序、口诀），不复制具体站点、私有密钥、登录态与成人内容。这些范式应与 `advanced-source-template.md`（统一引擎骨架）配合使用：**能套骨架就套骨架，本章补的是骨架没覆盖的分支写法。**

---

## A. 视频源：接口签名与 API 客户端（加密协议三档）

成熟视频源几乎都要伪装成官方 App 调接口。按加密复杂度从低到高分三档，**先判断目标接口属于哪一档再选封装**。

### A1. 低档：可信 HEADER 请求封装（POST 表单 + 按需 token）
`init/login/request` 三件套是通用底座：
```js
const client = {
    host: getItem('host', 'https://默认'),
    headers: {'User-Agent': 'Android', 'Accept': 'application/json'},
    init: function() {                       // 统一补全设备ID/版本/频道
        this.headers['Device-Id'] = getItem('dev_id') || genDeviceId();
        this.headers.token = getMyVar('token', '');
    },
    login: function() {                      // 按需取token并存持久
        if (this.headers.token) return;
        var r = JSON.parse(this.request(this.host + '/api/user/init', {}));
        if (r && r.result) { this.headers.token = r.result.token; putMyVar('token', this.headers.token); }
    },
    request: function(url, data) {           // 统一 POST + 表单编码，响应可二次解密
        this.init();
        var body = [];
        for (var k in (data||{})) body.push(k + '=' + data[k]);
        var html = fetch(url, {headers: this.headers, method: 'POST', body: body.join('&')});
        try { return JSON.parse(html); } catch (e) { return html; }
    }
};
$.exports = client;
```
通用要点：`init()` 补全每次都要的静态头（设备ID/版本）；`login()` 只补一次、token 用 `putMyVar` 会话存 + `setItem` 考虑持久；`request()` 统一方法/编码/Accept，响应层若 `json.data` 是密文再二次解密（先判 `typeof data=='string' && 含加密特征`）。

### A2. 中档：RSA+AES+MD5 叠加签名（瓜子影视式）
流程：请求体 AES-CBC 加密→拼成 `signature` 参与 MD5→带上众多静态头 POST→响应里 RSA 解出 `key/iv`→再解密 `response_key`。
```js
post: function(url, reqKey) {
    let t = Math.floor(Date.now()/1000)+'';
    let keys = this.rsa_en(JSON.stringify({iv:'<iv16>', key:'<key16>'}));   // RSA加密对称密钥
    reqKey = this.Encrypt(reqKey || '{}', '<key>', '<iv>');                  // 请求体AES-CBC
    let signature = md5('...&keys=' + keys + '*&<盐>');                      // 明文拼盐签名
    let body = 'time=' + t + '&keys=' + keys + '&request_key=' + reqKey + '&signature=' + signature.toUpperCase();
    let html = JSON.parse(fetch(getItem('host') + url, {
        headers: {'User-Agent':'Lavf/57.83.100','Content-Type':'application/x-www-form-urlencoded', /* deviceId/Version等 */},
        body, method:'POST'})).data;
    let k = this.rsa_de(html.keys);
    return JSON.parse(this.Decrypt_Hex(html.response_key, k.key, k.iv));     // 用RSA解出的key/iv再解响应
}
```
要点：`signature` 的原文拼法必须逐字符对上（含固定盐）；`request_key` 是**加密后的**再参与签名；响应密钥用 RSA 解不硬编码。

### A3. 高档：Wire 协议（AES-GCM + HMAC-SHA256 + 设备指纹，jciyuan 式）

> **DES 变体写法**：若目标源使用 DES（CBC/ECB）而非 AES，只需调整加解密与 key/iv 长度：
> - 加密：`CryptoJS.DES.encrypt(明文, key, {iv:iv, mode:CryptoJS.mode.CBC, padding:CryptoJS.pad.Pkcs7}).toString()`（ECB 不传 iv）。
> - 解密：`CryptoJS.DES.decrypt(密文, key, {iv:iv, mode:CryptoJS.mode.CBC, padding:CryptoJS.pad.Pkcs7}).toString(CryptoJS.enc.Utf8)`。
> - 签名：A3 的 HMAC 签名串不变，但 `bodySha256` 需对 DES 密文计算；若密钥由 RSA 下发，则 `rsa_en(JSON.stringify({iv:'<iv8>', key:'<key8>'}))` 中 iv/key 长度为 8。
> - 响应若带 `enc:v1:` 前缀，先剥前缀再 DES 解密。
请求头带 `X-App-Ts/Nonce/Sign`，body 经 `enc:v1:` 前缀加密。核心是**签名串**：
```
签名字符串 = METHOD + "\n" + path + "\n" + ts + "\n" + appId + "\n" + device + "\n" + nonce + "\n" + bodySha256
X-App-Sign = HmacSHA256(签名串, secret)
```
```js
function buildHeaders(method, path, body, cfg) {
    var h = {'Accept':'application/json','Content-Type':'application/json; charset=utf-8',
             'User-Agent': cfg.ua, 'X-App-Id': cfg.appId, 'X-App-Device': cfg.device,
             'X-App-Platform':'android', 'X-App-Version':'2.9', 'X-App-Version-Code':'2'};
    var ts = Math.floor(System.currentTimeMillis()/1000)+'', nonce = genNonceHex();
    h['X-App-Enc'] = 'v1'; h['X-App-Ts'] = ts; h['X-App-Nonce'] = nonce;
    h['X-App-Sign'] = buildResolveSign(method, path, ts, cfg.appId, cfg.device, nonce, body, cfg.secret);
    return h;
}
```
口诀：`签名串按行分，method/路径/时间戳/设备串/随机串/体哈希，HMAC 上坐`。响应若 `enc:v1:` 前缀先解密再 `JSON.parse`。设备指纹与密钥存 `getMyVar`（会话登录时下发），共享到各子页面用 `$.require("hiker://page/共享模块")`。

### A4. 多线路 / 播放分流的统一写法
把线路做成 `tabs[]`（名称）与 `lists[][]`（每线路选集，元素为 `name$url`）**两个索引严格一一对应的数组**，然后：
```js
tabs.forEach((t, i) => d.push({
    title: t,
    url: $('#noLoading#').lazyRule((idx) => { putMyVar('当前线路', idx); refreshPage(false); return 'hiker://empty'; }, i),
    col_type: 'scroll_button'
}));
```
选集项再按 `MY_URL` 当前线路读 `lists[getMyVar('当前线路')]`。播放地址点击时才解析：`$().lazyRule((码,键) => 调解析接口返回真实地址+'#isVideo=true#', code, key)`。**解析失败统一兜底**：`return 'toast://解析失败';` 而非裸异常。

### A5. 搜索关键字与动态域名
- 搜索词建议 `encodeURIComponent(word)`；需要 `MY_URL.split('#')[1]` 取词时注意页面标识 `#xxx#` 会混入，先按约定的分隔符切。
- 域名会被封而轮换时：用 `preRule` 或首页 `find_rule` 先 `fetch` 公告页再 `pdfh`/`eval` 取最新域名并 `setItem` 存，后续所有请求读 `getItem('host')`（见 `references/link-schemes.md` 与厂家的域名管理）。

### A6. 视频源高频坑
| 坑 | 解法 |
|---|---|
| token/登录失效 | 播放失败先回 `login()` 刷新 token 再重试，别硬编 |
| 设备指纹缺失 | `init()` 里统一生成/读取 Device-Id |
| 响应 `data` 是密文却直接 JSON.parse | 先判解密特征再解 |
| Base64 变体（标准/URL-safe/no-wrap）混用 | 全源统一用同一种 |
| Hex↔字节转换低版本失败 | 自写兼容转换，勿依赖新 API |
| 签名原文按行或拼错 | 严格复制签名字符串拼接格式 |
| `putMyVar` 后分类没变 | 补 `refreshPage(false)` |
| 图片无 `@Referer=` 防盗链 | 一律补 Referer/UA |

---

## B. 网盘 / 云盘源（登录态 + 加解密 + 分享转存）

网盘源绕不开：**登录态获取 → 加解密请求 → 目录分页 → 真实播放/转存**。四个范式循环出现。

### B1. 登录态四种方案（按平台选）
- **Cookie 校验 + WebView 登录（Quark 系）**：接通后由 `fba.getCookie(url)` 定时(200ms)轮询，写入文件/持久配置。校验函数形如 `isLogin = c && c.includes('__uid=') && c.includes('__pus=')`。未登录跳登录页 `hiker://page/login`。
- **短信验证码 + 拼图验证（移动云盘 139）**：发码→取 `puzzleLeft` 拼图→提交→`refreshToken`，把 `Basic mobile:{phone}:{token}` 拼进 `authorization` 持久化。
- **二维码登录（Quark 头像版）**：取 token→生成二维码 URL→轮询 `getServiceTicketByQrcodeToken`→状态 2000000 拿 ticket→访问 account/info 从 `set-cookie` 提 Cookie。
- **Token 手动输入**：`login_in_token()` 让用户粘 token，`copy_token()` 复制，供调试。

### B2. 加解密请求四种（按平台）
| 形态 | 写法要点 |
|---|---|
| CryptoJS AES-CBC（搜云盘） | `fetch→this.Decrypt()`，key/iv 固定，`eval(getCryptoJS())` 后 `CryptoJS.AES.decrypt` |
| Java AES-CBC 随机IV前置（移动云盘） | 请求体 `AES/CBC/PKCS5Padding` + 随机 IV，`IV(16字节) + 密文` 一起 Base64；解密 `copyOfRange(0..16)` 得 IV 再解 |
| AES/GCM/NoPadding（云帧享） | IV 取前 **12 字节**，`GCMParameterSpec(128, iv)`；播放头需 `vuk`=`md5(vid+key)` |
| Wire 直通 | 见 A3 |

统一封装 `post_en(url, body, headers)`：body 存在则先 `encrypt(JSON.stringify(body))`，空了用 GET；返回 `JSON.parse(this.decrypt(data))`。token 快过期（`now - atExpiretime`）时调 `refreshToken` 续期。

### B3. 目录分页与分享解析
- 分页参数各家不一：Quark 用 `_page/_size(size=100)`，139 用 `pageCursor/pageSize`（分享用 `bNum/eNum` 限 200 条），云帧享直接路径传页号。**先读真实返回字段再定**。
- 分享链接统一正则提 key+提取码：`/(shareKey)\/([\w-]+)/`、`/提取(?:码|密码)?[：:\s-]*([\w]{4})/i`。
- 转存两段式（Quark）：`token`→`createSaveTask`→`queryTask` 轮询（最多9次，`status==2` 取 `save_as_top_fids`）→再取播放/下载地址。提取码缺失弹 `$().input('请输入提取码')` 让用户补。
- 真实地址：转码模式取 `data.video_list`，直链模式取 `download_url`；统一 `+ '#isVideo=true#'` 返回。分享/我的用 `pwd_id === 'self'` 区分。

### B4. 错误识别与兜底（必做）
- 失效/需密码：`data.desc` 含"提取码"→ 弹输入框；其它 → `toast/空列表` 提示。
- 登录过期：报错含 `login/token/invalid` → `toast('未登录或登录过期')` + 跳 `hiker://page/login`。
- stoken 过期自动重试：`.errorWithRetry(...)` 里判 `message.includes('stoken')` 则重取 token 再发。
- 长时间请求给 `showLoading/hideLoading`，缺数据给空态，别裸崩。

### B5. 网盘共通坑
- 登录态持久化用 `setItem/getItem`（跨重启），不要放仅会话的 `putMyVar`。
- 一个网盘多入口的加解密/页数/UA 各平台不一致，独立封装模块用 `$.require('panLib')` 抽公共工具（CookieManage/httpRequest/登录态key映射）。
- 分页、播放字段不稳定，务必按真实返回写、加 try/catch 隔离单条。

---

## C. 阅读源（type=read）

### C1. 目录页解析（重点）
用 `MY_PARAMS.Toc` 里的选择器模板动态定位：
```js
var res = getResCode();
let list = pdfa(res, 'body&&' + Toc.chapterList);
list.forEach(item => {
  var title = pdfh(item, Toc.chapterName);
  let u1 = pd(item, Toc.chapterUrl) + '#autoPage##readTheme#';
  if (MY_PARAMS.charset) u1 += ';get;' + MY_PARAMS.charset;   // 编码追加
  d.push({ title, url: 'hiker://page/content', extra: { title, content: MY_PARAMS.content, url: u1 } });
});
```
要点：章节目录可跨页——把当前页章节并入缓存，读 `pd(res,'body&&'+nextUrl)` 取下一页并 `addItemBefore` 追加（目录 scroll 加载）。用 `chapter.json`/`readFile` 缓存已读章节，写 `saveFile` 时 `try/catch` 防文件损坏。

### C2. 正文内容页（富文本分段）
```js
var arr = pdfa(res, 'body&&' + rule区块);
for (let it of arr) {
  let t = pdfh(it, 'Text').replace(/\s*\n\s*/g, '<br>');
  for (let seg of t.split('<br>')) {
    if (!seg || seg.length==1) continue;
    seg = '　　' + seg.trim();                       // 段首缩进
    if (rule.replaceRegex) seg = seg.replace(new RegExp(rule.replaceRegex.replace('##',''), 'g'), '');
    d.push({ title: seg, url: '', col_type: 'rich_text' });
  }
}
```
编码：站是 GBK/GB2312 时在 URL 或 header 声明 `charset`；正文/标题统一 `pdfh`，空串做防御。

---

## D. 漫画源（type=cartoon：封面 + 章节 + 图片分P）

- **目录/封面**：给简介、多个标题色块、海报 `movie_1_vertical_pic_blur`，点封面 `hiker://search?s=xxx` 可再搜。
- **章节列表 + 下载**：章节目录用 `pdfh(item,'a&&Text')`+`pd(item,'a&&href')`；另供"下载漫画"入口跳 `hiker://page/download.view`，把 `chapterList` 与 `parseCode` 塞进 `extra.info`。
- **正文图片 P 解析**（每话一页 P 画，或自动下一页）：
```js
// 相对路径补全三连（务必做，缺了图片打不开）
httpHandle = x => x.startsWith('http') ? x : (x.startsWith('//') ? 'https:'+x : MY_HOME+x);
list = pdfa(html, imageList);
list.forEach(item => {
  let img = pdfh(item, imageRule);
  if (!img.startsWith('http')) img = pd(item, imageRule);   // 属性兜底
  img = httpHandle(img);
  d.push({ pic_url: img, url: img + rule.ref, col_type: 'pic_1_full' });   // 或按需求拆分P
});
```
- **自动下一页**：`Create.nextPage` 用 eval(下一页表达式) 取 next，空或等于当前则停，配合 `getMyVar(状态)` 记忆。
- **图片加密**（crypto 站点）：`$.exports.imageDecrypt` 用 JavaImporter 引 `Cipher AES/CBC/PKCS7Padding`（key/iv 从源码取）解字节流。As 图片解密函数发人 `hiker://page/xxx` 子页面管理。

---

## E. 直播源（type=live：M3U / #genre# 双格式 + 订阅管理）

- **数据源**：订阅列表存 `hiker://files/rules/live/config.json`，含 name/url/ua。加载 `$.require('page/config').getConfig()`（空/坏文件 `try/catch` 返默认）。
- **解析双模式**：
```js
let mode = m3u.indexOf('#EXTM3U') == 0;
let re = mode
  ? /(.*),(.*)\s*?((?:http|rtmp)[^#\s]*)/g
  : /()(?:\B\$c_start|\B#{1,2})?(.*?),?(##|头#|\$c_end|,?#genre#|(?:http|rtmp)[^\s]*)/g;
```
- **分组合并**：M3U 取 `group-title="..."` 分组；`#genre#`/`##`/`头#`/`$c_end` 切换当前组；同组同名多条地址与 headers 用数组 `concat` 合并。
- **设置页**：提供 新建订阅/导入/本地导入/分享/启用禁用/删除/排序/清缓存；写入后 `writeFile` + `refreshPage()`。本地源存 `hiker://files/cache/live/xxx.m3u`。
- 直播节目的 EPG/时间表：多数源只处理 `title/url/headers`，需 EPG 才另接节目单接口。

---

## F. 自定义 UI 组件（hikerPop）、模板引擎（模板·Q）、工具/资讯源（葫芦侠）

### F1. hikerPop：直接调原生 Android UI
- 通过 `$.require('hikerPop.js')` 引独立 UI 库，内部 `const XPopup=com.lxj.xpopup.XPopup` 等直接反射 Android 类（`com.example.hikerview.*`）。
- 能力：弹出(`confirm/input/AlertDialog`)、`ToastMgr`/`ChefSnackbar` 通知、`PlayerChooser` 选播放器、`BiometricPrompt` 生物识别。调用常包在 `$('#noLoading#').lazyRule(() => {...; return 'hiker://empty';})` 里。
- 钩子：安卓 8.0+ 有专用判定（`Build.VERSION.SDK_INT`）；嗅觉浏览器走 `MY_NAME` 分支。

### F2. 模板·Q：按 HTML 片段关键字批量匹配模板
- `Mapping` 导出 `home/search` 两组 `{keys:[片段], url:模板页}`；`autoPage` 按 `htmlSource.includes(key)`（allKey 时全中）挑命中模板，`getPageSource(url)` 拉模板源码后 `eval` 生成 `setResult(d)`。命中唯一时 break。
- 用法：`$.require('hiker://page/auto?rule=XXX').autoPage(htmlsrc, type)`。适合批量/快速适配同 CMS 框架站点。

### F3. 工具/资讯源（葫芦侠）：登录签名 + 加载更多
- 登录：`sign = 'account'+account+'&password'+md5(pwd)+'<盐>'` → `md5(sign)` 作为参数 → token `setItem('TOKEN')`；token 3 天校验一次（`now - TOKEN-Time > 259200000` 重取）。
- 富文本：正则把 `<text>`,`\n`,`<image(.*?)>(http...)</image>` 转成 `Text/<br>/<img>`（十转义使用），配合 `““””` 全角引号前缀的 HTML 颜色块。
- 加载更多：`getMyVar('Page')` 判空才渲染"加载更多"项，点了 `MY_PAGE=1` 重新请求、`putMyVar('Page', 新值)`、`addItemBefore` 追加、没数据 `deleteItemByCls` 删按钮。

---

## G. 跨类型通用口诀

- **视频要伪装 App**：加密协议从"头+表单"到"AES+RSA+盐签名"再到"Wire AES-GCM+HMAC"，先判档再封装。
- **线路与选集用双数组**：`tabs[]` 与 `lists[][]` 索引一一对应，点按时 `putMyVar(线路)` + `refreshPage`。
- **网盘三件事：登录态、加解密、分享转存**，公共能力抽 `panLib`，失效/需密码必兜底。
- **阅读讲编码、漫画讲相对路径**：`charset` 声明 + `httpHandle` 补全 `//`、`/`。
- **直播双格式**：`#EXTM3U` 与 `#genre#` 两套正则在 switch 分支。
- **UI/模板/工具各有自模式**：hikerPop 反射原生、模板·Q 按片段匹配、葫芦侠带盐登录+加载更多。
- **所有分类：先抓真实返回再写选择器，任何分页/筛选切换补 `refreshPage`**。

> 本速查是 `advanced-source-template.md`（统一引擎骨架）的**分支补充**。写源顺序仍是：判断类型/复杂度 → 选骨架 → 按本章对应小节挑选分支范式填充 → 逐级 App 实测 → 过 `references/checklist.md`。

---

## H. parseDom 数据源前端解析（语料 2070 源主流写法）

> 对 2070 份真实源的 API 覆盖率统计：`parseDomForHtml`(822)、`parseDom`(763)、`parseDomForArray`(744)，远高于 `pdfh`(468)/`pdfa`(403)。**凡"抓页面 + 选块 + 取字段"的数据源源，前端解析用 parseDom 三件套是绝对主流**，比纯选择器表达式更受青睐，且几乎总是配合 `getResCode()` / 显式 `request(...)` 使用。

### H1. 三件套语义
- `parseDom(html, rule)` → **取属性/单个值**：`parseDom(list[j], 'img&&data-url')` 取 `data-original`、`parseDom(list[j], 'a&&href')` 取链接、`parseDom(list[j], 'img&&src')`。等价 `pd`。缺省属性名时隐形取 `Text`。
- `parseDomForHtml(html, rule)` → **取 HTML/文本片段**：`parseDomForHtml(list[j], 'h3&&Text')` 标题、`parseDomForHtml(list[j], '.rightBox&&Text')` 描述、`parseDomForHtml(list[j], 'ul,-2&&Text')` 负索引取集合倒数元素。等价 `pdfh`（注意与 `parseDomForTag` 区分）。
- `parseDomForArray(html, rule)` → **取一组**：`parseDomForArray(getResCode(), '.celebrity&&a')` 返回匹配元素数组，`.movies&&.each` 按类取块。等价 `pdfa`。

### H2. 真实源用法骨架（影视艺人库式：首页/搜索/详情三处通用写法）
```js
var list = parseDomForArray(getResCode(), '.items&&li');      // 整块循环
for (var j in list) {
  d.push({
    title:   parseDomForHtml(list[j], 'h3&&Text'),            // 标题取Text
    desc:    parseDomForHtml(list[j], '.right&&Text'),        // 描述
    pic_url: parseDom(list[j], 'img&&data-original'),         // 懒加载图取 data-original
    url:     parseDom(list[j], 'a&&href') + '#immersiveTheme#' // 链接+沉浸式标识
  });
}
res.data = d; setHomeResult(res);   // setHomeResult 等价 setResult，自动识别回调类型
```
要点：`+ '#immersiveTheme#'` 二级沉浸式；`setHomeResult(res)` 传 `{data: d}` 而非直接传数组；搜索页用 `getResCode()` 且列表类名与首页不同。

### H3. 与 select 表达式混用 / 较高阶
- 多字段聚合：`parseDomForHtml(list[j], '')` 取整块文本当 desc 拼接。
- 二次请求：`var jshtml = request(parseDom(html, 'center&&a&&href'), {});` 详情页先取一个链接再 request 拉子页继续 parseDom。
- 回补域名：parseDom 默认回补 `m.`/当前 host 域名；`parseDomForHtml` 不回补，取纯片段/属性原生值。

### H4. 顶层字段模板（各源各文件模式的模子）
对 `url: 'hiker://empty#<真实base>'`（`#` 后放真实地址，首页判断 `MY_URL.split('#')[1]`）、`ua:'auto'`、类名即分类的源，首页标准骨架：
```js
var cnam = MY_URL.split('#')[1];                     // hiker://empty#http://a.com/fyAll
var html = cnam.includes('首页') ? request(base,{})  // 首页分支
                                : request(MY_URL.split('#')[1], {}); // 分类分支
var list = parseDomForArray(html, '.vodlist&&li');
for (...) d.push({title, pic_url, url: parseDom(list[j],'a&&href')+'#immersiveTheme#', ...});
res.data = d; setHomeResult(res);
```
分类/搜索复用同一 `parseDom` 思路，仅列表选择器不同。

> 一句话：**数据源 = `parseDomForArray` 分块 → 块内 `parseDom`取属性 + `parseDomForHtml`取文本 → `setHomeResult({data})`**，url 常带 `#immersiveTheme#`、入口用 `hiker://empty#<base>`。