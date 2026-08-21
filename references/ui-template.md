# 可复用 UI 渲染模板（UILib）—— AI 只填数据，渲染交给模板

> 来源：奈雪影视（成熟视频源）UI 布局技法蒸馏。核心理念：**把整套页面渲染逻辑（折叠筛选、轮播开关、随机色、快速搜索、TVBox 推送、线路/选集/排序）封进一个可复用的渲染模块，AI 写源时只负责"抓真实数据 + 按契约填数据对象"，排版与动效全交给 `UILib`。**

---

## 一、为什么要这么做

普通源把"抓数据"和"排版"混在一起写，每个源都重写一遍分类导航、折叠筛选、轮播、线路切换、TVBox 推送——重复劳动、样式不统一、改一处要全重来。

抽成 `UILib` 后：

- **站点差异压缩到最小**：AI 只写"抓取 + 字段映射"（这是各站真正的差异），不再手写排版。
- **样式/动效统一**：所有用该模板的源界面一致，"好看"直接继承。
- **一处修、处处生效**：折叠筛选、推送等逻辑只在 `UILib` 维持一份。

---

## 二、AI 填数据契约（唯一要遵守的接口）

`UILib` 暴露 4 个渲染入口，各自接收一个**固定结构的数据对象**。AI 只需要按下面结构填数据，其余别管。

### 2.1 `renderHome(data)` —— 首页

```js
var data = {
  sections: [        // 首页分区（必须，至少 1 个）
    {
      title: "正在热映",   // 分区标题（显示名）
      icon: "https://.../hot.svg", // 可选，分区图标
      colType: "movie_3",  // 可选，默认 movie_3；首板块可以为 movie_2
      max: 6,             // 可选，取前 N 条显示
      items: [           // 该分区影片
        { title, pic, desc, url }
      ]
    }
  ],
  navs: [           // 分类导航（可选）
    { title: "首页", id: "" },
    { title: "电影", id: "1" }
  ],
  filters: [        // 折叠筛选（可选）—— 分类模式用
    { label: "地区", key: "area", options: [{ name, value }] }
  ],
  search: {         // 搜索框（可选）
    placeholder: "搜索电影,电视剧..."
  },
  topBtns: {         // 搜索旁历史按钮（可选，默认显示 🕐🛠️）
    setting: "settings"  // 可选，长按/设置跳转的子页面 path，默认 "settings"
  },
  banner: {         // 轮播（可选）
    items: [...],   // 复用首页第1分区的 items 或自定义
    time: 4000, switchable: true
  },
  randomColor: true // 可选，按钮随机色
};
$.require("uil").renderHome(data);
```

> 搜索下方历史/设置按钮（青豆写法，**仅首页模式显示**）：`avatar` 行紧贴搜索框下方，`pic_url` 用系统图 `hiker://images/icon3`，`desc` 直接两个图标 `🕐 🛠️`（不加强调前缀/颜色）。**单击**进历史记录页 `hiker://history?rule=<源名>#noRecordHistory#`；**长按**弹菜单：🛠️管理(history) / ⭐收藏(collection) / ⚙️设置(跳 `hiker://page/<setting>`)。设置子页面默认 `path:"settings"`（各源按"设置子页面骨架"放配置项）。分类模式（筛选页）不显示此按钮。

### 2.2 `renderDetail(data)` —— 详情页

```js
var data = {
  poster: { title, desc, pic, url },  // 海报卡（movie_1_vertical_pic_blur）
  intro: "剧情简介文本",               // 可选，setDesc 折叠
  quickSearch: [                      // 可选，快速搜索/聚合搜索
    { name, pic, rule }               // rule 可选：为 "" 走 hiker://search，否则 copy://
  ],
  tvbox: {                           // 可选，TVBox 推送能力
    pushData: { name, pic, content, from, url } // 可选，推送当前列表用
  },
  tvboxShow: true,                   // 可选，源级开关：设 false 隐藏整块 TVBox 推送区块，默认显示
                                     // 只控制"是否渲染"，不动 UI.tvboxUI 与 push 子页代码
  lines: [                           // 线路（至少 1）
    { name: "线路1", episodes: [{ name, url }] }
  ],
  sortable: true                     // 可选，剧集正/倒序切换
};
$.require("uil").renderDetail(data);
```

### 2.3 `renderSearch/result(data)` —— 搜索/结果列表

```js
var data = { items: [{ title, pic, desc, url }] };
$.require("uil").renderSearch(data);
```

---

## 三、`UILib` 完整引擎代码（复制到子页面 `path:"uil"` 的 `rule`）

