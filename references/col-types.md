# 显示样式 col_type 参考

`col_type` 决定每条结果在列表中的呈现方式。首页用 `rule` 顶层字段统一指定；二级/`js:` 结果里可对单个对象指定 `col_type`（非 js 解析不支持混用）。

## 影视 / 图文卡片

| col_type | 说明 |
| --- | --- |
| `movie_3` | 一行三列，图文+描述，圆角矩形（默认影视样式） |
| `movie_3_marquee` | 同 movie_3，标题过长跑马灯 |
| `movie_2` | 一行两列，直播或视频 |
| `movie_1` | 一行一列，标题/描述多时用 |
| `movie_1_left_pic` | 图片在左，标题/描述在右 |
| `movie_1_vertical_pic` | 图片在左且竖图（高>宽） |
| `movie_1_vertical_pic_blur` | 同上，背景为图片高斯模糊 |
| `icon_1_left_pic` | 左图小一些，适合放图标 |
| `card_pic_1` / `card_pic_2` | 方形卡片，独占一行 / 一行两列，图片默认高斯模糊（desc 0-25 控制模糊度，默认 15，0 不模糊） |
| `card_pic_2_2` / `card_pic_2_2_left` | 方形卡片，上下两张，只能右侧 / 只能左侧 |
| `card_pic_3` / `card_pic_3_center` | 一行三列圆角矩形，居中 / 不居中 |

## 纯图片

| col_type | 说明 |
| --- | --- |
| `pic_3` / `pic_3_square` | 一行三列，只显示图片 / 正方形 |
| `pic_2` / `pic_2_card` | 一行两列，直角矩形 / 竖图卡片（壁纸） |
| `pic_1` / `pic_1_full` / `pic_1_center` | 一行一列大图 / 通栏自适应 / 居中自适应（验证码等小图） |

## 文本

| col_type | 说明 |
| --- | --- |
| `text_1` | 一行一列文本；支持红/橙混排（`““xx””`红字、`''xx''`橙字） |
| `text_center_1` | 一行一列，居中 |
| `text_2`/`text_3`/`text_4`/`text_5` | 一行 N 列（左对齐用 `extra:{textAlign:'left'}`） |
| `long_text` | 不限长度长文本（`extra:{textSize:18}`） |
| `rich_text` | 富文本 HTML（`extra:{textSize, lineSpacing, click}`；图片 `src...@originalSize#` 不横向放大） |

## 图标 / 按钮

| col_type | 说明 |
| --- | --- |
| `icon_2` / `icon_2_round` | 一行两列，左图标（圆形）右标题 |
| `icon_3_round_fill` / `icon_3_fill` | 一行三列圆形图标/图标+文字 |
| `icon_4` / `icon_4_card` / `icon_small_4` / `icon_round_4` / `icon_round_small_4` | 一行四列，顶图底标签（可圆角/裁圆/缩小） |
| `icon_small_3` | 一行三列，左图右标签（点赞/回复/浏览） |
| `icon_5` / `icon_5_no_crop` | 一行五列，圆形 / 不裁剪 |
| `flex_button` / `scroll_button` | 自适应流式 / 滚动按钮（`extra:{'backgroundColor':'#666'} `），可 html 标题 |
| `text_icon` | 左文字右图标 |

## 其它

| col_type | 说明 |
| --- | --- |
| `line` / `line_blank` | 分割线 / 空白块分割线 |
| `blank_block` | 空白块（宽屏、高 1dp） |
| `avatar` | 头像（需 title+图片，可选 desc 富文本） |
| `input` | 单行输入框（`extra:{onChange, defaultValue, type:'password'|'number'|'textarea', titleVisible}`） |
| `icon_1_search` | 搜索图标输入框（不可输入，触发点击） |
| `video` | 内嵌视频组件（基于 X5，不能和 x5 组件共用；用直链，`extra:{'Referer':'a'}`，动态刷新用 `refreshVideoSource`） |
| `x5_webview_single` | 内嵌 X5 网页（desc 控制高度：`240`、`auto`、`float&&...`；`extra:{ua, js, blockRules, floatVideo, autoPlay, canBack}`） |

## extra 常用通用项

- `lineVisible:false`：隐藏底部分界线（部分样式）。
- `pageTitle`：定制下一页面标题；`inheritTitle:false` 不继承。
- `newWindow:true`（+`windowId`）：新窗口打开。
- `id`：元素唯一 ID（播放进度记忆、`updateItem/deleteItem/findItem` 用）。
- `cls`：类标识，空格分隔多个（连续选集聚合、`findItemsByCls` 用）。
- `longClick:[{title,js}]`：自定义长按操作。
- 播放多线路/字幕/弹幕/歌词：见"播放与详情链接"与 `link-schemes.md`。