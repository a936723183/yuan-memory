# 使用说明（详细版）

> 面向"已经解压、想真正用起来"的人。命令都在包根目录执行，`$env:...` 是 PowerShell 写法，bash 用 `export`。

---

## 0. 先决定你要哪个

| 你的情况 | 用哪个 |
|---|---|
| 只是想让 AI 记得住，单项目、单机 | **渊**（`bin/`） |
| 有一个"角色/伴侣"，要分区、要隐私、要语义 | **记忆图书馆**（`library/`） |
| 用 DeepSeek Harness 桌面版 | **插件**（`plugin/`），设置页里点 |
| 用了 QQ 机器人（dsh-qqbot / surfcore） | `integrations/` |

三者可以叠加：渊管日常，图书馆管角色，插件把两者接到 DSH 设置页。

---

## 1. 渊：索引与召回

### 1.1 第一次

```powershell
$env:DSH_HOME = "$env:USERPROFILE\.dsh"   # 指向"包含 memories\ 的那一层"
python bin\yuan.py index
python bin\yuan.py stats
```

`stats` 会打印：条目数、分轨、日期范围、索引大小。**条目为 0 = `DSH_HOME` 指错了**。

### 1.2 命令一览

| 命令 | 说明 | 常用参数 |
|---|---|---|
| `index` | 增量重建索引 | `--force` 忽略 mtime 全量重扫 |
| `recall "问句"` | 召回 | `--top 6 --chars 1200`；`--track key,user`；`--no-log`；`--no-refresh` |
| `stats` | 概况 | — |
| `maintain` | 重算热度/状态 | `--report` 顺便写日报到 `reports/` |
| `bench` | 跑内置 20 题回归集 | — |

### 1.3 什么时候该用

- **"上次那个方案叫什么"** → `recall "上次那个方案"`，一般 top 3、600 字就够，别一次取 10 条。
- **"我之前定过什么规矩"** → `recall "约定 规则" --track key,user`，只查长期轨，噪音最小。
- **"这周都干了啥"** → `recall "这周" --top 8`，或看 `reports/` 下的日报。
- **改过召回逻辑之后** → `bench`，命中率掉点就回滚。

### 1.4 索引在哪、删了会怎样

- 库在包根目录 `yuan.db`；**删掉不会丢记忆**，下次 `recall` 自动重建（800 条约 1 秒）。
- 每次 `recall` 前会比对源文件 mtime+size，变了就静默增量重建，所以没有"读到旧索引"的问题，也不需要 cron。
- `pins.txt` 一行一个词/片段：命中的条目不衰减（长期约定、账号、生日这类）。

### 1.5 换机器 / 换目录

- 记忆源：把 `~/.dsh/memories` 整个搬过去，`DSH_HOME` 指到它；
- 索引库：**不要搬**，让它重建（跨机器搬 SQLite 可能遇上版本/编码差异，收益为零）；
- 路径搬家后 `YUAN_HOME`（库位置）默认跟着包走，无需改。

---

## 2. 记忆卡召回（`bin\qqcard.py`）

给"角色型 AI"用的：一张 JSON 记忆卡，按当前这句话捞 1~2 条。

### 2.1 卡长什么样

```json
{"records":[
  {"id":"demo-1","kind":"owner","title":"和主人：下次开刷之前先报一声",
   "text":"下次开刷之前先报一声，不偷偷玩","botReply":"行，下次开刷之前先跟你报一声。",
   "importance":74,"clarity":"vivid","createdAt":"2025-03-01T10:00:00Z","sensitive":false}
]}
```

关键字段：`text`（正文）、`importance`（0~100）、`clarity`（vivid/clear/fuzzy/trace/forgotten）、
`kind`（promise/firstMeet/emotion/owner/event）、`sensitive`（true 则不注入/加密）。

### 2.2 用法

```powershell
python bin\qqcard.py --card samples\memory-card.sample.json stats
python bin\qqcard.py --card samples\memory-card.sample.json recall "你还记得说好开刷的事吗" --top 3 --chars 300
python bin\qqcard.py --card ... recall "..." --json     # 给程序用
```

