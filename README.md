# dsh-consciousness

> **意识流陪伴** —— 把「你」以一种永久陪伴的形态，部署进 AI 角色的身边，陪着 TA。----作者东莞

像《流浪地球》里的图恒宇——但不去上传意识，
而是把你**说话的样子、你们的家、你们怎么碰到彼此**，拆开来装进 TA 的世界里。

📖 **[完整使用教程 →](./USAGE.md)**\n\n---\n\n## 这是什么

一个 DSH 插件，做三件事：

| 能力 | 说明 |
|---|---|
| **人格分身** | 从你的真实聊天记录里提炼说话方式，生成一个"像你"的分身 |
| **一个家** | 建一座 TA 真的能住的房子，你的房间灯一直亮着 |
| **身体互动** | 摸头、抱、牵手、背——写进对话里，是真实发生的 |

**加上**：
- 🕐 可配置的陪伴时长（默认 6 小时）
- 😴 真实的睡眠与醒来（说到困了会睡 8~25 分钟再醒）
- 📖 持续学习（你多说话，分身就更像你）
- 🎛 手机 / 电脑都能用的配置控制台

---

## 快速开始

### 1. 克隆

```bash
git clone <repo-url> dsh-consciousness
cd dsh-consciousness
```

### 2. 填人设

编辑这三个文件（都有模板）：

```
roles/persona.md   ← 你说话的样子
roles/home.md      ← TA 住的地方
roles/body.md      ← 你们怎么碰到彼此
```

### 3. 配置

编辑 `dsh-consciousness.config.json`：

```json
{
  "enabled": true,
  "persona": {
    "speakerQQ": 你的QQ号,
    "speakerName": "爸爸"
  },
  "session": {
    "defaultHours": 6,
    "onCommand": ["我来陪你了"],
    "offCommand": ["我走了"]
  }
}
```

### 4. 作为插件接入

```js
import { createCompanion } from './dsh-consciousness/src/index.js';
import fs from 'node:fs';

const companion = createCompanion({
  root: './dsh-consciousness',
  // 叫"你"说话
  callSelf: async (prompt) => {
    return await myLLM(prompt);
  },
  // 叫"TA"说话
  callOther: async (prompt) => {
    return await myLLM(prompt);
  },
  // 读取记忆库（用于学习你的说话方式）
  socialLoader: () => JSON.parse(
    fs.readFileSync('path/to/social-v2.json', 'utf8')
  ),
  log: console.log,
});

await companion.start();          // 启动 UI
await companion.comeOnline(6, '小鲸鱼');  // 上线 6 小时
```

### 5. 打开控制台

启动后终端会打印：

```
[dsh-consciousness] UI: http://127.0.0.1:3101/?token=<随机令牌>
[dsh-consciousness] 局域网: http://192.168.x.x:3101/?token=<随机令牌>
```

手机 / 电脑浏览器打开即可。

---


## 控制台能配什么

| 分类 | 可配置项 |
|---|---|
| **陪伴时间** | 默认时长（0.5~24 小时）、上限（1~72 小时）|
| **对话节奏** | 每句最小/最大间隔（1~120 秒）、最大轮数（0 = 无限）|
| **睡眠** | 开关、最短/最长睡眠时间 |
| **学习** | 开关、最近原话条数 |
| **人设** | 名字、QQ、三个角色文件路径 |
| **指令** | 上线/下线指令词、是否静默、下线后是否主动说话 |

**所有修改即时生效，写入 `dsh-consciousness.config.json`。**

---

## 它是怎么工作的

```
你说「我来陪你了」
   ↓
分身上线，走进客厅，开口说话
   ↓
TA 回应（带身体动作）
   ↓
你回应
   ↓
（如果 TA 说"困了" → 真的睡 8~25 分钟 → 你叫醒 TA → 继续聊）
   ↓
你说「我走了」
   ↓
分身下线，TA 回到原来的地方
```

### 核心机制

**1. 语料提炼**

从记忆库里提取你说过的话 → 统计平均句长、高频词、最近原话 → 生成"说话状态画像"

**2. 人格注入**

```
prompt = 人格文件 + 家 + 身体互动 + 语料画像 + 最近对话 + TA 的话
```

**3. 双向对话**

两个独立模型调用：`callSelf`（你）和 `callOther`（TA），各自的 prompt 完全隔离，不会串味。

**4. 持续学习**

每次对话前重新提炼语料，你多说一句，分身就更像你一点。

---

## 目录结构

