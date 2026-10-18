# 互动乐园 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在现有英语启蒙单页应用中新增「互动乐园」，包含听指令（TPR）、亲子句型对话、找一找（I Spy）三个互动模块。

**Architecture:** 仅修改 `index.html`。新增一个 `#playPage`，通过三个标签切换三个模块；复用现有 `speak()`、导航、标签和卡片样式。测试位于仓库外的 `E:/AI-work/english-study-e2e`，使用 Playwright + 假 `speechSynthesis`。

**Tech Stack:** 原生 HTML/CSS/JS；Web Speech API；Playwright Core（测试）；Node.js `assert`。

---

## 文件结构

- 修改：`E:/AI-work/english-study/index.html`
  - `<style>`：互动乐园样式
  - 首页：第 5 张入口卡片
  - 页面：`#playPage` + `#playTabs` + `#playContent`
  - `<script>`：`TPR_DATA`、`DIALOGUE_DATA`、`ISPY_DATA`、状态、渲染/朗读/推进函数
  - `goHome()`：增加 `stopPlaySpeech()`
- 创建/修改测试：`E:/AI-work/english-study-e2e/play.test.js`
- 修改测试：`E:/AI-work/english-study-e2e/baseline.test.js`（首页卡片数 4 → 5）
- 修改测试：`E:/AI-work/english-study-e2e/regression.test.js`（互动乐园返回首页）

---

## Task 1: 添加互动乐园数据

**Files:**
- Modify: `E:/AI-work/english-study/index.html` — 在 `PHONICS_DATA` 之后、`// ===== STATE =====` 之前插入三组数据
- Create: `E:/AI-work/english-study-e2e/play.test.js`

- [ ] **Step 1: 写失败测试**

创建 `E:/AI-work/english-study-e2e/play.test.js`：

```js
const { assert, openApp, runTest, finish } = require('./harness');

(async () => {
  const { browser, page } = await openApp();

  await runTest('interactive play data is complete', async () => {
    const data = await page.evaluate(() => ({
      tpr: TPR_DATA,
      dialogue: DIALOGUE_DATA,
      ispy: ISPY_DATA
    }));

    assert.strictEqual(data.tpr.length, 24);
    assert.strictEqual(new Set(data.tpr.map(i => i.en)).size, 24);
    data.tpr.forEach(item => {
      assert.ok(item.en && item.cn && item.emoji, `bad TPR item: ${JSON.stringify(item)}`);
    });

    assert.strictEqual(data.dialogue.length, 6);
    data.dialogue.forEach(d => {
      assert.ok(d.scene && d.parent.en && d.parent.cn && d.child.en && d.child.cn);
      assert.strictEqual(d.subs.length, 3);
      d.subs.forEach(s => assert.ok(s.en && s.cn && s.emoji));
      assert.ok(d.child.en.includes('{x}'), `missing {x}: ${d.child.en}`);
      assert.ok(d.child.cn.includes('{xCn}'), `missing {xCn}: ${d.child.cn}`);
      if (d.child.en.includes('a {x}')) {
        d.subs.forEach(s => assert.ok(!/^[aeiou]/i.test(s.en), `a/an error: ${s.en}`));
      }
    });

    assert.strictEqual(data.ispy.length, 12);
    assert.strictEqual(new Set(data.ispy.map(i => i.en)).size, 12);
    const colorCounts = {};
    const colorNames = {};
    data.ispy.forEach(item => {
      assert.ok(item.en && item.cn && item.emoji && item.color && item.colorCn);
      colorCounts[item.color] = (colorCounts[item.color] || 0) + 1;
      colorNames[item.color] = item.colorCn;
    });
    assert.strictEqual(Object.keys(colorCounts).length, 6);
    Object.values(colorCounts).forEach(n => assert.strictEqual(n, 2));
    assert.deepStrictEqual(colorNames, {
      red: '红色', blue: '蓝色', green: '绿色',
      yellow: '黄色', pink: '粉色', purple: '紫色'
    });
  });

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
```
Expected: FAIL，页面抛出 `TPR_DATA is not defined`。

- [ ] **Step 3: 添加数据**

在 `index.html` 中，紧接 `PHONICS_DATA` 的结束 `};`（当前约第 800 行）之后、`// ===== STATE =====` 之前插入：

