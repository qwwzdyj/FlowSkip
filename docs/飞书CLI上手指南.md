# 飞书 CLI 本地上手指南

> 让你在 10 分钟内跑通 [`larksuite/cli`](https://github.com/larksuite/cli)，从零到第一次成功调用飞书 API。
> 适用：macOS / Linux；Windows 用户建议在 WSL 里跑。
> 参考：[官方 README（中文）](https://github.com/larksuite/cli/blob/main/README.zh.md)

---

## 一、为什么用它

飞书官方维护的命令行工具，覆盖 17 大业务域、200+ 命令、24 个 AI Agent Skills。三个使用场景：

- **人类用户**：终端里查日程、发消息、读文档，比浏览器快
- **AI Agent**：Claude Code / Cursor 等自动加载 Skills，无需自己写飞书 SDK 集成
- **脚本/自动化**：`lark-cli api ...` 直接打任意飞书开放平台接口（2500+）

如果你只是要让 AI Agent 操作飞书 —— 装这个就够了。

---

## 二、前置环境

| 工具 | 版本 | 必需性 |
|---|---|---|
| Node.js（带 `npm`/`npx`） | 任意 LTS（≥18 即可） | **必需** |
| Go | ≥ 1.23 | 仅源码安装需要 |
| Python 3 | 任意 | 仅源码安装需要 |

检查：

```bash
node -v && npm -v
```

没装 Node.js 的话：
- macOS：`brew install node`
- 其他：去 https://nodejs.org 装 LTS

---

## 三、安装

**推荐方式：npm 全局安装**

```bash
# 1. 安装 CLI 本体
npm install -g @larksuite/cli

# 2. 安装 Agent Skills（AI 工具自动识别用，强烈建议装）
npx skills add larksuite/cli -y -g
```

第二条命令会拉取 24 个 skills 到 `~/.agents/skills/lark-*`，并为已检测到的 AI 工具（Claude Code、Cursor、Codex 等）创建符号链接。装完不需要重启终端。

> **如果不想用 npm**：克隆仓库 `make install`，需要 Go 1.23+ 和 Python 3。

验证安装：

```bash
lark-cli --version
# 应输出：lark-cli version x.x.xx
```

---

## 四、配置应用凭证（一次性）

飞书 CLI 需要绑定一个**飞书开放平台应用**才能调 API。CLI 会引导你创建。

```bash
lark-cli config init --new
```

这条命令会：
1. 在终端打印一个授权链接 + 二维码
2. **阻塞**，等你在浏览器完成应用创建/绑定
3. 浏览器完成后命令自动退出，凭证存入 OS 原生 keychain

链接长这样：

```
打开以下链接配置应用:
  https://open.feishu.cn/page/cli?user_code=XXXX-XXXX&...
等待配置应用...
```

浏览器里：登录飞书 → 按引导**新建一个应用**（名字随便起，比如 "我的-cli"）→ 跟随提示完成绑定。

完成后终端会输出：

```
OK: 应用配置成功! App ID: cli_xxxxxxxxxxxxxxxx
```

> **凭证存哪了？** macOS Keychain / Linux Secret Service。不会写明文文件，也不要手动去 `~/.lark-cli/` 翻 `app_secret`。

---

## 五、用户登录授权

凭证只是"应用身份"，还需要"用户授权"才能以你的身份操作飞书资源（你的日历、你的消息……）。

```bash
lark-cli auth login --recommend
```

`--recommend` 表示**自动勾选官方推荐的常用 scope 集合**（覆盖日历、消息、文档、多维表格、表格、任务、邮件等所有主流业务域）。如果你只想授权部分能力：

```bash
# 只授权日历 + 任务
lark-cli auth login --domain calendar,task

# 完全自定义 scope
lark-cli auth login --scope "calendar:calendar:read,im:message"
```

这条命令同样**阻塞**并打印一个授权 URL：

```
在浏览器中打开以下链接进行认证:
  https://accounts.feishu.cn/oauth/v1/device/verify?flow_id=...&user_code=XXXX-XXXX
等待用户授权...
```

浏览器里登录飞书 → 在权限确认页点"同意" → 终端自动收到 token：

```
OK: 授权成功! 用户: 你的名字 (ou_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx)
```

---

## 六、验证

```bash
# 看认证状态（应输出 tokenStatus: valid）
lark-cli auth status

# 烟测：列今天的日程（即使为空也算通过）
lark-cli calendar +agenda
```

返回 `{"ok": true, ...}` 就成了。

---

## 七、常见问题

### 1. 本机有代理（Clash / Surge / V2Ray）

CLI 会警告：

```
[WARN] proxy detected: HTTPS_PROXY=http://127.0.0.1:xxxx
       requests (including credentials) will transit through this proxy.
```

走自己本机的代理一般无害。介意的话单次跑：

```bash
LARK_CLI_NO_PROXY=1 lark-cli auth login --recommend
```

永久生效在 `~/.zshrc` 或 `~/.bashrc` 里 `export LARK_CLI_NO_PROXY=1`。

### 2. Token 多久过期

- access token：1 小时，CLI 自动用 refresh token 续期
- refresh token：7 天，过期后再跑一次 `lark-cli auth login --recommend` 即可

### 3. 调某条命令报"权限不足 / scope missing"

说明 `--recommend` 没覆盖到，补授权：

```bash
# 先看缺什么 scope
lark-cli auth check <scope-name>

# 补授权
lark-cli auth login --scope "<scope-name>"
```

`auth login` 是**累加**的，不会清掉旧 scope。

### 4. 想换账号 / 重置一切

```bash
lark-cli auth logout         # 删凭证，保留应用配置
# 或彻底重来：
rm -rf ~/.lark-cli           # 删配置文件
# 同时去 keychain 里找 "lark-cli" 项目删掉
```

### 5. 命令找不到（`command not found: lark-cli`）

npm 全局 bin 目录不在 `PATH` 里。查一下：

```bash
npm bin -g
```

把输出路径加到 `PATH`。Homebrew 安装的 Node 一般自动包含。

---

## 八、三层命令速查

CLI 提供三种调用粒度，**优先级从高到低**：

```bash
# 1. 快捷命令（+ 前缀）：人/AI 友好，封装常用场景
lark-cli calendar +agenda
lark-cli im +messages-send --chat-id oc_xxx --text "Hello"

# 2. API 命令：从 OAPI 元数据生成，与平台端点一一对应
lark-cli calendar calendars list
lark-cli im messages create --params '...' --data '...'

# 3. 通用调用：任意飞书开放平台 endpoint，2500+ 全覆盖
lark-cli api GET /open-apis/calendar/v4/calendars
```

实操建议：**优先 `+shortcut` → 找不到再用 `service subcmd` → 都没有再用 `api`**。

---

## 九、AI Agent 集成提示

如果你用 Claude Code / Cursor / Codex 等 AI 工具：

- 第三步装的 24 个 Skills 已经自动对这些工具可见
- 直接在对话里说"帮我看下今天日程"、"在 X 群里发条消息" —— Agent 会自己调 lark-cli
- 不需要再写 MCP server 或配 SDK

查看已装的 skills：

```bash
ls ~/.agents/skills/ | grep lark
# lark-shared, lark-calendar, lark-im, lark-doc, lark-base, ...
```

---

## 十、一些有用的命令片段

```bash
# 看应用支持的所有 scope
lark-cli auth scopes

# 看某个 API 的参数 schema
lark-cli schema calendar.events.instance_view

# 任何写操作都先 dry-run 看一眼请求
lark-cli im +messages-send --chat-id oc_xxx --text "hi" --dry-run

# 输出格式
lark-cli calendar +agenda --format table   # 表格
lark-cli calendar +agenda --format ndjson  # 管道友好
lark-cli calendar +agenda --jq '.data[].summary'  # jq 过滤

# 自动翻页
lark-cli im messages list --params '{"chat_id":"oc_xxx"}' --page-all
```

---

## 安全提醒（来自官方 README）

- 这个 CLI 可被 AI Agent 调用，存在模型幻觉、提示词注入等固有风险
- 授权后，AI 会以**你的用户身份**执行操作，可能导致敏感数据泄露
- **建议**：把这个 CLI 绑定的飞书应用当做**私人对话助手**用，不要拉进群、不要让其他用户接触
- 不要主动放开默认安全配置（输入防注入、输出净化等）

---

## 参考链接

- 官方仓库：https://github.com/larksuite/cli
- 中文 README：https://github.com/larksuite/cli/blob/main/README.zh.md
- 飞书开放平台：https://open.feishu.cn
- Skills 安全评估：https://skills.sh/larksuite/cli