```js
js:
// ==== UILib 可复用 UI 渲染模块（奈雪影视技法蒸馏）====
(function () {
  var UI = {};
  // ★★ 必填：本源的引擎子页面 path（`pages` 里数据层那个引擎的 path，如 'iys'）。
  // 首页自绘搜索框会用它 `$.require(UI.engine).search()` 跳回本源子页执行搜索。
  // 不填或填错 → 首页点搜索无法回到本源引擎。接源时第一步就改成你的引擎 path。
  UI.engine = '引擎path';   // TODO: 替换成你引擎子页的 path
  // 强调前缀：两个左引号+两个右引号。源文件里既可直接敲 ‘‘’’（全角字符），
  // 也可用转义写法 '\u2018\u2018\u2019\u2019'（奈雪影视实际就是用 \u2018 转义）。
  // 注意：在 JSON 里存成 "\\u2018\\u2018\\u2019\\u2019"（双反斜杠）会被 JSON 解析成 \u2018 转义；
  // 在纯 JS 字符串里写 '\u2018\u2018\u2019\u2019'（单反斜杠）即代表这两个全角引号。
  // 搜索强调前缀时，直接搜可见字符 ‘‘’’ 或搜 '\u2018' 都行，两者是同一字符。
  var Q = '\u2018\u2018\u2019\u2019';

  // 随机色
  UI.rndColor = function () {
    return '#' + ('00000' + (Math.random() * 0x1000000 << 0).toString(16)).substr(-6);
  };

  // 高亮按钮
  UI.hi = function (text, active) {
    return active ? Q + '<b><font color="#FF5D50">' + text + '</font></b>' : text;
  };

  // 轮播（含开关）
  UI.banner = function (d, cfg) {
    var id = Math.random().toString(36).substr(2, 5) + 'lunbo';
    var items = cfg.items || [];
    if (!items.length) return;
    var time = cfg.time || 4000;
    var rnum = Math.floor(Math.random() * items.length);
    putMyVar('rnum', rnum);
    var first = items[rnum];
    d.push({
      col_type: 'pic_1_card', img: first.img || first.pic,
      desc: first.desc || '', title: first.title, url: first.url,
      extra: { id: id + 'bar' }
    });
    if (cfg.switchable === false) return;
    registerTask(id, time, $.toString(function (obj) {
      var id = obj.id;
      var data = obj.data;
      var i = Number(getMyVar('rnum', '0')) + 1;
      if (i > data.length - 1) i = 0;
      var item = data[i];
      try {
        updateItem(id + 'bar', { title: item.title, desc: item.desc || '', img: item.img || item.pic, url: item.url });
      } catch (e) { unRegisterTask(id); }
      putMyVar('rnum', i);
    }, { id: id, data: items }));
  };

  // 分类导航（scroll_button，高亮）。注意点首页时必须同时清全局筛选，否则返回首页残留筛选/分页状态。
  UI.navs = function (d, navs, current) {
    for (var i = 0; i < navs.length; i++) {
      var active = String(current) === String(navs[i].id);
      var sid = navs[i].id;
      d.push({
        title: UI.hi(navs[i].title, active),
        col_type: 'scroll_button',
        url: $('#noLoading##noRecordHistory#').lazyRule(function (s) {
          putMyVar('sd_cls', s); clearMyVar('sd_filter');
          putVar('sd_filter', '{}'); putVar('sd_fold', '0');  // 清全局筛选+折叠态
          refreshPage(true); return 'hiker://empty';
        }, sid),
        extra: { backgroundColor: active ? '#20FA7298' : '' }
      });
    }
    d.push({ col_type: 'line_blank' });
  };

  // 折叠筛选（推入主数组，保证“大类在上、筛选在下”）
  UI.filters = function (d, filters, filterKey, foldKey) {
    if (!filters.length) return;
    var s = d;   // 直接复用主数组，不再 setPreResult
    var fold = getVar(foldKey, '0');
    s.push({
      title: fold === '1' ? Q + '➥'.fontcolor('#9370db') : Q + '➦'.fontcolor('#ffbf00'),
      url: $('#noLoading#').lazyRule(function (fk, fold) {
        putVar(fk, fold === '1' ? '0' : '1'); refreshPage(false); return 'hiker://empty';
      }, foldKey, fold),
      col_type: 'scroll_button'
    });
    var state = JSON.parse(getVar(filterKey, '{}'));
    for (var fi = 0; fi < filters.length; fi++) {
      var section = filters[fi];
      if (fold !== '1' && fi > 0) break;   // 折叠时只显示第1维度
      for (var oi = 0; oi < section.options.length; oi++) {
        var opt = section.options[oi];
        var isActive = state[section.key] === opt.value;
        var t = opt.name;
        if (fold === '1' && fi > 0) t = Q + (isActive ? '<b>' + opt.name.fontcolor('#ffffff').small() + '</b>' : opt.name.small());
        else if (isActive) t = Q + opt.name.fontcolor('#ffffff').bold();
        s.push({
          title: t,
          url: $('#noLoading#').lazyRule(function (params) {
            var st = JSON.parse(getVar(params.k, '{}'));
            st[params.key] = params.val; putVar(params.k, JSON.stringify(st));
            refreshPage(true); return 'hiker://empty';
          }, { k: filterKey, key: section.key, val: opt.value }),
          col_type: 'scroll_button',
          extra: { backgroundColor: isActive ? UI.rndColor() : '' }
        });
      }
      s.push({ col_type: 'blank_block' });
    }
    d.push({ col_type: 'line_blank' });
  };

  // 分区标题（text_icon 带图标）
  UI.sectionTitle = function (d, title, icon, url) {
    d.push({
      title: '▍'.fontcolor(UI.rndColor()) + '<b>' + title + '</b>'.fontcolor('Black'),
      pic_url: icon || '', url: url || 'hiker://empty',
      col_type: 'text_icon'
    });
  };

  // 搜索下方历史/设置按钮：avatar 行紧贴搜索框下方，使用系统图标 icon3。
  // 单击 → 进历史记录页(hiker://history?rule=<源名>#noRecordHistory#)；
  // desc 直接两个图标 "🕐 🛠️"；长按弹菜单：🛠️管理 / ⭐收藏 / ⚙️设置。
  // 依赖：setting 子页面（默认 path "settings"，可用 cfg.setting 覆盖）。
  UI.topBtns = function (d, cfg) {
    var setting = (cfg && cfg.setting) || 'settings';
    var rule = MY_RULE.title;
    d.push({
      url: 'hiker://history?rule=' + rule + '#noRecordHistory#',
      pic_url: 'hiker://images/icon3',
      desc: '🕐 🛠️',
      col_type: 'avatar',
      title: '',
      extra: {
        longClick: [
          { title: '🛠️ 管理', js: $.toString(function (r) { return 'hiker://history?rule=' + r; }, rule) },
          { title: '⭐ 收藏', js: $.toString(function (r) { return 'hiker://collection?rule=' + r; }, rule) },
          { title: '⚙️ 设置', js: $.toString(function (s) { return 'hiker://page/' + s + '#noHistory##noRecordHistory#'; }, setting) }
        ]
      }
    });
  };

  // ---------- 首页 ----------
  UI.renderHome = function (data) {
    var d = [];
    var page = MY_PAGE;
    var cls = String(getMyVar('sd_cls', ''));

    if (!cls) {  // 首页模式（首页无分页，忽略 page 直接渲染，防止从筛选页返回首页时 page 残留）
      // 分类导航（首页也显示，首页按钮放第一位，点击进入分类模式）
      UI.navs(d, [{ title: '首页', id: '' }].concat(data.navs || []), '');
      // 搜索框（★ 必须开 fypage 子页再搜，绝不能内联直接跑 search()，否则 fetch 返回的 "error" 会被原生 ArticleListModel 当 URL 报 Expected URL scheme）
  //   使用说明：先在模块顶部把引擎子页 path 赋给本模块变量（如 UI.engine='iys'），
  //   否则默认要求你把自己的引擎子页 path 替换下方 `$.require(en)` 的执行参数。
  if (data.search) {
    d.push({
      title: '🔍 搜索',
      desc: data.search.placeholder || '',
      url: 'hiker://empty?page=fypage#noLoading##noHistory#@rule=js:' + $.toString(function (en) {
        if (input) { putMyVar('keyword', input); return $.require(en).search(); }
        return 'hiker://empty';
      }, UI.engine),
      col_type: 'input',
      extra: {
        onChange: $.toString(function () { putMyVar('keyword', input); }),
        defaultValue: getMyVar('keyword', '')
      }
    });
    d.push({ col_type: 'line_blank' });
  }
      // 搜索下方历史/设置按钮（仅首页显示）
      UI.topBtns(d, data.topBtns);
      // 轮播：显示与否由设置页 banner_on 控制，首页不再含开关按钮
      if (data.banner && getItem('banner_on', 'true') == 'true') {
        UI.banner(d, { items: data.banner.items || [], time: data.banner.time || 4000 });
        d.push({ col_type: 'line_blank' });
      }
      // 分区
      for (var si = 0; si < data.sections.length; si++) {
        var sec = data.sections[si];
        UI.sectionTitle(d, sec.title, sec.icon, sec.url);
        var max = sec.max || (si === 0 ? 6 : 9);
        var col = sec.colType || (si === 0 ? 'movie_2' : 'movie_3');
        for (var j = 0; j < sec.items.length && j < max; j++) {
          var it = sec.items[j];
          d.push({
            title: it.title, pic_url: it.pic, desc: it.desc || '',
            url: it.url, col_type: col
          });
        }
        d.push({ col_type: 'line_blank' });
      }
      setResult(d);
    } else {  // 分类模式
      // 分类导航（首页按钮第一位）
      UI.navs(d, [{ title: '首页', id: '' }].concat(data.navs || []), cls);
      // 折叠筛选（排在分类下方）
      UI.filters(d, data.filters || [], 'sd_filter', 'sd_fold');
      // 列表数据由各源在 homeData 里提供（AI 抓完塞入 data.listItems）
      var items = data.listItems || data.sections && data.sections[0] && data.sections[0].items || [];
      if (!items.length) { setResult([{ title: '暂无数据', col_type: 'text_center_1' }]); return; }
      for (var k = 0; k < items.length; k++) {
        d.push({ title: items[k].title, pic_url: items[k].pic, desc: items[k].desc || '', url: items[k].url, col_type: 'movie_3' });
      }
      setResult(d);
    }
  };

  // ---------- 搜索结果页 ----------
  // 数据契约：data.items = [{ title, pic, desc, url }]（由引擎 search() 只取数据后 $.require('uil').renderSearch({items}) 交付）
  // 渲染：首屏 loading（仅第1页）→ 结果列表 → 空态；setPreResult([]) 放 setResult(d) 前先清 loading。
  UI.renderSearch = function (data) {
    var d = [];
    var items = data.items || [];

    // ① 首屏先放 loading 占位，界面立即出现，数据随后 setResult 覆盖
    if (MY_PAGE == 1) {
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

    // ③ 空态提示（无结果时给用户明确反馈，而非白屏）
    if (!d.length) {
      d.push({ title: '未找到相关内容', desc: '请换个关键词重试', pic_url: '', url: 'javascript:;', col_type: 'list_1' });
    }

    // ④ 先清掉首屏 loading，再正式 setResult
    setPreResult([]);
    setResult(d);
  };

  // ---------- 详情页 ----------
  UI.renderDetail = function (data) {
    var d = [];
    var p = data.poster || {};
    // ① 海报卡
    d.push({
      title: p.title, desc: p.desc || '',
      pic_url: p.pic ? p.pic + '@Referer=' + (data.referer || '') : '',
      url: p.pic ? p.pic + '#.jpg#' : 'hiker://empty',
      col_type: 'movie_1_vertical_pic_blur',
      extra: { gradient: true }
    });
    // ② 简介 setDesc
    if (data.intro) UI.setDesc(d, data.intro, 60);
    d.push({ col_type: 'line' });
    // ③ 快速搜索（可选，受设置页开关 quickSearch_on 控制）
    if (getItem('quickSearch_on', 'true') == 'true' && data.quickSearch && data.quickSearch.length) {
      var qTitle = ''; try { qTitle = getPageTitle(); } catch (e) {}
      for (var q = 0; q < data.quickSearch.length; q++) {
        var qo = data.quickSearch[q];
        var isSys = !qo.rule || qo.name === '';
        var url = $('#noLoading#').lazyRule(function (obj, qTitle) {
          return obj.rule ? obj.rule
            : ('hiker://search?s=' + qTitle + '&rule=' + obj.name);
        }, qo, qTitle);
        d.push({ title: qo.name || '海阔搜索', pic_url: qo.pic || '', url: url, col_type: 'icon_round_small_4', extra: { id: 'qs' + q } });
      }
      d.push({ col_type: 'line_blank' });
    }
    // ④ TVBox 推送区域（默认显示；设置页 tvbox_on 真值 + 数据变量 tvboxShow 仍非 false 才渲染。UI.tvboxUI、webpush 单集推送按钮与 push 子页原样不改）
    if (getItem('tvbox_on', 'true') == 'true' && data.tvboxShow !== false && data.tvbox) UI.tvboxUI(d, data.tvbox);
    // ⑤ 排序（在最前）+ 线路 + 剧集
    var lines = data.lines || [];
    var cur = parseInt(getMyVar('sd_line', '0'), 10);
    if (cur < 0 || cur >= lines.length) cur = 0;
    if (data.sortable) {
      var sort = getMyVar('sd_sort', 'asc');
      d.push({
        title: Q + '<b><font color="#4285F4">' + (sort === 'asc' ? '正序 ▲' : '倒序 ▼') + '</font></b>',
        col_type: 'scroll_button',
        url: $('#noLoading#').lazyRule(function (old) {
          putMyVar('sd_sort', old === 'asc' ? 'desc' : 'asc'); refreshPage(); return 'hiker://empty';
        }, sort),
        extra: { backgroundColor: '#EEEEEE', textSize: 14 }
      });
    }
    if (lines.length > 1) {
      for (var li = 0; li < lines.length; li++) {
        var active = li === cur;
        d.push({
          title: UI.hi(lines[li].name, active), col_type: 'scroll_button',
          url: $('#noLoading#').lazyRule(function (idx) {
            putMyVar('sd_line', String(idx)); refreshPage(); return 'hiker://empty';
          }, li),
          extra: { backgroundColor: active ? '#20FA7298' : '', textSize: 14 }
      });
      }
    }
    d.push({ col_type: 'blank_block', extra: { height: 8 } });
    var eps = (lines[cur] && lines[cur].episodes) || [];
    if (data.sortable && getMyVar('sd_sort', 'asc') === 'desc') eps = eps.slice().reverse();
    var ct = eps.length > 50 ? 'text_5' : eps.length > 20 ? 'text_4' : eps.length > 8 ? 'text_3' : 'text_2';
    for (var j = 0; j < eps.length; j++) {
      d.push({
        title: eps[j].name || '-',
        url: $().lazyRule(function (pu) {
          try { return '' + pu + ''; } catch (e) { return 'toast://解析失败'; }
        }, eps[j].url),
        col_type: ct
      });
    }
    setResult(d);
  };

  // ---------- TVBox 推送（奈雪技法） ----------
  UI.tvboxUI = function (d, tvbox) {
    d.push({
      title: (getItem('webpush', '0') == '1' ? '单集推送🟢' : '单集推送🔴'),
      url: "hiker://empty@lazyRule=.js:getItem('webpush','0')=='1'?setItem('webpush','0'):setItem('webpush','1');refreshPage();'toast://切换成功！'",
      col_type: 'scroll_button'
    });
    d.push({
      title: '设置电视IP',
      url: "input://" + getItem('hikertvboxset', 'http://' + getIP() + ':9978') + "////请输入tvbox推送IP地址，如http://" + getIP() + ":9978.js:setItem('hikertvboxset',input);refreshPage()",
      col_type: 'scroll_button'
    });
    if (tvbox.pushData) {
      var pushJson = JSON.stringify(tvbox.pushData);
      d.push({
        title: '推送当前页列表到TVBox',
        url: $(pushJson).lazyRule(function () {
          return 'hiker://page/push?pushurl=' + encodeURIComponent(input);
        }),
        col_type: 'scroll_button'
      });
    }
    d.push({ col_type: 'blank_block', extra: { height: 8 } });
  };

  // ---------- 简介折叠（奈雪/河马技法） ----------
  UI.setDesc = function (d, desc, num) {
    if (!desc || !String(desc).trim()) return;
    num = num || 60;
    desc = String(desc).replace(/\s+/g, ' ').replace(/&nbsp;/g, ' ').trim();
    var head = '<b><font color="#098AC1">∷剧情简介\t</font></b>';
    var m = 'desc';
    var full = '　　' + desc;
    var sdc = desc.length > num ? '　　' + desc.substr(0, num) : full;
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
      title += "<small><a style='text-decoration:none;' href='" + lazy + "'>展开:</a></small>";
    }
    title += '<br><font color="grey">' + sdc + '</small>';
    d.push({ title: title, col_type: 'rich_text', url: 'javascript:;', extra: { id: m, lineSpacing: 6, textSize: 15, lineVisible: false } });
  };

  $.exports = UI;
})();
```

