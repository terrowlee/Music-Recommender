# 可选集成：网易云音乐（ncm-cli）

> ⚠️ **可选功能**。仅当用户明确要求**创建歌单 / 播放 / 添加到网易云**时执行。
> 用户只要推荐曲目时，禁止启动本节流程。

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

| 用户意图 | 动作 |
|---|---|
| 新手引导（首次使用，未装 ncm-cli） | **主动询问**用户是否安装，要装则执行安装+登录流程 |
| 只推荐曲目 | 不启动 ncm-cli，按标准输出规范推荐 |
| 创建歌单 / 播放 / 添加到网易云 | 启动安装和登录流程 |

## 安装与配置

### 1. 安装

```bash
npm install -g @music163/ncm-cli
```

> ⚠️ npm 上有多个同名包，请确认安装的是 `@music163/ncm-cli`（网易云音乐官方出品），不要安装其他同名包。

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

使用网易云音乐 App 扫描终端中的二维码完成登录授权。

## CLI 常用命令

### 搜索

```bash
ncm-cli search all --keyword "关键词"          # 综合搜索
ncm-cli search song --keyword "关键词" --limit 10  # 搜索歌曲
```

### 播放控制

```bash
ncm-cli play --playlist --encrypted-id <加密ID> --original-id <原始ID>
ncm-cli pause / resume / stop
ncm-cli prev / next
ncm-cli seek 90          # 跳转到指定秒数
ncm-cli volume 60        # 设置音量 (0-100)
ncm-cli state            # 查看播放状态
```

### 每日推荐

```bash
ncm-cli recommend daily --limit 10
```

### 歌单管理

```bash
ncm-cli playlist created          # 查看已有歌单
ncm-cli playlist create           # 创建歌单
ncm-cli playlist updateName       # 重命名歌单
```

### TUI 播放器

```bash
ncm-cli tui              # 启动终端播放器（旋转黑胶、歌词同步）
```

### 查看帮助

```bash
ncm-cli --help
```

## 红心歌曲分析

用户同意安装并登录后，可拉取红心歌单辅助建立画像：

1. 拉取用户红心曲目（最多 200 首），关注：艺人、专辑、歌曲名、红心时间；
2. **风格判定不使用网易云的 `songTag` 标签和 BPM 数据**（不准确），改用艺人/专辑信息结合 Discogs 判定风格；
3. 分析内容：常听风格分布、高频艺人、红心时间规律；
4. **近期趋势**：单独看最近 20 首红心，识别最近偏好的风格变化，写入画像"近期趋势"字段（推荐时按约 40% 配额加权）；
5. 分析结果写入 `assets/user-profile.md`，请用户确认。

## 执行纪律

1. **参数不要猜**：执行前先 `ncm-cli commands` 查看命令树，具体参数用 `ncm-cli <command> --help` 确认。
2. **未授权处理**：命令输出含"请先登录""未授权"等提示时，直接执行 `ncm-cli login --background` 并把链接给用户，跑完登录流程后重试。
3. **频率限制**：命令返回"请求总量超限"时，直接如实告知用户并停止，不要重试或二次加工原因。
4. **visible 检查**：搜索结果的 `visible` 为 false 的歌曲无法播放、不要加入播放队列，也不要写入歌单。

## 链接输出

返回歌曲/歌单/专辑时尽量附带可点击链接（用明文 originalId，不要用加密 ID）：

```
https://music.163.com/#/song?id=<明文ID>
https://music.163.com/#/playlist?id=<明文ID>
https://music.163.com/#/album?id=<明文ID>
```

## 歌单创建逻辑（防重复）

1. **只在用户明确要求创建/重建歌单时**才执行创建。
2. **创建前必须检查**：用 `ncm-cli playlist created` 查看已有歌单列表；若已有同名歌单：
   - 用 `ncm-cli playlist updateName` 将旧歌单重命名为 `[已废弃] 旧歌单名`；
   - **确认旧歌单已改名成功后再创建新歌单**。
3. 执行创建前，确保当前没有任何同名歌单存在（包括带"已废弃"标签的重名歌单）。
4. "重建"歌单的标准流程：
   - `ncm-cli playlist created` 获取所有歌单列表；
   - 找到同名歌单，`ncm-cli playlist updateName` 重命名为 `[已废弃] 原歌单名`；
   - **确认改名成功**后再 `ncm-cli playlist create`。

## 本模式附加约束

- **红心排除**：检查歌曲的 `liked` 字段，用户已红心（`liked: true`）的歌曲不推荐、不入歌单。（此约束仅在本模式生效——纯推荐模式下没有红心数据，自动跳过。）
- 每艺人不超过 1 首歌（必听专辑豁免）；如画像中配置了语言比例等个性化约束，按其配置执行。
- **网易云 BPM 和曲风数据不可靠**：网易云平台标注的 BPM 和曲风/流派标签大多不准确，**不得**作为风格判定或 BPM 标注的依据。风格判定仍遵循主文档中的规则（Discogs > 歌曲特征 > 平台标签），网易云标签仅作辅助参考。

## 歌单命名规则

- 用户**没有指定风格** → 歌单名用当前日期 + "AI推荐"，如 `2026-08-09 AI推荐`
- 用户**指定了风格**（如"给我建一个 Neo-Soul 歌单"）→ 歌单名用该风格名
- **禁止**在歌单名中使用场景/环境词（如"驾驶""雨天""通勤""微醺"等）

## 常见问题排查

| 问题 | 解决方法 |
|---|---|
| `ncm-cli: command not found`（命令不存在） | npm 全局 bin 目录不在 PATH 中。用 `npm bin -g`（或 `npm config get prefix`）查看全局目录，确认其加入 PATH；Windows 上重新打开终端再试 |
| 登录超时 | 重新执行 `ncm-cli login`（或 `ncm-cli login --background`） |
| 播放无声 | 检查 mpv 是否安装（`mpv --version`）；未装则按"系统要求"一节安装 |
| 提示 API Key 未设置 | 引导用户到网易云音乐开放平台申请后执行 `ncm-cli configure` |
| 请求总量超限 | 如实告知用户并停止，不要重试 |
