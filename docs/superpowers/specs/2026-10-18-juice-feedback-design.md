# 趣味反馈增强设计

日期：2026-10-18
状态：已与用户确认

## 背景

现有 `index.html` 已包含字母学习、单词学习、趣味练习、音标乐园、互动乐园五个模块。当前反馈只有答对变色 + 弹跳、答错抖动、完成页静态星级和背景飘星；学习浏览页几乎没有即时回应。用户反馈孩子学习一会儿就失去耐心，主要原因是**反馈不够热闹**。

本次在不改变任何学习内容和答题逻辑的前提下，增加统一的即时反馈：合成音效、庆祝动画、🐱 小猫吉祥物，覆盖全部模块。

## 目标与非目标

**目标**

- 全部模块接入统一的即时反馈
- 三档强度：轻（点击）、中（答对）、重（完成）
- Web Audio 现场合成音效，不引入任何音频文件
- 🐱 小猫吉祥物对答对、答错、完成做出不同反应
- 音效默认开启，导航栏提供静音开关
- 保持单文件、零构建、零网络依赖

**非目标**

- 不做贴纸收集、等级、连续学习天数等长期激励
- 不做英文语音表扬（避免与现有朗读互相打断）
- 不新增页面、外部音频、图片、字体或第三方库
- 不改变现有答题节奏、判定逻辑、进度与计分
- 不写 `localStorage`（静音设置仅当前会话有效）

## 页面结构

在 `#app` 内、`#navBar` 之前加入全屏粒子层：

```html
<div id="fxLayer" aria-hidden="true"></div>
```

`#navBar` 追加右侧操作区：

```html
<div class="nav-actions">
  <button id="soundToggle" class="nav-sound" onclick="toggleSound()" aria-label="关闭音效">🔊</button>
  <span id="mascot" class="nav-mascot" aria-hidden="true">🐱</span>
</div>
```

- `#fxLayer` 固定覆盖全屏，`pointer-events: none`，`overflow: hidden`，层级高于页面内容
- `.nav-actions` 使用 `margin-left: auto` 靠右；静音按钮与 `.nav-back` 同为 56×56，吉祥物 32px
- 首页导航栏隐藏，因此吉祥物只在五个学习/游戏页面出现；首页自身已有书本装饰

## 反馈引擎

全部逻辑内联在 `index.html` 的 `<script>` 中，约 150 行（含 CSS）。

### 音效

```js
let soundOn = true;
let audioCtx = null;

function getAudioCtx() {
  if (!audioCtx) {
    const Ctx = window.AudioContext || window.webkitAudioContext;
    if (!Ctx) return null;
    audioCtx = new Ctx();
  }
  if (audioCtx.state === 'suspended') audioCtx.resume();
  return audioCtx;
}

function tone(freq, startOffset, duration, type = 'sine', volume = 0.1) { ... }

function playSfx(kind) {
  if (!soundOn) return;
  const ctx = getAudioCtx();
  if (!ctx) return;
  // tap / correct / wrong / complete
}
```

音效定义：

| kind | 音高与节奏 | 用途 |
|---|---|---|
| `tap` | 660Hz 正弦波，80ms | 点击卡片、标签、按钮 |
| `correct` | C5-E5-G5（523/659/784Hz），每个 80ms 起始、180ms 时值 | 答对 |
| `wrong` | 330Hz → 262Hz，三角波，柔和下行 | 答错 |
| `complete` | C5-E5-G5-C6，间隔 120ms | 完成一关 |

- 每个音用独立的 `OscillatorNode` + `GainNode`，`gain` 峰值约 0.1，指数衰减
- 所有调用都发生在用户点击回调内，满足浏览器音频解锁要求
- `AudioContext` 创建失败时静默跳过，不影响页面功能
- 音效不调用 `synth.cancel()`，不打断英文朗读

### 粒子

```js
function fxBurstFrom(el, count)   // 以元素中心为起点
function fxBurst(x, y, count)     // 以视口坐标为起点
function fxConfetti(count)        // 顶部飘落的彩带
function fxClear()                // 清空粒子层
```

- `fxBurst`：生成 8~12 个 emoji 星点（⭐ ✨ 💛 🌟），用 CSS 变量 `--dx` / `--dy` / `--rot` 向随机方向散开，650ms 后移除
- `fxConfetti`：生成 24 片 8×14px 彩色矩形，从顶部随机横向位置下落，1.8~2.4s 后移除
- 粒子数量上限 24，元素通过 `setTimeout` 自清理，不依赖 `animationend`
- `fxClear()` 在 `goHome()` 时调用，避免离开页面后残留

### 吉祥物

```js
function fxMascot(state) // 'peek' | 'cheer' | 'oops'
```

- `peek`：轻轻探头缩放，400ms
- `cheer`：向上跳起并左右摆动，700ms
- `oops`：温和左右摇头，600ms
- 使用单一 `mascotTimer`，重复触发时清除上一个定时器并重启动画
- 完成时的中央大猫：

```js
function fxMascotDance() // 在 #fxLayer 生成 96px 🐱，跳舞 1.8s 后移除
```

### 完成反馈

```js
function fxCelebrate() {
  playSfx('complete');
  fxConfetti(24);
  fxMascotDance();
}
```

## 接入点

### 轻反馈（点击音 + 部分小猫探头）

使用一个全局委托监听，避免逐个修改现有函数：

```js
const TAP_SELECTOR = '.home-card, .letter-card, .word-card, .phonics-card, .phonics-example, .cat-tab, .tool-btn, .big-btn, .quiz-mode-card, .bubble, .sub-chip, .quiz-speaker-btn';

document.addEventListener('click', event => {
  if (event.target.closest('#soundToggle')) return;
  const el = event.target.closest(TAP_SELECTOR);
  if (!el) return;
  playSfx('tap');
  if (el.matches('.letter-card, .word-card, .phonics-card, .phonics-example, .bubble, .sub-chip')) {
    fxMascot('peek');
  }
});
```

