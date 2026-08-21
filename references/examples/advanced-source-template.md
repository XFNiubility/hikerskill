# 高阶源统一引擎骨架（可直接复制 · 填坑即用）

> 用途：当你需要写一个**聚合 / 网盘 / 信息型（带签名 API、缓存、过滤、多源播放）**的源时，**先整体复制本骨架到 `pages` 的一个子页面里，再只替换其中的占位符**，而不是从零拼凑。这样保证高级结构（分层请求、三层缓存、过滤按钮、详情头、播放分流、网盘 openShare）不遗漏。
>
> 占位符一律用 `{占位}` 形式；区分：**必须改**（`{站名}`、`{DOMAIN}`、`{path}`）与**按需增删**（如 `{Netdisk}`、`{输入项}`）。
> 顶层 `rule.json` 的字段仍是"一行转发"（见 `production-patterns.md`），本骨架是 */该子页面 `rule` 的完整引擎代码/*。

## 一、复制这个引擎（子页面 `path: "{engine}"` 的 rule，`js:` 开头）

```js
js:
// == {站名} 高阶引擎（模块化单例） ===========================================
(function () {
  var g = {
    name: '{站名}',
    d: [],
    // --- 配置（可被用户改 / 持久化）---
    defaultHost: '{DOMAIN}',
    host: function () {
      var h = getItem('{engine}_host', this.defaultHost) || this.defaultHost;
      return String(h).replace(/\/+$/, '');
    },
    // --- ① 分级请求封装：带 签名 + UA + 限额/重试 的官方 API 客户端 ---
    getRes: function (url) {                 // {官方接口} 需要 apikey/HMAC 签名时改这里
      var cc = null;
      if ($.type(url) == 'object') { cc = url.cc; url = url.url; }
      var _url = url;
      if (url.indexOf('apikey') === -1)
        _url += (url.indexOf('?') === -1 ? '?' : '&') + 'apikey={APIKEY}';
      var _ts = parseInt(Date.now() / 1000).toString();
      eval(getCryptoJS());
      var _sig = hexToBase64(CryptoJS.HmacSHA1(
        'GET&' + encodeURIComponent(url.split('{DOMAIN}')[1].split('?')[0]) + '&' + _ts,
        '{SECRET}').toString());
      var s = fetch(_url + '&_sig=' + _sig + '&_ts=' + _ts, {
        headers: { 'User-Agent': '{OFFICIAL_UA}' }, method: 'GET' });
      var json = JSON.parse(s);
      if (json.localized_message) {          // 限流/超时
        cc = cc == null ? 1 : cc + 1;
        if (cc > 2) throw new Error('{站名}超时');
        java.lang.Thread.sleep(2000);
        return this.getRes({ url: url, cc: cc });
      }
      return json;
    },
    // --- ② 三层缓存 ---
    cachedFetch: function (cacheKey, id, fetchFn) {   // 内存缓存（短频，30min）
      var CACHE_EXPIRY = 30 * 60 * 1000, cached = getMyVar(cacheKey, '');
      if (cached) { try { var data = JSON.parse(cached);
        if (data.id === id && data.ts && (Date.now() - data.ts < CACHE_EXPIRY)) return data.data; } catch (e) {} }
      var result = fetchFn();
      if (result) putMyVar(cacheKey, JSON.stringify({ id: id, ts: Date.now(), data: result }));
      return result;
    },
    setLayerCache: function (layer, key, data) {      // 文件分层缓存（长 TTL）
      var body = JSON.stringify(data);
      var out = body.length > 512 ? gzipCompress(body) : body;
      saveFile('hiker://files/cache/{engine}_' + layer + '_' + key + '.json', out);
    },
    getLayerCache: function (layer, key) {
      var p = 'hiker://files/cache/{engine}_' + layer + '_' + key + '.json';
      return fileExist(p) ? JSON.parse(gzipUncompress(readFile(p)) || readFile(p)) : null;
    },
    // --- ③ 首页过滤按钮（scroll_button / input，状态化刷新）---
    addFilterOptions: function (d, locals) {
      for (var i = 0; i < locals.length; i++) {
        var v = locals[i], isSel = (getMyVar('filter_{KEY}') === v);
        d.push({ title: isSel ? '““””<b> <font color="#FF5D50">' + v + ' </font></b>' : v,
          col_type: 'scroll_button',
          url: $('hiker://empty#noLoading##noRecordHistory#').lazyRule((val) => {
            putMyVar('filter_{KEY}', val); refreshPage(); return 'hiker://empty';
          }, v) });
      }
      d.push({ title: '自定义', col_type: 'scroll_button',
        url: $('hiker://empty#noRecordHistory##noHistory#').lazyRule(() => {
          return $('', '输入具体{KEY}').input(() => {
            if (input.trim() !== '') { putMyVar('filter_{KEY}', input.trim()); } else clearMyVar('filter_{KEY}');
            refreshPage(); return 'hiker://empty';
          });
        }) });
    },
    // --- ④ 数据组装（home 核心；缓存 key 必含全部过滤维度，防串缓存）---
    GetDatas: function (keyword) {
      var page = parseInt(MY_PAGE) || 1;
      var url = this.host() + '/list?page=' + page + '&class=' + (getMyVar('fyclass') || '');
      var fA = getMyVar('filter_{KEY}',''), fB = getMyVar('filter_{KEY2}','');
      var cacheKey = 'home_' + keyword + '_' + page + '_' + fA + '_' + fB;
      var json = this.cachedFetch(cacheKey, url, () => this.getRes(url));
      var d = [], items = json.list || json.data || [];
      for (var it of items) {
        d.push({ title: it.name, pic_url: (it.pic || '') + '@Referer={REFERER}',
          desc: it.rate || '',
          url: 'hiker://page/{detail}#noHistory##immersiveTheme#',
          extra: { id: it.id, title: it.name } });
      }
      setResult(d);
    },
    // --- ⑤ 详情页：海报卡 + setDesc 简介 + 小菜单 + 列表 ---
    detail: function () {
      var json = this.cachedFetch('{engine}_d', MY_PARAMS.id,
        () => this.getRes(this.host() + '/detail/' + MY_PARAMS.id));
      var d = [];
      d.push({ title: json.title, desc: json.card_subtitle || json.year || '',
        pic_url: (json.pic ? json.pic.large : '') + '@Referer={REFERER}',
        col_type: 'movie_1_vertical_pic_blur', url: 'hiker://empty' });
      setDesc(d, (json.intro || '').replace(/\r\n|\r|\n/g, '<br>'), 100);
      d.push({ title: '演职员', col_type: 'scroll_button', url: 'hiker://page/{cast}#noHistory#',
        extra: { id: MY_PARAMS.id } });   // {小菜单：按需增减}
      var eps = this.parseEps(json);       // 见⑥
      setResult(d.concat(eps));
    },
    // --- ⑥ 选集 + 播放分流（统一判断链）---
    parseEps: function (json) {
      var d = [], epUrls = [], sNames = [];
      var raw = json.episodes || [];
      for (var ep of raw) {
        var link = ep.url, name = ep.name || '';
        // 播放地址分流顺序：pan → JSON对象 → video/#isVideo → m3u8/mp4 → 其它 http
        if (typeof link === 'string' && link.indexOf('pan://') === 0) { epUrls.push(link); sNames.push(name); }
        else if (link && typeof link === 'object') {
          if (link.url !== undefined) link = link.url;
          else if (Array.isArray(link.urls)) { for (var u of link.urls) epUrls.push(u); continue; }
        }
        var p = String(link);
        if (/^video:\/\//i.test(p) || p.indexOf('#isVideo=true#') >= 0) epUrls.push(p);
        else if (/\.m3u8|\.mp4|\.flv/i.test(p)) epUrls.push(p + '#isVideo=true#');
        else if (/^https?:\/\//i.test(p)) epUrls.push('video://' + p + '#isVideo=true#');
        else epUrls.push(p);
        sNames.push(name);
      }
      // 多来源合并：官方源 / 各规则源 / 网盘源 分组（buildCategories）
      var result = { urls: epUrls, names: sNames };
      if (json.danmu) { result.danmu = json.danmu; putVar('dm_share', json.danmu); }
      // 真实地址用 lazyRule 点击时才解析；此处先返回描述串
      return [{ title: (json.title||'') + ' 播放', col_type: 'text_1', url: JSON.stringify(result) }];
    }
  };
  // --- 网盘源骨架（{如有网盘}时启用）---
  g.parseShareUrl = function (url) {
    var shareKey = '', sharePwd = '';
    var keyPatterns = [ /\/\/[^\/]*{DOMAIN}.*\/s\/([a-zA-Z0-9_-]+)/i, /[?&](?:sharekey|key)=([a-zA-Z0-9_-]+)/i ];
    for (var kp of keyPatterns) { var m1 = url.match(kp); if (m1) { shareKey = m1[1]; break; } }
    var pwdPatterns = [ /[?&](?:密码|code|pwd|key)=([a-zA-Z0-9]{4})/i, /提取码[^\w]*([a-zA-Z0-9]{4})/ ];
    for (var pp of pwdPatterns) { var m2 = url.match(pp); if (m2) { sharePwd = m2[1]; break; } }
    return { shareKey: shareKey, sharePwd: sharePwd };
  };
  g.openShare = function (shareUrl, pwd) {
    var parsed = g.parseShareUrl(decodeURIComponent(shareUrl || ''));
    if (pwd && !parsed.sharePwd) parsed.sharePwd = pwd;
    if (!parsed.shareKey) { setResult([{ title: '未识别到{站名}分享链接', url: 'hiker://empty', col_type: 'text_center_1' }]); return; }
    if (!g.ensureAuth()) { setResult([{ title: '{站名}未登录', desc: '请在设置页登录', col_type: 'text_center_1' }]); return; }
    g.listShare(parsed.shareKey, parsed.sharePwd || pwd, 0);
  };
  g.ensureAuth = function () { return getItem('{engine}_login', '') !== ''; }; // {按网盘补 CookieManage 校验}
  g.listShare = function (shareKey, pwd, page) {
    // {按网盘构造 list 请求}：setResult(文件/目录数组)，目录项点进继续列表
  };

  // --- 导出 ---
  g.home = function () { g.addFilterOptions(g.d, ['{选项A}', '{选项B}']); g.GetDatas(MY_KEYWORD || ''); };
  g.search = function () { g.GetDatas(getMyVar('{kw}var') || getParam('word') || MY_KEYWORD || ''); };
  try { g.ping = undefined; } catch (e) {}
  $.exports = g;
})();
```

## 二、把它接进 rule.json（顶层仍是"一行转发"）

```json
{
  "title": "{站名}",
  "type": "video",
  "ua": "mobile",
  "url": "hiker://page/{engine}#noLoading##noRecordHistory#",
  "find_rule": "js:$.require(\"{engine}\").home()",
  "search_url": "hiker://empty?word=**&page=fypage",
  "searchFind": "js:putMyVar('{kw}var', getParam('word'));$.require(\"{engine}\").search()",
  "detail_find_rule": "js:$.require(\"{engine}\").detail()",
  "pages": "[{\"name\":\"引擎\",\"path\":\"{engine}\",\"rule\":\"js: ...（粘贴上文完整引擎）\"},{\"name\":\"详情\",\"path\":\"{detail}\",\"rule\":\"js:setResult(...)\"}]",
  "icon": "#ff6354ef"
}
```

## 三、填坑标检查（交付前逐项自查）

- [ ] `{DOMAIN}` 写成了真实域名，签名 `{APIKEY}/{SECRET}/{OFFICIAL_UA}` 已换成官方接口真实值。
- [ ] `cachedFetch` 的 `cacheKey` 已含**全部过滤维度**（页/分类/地区/类型/年代/平台/排序/评分），否则筛选会串缓存。
- [ ] 详情返回顺序是 `海报卡 → setDesc简介 → 小菜单 → 列表`，海报/简介不触发跳转（`url:'hiker://empty'`）。
- [ ] 播放分流顺序正确（pan → JSON 对象 → video → m3u8/mp4 → 其它 http 包 `video://`）。
- [ ] 网盘源确认 `openShare → parseShareUrl → ensureAuth → listShare → lazyRule` 五段齐备。
- [ ] 逐级验证：首页 → 过滤 → 详情 → 选集（懒解析）→ 播放，各级用 `log()` 埋点，确认再交付。

## 四、为什么这是"能力"而非"文档"

本骨架把《青豆剧场》验证过的六个最常用高级范式（签名客户端、三层缓存、过滤按钮、详情头、播放分流、网盘 openShare）压成一个可整体复制、只改占位符的模块。AI 写复杂源时**默认复制它再填充**，结构上就不会漏、不会从零瞎写。对照参考：

- 概念/原理 → `SKILL.md` 的「高阶源能力」章（读它理解为什么）。
- 完整真实代码 → `qingdou-architecture.md`（看它在真实源里长什么样）。
- **直接照抄填充 → 本文件（真正动手时用这个）。**
- **写源前后必看 → `verified-pitfalls.md`**（多线路直链/解析、`@rule` 接管、RSA 解密等已验证坑，避开即可满分）。