> ⚠️ 本模板为**初版渲染核心**：AI 填数据即可渲染首页/详情/搜索。剧集点击的"真正播放地址解析"（`pu` 那段）是各站差异，模板里给的是占位 `return ''+pu+''`，真实源需在 `data` 里把每个 `episode.url` 预先解析成最终可播地址，或替换模板中该 lazyRule 为各站的解析逻辑。PU 字段契约见下节。

---

## 三·五、TVBox 推送子页面（`pages` 里的独立子页，不属于 UILib 引擎对象）

TVBox 推送的"把当前页列表 POST 到电视机顶盒"这一步，是放在 `rule.json` 的 `pages` 里**一个独立子页面**（奈雪影视为 `path:"push"`）完成的。UILib 摘要页只负责在详情页造好 `hiker://page/push?pushurl=...` 的入口（见前文 `UI.tvboxUI`），真正的推送动作由这个子页面执行。

> 结构关系：`UILib`（`path:"uil"` 渲染库）与 `push`（`path:"push"` 推送执行器）是两个**不同的 `pages` 子页**，靠 `hiker://page/push?pushurl=xxx` 联动。写源时两个子页的代码都要写进 `pages` 数组。

### 完整子页面代码（放入 `pages`，`path:"push"`）

```jsonc
{
  "col_type": "movie_3",
  "name": "TVBox推送",
  "path": "push",
  "rule": "js: ··· 下面的代码 ··· "
}
```