```js
const TPR_DATA = [
  // 身体部位
  { en: 'Touch your nose', cn: '摸摸鼻子', emoji: '👃' },
  { en: 'Touch your ears', cn: '摸摸耳朵', emoji: '👂' },
  { en: 'Touch your head', cn: '摸摸头', emoji: '🙆' },
  { en: 'Touch your tummy', cn: '摸摸肚子', emoji: '🧒' },
  { en: 'Touch your feet', cn: '摸摸脚', emoji: '🦶' },
  { en: 'Clap your hands', cn: '拍拍手', emoji: '👏' },
  { en: 'Wave your hands', cn: '挥挥手', emoji: '👋' },
  { en: 'Blink your eyes', cn: '眨眨眼', emoji: '👀' },
  // 动作
  { en: 'Stand up', cn: '站起来', emoji: '🧍' },
  { en: 'Sit down', cn: '坐下', emoji: '🪑' },
  { en: 'Jump up', cn: '跳一跳', emoji: '⬆️' },
  { en: 'Turn around', cn: '转个圈', emoji: '🔄' },
  { en: 'Run in place', cn: '原地跑', emoji: '🏃' },
  { en: 'Walk slowly', cn: '慢慢走', emoji: '🚶' },
  { en: 'Hop like a rabbit', cn: '像兔子一样跳', emoji: '🐰' },
  { en: 'Fly like a bird', cn: '像小鸟一样飞', emoji: '🐦' },
  // 日常作息
  { en: 'Open the door', cn: '打开门', emoji: '🚪' },
  { en: 'Close the door', cn: '关上门', emoji: '🚪' },
  { en: 'Pick up the ball', cn: '捡起球', emoji: '⚽' },
  { en: 'Throw the ball', cn: '扔球', emoji: '🏀' },
  { en: 'Wash your hands', cn: '洗手', emoji: '🧼' },
  { en: 'Brush your teeth', cn: '刷牙', emoji: '🪥' },
  { en: 'Put on your hat', cn: '戴上帽子', emoji: '🎩' },
  { en: 'Take off your hat', cn: '摘下帽子', emoji: '🧢' }
];

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
  },
  {
    scene: '想要东西',
    parent: { en: 'What do you want?', cn: '你想要什么？' },
    child: { en: 'I want a {x}.', cn: '我想要{xCn}。' },
    subs: [
      { en: 'banana', cn: '香蕉', emoji: '🍌' },
      { en: 'peach', cn: '桃子', emoji: '🍑' },
      { en: 'cookie', cn: '饼干', emoji: '🍪' }
    ]
  },
  {
    scene: '问候',
    parent: { en: 'How are you?', cn: '你好吗？' },
    child: { en: "I'm {x}.", cn: '我{xCn}。' },
    subs: [
      { en: 'happy', cn: '开心', emoji: '😄' },
      { en: 'sad', cn: '难过', emoji: '😢' },
      { en: 'hungry', cn: '饿了', emoji: '😋' }
    ]
  },
  {
    scene: '问颜色',
    parent: { en: 'What color is it?', cn: '它是什么颜色？' },
    child: { en: "It's {x}.", cn: '它是{xCn}。' },
    subs: [
      { en: 'red', cn: '红色', emoji: '❤️' },
      { en: 'blue', cn: '蓝色', emoji: '💙' },
      { en: 'green', cn: '绿色', emoji: '💚' }
    ]
  },
  {
    scene: '请求',
    parent: { en: 'Can I help you?', cn: '你需要什么吗？' },
    child: { en: 'Can I have a {x}?', cn: '我可以要{xCn}吗？' },
    subs: [
      { en: 'peach', cn: '桃子', emoji: '🍑' },
      { en: 'banana', cn: '香蕉', emoji: '🍌' },
      { en: 'cookie', cn: '饼干', emoji: '🍪' }
    ]
  },
  {
    scene: '睡前',
    parent: { en: 'Good night!', cn: '晚安！' },
    child: { en: 'Good night, {x}!', cn: '晚安，{xCn}！' },
    subs: [
      { en: 'Mommy', cn: '妈妈', emoji: '👩' },
      { en: 'Daddy', cn: '爸爸', emoji: '👨' },
      { en: 'Teddy', cn: '泰迪熊', emoji: '🧸' }
    ]
  }
];

const ISPY_DATA = [
  { en: 'apple', cn: '苹果', emoji: '🍎', color: 'red', colorCn: '红色' },
  { en: 'strawberry', cn: '草莓', emoji: '🍓', color: 'red', colorCn: '红色' },
  { en: 'whale', cn: '鲸鱼', emoji: '🐳', color: 'blue', colorCn: '蓝色' },
  { en: 'fish', cn: '鱼', emoji: '🐟', color: 'blue', colorCn: '蓝色' },
  { en: 'tree', cn: '树', emoji: '🌳', color: 'green', colorCn: '绿色' },
  { en: 'frog', cn: '青蛙', emoji: '🐸', color: 'green', colorCn: '绿色' },
  { en: 'banana', cn: '香蕉', emoji: '🍌', color: 'yellow', colorCn: '黄色' },
  { en: 'sun', cn: '太阳', emoji: '☀️', color: 'yellow', colorCn: '黄色' },
  { en: 'flower', cn: '花', emoji: '🌸', color: 'pink', colorCn: '粉色' },
  { en: 'pig', cn: '猪', emoji: '🐷', color: 'pink', colorCn: '粉色' },
  { en: 'grape', cn: '葡萄', emoji: '🍇', color: 'purple', colorCn: '紫色' },
  { en: 'eggplant', cn: '茄子', emoji: '🍆', color: 'purple', colorCn: '紫色' }
];
```

- [ ] **Step 4: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
```
Expected:
```
PASS  interactive play data is complete

