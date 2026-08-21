# 内嵌 JS 引擎 API 速查

解析规则可用 `js:` 前缀编写，运行在 App 内置 JS 引擎。以下为高频 API，按用途分类（`input`为管道变量；`getResCode()` 返回当前页面源码字符串）。

## 结果输出

- `setResult(数组)`：返回首页/搜索/二级等解析结果。对象字段：`title`、`url`、`pic_url` 或 `img`、`desc`、`col_type`、`extra`、`content`（仅搜索用）。漫画类可返回字符串集数组或 `pics://` 链接。
- `setPreResult()` + `setResult()`：成对使用，先返回优先展示部分，网络请求完成后追加。
- `setSearchResult` / `setHomeResult`：与 `setResult` 等价，自动识别回调类型。
- `setError('...')`：显示报错/调试信息。

## DOM 解析

- `parseDom(html, rule)`（`pd`）
- `parseDomForHtml(html, rule)`（`pdfh`）：获取一块，不回补域名。
- `parseDomForArray(html, rule)`（`pdfa`）：返回数组。
- `getResCode()`：获取当前网络请求的源码。
- `xpath(html, '//div[@id]/a[1]/@href')`、`xpathArray(...)`（`xpa`）：XPath 支持。

## 网络请求

- `fetch(url, options)`：`options` = `{headers,body,method}`。
  - `fetch(url, {body:{a:'xx',b:1}})` 自动 JSON + `application/json`。
  - `withHeaders:true` 返回 `'{body:'...',headers:{'Set-Cookie':['a=b']}}'` 字符串。
  - `withStatusCode:true` 返回带 `statusCode` 的 JSON 字符串。
  - `timeout:5000`、`redirect:false`、`toHex:true`、`onlyHeaders:true`、`inputStream:true`（用完 `closeMe(stream)`）。
- `post(url, {body})`；`fetchPC()/postPC()` 用电脑 UA。
- `request(url, opts)`：同步请求，支持本地文件（页内后期使用）。
- `batchFetch([{url, options}])`（`bf`）：并发批量。
- `fetchCookie(url, opts)`：单独取 Cookie。
- `getCookie(url)`：读取已存 Cookie。
- `http.fetch(...).success(...).start()`：链式 ajax 风格封装。
- `registerDNS({ '.example.org': '1.1.1.1' })`、`findReachableIP([...])`、`ipping(ip, ms)`。
- `fetchCodeByWebView(url, opts)`：用 WebView 拿源码（需 JS 渲染页面）。

## 常用变量（引擎注入）

`MY_URL`、`MY_HOME`、`MY_RULE`（当前规则对象）、`MY_PAGE`（1 起）、`MY_KEYWORD`、`MY_TYPE`（`home`/`search`）、`MY_CLASS_NAME`/`MY_CLASS_URL`、`MY_YEAR_*`、`MY_AREA_*`、`MY_SORT_*`、`MY_PARAMS`（上级按钮 extra）、`MY_HEADERS`（响应头）、`MY_NAME`（应用名）、`MOBILE_UA`、`PC_UA`。

## 编解码 / 加解密

- `base64Encode/base64Decode`、`atob/btoa`（用 `window0.` 前缀或 `with(window0){}`）。
- `encodeStr/decodeStr(input, 'UTF-8'|'GBK')`。
- `aesEncode('key', input)` / `aesDecode('key', input)`。
- `rsaEncrypt(data, key, options)` / `rsaDecrypt(...)`。
- 增强：`eval(getCryptoJS())` 后可用 `CryptoJS`。
- `md5('xxx')`、`hexToBytes`、`writeHexFile`、`hexToBase64`。
- `convertBase64Image(url)`、`_base64`（byte 数组 base64）。

## 存储

- 全局变量（重启失效）：`putVar/getVar('k','def')/clearVar`。
- 私有持久化（随规则删除而删）：`setItem/getItem/clearItem`；对象用 `storage0.setItem('a',{a:1})`。
- 私有文件：`saveFile(name, content)`、`readFile(name)`、`deleteFile(name)`、`fileExist(name)`（存于 `rules/files/<规则名>/`，加密）。
- 规则内共享：`putMyVar/getMyVar/clearMyVar/listMyVarKeys`。
- 公开存储：`setPublicItem/getPublicItem`（对所有规则公开，勿存私密内容）。
- 全局 Map：`globalMap0`（任意类型），如 `globalMap0.putMyVar('a',{a:1})`。

## 页面 / 交互 / 跳转

- `refreshPage(bool)`（是否回顶）、`back()`/`back(true)`（二级页可用）、`backToHome()`、`back(true)` 退出并刷新上级。
- `toast('...')`、`copy('text')`、`setPageTitle('...')`、`setPagePicUrl(url)`、`setPageParams({a:1})`。
- `getParam('key','def')`、`getPageTitle()`。
- `showSelectOptions({title, options, js})`、`showLoading/hideLoading`、`confirm({title,content,...})`（立即显示，代码内不能引用外部变量）。
- `addListener('onRefresh'|'onClose', s)`、`updateItem(id, obj)`、`deleteItem`、`addItemAfter/Before`、`findItem`。
- `registerTask/unRegisterTask`、`syncExecute({func,param})`、`batchExecute`。
- 投屏/下载：`sharePaste`、`parsePaste`、`getPastes`、`downloadFile`、`createQRCode`、`shareDirectory`。

## 播放相关

- `cacheM3u8(url, opts, fileName)`、`fixM3u8`、`batchCacheM3u8`（`bcm`）。
- `clearM3u8AdLazy(url, opts, rules)` / `clearM3u8Ad`：去 m3u8 广告片段。
- `refreshVideoUrl(url, fromHead)`、`isVideoOrMusic(url)`。
- `startProxyServer($.toString(()=>{...}))`：启本地代理，可加参数让地址唯一。
- `refreshX5WebView(url)`、`refreshX5Desc('float&&255')`。

## 模块 / 配置 / 杂项

- `require(url, opts, version)` / `requireCache(url, hours)` / `fetchCache`（远程代码块缓存）。
- `requirejs('a')`（CommonJS 全局模块，加载 `libs/a.js`）；`require('a')` 本地模块同效。
- `initConfig({key:url})` 后在 JS 内外用 `config.key` 读版本/依赖地址。
- `evalPrivateJS(code)` / `getPrivateJS(code)`：加密代码。
- `getAppVersion()`、`getRuleCount()`、`getColTypes()`、`getCpuAbi()`、`getIP()`、`getPath('hiker://files/x')`。
- `publishRule(rule)`：提交规则到云仓库。
- `loadJavaClass(path, class, soDir)`：加载 dex（高危，需授权）。

> 网页内（webRule/x5Rule/X5 组件）使用 `fy_bridge_app.xxx`（可简写 `fba`），如 `fba.getVar`、`fba.open`、`fba.getHeaderUrl`、`fba.getUrls()`/`window._getUrls()`、`fba.log`。前台在 JS 中调函数即可。