```js
js:
// ==== TVBox 推送子页（奈雪影视技法）====
// 收到详情页 encoded pushData，POST 到 TVBox /action 完成推送
let items = [];
let pushin = decodeURIComponent(getParam("pushurl"));   // 收到编码后的推送数据
let tvip = getItem('hikertvboxset', '');               // 电视IP（跨页持久化）

// ① 当前IP + 可编辑入口
items.push({
  title: '当前IP：' + tvip,
  url: "input://" + getItem('hikertvboxset', 'http://' + getIP() + ':9978')
      + "////请输入TVBox端的IP地址，如http://" + getIP() + ":9978.js:setItem('hikertvboxset',input);refreshPage()",
  col_type: 'flex_button'
});

var purl = pushin;
// ② 数据形态分支：多集文本 / 直链 / 阿里云盘 / 网页投屏
if (pushin.includes('\n') && !pushin.startsWith('{')) {
  purl = pushin.replace("\n", '$');                    // 多集 "名$url 换行" → "$"
}
if (pushin.startsWith('http') && pushin.includes(';{')) {
  purl = JSON.stringify({ "name": "海阔直链推送", "url": pushin });
}
if (pushin.startsWith('https://www.aliyundrive.com/s/')) {
  let reftoken = JSON.parse(fetch('hiker://files/rules/icy/icy-ali-token.json', {}))[0].refresh_token;
  purl = JSON.stringify({ "url": pushin, "refresh_token": reftoken });  // 阿里云盘带 refresh_token
}
if (pushin.includes(':52020')) {
  purl = JSON.stringify({                                  // 网页投屏/投屏直链
    "name": pushin.includes('redirectPlayUrl') ? "海阔投屏直链" : "海阔网页投屏",
    "url": pushin
  });
}

// ③ POST 到 TVBox
var state = request(tvip + '/action', {
  headers: { 'Content-Type': 'application/x-www-form-urlencoded', 'Referer': tvip },
  timeout: 2000,
  body: 'do=push&url=' + purl.replace(/\&/g, '＆＆'),   // & 转义为全角，避免解析歧义
  method: 'POST'
});
// ④ 结果提示
items.push({
  title: (state == 'ok' ? "✅" : "❎") + state,
  url: "hiker://empty",
  col_type: "text_center_1"
});
setResult(items);
```