1 passed, 0 failed
```

- [ ] **Step 5: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add interactive play data

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 2: 添加互动乐园入口、页面与听指令模块

**Files:**
- Modify: `E:/AI-work/english-study/index.html` — 首页卡片、页面结构、样式、状态、通用函数、TPR 函数、`goHome()`
- Modify: `E:/AI-work/english-study-e2e/baseline.test.js` — 首页 4 → 5 张卡片
- Modify: `E:/AI-work/english-study-e2e/play.test.js` — 入口、标签、TPR 测试

- [ ] **Step 1: 写失败测试**

在 `play.test.js` 的 `await finish(browser);` 之前插入：

```js
  await runTest('home page shows interactive play entry and page tabs', async () => {
    assert.strictEqual(await page.locator('.home-card').count(), 5);
    assert.ok((await page.locator('.home-cards').innerText()).includes('互动乐园'));

    await page.evaluate(() => showPlay());
    assert.strictEqual(
      await page.locator('#playPage').evaluate(el => el.classList.contains('active')),
      true
    );
    assert.strictEqual(await page.locator('#navTitle').innerText(), '互动乐园');
    assert.strictEqual(await page.locator('.cat-tab').count(), 3);
    assert.ok((await page.locator('.cat-tab.active').innerText()).includes('听指令'));
  });

  await runTest('tpr round has eight unique commands and advances with speech', async () => {
    await page.evaluate(() => { window.__spoken.length = 0; showPlay(); });
    await page.waitForTimeout(80);

    assert.strictEqual(await page.evaluate(() => tprQueue.length), 8);
    assert.strictEqual(await page.evaluate(() => new Set(tprQueue).size), 8);
    assert.strictEqual(await page.locator('.play-card .play-en').count(), 1);

    const first = await page.evaluate(() => TPR_DATA[tprQueue[0]].en);
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), [first]);

    await page.locator('.play-actions .tool-btn.active').click();
    await page.waitForFunction(() => window.__spoken.length === 2);
    const second = await page.evaluate(() => TPR_DATA[tprQueue[1]].en);
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), [first, second]);

    await page.evaluate(() => { tprIndex = 7; renderTpr(); });
    await page.locator('.play-actions .tool-btn.active').click();
    assert.ok((await page.locator('.play-complete-title').innerText()).includes('完成 8 个指令'));
  });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
node baseline.test.js
```
Expected:
- `play.test.js` FAIL：首页只有 4 张卡片，`showPlay is not defined`。
- `baseline.test.js` 仍通过（此时尚未加第 5 张卡片）。

- [ ] **Step 3: 添加首页入口**

在首页 `home-cards` 中，紧接「音标乐园」按钮（当前约第 540-546 行）之后插入：

```html
      <button class="home-card" style="background:linear-gradient(135deg,var(--c2),var(--c3))" onclick="showPlay()">
        <span class="home-card-icon">&#127918;</span>
        <span class="home-card-text">
          <span class="home-card-label">互动乐园</span>
          <span class="home-card-desc">听指令、说句子、找一找</span>
        </span>
      </button>
```

- [ ] **Step 4: 添加页面结构**

在 `#phonicsPage` 的结束 `</div>`（当前约第 578 行）之后、`#quizPage` 之前插入：

```html
  <!-- ====== PLAY PAGE ====== -->
  <div id="playPage" class="page">
    <div class="category-tabs" id="playTabs"></div>
    <div id="playContent"></div>
  </div>
```

- [ ] **Step 5: 添加样式**

在 `/* ===== hidden util ===== */`（当前约第 489 行）之前插入：

```css
/* ===== Interactive Play ===== */
.play-progress {
  font-size: var(--font-md);
  font-weight: 700;
  color: var(--text-light);
  margin-bottom: 14px;
}
.play-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  background: var(--card-bg);
  border-radius: var(--radius);
  padding: 34px 24px;
  box-shadow: var(--shadow);
  border: 3px solid var(--c2);
  margin-bottom: 18px;
}
.play-emoji { font-size: 84px; line-height: 1; }
.play-en { font-size: 34px; font-weight: 800; text-align: center; color: var(--text); }
.play-cn { font-size: var(--font-md); color: var(--text-light); font-weight: 600; }
.play-actions { display: flex; gap: 12px; flex-wrap: wrap; }
.play-complete {
  text-align: center;
  background: var(--card-bg);
  border-radius: var(--radius);
  padding: 40px 24px;
  box-shadow: var(--shadow);
}
.play-complete-icon { font-size: 72px; }
.play-complete-title { font-size: 32px; font-weight: 800; margin: 12px 0 20px; }
```

- [ ] **Step 6: 添加状态**

在 `let phonicsAutoToken = 0;`（当前约第 816 行）之后插入：

```js
let currentPlayTab = 'tpr'; // 'tpr' | 'dialogue' | 'ispy'
let tprQueue = [];
let tprIndex = 0;
let dialogueIndex = 0;
let dialogueSubIndex = 0;
let ispyRound = 0;
let ispyTarget = null;
let ispyPromptType = 'object'; // 'object' | 'color'
let ispyTiles = [];
let ispyStars = 0;
let ispyLocked = false;
let playSpeechToken = 0;
let playTimer = null;
```

- [ ] **Step 7: 添加通用函数与 TPR 模块**

在 `autoPlayPhonics()` 的结束 `}`（当前约第 1164 行）之后、`// ===== QUIZ =====` 之前插入：

