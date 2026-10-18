# 趣味反馈增强实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为现有英语学习应用的全部模块增加统一的音效、粒子庆祝动画和 🐱 小猫吉祥物反馈，并提供会话内静音开关。

**Architecture:** 在唯一的 `index.html` 内新增一个自包含的反馈引擎：Web Audio 合成音效、`#fxLayer` 全屏粒子层、导航栏吉祥物。三个模块通过少量调用接入三档反馈（点击 / 答对答错 / 完成），不改变任何既有学习内容、判定逻辑或计时。测试位于仓库外的 `E:/AI-work/english-study-e2e/`，使用 Playwright 与假的 `AudioContext` 统计音效触发。

**Tech Stack:** 原生 HTML/CSS/JS、Web Audio API、DOM 动画、Playwright Core + 本地 Chrome headless。

---

## 文件结构

- Modify: `E:/AI-work/english-study/index.html` — 唯一应用文件；新增约 60 行 CSS、3 处 HTML、约 150 行 JS，并在 6 个既有函数中增加调用
- Modify: `E:/AI-work/english-study-e2e/harness.js` — 测试基建；增加假 `AudioContext`，记录振荡器频率到 `window.__audioStarts`（仓库外，不提交）
- Create: `E:/AI-work/english-study-e2e/feedback.test.js` — 反馈行为测试（仓库外，不提交）

不要修改任何既有测试文件的断言。不要触碰未跟踪的 `openspec/`。

---

## Task 1: 反馈引擎与静音开关

**Files:**
- Modify: `E:/AI-work/english-study/index.html`（`</style>` 前、`#app` 内、`#navBar` 内、`// ===== NAVIGATION =====` 前）
- Modify: `E:/AI-work/english-study-e2e/harness.js`
- Create: `E:/AI-work/english-study-e2e/feedback.test.js`

- [ ] **Step 1: 在测试基建中加入假 AudioContext**

打开 `E:/AI-work/english-study-e2e/harness.js`。在 `page.addInitScript(() => { ... })` 内部，`Object.defineProperty(window, 'SpeechSynthesisUtterance', ...)` 这一行之后、`  });` 之前插入：

```js
    window.__audioStarts = [];
    class FakeOscillator {
      constructor() {
        this.type = 'sine';
        this.frequency = {
          value: 0,
          setValueAtTime(v) { this.value = v; }
        };
      }
      connect() { return this; }
      start() {
        window.__audioStarts.push({ freq: this.frequency.value, type: this.type });
      }
      stop() {}
    }
    class FakeGain {
      constructor() {
        this.gain = {
          value: 0,
          setValueAtTime() {},
          exponentialRampToValueAtTime() {},
          linearRampToValueAtTime() {}
        };
      }
      connect() { return this; }
    }
    class FakeAudioContext {
      constructor() {
        this.state = 'running';
        this.currentTime = 0;
        this.destination = {};
      }
      resume() { this.state = 'running'; }
      createOscillator() { return new FakeOscillator(); }
      createGain() { return new FakeGain(); }
    }
    Object.defineProperty(window, 'AudioContext', { value: FakeAudioContext, configurable: true });
```

- [ ] **Step 2: 编写失败测试**

创建 `E:/AI-work/english-study-e2e/feedback.test.js`：

