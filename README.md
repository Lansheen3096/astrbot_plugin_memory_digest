# astrbot_plugin_memory_digest

**长期记忆（写入侧）**：把对话提炼成 bot 的第一人称记忆，写进 AstrBot 知识库。

版本 `0.2.4` · AstrBot `>= 4.28.2` · 需要 [astrbot_plugin_memory_recall](https://github.com/Lansheen3096/astrbot_plugin_memory_recall) 配合

---

## 三层记忆

一条平铺的记忆列表在长会话里只有两种结局：要么全塞进上下文（噪声淹掉信号），
要么全靠相似度取 top-N（「他一直住在哪」这种长期事实，会被当天一句无关的话挤掉）。

这里的切法不是按相似度，是按**这条记忆加不加日期才成立**：

```
state        界定双方「是什么关系」的：称呼 / 关系定位 / 陪伴承诺
             —— 长期且不依赖某天。每轮全量注入，不过闸门。
preference   长期成立的习惯、喜好、准则、环境事实 ——「总是这样」
event        具体某天发生的事，或有明确时限的事实 ——「那次这样」
             —— 这两层走检索 + 闸门，event 还要按时间递减。
```

判据交给模型自己判（`kind` 字段），但**代码做硬校验**：`kind` 不在枚举里就丢掉
（不兜底成 `event` —— 模型判不出来说明它根本没读懂这条，留着是噪声）。

## 流水线

```
读 conversations 表 → 渲染成带时间锚的文本 → 交给 LLM 按 DIGEST_SYSTEM 提炼
  → 逐行解析 JSONL（时间锚 / kind / confidence 三项硬校验）
  → bge-m3 余弦去重（对库里已有的 + 对本批内部的，阈值 0.85）
  → 来源校验（闸门①）→ 写进知识库
```

## 几个关键设计

**① 进度按「会话 → 已处理消息数」记，只处理新增部分。**
状态在 `data/plugin_data/astrbot_plugin_memory_digest/state.json`；旧版写在
`data/memory_digest_state.json`，第一次加载会自动迁移，不重扫历史。

**② 独立 session 调模型**，不污染主对话的上下文。

**③ 时间锚只能照抄，不能推算。**
每条记忆前面带 `[YYYY-MM-DD HH:MM]`，来自上游写在 user 消息里的 `<system_reminder>`。
模型只负责**从给定的锚里原样抄一个**，抄了不在本批对话里的时间就整条丢掉（`bad_anchor`）。
模型很爱自己编日期，这一条拦掉的是最常见的一类假记忆。

> 时间锚是可选依赖：把「收到消息那一刻」写进 user 消息的插件（例如
> [pacer](https://github.com/Lansheen3096/astrbot_plugin_pacer) 的提醒逻辑）能提供它。
> 没有也能跑，只是条目不代日期，`event` 层的时间衰减也无从算起。

**④ 来源校验（闸门①）—— 拦「模型编的记忆」。**
写库之后，拿条目正文去和「它声称来源那一轮用户说过的话」算句余弦；低于阈值就打
`metadata.suspect = 1`，注入侧一律不放行。

阈值是实测校准的：

```
真来源轮     0.573 ~ 0.856
合成的编造句  0.543 ~ 0.629
```

两条分布**有重叠**，所以这一闸只拦「明显对不上」的。语言上确实出现在那一轮、
但其实是模型自己的叙述被当成事实那类，它拦不住 —— 那类靠 `status="new"` 标记待人眼复核
（注入时降权 0.5）。

**⑤ 硬过滤词表**（`BLOCK_WORDS`）。提示词里写「不许记账号、报错、配置项」，模型会选择性忽略；
代码过滤不会，命中即不写。

**⑥ 提炼提示词刻意不写角色身份。** `DIGEST_SYSTEM` 只描述任务（提炼哪三类、`kind` 怎么判、
输出什么格式），不提任何具体角色的设定与来历 —— 那部分属于你的部署，用你自己的 system prompt 管。
换一个 bot 人设不用改这个插件。

**⑦ 人称一律用字段，不用代词。** 两个插件在文件顶部声明**同一对**常量，取值必须一致：

```
USER_REF = "[用户]"    用户
SELF_REF = "我"        助手自己
```

对话记录的说话人标签、写进库里的记忆正文、检索时归一后的查询 —— 三处同一套字面量。
没有性别，也没有「你 / 他 / 她到底指谁」的歧义，模型不用做人称翻译。

⚠️ **迁库注意**：换人称字段等于换了一套字面量。库里已有条目如果是「他 / 我」这类写法，
改了字段就检索不回来 —— 要么保持原写法，要么清库重跑一次提炼。

## 指令

```
/记住        手动跑一次提炼
/记住状态    看进度（处理到哪了、库里有多少条）
```

另有每天 `02:00` 的自动整理（`DAILY_HOUR` / `DAILY_MINUTE`，按容器时区）。

## Embedding 配置

写入侧提供两个入口，改的是同一组配置：

- **标准插件配置页**：插件页 →「长期记忆（写入侧）」→ 配置，字段来自 `_conf_schema.json`。
  保存后 AstrBot 热重载插件并自动应用。
- **插件可视化视图**：「Embedding 控制台」，可测连接、看当前 provider / 维度，再保存。
  需要 AstrBot >= 4.29；4.28.2 上配置页可用，视图不显示。

配置页或控制台保存后，插件在 `on_astrbot_loaded` 里：

1. 探测端点拿真实向量维度（配置维度留 0 时）；
2. 创建或原地更新固定 ID 的 embedding provider（`memory_layer_embedding`）；
3. 把 `kb_name` 指向的知识库绑过去，重新初始化向量库；
4. 把探测到的维度写回配置，下次加载直接复用。

已经一致就跳过，不会每次启动都重建向量库。

| 配置项 | 默认 | 说明 |
|---|---|---|
| `kb_name` | `长期记忆` | 必须和注入侧同名配置一致 |
| `embedding_mode` | `kb` | `kb` / `cloud` / `local` |
| `cloud_api_base` | 空 | 云端 OpenAI 兼容 Base |
| `cloud_api_key` | 空 | 云端 Key，`secret` 遮罩 |
| `cloud_model` | `text-embedding-3-small` | 云端模型 |
| `cloud_dimensions` | `0` | 0 = 自动探测 |
| `local_host` | `127.0.0.1` | 本机服务地址 |
| `local_port` | `11434` | 本机服务端口 |
| `local_api` | `ollama` | `ollama` / `openai` |
| `local_model` | `bge-m3` | 本机模型 |
| `local_dimensions` | `0` | 0 = 自动探测 |
| `embedding_timeout` | `60` | 请求超时（秒） |
| `create_kb_if_missing` | `true` | 找不到知识库时自动创建 |
| `force_rebind` | `false` | 允许同维度换模型 |

### 安全闸门（重要）

知识库里的旧向量和新 embedding 必须在同一个向量空间，否则不会报错、只会**静默检索错乱**。
所以写绑定之前会检查：

- **维度不一致 → 直接拒绝。** 例：现有索引 1024 维，新服务返回 768 维。请新建知识库，或清空旧条目后再切。
- **模型名不一致 → 默认拒绝**，需要打开「允许同维度换模型」（控制台里是「强制切换」）。
  不同模型即使都是 1024 维，向量空间也不一样。
- **知识库不存在** → 由 `create_kb_if_missing` 控制；有可用 rerank provider 会一并挂上，
  否则提示你补一个（注入侧没有 rerank 时只注入状态层）。

API Key 标了 `secret: true`，配置页遮罩显示；控制台只返回 `api_key_set` 和末四位掩码，不回显明文。
留空表示沿用已保存的 Key。

### 容器部署的地址坑

`127.0.0.1` 在 AstrBot 容器里指的是**容器自己**，不是宿主机。本机 embedding 服务跑在宿主机上时，
填 `host.docker.internal`；Docker 不支持这个名字就用 `ip route` 查网桥地址
（Linux 上就是 Docker 默认网桥的网关地址，或你自己创建的网络网关）。

### 权限边界

配置页和控制台都走 AstrBot Dashboard 的插件鉴权，只有已登录的管理员能用。插件按设计可以向
填写的任意地址发请求（包括内网 / 本机端口），所以 **6185 面板不要直接暴露公网**；
要远程管理就走 Tailscale / 反代加认证。

## 安装

```bash
# AstrBot 插件市场搜索「长期记忆（写入侧）」，或：
cd <AstrBot>/data/plugins
git clone --depth 1 --branch v0.2.2 https://github.com/Lansheen3096/astrbot_plugin_memory_digest
git clone --depth 1 --branch v0.2.0 https://github.com/Lansheen3096/astrbot_plugin_memory_recall
# docker 部署：拷进宿主机挂载的数据目录后重启容器
```

**两个插件必须一起装**，两个配置页里的 `kb_name` 必须一致。

复核台（清 `status="new"` 和 `suspect=1` 的队列、删错条目）不住在插件里，
在聚合仓 [astrbot-memory-layer](https://github.com/Lansheen3096/astrbot-memory-layer) 的 `tools/`。

## License

MIT