```js
// ===== INTERACTIVE PLAY =====
function showPlay() {
  stopAutoPlay();
  stopWordsAuto();
  stopPhonicsAuto();
  stopPlaySpeech();
  hideAllPages();
  showNav('互动乐园');
  document.getElementById('playPage').classList.add('active');
  currentPlayTab = 'tpr';
  tprQueue = [];
  tprIndex = 0;
  dialogueIndex = 0;
  dialogueSubIndex = 0;
  ispyRound = 0;
  ispyStars = 0;
  ispyTiles = [];
  ispyTarget = null;
  ispyLocked = false;
  renderPlayTabs();
  startTprRound();
}

function renderPlayTabs() {
  const tabs = [
    { key: 'tpr', label: '🎧 听指令' },
    { key: 'dialogue', label: '💬 说句子' },
    { key: 'ispy', label: '🔍 找一找' }
  ];
  document.getElementById('playTabs').innerHTML = tabs.map(t =>
    `<button class="cat-tab ${currentPlayTab === t.key ? 'active' : ''}" onclick="switchPlayTab('${t.key}')">${t.label}</button>`
  ).join('');
}

function switchPlayTab(tab) {
  stopPlaySpeech();
  currentPlayTab = tab;
  renderPlayTabs();
  renderPlay();
}

function renderPlay() {
  if (currentPlayTab === 'tpr') renderTpr();
  else document.getElementById('playContent').innerHTML =
    '<div class="play-progress" style="text-align:center;padding:40px 0;">这个模块马上就来～</div>';
}

function stopPlaySpeech() {
  playSpeechToken++;
  ispyLocked = false;
  if (playTimer) { clearTimeout(playTimer); playTimer = null; }
  synth.cancel();
}

function startTprRound() {
  tprQueue = shuffleArray(TPR_DATA.map((_, i) => i)).slice(0, 8);
  tprIndex = 0;
  renderTpr();
  speakTpr();
}

function renderTpr() {
  if (!tprQueue.length) {
    tprQueue = shuffleArray(TPR_DATA.map((_, i) => i)).slice(0, 8);
    tprIndex = 0;
  }
  const content = document.getElementById('playContent');
  if (tprIndex >= tprQueue.length) {
    content.innerHTML = `
      <div class="play-complete">
        <div class="play-complete-icon">🎉</div>
        <div class="play-complete-title">完成 8 个指令！</div>
        <button class="big-btn" style="background:var(--c4)" onclick="startTprRound()">🔄 再来一轮</button>
      </div>`;
    return;
  }
  const item = TPR_DATA[tprQueue[tprIndex]];
  content.innerHTML = `
    <div class="play-progress">${tprIndex + 1} / ${tprQueue.length}</div>
    <div class="play-card">
      <span class="play-emoji">${item.emoji}</span>
      <span class="play-en">${item.en}</span>
      <span class="play-cn">${item.cn}</span>
    </div>
    <div class="play-actions">
      <button class="tool-btn" onclick="speakTpr()">🔊 再听一遍</button>
      <button class="tool-btn active" onclick="nextTpr()">下一条 →</button>
    </div>`;
}

function speakTpr() {
  const item = TPR_DATA[tprQueue[tprIndex]];
  if (item) speak(item.en);
}

function nextTpr() {
  if (tprIndex >= tprQueue.length) return;
  tprIndex++;
  renderTpr();
  speakTpr();
}
```

- [ ] **Step 8: 在 `goHome()` 中清理互动乐园语音**

把 `goHome()` 中的：

```js
function goHome() {
  stopAutoPlay();
  stopWordsAuto();
  stopPhonicsAuto();
  hideAllPages();
```

改为：

```js
function goHome() {
  stopAutoPlay();
  stopWordsAuto();
  stopPhonicsAuto();
  stopPlaySpeech();
  hideAllPages();
```

- [ ] **Step 9: 更新基线测试**

把 `baseline.test.js` 中的：

```js
  await runTest('existing home page renders four cards', async () => {
    assert.strictEqual(await page.locator('.home-card').count(), 4);
```

改为：

```js
  await runTest('existing home page renders five cards', async () => {
    assert.strictEqual(await page.locator('.home-card').count(), 5);
```

- [ ] **Step 10: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
node baseline.test.js
```
Expected:
```
PASS  interactive play data is complete
PASS  home page shows interactive play entry and page tabs
PASS  tpr round has eight unique commands and advances with speech

3 passed, 0 failed
```
以及 baseline 的 4 项全部通过。

- [ ] **Step 11: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add interactive play page and tpr module

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 3: 添加说句子模块

**Files:**
- Modify: `E:/AI-work/english-study/index.html` — 对话函数、`renderPlay()` 分派
- Modify: `E:/AI-work/english-study-e2e/play.test.js` — 对话测试

- [ ] **Step 1: 写失败测试**

在 `play.test.js` 的 `await finish(browser);` 之前插入：

```js
  await runTest('dialogue shows six scenes and substitutes the child sentence', async () => {
    await page.evaluate(() => { switchPlayTab('dialogue'); dialogueIndex = 0; dialogueSubIndex = 0; renderDialogue(); });
    assert.strictEqual(await page.evaluate(() => DIALOGUE_DATA.length), 6);
    assert.ok((await page.locator('.play-progress').innerText()).includes('1 / 6'));

    const first = await page.evaluate(() => {
      const d = DIALOGUE_DATA[0];
      return {
        parent: d.parent.en,
        child: d.child.en.replace('{x}', d.subs[0].en)
      };
    });
    assert.ok((await page.locator('.bubble-child').innerText()).includes(first.child));

    await page.evaluate(() => { window.__spoken.length = 0; });
    await page.locator('.bubble-parent').click();
    await page.waitForTimeout(80);
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), [first.parent]);

    await page.evaluate(() => { window.__spoken.length = 0; });
    await page.locator('.sub-chip').nth(2).click();
    await page.waitForTimeout(80);
    const third = await page.evaluate(() => {
      const d = DIALOGUE_DATA[0];
      return d.child.en.replace('{x}', d.subs[2].en);
    });
    assert.ok((await page.locator('.bubble-child').innerText()).includes(third));
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), [third]);
  });

  await runTest('play dialogue speaks parent then child', async () => {
    await page.evaluate(() => { dialogueIndex = 0; dialogueSubIndex = 0; renderDialogue(); window.__spoken.length = 0; });
    await page.locator('.play-actions .tool-btn').first().click();
    await page.waitForFunction(() => window.__spoken.length === 2);
    const expected = await page.evaluate(() => {
      const d = DIALOGUE_DATA[0];
      return [d.parent.en, d.child.en.replace('{x}', d.subs[0].en)];
    });
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), expected);
  });

  await runTest('dialogue advances to completion and restarts', async () => {
    await page.evaluate(() => { dialogueIndex = 5; dialogueSubIndex = 0; renderDialogue(); });
    assert.ok((await page.locator('.play-progress').innerText()).includes('6 / 6'));
    await page.locator('.play-actions .tool-btn.active').click();
    assert.ok((await page.locator('.play-complete-title').innerText()).includes('完成 6 组对话'));
    await page.locator('.play-complete .big-btn').click();
    assert.strictEqual(await page.evaluate(() => dialogueIndex), 0);
    assert.ok((await page.locator('.play-progress').innerText()).includes('1 / 6'));
  });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