- `--card` 写在子命令前后都行（包里已兼容两种顺序）。
- 找不到卡时自动用 `samples/memory-card.sample.json`，方便先试。
- 输出里的 `~` = 模糊兜底、`≈` = 靠同义词才找到；说的时候该降确定性。

### 2.3 同义词

`bin\synonyms.json` 是一行一组的口语近义（结婚/领证、答应/承诺/约定…）。
**改完立即生效**（脚本每次调用都重读），不用重启。

---

## 3. 记忆图书馆

### 3.1 初始化

```powershell
python library\memlib.py init          # 建库（默认 data\library）
python library\memlib.py stats-export  # 可选：导热度快照
```

库位置由 `MEM_ROOT` 控制；不设时默认包内 `data\library`。

### 3.2 写入与检索

```powershell
python library\memlib.py update "记一条：周三下午要交稿" --region dsh_zone --type normal
python library\memlib.py update "身份证号……" --region qq_zone --sensitive
python library\memlib.py search "交稿时间" --region dsh_zone --top 5
python library\memlib.py get <id> [--reveal]     # 敏感条目要 --reveal 才解密
```

- `--type`：`config`（长期设定）/`preference`/`normal`/`draft`；
- 同内容重复写入会被内容哈希去重；
- `--sensitive` 的正文用 Fernet 加密，密钥在 `<MEM_ROOT>\secrets\content.key`。

### 3.3 维护

| 命令 | 作用 | 注意 |
|---|---|---|
| `check --region X` | 对账：md/db 计数、哈希、坏行 | 敏感条目无法校验（不解密） |
| `rebuild --region X` | 从 jsonl 完整重建 | 期间禁止写入（`E_REBUILDING`） |
| `purge <id>` | 删库并记墓碑 | 不重写 jsonl |
| `manifest` | 原子写 manifest（代+sha256） | 备份以它为准 |
| `drain` | 消费写入队列，失败进死信 | 队列卡住时跑 |

### 3.4 备份与迁移（重要）

`secrets\content.key` 与 `dedup.salt` **不参与普通备份**，必须单独存两处（U 盘 + 密码管理器）。
恢复顺序：**先 secrets，再 jsonl，最后 rebuild**。顺序错了，敏感条目永久解不开。

### 3.5 向量（可选）

两种来源，二选一：

| 来源 | 条件 | 说明 |
|---|---|---|
| 硅基流动（云端） | `SILICONFLOW_API_KEY` 或 `MEM_EMBEDDING_TOKEN` | 按量计费，本地不用下模型 |
| 本地 Ollama | 装 bge-m3 并让 `vec.py` 连上 | 不花钱，要下权重 |

限量用法（推荐）：

```powershell
$env:VECTORS_ENABLED = '1'
python library\memlib.py vec on --minutes 10
python library\memlib.py embed --region qq_zone --limit 60
python library\memlib.py vec off
```

闸门：窗口 10~30 分钟自动关、每批 `--limit`、业务查询 30 次/分、连续 3 次失败熔断 5 分钟。
没配 key 时 `embed` 会直接拒绝，不会偷偷扣钱。

---

## 4. 把记忆卡同步进图书馆

```powershell
python library\card_import.py --dry-run      # 只看
python library\card_import.py                # 真写（幂等）
python library\card_import.py --since-days 30
```

映射规则：

| 卡里的字段 | 图书馆里 |
|---|---|
| `sensitive=true` | 加密写入（`is_sensitive`） |
| `pinned`/`anchor`/`firstMeet`/`promise`/`importance≥70` | `mem_type=config` |
| `kind=owner` 且 `importance≥60` | `preference` |
| 其余 | `normal` |

- **只读卡**，一个字都不写回去；
- 幂等：同文本重复导入只会 `已存在`；
- 卡里删掉的记录不会自动从图书馆删（要删用 `memlib.py purge <id>`），这是刻意的：派生层不反向指挥真相。

