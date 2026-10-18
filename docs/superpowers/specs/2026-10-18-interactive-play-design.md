# 互动乐园模块设计

日期：2026-10-18
状态：已与用户确认

## 背景

现有 `index.html` 是面向 3-6 岁幼儿的纯前端英语启蒙单页应用，已有字母学习、单词学习、趣味练习、音标乐园四个模块。本次新增「互动乐园」，用三种更适合 4-5 岁孩子的互动方式补充现有模块：听指令做动作（TPR）、亲子句型小对话、听音找物（I Spy）。

项目约定保持不变：单文件 HTML（内联 CSS/JS）、零构建、零网络依赖、使用 Web Speech API 发音。

## 目标与非目标

**目标**

- 首页新增第 5 个入口「互动乐园」
- 一个页面内用三个标签切换「听指令 / 说句子 / 找一找」
- 听指令：24 条日常指令，每轮随机 8 条，家长或孩子点击推进
- 说句子：6 组亲子问答，每组 3 个替换词，支持单独朗读和 A→B 连读
- 找一找：12 个带颜色属性的物品，每轮 6 张图卡、5 轮，听题目点图卡
- 复用现有语音、导航、卡片和反馈样式，保持单文件架构

**非目标**

- 不做麦克风录音、跟读评分或语音识别
- 不把三个模块加入现有选择题练习
- 不做学习进度持久化
- 不新增外部音频、图片、字体或第三方库
- 不改动字母、单词、音标、练习四个模块的既有行为

## 页面结构

首页 `home-cards` 新增第 5 张卡片：

- 标签：互动乐园
- 描述：听指令、说句子、找一找
- 图标：🎮
- 配色：`linear-gradient(135deg, var(--c2), var(--c3))`（橙色→黄色，未占用组合）

新增页面：

```html
<div id="playPage" class="page">
  <div class="category-tabs" id="playTabs"></div>
  <div id="playContent"></div>
</div>
```

三个标签复用现有 `.category-tabs` / `.cat-tab` 样式：

- `tpr`：🎧 听指令
- `dialogue`：💬 说句子
- `ispy`：🔍 找一找

默认进入「听指令」。切换标签时停止当前模块的语音和定时器，并重新渲染 `#playContent`。

## 模块一：听指令（TPR）

### 数据

```js
const TPR_DATA = [
  { en: 'Touch your nose', cn: '摸摸鼻子', emoji: '👃' },
  // 共 24 条
];
```

24 条指令分三类，每类 8 条：

**身体部位**

| 英文 | 中文 | emoji |
|---|---|---|
| Touch your nose | 摸摸鼻子 | 👃 |
| Touch your ears | 摸摸耳朵 | 👂 |
| Touch your head | 摸摸头 | 🙆 |
| Touch your tummy | 摸摸肚子 | 🧒 |
| Touch your feet | 摸摸脚 | 🦶 |
| Clap your hands | 拍拍手 | 👏 |
| Wave your hands | 挥挥手 | 👋 |
| Blink your eyes | 眨眨眼 | 👀 |

**动作**

| 英文 | 中文 | emoji |
|---|---|---|
| Stand up | 站起来 | 🧍 |
| Sit down | 坐下 | 🪑 |
| Jump up | 跳一跳 | ⬆️ |
| Turn around | 转个圈 | 🔄 |
| Run in place | 原地跑 | 🏃 |
| Walk slowly | 慢慢走 | 🚶 |
| Hop like a rabbit | 像兔子一样跳 | 🐰 |
| Fly like a bird | 像小鸟一样飞 | 🐦 |

**日常作息**

| 英文 | 中文 | emoji |
|---|---|---|
| Open the door | 打开门 | 🚪 |
| Close the door | 关上门 | 🚪 |
| Pick up the ball | 捡起球 | ⚽ |
| Throw the ball | 扔球 | 🏀 |
| Wash your hands | 洗手 | 🧼 |
| Brush your teeth | 刷牙 | 🪥 |
| Put on your hat | 戴上帽子 | 🎩 |
| Take off your hat | 摘下帽子 | 🧢 |

### 交互

- 每轮把 24 条洗牌后取前 8 条，本轮内不重复；「再来一轮」重新洗牌。
- 卡片从上到下显示：大 emoji、英文指令、中文提示。
- 按钮：
  - `🔊 再听一遍`：朗读当前英文指令
  - `下一条 →`：前进到下一题并自动朗读
- 进度显示 `3 / 8`。
- 不自动播放、不自动前进，给孩子留出做动作的时间。
- 第 8 条点击「下一条」后显示完成卡片：「完成 8 个指令！」和「再来一轮」。