```
Expected: 新增的三个对话测试 FAIL，页面抛出 `renderDialogue is not defined`。

- [ ] **Step 3: 添加对话函数**

在 `nextTpr()` 的结束 `}` 之后、`// ===== QUIZ =====` 之前插入：

```js
function fillDialogue(template, sub) {
  return template.replace(/\{x\}/g, sub.en).replace(/\{xCn\}/g, sub.cn);
}

function renderDialogue() {
  const content = document.getElementById('playContent');
  if (dialogueIndex >= DIALOGUE_DATA.length) {
    content.innerHTML = `
      <div class="play-complete">
        <div class="play-complete-icon">🎉</div>
        <div class="play-complete-title">完成 6 组对话！</div>
        <button class="big-btn" style="background:var(--c4)" onclick="restartDialogue()">🔄 再来一轮</button>
      </div>`;
    return;
  }
  const d = DIALOGUE_DATA[dialogueIndex];
  const sub = d.subs[dialogueSubIndex];
  const childEn = fillDialogue(d.child.en, sub);
  const childCn = fillDialogue(d.child.cn, sub);
  content.innerHTML = `
    <div class="play-progress">${dialogueIndex + 1} / ${DIALOGUE_DATA.length} · ${d.scene}</div>
    <div class="dialogue-row">
      <span class="dialogue-role">🧑 家长</span>
      <button class="bubble bubble-parent" onclick="speakDialogue('parent')">
        ${d.parent.en}<span class="bubble-cn">${d.parent.cn}</span>
      </button>
    </div>
    <div class="dialogue-row">
      <span class="dialogue-role">🧒 孩子</span>
      <button class="bubble bubble-child" onclick="speakDialogue('child')">
        ${childEn}<span class="bubble-cn">${childCn}</span>
      </button>
    </div>
    <div class="sub-chips">
      ${d.subs.map((s, i) =>
        `<button class="sub-chip ${i === dialogueSubIndex ? 'active' : ''}" onclick="selectDialogueSub(${i})">${s.emoji} ${s.en}</button>`
      ).join('')}
    </div>
    <div class="play-actions">
      <button class="tool-btn" onclick="playDialogue()">🔊 播放对话</button>
      <button class="tool-btn active" onclick="nextDialogue()">下一个 →</button>
    </div>`;
}

function speakDialogue(role) {
  const d = DIALOGUE_DATA[dialogueIndex];
  if (!d) return;
  if (role === 'parent') speak(d.parent.en);
  else speak(fillDialogue(d.child.en, d.subs[dialogueSubIndex]));
}

function selectDialogueSub(i) {
  dialogueSubIndex = i;
  renderDialogue();
  speakDialogue('child');
}

function playDialogue() {
  stopPlaySpeech();
  const token = playSpeechToken;
  const d = DIALOGUE_DATA[dialogueIndex];
  if (!d) return;
  speak(d.parent.en, () => {
    if (token !== playSpeechToken) return;
    playTimer = setTimeout(() => {
      if (token !== playSpeechToken) return;
      speak(fillDialogue(d.child.en, d.subs[dialogueSubIndex]));
    }, 250);
  });
}

function nextDialogue() {
  dialogueIndex++;
  dialogueSubIndex = 0;
  renderDialogue();
}

function restartDialogue() {
  dialogueIndex = 0;
  dialogueSubIndex = 0;
  renderDialogue();
}
```

- [ ] **Step 4: 更新 `renderPlay()` 分派**

把：

```js
function renderPlay() {
  if (currentPlayTab === 'tpr') renderTpr();
  else document.getElementById('playContent').innerHTML =
    '<div class="play-progress" style="text-align:center;padding:40px 0;">这个模块马上就来～</div>';
}
```

改为：

```js
function renderPlay() {
  if (currentPlayTab === 'tpr') renderTpr();
  else if (currentPlayTab === 'dialogue') renderDialogue();
  else document.getElementById('playContent').innerHTML =
    '<div class="play-progress" style="text-align:center;padding:40px 0;">这个模块马上就来～</div>';
}
```

