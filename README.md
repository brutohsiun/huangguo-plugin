# 黄果短剧 · HGhub 插件

按 [HGhub 插件协议（HPP 1.0）](https://github.com/Minireel/HGhub/blob/main/docs/plugin-protocol/README.md) 编写的单文件插件，接入 [huangguoai.com](https://huangguoai.com) 的短剧内容。

> **含成人内容**：`meta.mature = true`，宿主默认隐藏，需在设置里打开成人内容后才会出现。
> 本仓库只包含解析代码，不提供、不存储任何音视频内容。

## 导入

HGhub → **插件中心 → 导入与订阅**，粘贴下面任一地址：

| 用途 | 地址 |
| --- | --- |
| 插件列表（推荐） | `https://raw.githubusercontent.com/hacksynth/huangguo-plugin/main/manifest.json` |
| 单个插件 | `https://raw.githubusercontent.com/hacksynth/huangguo-plugin/main/plugins/huangguo.js` |
| jsDelivr 镜像（GitHub Raw 不通时） | `https://cdn.jsdelivr.net/gh/hacksynth/huangguo-plugin@main/manifest.json` |

导入后确认状态为 **已就绪**，再从首页、分类或搜索进入详情点集播放。

## 能力

| 能力 | 内容 |
| --- | --- |
| `home` | 热播榜 banner、最近更新网格、各分类行，均带「查看更多」 |
| `feed` | 全站信息流，卡片带 `entryEpisodeId` 可直接连播 |
| `catalog` | 全部 / 4 个分类 / 排行榜（热播、推荐、潜力），排序：最近更新、当前最热、小蓝原创 |
| `search` | 关键词搜索与分页，`trending` 取站点热搜词 |
| `media` | 详情（分集、演职员、互动状态）、HLS 播放（正片 / HEVC / 预览）、`matchUrl` 识别 `/video/{id}/` 链接 |
| `articles` | 站点图文 / 评测栏目列表与正文 |
| `endpoints` | 主站单线路 |

## 实现要点

**封面是加密的。** 站点把图床协议写死在 `crypto-image-protocol.js` 里，AES-128-CBC、无填充，16 字节 ASCII key/iv。插件按 HPP 的 `ResourceRef.transform` 交给宿主原生解密，不额外占用插件运行时：

```js
{ url, transform: { type: 'aes-cbc', key, iv, padding: 'none' } }
```

key/iv 用 `hg.codec.base64Encode` 在运行时现算，图床按 `pic.` 前缀识别，域名再换也不会失效。

**播放接口的请求体必须是空 JSON。** `POST /api/videos/{id}/episodes/{ep}/playback`，body `{}`。带上 `id` / `ep` 会被上游判为「播放参数无效」。响应里的 `timestamp` 是签名过期时间（约 2 小时），写进 `expiresAt`，让宿主提前刷新。

**榜单条目字段和列表不同。** 列表用 `id`，榜单用 `video_id` + `metrics` + `rank`，不做兼容映射会得到一个空的热播榜且不报错。

**排序参数是 `hot_window`。** 「小蓝原创」= `sort=hot_window&is_original=1`（全站 1781 部中约 209 部），不是 `sort=hot`。

## 已知限制

- 分类「AI换脸」「AI魔改」上游已下线，接口返回空列表，前端表现为空分类，属正常。
- 只读公开接口，不含登录、评论、收藏等账号侧能力。
- 站点改版后相关方法会抛 `UPSTREAM_CHANGED`，提示更新插件。

## 变更

### 1.1.0

- 封面改走站点当前的 `pic.qnlycc.cn` + AES-CBC 协议（旧实现的图床白名单已过期）。
- 榜单条目兼容 `video_id` / `metrics` / `rank`，热播榜分区恢复。
- 排序修正为 `hot_window` 与 `is_original`。
- 补齐 `feed`、`articles`、`matchUrl`。
- 分页按 `pagination.pages` 收敛，末页 `next` 返回 `null`。

## 说明

第三方内容源的可用性由上游决定。使用前请自行判断所在地法律法规与站点条款。