### 关键点

- **状态共用**：电视 IP 存在 `hikertvboxset`（`setItem/getItem` 持久化），detail 页的 `UI.tvboxUI` 与 push 子页都读写它；`webpush` 控制"单集推送开关"。
- **联动方式**：detail 页 `UI.tvboxUI` 里造 `$(pushData).lazyRule(function(){ return 'hiker://page/push?pushurl=' + encodeURIComponent(input) })`，把当前列表 `pushData` 传入子页。
- **`&` 转义**：POST 的 `body` 里 URL 的 `&` 必须 `.replace(/\&/g,'＆＆')`（转全角），否则被当作请求分隔参数、推送失败。
- 第三方多集数据形态按 ② 分支识别（`\n`多集 / http直链 / 阿里云盘 / 网页投屏）。

---

## 三·六、快速搜索的实现方法（奈雪影视 `quickSearchConfigs` 完整技法）

快速搜索是详情页里一排圆形图标（`icon_round_small_4`），点击后**用当前影片标题去别的源/规则再搜一遍**（"聚影""香情""云盘君"等），实现"我装了一源，放遍全网"。核心是 4 点：**定义配置 → 判断规则是否已装 → 三态跳转（已装去搜 / 未装给口令 / 依赖需安装）→ 展示图标栏**。

### ① 配置结构：`quickSearchConfigs`

```js
var qSearchRules = {
  "quickSearchConfigs": {
    "mode": "icon_round_small_4",          // col_type：一排圆形小图标
    "order": ["海阔搜索", "云盘君.简", "香情", "聚影"],  // 显示顺序
    // 每个成员 3 个子：name 规则名、pic 图标、rule 可选(安装口令)
    "海阔搜索": { "name": "", "pic": "https://.../135.svg" },           // name="" → 走海阔系统搜索
    "云盘君.简": { "name": "云盘君.简", "pic": "https://...jpg", "rule": "copy://https://netcut.cn/p/xxxx" },
    "香情": { "name": "香情影视", "pic": "https://.../xqys.png" },       // 无 rule → 未装时提示"请安装"
    "聚影": { "name": "聚影√", "pic": "https://.../QqyC.png", "rule": "copy:https://pasteme.tyrantg.com/.../vkvhbqop" }
  }
};
```

### ② 详情页渲染（三态跳转逻辑）