```js
const { assert, openApp, runTest, finish } = require('./harness');

(async () => {
  const { browser, page } = await openApp();

  await runTest('sound is on by default and mute toggle stops sfx', async () => {
    await page.evaluate(() => showAlphabet());
    assert.strictEqual(await page.evaluate(() => soundOn), true);
    assert.strictEqual(await page.locator('#soundToggle').textContent(), '🔊');

    await page.evaluate(() => { window.__audioStarts.length = 0; });
    await page.locator('#soundToggle').click();
    assert.strictEqual(await page.evaluate(() => soundOn), false);
    assert.strictEqual(await page.locator('#soundToggle').textContent(), '🔇');

    await page.evaluate(() => playSfx('correct'));
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), 0);

    await page.locator('#soundToggle').click();
    assert.strictEqual(await page.evaluate(() => soundOn), true);
    assert.strictEqual(await page.locator('#soundToggle').textContent(), '🔊');
  });

  await runTest('correct sfx plays three ascending notes', async () => {
    await page.evaluate(() => {
      window.__audioStarts.length = 0;
      playSfx('correct');
    });
    assert.deepStrictEqual(
      await page.evaluate(() => window.__audioStarts.map(a => a.freq)),
      [523, 659, 784]
    );
  });

  await runTest('wrong sfx plays two descending notes', async () => {
    await page.evaluate(() => {
      window.__audioStarts.length = 0;
      playSfx('wrong');
    });
    assert.deepStrictEqual(
      await page.evaluate(() => window.__audioStarts.map(a => a.freq)),
      [330, 262]
    );
  });

  await runTest('complete sfx plays four notes', async () => {
    await page.evaluate(() => {
      window.__audioStarts.length = 0;
      playSfx('complete');
    });
    assert.deepStrictEqual(
      await page.evaluate(() => window.__audioStarts.map(a => a.freq)),
      [523, 659, 784, 1047]
    );
  });

  await runTest('fxBurst adds particles that clean themselves up', async () => {
    await page.evaluate(() => fxBurst(200, 200, 10));
    assert.strictEqual(await page.locator('#fxLayer .fx-particle').count(), 10);
    await page.waitForTimeout(900);
    assert.strictEqual(await page.locator('#fxLayer .fx-particle').count(), 0);
  });

  await runTest('mascot cheer sets and clears its state', async () => {
    await page.evaluate(() => fxMascot('cheer'));
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('cheer')),
      true
    );
    await page.waitForTimeout(800);
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('cheer')),
      false
    );
  });

  await runTest('fxCelebrate plays fanfare and spawns confetti with dancer', async () => {
    await page.evaluate(() => {
      window.__audioStarts.length = 0;
      fxCelebrate();
    });
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), 4);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 24);
    assert.strictEqual(await page.locator('#fxLayer .fx-mascot').count(), 1);
    await page.waitForTimeout(2500);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 0);
    assert.strictEqual(await page.locator('#fxLayer .fx-mascot').count(), 0);
  });

  await runTest('fxClear empties the layer and resets the mascot', async () => {
    await page.evaluate(() => {
      fxBurst(100, 100, 5);
      fxConfetti(5);
      fxMascot('cheer');
      fxClear();
    });
    assert.strictEqual(await page.locator('#fxLayer .fx-particle').count(), 0);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 0);
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('cheer')),
      false
    );
  });

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

- [ ] **Step 3: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node feedback.test.js
```

Expected: 8 项全部 FAIL，错误包含 `soundOn is not defined` 或 `#soundToggle` 不存在。

- [ ] **Step 4: 添加 CSS**

在 `index.html` 的 `</style>` 之前插入：