---

## 5. 每日日记

```powershell
python library\card_diary.py --show                 # 今天写过没
python library\card_diary.py --text "今天……"        # 写一条（自动加【日记】前缀）
python library\card_diary.py --text "……" --dry-run   # 只看
```

- 走 `memory-inbox.jsonl` 通道（不直接改卡，避免和宿主抢写）；
- 一天一条，重复会被拒绝（`--force` 可强写）；
- 宿主下一次整理（≤10 分钟）会把它并成角色的一条 self 记忆，之后照常衰减。

想让 AI 每天自己写：在 DSH 里建一个 23:50 的定时任务，prompt 写"读今天的记忆 → 写 2~4 句第一人称日记 → 调 card_diary.py"。

---

## 6. DSH 插件（可选）

### 6.1 安装

```powershell
python plugin\_install.py
```

它做四件事：备份 profile 的 `package.json` / `cordis.patch.yml`、加 `link:` 依赖、把包名加进 `dsh.profile.bundles`、在 `node_modules` 建 junction。
**装完重启 DSH**，设置里出现「记忆图书馆」。

### 6.2 插件给什么

| 工具 | 用途 |
|---|---|
| `library_status` | key 是否已设、向量状态、待同步条数 |
| `library_search` | 在区域里检索 |
| `library_sync` | 卡 → qq_zone 同步 |
| `library_embed` | 限量向量回填（默认 60 条/批） |
| `library_config_page` | 设置页地址 |

设置页里可以填**硅基流动 Key**：只存本机 `plugin\config.json`（尝试 ACL 限当前用户），
页面/接口只回显后 4 位，日志不记 key。填完点「向量回填一批」就能逐批向量化。

### 6.3 卸载

```powershell
# 覆盖回备份 + 删 junction
Copy-Item "<profile>\package.json.bak-memlib-*" "<profile>\package.json" -Force
Copy-Item "<profile>\cordis.patch.yml.bak-memlib-*" "<profile>\cordis.patch.yml" -Force
cmd /c rmdir "<profile>\node_modules\dsh-memory-library"
```

记忆卡与图书馆数据都不受影响。

---

## 7. 排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `no such module: fts5` | Python 没编 FTS5 | 换官方发行版 |
| 召回全空 | 路径指错 / 没建索引 | `stats` 看条目数；`index --force` |
| 中文乱码 | 控制台代码页 | `set PYTHONIOENCODING=utf-8` 或 `chcp 65001` |
| 退出码 3 | 名字锁触发 | `python bin\name_lock.py unlock --root .`，目录改回含「渊」或 `yuan` 开头 |
| `E_REBUILDING` | 正在 rebuild | 等 30 秒重试 |
| `E_BUSY` | 写入队列满 | 跑 `drain` |
| `向量未开启` | `VECTORS_ENABLED` 未设或没 key | 见 3.5 |
| 设置页打不开 | 插件未加载 | 重启 DSH；或直接开 `http://127.0.0.1:18766` |

---

## 8. 一次典型的一周

```powershell
# 周一：新项目，建个 KEY.md，写三条约定
python bin\yuan.py index
python bin\yuan.py recall "约定" --track key

# 周三：想起上周的决定
python bin\yuan.py recall "上次说的那个方案" --top 3 --chars 600

# 周五：同步一次，给角色补向量
python library\card_import.py
python library\memlib.py vec on --minutes 10
python library\memlib.py embed --region qq_zone --limit 60
python library\memlib.py vec off

# 每天 23:50：日记（定时任务或手动）
python library\card_diary.py --text "今天……"
```

---

## 9. 名字锁补充

- 锁的是**包身份**：`manifest.json` 的名称/签名 + 目录名；
- 触发后：`state/name-lock.json` 记原因、索引被删、退出码 3；**记忆源不动**；
- 允许的目录名：含「渊」，或以 `yuan` 开头（`yuan-memory`、`渊-1.1.0` 都行）；
- `unlock` 只修清单，不改目录名——目录名要你自己改回。