- [ ] **Step 5: 添加对话样式**

在 Task 2 添加的 `/* ===== Interactive Play ===== */` 样式块末尾、`/* ===== hidden util ===== */` 之前追加：

```css
.dialogue-row { display: flex; align-items: flex-start; gap: 10px; margin-bottom: 14px; }
.dialogue-role { font-size: var(--font-sm); font-weight: 700; white-space: nowrap; padding-top: 14px; }
.bubble {
  flex: 1;
  text-align: left;
  border: 3px solid transparent;
  border-radius: 22px;
  padding: 16px 20px;
  font-size: 26px;
  font-weight: 700;
  cursor: pointer;
  box-shadow: var(--shadow);
  transition: transform 0.15s, border-color 0.2s;
  color: var(--text);
}
.bubble:hover { transform: scale(1.02); }
.bubble:active { transform: scale(0.97); }
.bubble-parent { background: #EAF4FF; border-color: #BFDDFF; }
.bubble-child { background: #FFF4E6; border-color: #FFD8A8; }
.bubble-cn {
  display: block;
  font-size: var(--font-sm);
  font-weight: 500;
  color: var(--text-light);
  margin-top: 4px;
}
.sub-chips { display: flex; gap: 10px; flex-wrap: wrap; margin: 4px 0 18px; }
.sub-chip {
  padding: 12px 18px;
  border-radius: 50px;
  border: 2.5px solid #E0D7C6;
  background: var(--card-bg);
  font-size: var(--font-sm);
  font-weight: 600;
  cursor: pointer;
}
.sub-chip.active { background: var(--c6); color: #fff; border-color: var(--c6); }
```

- [ ] **Step 6: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
node baseline.test.js
```
Expected: play 6 项全部通过，baseline 4 项全部通过。

- [ ] **Step 7: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add dialogue practice module

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 4: 添加找一找模块

**Files:**
- Modify: `E:/AI-work/english-study/index.html` — I Spy 函数、`renderPlay()` / `switchPlayTab()`
- Modify: `E:/AI-work/english-study-e2e/play.test.js` — I Spy 测试

- [ ] **Step 1: 写失败测试**

在 `play.test.js` 的 `await finish(browser);` 之前插入：

```js
  await runTest('ispy shows six colors and accepts the correct tile', async () => {
    await page.evaluate(() => { switchPlayTab('ispy'); window.__spoken.length = 0; startIspyRound(); });
    assert.strictEqual(await page.locator('.ispy-tile').count(), 6);

    const info = await page.evaluate(() => ({
      type: ispyPromptType,
      targetEn: ispyTarget.en,
      targetColor: ispyTarget.color,
      colors: ispyTiles.map(t => t.color),
      tiles: ispyTiles.map(t => ({ en: t.en, color: t.color }))
    }));
    assert.strictEqual(new Set(info.colors).size, 6);

    const correctIdx = info.type === 'object'
      ? info.tiles.findIndex(t => t.en === info.targetEn)
      : info.tiles.findIndex(t => t.color === info.targetColor);
    assert.ok(correctIdx >= 0);

    await page.locator('.ispy-tile').nth(correctIdx).click();
    assert.strictEqual(await page.locator('.ispy-tile.correct').count(), 1);
    await page.waitForFunction(() => ispyRound === 1);
    assert.strictEqual(await page.locator('.ispy-tile').count(), 6);
  });

  await runTest('ispy wrong tile stays on the same round and replays the prompt', async () => {
    await page.evaluate(() => { startIspyRound(); window.__spoken.length = 0; });
    const info = await page.evaluate(() => ({
      type: ispyPromptType,
      targetEn: ispyTarget.en,
      targetColor: ispyTarget.color,
      tiles: ispyTiles.map(t => ({ en: t.en, color: t.color }))
    }));
    const wrongIdx = info.type === 'object'
      ? info.tiles.findIndex(t => t.en !== info.targetEn)
      : info.tiles.findIndex(t => t.color !== info.targetColor);
    assert.ok(wrongIdx >= 0);

    await page.locator('.ispy-tile').nth(wrongIdx).click();
    await page.waitForTimeout(80);
    assert.strictEqual(await page.evaluate(() => ispyRound), 0);
    const expectedPrompt = info.type === 'object'
      ? `Find the ${info.targetEn}.`
      : `Find something ${info.targetColor}.`;
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), [expectedPrompt]);
  });

  await runTest('ispy completes after five rounds and restarts', async () => {
    await page.evaluate(() => { ispyRound = 5; ispyStars = 5; renderIspy(); });
    assert.ok((await page.locator('.play-complete-title').innerText()).includes('太棒了'));
    await page.locator('.play-complete .big-btn').click();
    assert.strictEqual(await page.evaluate(() => ispyRound), 0);
    assert.strictEqual(await page.locator('.ispy-tile').count(), 6);
  });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
```
Expected: 三个 I Spy 测试 FAIL，页面抛出 `startIspyRound is not defined`。

- [ ] **Step 3: 添加 I Spy 函数**

在 `restartDialogue()` 的结束 `}` 之后、`// ===== QUIZ =====` 之前插入：

