# Salvy 出图执行器 · AstrBot 插件

把文生图 API 接进 QQ。装上、填地址和 Key，就能在聊天里让它出图。

- 对话里说要图 → 它判断该不该画 → 画好发回
- 出图带频率闸，保护显卡不被打崩
- 默认只有管理员能用，群和 QQ 需要手动加白

## 安装

### 方式一：面板安装

下载 `astrbot_plugin_salvy_image.zip`，在 AstrBot 面板 → 插件 → 安装插件里选中压缩包。

### 方式二：手动放目录

解压到 AstrBot 的数据目录下：

```
<AstrBot 数据目录>/plugins/astrbot_plugin_salvy_image/
```

Windows 便携版的默认数据目录是 `C:\Users\<你>\.astrbot\data\`，所以完整路径是：

```
C:\Users\<你>\.astrbot\data\plugins\astrbot_plugin_salvy_image\
```

放好后重启 AstrBot。

## 配置

面板 → 插件 → Salvy 出图执行器 → 配置。

必填两项：

| 项 | 说明 | 示例 |
|---|---|---|
| `api_base` | 出图服务的根地址，末尾不带斜杠 | `http://127.0.0.1:8199` |
| `api_key` | 出图服务的 Key，请求头 `X-API-Key` | `your-key-here` |

其余有默认值，一般不用动。想调的话：

| 项 | 默认 | 作用 |
|---|---|---|
| `size` | `512x768` | 出图尺寸，显存紧就用这个 |
| `rate_min_interval` | 15 | 两次请求最小间隔（秒） |
| `rate_burst` | 2 | 连发几张后强制休息 |
| `rate_rest` | 120 | 休息时长（秒） |
| `rate_hourly` | 20 | 每小时上限 |
| `vram_min_gb` | 1.5 | 出图前要求的剩余显存，服务端不返回就跳过 |
| `admin_qqs` | 空 | 管理员 QQ 号 |
| `allowed_groups` | 空 | 允许使用的群号 |
| `allowed_users` | 空 | 允许使用的 QQ 号（私聊） |
| `silent_when_denied` | true | 无权限时静默 |
| `template_file` | `normal.json` | 模板文件名 |

改了配置不用重启插件，下一次出图会读新值。

## 指令

| 指令 | 谁能用 | 作用 |
|---|---|---|
| `/salvy 画 <描述>` | 有权限的人 | 立刻出图 |
| `/salvy 状态` | 有权限的人 | 看当前配置、权限来源、频率余量 |
| `/salvy 名单` | 有权限的人 | 看白名单 |
| `/salvy 允许群 <群号>` | 管理员 | 加群白名单 |
| `/salvy 允许QQ <QQ号>` | 管理员 | 加用户白名单 |

例：

```
/salvy 画 白衬衫，坐在窗边，午后阳光
```

## 它会自己判断该不该画

三条路，从便宜到贵：

1. **你说要图** —— 句子里有「画」「来一张」「自拍」这类词，直接画
2. **模型觉得该配图** —— 它在回复里输出 `[SALVY_IMG]描述[/SALVY_IMG]`，插件把这个标记换成真图
3. **都没触发** —— 纯聊天，不画

**默认不画**。没有标记、没有要图的话，就只是聊天。宁可少画一张，也不误耗一次显卡。

同一会话 60 秒内最多让模型判定一次，防止有人刷屏把你显卡烧了。

## 提示词净化

任何描述在送出前都会过一遍过滤，把**人物身份规格**剔掉：

```
loli / petite / small stature / child / 萝莉 / 幼女 / 娇小 /
身材 / 体型 / 身高 / 年龄 / 发育 / 三围 / 148cm / 45kg ...
```

出图提示词只描述「穿什么、在哪、什么表情、什么光」，不描述人物身体规格。命中的词记在 `plugin_data/salvy_image/sanitize.log`，方便你回查。

## 换自己的角色

模板是纯数据，改 `templates/normal.json` 就行：

```json
{
  "_style_positive": "质量词与画风",
  "_style_negative": "负向词（务必保留 nude/naked/nsfw 那一串）",
  "outfit_pool": ["白衬衫, beige cardigan"],
  "scene_pool": ["sitting by window, looking at viewer"],
  "mood_pool": ["gentle smile"],
  "camera_pool": ["upper body, portrait"],
  "env_budget_max": 4
}
```

几个踩过的坑，写在模板的 `api_rules` 里：

- `extra` 里不要再写 `1girl` —— 服务端人设已经有了，重复会画出第二个人
- 不要写眼瞳颜色 —— 和人设冲突，会出异瞳
- 环境词最多 4 个 —— 多了会抢人物焦点
- **负向必须保留 `nude, naked, nipples, topless, nsfw`** —— 不写服装词又不过滤，出来就是裸的

## 出图服务要满足的协议

插件按这套接口工作（大多数自建 SD/ComfyUI 网关都是这个形状）：

```
GET  /health                     → {"ok": true, "comfy": "up", "vram_free": 3.2}
POST /generate                   → {"urls": ["/images/<dir>/<file>.png"], "seed": 123}
     body: {"scene": "...", "extra": "...", "negative": "...",
            "size": "512x768", "count": 1, "negative_mode": "append"}
GET  /images/<dir>/<file>.png    → 图片二进制
```

两个容易踩的点：

- **`/images` 也要带 `X-API-Key`**。很多实现只在 `/generate` 上校验，结果图片下载 401。
- **`vram_free` 是可选字段**。服务端不返回，插件就跳过显存检查，不会因此拒绝出图。

## 常见问题

**装上了但提示「还没配置」**
去面板填 `api_base` 和 `api_key`，填完保存即可，不用重启。

**出图返回「出图后端没起来」**
`api_base` 指向的服务挂了。插件会读 `/health` 分辨是后端掉线还是显存不足，按提示处理。

**出图返回「显存不够」**
重启你的出图服务（ComfyUI 之类），或者把 `size` 调小。

**一直说「还在画上一张」**
`rate_min_interval` 默认 15 秒。第一次装完会误以为卡住，等十几秒就好。想放宽就调小这个值。

**图片画出来了但没发出去**
检查 AstrBot 的消息平台适配器能不能发图片消息。

**它一直不主动画图**
这是设计。默认不出图，只有你明确要图、或者模型自己打了标记才画。可以在模板的 `_route_hint` 里写一句鼓励它多画的话。

## 目录

```
astrbot_plugin_salvy_image/
├─ metadata.yaml          插件元信息
├─ _conf_schema.json      面板配置
├─ main.py                入口、指令族、钩子
├─ config.py              配置读取与降级
├─ route.py               分流判定
├─ sanitize.py            提示词净化
├─ rate_limit.py          频率闸
├─ api.py                 出图服务客户端
├─ guard.py               权限矩阵
├─ render.py              消息构造
├─ templates_loader.py    模板加载与拼装
└─ templates/normal.json  模板
```

运行时数据写到 `plugin_data/salvy_image/`，不在插件目录里，更新插件不会丢。

## 开发

```bash
# 自测，不需要 AstrBot 运行
py -3.12 scripts/selftest.py

# 打分发包（会把 api_key 默认值清空）
py -3.12 scripts/pack.py
```

## 许可

MIT