```js
// 在 detail() 里，取出配置并遍历 order 渲染图标
var qTitle = ''; try { qTitle = getPageTitle(); } catch (e) {}   // 当前影片名
var qList = qSearchRules.quickSearchConfigs.order;
for (var ql in qList) {
  var qObj = qSearchRules.quickSearchConfigs[qList[ql]];
  d.push({
    title: qList[ql],
    url: $('#noLoading#').lazyRule(function (obj, qTitle) {
      // ① 判断该规则（obj.name）是否已安装：request("hiker://home") 返回全部已装规则 JSON
      var ruleList = JSON.parse(request("hiker://home"));
      var hasRule = ruleList.some(function (item) { return item.title === obj.name; });
      if (hasRule || obj.name == '') {
        // ② 已装 / 海阔搜索 → 用当前片名去该来源搜
        return 'hiker://search?s=' + qTitle + '&rule=' + obj.name;
      } else {
        // ③ 未装：若有安装口令(rule) 则给 copy，否则提示需安装
        if (obj.hasOwnProperty("rule")) {
          if (obj.rule.indexOf('copy') != -1) { toast("已经复制请导入规则"); }
          return obj.rule;    // copy:// 口令，点击即复制导入链接
        } else {
          return "toast://请安装" + obj.name + "小程序";
        }
      }
    }, qObj, qTitle),
    pic_url: qObj.pic,
    col_type: qSearchRules.quickSearchConfigs.mode,
    extra: { id: 'qsearch' }
  });
}
d.push({ col_type: 'line_blank' });
```

### ③ 三态跳转语义（务必区分）

| 分支 | 判断 | 返回 | 效果 |
|---|---|---|---|
| 已安装 | `hasRule === true` | `hiker://search?s={片名}&rule={规则名}` | 跳去该规则的搜索页搜片名 |
| 海阔系统搜索 | `obj.name == ''` | 同上 `rule=` 留空 | 走海阔 App 自带聚合搜索 |
| 未装 + 有口令 | `obj.hasOwnProperty('rule')` | `copy://...`（含 `copy:`） | 弹 toast"已复制请导入"，复制导入链接 |
| 未装 + 无口令 | 其它 | `toast://请安装{名}` | 提示用户手动装 |

### ④ 关键点

- **判定"已装"**：`request("hiker://home")` 返回当前所有已装规则（JSON 数组），`ruleList.some(item => item.title === 规则名)` 判存在。注意 `name:'海阔搜索'` 用空串 `''` 标识（它是系统规则，永远算"已装"）。
- **`rule` 字段**：写 `copy://完整链接` 或 `copy:完整链接`（注意 `://` vs `:` 都能被复识别）；点击时 `toast('已经复制请导入规则')` 后返回该口令实现"一键复制导入"。
- **`pic_url` 是图标圆按钮**：`icon_round_small_4`（一排 4 个圆形图标）；无图标成员可省 `pic`。
- **依赖安装引导**：这属于"前置依赖安装"的 UI 实现，与 `preRule` 里的 `confirm/request` 依赖检查是同族能力，但快速搜索是"详情页当点击时"才触发，比 preRule 预检查更省、更精准。

---

## 四、AI 使用流程（三步接一个源）

### 步骤 1：抓真实数据（各站差异，AI 负责）
```js
function fetchHomeData() {
  var html = fetch(_H + '/');
  return { sections: parseSections(html), navs: parseNav(html) };
}
```

### 步骤 2：映射成契约结构
```js
var data = {
  sections: [...], navs: [...], filters: [...],
  banner: { items: [...], time: 4000, switchable: true }
};
```

### 步骤 3：交给 UILib 渲染
```js
function home() { $.require('uil').renderHome(fetchHomeData()); }
function detail() { $.require('uil').renderDetail(fetchDetailData(MY_URL)); }
```

---

## 四·一、数据层如何适配 UILib（三入口改造规范）

**核心边界：数据层只"抓取 + 填契约字段"，绝不再碰任何排版字段（配色、高亮、`col_type`、`extra`、`取几条`等）。** 排版 100% 交给 `UILib`。凡是改造成 `$.require('uil').renderXxx(data)` 的源，其数据层函数里就不允许再出现 `d.push({ col_type... })` 的散装 UI 代码（`setPreResult` 的折叠筛选、`banner`、`setDesc` 同理全部搬进 UILib）。

### 三入口改造前后对照

**① home() —— 返回纯数据给 renderHome**

```js
// ❌ 改前（排版混在数据层）：自己 push 轮播/搜索框/导航/板块标题/col_type，最后 setResult(d)
function home() {
  var d = [];
  d.push({ title: '🔍 搜索', col_type: 'input', ... });
  d.push({ title: '轮播', col_type: 'pic_1_card', ... });
  for (...) d.push({ title: it.title, col_type: 'movie_3', ... });
  setResult(d);
}

// ✅ 改后：只组装数据对象，交给 UILib 渲染
function homeData() {          // 数据层只负责抓 + 映射
  var html = _fetch(_H + '/');
  if (!html) return { sections: [] };
  return {
    search: { placeholder: '搜索电影,电视剧...' },
    banner: extractBanner(html),             // 产出 banner.items/time/switchable
    navs: [{ title: '电影', id: '1' }],        // 分类导航，id 对应分类页标识
    filters: parseFilters(html),             // 折叠筛选 { label,key,options }
    sections: parseSections(html),           // 各板块 items:[{title,pic,desc,url}]
    listItems: parseList(html)               // 分类模式下的列表数据
  };
}
function home() { $.require('uil').renderHome(homeData()); }  // MY_PAGE/MY_URL 仍由数据层读取后塞进数据
```

**② detail() —— 返回纯数据给 renderDetail**

```js
function detailData() {
  var res = _fetch(MY_URL);
  return {
    poster: { title, desc, pic, url },        // 海报卡字段
    intro: '剧情简介文本',                    // setDesc 折叠由 UILib 做
    quickSearch: [{ name, pic, rule }],       // 快速搜索图标栏
    tvbox: { pushData: { name, pic, content, from, url } },  // TVBox 推送数据
    tvboxShow: true,                          // 源级开关：false 隐藏整块推送区块（可不写，默认显示）
    lines: [{ name, episodes: [{ name, url }] }],  // 线路 + 剧集（**url 须为最终可播地址**）
    sortable: true                            // 剧集正/倒序
  };
}
function detail() { $.require('uil').renderDetail(detailData()); }
```