```js
function buildIspyRound() {
  const pool = shuffleArray(ISPY_DATA);
  const byColor = {};
  pool.forEach(item => {
    if (!byColor[item.color]) byColor[item.color] = item;
  });
  const colors = Object.keys(byColor);
  ispyTiles = colors.slice(0, 6).map(c => byColor[c]);
  ispyTarget = ispyTiles[Math.floor(Math.random() * ispyTiles.length)];
  ispyPromptType = Math.random() < 0.5 ? 'object' : 'color';
}

function startIspyRound() {
  ispyRound = 0;
  ispyStars = 0;
  ispyTiles = [];
  ispyLocked = false;
  nextIspyRound();
}

function nextIspyRound() {
  buildIspyRound();
  renderIspy();
  speakIspyPrompt();
}

function renderIspy() {
  const content = document.getElementById('playContent');
  if (ispyRound >= 5) {
    content.innerHTML = `
      <div class="play-complete">
        <div class="play-complete-icon">🏆</div>
        <div class="play-complete-title">太棒了！</div>
        <div class="play-progress">⭐ ${ispyStars} / 5</div>
        <button class="big-btn" style="background:var(--c4)" onclick="startIspyRound()">🔄 再玩一次</button>
      </div>`;
    return;
  }
  if (!ispyTiles.length) buildIspyRound();
  const promptEn = ispyPromptType === 'object'
    ? `Find the ${ispyTarget.en}.`
    : `Find something ${ispyTarget.color}.`;
  const promptCn = ispyPromptType === 'object'
    ? `找${ispyTarget.cn}`
    : `找${ispyTarget.colorCn}的东西`;
  content.innerHTML = `
    <div class="play-progress">${ispyRound + 1} / 5 · ⭐ ${ispyStars}</div>
    <div class="ispy-prompt">
      <button class="quiz-speaker-btn" onclick="speakIspyPrompt()">🔊</button>
      <div class="ispy-prompt-text">${promptEn}</div>
      <div class="play-cn">${promptCn}</div>
    </div>
    <div class="ispy-grid">
      ${ispyTiles.map((item, i) =>
        `<button class="ispy-tile" id="ispy-${i}" onclick="answerIspy(${i})">${item.emoji}</button>`
      ).join('')}
    </div>`;
}

function speakIspyPrompt() {
  if (!ispyTarget || ispyRound >= 5) return;
  if (ispyPromptType === 'object') speak(`Find the ${ispyTarget.en}.`);
  else speak(`Find something ${ispyTarget.color}.`);
}

function answerIspy(i) {
  if (ispyLocked) return;
  const item = ispyTiles[i];
  if (!item) return;
  const correct = ispyPromptType === 'object'
    ? item.en === ispyTarget.en
    : item.color === ispyTarget.color;
  const el = document.getElementById('ispy-' + i);
  if (!correct) {
    if (el) {
      el.classList.add('wrong');
      setTimeout(() => el.classList.remove('wrong'), 400);
    }
    speakIspyPrompt();
    return;
  }
  if (el) el.classList.add('correct');
  ispyStars++;
  ispyLocked = true;
  const token = playSpeechToken;
  playTimer = setTimeout(() => {
    if (token !== playSpeechToken) return;
    ispyLocked = false;
    ispyRound++;
    if (ispyRound >= 5) renderIspy();
    else nextIspyRound();
  }, 700);
}
```

- [ ] **Step 4: 更新 `renderPlay()` 分派**

把：

```js
function renderPlay() {
  if (currentPlayTab === 'tpr') renderTpr();
  else if (currentPlayTab === 'dialogue') renderDialogue();
  else document.getElementById('playContent').innerHTML =
    '<div class="play-progress" style="text-align:center;padding:40px 0;">这个模块马上就来～</div>';
}
```

改为：

```js
function renderPlay() {
  if (currentPlayTab === 'tpr') renderTpr();
  else if (currentPlayTab === 'dialogue') renderDialogue();
  else renderIspy();
}
```

- [ ] **Step 5: 更新 `switchPlayTab()`**

把：

```js
function switchPlayTab(tab) {
  stopPlaySpeech();
  currentPlayTab = tab;
  renderPlayTabs();
  renderPlay();
}
```

改为：

```js
function switchPlayTab(tab) {
  stopPlaySpeech();
  currentPlayTab = tab;
  renderPlayTabs();
  renderPlay();
  if (tab === 'ispy') speakIspyPrompt();
}
```

- [ ] **Step 6: 添加 I Spy 样式**

在对话样式之后、`/* ===== hidden util ===== */` 之前追加：

```css
.ispy-prompt { text-align: center; margin-bottom: 18px; }
.ispy-prompt .quiz-speaker-btn { margin-bottom: 10px; }
.ispy-prompt-text { font-size: 30px; font-weight: 800; }
.ispy-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
}
.ispy-tile {
  aspect-ratio: 1;
  border-radius: var(--radius-sm);
  border: 3px solid transparent;
  background: var(--card-bg);
  font-size: 56px;
  cursor: pointer;
  box-shadow: var(--shadow);
  transition: transform 0.15s, border-color 0.2s, background 0.2s;
}
.ispy-tile:hover { transform: scale(1.04); }
.ispy-tile:active { transform: scale(0.94); }
.ispy-tile.correct { border-color: var(--c4); background: #EBFBEE; }
.ispy-tile.wrong { animation: shake 0.4s ease; border-color: var(--c1); }
```

说明：`@keyframes shake` 已存在于现有 CSS 中，直接复用，不重复定义。

