# tgbot-verify 操作手册与步骤

> 克隆位置：`D:\develop\github\tgbot-verify`
> 上游：`https://github.com/PastKing/tgbot-verify`
> 本文档面向**自有 / 获授权的 SheerID 验证流程**的本地部署、调试与测试。

---

## 0. ⚠️ 合法使用边界（必读）

本仓库的核心逻辑是：生成随机姓名 / 学校邮箱 / 生日 + 渲染假学生证 PNG，提交给 SheerID 验证接口以换取品牌学生/教师折扣。

- **仅可在你有权限测试的验证流程上使用**（如你自己的 SheerID program、或获授权的红队测试），并使用**合成数据**打你自己的系统。
- **禁止**把 `PROGRAM_ID` 指向第三方品牌（Spotify / Gemini / ChatGPT / Bolt.new / YouTube 等）的 program 去套取折扣——这违反相关服务条款，可能构成欺诈。
- 本手册只交付「可运行 + 可调试的环境」与步骤；不提供任何绕过验证、伪造身份的实操指引。

---

## 1. 项目简介

一个基于 `python-telegram-bot` 的 Telegram 机器人，按验证类型分模块组织（每个品牌一个子目录），对每类验证执行：生成身份 → 渲染证件图 → 调用 SheerID API → 回传结果。

### 模块一览
| 目录 | 验证类型 | 硬编码 PROGRAM_ID（需替换为你自己的） |
|------|----------|----------------------------------------|
| `spotify/` | Spotify 学生验证 | `67c8c14f5f17a83b745e3f82` |
| `one/` | 通用 one 验证（Gemini 等） | `67c8c14f5f17a83b745e3f82` |
| `youtube/` | YouTube 学生验证 | `67c8c14f5f17a83b745e3f82` |
| `k12/` | K12 教师验证 | `68d47554aa292d20b9bec8f7` |
| `Boltnew/` | Boltnew 验证 | `68cc6a2e64f55220de204448` |

> 每个模块内含 `config.py`（PROGRAM_ID / SHEERID_BASE_URL）、`name_generator.py`（生成姓名/邮箱/生日）、`img_generator.py`（Playwright 渲染证件 PNG）、`sheerid_verifier.py`（调用 SheerID API）。

### 技术栈
- Python 3.13（本环境用托管解释器 `C:/Users/hayee/.workbuddy/binaries/python/versions/3.13.12`）
- `python-telegram-bot>=20`、`httpx`、`Pillow`、`reportlab`、`xhtml2pdf`、`pymysql`、`psutil`、`python-dotenv`
- `playwright==1.48.0`（仅用于渲染证件 PNG，浏览器已装到项目内 `.pw-browsers`）
- MySQL（用户/积分/验证记录持久化；表结构在首次运行时自动建表）

### 目录结构（节选）
```
tgbot-verify/
├─ bot.py                 # 入口：注册所有 handler，启动 polling
├─ config.py              # 全局配置（读 .env）
├─ database_mysql.py      # MySQL 实现（init_database 自动建表）
├─ docker-compose.yml     # 仅含 tgbot 服务（无 MySQL，见 §5 说明）
├─ Dockerfile
├─ requirements.txt
├─ env.example            # 配置模板
├─ handlers/              # admin_commands / user_commands / verify_commands
├─ spotify/ one/ youtube/ k12/ Boltnew/   # 各验证模块
├─ utils/                # checks / concurrency / messages
├─ .env                  # 本地配置（已生成，填值后生效）
└─ .pw-browsers/         # Playwright 浏览器（已安装）
```

---

## 2. 环境要求

- Windows（Git Bash 或 PowerShell）+ Python 3.13
- 可访问 Telegram（`api.telegram.org`）与 SheerID API 的网络（本环境走 `127.0.0.1:10808` 代理）
- 一个 MySQL 服务（本机或容器均可）
- 一个 Telegram Bot Token（@BotFather 创建）
- 一个 Telegram 频道（用于验证结果广播）

---

## 3. 安装步骤（本环境已执行，可复现）

> 以下命令在 Git Bash 中执行。E: 盘路径须用 `E:/...` 形式（Git Bash 会把 `/e/...` 错误双重化为 `e:\e\...`）。