```css
/* ===== Playful Feedback ===== */
#fxLayer {
  position: fixed;
  inset: 0;
  pointer-events: none;
  overflow: hidden;
  z-index: 100;
}
.nav-actions {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: 8px;
}
.nav-sound {
  width: 56px;
  height: 56px;
  border-radius: 50%;
  border: none;
  background: var(--card-bg);
  font-size: 24px;
  cursor: pointer;
  box-shadow: var(--shadow);
  transition: transform 0.15s, box-shadow 0.15s;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.nav-sound:hover { transform: scale(1.08); box-shadow: var(--shadow-hover); }
.nav-sound:active { transform: scale(0.95); }
.nav-mascot {
  font-size: 32px;
  line-height: 1;
  display: inline-block;
  transform-origin: 50% 90%;
}
.nav-mascot.peek { animation: mascotPeek 0.4s ease; }
.nav-mascot.cheer { animation: mascotCheer 0.7s ease; }
.nav-mascot.oops { animation: mascotOops 0.6s ease; }
.fx-particle {
  position: fixed;
  font-size: 22px;
  line-height: 1;
  animation: fxBurst 0.65s ease-out forwards;
}
.fx-confetti {
  position: fixed;
  top: -20px;
  width: 8px;
  height: 14px;
  border-radius: 2px;
  animation: fxFall 1.8s linear forwards;
}
.fx-mascot {
  position: fixed;
  left: 50%;
  top: 50%;
  font-size: 96px;
  line-height: 1;
  transform: translate(-50%, -50%);
  animation: fxDance 1.8s ease-in-out forwards;
}
@keyframes fxBurst {
  to { transform: translate(var(--dx), var(--dy)) rotate(var(--rot)); opacity: 0; }
}
@keyframes fxFall {
  to { transform: translateY(110vh) rotate(720deg); opacity: 0; }
}
@keyframes fxDance {
  0%, 100% { transform: translate(-50%, -50%) scale(1) rotate(0); opacity: 0; }
  10% { opacity: 1; }
  25% { transform: translate(-50%, -50%) scale(1.15) rotate(-10deg); }
  50% { transform: translate(-50%, -50%) scale(1.1) rotate(10deg); }
  75% { transform: translate(-50%, -50%) scale(1.15) rotate(-8deg); }
  90% { opacity: 1; }
}
@keyframes mascotPeek {
  50% { transform: translateY(-6px) scale(1.12); }
}
@keyframes mascotCheer {
  30% { transform: translateY(-12px) rotate(-8deg); }
  60% { transform: translateY(-4px) rotate(8deg); }
}
@keyframes mascotOops {
  25% { transform: rotate(-8deg); }
  75% { transform: rotate(8deg); }
}
```

- [ ] **Step 5: 添加 HTML**

在 `<div id="app">` 之后、`<!-- Nav Bar (hidden on home) -->` 之前插入：

```html
  <div id="fxLayer" aria-hidden="true"></div>
```

在 `<span id="navTitle" class="nav-title"></span>` 之后、`</div>` 之前插入：

```html
    <div class="nav-actions">
      <button id="soundToggle" class="nav-sound" onclick="toggleSound()" aria-label="关闭音效">🔊</button>
      <span id="mascot" class="nav-mascot" aria-hidden="true">🐱</span>
    </div>
```

- [ ] **Step 6: 添加反馈引擎 JS**

在 `// ===== NAVIGATION =====` 之前插入：