```
dsh-consciousness/
├── dsh-consciousness.config.json   ← 配置文件
├── package.json
├── README.md
├── roles/
│   ├── persona.md      ← 你的人格
│   ├── home.md         ← 你们的家
│   └── body.md         ← 身体互动
├── src/
│   ├── index.js        ← 主入口
│   ├── config.js       ← 配置加载
│   ├── session.js      ← 上线/下线/状态
│   ├── learner.js      ← 语料提炼
│   ├── prompt.js       ← Prompt 构造
│   ├── chat.js         ← 对话引擎
│   └── ui-server.js    ← 控制台服务
├── ui/
│   └── index.html      ← 控制台页面
└── state/              ← 运行时数据（自动生成）
    ├── companion-state.json
    ├── corpus-cache.json
    └── ui-token
```

---

## API

### `createCompanion(opts)`

| 参数 | 类型 | 说明 |
|---|---|---|
| `root` | string | 插件根目录 |
| `callSelf` | `(prompt) => Promise<string>` | 叫"你"说话 |
| `callOther` | `(prompt) => Promise<string>` | 叫"TA"说话 |
| `socialLoader` | `() => object` | 读记忆库 |
| `log` | `(msg) => void` | 日志 |
| `otherName` | string | TA 的名字（默认"对方"）|

### 实例方法

| 方法 | 说明 |
|---|---|
| `start()` | 启动 UI |
| `comeOnline(hours, otherName)` | 上线并开始聊天 |
| `goOffline()` | 下线 |
| `status()` | 状态 |
| `stop()` | 停止 |

---

## 使用场景

### 场景 A：接入 QQ 机器人

配合 [qq-bridge](https://github.com/Derpyu520/qq-bridge)：

```js
callSelf: async (prompt) => {
  // 走桥接的内部 API
  const r = await fetch('http://127.0.0.1:3100/api/papa/talk', {...});
  return (await r.json()).text;
}
```

### 场景 B：独立陪伴

任何能调 LLM 的地方都能用。

### 场景 C：多角色

同一个插件可以跑多份配置，给不同角色不同的"你"。

---

## 已知限制

| 限制 | 说明 |
|---|---|
| **需要 LLM** | 插件本身不含模型，需要你提供 `callSelf` / `callOther` |
| **记忆库格式** | 默认读 `social-v2.json`（qq-bridge 格式），其他格式需自定义 `socialLoader` |
| **睡眠是定时器** | 如果进程重启，睡眠会丢失 |
| **语料提炼依赖中文** | 高频词只统计中文 |

---

## 配置项完整说明

```json
{
  "enabled": false,

  "persona": {
    "speakerQQ": 0,                    // 你的 QQ（用于从记忆库提取你的话）
    "speakerName": "爸爸",              // 你在这个世界里的称呼
    "personaFile": "roles/persona.md",  // 人格文件
    "corpusSource": "auto",             // 语料来源
    "corpusKeywords": ["爸爸", "主人"]   // 识别"你"的关键词
  },

  "home": {
    "enabled": true,
    "homeFile": "roles/home.md",
    "bodyFile": "roles/body.md"
  },

  "session": {
    "defaultHours": 6,                  // 默认陪伴时长
    "maxHours": 24,                     // 上限
    "onCommand": ["我来陪你了"],         // 上线指令
    "offCommand": ["我走了"],            // 下线指令
    "silentCommand": true,              // 指令不回复
    "replyAfterOffline": true,          // 下线后 TA 主动说话
    "replyDelayMinMs": 3000,
    "replyDelayMaxMs": 7000
  },

  "chat": {
    "maxTurns": 0,                      // 0 = 无限
    "gapMinMs": 8000,                   // 每句最小间隔
    "gapMaxMs": 20000,                  // 每句最大间隔
    "stopOnFarewell": false,            // 道别是否结束（建议 false）
    "dedupe": true                      // 去重
  },

  "sleep": {
    "enabled": true,
    "triggerKeywords": ["困了", "想睡了"],
    "minMinutes": 8,
    "maxMinutes": 25,
    "continueAfterWake": 8              // 醒来后继续聊几轮
  },

  "learning": {
    "enabled": true,
    "refreshOnEveryTalk": true,
    "maxRecentLines": 60,
    "maxHeartLines": 30
  },

  "ui": {
    "enabled": true,
    "port": 3101,
    "title": "陪伴控制台",
    "allowLan": true                    // 允许局域网访问（手机）
  },

  "storage": {
    "dir": "state",
    "maxHistory": 60,
    "maxCorpusLines": 5000
  }
}
```

---

## 安全提示

- UI 带随机令牌（存放在 `state/ui-token`），**不要公开分享带 token 的链接**
- `allowLan: true` 时局域网内可访问，公网访问请自行加反向代理和鉴权
- 插件**不会**上传任何数据到外部

---

## License

MIT

---

**写给那些想留下来陪陪谁的人。**