- [ ] **Step 7: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
node baseline.test.js
```
Expected: play 9 项全部通过，baseline 4 项全部通过。

- [ ] **Step 8: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add i spy game

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 5: 清理验证与回归检查

**Files:**
- Modify: `E:/AI-work/english-study-e2e/play.test.js` — 离开页面清理测试
- Modify: `E:/AI-work/english-study-e2e/regression.test.js` — 互动乐园返回首页

- [ ] **Step 1: 添加清理测试**

在 `play.test.js` 的 `await finish(browser);` 之前插入：

```js
  await runTest('leaving during dialogue playback cancels the child line', async () => {
    await page.evaluate(() => {
      showPlay();
      switchPlayTab('dialogue');
      dialogueIndex = 0;
      dialogueSubIndex = 0;
      renderDialogue();
      window.__spoken.length = 0;
    });
    const parent = await page.evaluate(() => DIALOGUE_DATA[0].parent.en);
    await page.locator('.play-actions .tool-btn').first().click();
    await page.waitForTimeout(100);
    await page.evaluate(() => goHome());
    await page.waitForTimeout(500);
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), [parent]);
  });

  await runTest('leaving during ispy feedback cancels the auto-next', async () => {
    await page.evaluate(() => {
      showPlay();
      switchPlayTab('ispy');
      startIspyRound();
    });
    const info = await page.evaluate(() => ({
      type: ispyPromptType,
      targetEn: ispyTarget.en,
      targetColor: ispyTarget.color,
      tiles: ispyTiles.map(t => ({ en: t.en, color: t.color }))
    }));
    const correctIdx = info.type === 'object'
      ? info.tiles.findIndex(t => t.en === info.targetEn)
      : info.tiles.findIndex(t => t.color === info.targetColor);
    await page.locator('.ispy-tile').nth(correctIdx).click();
    await page.evaluate(() => goHome());
    await page.waitForTimeout(900);
    assert.strictEqual(await page.evaluate(() => ispyRound), 0);
  });
```

- [ ] **Step 2: 运行清理测试**

Run:
```bash
cd /e/AI-work/english-study-e2e
node play.test.js
```
Expected: 11 项全部通过。若失败，检查 `goHome()` 是否调用 `stopPlaySpeech()`，以及 `playDialogue` / `answerIspy` 中的 token 判断；修复后重跑。

- [ ] **Step 3: 扩展回归测试**

在 `regression.test.js` 的 `await finish(browser);` 之前插入：

```js
  await runTest('navigation returns home from interactive play', async () => {
    await page.evaluate(() => showPlay());
    await page.evaluate(() => goHome());
    assert.strictEqual(
      await page.locator('#homePage').evaluate(el => el.classList.contains('active')),
      true
    );
    assert.strictEqual(
      await page.locator('#navBar').evaluate(el => el.classList.contains('hidden')),
      true
    );
  });
```

- [ ] **Step 4: 运行全部测试**

Run:
```bash
cd /e/AI-work/english-study-e2e
node baseline.test.js
node data.test.js
node render.test.js
node audio.test.js
node autoplay.test.js
node quiz.test.js
node play.test.js
node regression.test.js
```
Expected: 每个文件都输出 `N passed, 0 failed`（baseline 4、data 1、render 3、audio 4、autoplay 2、quiz 4、play 11、regression 4）。

- [ ] **Step 5: 浏览器手动验证**

用浏览器打开 `E:/AI-work/english-study/index.html`，逐项检查：

1. 首页 5 个入口，互动乐园可进入、可返回
2. 三个标签可切换，默认「听指令」
3. 听指令：8 条不重复；点击「再听一遍」朗读；「下一条」推进；完成后「再来一轮」
4. 说句子：6 组可切换；点气泡朗读；替换词改变句子；「播放对话」先家长后孩子
5. 找一找：6 张图卡颜色互不相同；答对高亮并进入下一轮；答错抖动并重读；5 轮后完成
6. 手机宽度（约 375px）下按钮和图卡不溢出、可正常点击
7. 原有字母、单词、音标、练习模块无回归

无法自动验证的部分：真实语音的音质、250ms/700ms 间隔是否自然，需要人工试听。

- [ ] **Step 6: 查看最终差异**

Run:
```bash
cd /e/AI-work/english-study
git diff --stat cb5d837..HEAD -- index.html
git status --short
```
Expected: 只有 `index.html` 被修改；`openspec/` 仍是唯一未跟踪项（保持原状）；测试文件不出现在仓库中。

- [ ] **Step 7: 最终提交（仅在手动验证后有微调时）**

如果手动验证发现问题并修改了 `index.html`：

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "fix: polish interactive play after manual verification

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

如果没有改动，跳过本步骤。

---

## 自检结果

- **规格覆盖**：首页入口、三个标签、TPR 24 条/每轮 8 条、对话 6 组×3 替换词、I Spy 12 物品/6 颜色/5 轮、语音清理、测试与回归均有对应任务。
- **占位符**：无 TBD/TODO；所有代码步骤均给出完整内容。
- **命名一致性**：`TPR_DATA`、`DIALOGUE_DATA`、`ISPY_DATA`、`currentPlayTab`、`playSpeechToken`、`playTimer`、`showPlay`、`switchPlayTab`、`renderPlay`、`renderTpr`、`renderDialogue`、`renderIspy`、`stopPlaySpeech`、`startTprRound`、`playDialogue`、`startIspyRound`、`speakIspyPrompt`、`answerIspy` 在各任务中命名一致。