```js
// ===== PLAYFUL FEEDBACK =====
let soundOn = true;
let audioCtx = null;
let mascotTimer = null;

function toggleSound() {
  soundOn = !soundOn;
  const btn = document.getElementById('soundToggle');
  if (!btn) return;
  btn.textContent = soundOn ? '🔊' : '🔇';
  btn.setAttribute('aria-label', soundOn ? '关闭音效' : '开启音效');
}

function getAudioCtx() {
  if (!audioCtx) {
    const Ctx = window.AudioContext || window.webkitAudioContext;
    if (!Ctx) return null;
    try { audioCtx = new Ctx(); } catch (e) { return null; }
  }
  if (audioCtx.state === 'suspended') audioCtx.resume();
  return audioCtx;
}

function tone(freq, startOffset, duration, type = 'sine', volume = 0.1) {
  const ctx = getAudioCtx();
  if (!ctx) return;
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();
  const start = ctx.currentTime + startOffset;
  osc.type = type;
  osc.frequency.setValueAtTime(freq, start);
  gain.gain.setValueAtTime(0.0001, start);
  gain.gain.exponentialRampToValueAtTime(volume, start + 0.01);
  gain.gain.exponentialRampToValueAtTime(0.0001, start + duration);
  osc.connect(gain);
  gain.connect(ctx.destination);
  osc.start(start);
  osc.stop(start + duration + 0.02);
}

function playSfx(kind) {
  if (!soundOn) return;
  if (kind === 'tap') {
    tone(660, 0, 0.08, 'sine', 0.06);
  } else if (kind === 'correct') {
    [523, 659, 784].forEach((f, i) => tone(f, i * 0.08, 0.18, 'sine', 0.1));
  } else if (kind === 'wrong') {
    tone(330, 0, 0.12, 'triangle', 0.08);
    tone(262, 0.14, 0.18, 'triangle', 0.08);
  } else if (kind === 'complete') {
    [523, 659, 784, 1047].forEach((f, i) => tone(f, i * 0.12, 0.24, 'sine', 0.1));
  }
}

function fxBurstFrom(el, count) {
  if (!el) return;
  const rect = el.getBoundingClientRect();
  fxBurst(rect.left + rect.width / 2, rect.top + rect.height / 2, count);
}

function fxBurst(x, y, count) {
  const layer = document.getElementById('fxLayer');
  if (!layer) return;
  const emojis = ['⭐', '✨', '💛', '🌟'];
  for (let i = 0; i < count; i++) {
    const p = document.createElement('span');
    p.className = 'fx-particle';
    p.textContent = emojis[i % emojis.length];
    p.style.left = x + 'px';
    p.style.top = y + 'px';
    const angle = Math.random() * Math.PI * 2;
    const dist = 40 + Math.random() * 70;
    p.style.setProperty('--dx', Math.cos(angle) * dist + 'px');
    p.style.setProperty('--dy', Math.sin(angle) * dist + 'px');
    p.style.setProperty('--rot', (Math.random() * 360 - 180) + 'deg');
    layer.appendChild(p);
    setTimeout(() => p.remove(), 800);
  }
}

function fxConfetti(count) {
  const layer = document.getElementById('fxLayer');
  if (!layer) return;
  const colors = ['#FF6B6B', '#FFD43B', '#4DABF7', '#51CF66', '#CC5DE8', '#FF922B'];
  for (let i = 0; i < count; i++) {
    const c = document.createElement('span');
    c.className = 'fx-confetti';
    c.style.left = Math.random() * 100 + '%';
    c.style.background = colors[i % colors.length];
    c.style.animationDelay = (Math.random() * 0.4) + 's';
    layer.appendChild(c);
    setTimeout(() => c.remove(), 2400);
  }
}

function fxMascot(state) {
  const el = document.getElementById('mascot');
  if (!el) return;
  clearTimeout(mascotTimer);
  el.classList.remove('peek', 'cheer', 'oops');
  void el.offsetWidth;
  el.classList.add(state);
  const durations = { peek: 400, cheer: 700, oops: 600 };
  mascotTimer = setTimeout(() => el.classList.remove(state), durations[state] || 500);
}

function fxMascotDance() {
  const layer = document.getElementById('fxLayer');
  if (!layer) return;
  const el = document.createElement('div');
  el.className = 'fx-mascot';
  el.textContent = '🐱';
  layer.appendChild(el);
  setTimeout(() => el.remove(), 1800);
}

function fxCelebrate() {
  playSfx('complete');
  fxConfetti(24);
  fxMascotDance();
}

function fxClear() {
  const layer = document.getElementById('fxLayer');
  if (layer) layer.innerHTML = '';
  clearTimeout(mascotTimer);
  const mascot = document.getElementById('mascot');
  if (mascot) mascot.classList.remove('peek', 'cheer', 'oops');
}
```

