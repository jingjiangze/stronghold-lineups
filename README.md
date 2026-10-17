# stronghold-lineups — AI 布阵学习库（公开只读镜像）

> 这里**只有数据，没有代码**。每个模式一个文件，只保留**最新一份**；历史档案在 R2 的按日期键里。
> 游戏服务器写回 CDN / R2；这个仓库是给读取端用的镜像。

## 这个仓库是什么

服务器在每局**正常结算**（`victory` / `defeat`）之后，把真人座位的最终布阵记进本地库。本地库积累到一定量后，
服务器通过 CDN 的数据上传通道写回 R2（键带日期，只增不改）。发布轮再把每个模式最新的一份复制成稳定别名，
本仓库的定时任务把稳定别名同步到这里。

```
游戏服务器 ──写回──▶ CDN 数据通道 /api/lineup/* ──▶ R2 packs/lineup-library/<模式>-<日期>.jsonl
                                                        │
                                              发布轮 CopyObject 稳定别名
                                                        ▼
                              packs/lineup-library/lineups_<模式>.jsonl
                                                        │
                                          本仓库定时任务（每 6 小时）
                                                        ▼
                                    data 分支 lineups/lineups_<模式>.jsonl
```

- **本仓库只保留最新**：`data` 分支每次都是一个孤儿单提交（force-push），所以仓库体积恒定为「9 个文件 × ≤3 MiB」。
- **历史不在这里**：R2 的 `<模式>-<日期>.jsonl` 是不可变档案，需要旧版本请去 R2 取。

## 文件布局

`data` 分支：

```
lineups/lineups_mode_single_normal.jsonl
lineups/lineups_mode_single_hard.jsonl
lineups/lineups_mode_single_abyss.jsonl
lineups/lineups_mode_multi_normal.jsonl
lineups/lineups_mode_multi_hard.jsonl
lineups/lineups_mode_multi_abyss.jsonl
（以及 mode_single_funny / mode_multi_funny / mode_training_1，若该模式有数据）
```

每条记录一行 JSON，字段：`t`（恒为 `lineup`）、`at`、`modeId`、`stageId`、`difficulty`、`round`、`sig`、
`cells`。`cells` 里每个元素是 `{ id, golden, row, col, dir }`。

**不含任何玩家身份**：没有玩家名、账号、session id，只有棋子 id 与坐标。

## 读取：部署你自己的服务器

只用一个变量：`SP_LINEUP_MODE`。

| 值 | 含义 | 读 | 写回 |
|---|---|---|---|
| `public` | 只读公共镜像（R2 优先、GitHub 兜底） | ✅ 远程拉取 | ❌ 不写回 |
| `local` | 自建存储（默认 `/app/lineups`，可挂持久卷） | ✅ 本地 | ✅ 可写回 CDN |
| 不设 | 功能关闭（默认） | ❌ | ❌ |

```bash
# 公共托管（只读）：不占本地磁盘，容器重建不丢数据
docker run -d --name stronghold -p 3000:3000 \
  -e SP_LINEUP_MODE=public \
  stronghold-protocol

# 自行存储：自己攒库、并写回 CDN
docker run -d --name stronghold -p 3000:3000 \
  -e SP_LINEUP_MODE=local \
  -v "$PWD/lineups:/app/lineups" \
  -e SP_LINEUP_PUBLISH=1 -e CDN_ADMIN_KEY="<上传口令>" \
  stronghold-protocol
```

优先级：**显式设置 `SP_LINEUP_LIBRARY` 时，`SP_LINEUP_MODE` 被忽略**。需要更细控制时直接用它：

```bash
# 目录（每模式一个文件，可挂卷）
SP_LINEUP_LIBRARY=/app/lineups
# 单个文件（旧布局，不分模式）
SP_LINEUP_LIBRARY=/app/lineups/lineups.jsonl
# 远端只读：逗号分隔、按序回退，| 后面是本地缓存目录
SP_LINEUP_LIBRARY=remote:<url1>,<url2>|/tmp/lineups
```

### 公共镜像的两个地址

```
R2（首选）   https://speedcdn.jiangjiangze.icu/packs/lineup-library
GitHub（兜底）https://ghfast.top/https://raw.githubusercontent.com/jingjiangze/stronghold-lineups/data/lineups
```

⚠️ **不要直连 `raw.githubusercontent.com`**：大陆三网实测只有 3/5 可用（DNS 污染），必须走 `ghfast.top` 包装。
更细的读法：把两个地址逗号拼进 `remote:`，第一个可用即停。

## 上传规则（只能传这个，不能传别的）

数据通道照后台素材上传的模式另起一条，**只收布阵学习库**：

```
POST /api/lineup/begin   { key, size, sha256, note }
PUT  <putUrl>            原始字节（预签名直传 R2，15 分钟有效）
POST /api/lineup/commit  { id, stagingKey, key, sha256, size, what }
```

硬规则（服务端三步各拦一次，不是只靠界面）：

1. **来源固定**：通道把 `source` 钉死为 `lineup-library`，调用方传什么都不算数。
2. **键只能是** `packs/lineup-library/<modeId>-<YYYY-MM-DD>.jsonl`；`modeId` 只允许 `[A-Za-z0-9_]`。
3. **扩展名只能是 `.jsonl`**；`.json` / `.txt` / `.zip` / 任何可执行内容位一律 400。
4. **只增不改**：同一天同一模式重复发布会收到 409 —— 当天的第一份就是答案。
5. **单份 ≤ 3 MiB**；空文件、`.partial` 中间文件不发布。
6. **异常数据在服务器侧就被丢掉**：未成功结算的对局（`error` / `abandoned` / `eliminated`）、
   单干员/空盘、越界坐标、非法朝向、一小时内同签名 40 次以上的刷分，都不会进库。

不要用 `/admin` 页面上传学习库，也不要往这里放代码、素材、密钥或别的数据。

## 更新节奏

- 服务器：启动时发布一次 + 每 24 小时一次（`SP_LINEUP_PUBLISH=0` 可关）。
- 发布轮：把暂存字节核对 sha256 后复制到对外键，再刷新稳定别名。
- 本仓库：`sync-lineups.yml` 每 6 小时拉一次稳定别名；也可手动 dispatch。
- 读取端：默认每 6 小时同步一次（`SP_LINEUP_SYNC_MS` 可改），带 ETag / 304，拉到新字节后无需重启。

任何一步失败都只记日志：缺了学习库，AI 只是回到默认布阵，对局照常进行。