## 模块二：说句子（亲子小对话）

### 数据

```js
const DIALOGUE_DATA = [
  {
    scene: '认识东西',
    parent: { en: "What's this?", cn: '这是什么？' },
    child: { en: "It's a {x}.", cn: '这是一只{xCn}。' },
    subs: [
      { en: 'cat', cn: '猫', emoji: '🐱' },
      { en: 'dog', cn: '狗', emoji: '🐶' },
      { en: 'duck', cn: '鸭子', emoji: '🦆' }
    ]
  }
  // ...
];
```

6 组对话：

| 场景 | 家长 | 孩子句式 | 替换词 |
|---|---|---|---|
| 认识东西 | What's this? | It's a {x}. | cat 🐱、dog 🐶、duck 🦆 |
| 想要东西 | What do you want? | I want a {x}. | banana 🍌、apple 🍎、cookie 🍪 |
| 问候 | How are you? | I'm {x}. | happy 😄、sad 😢、hungry 😋 |
| 问颜色 | What color is it? | It's {x}. | red ❤️、blue 💙、green 💚 |
| 请求 | Can I help you? | Can I have a {x}? | apple 🍎、banana 🍌、cookie 🍪 |
| 睡前 | Good night! | Good night, {x}! | Mommy 👩、Daddy 👨、Teddy 🧸 |

说明：

- 每组固定 3 个替换词，便于页面布局。
- 中文模板中的 `{xCn}` 由替换词的 `cn` 填入；例如「问候」的 `cn` 值为开心、难过、饿了，「问颜色」的模板为 `它是{xCn}。`。
- 替换后的句子用于显示和朗读，不修改原始数据。

### 交互

- 页面显示两个对话气泡：🧑 家长、🧒 孩子。
- 点击任一气泡，单独朗读该句；孩子气泡显示当前替换词生成的完整句子。
- 孩子气泡下方显示 3 个替换词按钮（emoji + 英文），点击后：
  - 更新孩子气泡文字
  - 朗读替换后的完整句子
- `🔊 播放对话`：先读家长句，250ms 后读孩子句；使用 token 防止离开页面后继续朗读。
- `下一个 →`：切换到下一组，重置替换词为第一个。
- 进度显示 `1 / 6`；第 6 组点击「下一个」后显示完成卡片和「再来一轮」。

## 模块三：找一找（I Spy）

### 数据

```js
const ISPY_DATA = [
  { en: 'apple', cn: '苹果', emoji: '🍎', color: 'red', colorCn: '红色' },
  // 共 12 个，覆盖 6 种颜色
];
```

12 个物品：

| 物品 | 中文 | emoji | 颜色 |
|---|---|---|---|
| apple | 苹果 | 🍎 | red 红色 |
| strawberry | 草莓 | 🍓 | red 红色 |
| car | 汽车 | 🚗 | blue 蓝色 |
| fish | 鱼 | 🐟 | blue 蓝色 |
| tree | 树 | 🌳 | green 绿色 |
| frog | 青蛙 | 🐸 | green 绿色 |
| banana | 香蕉 | 🍌 | yellow 黄色 |
| sun | 太阳 | ☀️ | yellow 黄色 |
| flower | 花 | 🌸 | pink 粉色 |
| pig | 猪 | 🐷 | pink 粉色 |
| grape | 葡萄 | 🍇 | purple 紫色 |
| balloon | 气球 | 🎈 | purple 紫色 |

### 出题规则

- 每轮从 12 个物品中选出 6 个，**颜色互不相同**（6 种颜色各取一个）。
- 从这 6 个中随机选一个作为目标。
- 题目类型各 50%：
  - 找物品：`Find the apple.` / 中文提示「找苹果」
  - 找颜色：`Find something red.` / 中文提示「找红色的东西」
- 因为图中每种颜色只有一个物品，颜色题有且只有一个正确答案；物品题的目标 `en` 在数据中唯一，也只有一个正确答案。
- 共 5 轮，每轮重新生成图卡和目标。

### 交互

- 顶部显示进度 `1 / 5`、题目中文提示和喇叭按钮（重播题目）。
- 下方 6 张 emoji 图卡，大号可点击。
- 点对：
  - 图卡绿色高亮
  - 显示星星反馈
  - 700ms 后自动进入下一轮；第 5 轮后显示完成卡片（5 颗星）和「再玩一次」
- 点错：
  - 图卡抖动
  - 重新朗读题目
  - 不扣分、不前进，可继续尝试
