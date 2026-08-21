# 搜索实现规范（数据层 / 渲染层分离）

本文件沉淀一套经过真实 App 实测验证的搜索实现规范：**数据获取与结果渲染彻底分离**。搜索功能拆成三层，各司其职：

- **顶层字段（`search_url` + `searchFind`）**——只负责"把搜索动作接到本源引擎"，不做任何解析。
- **数据层（引擎内 `search()`）**——只负责抓 HTML / 取数据，返回 `{items:[...]}` 数据契约，**不调用渲染**。
- **渲染层（UILib 内 `renderSearch(data)`）**——只负责把 `data.items` 渲染成界面（首屏 loading、列表、空态）。

> 这套规范的核心动机：把渲染从数据请求里拆出去后，`search()` 函数体积大幅缩小、职责单一、可并发维护；同时 `renderSearch` 统一接管 loading 与空态，所有来源的搜索页表现一致。

---

## 1. 顶层字段规范

`search_url` 与 `searchFind` 两个字段配合，把搜索动作引导到本源引擎的 `search()`：

```json
{
  "search_url": "hiker://empty?kw=**",
  "searchFind": "js:putMyVar('keyword', getParam('kw'));$.require('iys').search()"
}
```

要点：

- **`search_url` 用占位地址（`hiker://empty?kw=**`）**：不指向任何真实抓取页，避免海阔原生搜索去请求一个无关页面。关键词 `**` 占位符保留。
- **`searchFind` 只做两件事**：① 把海阔传进来的 `kw` 参数写入规则内变量 `keyword`（`putMyVar('keyword', getParam('kw'))`）；② 一行转发到引擎 `search()`（`$.require("引擎名").search()`）。
- **关键词统一用 `keyword` 这个 myVar 变量名**（与首页自绘搜索框的 `onChange` 写入保持一致），`search()` 里再兼容读取 `MY_KEYWORD` 兜底。

---

## 2. 数据层：`search()` 只取数据

在引擎子页面（如 `path: "iys"`）里，`search()` 只负责：取关键词 → 拼分页 URL → `fetch` → 用 parseList 解析成 items → 交给 UILib，**不做任何 `setResult`**：

```js
function search() {
    var items = [];
    // ① 取关键词：统一读 keyword，MY_KEYWORD / getParam 兜底
    var kw = getMyVar('keyword') || getMyVar('kw') || MY_KEYWORD || '';
    if (!kw) { try { kw = String(getParam('kw') || ''); } catch (e) {} }

    // ② 取数据（仅当有关键词）
    if (kw) {
        var pg = parseInt(MY_PAGE || '1', 10) || 1; if (pg < 1) pg = 1;
        // 用 MY_PAGE 自己拼分页 URL（别依赖可能被重写的 url 字段）
        var url = _H + '/vodsearch/' + encodeURIComponent(kw) + '----------' + pg + '---.html';
        try {
            var htm = _fetch(url);
            if (htm && htm !== 'error' && htm.indexOf('<') >= 0) {
                var sl = parseList(htm);           // 复用统一的卡片解析
                for (var j = 0; j < sl.length; j++)
                    items.push({ title: sl[j].title, pic: sl[j].pic, desc: sl[j].desc, url: sl[j].url });
            }
        } catch (e) {}
    }

    // ③ 数据契约交给 UILib 渲染（search 不 setResult）
    $.require('uil').renderSearch({ items: items });
}
```

要点：

- **`fetch` 返回的判断必须先 `htm === 'error'` 过滤**（fetch 失败时海阔返回字符串 `"error"` ），再 `indexOf('<')` 判断是 HTML，否则拿 `"error"` 去 parse 必然出错。
- **分页用 `MY_PAGE` 自己拼**，不要依赖可能被改写的时间戳/url 字段。
- **结果项字段名必须与渲染层契约一致**：`title` / `pic` / `desc` / `url`。
- **成功 / 失败 / 无关键词三种情况，最终都统一 `$.require('uil').renderSearch({items})`**，空态交给渲染层，数据层不 print 任何占位。

---

## 3. 渲染层：`renderSearch(data)` 接管 loading 与空态

在 UILib 子页面里，`renderSearch` 只负责把数据契约渲染成界面，包含首屏 loading、结果列表、空态三部分：