**③ search() —— 返回 { items } 给 renderSearch**

```js
function search() {
  var items = JSON.parse(_fetch(_H + '/index.php/ajax/suggest?...&wd=' + MY_KEYWORD)).list
    .map(function (it) { return { title: it.name, pic: it.pic, desc: it.en, url: _full(...) }; });
  $.require('uil').renderSearch({ items: items });
}
```

### 留在数据层、绝不能移走的（各站真正差异）

| 保留项 | 原因 |
|---|---|
| `parseList`/正则抓字段（片名/海报/简介/剧集href） | 每个站 DOM/接口不同，UILib 无法代劳 |
| **播放地址解析**（`_fetch(pu)` + 正则提取 m3u8/mp4、二次解析 playerUrl、线路去重） | UILib 的 `episode.url` 直接透传，真实源必须在数据层把每个 `episode.url` 预先算成最终可播地址（或替换模板中占位 lazyRule） |
| 线路/剧集的**结构组装** | 只产出 `lines[].episodes[].name/url`，不碰任何渲染字段 |

### 同名函数改造守则（迁移存量源时必读）

把旧源改成 UILib 时，旧 `home()/detail()/search()` 里**所有函数级 UI**（`banner()`、`navBar()`、`setDesc()`、折叠筛选的 `setPreResult(s)`、线路/排序/剧集渲染循环）全部删掉，只保留抓取函数与正则。若同一个文件名重复（旧 `home` 和新 `homeData`），建议按上面的命名 `xxxData()` 返回数据、`xxx()` 只调 `renderXxx`，让"数据"与"渲染"在代码层面也能一眼区分。

### 子页面职责分离（谁写什么，务必分清）

UILib 模式下，`pages` 数组里至少有 **3 个子页面**，职责各一、缺一不可：

| 子页 path | 职责 | 由谁产出 |
|---|---|---|
| **引擎子页**（如 `kuke`/`xxx`） | **数据层**：抓取 + `homeData/detailData/search`，入口函数里调 `$.require('uil').renderXxx(data)` | **数据层 AI 写** |
| **`uil`** | **渲染库**：`UI.renderHome/renderDetail/renderSearch`，`$.exports = UI` | 模板代码（`三、`节的引擎）机器复制 |
| **`push`** | TVBox 推送执行器（`path:'push'`） | 模板代码（`三·五`节）机器复制，TVBox 用才需要 |

> ⚠️ 铁律：`$.require('uil')` 永远指向 `path:'uil'` 那个子页，**绝不指向引擎子页自己**。引擎子页只放抓取 + 约定，渲染函数（`renderXxx`）必须在 `uil` 子页里。数据层里**不得**出现 `renderHome/renderDetail` 的函数定义，只能 `$.require('uil').renderDetail(...)` 调用。

---

## 四·二、UILib 模式的顶层字段固定配方（rule.json 骨架）

底层 schema 见 SKILL.md「rule.json 顶层字段」。用 UILib 时**顶层字段必须按下述固定配方**写，否则 `require` 失败或与渲染层冲突。可直接套这份骨架：

```jsonc
{
  "last_chapter_rule": "",
  "title": "xxx影视",        // 分享/导入名
  "author": "AI",
  "version": 1,              // 每次更新 +1
  "type": "video",           // video/music/cartoon/news/read/live
  "group": "AI生成",          // AI 写源默认分组
  "ua": "mobile",
  "url": "https://site.com/show/fyclass--------fypage---.html",   // 首页地址，替换词照通用规则
  "col_type": "movie_3",      // ★ UILib 首页排版生效，可占位；首板块 movie_2 由 UILib 接管
  "find_rule": "js:$.require('xxx').home()",          // ★ 数据层 home 入口
  "detail_col_type": "text_3",                        // ★ 占位即可，UILib 覆盖
  "detail_find_rule": "js:$.require('xxx').detail()", // ★ 数据层 detail 入口
  "sdetail_col_type": "*", "sdetail_find_rule": "*",   // 三级继承
  "search_url": "https://site.com/so/-------------.html?wd=**",  // 照通用规则（也可在数据层用 suggest 接口）
  "searchFind": "js:$.require('xxx').search()",        // ★ 数据层 search 入口（如用同步 js 检索则必写）
  "preRule": "",              // 有需求才写（token/http/cookie/依赖检查等）
  "pages": "[
    { \"name\": \"xxx引擎\", \"path\": \"xxx\", \"col_type\": \"movie_3\", \"rule\": \"js: <数据层代码>\" },
    { \"name\": \"UILib\", \"path\": \"uil\", \"col_type\": \"movie_3\", \"rule\": \"js: <三、节 UILib 引擎代码>\" },   // ★ 必须
    { \"name\": \"TVBox推送\", \"path\": \"push\", \"col_type\": \"movie_3\", \"rule\": \"js: <三·五节推送代码>\" }        // TVBox 用才必须
  ]",
  "icon": "#ff6354ef"         // 可省
}
```

### 顶层字段 ↔ 数据层入口绑定表

| 顶层字段 | 值（UILib 模式固定写法） | 对应数据层函数 |
|---|---|---|
| `find_rule` | `js:$.require('引擎path').home()` | 引擎子页 `home()` |
| `searchFind` | `js:$.require('引擎path').search()` | 引擎子页 `search()` |
| `detail_find_rule` | `js:$.require('引擎path').detail()` | 引擎子页 `detail()` |
| `pages` | 含 引擎 / `uil`（必）/ `push`（TVBox 时必） | 各子页代码 |