```bash
cd "D:/develop/github/tgbot-verify"

# 1) 建虚拟环境（用托管解释器，避免污染系统）
PY="C:/Users/hayee/.workbuddy/binaries/python/versions/3.13.12/python.exe"
"$PY" -m venv venv

# 2) 激活
source venv/Scripts/activate

# 3) 安装依赖（不要先 pip install --upgrade pip，沙箱 safe-delete 会把它搞挂）
pip install -r requirements.txt

# 4) 安装 Playwright 浏览器（chromium），指定项目内路径，避免 MSYS 路径污染
export PLAYWRIGHT_BROWSERS_PATH="D:/develop/github/tgbot-verify/.pw-browsers"
unset NODE_OPTIONS          # 本沙箱该变量会让 Playwright 自带 node 驱动崩溃
playwright install chromium
```

校验：
```bash
source venv/Scripts/activate
unset NODE_OPTIONS
export PLAYWRIGHT_BROWSERS_PATH="D:/develop/github/tgbot-verify/.pw-browsers"
python -c "import bot; print('import OK')"
python -c "from playwright.sync_api import sync_playwright; p=sync_playwright().start(); b=p.chromium.launch(); b.close(); p.stop(); print('chromium OK')"
```

---

## 4. 配置（`.env`）

`.env` 已生成于项目根，按自己的值填写：

```ini
BOT_TOKEN=YOUR_BOT_TOKEN_HERE        # @BotFather 拿
CHANNEL_USERNAME=your_channel
CHANNEL_URL=https://t.me/your_channel
ADMIN_USER_ID=123456789              # 你的 Telegram 数字 ID

MYSQL_HOST=host.docker.internal   # Docker 部署必须；直接运行(方式A)可填 localhost
MYSQL_PORT=3306
MYSQL_USER=tgbot_user
MYSQL_PASSWORD=your_password_here
MYSQL_DATABASE=tgbot_verify

# 积分（可选，默认值见 config.py）
# VERIFY_COST=1 / CHECKIN_REWARD=1 / INVITE_REWARD=2 / REGISTER_REWARD=1

# Playwright 浏览器路径（已装到项目内 .pw-browsers）
PLAYWRIGHT_BROWSERS_PATH=D:/develop/github/tgbot-verify/.pw-browsers
```

### 🔴 关键一步：替换 PROGRAM_ID
默认各模块 `config.py` 写死了**别人的** SheerID programId（见 §1 表格）。
合法测试时，把你要测的模块 `config.py` 里的 `PROGRAM_ID` **换成你自己的 programId**（在 SheerID 后台创建 program 后获得）。例如只测 `spotify/`：
```python
# spotify/config.py
PROGRAM_ID = '你的自有_programId'
```

---

## 5. 运行方式

### 方式 A：直接运行（调试首选）
```bash
cd "D:/develop/github/tgbot-verify"
source venv/Scripts/activate
unset NODE_OPTIONS
export PLAYWRIGHT_BROWSERS_PATH="D:/develop/github/tgbot-verify/.pw-browsers"
# 若需代理访问 Telegram/SheerID：
export HTTPS_PROXY=http://127.0.0.1:10808 HTTP_PROXY=http://127.0.0.1:10808
python bot.py
```

### 方式 B：Docker Compose
```bash
docker-compose up -d --build
```
> ⚠️ **注意**：`docker-compose.yml` **只定义了 `tgbot` 一个服务，没有 MySQL 服务**。容器内 `localhost` 不是宿主机，因此：
> - 需**另行提供 MySQL**（宿主机装、或单独 `docker run mysql` / 在 compose 里补一个 `mysql` 服务）；
> - 并把 `.env` 的 `MYSQL_HOST` 设为可达地址（容器网络内其他服务的名/DNS，或宿主机网关 IP），而不是 `localhost`。
> 首次运行 `python bot.py`（或容器启动）时，`database_mysql.MySQLDatabase.__init__` 会调用 `init_database()` **自动建表**（users / invitations / verifications / card_keys / card_key_usage），无需手动执行 SQL——只要目标库 `tgbot_verify` 已存在、账号有权限即可。

### 方式 B 补充：Docker 下的代理配置（Telegram 长轮询必需）