```js
UI.renderSearch = function(data) {
    var d = [];
    var items = data.items || [];

    // ① 首屏先放 loading 占位，界面立即出现，数据随后 setResult 覆盖
    if ((parseInt(MY_PAGE || '1', 10) || 1) == 1) {
        var dL = [{
            img: "http://123.56.105.145/weisyr/img/Loading1.gif",
            url: "hiker://empty",
            col_type: "pic_1_full",
            extra: { id: "loading_" }
        }];
        setPreResult(dL);
    }

    // ② 渲染结果列表
    for (var i = 0; i < items.length; i++) {
        if (!items[i].title) continue;
        d.push({
            title: items[i].title || (items[i].name || ''),
            pic_url: items[i].pic || '',
            desc: items[i].desc || '',
            url: items[i].url || 'javascript:;',
            col_type: 'movie_3'
        });
    }

    // ③ 空态提示
    if (!d.length) {
        d.push({ title: '未找到相关内容', desc: '请换个关键词重试', pic_url: '', url: 'javascript:;', col_type: 'list_1' });
    }

    // ④ 先清掉首屏 loading，再正式 setResult
    setPreResult([]);
    setResult(d);
};
```

要点：

- **首屏 loading 只在第 1 页（`MY_PAGE==1`）放**，翻页时不要重复放。
- **`setPreResult([])` 必须放在 `setResult(d)` 之前**，先把 loading 占位清掉再渲染真实数据，否则会出现 loading 与结果叠一起。
- **空态用 `list_1` / `text_center_1` 这类单行样式**，给用户明确反馈而非白屏。
- 渲染层字段用 `pic_url`（渲染层专用），数据层用 `pic`（数据契约），在 `renderSearch` 里把 `pic` 映射为 `pic_url`——**数据层与渲染层字段名解耦**，改一边不动另一边。

---

## 4. 首页自绘搜索框入口（关键坑，防 regress）

首页 `col_type:'input'` 自绘搜索框的确认跳转，**必须先用 `hiker://empty?page=fypage` 开一个专属搜索子页，再在子页内 `$.require('引擎').search()`**，**不能**直接在 input 的 url 里内联执行 `$.require('引擎').search()`：

```js
d.push({
  title: '\ud83d\udd0d 搜索',
  desc: data.search.placeholder || '',
  url: $.toString(() => {
      if (input) {
          putMyVar('keyword', input);
          return $('hiker://empty?page=fypage').rule(() => { $.require('iys').search(); });
          return "hiker://empty";
      }
  }),
  col_type: 'input',
  extra: {
      onChange: $.toString(() => { putMyVar('keyword', input); }),
      defaultValue: getMyVar('keyword', '')
  }
});
```

### 为什么不能内联直接执行

❌ **错误写法**（触发原生 ArticleListModel 把 `"error"` 当 URL）：

```js
url: "hiker://empty#noLoading##noHistory#@rule=js:putMyVar('keyword',input);$.require('iys').search()", col_type:'input'
```

这种写法在**当前（非搜索页）上下文**里直接跑数据层 `search()`，框架的原生 `ArticleListModel` 不知道正确上下文，就用数据层 `fetch()` 失败时返回的字符串 `"error"` 去发请求，抛异常：

```
java.lang.IllegalArgumentException: Expected URL scheme 'http' or 'https' but was 'error'
```

✅ **正确写法**（先开子页再搜）：`$.toString(() => { if(input){ putMyVar('keyword',input); return $('hiker://empty?page=fypage').rule(() => { $.require('iys').search(); }); return "hiker://empty"; } })`——先把页面切到一个专属搜索子页（`fypage`），在子页的独立上下文里再执行 `search()`，此时 `fetch` 失败返回的 `"error"` 只会在数据层内部被过滤，不会再被原生框架当作 URL。

### 配套约定

- **input 用 `extra.onChange` 实时把输入写入 `keyword`**，再配合 `defaultValue` 记住上次关键词，搜索框体验更稳。
- **关键词变量名顶层 `searchFind` 与首页 input 统一为 `keyword`**，`search()` 里一并读取。
- 这条已写入 `SKILL.md` 的 `实战高频坑` 与 `禁区与口诀`，与"自绘搜索框 `@rule` 接本源 search"互为补充：**自绘搜索框绝不要 `hiker://search?s=...`（丢本源），也不要内联直接跑 `search()`（触发 `error` 当 URL），必须开子页 `fypage` 再搜。**