- `.quiz-option` 与 `.ispy-tile` **不**在轻反馈列表中，由答对/答错反馈统一发声，避免双重音效
- `.nav-back` 不发声
- 静音按钮自身不发声

### 中反馈（答对 / 答错）

| 位置 | 答对 | 答错 |
|---|---|---|
| `answerQuiz(idx)` | `playSfx('correct')` + `fxBurstFrom(选项按钮, 10)` + `fxMascot('cheer')` | `playSfx('wrong')` + `fxMascot('oops')` |
| `answerIspy(i)` | 同上，起点为被点中的图卡 | 同上 |

- 在现有 `.correct` / `.wrong` class 添加之后调用
- 不新增、不延长任何 `setTimeout` 延迟；粒子和动画都在既有 700ms / 900ms 窗口内完成

### 重反馈（完成）

| 位置 | 触发条件 |
|---|---|
| `showQuizComplete()` | 进入完成页时调用 `fxCelebrate()` |
| `nextTpr()` | `tprIndex` 自增后达到 `tprQueue.length` 时调用 |
| `nextDialogue()` | `dialogueIndex` 自增后达到 `DIALOGUE_DATA.length` 时调用 |
| `answerIspy(i)` 的 700ms 回调 | `ispyRound` 自增后达到 5 时，在 `renderIspy()` 之后调用 |

- 只在**首次进入完成态**时触发；切换标签、重新渲染完成页不重复庆祝
- 「再来一轮 / 再来一次 / 再玩一次」按钮不触发完成反馈，只走轻反馈

## 状态与函数

新增：

```js
let soundOn = true;       // 当前会话音效开关
let audioCtx = null;      // 惰性创建的 AudioContext
let mascotTimer = null;   // 吉祥物动画复位定时器
```

新增函数：

- `toggleSound()`
- `getAudioCtx()` / `tone()` / `playSfx(kind)`
- `fxBurstFrom(el, count)` / `fxBurst(x, y, count)` / `fxConfetti(count)` / `fxClear()`
- `fxMascot(state)` / `fxMascotDance()` / `fxCelebrate()`

修改函数仅增加调用，不改变原有分支：

- `answerQuiz`、`showQuizComplete`
- `answerIspy`、`nextTpr`、`nextDialogue`
- `goHome`（增加 `fxClear()`）

## 样式

新增约 60 行 CSS：

```css
#fxLayer { position: fixed; inset: 0; pointer-events: none; overflow: hidden; z-index: 100; }
.nav-actions { margin-left: auto; display: flex; align-items: center; gap: 8px; }
.nav-sound { width: 56px; height: 56px; border-radius: 50%; ... }
.nav-mascot { font-size: 32px; display: inline-block; ... }
.fx-particle { position: fixed; font-size: 22px; animation: fxBurst 0.65s ease-out forwards; }
.fx-confetti { position: fixed; top: -20px; width: 8px; height: 14px; animation: fxFall 1.8s linear forwards; }
.fx-mascot { position: fixed; left: 50%; top: 50%; font-size: 96px; transform: translate(-50%, -50%); animation: fxDance 1.8s ease-in-out forwards; }
@keyframes fxBurst { ... }
@keyframes fxFall { ... }
@keyframes fxDance { ... }
@keyframes mascotCheer { ... }
@keyframes mascotOops { ... }
@keyframes mascotPeek { ... }
```

- 复用现有 CSS 变量与阴影样式
- 静音按钮与返回按钮尺寸一致，保持顶部视觉平衡
- 吉祥物在导航栏内，不遮挡页面内容

## 错误处理与降级

- 浏览器不支持 `AudioContext`：所有音效静默跳过，动画与吉祥物照常
- `#fxLayer` 或 `#mascot` 不存在：对应函数直接返回，不报错
- 连续快速点击：`mascotTimer` 复位，粒子各自独立清理，数量上限 24
- 离开页面：`goHome()` 调用 `fxClear()`；中央大猫和彩带也会在 1.8~2.4s 内自行移除
- 静音：只阻止后续音效，动画不受影响；刷新页面恢复开启
- 无网络、无外部资源依赖不变

## 验证标准

- 点击字母/单词/音标卡片、对话气泡、替换词：有短音效，小猫探头
- 趣味练习答对：上行三音、选项位置散出星星、小猫欢呼
- 趣味练习答错：柔和下行音、小猫摇头，选项抖动与重读行为不变
- 找一找答对/答错：与练习一致，700ms 自动进入下一轮不被延长
- 完成练习、TPR、对话、找一找：号角旋律 + 彩带 + 中央大猫跳舞，1.8s 后自动清空
- 切换标签或重新渲染完成页不重复触发庆祝
- 静音后无任何音效，动画照常；再次点击恢复
- 375px 下导航栏不换行、无横向溢出
- 原有 39 项测试全部通过；新增反馈测试覆盖音效触发次数、粒子/吉祥物状态、静音开关

## 改动范围

仅修改 `index.html`：

1. `<style>` 新增反馈样式与关键帧（约 60 行）
2. `#app` 内新增 `#fxLayer`
3. `#navBar` 新增静音按钮与吉祥物
4. `<script>` 新增状态、音效、粒子、吉祥物与庆祝函数
5. 在 `answerQuiz`、`showQuizComplete`、`answerIspy`、`nextTpr`、`nextDialogue`、`goHome` 中增加调用
6. 新增全局点击委托监听

不新增文件、不引入依赖、不改动数据与任何既有判定逻辑。