- [ ] **Step 7: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node feedback.test.js
```

Expected: `8 passed, 0 failed`。

- [ ] **Step 8: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add playful feedback engine

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 2: 全局点击轻反馈

**Files:**
- Modify: `E:/AI-work/english-study/index.html`（反馈引擎之后）
- Modify: `E:/AI-work/english-study-e2e/feedback.test.js`

- [ ] **Step 1: 追加失败测试**

在 `feedback.test.js` 的 `await finish(browser);` 之前插入：

```js
  await runTest('tapping a letter card plays a tap and makes the mascot peek', async () => {
    await page.evaluate(() => {
      showAlphabet();
      window.__audioStarts.length = 0;
      fxClear();
    });
    await page.locator('.letter-card').first().click();
    assert.deepStrictEqual(
      await page.evaluate(() => window.__audioStarts.map(a => a.freq)),
      [660]
    );
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('peek')),
      true
    );
  });

  await runTest('tapping a tab plays a tap without the mascot peek', async () => {
    await page.evaluate(() => {
      showWords();
      window.__audioStarts.length = 0;
      fxClear();
    });
    await page.locator('#catTabs .cat-tab').first().click();
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), 1);
    assert.strictEqual(await page.evaluate(() => window.__audioStarts[0].freq), 660);
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('peek')),
      false
    );
  });

  await runTest('tapping a home card plays a tap sound', async () => {
    await page.evaluate(() => {
      goHome();
      window.__audioStarts.length = 0;
      fxClear();
    });
    await page.locator('.home-card').first().click();
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), 1);
    assert.strictEqual(await page.evaluate(() => window.__audioStarts[0].freq), 660);
  });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node feedback.test.js
```

Expected: 前 8 项通过，新增 3 项失败（`__audioStarts` 长度为 0，小猫没有 `peek`）。

- [ ] **Step 3: 添加全局委托监听**

在 `index.html` 的 `fxClear()` 函数之后、`// ===== NAVIGATION =====` 之前插入：

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
}, true);
```

说明：使用捕获阶段（第三个参数 `true`），确保音标例词的 `event.stopPropagation()` 不会吞掉点击音。

- [ ] **Step 4: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node feedback.test.js
```

Expected: `11 passed, 0 failed`。

- [ ] **Step 5: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add tap feedback across modules

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 3: 答对与答错反馈

**Files:**
- Modify: `E:/AI-work/english-study/index.html`（`answerQuiz`、`answerIspy`）
- Modify: `E:/AI-work/english-study-e2e/feedback.test.js`

- [ ] **Step 1: 追加失败测试**

在 `feedback.test.js` 的 `await finish(browser);` 之前插入：

```js
  await runTest('correct quiz answer triggers chime, particles and cheer', async () => {
    await page.evaluate(() => {
      showQuizSetup();
      setDifficulty('normal');
      startQuiz('letter');
      window.__audioStarts.length = 0;
      fxClear();
    });
    const correctIdx = await page.evaluate(() =>
      quizQuestions[quizIndex].options.findIndex(o => o === quizQuestions[quizIndex].correct)
    );
    await page.locator('.quiz-option').nth(correctIdx).click();
    assert.deepStrictEqual(
      await page.evaluate(() => window.__audioStarts.map(a => a.freq)),
      [523, 659, 784]
    );
    assert.strictEqual(await page.locator('#fxLayer .fx-particle').count(), 10);
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('cheer')),
      true
    );
  });

  await runTest('wrong quiz answer triggers gentle tone and mascot oops', async () => {
    await page.evaluate(() => {
      showQuizSetup();
      setDifficulty('normal');
      startQuiz('letter');
      window.__audioStarts.length = 0;
      fxClear();
    });
    const wrongIdx = await page.evaluate(() =>
      quizQuestions[quizIndex].options.findIndex(o => o !== quizQuestions[quizIndex].correct)
    );
    await page.locator('.quiz-option').nth(wrongIdx).click();
    assert.deepStrictEqual(
      await page.evaluate(() => window.__audioStarts.map(a => a.freq)),
      [330, 262]
    );
    assert.strictEqual(await page.locator('#fxLayer .fx-particle').count(), 0);
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('oops')),
      true
    );
  });

  await runTest('correct ispy tile triggers chime, particles and cheer', async () => {
    await page.evaluate(() => {
      showPlay();
      switchPlayTab('ispy');
      startIspyRound();
      window.__audioStarts.length = 0;
      fxClear();
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
    assert.deepStrictEqual(
      await page.evaluate(() => window.__audioStarts.map(a => a.freq)),
      [523, 659, 784]
    );
    assert.strictEqual(await page.locator('#fxLayer .fx-particle').count(), 10);
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('cheer')),
      true
    );
  });

  await runTest('wrong ispy tile triggers gentle tone and mascot oops', async () => {
    await page.evaluate(() => {
      showPlay();
      switchPlayTab('ispy');
      startIspyRound();
      window.__audioStarts.length = 0;
      fxClear();
    });
    const info = await page.evaluate(() => ({
      type: ispyPromptType,
      targetEn: ispyTarget.en,
      targetColor: ispyTarget.color,
      tiles: ispyTiles.map(t => ({ en: t.en, color: t.color }))
    }));
    const wrongIdx = info.type === 'object'
      ? info.tiles.findIndex(t => t.en !== info.targetEn)
      : info.tiles.findIndex(t => t.color !== info.targetColor);
    await page.locator('.ispy-tile').nth(wrongIdx).click();
    assert.deepStrictEqual(
      await page.evaluate(() => window.__audioStarts.map(a => a.freq)),
      [330, 262]
    );
    assert.strictEqual(await page.locator('#fxLayer .fx-particle').count(), 0);
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('oops')),
      true
    );
  });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node feedback.test.js
```

