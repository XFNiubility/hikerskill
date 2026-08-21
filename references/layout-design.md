# 海阔小程序界面布局设计指南（Layout Design）

> 本文件从**真实成熟源**（2070 个规则 + 1.4MB 青豆剧场 + 多类型实战源）中提炼出**界面布局和渲染设计**的完整体系。目的不是教"怎么写规则"，而是教**怎么把页面写得好看、布局合理**。

---

## 目录

1. [核心设计原则](#1-核心设计原则)
2. [首页布局体系](#2-首页布局体系)
3. [详情页布局体系](#3-详情页布局体系)
4. [搜索页布局](#4-搜索页布局)
5. [播放页布局](#5-播放页布局)
6. [子页面与工具页布局](#6-子页面与工具页布局)
7. [col_type 选择决策树](#7-col_type-选择决策树)
8. [页面美化技巧大全](#8-页面美化技巧大全)
9. [完整布局模板](#9-完整布局模板)
10. [常见布局缺陷与修复](#10-常见布局缺陷与修复)

---

## 1. 核心设计原则

### 1.1 页面层次结构

每个页面应遵循"从上到下、由主到次"的层次结构：

```
[导航/筛选栏]  ← 顶部，交互入口
[轮播/横幅]    ← 视觉焦点（可选，首页第一屏）
[内容区块]     ← 主内容（列表/卡片）
[分页/加载更多] ← 底部
```

### 1.2 数据驱动的布局决策

col_type 的选择取决于**数据量**和**内容类型**：

| 数据特征 | 推荐 col_type | 原因 |
|---------|--------------|------|
| 图片精美、标题短 | `movie_3` | 图片占主导，信息密度高 |
| 标题长、描述多 | `movie_1` / `movie_1_left_pic` | 一行一列，文字空间充足 |
| 只有文字、无图片 | `text_2` ~ `text_5` | 纯文本，按列数控制密度 |
| 大量剧集（>20集） | `text_5` | 一屏显示更多，减少滚动 |
| 少量剧集（<10集） | `text_2` / `text_3` | 每行少些，可读性更好 |
| 需要展示额外信息 | `movie_1_vertical_pic_blur` | 详情页头卡，信息+背景 |

### 1.3 颜色与视觉层次

- 选中态用**高亮色**（`#FF5D50` 橙红、`#FF6699` 粉红、`#1a5e1a` 深绿）
- 分类标题用**彩色**（`<font color=#1e90ff>简介</font>`）
- 重要信息用**加粗**（`<b>text</b>` 或 `‘’text` 全角引号前缀）
- 次要信息用**小字**（`.small()` 方法）
- 背景色用**半透明**（`#20FA7298` 表示 20% 透明度绿色）

---

## 2. 首页布局体系

### 2.1 完整首页结构（从真实源提炼）

成熟源的首页通常按以下顺序输出：

```
1. [搜索框]         ← input col_type，可选
2. [分类导航栏]     ← icon_5 或 scroll_button（大类：电影/剧集/动漫…）
3. [空白分割]       ← blank_block
4. [过滤按钮行]     ← scroll_button 数组（地区/类型/年份/排序）
5. [轮播]           ← card_pic_1，registerTask 定时切换
6. [分区标题]       ← rich_text 带彩色HTML
7. [内容卡片]       ← movie_3 / movie_2 / movie_1_left_pic
8. [更多内容]       ← 分页自动加载
```

> **顺序铁律：大类（分类导航）在最上，筛选（地区/类型/年份）在它之下。** 用 UILib 时，`UI.navs` 负责大类（最上），`UI.filters` 排在 navs 之后、内容之前。**不要用 `setPreResult` 把筛选钉在大类之上**——那会让大类跑到下面、筛选复刻大类，导致"布局上下颠倒"。`首页` 按钮必须 prepend 到大类第一位（`[{title:'首页',id:''}].concat(data.navs||[])`），切页时同时清全局筛选（`putVar('sd_filter','{}')`），否则返回首页会残留筛选/分页状态而无数据。

### 2.2 分类导航栏

**模式 A：icon_5（一行5列，图文并茂）**

```js
// 青豆剧场模式 - 顶部大图标导航
var navButtons = [
    { title: "剧集", icon: "剧集.png" },
    { title: "电影", icon: "电影.png" },
    { title: "综艺", icon: "综艺.png" },
    { title: "动漫", icon: "动漫.png" }
];
for (var i = 0; i < navButtons.length; i++) {
    var btn = navButtons[i];
    var isActive = getMyVar("fyclass", "剧集") === btn.title;
    d.push({
        title: isActive ? '‘‘’’<b> <font color=#FF5D50>' + btn.title + ' </font></b>' : btn.title,
        icon: btn.icon,
        col_type: 'icon_5',
        url: $('#noLoading##noRecordHistory#').lazyRule(function(param) {
            putMyVar('fyclass', param);
            putMyVar('fypage', '1');
            clearMyVar('filter_local'); clearMyVar('filter_type');
            clearMyVar('filter_year'); clearMyVar('filter_platform');
            clearMyVar('filter_score'); clearMyVar('filter_rank');
            refreshPage(true);
            return 'hiker://empty';
        }, btn.title)
    });
}
```

**模式 B：scroll_button（一行滚动，纯文字）**

```js
// 常用模式 - 文字标签滚动导航
var classes = [
    { title: "推荐", id: "0" },
    { title: "电影", id: "1" },
    { title: "电视剧", id: "2" },
    { title: "综艺", id: "3" },
    { title: "动漫", id: "4" }
];
var currentId = getMyVar('currentClass', '0');
for (var i = 0; i < classes.length; i++) {
    d.push({
        title: currentId === classes[i].id
            ? '‘‘’’<b><font color=#FF6699>' + classes[i].title + '</font></b>'
            : classes[i].title,
        col_type: 'scroll_button',
        url: $('#noLoading#').lazyRule(function(id) {
            putMyVar('currentClass', id);
            refreshPage(false);
            return 'hiker://empty';
        }, classes[i].id),
        extra: {
            backgroundColor: currentId === classes[i].id ? '#20FA7298' : ''
        }
    });
}
```

### 2.3 过滤按钮行

```js
// 地区/类型/年份过滤按钮（scroll_button 选中高亮）
var locals = ["全部", "中国大陆", "美国", "日本", "韩国"];
var filter_local = getMyVar("filter_local", "全部");
for (var i = 0; i < locals.length; i++) {
    var isSelected = filter_local === locals[i];
    d.push({
        title: isSelected
            ? '‘‘’’<b> <font color=#FF5D50>' + locals[i] + ' </font></b>'
            : locals[i],
        col_type: 'scroll_button',
        url: $('#noLoading#').lazyRule(function(val) {
            putMyVar('filter_local', val);
            refreshPage();
            return 'hiker://empty';
        }, locals[i])
    });
}
// 自定义输入按钮
d.push({
    title: '自定义',
    col_type: 'scroll_button',
    url: $('', '直接输入地区').input(function() {
        if (input.trim() !== '') {
            putMyVar('filter_local', input.trim());
            refreshPage();
        }
        return 'hiker://empty';
    })
});
```

### 2.4 轮播（banner）

```js
// 轮播数据
var lundata = [
    { title: '海报1', img: 'url1', url: 'hiker://page/detail' },
    { title: '海报2', img: 'url2', url: 'hiker://page/detail' }
];
// 方式1：用 registerTask 自动轮播
function banner(title, start, arr, data, cfg) {
    var id = title + 'lunbo';
    var initIndex = 0;
    putMyVar('banneri', initIndex);
    var time = cfg.time || 5000;
    var col_type = cfg.col_type || 'card_pic_1';
    var desc = cfg.desc || '';
    var item = data[initIndex];
    arr.push({
        col_type: col_type,
        img: item.img,
        desc: desc,
        title: item.title,
        url: item.url,
        extra: { id: id + 'bar' }
    });
    registerTask(id, time, $.toString(function(obj) {
        var data = obj.data;
        var taskId = obj.taskId;
        var cardId = taskId + 'bar';
        var i = Number(getMyVar('banneri', '0'));
        i = i + 1;
        if (i > data.length - 1) i = 0;
        var item = data[i];
        try {
            updateItem(cardId, {
                title: item.title,
                img: item.img,
                url: item.url
            });
        } catch (e) {}
        putMyVar('banneri', i);
    }, { data: data, taskId: id }));
}
// 调用
banner('首页轮播', true, d, lundata, {
    time: 5000,
    col_type: 'card_pic_1',
    desc: '0'
});
```

### 2.5 分区标题

```js
// 彩色分区标题
d.push({
    title: '<font color=#5cb85c>▍</font><b>' + '热门推荐' + '</b>',
    col_type: 'rich_text',
    extra: { lineVisible: false }
});
```

### 2.6 首页内容列表

```js
// 最常用的 movie_3（一行三列，图片+标题+描述）
d.push({
    title: item.title,
    pic_url: item.pic + '@Referer=http://site.com/',
    desc: item.score || item.remark || '',
    url: 'hiker://page/detail',
    col_type: 'movie_3',  // 或 movie_3_marquee（标题过长跑马灯）
    extra: { id: item.id, title: item.title }
});

// 资讯类用 movie_1（一行一列，图文丰富）
d.push({
    title: item.title,
    pic_url: item.pic,
    desc: item.summary,
    url: item.url,
    col_type: 'movie_1',
    extra: { id: item.id }
});
```

---

## 3. 详情页布局体系

> **硬性实施规范（本 skill 对视频/动漫类源详情页的唯一标准实现，AI/开发者必须照此编写，不得降级）**
>
> 详情页返回顺序固定为：`[海报卡] → [剧情简介] → [小菜单/线路] → [剧集列表]`。各部分**唯一标准实现**：
>
> 1. **海报卡**：仅用 `col_type:'movie_1_vertical_pic_blur'`，标题、封面、描述放 title/desc/pic_url。**已验证范式（低端影视，App 已实测无留白问题）**：`title` 只放**纯标题单行**；`desc` 放**多行元信息**（`类型/标签/年份/评分` + `\n主演：` + `\n导演：` + `\n地区：` + `\n更新：`，行与行之间用 `\n` 拼接）；`pic_url` 为封面（防盗链加 `@Referer=`/`@headers=`）；`url` 用 `封面+'#.jpg#'`（点击查大图）；`extra:{gradient:true}`。⚠️ **"海报上方留白过高"主因不是 `#.jpg#`/gradient，而是把信息堆进 `title` 的 `\n`、同时把 `desc` 掏空**——正确做法是 title 单行 + desc 多行元信息。禁止只用 `text_1` 标题行当"头"。
> 2. **剧情简介**：**必须用 `setDesc`（rich_text + `<a href='#noLoading#@lazyRule=.js:…'>展开:</a>` 原地展开/收起）**。**禁止**用 `long_text`"假装折叠"、**禁止**手写无 `href` 的裸 `<a>展开:</a>`（只显示字、点不动）。完整可复制的 setDesc 定义见本节 `3.3 方式1`。`<a>` 的 `href` 必须单引号包裹（代码内全双引号）。
> 3. **线路切换**：多线路用 `scroll_button`，每条 `url:$('#noLoading#').lazyRule(()=>{putMyVar('pl',s);refreshPage();return 'hiker://empty'},s)`，省略号要有强制更新标记。少线路（1个）可不输出。
> 4. **剧集列表**：按集数动态选 `text_2`~`text_5`；不允许硬性 `text_2`。
>
> 任何源只要类型是 video/cartoon，详情页就必须满足以上 4 点；做不到或站点结构不支持的，在交付说明里注明原因，不得静默降级。

### 3.1 完整详情页结构（最重要的页面）

```
1. [海报卡]         ← movie_1_vertical_pic_blur，标题+封面+评分，url=封面'#.jpg#'
2. [剧情简介]       ← setDesc（rich_text+展开/收起，3.3 方式1，唯一标准）
3. [空白分割]       ← blank_block
4. [小菜单]         ← scroll_button（演职员/剧照/剧评，跳转子页面）
5. [排序/线路切换]  ← scroll_button 或 text_center_1
6. [剧集列表]       ← text_2~text_5 动态选择
```

### 3.2 海报卡（详情页头部）

```js
// 标准海报卡 - 青豆剧场模式
d.push({
    title: json.title + '\n' + json.card_subtitle,
    desc: (json.rating ? '评分：' + json.rating.value : ''),
    pic_url: (json.pic && json.pic.large)
        ? json.pic.large + '@Referer=https://m.douban.com/'
        : '默认海报URL',
    col_type: 'movie_1_vertical_pic_blur',
    url: 'hiker://empty'  // 不跳转
});

// 河马影视模式 - 带演员和标签信息
d.push({
    title: res.title + '\n' + ('``' + (res.tags || '') + ' ' + (res.year || '')).small(),
    desc: (res.actor ? '演员：' + res.actor + '\n' : '') + (res.remarks || ''),
    img: res.pic,
    url: res.pic + '#.jpg#',  // 点击查看大图
    col_type: 'movie_1_vertical_pic_blur',
    extra: { gradient: true }  // 背景渐变
});

// 豆花电影模式 - 彩色标题+渐变背景
d.push({
    title: '‘‘’’' + title.bold() + '\n\n' + updateInfo,
    desc: actorInfo,
    pic_url: coverUrl,
    url: detailUrl + '#noHistory#',
    col_type: 'movie_1_vertical_pic_blur',
    extra: { gradient: true }
});
```

> ⚠️ **消除海报上方留白的关键（App 实测定论）**：海报卡本身的写法（title 单行 + desc 多行元信息，见上节硬规范第 1 条）只是其中一半；**另一半是在列表/搜索项跳转详情时给 URL 加 `#immersiveTheme#`**（沉浸式主题，让详情页顶格到屏幕顶、海报卡背景上下贯通），否则无论海报卡怎么写，详情页顶部都会留出高一块空白。两者缺一不可。写法：详情链接 `url: 详情页URL + '#immersiveTheme#'`；请求时会自动清除该标记，不影响详情页内部 `fetch(MY_URL)`。（iysdq/低端影视 均已实测。）

### 3.3 简介（可展开/收起）

显示富文本，点击"展开"/"收起"原地切换。核心是**方式1 setDesc**（河马影视实测范式），它比 `long_text` 更正确——`rich_text` 富文本内嵌一个 `<a>`，`href` 里放一串 `@lazyRule=.js:` 代码，点击时内联执行 `updateItem("desc",…)` 原地替换自身 title 的展开/收起标记和正文。

```js
// 方式1：setDesc 函数（推荐，河马实测可原地展开/收起）
function setDesc(d, desc, num) {
    if (!desc || !desc.trim()) return;
    num = num || 60;
    desc = String(desc).replace(/\s+/g, ' ').replace(/&nbsp;/g, ' ').trim();
    var head = '<b><font color="#098AC1">∷剧情简介\t</font></b>';
    var m = 'desc';                       // extra.id 必须与 updateItem/findItem 的 key 一致
    var full = '　　' + desc;
    var sdc = (desc.length > num) ? '　　' + desc.substr(0, num) : full;
    var isLong = desc.length > num;

    var title = head;
    if (isLong) {
        var fn = '((dc,sdc,m,cs)=>{'
            + 'var show=storage0.getItem(m,"0");'
            + 'var title=findItem(m).title;'
            + 'var re=/(<\\/small><br>.*?>).+/g;'
            + 'var exp="展开:";var ret="收起:";'
            + 'if(show=="1"){'
            + 'updateItem(m,{title:title.replace(ret,exp).replace(re,"$1"+sdc+"</small>").replace(/(<\\/small><br>\\<font color=").*?(">)/,"$1"+cs.hide+"$2")});'
            + 'storage0.setItem(m,"0");'
            + '}else{'
            + 'updateItem(m,{title:title.replace(exp,ret).replace(re,"$1"+dc+"</small>").replace(/(<\\/small><br>\\<font color=").*?(">)/,"$1"+cs.show+"$2")});'
            + 'storage0.setItem(m,"1");'
            + '}'
            + 'return "hiker://empty";})';
        var args = '(' + JSON.stringify(full) + ',' + JSON.stringify(sdc)
            + ',"' + m + '",{"show":"black","hide":"grey"})';
        var lazy = '#noLoading#@lazyRule=.js:' + fn + args;
        // ⚠️ href 必须用单引号包裹！lazyRule 代码内部有大量双引号，用双引号包裹会把属性截断
        title += "<small><a style='text-decoration:none;' href='" + lazy + "'>展开:</a></small>";
    }
    title += '<br><font color="grey">' + sdc + '</small>';
    d.push({
        title: title,
        col_type: 'rich_text',
        url: 'javascript:;',
        extra: { id: m, lineSpacing: 6, textSize: 15, lineVisible: false }
    });
}
setDesc(d, json.intro, 100);
```

要点（实战踩坑总结）：
- **`<a>` 的 `href` 属性必须用单引号** `href='...'`，因为 `@lazyRule=.js:` 代码里全是双引号（`"0"`、`"1"`、`JSON.stringify` 的输出等）。若用 `href="..."`，第一个内层双引号就把属性截断，lazyRule 代码被破坏 → 出现"展开两个字在，但点了没功能"甚至 JS 语法错误。
- 正文与切换标记用 `storage0` 记态(`"0"`收起/`"1"`展开)，点击时 `updateItem(m,{title:…})` 原地替换，而非新开页面。`extra.id`、`findItem(m)`、`updateItem(m,…)`、`storage0` 的 key 必须同为 `desc`。
- 简介参数用 `JSON.stringify` 传入（自动转义双引号/换行），`dc` 全文与 `sdc` 截断文本作为固定函数参数。
- 短文本（≤num）不生成 `展开:`，只显示全文。

// 方式2：long_text（简单，不可展开）—— 只适合不要求折叠的场合；折叠需求优先方式1
d.push({
    title: '剧情简介',
    desc: json.intro,
    url: 'javascript:;',
    col_type: 'long_text',
    extra: { textSize: 16 }
});

// 方式3：text_1 + @rule 点击展开（豆花模式）—— 用完整地址切页，不原地切换
d.push({
    title: '简介：' + intro.substr(0, 55) + '...<font color=red>详情</font>',
    url: 'hiker://empty##noHistory#' + '\n' + fullIntro
        + '@rule=js:var d=[];d.push({title:fullIntro, col_type: "long_text"});setResult(d);',
    col_type: 'text_1'
});
```

### 3.4 小菜单（跳转子页面）

```js
// 子页面跳转菜单（scroll_button）
d.push({
    title: '演职员',
    col_type: 'scroll_button',
    url: 'hiker://page/workers#noHistory#',
    extra: { id: MY_PARAMS.id }
});
d.push({
    title: '剧照',
    col_type: 'scroll_button',
    url: 'hiker://page/photos#noHistory#',
    extra: { id: MY_PARAMS.id }
});
d.push({
    title: '剧评',
    col_type: 'scroll_button',
    url: 'hiker://page/review#noHistory#',
    extra: { id: MY_PARAMS.id }
});
```

### 3.5 线路切换

```js
// 线路选择（scroll_button 高亮当前线路）
var channels = res.playChannels || [];
var currentLine = getMyVar('currentLine', '0');
for (var i = 0; i < channels.length; i++) {
    var isActive = String(currentLine) === String(i);
    d.push({
        title: isActive
            ? '‘‘’’<b><font color=#FF5D50>' + channels[i].name + '</font></b>'
            : channels[i].name,
        col_type: 'scroll_button',
        url: $('#noLoading#').lazyRule(function(idx) {
            putMyVar('currentLine', String(idx));
            refreshPage(false);
            return 'hiker://empty';
        }, i),
        extra: {
            backgroundColor: isActive ? '#20FA7298' : ''
        }
    });
}
```

### 3.6 剧集列表（动态选择列数）

```js
// 根据集数动态选择 col_type
var episodes = getEpisodes();
var colType = 'text_2';
if (episodes.length > 30) colType = 'text_5';
else if (episodes.length > 20) colType = 'text_4';
else if (episodes.length > 10) colType = 'text_3';

for (var i = 0; i < episodes.length; i++) {
    d.push({
        title: episodes[i].name,
        url: $().lazyRule(function(ep) {
            return parsePlayUrl(ep);
        }, episodes[i]),
        col_type: colType,
        extra: { id: 'ep_' + episodes[i].id }
    });
}

// 剧集列表排序切换
d.push({
    title: (getMyVar('epSort', '0') === '1') ? '降序' : '正序',
    col_type: 'text_center_1',
    url: $('#noLoading#').lazyRule(function() {
        var sort = getMyVar('epSort', '0');
        putMyVar('epSort', sort === '1' ? '0' : '1');
        refreshPage(false);
        return 'hiker://empty';
    })
});
```

---

## 4. 搜索页布局

### 4.1 搜索输入框

```js
// 方式1：col_type: 'input' 内置搜索框
d.push({
    title: '搜索',
    col_type: 'input',
    url: $.toString(function() {
        if (input) {
            putMyVar('keyword', input);
            refreshPage();
        }
        return 'hiker://empty';
    }),
    extra: {
        defaultValue: getMyVar('keyword', ''),
        onChange: $.toString(function() {
            putMyVar('keyword', input);
        })
    }
});

// 方式2：顶层 search_url 字段
// "search_url": "https://site.com/search?wd=**",
// "searchFind": "js:$.require('engine').search()"
```

### 4.2 搜索结果列表

搜索结果列表与首页列表使用相同的 col_type 和解析逻辑，确保风格一致。

---

## 5. 播放页布局

### 5.1 视频播放

```js
// 方式1：直接返回视频URL
setResult([{
    title: videoName,
    url: m3u8Url + '#isVideo=true#',
    col_type: 'video'
}]);

// 方式2：带请求头的视频
setResult([{
    title: videoName,
    url: m3u8Url,
    col_type: 'video',
    extra: { Referer: 'http://site.com/', 'User-Agent': 'Mozilla/5.0' }
}]);

// 方式3：多线路播放（JSON 字符串）
var result = {
    urls: [m3u8Url1, m3u8Url2],
    names: ['线路1', '线路2'],
    danmu: danmuUrl  // 可选，弹幕
};
setResult([{
    title: videoName,
    url: JSON.stringify(result),
    col_type: 'video'
}]);
```

---

## 6. 子页面与工具页布局

### 6.1 设置页

```js
// 设置页 - 使用 text_1 + input 组合
d.push({
    title: '当前域名：' + getItem('my_host', '默认'),
    col_type: 'text_center_1'
});
d.push({
    title: '修改域名',
    col_type: 'input',
    extra: {
        defaultValue: getItem('my_host', ''),
        onChange: $.toString(function() {
            setItem('my_host', input);
        })
    }
});
d.push({
    title: '保存设置',
    col_type: 'text_center_1',
    url: $.toString(function() {
        toast('设置已保存');
        return 'hiker://empty';
    })
});
```

### 6.2 关于页

```js
d.push({
    title: '规则名称',
    desc: '我的规则 v1.0',
    col_type: 'text_1'
});
d.push({
    title: '作者',
    desc: 'trae',
    col_type: 'text_1'
});
d.push({
    title: '说明',
    desc: '这是一个视频源，支持分类浏览和搜索',
    col_type: 'long_text'
});
```

---

## 7. col_type 选择决策树

### 7.1 首页列表选择

```
需要展示图片吗？
├── 是 → 图片质量高？
│   ├── 是 → 标题短？
│   │   ├── 是 → movie_3（三列，默认）
│   │   └── 否 → movie_3_marquee（跑马灯标题）
│   └── 否 → 图片小？
│       ├── 是 → icon_2 / icon_2_round（左图标右文字）
│       └── 否 → movie_1_left_pic（左图右文）
└── 否 → 纯文字列表？
    ├── 是 → 信息多？
    │   ├── 是 → text_1（一行一列）
    │   └── 否 → text_2/3/4/5（多列，按数据量）
    └── 否 → 工具/设置页 → text_1 / input
```

### 7.2 详情页头部选择

```
需要展示海报封面？
├── 是 → 需要背景模糊效果？
│   ├── 是 → movie_1_vertical_pic_blur（推荐，视觉效果好）
│   └── 否 → movie_1_vertical_pic（无模糊）
└── 否 → card_pic_1（方形卡片，适合音乐/资讯）
```

### 7.3 剧集列表选择

```
剧集数量？
├── >50集 → text_5（一行5列，密集）
├── 20-50集 → text_4（一行4列）
├── 10-20集 → text_3（一行3列）
├── 5-10集 → text_2（一行2列）
└── <5集 → movie_1_left_pic（一行一列，展示更多信息）
```

---

## 8. 页面美化技巧大全

### 8.1 文字样式

```js
// 红色文字（开头的全角引号 ‘’ 是固定前缀，缺了会被 App 当纯文本原样渲染）
'‘’这是红色文字'

// 文字颜色由前缀全角引号包裹 HTML 决定，不要用反引号（```` 会原样显示 <font> 标签）
'‘’<font color=#FF5D50>彩色文字</font>'

// 加粗
'<b>加粗文字</b>'

// 组合使用（注意：前缀必须是全角引号 ‘‘’’，不是反引号）
'‘‘’’<b><font color=#FF5D50>高亮选中</font></b>'

// 小字
('副标题').small()

// 多行拼接
title + '\n' + subtitle.small()
```

### 8.2 背景与边框

```js
// 按钮背景色
extra: { backgroundColor: '#20FA7298' }

// 隐藏底部分界线
extra: { lineVisible: false }

// 海报背景渐变
extra: { gradient: true }

// 分割线
col_type: 'line'

// 空白块分割
col_type: 'blank_block'
```

### 8.3 布局控制

```js
// 文本对齐
extra: { textAlign: 'left' }  // 左对齐（默认居中）

// 字体大小
extra: { textSize: 18 }  // long_text/rich_text 字体大小

// 行间距
extra: { lineSpacing: 6 }  // rich_text 行间距

// 标题跑马灯
col_type: 'movie_3_marquee'  // 标题过长自动滚动

// 图片模糊度控制（card_pic_1/2）
desc: '0'  // 0=不模糊，15=默认模糊，25=最大模糊
```

### 8.4 交互增强

```js
// 点击展开详情
url: $().lazyRule(function() { return 'toast://点击了'; })

// 切换状态 + 刷新
url: $('#noLoading#').lazyRule(function() {
    putMyVar('state', 'new');
    refreshPage(false);
    return 'hiker://empty';
})

// 输入框交互
col_type: 'input',
extra: {
    defaultValue: '默认值',
    onChange: $.toString(function() { putMyVar('val', input); }),
    type: 'text'  // 或 'password', 'number', 'textarea'
}

// 长按操作
extra: {
    longClick: [
        { title: '收藏', js: $.toString(function() { return 'hiker://collection'; }) },
        { title: '分享', js: $.toString(function() { copy('text'); return 'toast://已复制'; }) }
    ]
}
```

### 8.5 常见布局组合

**首页分类导航（美观方案）**：
```js
// 顶部 icon_5 分类导航 + 搜索框
d.push({
    title: '搜索影片',
    col_type: 'input',
    // ...
});
// 分类导航
for (var i = 0; i < classes.length; i++) {
    d.push({
        title: isActive ? '‘‘’’<b><font color=#FF5D50>' + name + '</font></b>' : name,
        col_type: 'icon_5',  // 或 scroll_button
        icon: iconUrl,
        // ...
    });
}
d.push({ col_type: 'blank_block' });  // 分割
```

**详情页标题美化**：
```js
// 标题行：片名 + 评分 + 年份
title: '‘‘’’' + name.bold()
    + '\n<font color="#FF5D50">评分：' + score + '</font>'
    + '  <font color="#9370db">' + year + '</font>'
    + '  <font color="#666">' + area + '</font>',
```

---

## 9. 完整布局模板

### 9.1 视频源首页完整模板

```js
// 首页引擎 - 分类导航 + 搜索 + 轮播 + 列表
home: function() {
    var d = [];
    var d_ = [];  // 前置队列（setPreResult）

    // === 第1行：搜索框 ===
    d_.push({
        title: '搜索',
        col_type: 'input',
        url: $.toString(function() {
            if (input) { putMyVar('keyword', input); refreshPage(); }
            return 'hiker://empty';
        }),
        extra: { defaultValue: getMyVar('keyword', '') }
    });

    // === 第2行：分类导航（scroll_button） ===
    var classes = [
        { title: '推荐', id: '0' },
        { title: '电影', id: '1' },
        { title: '电视剧', id: '2' },
        { title: '综艺', id: '3' }
    ];
    var currentId = getMyVar('classId', '0');
    for (var i = 0; i < classes.length; i++) {
        d_.push({
            title: currentId === classes[i].id
                ? '‘‘’’<b><font color=#FF5D50>' + classes[i].title + '</font></b>'
                : classes[i].title,
            col_type: 'scroll_button',
            url: $('#noLoading#').lazyRule(function(id) {
                putMyVar('classId', id); refreshPage(false);
                return 'hiker://empty';
            }, classes[i].id),
            extra: { backgroundColor: currentId === classes[i].id ? '#20FA7298' : '' }
        });
    }
    d_.push({ col_type: 'blank_block' });

    // === 第3行：过滤按钮 ===
    var filters = ['全部', '中国大陆', '美国', '日本'];
    var filterLocal = getMyVar('filterLocal', '全部');
    for (var i = 0; i < filters.length; i++) {
        d_.push({
            title: filterLocal === filters[i]
                ? '‘‘’’<b><font color=#FF5D50>' + filters[i] + '</font></b>'
                : filters[i],
            col_type: 'scroll_button',
            url: $('#noLoading#').lazyRule(function(val) {
                putMyVar('filterLocal', val); refreshPage();
                return 'hiker://empty';
            }, filters[i])
        });
    }
    d_.push({ col_type: 'blank_block' });

    // 先输出前置
    setPreResult(d_);

    // === 第4行：轮播（仅第一页） ===
    if (MY_PAGE === 1) {
        var lundata = getBannerData();
        callBanner('首页轮播', true, d, lundata, {
            time: 5000, col_type: 'card_pic_1', desc: '0'
        });
    }

    // === 第5行：分区标题 ===
    d.push({
        title: '<font color=#5cb85c>▍</font><b>热门推荐</b>',
        col_type: 'rich_text', extra: { lineVisible: false }
    });

    // === 第6行：内容列表 ===
    var list = getContentList(MY_PAGE, currentId, filterLocal);
    for (var i = 0; i < list.length; i++) {
        d.push({
            title: list[i].title,
            pic_url: list[i].pic + '@Referer=http://site.com/',
            desc: list[i].remark || '',
            url: 'hiker://page/detail',
            col_type: 'movie_3',
            extra: { id: list[i].id, title: list[i].title }
        });
    }

    setResult(d);
}
```

### 9.2 视频源详情页完整模板

```js
detail: function() {
    var d = [];
    var data = getDetailData(MY_PARAMS.id);
    if (!data) { setResult(d); return; }

    // === 第1行：海报卡 ===
    d.push({
        title: '‘‘’’' + data.title.bold() + '\n'
            + '<font color=#FF5D50>' + (data.score || '') + '</font>'
            + '  <font color=#9370db>' + (data.year || '') + '</font>'
            + '  <font color=#666>' + (data.area || '') + '</font>',
        desc: (data.actor ? '主演：' + data.actor : ''),
        pic_url: data.pic + '@Referer=http://site.com/',
        url: 'hiker://empty',
        col_type: 'movie_1_vertical_pic_blur',
        extra: { gradient: true }
    });

    // === 第2行：简介 ===
    if (data.intro) {
        setDesc(d, data.intro, 100);
    }

    // === 第3行：空白分割 ===
    d.push({ col_type: 'blank_block' });

    // === 第4行：小菜单（可选） ===
    d.push({
        title: '演职员', col_type: 'scroll_button',
        url: 'hiker://page/workers#noHistory#',
        extra: { id: data.id }
    });
    d.push({
        title: '剧照', col_type: 'scroll_button',
        url: 'hiker://page/photos#noHistory#',
        extra: { id: data.id }
    });

    // === 第5行：线路切换 ===
    var lines = data.playLines || [];
    var currentLine = parseInt(getMyVar('line_' + data.id, '0'));
    for (var i = 0; i < lines.length; i++) {
        d.push({
            title: currentLine === i
                ? '‘‘’’<b><font color=#FF5D50>' + lines[i].name + '</font></b>'
                : lines[i].name,
            col_type: 'scroll_button',
            url: $('#noLoading#').lazyRule(function(idx) {
                putMyVar('line_' + data.id, String(idx));
                refreshPage(false);
                return 'hiker://empty';
            }, i),
            extra: { backgroundColor: currentLine === i ? '#20FA7298' : '' }
        });
    }

    // === 第6行：排序切换 ===
    d.push({
        title: getMyVar('sort_' + data.id, '0') === '1' ? '降序' : '正序',
        col_type: 'text_center_1',
        url: $('#noLoading#').lazyRule(function() {
            var s = getMyVar('sort_' + data.id, '0');
            putMyVar('sort_' + data.id, s === '1' ? '0' : '1');
            refreshPage(false);
            return 'hiker://empty';
        })
    });

    // === 第7行：剧集列表 ===
    var episodes = lines[currentLine] ? lines[currentLine].episodes : [];
    var colType = episodes.length > 30 ? 'text_5'
        : episodes.length > 20 ? 'text_4'
        : episodes.length > 10 ? 'text_3'
        : 'text_2';
    if (getMyVar('sort_' + data.id, '0') === '1') episodes = episodes.slice().reverse();
    for (var i = 0; i < episodes.length; i++) {
        d.push({
            title: episodes[i].name,
            url: $().lazyRule(function(ep) {
                return playEpisode(ep);
            }, episodes[i]),
            col_type: colType,
            extra: { id: 'ep_' + data.id + '_' + i }
        });
    }

    setResult(d);
}
```

---

## 10. 常见布局缺陷与修复

### 10.1 首页空洞（只有列表，没有导航/搜索）

**问题**：首页直接输出列表，没有分类导航和搜索入口，用户无法筛选。

**修复**：在 `setResult` 之前先 `setPreResult` 输出导航和搜索框。

### 10.2 详情页只有剧集列表，没有海报卡

**问题**：点进详情后直接看到一排排的剧集编号，没有封面、简介。

**修复**：在剧集列表前先输出 `movie_1_vertical_pic_blur` 海报卡 + `setDesc` 简介。

### 10.3 详情页顶部（海报上方）留白过高

**问题**：详情页顶上总有高一块空白，海报卡没顶到屏幕顶，背景不贯通。

**修复（两步都做，缺一不可）**：① 海报卡 `title` 单行标题、`desc` 多行元信息（见 3.2 硬规范第 1 条），别把信息堆进 title；② **列表/搜索项跳转详情的 URL 加 `#immersiveTheme#`**（沉浸式顶格）。多数"留白修不掉"是只改了 ① 没加 ②。（iysdq/低端影视 实测：加 `#immersiveTheme#` 后留白即消失。）

### 10.4 所有剧集用同一 col_type，不管集数多少

**问题**：10集和100集都用 `text_2`，100集要翻很多屏。

**修复**：根据 `episodes.length` 动态选择 `text_2` ~ `text_5`。

### 10.5 选中态没有视觉反馈

**问题**：点了分类/线路后，没有任何高亮提示，用户不知道当前选的是哪个。

**修复**：比较当前值与选中值，用 `‘‘’’<b><font color=#FF5D50>text</font></b>` 高亮。**强调前缀固定为两个全角左单引号+两个全角右单引号 `‘‘’’`（`\u2018\u2018\u2019\u2019`）**，不是 `“”`、不是 `‘’‘’`(一左一右交替)。用错字符会让高亮态渲染成乱码/多余字符。反引号会原样渲染标签，勿用。

### 10.6 简介不能展开/收起，占用大量空间

**问题**：长的剧情简介直接显示，占了半屏。

**修复**：用 `setDesc` 函数封装可展开/收起的简介，或用 `long_text` 但不限制长度。

### 10.7 多线路重复列出所有剧集

**问题**：每个线路 tab 都显示同样的剧集列表，用户混淆。

**修复**：只取第一个有内容的线路组，线路切换在播放器内部完成。

### 10.8 图片加载失败（防盗链）

**问题**：图片无法显示，因为目标站做了 Referer 防盗链。

**修复**：在图片 URL 后加 `@Referer=http://target-site.com/` 或 `@headers={"Referer":"..."}`。

### 10.9 轮播不自动切换

**问题**：轮播展示后不自动切换下一张。

**修复**：使用 `registerTask` 注册定时任务，回调中调用 `updateItem` 更新。

---

## 11. 验证清单

交付前检查以下布局项：

- [ ] 首页有分类导航（`icon_5` 或 `scroll_button`）
- [ ] 首页有搜索入口（`input` 或 `search_url`）
- [ ] 选中分类/线路有高亮反馈
- [ ] 详情页有海报卡（`movie_1_vertical_pic_blur`，`url=封面'#.jpg#'`可点大图）
- [ ] 详情页简介**必须用 `setDesc`**（rich_text 展开/收起，3.3 方式1）；不得用 long_text/裸 `<a>`
- [ ] setDesc 的 `<a>` href 用**单引号**包裹（lazyRule 内全双引号）
- [ ] 剧集列表根据集数选择合适 col_type
- [ ] 图片URL加了防盗链处理（`@Referer`）
- [ ] 多线路时只取第一组剧集
- [ ] 主页滚动回顶部，不是从中间开始
- [ ] 搜索页和首页风格一致