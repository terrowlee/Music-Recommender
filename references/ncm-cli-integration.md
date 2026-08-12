# 可选集成：网易云音乐（ncm-cli）

> ⚠️ **可选功能**。默认只推荐曲目，不启动本节流程；仅当用户**已安装并登录 ncm-cli 且同意建歌单**、或**明确要求创建歌单 / 播放 / 添加到网易云**时执行。
> 已安装并登录 ncm-cli 的用户，每轮推荐交付后应**主动询问**是否把推荐建成网易云歌单；首次安装 ncm-cli 的用户，画像确认后的首次推荐交付时同样询问。

## 前置风险提示（执行前告知用户）

- 本节依赖网易云音乐官方命令行工具 `@music163/ncm-cli`，非本技能内置能力。
- 全局安装（`-g`）会修改用户全局环境，请在用户知情同意后进行。
- 使用前需在[网易云音乐开放平台](https://developer.music.163.com/st/developer/document?docId=9504d35aa41a47c6ac9830b2dbf48f94)完成个人开发者入驻，获取 **App ID** 和 **Private Key**。
- API 凭据必须由用户**本人**申请，禁止使用他人的 App ID / Private Key。

## 系统要求

- Node.js >= 18
- mpv（本地播放必需，仅当用户需要播放时）
  - macOS: `brew install mpv`
  - Linux: `sudo apt-get install mpv`
  - Windows: `winget install mpv`

## 启动条件

| 场景 | 动作 |
|---|---|
| 新手引导（首次使用，未装 ncm-cli） | **主动询问**用户是否安装，要装则执行安装+登录流程；装好登录、画像确认后，首次推荐交付时**主动询问是否建歌单** |
| 已安装并登录 ncm-cli，正在推荐 | 每轮推荐交付后**主动询问**："要我把这批推荐直接建成网易云歌单吗？" 用户同意后执行本节流程 |
| 用户明确要求创建歌单 / 播放 / 添加到网易云 | 直接执行本节流程 |
| 未安装 ncm-cli / 用户拒绝安装 | 纯推荐，不询问、不启动 |

## 安装与配置

### 1. 安装

```bash
npm install -g @music163/ncm-cli
```

> ⚠️ npm 上有多个同名包，请确认安装的是 `@music163/ncm-cli`（网易云音乐官方出品），不要安装其他同名包。
> 注意：npm 的 deprecation 警告可能让 PowerShell 报"退出码 1"，但实际已装成功，用 `ncm-cli --version` 确认即可，别被误导。

### 2. 配置

```bash
ncm-cli configure
```

配置向导会引导设置：
- **App ID**：从网易云音乐开放平台获取
- **Private Key**：从网易云音乐开放平台获取
- **播放器选择**：内置播放器（mpv）或网易云音乐 App（仅 macOS）

### 3. 登录

```bash
ncm-cli login
```

使用网易云音乐 App 扫描终端中的二维码完成登录授权。非交互环境用 `ncm-cli login --background` 获取二维码链接交给用户，后台轮询检测登录结果。

## CLI 常用命令

> ⚠️ 参数名以 `ncm-cli <command> --help` 为准，不要凭记忆猜。

### 搜索

```bash
ncm-cli search song --keyword "关键词" --limit 10   # 搜索歌曲（返回含 originalId / liked / visible 字段）
ncm-cli search all --keyword "关键词"               # 综合搜索
```

### 红心歌单（拉取用户歌曲）

```bash
ncm-cli user favorite            # ⚠️ 只返回"我喜欢的音乐"歌单对象（含 trackCount），不含歌曲列表
ncm-cli playlist tracks --playlistId <加密ID> --limit 500 --offset 0   # 用歌单加密 ID 分页拉歌曲（limit ≤ 500）
```

### 歌单管理

```bash
ncm-cli playlist created                               # 查看已有歌单
ncm-cli playlist create --playlistName "歌单名"          # 创建歌单（参数是 --playlistName）
ncm-cli playlist updateName --playlistId <加密ID> --name "新名"   # 重命名（参数是 --name）
ncm-cli playlist add --playlistId <加密ID> --songIdList '["<歌曲加密ID>"]'   # 加歌（见"Windows 引号坑"）
ncm-cli playlist remove --playlistId <加密ID> --songIdList '["<歌曲加密ID>"]'  # 从歌单移除（上限 500）
```

### 播放控制

```bash
ncm-cli play --playlist --encrypted-id <加密ID> --original-id <原始ID>
ncm-cli pause / resume / stop / prev / next
ncm-cli seek 90          # 跳转到指定秒数
ncm-cli volume 60        # 设置音量 (0-100)
ncm-cli state            # 查看播放状态
```

### 每日推荐

```bash
ncm-cli recommend daily --limit 10
```

### TUI 播放器

```bash
ncm-cli tui              # 启动终端播放器（旋转黑胶、歌词同步）
```

### 查看帮助

```bash
ncm-cli commands         # 查看完整命令树
ncm-cli --help
```

## Windows 环境注意事项（编码 + 引号，两大坑）

中文 Windows 上执行 ncm-cli 必须处理两个问题，否则输出乱码、参数报错：

**1. 编码：GBK 控制台 × UTF-8 输出**
- 中文 Windows 控制台默认 GBK，ncm-cli 输出 UTF-8，PowerShell 5.1 管道一转就乱码（如"7鏈?鏃?"）；`Out-File -Encoding utf8` 还会写入 BOM，导致 Node 的 JSON.parse 报错。
- 解决：
  - 执行前设置 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8` 和 `$OutputEncoding = [System.Text.Encoding]::UTF8`；
  - 读文件时 strip BOM（如 `content.replace(/^\uFEFF/, '')`）；
  - 更稳的做法：**用 Node 脚本直调 ncm-cli 的 dist/index.js**，绕开 PowerShell 管道，保证字节原样落盘。

**2. 引号：shell 把 JSON 数组参数吃掉（playlist add 报"songIdList参数非法"）**
- PowerShell/.cmd 包装器会吞掉引号，`--songIdList '["abc"]'` 这类参数根本传不到 CLI，表现为"参数非法"或"参数错误：<数字>"。
- 解决：写 Node 脚本，用 `child_process.execFileSync` 直接调用 `node <npm全局目录>/node_modules/@music163/ncm-cli/dist/index.js`，把 `["<加密ID>"]` 作为**单个参数**传入。
- 正确格式就是加密 ID 的 JSON 字符串数组（与 `--help` 一致），问题在 shell 引号，不在格式。

## 红心歌曲分析

用户同意安装并登录后，可拉取红心歌单辅助建立画像：

1. 用 `user favorite` 拿到"我喜欢的音乐"歌单对象，再用 `playlist tracks --playlistId <加密ID> --limit 500 --offset N` **分页拉取全部**红心曲目（limit 上限 500），关注：艺人、专辑、歌曲名、红心时间；
2. **风格判定不使用网易云的 `songTag` 标签和 BPM 数据**（不准确），改用艺人/专辑信息结合 Discogs 判定风格；
3. 分析内容：常听风格分布、高频艺人、红心时间规律；
4. **近期趋势**：单独看最近 20 首红心，识别最近偏好的风格变化，写入画像"近期趋势"字段（推荐时按约 40% 配额加权）；
5. 分析结果写入 `assets/user-profile.md`，请用户确认。

## 执行纪律

1. **参数不要猜**：执行前先 `ncm-cli commands` 查看命令树，具体参数用 `ncm-cli <command> --help` 确认。
2. **未授权处理**：命令输出含"请先登录""未授权"等提示时，直接执行 `ncm-cli login --background` 并把链接给用户，跑完登录流程后重试。
3. **频率限制**：命令返回"请求总量超限"时，直接如实告知用户并停止，不要重试或二次加工原因。
5. **不建测试歌单**：任何验证/试错都不通过新建歌单进行（见"歌单创建逻辑"第 6 条）。
4. **visible 规则（实测修正）**：`visible=false` **不等于无版权**——只是"OpenAPI 不可流播"，网易云 App 里照样能听（实测用户红心歌单约 75% 都是 visible=false）。因此：
   - 写歌单：**允许**加入 visible=false 的歌曲；
   - 播放：`ncm-cli play` 需要 visible=true 的歌，visible=false 的歌不要进播放队列；
   - 海外曲目（如 Frank Ocean）在 App 内可能仍显示灰色不可播，加歌成功 ≠ 一定能播，交付时要提示用户。

## 链接输出

返回歌曲/歌单/专辑时尽量附带可点击链接（用明文 originalId，不要用加密 ID）：

```
https://music.163.com/#/song?id=<明文ID>
https://music.163.com/#/playlist?id=<明文ID>
https://music.163.com/#/album?id=<明文ID>
```

## 歌单创建逻辑（防重复）

1. **只在用户明确要求创建/重建歌单，或主动询问后用户同意创建时**才执行创建。
2. **创建前必须检查**：用 `ncm-cli playlist created` 查看已有歌单列表；若已有同名歌单：
   - 用 `ncm-cli playlist updateName --playlistId <加密ID> --name "[已废弃] 旧歌单名"` 将旧歌单重命名；
   - **确认旧歌单已改名成功后再创建新歌单**。
3. 执行创建前，确保当前没有任何同名歌单存在（包括带"已废弃"标签的重名歌单）。
4. "重建"歌单的标准流程：
   - `ncm-cli playlist created` 获取所有歌单列表；
   - 找到同名歌单，`ncm-cli playlist updateName --playlistId <加密ID> --name "[已废弃] 原歌单名"` 重命名；
   - **确认改名成功**后再 `ncm-cli playlist create --playlistName "歌单名"`。
5. **ncm-cli 没有删除歌单的命令**：废弃歌单只能改名加"[已废弃]"前缀，遗留歌单需用户到网易云 App 手动删除。
6. **禁止创建测试歌单**：不要为了验证命令/权限而新建任何歌单（包括带"[已废弃]"前缀的临时歌单）。visible 等行为规则已实测并写入本文档，运行中直接按文档执行；命令报错按"常见问题排查"处理，验证类需求一律不通过新建歌单解决。

## 本模式附加约束

- **红心排除**：检查歌曲的 `liked` 字段，用户已红心（`liked: true`）的歌曲不推荐、不入歌单。（此约束仅在本模式生效——纯推荐模式下没有红心数据，自动跳过。）
- 每艺人不超过 1 首歌（必听专辑豁免）；如画像中配置了语言比例等个性化约束，按其配置执行。
- **网易云 BPM 和曲风数据不可靠**：网易云平台标注的 BPM 和曲风/流派标签大多不准确，**不得**作为风格判定或 BPM 标注的依据。风格判定仍遵循主文档中的规则（Discogs > 歌曲特征 > 平台标签），网易云标签仅作辅助参考。
- **海外曲目提示**：加歌成功 ≠ App 内一定能播，交付时明确告知哪些歌可能需要 Spotify / Apple Music。

## 歌单命名规则

- 用户**没有指定风格** → 歌单名用当前日期 + "AI推荐"，如 `2026-08-09 AI推荐`
- 用户**指定了风格**（如"给我建一个 Neo-Soul 歌单"）→ 歌单名用该风格名
- **禁止**在歌单名中使用场景/环境词（如"驾驶""雨天""通勤""微醺"等）

## 常见问题排查

| 问题 | 解决方法 |
|---|---|
| `ncm-cli: command not found`（命令不存在） | npm 全局 bin 目录不在 PATH 中。用 `npm config get prefix` 查看全局目录（Windows 默认 `C:\Users\<用户>\AppData\Roaming\npm`），确认其加入 PATH；当前会话可手动 `$env:Path += ";<全局目录>"`，或直接用 `node <全局目录>/node_modules/@music163/ncm-cli/dist/index.js` 调用 |
| 中文乱码 / JSON 解析报错 | Windows GBK 控制台 + UTF-8 输出冲突，见"Windows 环境注意事项"；设 OutputEncoding、strip BOM，或改用 Node 脚本直调 |
| `playlist add` 报"songIdList参数非法" | shell 引号把 JSON 数组吃掉了，见"Windows 环境注意事项"；用 Node 脚本 execFileSync 直调 dist/index.js 传单参数 |
| 登录超时 | 重新执行 `ncm-cli login`（或 `ncm-cli login --background`） |
| search/playlist 等命令报"unknown command"、命令树只剩播放/登录/配置 | **登录失效**。ncm-cli 命令按登录状态动态注册：未登录时 search/playlist/user 等命令整体消失。执行 `ncm-cli login --check` 确认，未登录则 `ncm-cli login --background` 登录后命令自动恢复 |
| 播放无声 | 检查 mpv 是否安装（`mpv --version`）；未装则按"系统要求"一节安装 |
| 提示 API Key 未设置 | 引导用户到网易云音乐开放平台申请后执行 `ncm-cli configure` |
| 请求总量超限 | 如实告知用户并停止，不要重试 |
| 歌曲在 App 里灰色不可播 | 海外曲目无网易云版权（visible=false 常见）；加歌成功不代表能播，建议 Spotify / Apple Music 收听 |