Expected: 前 11 项通过，新增 4 项失败（答对/答错没有音效与粒子，小猫没有反应）。

- [ ] **Step 3: 修改 `answerQuiz`**

将 `index.html` 中 `answerQuiz` 的 `if (isCorrect) { ... } else { ... }` 替换为：

```js
  if (isCorrect) {
    quizScore++;
    document.getElementById('quizScoreDisplay').textContent = quizScore;
    const el = document.getElementById('qopt-' + idx);
    el.classList.add('correct');
    playSfx('correct');
    fxBurstFrom(el, 10);
    fxMascot('cheer');
    setTimeout(() => {
      quizIndex++;
      renderQuizQuestion();
    }, 900);
  } else {
    const el = document.getElementById('qopt-' + idx);
    el.classList.add('wrong');
    playSfx('wrong');
    fxMascot('oops');
    setTimeout(() => {
      speakQuizQuestion(q);
      document.querySelectorAll('.quiz-option').forEach(b => b.disabled = false);
      document.getElementById('qopt-' + idx).classList.remove('wrong');
    }, 600);
  }
```

- [ ] **Step 4: 修改 `answerIspy` 的答错与答对分支**

将 `answerIspy` 中：

```js
  if (!correct) {
    if (el) {
      el.classList.add('wrong');
      setTimeout(() => el.classList.remove('wrong'), 400);
    }
    speakIspyPrompt();
    return;
  }
  if (el) el.classList.add('correct');
  ispyLocked = true;
```

替换为：

```js
  if (!correct) {
    if (el) {
      el.classList.add('wrong');
      setTimeout(() => el.classList.remove('wrong'), 400);
    }
    playSfx('wrong');
    fxMascot('oops');
    speakIspyPrompt();
    return;
  }
  if (el) el.classList.add('correct');
  playSfx('correct');
  fxBurstFrom(el, 10);
  fxMascot('cheer');
  ispyLocked = true;
```