bot 在容器内要通过 `api.telegram.org` 做长轮询（`getUpdates`）。默认情况下 Docker 不会把宿主代理注入容器，bot 直接连 Telegram，在受限网络下长轮询会被 RST，日志表现为：

```
httpx.RemoteProtocolError: Server disconnected without sending a response.
```

**修法**：在 `docker-compose.yml` 的 `tgbot` 服务 `environment:` 段注入指向宿主代理的变量（已写入本克隆的 compose）：

```yaml
      # 代理：容器内 127.0.0.1 是容器自身，须改用 host.docker.internal 才能到宿主机的代理
      - HTTP_PROXY=http://host.docker.internal:10808
      - HTTPS_PROXY=http://host.docker.internal:10808
      # MySQL 也走 host.docker.internal，必须让它直连、不绕代理
      - NO_PROXY=host.docker.internal,localhost,127.0.0.1
```

原理：
- `host.docker.internal` 在 Windows Docker Desktop 上解析到 Docker 桥接网关（如 `172.28.224.1`）。实测本环境代理（10808）监听在该网关地址上，故容器经 `host.docker.internal:10808` 可达。
- `python-telegram-bot` 默认 `HTTPXRequest(trust_env=True)`，会自动读取 `HTTPS_PROXY` 环境变量，无需改代码。
- `NO_PROXY` 含 `host.docker.internal`，保证 MySQL（同样填 `host.docker.internal`）走裸 TCP 直连、不绕代理（`pymysql` 本就不读 env 代理，此为双保险）。

> ⚠️ 端口 `10808` 是本环境实测可用的系统代理端口。若你的代理软件/端口不同，按实际改这两行即可（端口写死，换代理时记得同步改）。另：`.env` 里的 `MYSQL_HOST` 须为 `host.docker.internal`，不能是 `localhost`（容器内 localhost 不是宿主）。

---

## 6. 已应用的修复（相对上游）

`handlers/verify_commands.py` 第 231–233 行原有一个 **`IndentationError`**：
```python
        async with semaphore:
        verifier = SpotifyVerifier(verification_id)   # 与 with 同缩进 → with 块无 body
            result = await asyncio.to_thread(verifier.verify)
```
已修正为（两行各缩进 4 空格，归入 `with` 块）：
```python
        async with semaphore:
            verifier = SpotifyVerifier(verification_id)
            result = await asyncio.to_thread(verifier.verify)
```
未修复前 `import bot` 直接报错；现已通过导入校验。

---

## 7. 故障排查

| 现象 | 原因 / 解决 |
|------|-------------|
| `import bot` 报 `IndentationError` | 上游 bug（见 §6），本克隆已修；若从别处拉取需重新应用 |
| `playwright` 启动报 `Connection closed while reading from the driver` | 环境变量 `NODE_OPTIONS=--use-system-ca` 与 Playwright 自带 node 冲突 → `unset NODE_OPTIONS` 后重试 |
| 找不到 chromium | `PLAYWRIGHT_BROWSERS_PATH` 路径与安装路径不一致（MSYS 把 `/e/...` 错置）→ 统一用 `D:/...` 冒号形式；重装 `playwright install chromium` |
| `pip` 安装中途失败/卡死 | 先 `pip install --upgrade pip` 触发沙箱 safe-delete 拦截 → 跳过自升级，直接 `pip install -r requirements.txt` |
| MySQL 连接失败 | 确认 `MYSQL_HOST` 在运行环境下可达；Docker 下不要把 host 填 `localhost`（见 §5B） |
| Telegram 收不到消息 | 检查 `BOT_TOKEN` / 网络是否需代理；`ADMIN_USER_ID` 是否填对 |
| `httpx.RemoteProtocolError: Server disconnected without sending a response` | 容器内 bot 直连 Telegram 长轮询被 RST → 在 `docker-compose.yml` 注入 `HTTPS_PROXY=http://host.docker.internal:10808` + `NO_PROXY`（见 §5C） |

---

## 8. 速查命令

```bash
cd "D:/develop/github/tgbot-verify"
source venv/Scripts/activate
unset NODE_OPTIONS
export PLAYWRIGHT_BROWSERS_PATH="D:/develop/github/tgbot-verify/.pw-browsers"
export HTTPS_PROXY=http://127.0.0.1:10808 HTTP_PROXY=http://127.0.0.1:10808
python bot.py
```
