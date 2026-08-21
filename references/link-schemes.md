# 特殊协议链接、增强写法与页面标识

小程序结果中的 `url` 不止普通网址。以下协议/写法会被 App 特殊处理。

## 系统级链接（hiker://…）

- `hiker://home`：展开全部小程序；`hiker://home@规则名`（或 `规则1||规则2||http://…` 逐个回退）。
- `hiker://search?s=关键词&group=分组&rule=规则名`：打开搜索页（带关键词/自动分组/固定规则）。
- `hiker://history` / `hiker://collection` / `hiker://download` / `hiker://bookmark`：对应页面。
- `hiker://debug?url=xxx`：开发助手（看源码）。
- `hiker://page/标识`：跳子页面；`?rule=<规则名>` 用其它规则的子页面。
- `hiker://empty`：返回空字符串（配合 `#http://a@rule=js:fetch(...)` 在二级手动请求）。
- `hiker://explore?path=...` / `hiker://localMedia` / `hiker://webSearch` / `hiker://webdav` / `hiker://webRule`。

## 动作类协议

| 协议 | 作用 |
| --- | --- |
| `toast://文本` | 提示信息 |
| `rule://base64口令` / `海阔视界，…` | 识别并导入规则口令 |
| `javascript:代码` | 彩蛋模式，浏览器执行 JS |
| `web://http://…` | 强制网页打开（忽略二级/DOM 分析） |
| `x5://http://…`、`x5Play://` | X5 全屏 / X5 播放器播放 |
| `x5WebView://http://…`、`webRule://`、`x5Rule://` | 嗅探/获取资源，`链接@JS代码` |
| `video://网页` | 自动提取网页视频（具体嗅探规则可用 `extra`） |
| `pics://a&&b` | 多图模式（漫画自动下一章） |
| `copy://文本`、`confirm://提示.js:…`、`input://{json}`、`select://{json}`、`editFile://`、`openFile://`、`download://地址`、`share://路径`、`fileSelect://js` | 剪贴板/确认/输入/选择框/编辑/打开/下载/分享/选文件 |

## 播放地址增强

- **带 header 直链**：`https://a.com/2.mp4;{Cookie@xxx&&Referer=a.com}`。
- **多线路/字幕/弹幕/歌词**（url 用对象 JSON）：
  - `{urls:['http://1.mp4','http://2.mp4'],names:['超清','高清']}`
  - `{urls:[...], headers:[{'Referer':'x'},{'Referer':'y'}]}`（header 内含英文 `;` 用两个中文 `；；`）
  - `{urls:[...], subtitle:'http://1.srt'}`、`{urls:[...], danmu:'http://1.xml'|'1.json'|'web://...'}`、`{urls:['...mp3'], lyric:'http://1.lrc'}`
- **音频/视频分离**：`{url: JSON.stringify({urls:[u], audioUrls:[a]}), col_type:'text_3'}`。
- **分段连接**：多个播放地址用 `#concat#` 连接。
- **极速播放**：`#fastPlayMode#`（可加 `#threads=10#`）。
- 无条件仍强识别：`.m3u8` 缺识别时加 `#isM3u8#`；mp4 误识别加 `#ignoreM3U8#`。
- **播放进度记忆**：多选级给 `extra.id` 设全局唯一值。

## 动态解析 / 深层嵌套

参考 `selector-syntax.md` 的"动态解析与 $ 工具"一节。要点：

- 链接后 `@lazyRule=规则`（结尾才能用，`&&` 转义为 `＆＆＆＆`），点击时才解析真资源。
- 深层（三级四级）用多个 `@rule=`。
- 代价大时用 `$('url').lazyRule(fn)` / `$('url').rule(fn)` 生成。

## 页面标识（url 上 `#xxx#`，请求时自动清除）

| 标识 | 作用 |
| --- | --- |
| `#noLoading#` | 不显示 loading |
| `#noHistory#` | 不记录足迹（放在跳转这一级的链接上） |
| `#noRecordHistory#` | 不记录历史记录 |
| `#noRefresh#` | 禁止下拉刷新 |
| `#immersiveTheme#` | 沉浸式（仅二级/子页面） |
| `#fullTheme#` | 全屏 |
| `#readTheme#` | 阅读模式（点击/音量键翻页） |
| `#autoPage#` | 自动翻页（配合阅读，自动加载下一章节） |
| `#gameTheme#` | 游戏模式 |
| `#autoCache#` / `#cacheOnly#` | 页面缓存秒开（勿用于正文页） |
| `#background#` | 后台播放音频（前台通知防杀） |
| `#pre#` / `#noPre#` | 强制预加载 / 禁止预加载 |
| `#originalSize#` | 大图原图尺寸（超大图慎用） |
| `#memoryPage#` | 自动记忆翻页页数 |
| `#isVideo=true#` / `#ignoreVideo=true#` | 强制视频 / 非视频 |
| `#isMusic=true#` / `#ignoreMusic=true#` | 强制音频 / 非音频 |
| `#ignoreImg=true#` | 非图片 |
| `#checkMetadata=false#` | 不自动获取音频元数据 |

## 网络代理 / Cookie

- Cookie：默认自动带 `Set-Cookie`；手动注入优先；强制无 Cookie 用 `headers:{Cookie:'#noCookie#'}`。
- 浏览器代理替换：`addWebProxyRule({name, match, replace, requestHeaders, responseHeaders})`、`removeWebProxyRule(name)`。

## 其它数据格式

> 完整细节与在线手册可参考海阔开发者文档库（`docs.189.tyrantg.com`）。以上为本 skill 依据官方 help 提炼的可复用要点；接口含义如有疑问，以实际 App 运行结果为准。