- [ ] **Step 5: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node feedback.test.js
```

Expected: `15 passed, 0 failed`。

- [ ] **Step 6: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add correct and wrong feedback

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 4: 完成庆祝与离开清理

**Files:**
- Modify: `E:/AI-work/english-study/index.html`（`showQuizComplete`、`nextTpr`、`nextDialogue`、`answerIspy` 计时回调、`goHome`）
- Modify: `E:/AI-work/english-study-e2e/feedback.test.js`

- [ ] **Step 1: 追加失败测试**

在 `feedback.test.js` 的 `await finish(browser);` 之前插入：

```js
  await runTest('completing the quiz triggers a celebration', async () => {
    await page.evaluate(() => {
      showQuizSetup();
      quizQuestions = [{ type: 'letter', correct: 'A', options: ['A', 'B', 'C', 'D'] }];
      quizScore = 1;
      quizIndex = 1;
      window.__audioStarts.length = 0;
      fxClear();
      showQuizComplete();
    });
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), 4);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 24);
    assert.strictEqual(await page.locator('#fxLayer .fx-mascot').count(), 1);
  });

  await runTest('finishing tpr celebrates once and not on tab re-entry', async () => {
    await page.evaluate(() => {
      showPlay();
      startTprRound();
      tprIndex = tprQueue.length - 1;
      window.__audioStarts.length = 0;
      fxClear();
      nextTpr();
    });
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), 4);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 24);
    const afterFirst = await page.evaluate(() => window.__audioStarts.length);
    await page.evaluate(() => {
      switchPlayTab('dialogue');
      switchPlayTab('tpr');
    });
    await page.waitForTimeout(100);
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), afterFirst);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 24);
  });

  await runTest('finishing dialogue celebrates once and not on tab re-entry', async () => {
    await page.evaluate(() => {
      showPlay();
      switchPlayTab('dialogue');
      dialogueIndex = DIALOGUE_DATA.length - 1;
      dialogueSubIndex = 0;
      window.__audioStarts.length = 0;
      fxClear();
      nextDialogue();
    });
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), 4);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 24);
    const afterFirst = await page.evaluate(() => window.__audioStarts.length);
    await page.evaluate(() => {
      switchPlayTab('tpr');
      switchPlayTab('dialogue');
    });
    await page.waitForTimeout(100);
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), afterFirst);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 24);
  });

  await runTest('fifth ispy answer triggers a celebration', async () => {
    await page.evaluate(() => {
      showPlay();
      switchPlayTab('ispy');
      startIspyRound();
      ispyRound = 4;
      ispyStars = 4;
      renderIspy();
      window.__audioStarts.length = 0;
      fxClear();
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
    await page.waitForTimeout(800);
    assert.strictEqual(await page.evaluate(() => ispyRound), 5);
    assert.strictEqual(await page.evaluate(() => window.__audioStarts.length), 7);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 24);
  });

  await runTest('leaving the page clears feedback effects', async () => {
    await page.evaluate(() => {
      fxBurst(100, 100, 5);
      fxConfetti(5);
      fxMascot('cheer');
      goHome();
    });
    assert.strictEqual(await page.locator('#fxLayer .fx-particle').count(), 0);
    assert.strictEqual(await page.locator('#fxLayer .fx-confetti').count(), 0);
    assert.strictEqual(
      await page.locator('#mascot').evaluate(el => el.classList.contains('cheer')),
      false
    );
  });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node feedback.test.js
```

Expected: 前 15 项通过，新增 5 项失败（没有彩带/大猫，`goHome` 不清空特效）。

- [ ] **Step 3: 修改 `showQuizComplete`**

在函数末尾 `document.getElementById('completeStars').textContent = stars;` 之后添加：

```js
  fxCelebrate();
```

- [ ] **Step 4: 修改 `nextTpr`**

将 `nextTpr` 整体替换为：

```js
function nextTpr() {
  if (tprIndex >= tprQueue.length) return;
  tprIndex++;
  renderTpr();
  if (tprIndex >= tprQueue.length) {
    fxCelebrate();
    return;
  }
  speakTpr();
}
```

- [ ] **Step 5: 修改 `nextDialogue`**

将 `nextDialogue` 整体替换为：

```js
function nextDialogue() {
  stopPlaySpeech();
  dialogueIndex++;
  dialogueSubIndex = 0;
  renderDialogue();
  if (dialogueIndex >= DIALOGUE_DATA.length) fxCelebrate();
}
```

- [ ] **Step 6: 修改 `answerIspy` 的计时回调**

将：

```js
    ispyStars++;
    ispyLocked = false;
    ispyRound++;
    if (ispyRound >= 5) renderIspy();
    else nextIspyRound();
```

替换为：

```js
    ispyStars++;
    ispyLocked = false;
    ispyRound++;
    if (ispyRound >= 5) {
      renderIspy();
      fxCelebrate();
    } else nextIspyRound();
```

- [ ] **Step 7: 修改 `goHome`**

在 `goHome` 的 `stopPlaySpeech();` 之后添加：

```js
  fxClear();
```

- [ ] **Step 8: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node feedback.test.js
```

Expected: `20 passed, 0 failed`。

- [ ] **Step 9: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add completion celebrations

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 5: 全量回归与手动验证

**Files:**
- Verify only: `E:/AI-work/english-study/index.html`
- 测试文件保持仓库外，不提交

- [ ] **Step 1: 运行全部 9 个测试文件**

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
node feedback.test.js
```

Expected: 每个文件 `0 failed`。数量：baseline 4、data 1、render 3、audio 4、autoplay 2、quiz 4、play 17、regression 4、feedback 20，共 **59 passed**。

- [ ] **Step 2: 浏览器程序化验证**

用 Playwright 临时脚本（放 `E:/AI-work/english-study-e2e/`，不提交）验证：

1. 导航栏在 375px 下：`#navBar` 不换行（`scrollWidth <= clientWidth`），`#soundToggle` 为 56×56，`#mascot` 可见
2. 点击 🔊/🔇：图标切换，`soundOn` 翻转；静音后答对无音效但星星与小猫动画照常
3. 学习页点字母/单词/音标卡片：一个小猫 `peek`，不产生粒子
4. 趣味练习答对：星星粒子出现，小猫 `cheer`；答错：小猫 `oops`，选项抖动
5. 完成趣味练习、TPR 8 条、对话 6 组、I Spy 5 轮：各有一次彩带 + 中央 🐱；切换标签返回不重复
6. 原有模块：字母 26、单词 8、音标元音 11/辅音 14、练习 4 选项、互动乐园三标签正常
7. 页面加载与所有点击均无 console error

真实设备上的音效音量、音色是否悦耳需要人工试听，报告中列为「需人工确认」。

- [ ] **Step 3: 最终差异检查**

Run:
```bash
cd /e/AI-work/english-study
git diff --stat 23d0c41..HEAD -- index.html
git status --short
```

Expected: 只有 `index.html` 被修改；工作区只有 `?? openspec/`；测试文件不出现在仓库中。

- [ ] **Step 4: 提交（仅当验证后有微调）**

如果 Step 2 发现问题并修改了 `index.html`：

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "fix: polish playful feedback after verification

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

没有改动则跳过，不要提交任何测试文件。

---

## 自检结果

- **规格覆盖**：全部模块轻反馈、答对/答错中反馈、四类完成重反馈、静音开关、Web Audio 音效、粒子/彩带、吉祥物状态、`goHome` 清理、375px 验证均有对应任务。
- **占位符**：无 TBD/TODO；所有代码步骤均为完整可粘贴内容。
- **命名一致性**：`soundOn`、`audioCtx`、`mascotTimer`、`toggleSound`、`getAudioCtx`、`tone`、`playSfx`、`fxBurstFrom`、`fxBurst`、`fxConfetti`、`fxMascot`、`fxMascotDance`、`fxCelebrate`、`fxClear`、`TAP_SELECTOR` 在各任务中一致。
- **测试数量**：8 + 3 + 4 + 5 = 20 项反馈测试；与既有 39 项合计 59 项。
