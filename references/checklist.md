# 交付前自检清单（Final Checklist）

> 写任何海阔源在交付前，逐项过一遍本清单。**写复杂源（聚合/网盘/信息型/多页面）时本清单为必走流程。**

## A. 通用（所有源）

- [ ] 顶层字段齐全：`title` / `author` / `version` / `type` / `group` / `ua` / `url` / 对应 `find_rule`。
- [ ] 替换词正确：`fyclass` 分类、`fypage` 页码、`fyAll` 通用；分页替换词不在 URL 最末尾（末尾补无效串如 `?_t=0`）。
- [ ] 每个选择器/正则都对着真实 HTML 写过，没有一个是从头臆造的；能说出它命中页面哪一段。
- [ ] `col_type` 与内容匹配；`pages` 是用 JSON 字符串正确转义的数组。
- [ ] 逐级验证过：首页 → 分类/过滤 → 搜索 → 详情 → 选集 → 播放，每一级 App 实测命中正常。

## B. 首页 / 分类 / 搜索

- [ ] 分类名称与地址用 `&` 一一配对。
- [ ] 搜索格式 `列表;标题;链接;描述;详情;图片`（前 3 必选，缺省用 `*` 占位）；`**` 或 `%%` 关键词占位符用对（与站内字符冲突就换）。
- [ ] 搜索链接有没有编码问题：GBK 站要有 `;POST;GBK` 或对应 charset 处理。
- [ ] 懒加载图片：优先 `data-original`/`data-src`，备选 `src`；无封面图时该字段留空，不硬凑。
- [ ] 列表/资料放 HTML 单引号属性时用了正则而非 `pdfa`（口诀：`取属性，正则稳`）。

## C. 详情页

- [ ] 返回顺序为 `[海报卡] + [简介] + [（可选）小菜单] + [选集]`，不是只甩剧集列表。
- [ ] 海报/简介项不触发跳转：`url:'javascript:;'`（或 `hiker://empty`）。
- [ ] 多线路站详情只回一套去重后的剧集（线路在播放器内部切换），避免重复列。
- [ ] 多集播放记忆：每项 `extra.id` 是全局唯一值（详情页链接或"规则名+资源ID"），否则多集进度会串。

## D. 播放 / 链接分流

- [ ] 播放地址分流顺序正确：`pan://` → JSON 对象 `{urls}` → `video://`/`#isVideo=true#` → `.m3u8/.mp4/.flv` 加 `#isVideo=true#` → 其它 `http(s)` 包 `video://`。
- [ ] 需要 header 的播放/图片链接加了 `;{Cookie@...&&Referer@...}` 或 `@headers=`.
- [ ] 动态解析用对：点击才取真实地址用 `@lazyRule=`，深层嵌套用 `@rule=`/`.js:`。
- [ ] 多线路/弹幕/歌词用对象 JSON 字符串作为 url，字段名 `urls`/`names`/`danmu`/`headers` 正确。

## E. 特殊站点处理

- [ ] 登录/Cookie 源：取 Cookie 逻辑放 `preRule`，或登录态管理用 `setItem/getItem`。
- [ ] 有验证页/风控（403/503/`Just a moment`/`Checking your browser`/需等 5s）：已处理（验证码、Cookie、或提示用户）。
- [ ] GBK/GB2312 编码站已处理 charset，避免乱码。
- [ ] 聚合/网盘/信息型源：确认用了 `advanced-source-template.md` 骨架，缓存 key 含全部过滤维度，签名/UA/重试齐备，`openShare→parseShareUrl→ensureAuth→listShare→lazyRule` 五段完整。

## F. 交付

- [ ] 给出可直接导入的口令或 `.hk小程序.zip` 文件；多文件小程序含 `rule.json` + 所需 `libs.zip`/`data.zip`/`require.json`。
- [ ] 说明规则名、分组（`group`）、作者（`author`）。
- [ ] 复杂源已在 App 实测通过关键链路，附上实测说明。

> 自检通过后再交付；有任一 `[ ]` 未勾，先补齐再给用户。