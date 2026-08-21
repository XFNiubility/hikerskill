# 选择器与解析规则参考

海阔视界的解析规则分两类：**选择器规则**（表单规则字符串）和 **JS 规则**（以 `js:` 开头的整段 JS 代码）。两者都可作为 `find_rule`/`detail_find_rule`/`searchFind` 等字段的内容。

## 首页/详情解析规则格式

格式：`列表;标题;图片;描述;链接`

示例：`body&&#post-list&&li;a&&title;img&&src;.index-intro&&Text;a&&href`

## 搜索解析规则格式

格式：`列表;标题;链接;描述;详情;图片`

示例：`.list-content&&.u-movie;h2&&Text;a&&href;.pingfen&&Text;.meta&&Text;img&&data-original`

- 前三个必选，后三个可省略；省略但保留占位用 `*`，如 `.list;//;h2&&Text;a&&href;*;*;img&&data-original`。
- 搜索二级规则若与首页完全一致，可在 `sdetail_find_rule` 写 `*` 表示继承首页的二级规则。

## 基本语法

| 语法 | 含义 | 示例 |
| --- | --- | --- |
| `&&` | 取子元素（分隔选择器路径） | `body&&a` |
| `#` | id | `#post-list` |
| `.` | class | `.desc` |
| 直接写 tag | 标签 | `li` |
| `Text` | 取文本 | `a&&Text` |
| `Html` | 取含标签的完整文本 | `.content&&Html` |
| 默认（其它后缀） | 取属性 | `a&&href`、`img&&src` |
| `--` | 排除匹配 | `body--a&&a&&href`（排除第一个 a 后取剩下的第一个 a 的 href） |
| `&,索引` | 取指定下标子元素（可负数倒取） | `body&&a,1`（第 2 个 a）、`body&&a,-1&&href` |
| jsoup 原生语法 | 复杂属性/过滤 | `body img[src$=.png]&&src` |
| `||` | 或语法，找不到前者找后者 | `body&&#app||#app2&&Text` |

## 值增强（后缀 `.js:`）

- 取得文字/属性后可继续用 JS 处理，`input` 为取值结果：
  `a&&href.js:'https://example.com/'+input.split('?from')[0].replace('/','')`
- 多个选择器结果拼接用 `+`：`li;a,0&&title+'--'+a,1&&title;...；...`
- 末尾要用 `.js:` 处理的拼接，其中的 `+` 用中文 `＋` 书写以免和拼接混淆。
- url 链接本身也可 JS：`js:input+'??...'` 或 `http://a.com.js:input+'/'`。

## 用 `js:` 写整段解析

以 `js:` 前缀，后面是标准 JS，走引擎内建的 `fetch/getResCode/pdfa` 等 API，最后用 `setResult(d)` 输出。适合 JSON 接口、加密、需要循环或 IO 的场景。示例（JSON 接口首页）：

```js
js:
var json = JSON.parse(getResCode());
var d = [];
for (var it of json.list) {
    d.push({
        title: it.name,
        url: 'https://example.com/detail/' + it.id + '.html',
        pic_url: it.cover,
        desc: it.note
    });
}
setResult(d);
```

## 二级列表（选集/章节页）规则

二级列表格式与首页类似，但能力更多：

- **嵌套**：`列表;标题;图片;描述;链接;样式`，可多级 `==>` 连接。
- **链接继承**：直接使用上一级链接写 `*`（不能为空）。
- **按点击位置**：用 `fyIndex` 占位，解析时替换为点击的位置。
- **纯 JS 嵌套**：js 规则里链接加 `@rule=列表;标题;...`，深层嵌套用多个 `@rule=`。

### 动态解析（点击时才解析真链接）

url 后加 `@lazyRule=` 即可（把 `&&` 转义为中文 `＆＆＆＆`），点击结果后才请求该链接用规则解析出真正资源（可识别为视频/图片/网页）。不能嵌套在普通规则里，只能用于最后一级。

- 普通选择器：`a&&href.js:input+'@lazyRule=body＆＆＆＆a＆＆＆＆href'`
- 纯 JS：`a&&href.js:input+'@lazyRule=.js:input'`
- 不显示 loading：`...+'#noLoading#@lazyRule=.js:input'`

### 编写便利工具（`$` 系列）

用 `$()` 方法链优雅地生成动态/嵌套/弹窗链接（LoyDglk 提供），生成结果仍是上述字符串：

```js
// 动态解析
d.push({url: $('url').lazyRule(() => setError(input))});
// 普通规则动态解析
d.push({url: $('url', 'iframe&&src').lazyRule(() => setError(input))});
// 深层嵌套
d.push({url: $('url').rule(() => setError(input))});
// 带参数传递：$.lazyRule 第二参为参数对象，传给下一层
$('url').lazyRule((obj) => { var html = fetch(input); }, {word: 'x'});
// 输入/确认弹窗
$('默认值', 'hint').input(() => 'toast://' + input);
$('hint').confirm(() => 'toast://xiaomi'); // .confirm 参数应为 ()=>'toast://...'
```

注意生成的是立即执行函数 `(function(){})()` 字符串形式。