- 离开页面或切换标签时，取消 700ms 自动前进定时器。

## 状态与函数

新增状态：

```js
let currentPlayTab = 'tpr';   // 'tpr' | 'dialogue' | 'ispy'
let tprQueue = [];
let tprIndex = 0;
let dialogueIndex = 0;
let dialogueSubIndex = 0;
let ispyRound = 0;
let ispyTarget = null;
let ispyPromptType = 'object'; // 'object' | 'color'
let ispyTiles = [];
let ispyStars = 0;
let playSpeechToken = 0;
let playTimer = null;
```

核心函数：

- `showPlay()`：停止其它模块的自动播放和本模块语音，切换页面，渲染标签和内容
- `switchPlayTab(tab)`：停止语音和定时器，切换标签并重新渲染
- `renderPlay()`：渲染标签，按 `currentPlayTab` 分派到三个模块
- `startTprRound()` / `renderTpr()` / `speakTpr()` / `nextTpr()`
- `renderDialogue()` / `fillDialogue(template, sub)` / `speakDialogue(role)` / `selectDialogueSub(i)` / `playDialogue()` / `nextDialogue()`
- `startIspyRound()` / `renderIspy()` / `speakIspyPrompt()` / `answerIspy(idx)` / `nextIspy()`
- `stopPlaySpeech()`：递增 `playSpeechToken`，清除 `playTimer`，调用 `synth.cancel()`

`goHome()` 增加 `stopPlaySpeech()`，与现有三个自动播放的清理方式一致。

## 语音方案

- 复用现有 `speak()`：`lang = 'en-US'`、`rate = 0.85`、`pitch = 1.1`
- 每次播放前由 `speak()` 调用 `synth.cancel()`，保证不重叠
- 对话 A→B 连读使用 `speak(parent, callback)` + 250ms `setTimeout`，回调内检查 token
- I Spy 自动前进使用 `playTimer` + token 检查
- 浏览器无可用英文语音时静默失败，文字和 emoji 仍可正常使用

## 样式

复用现有变量与组件：

- 页面：`.page` / `.active`
- 标签：`.category-tabs` / `.cat-tab`
- 按钮：`.tool-btn` / `.big-btn`
- 进度：`.progress-bar-wrap` / `.progress-bar-fill`
- 卡片：沿用 `--radius-sm`、`--shadow`、`--c1` 至 `--c7`

新增样式约 100 行：

- `.play-panel`：模块容器
- `.play-card` / `.play-emoji` / `.play-en` / `.play-cn`：指令卡
- `.play-progress`：进度文字
- `.dialogue-row` / `.bubble` / `.bubble-parent` / `.bubble-child`：对话气泡
- `.sub-chip`：替换词按钮
- `.ispy-grid` / `.ispy-tile` / `.ispy-tile.correct` / `.ispy-tile.wrong`
- `.play-complete`：完成卡片

所有可点击元素不小于 64×64px，保持幼儿友好。

## 改动范围

仅修改 `index.html`：

1. `<style>` 新增互动乐园样式（约 100 行）
2. 首页新增第 5 张入口卡片
3. 新增 `#playPage` 页面结构
4. `<script>` 新增 `TPR_DATA`、`DIALOGUE_DATA`、`ISPY_DATA`、状态、渲染、朗读、推进函数
5. `goHome()` 增加一行 `stopPlaySpeech()`
6. 现有四个模块的代码不变

不新增文件、不引入依赖、不改动 `pinyin.html`。

## 错误处理与降级

- 无英文语音：静默失败，页面文字、中文提示和 emoji 仍可学习
- 快速点击或多个标签切换：`speak()` 的 `synth.cancel()` 与 token 保证只播放最后一次
- 离开页面：`goHome()` 和 `switchPlayTab()` 清理语音、定时器和高亮
- I Spy 图卡生成失败（数据不足）：本轮回退为前 6 个不重复物品，保证页面不空白

## 验证标准

- 首页显示 5 个入口，互动乐园可进入、可返回
- 三个标签可切换，默认显示「听指令」
- 听指令每轮 8 条、不重复；点击朗读、下一条推进、完成后可再来一轮
- 说句子 6 组可切换；点气泡朗读正确句子；替换词能改变孩子句；A→B 连读顺序正确
- 找一找每轮 6 张图卡、颜色互不相同、目标在图中；答对进入下一轮，答错不前进；5 轮后完成
- 离开互动乐园或切换标签后不再有残留朗读或定时器
- 原字母、单词、音标、练习模块的既有测试全部通过