### 关键约束（用 UILib 时顶层字段必须遵守）

- **`find_rule/searchFind/detail_find_rule` 三个入口全写成 `js:$.require('<引擎path>').home()/search()/detail()`**，不要用选择器，也不要直接写散装 `setResult`。
- **`col_type/detail_col_type` 给占位即可**（`movie_3`/`text_3`），UILib 渲染时会覆盖真正排版，别在顶层精心设计。
- **`pages` 必须有 `path:'uil'` 子页**，否则 `$.require('uil')` 直接失败；引擎子页 `path` 与顶层 `$.require('<引擎path>')` 必须一致（本例为 `xxx`）。
- `url`（首页地址）与 `search_url` 仍按通用替找词写；若站点搜索走 JSON suggest，也可不在顶层写 `search_url`、直接让 `search()` 用 `MY_KEYWORD` 请求接口（见 `四·一` 搜索样例）。

---

## 五、数据契约字段速查表

| 入口 | 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|---|
| renderHome | `sections[].items[].title/pic/url` | string | ✅ | 卡片数据 |
| renderHome | `sections[].colType/max/icon` | string/number | | 分区样式 |
| renderHome | `navs[].title/id` | string | | 分类导航 |
| renderHome | `filters[].label/key/options` | object[] | | 折叠筛选 |
| renderHome | `banner.items/time/switchable` | object | | 轮播+开关 |
| renderHome | `topBtns.setting` | string | | 设置子页面 path，默认 `settings` |
| renderDetail | `poster.title/desc/pic/url` | object | ✅ | 海报卡 |
| renderDetail | `intro` | string | | 折叠简介 |
| renderDetail | `lines[].name/episodes[]` | object[] | ✅ | 线路+剧集 |
| renderDetail | `quickSearch[].name/pic/rule` | object[] | | 快速搜索 |
| renderDetail | `tvbox.pushData` | object | | TVBox 推送 |
| renderDetail | `tvboxShow` | boolean | | 源级开关，默认 true；false 隐藏整块 TVBox 推送区块 |

---

## 五·五、设置子页面骨架（历史/设置按钮默认跳到这里）

顶部"设置"按钮默认跳 `hiker://page/settings`。`pages` 里需有 `path:"settings"` 的子页面；它是一份**普通列页面**，里面放源的配置项（用 `text_1` 开关 + `setItem` 持久化）。默认已内置三个功能开关：**轮播开关** `banner_on`（控制首页轮播显示）、**快速搜索开关** `quickSearch_on`、**TVBox推送开关** `tvbox_on`（值域均 `true/false`，其中 `tvbox_on` 控制详情页整块 TVBox 推送区域的显示，与 renderDetail 判断一致）。开关改动即时生效，可在骨架下方继续追加其它配置项。

```jsonc
// pages[] 里新增：
{ "name": "设置", "path": "settings", "col_type": "text_1", "rule": "js: $.require('SETTINGS_页path').main()" }
```

```js
// path:"settings" 的 rule（数据层放一个 main()）：
js:(function () {
  var d = [];
  // 标题头
  d.push({ title: '设置', col_type: 'text_center_1' });
  d.push({ col_type: 'blank_block' });

  // 通用开关渲染（setItem 持久化，取值 'true'/'false'）
  function sw(key, label, def) {
    var on = getItem(key, def) == 'true';
    d.push({
      title: (on ? '✅ ' : '⬜ ') + label,
      url: $('#noLoading#').lazyRule(function (k, dflt) {
        var cur = getItem(k, dflt) == 'true';
        setItem(k, cur ? 'false' : 'true');
        refreshPage(false); return 'hiker://empty';
      }, key, def),
      col_type: 'text_1'
    });
  }

  d.push({ title: '● 功能开关（立即生效）', col_type: 'text_center_1' });
  sw('banner_on', '轮播开关', 'true');          // 首页轮播显示开关（首页轮播渲染时读它，首页不含开关按钮）
  sw('quickSearch_on', '快速搜索开关', 'true');  // 详情页快速搜索栏
  sw('tvbox_on', 'TVBox推送开关', 'true');       // 详情页整块 TVBox 推送区域（默认开，值域 true/false，与 renderDetail 判断一致）

  d.push({ col_type: 'blank_block' });
  setResult(d);
})()
```

---

## 六、验证清单

- [ ] 首页渲染出：搜索框 + 轮播(开关) + 分区（首块 movie_2 其余 movie_3）
- [ ] 首页搜索框下方历史按钮：avatar(icon3) 显示 🕐 🛠️；单击进 history，长按弹 🛠️管理/⭐收藏/⚙️设置；设置跳 settings 子页；分类(筛选)页不显示
- [ ] 设置页含三个开关：轮播(banner_on)、快速搜索(quickSearch_on)、TVBox(tvbox_on)，切换即时生效
- [ ] 分类模式：`sd_cls` 切换分类，折叠筛选置顶，`refreshPage` 生效
- [ ] 详情页：海报卡 + 折叠简介 + 线路切换 + 排序 + 剧集（动态列数）；排序按钮与分线同排、不另起一行
- [ ] 快速搜索：`hiker://search?s=&rule=` 或 `copy://` 都能跳
- [ ] TVBox：tvbox_on 开关（关→整块推送区域隐藏）、IP 设置、推列表均可用
- [ ] 详情页数据里 `tvboxShow:false` 时整块 TVBox 推送区块被隐藏，且 `UI.tvboxUI`/push 子页代码未被改动
- [ ] 强调前缀已用 `‘’‘’`（`\u2018\u2018\u2019\u2019`）