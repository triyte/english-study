# 音标学习模块实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在现有单文件英语启蒙应用中新增 25 个美式音标的学习页面和音标练习模式。

**Architecture:** 全部改动集中在 `index.html`。新增 `PHONICS_DATA` 数据、音标页面 DOM/CSS、发音与自动播放函数，并扩展现有练习模块的出题与判题分支。语音继续复用 Web Speech API；测试使用仓库外的 Playwright 脚本，通过假语音实现捕获朗读内容。

**Tech Stack:** 原生 HTML/CSS/JavaScript、Web Speech API、Playwright Core（仅测试，仓库外）、Chrome headless。

**Spec:** `docs/superpowers/specs/2026-10-17-phonetic-learning-design.md`

---

## 文件结构

- 修改：`E:/AI-work/english-study/index.html` — 唯一交付文件，包含全部样式、结构、数据和逻辑
- 创建（仓库外）：`E:/AI-work/english-study-e2e/harness.js` — Playwright 测试公共工具
- 创建（仓库外）：`E:/AI-work/english-study-e2e/*.test.js` — 分模块验证脚本
- 不修改：`pinyin.html`、`openspec/`、`.gitignore`

测试目录不提交到仓库，避免给零构建项目引入 Node 依赖。

---

## Task 1: 搭建测试脚手架并验证基线

**Files:**
- Create: `E:/AI-work/english-study-e2e/harness.js`
- Create: `E:/AI-work/english-study-e2e/baseline.test.js`

- [ ] **Step 1: 安装 Playwright Core**

Run:
```bash
mkdir -p /e/AI-work/english-study-e2e
cd /e/AI-work/english-study-e2e
npm init -y
npm i playwright-core@1.64.1 --no-audit --no-fund
```
Expected: `added 1 package`，目录下出现 `node_modules/playwright-core`。

- [ ] **Step 2: 写测试公共工具**

创建 `E:/AI-work/english-study-e2e/harness.js`：

```js
const assert = require('node:assert');
const { chromium } = require('playwright-core');

const APP_URL = 'file:///E:/AI-work/english-study/index.html';
const CHROME = 'C:/Program Files/Google/Chrome/Application/chrome.exe';

let passed = 0;
let failed = 0;

async function openApp() {
  const browser = await chromium.launch({ executablePath: CHROME, headless: true });
  const page = await browser.newPage();
  await page.addInitScript(() => {
    window.__spoken = [];
    class FakeUtterance {
      constructor(text) {
        this.text = text;
        this.onend = null;
        this.onerror = null;
      }
    }
    const fakeSynth = {
      speak(u) {
        if (u.text) window.__spoken.push(u.text);
        setTimeout(() => { if (u.onend) u.onend(); }, 30);
      },
      cancel() {},
      pause() {},
      resume() {},
      getVoices() { return [{ lang: 'en-US', name: 'Test Voice' }]; },
      onvoiceschanged: null
    };
    Object.defineProperty(window, 'speechSynthesis', { value: fakeSynth, configurable: true });
    Object.defineProperty(window, 'SpeechSynthesisUtterance', { value: FakeUtterance, configurable: true });
  });
  await page.goto(APP_URL);
  return { browser, page };
}

async function runTest(name, fn) {
  try {
    await fn();
    passed++;
    console.log(`PASS  ${name}`);
  } catch (err) {
    failed++;
    console.error(`FAIL  ${name}`);
    console.error(err && err.stack ? err.stack : err);
  }
}

async function finish(browser) {
  await browser.close();
  console.log(`\n${passed} passed, ${failed} failed`);
  if (failed > 0) process.exit(1);
}

module.exports = { assert, openApp, runTest, finish, APP_URL };
```

- [ ] **Step 3: 写基线测试**

创建 `E:/AI-work/english-study-e2e/baseline.test.js`：

```js
const { assert, openApp, runTest, finish } = require('./harness');

(async () => {
  const { browser, page } = await openApp();

  await runTest('existing home page renders three cards', async () => {
    assert.strictEqual(await page.locator('.home-card').count(), 3);
    assert.strictEqual(
      await page.locator('#homePage').evaluate(el => el.classList.contains('active')),
      true
    );
  });

  await runTest('existing alphabet page renders 26 letter cards', async () => {
    await page.evaluate(() => showAlphabet());
    assert.strictEqual(await page.locator('.letter-card').count(), 26);
  });

  await runTest('existing word page renders 8 animal cards', async () => {
    await page.evaluate(() => showWords());
    assert.strictEqual(await page.locator('.word-card').count(), 8);
  });

  await runTest('speech stub captures letter pronunciation', async () => {
    await page.evaluate(() => showAlphabet());
    await page.evaluate(() => { window.__spoken.length = 0; });
    await page.locator('.letter-card').first().click();
    await page.waitForTimeout(100);
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), ['A']);
  });

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

- [ ] **Step 4: 运行基线测试**

Run:
```bash
cd /e/AI-work/english-study-e2e
node baseline.test.js
```
Expected:
```
PASS  existing home page renders three cards
PASS  existing alphabet page renders 26 letter cards
PASS  existing word page renders 8 animal cards
PASS  speech stub captures letter pronunciation

4 passed, 0 failed
```

- [ ] **Step 5: 不提交**

测试文件在仓库外，本任务不产生仓库改动，跳过 commit。

---

## Task 2: 添加音标数据

**Files:**
- Modify: `E:/AI-work/english-study/index.html` — 在 `const WORD_DATA = {...};` 之后、`// ===== STATE =====` 之前插入数据
- Create: `E:/AI-work/english-study-e2e/data.test.js`

- [ ] **Step 1: 写失败测试**

创建 `E:/AI-work/english-study-e2e/data.test.js`：

```js
const { assert, openApp, runTest, finish } = require('./harness');

(async () => {
  const { browser, page } = await openApp();

  await runTest('phonics data has 11 vowels and 14 consonants with 3 examples each', async () => {
    const info = await page.evaluate(() => {
      if (typeof PHONICS_DATA === 'undefined') return null;
      const all = [...PHONICS_DATA.vowels.items, ...PHONICS_DATA.consonants.items];
      return {
        vowels: PHONICS_DATA.vowels.items.length,
        consonants: PHONICS_DATA.consonants.items.length,
        bad: all.filter(item =>
          !item.ipa || !item.say || item.examples.length !== 3 ||
          item.examples.some(ex => !ex.en || !ex.cn || !ex.emoji || !ex.ipa)
        ).length
      };
    });
    assert.ok(info, 'PHONICS_DATA is not defined');
    assert.strictEqual(info.vowels, 11);
    assert.strictEqual(info.consonants, 14);
    assert.strictEqual(info.bad, 0);
  });

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node data.test.js
```
Expected:
```
FAIL  phonics data has 11 vowels and 14 consonants with 3 examples each
AssertionError ... PHONICS_DATA is not defined

0 passed, 1 failed
```

- [ ] **Step 3: 写入完整音标数据**

在 `index.html` 的 `const WORD_DATA = {...};` 结束后插入：

```js
const PHONICS_DATA = {
  vowels: {
    label: '元音',
    items: [
      { ipa: '/i/', say: 'ee', examples: [
        { en: 'bee', cn: '蜜蜂', emoji: '🐝', ipa: '/bi/' },
        { en: 'green', cn: '绿色', emoji: '💚', ipa: '/grin/' },
        { en: 'tree', cn: '树', emoji: '🌳', ipa: '/tri/' }
      ]},
      { ipa: '/ɪ/', say: 'ih', examples: [
        { en: 'fish', cn: '鱼', emoji: '🐟', ipa: '/fɪʃ/' },
        { en: 'pig', cn: '猪', emoji: '🐷', ipa: '/pɪg/' },
        { en: 'sit', cn: '坐', emoji: '🪑', ipa: '/sɪt/' }
      ]},
      { ipa: '/ɛ/', say: 'eh', examples: [
        { en: 'red', cn: '红色', emoji: '❤️', ipa: '/rɛd/' },
        { en: 'egg', cn: '鸡蛋', emoji: '🥚', ipa: '/ɛg/' },
        { en: 'bed', cn: '床', emoji: '🛏️', ipa: '/bɛd/' }
      ]},
      { ipa: '/æ/', say: 'ah', examples: [
        { en: 'cat', cn: '猫', emoji: '🐱', ipa: '/kæt/' },
        { en: 'apple', cn: '苹果', emoji: '🍎', ipa: '/ˈæpəl/' },
        { en: 'hat', cn: '帽子', emoji: '🎩', ipa: '/hæt/' }
      ]},
      { ipa: '/ɑ/', say: 'ahh', examples: [
        { en: 'star', cn: '星星', emoji: '⭐', ipa: '/stɑr/' },
        { en: 'car', cn: '汽车', emoji: '🚗', ipa: '/kɑr/' },
        { en: 'father', cn: '爸爸', emoji: '👨', ipa: '/ˈfɑðər/' }
      ]},
      { ipa: '/ʌ/', say: 'uh', examples: [
        { en: 'duck', cn: '鸭子', emoji: '🦆', ipa: '/dʌk/' },
        { en: 'cup', cn: '杯子', emoji: '🥤', ipa: '/kʌp/' },
        { en: 'sun', cn: '太阳', emoji: '☀️', ipa: '/sʌn/' }
      ]},
      { ipa: '/ɔ/', say: 'aw', examples: [
        { en: 'dog', cn: '狗', emoji: '🐶', ipa: '/dɔg/' },
        { en: 'ball', cn: '球', emoji: '⚽', ipa: '/bɔl/' },
        { en: 'horse', cn: '马', emoji: '🐴', ipa: '/hɔrs/' }
      ]},
      { ipa: '/ʊ/', say: 'uu', examples: [
        { en: 'book', cn: '书', emoji: '📖', ipa: '/bʊk/' },
        { en: 'cook', cn: '厨师', emoji: '👨‍🍳', ipa: '/kʊk/' },
        { en: 'foot', cn: '脚', emoji: '🦶', ipa: '/fʊt/' }
      ]},
      { ipa: '/u/', say: 'oo', examples: [
        { en: 'moon', cn: '月亮', emoji: '🌙', ipa: '/mun/' },
        { en: 'blue', cn: '蓝色', emoji: '💙', ipa: '/blu/' },
        { en: 'zoo', cn: '动物园', emoji: '🦒', ipa: '/zu/' }
      ]},
      { ipa: '/ɜr/', say: 'er', examples: [
        { en: 'bird', cn: '鸟', emoji: '🐦', ipa: '/bɜrd/' },
        { en: 'purple', cn: '紫色', emoji: '💜', ipa: '/ˈpɜrpəl/' },
        { en: 'girl', cn: '女孩', emoji: '👧', ipa: '/gɜrl/' }
      ]},
      { ipa: '/eɪ/', say: 'ay', examples: [
        { en: 'cake', cn: '蛋糕', emoji: '🎂', ipa: '/keɪk/' },
        { en: 'game', cn: '游戏', emoji: '🎮', ipa: '/geɪm/' },
        { en: 'name', cn: '名字', emoji: '🏷️', ipa: '/neɪm/' }
      ]}
    ]
  },
  consonants: {
    label: '辅音',
    items: [
      { ipa: '/b/', say: 'buh', examples: [
        { en: 'ball', cn: '球', emoji: '⚽', ipa: '/bɔl/' },
        { en: 'bear', cn: '熊', emoji: '🐻', ipa: '/bɛr/' },
        { en: 'blue', cn: '蓝色', emoji: '💙', ipa: '/blu/' }
      ]},
      { ipa: '/p/', say: 'puh', examples: [
        { en: 'pig', cn: '猪', emoji: '🐷', ipa: '/pɪg/' },
        { en: 'peach', cn: '桃子', emoji: '🍑', ipa: '/pitʃ/' },
        { en: 'pink', cn: '粉色', emoji: '💗', ipa: '/pɪŋk/' }
      ]},
      { ipa: '/m/', say: 'mmm', examples: [
        { en: 'moon', cn: '月亮', emoji: '🌙', ipa: '/mun/' },
        { en: 'mango', cn: '芒果', emoji: '🥭', ipa: '/ˈmæŋgoʊ/' },
        { en: 'monkey', cn: '猴子', emoji: '🐒', ipa: '/ˈmʌŋki/' }
      ]},
      { ipa: '/f/', say: 'fff', examples: [
        { en: 'fish', cn: '鱼', emoji: '🐟', ipa: '/fɪʃ/' },
        { en: 'five', cn: '五', emoji: '✋', ipa: '/faɪv/' },
        { en: 'frog', cn: '青蛙', emoji: '🐸', ipa: '/frɔg/' }
      ]},
      { ipa: '/d/', say: 'duh', examples: [
        { en: 'dog', cn: '狗', emoji: '🐶', ipa: '/dɔg/' },
        { en: 'duck', cn: '鸭子', emoji: '🦆', ipa: '/dʌk/' },
        { en: 'red', cn: '红色', emoji: '❤️', ipa: '/rɛd/' }
      ]},
      { ipa: '/t/', say: 'tuh', examples: [
        { en: 'tiger', cn: '老虎', emoji: '🐯', ipa: '/ˈtaɪgər/' },
        { en: 'ten', cn: '十', emoji: '🔟', ipa: '/tɛn/' },
        { en: 'cat', cn: '猫', emoji: '🐱', ipa: '/kæt/' }
      ]},
      { ipa: '/n/', say: 'nnn', examples: [
        { en: 'nose', cn: '鼻子', emoji: '👃', ipa: '/noʊz/' },
        { en: 'nine', cn: '九', emoji: '9️⃣', ipa: '/naɪn/' },
        { en: 'sun', cn: '太阳', emoji: '☀️', ipa: '/sʌn/' }
      ]},
      { ipa: '/l/', say: 'lll', examples: [
        { en: 'lion', cn: '狮子', emoji: '🦁', ipa: '/ˈlaɪən/' },
        { en: 'leg', cn: '腿', emoji: '🦵', ipa: '/lɛg/' },
        { en: 'yellow', cn: '黄色', emoji: '💛', ipa: '/ˈyɛloʊ/' }
      ]},
      { ipa: '/g/', say: 'guh', examples: [
        { en: 'goat', cn: '山羊', emoji: '🐐', ipa: '/goʊt/' },
        { en: 'green', cn: '绿色', emoji: '💚', ipa: '/grin/' },
        { en: 'grape', cn: '葡萄', emoji: '🍇', ipa: '/greɪp/' }
      ]},
      { ipa: '/k/', say: 'kuh', examples: [
        { en: 'cat', cn: '猫', emoji: '🐱', ipa: '/kæt/' },
        { en: 'kite', cn: '风筝', emoji: '🪁', ipa: '/kaɪt/' },
        { en: 'duck', cn: '鸭子', emoji: '🦆', ipa: '/dʌk/' }
      ]},
      { ipa: '/h/', say: 'huh', examples: [
        { en: 'hat', cn: '帽子', emoji: '🎩', ipa: '/hæt/' },
        { en: 'horse', cn: '马', emoji: '🐴', ipa: '/hɔrs/' },
        { en: 'hello', cn: '你好', emoji: '👋', ipa: '/hɛˈloʊ/' }
      ]},
      { ipa: '/s/', say: 'sss', examples: [
        { en: 'sun', cn: '太阳', emoji: '☀️', ipa: '/sʌn/' },
        { en: 'star', cn: '星星', emoji: '⭐', ipa: '/stɑr/' },
        { en: 'six', cn: '六', emoji: '6️⃣', ipa: '/sɪks/' }
      ]},
      { ipa: '/w/', say: 'wuh', examples: [
        { en: 'water', cn: '水', emoji: '💧', ipa: '/ˈwɔtər/' },
        { en: 'watermelon', cn: '西瓜', emoji: '🍉', ipa: '/ˈwɔtərmɛlən/' },
        { en: 'window', cn: '窗户', emoji: '🪟', ipa: '/ˈwɪndoʊ/' }
      ]},
      { ipa: '/r/', say: 'ruh', examples: [
        { en: 'rabbit', cn: '兔子', emoji: '🐰', ipa: '/ˈræbɪt/' },
        { en: 'red', cn: '红色', emoji: '❤️', ipa: '/rɛd/' },
        { en: 'rainbow', cn: '彩虹', emoji: '🌈', ipa: '/ˈreɪnboʊ/' }
      ]}
    ]
  }
};
```

- [ ] **Step 4: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node data.test.js
```
Expected:
```
PASS  phonics data has 11 vowels and 14 consonants with 3 examples each

1 passed, 0 failed
```

- [ ] **Step 5: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add American phoneme data for phonics module

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 3: 添加音标页面、入口、渲染与点击发音

**Files:**
- Modify: `E:/AI-work/english-study/index.html` — CSS、首页入口、页面结构、状态、页面函数
- Create: `E:/AI-work/english-study-e2e/render.test.js`
- Create: `E:/AI-work/english-study-e2e/audio.test.js`

- [ ] **Step 1: 写失败的渲染与发音测试**

创建 `E:/AI-work/english-study-e2e/render.test.js`：

```js
const { assert, openApp, runTest, finish } = require('./harness');

(async () => {
  const { browser, page } = await openApp();

  await runTest('home page shows the phonics entry', async () => {
    assert.strictEqual(await page.locator('.home-card').count(), 4);
  });

  await runTest('phonics page shows 11 vowel cards', async () => {
    await page.locator('.home-card', { hasText: '音标乐园' }).click();
    assert.strictEqual(
      await page.locator('#phonicsPage').evaluate(el => el.classList.contains('active')),
      true
    );
    assert.deepStrictEqual(await page.locator('#phonicsTabs .cat-tab').allTextContents(), ['元音', '辅音']);
    assert.strictEqual(await page.locator('.phonics-card').count(), 11);
    const first = await page.locator('.phonics-card').first().innerText();
    assert.ok(first.includes('/i/'));
    assert.ok(first.includes('bee'));
    assert.strictEqual(await page.locator('#phonics-0 .phonics-example').count(), 3);
  });

  await runTest('switching to consonants shows 14 cards', async () => {
    await page.locator('#phonicsTabs .cat-tab', { hasText: '辅音' }).click();
    assert.strictEqual(await page.locator('.phonics-card').count(), 14);
    const first = await page.locator('.phonics-card').first().innerText();
    assert.ok(first.includes('/b/'));
    assert.ok(first.includes('ball'));
  });

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

创建 `E:/AI-work/english-study-e2e/audio.test.js`：

```js
const { assert, openApp, runTest, finish } = require('./harness');

(async () => {
  const { browser, page } = await openApp();
  await page.evaluate(() => showPhonics());

  await runTest('clicking a vowel card speaks approximation then example word', async () => {
    await page.evaluate(() => { window.__spoken.length = 0; });
    await page.locator('#phonics-0').click();
    await page.waitForTimeout(500);
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), ['ee', 'bee']);
    assert.strictEqual(
      await page.locator('#phonics-0').evaluate(el => el.classList.contains('highlight')),
      true
    );
  });

  await runTest('clicking an example word speaks only that word', async () => {
    await page.evaluate(() => { window.__spoken.length = 0; });
    await page.locator('#phonics-0 .phonics-example').nth(1).click();
    await page.waitForTimeout(150);
    assert.deepStrictEqual(await page.evaluate(() => window.__spoken.slice()), ['green']);
  });

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node render.test.js
node audio.test.js
```
Expected: 两个文件都失败（首页仍为 3 张卡片、`showPhonics` 未定义）。

- [ ] **Step 3: 添加音标页样式**

在 `index.html` 的 `/* ===== Quiz Page ===== */` 之前插入：

```css
/* ===== Phonics Page ===== */
.phonics-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 14px;
}
.phonics-card {
  background: var(--card-bg);
  border-radius: var(--radius-sm);
  padding: 16px 10px 12px;
  text-align: center;
  cursor: pointer;
  box-shadow: var(--shadow);
  transition: transform 0.12s, box-shadow 0.12s, background 0.2s;
  border: 3px solid transparent;
}
.phonics-card:hover { transform: scale(1.05); box-shadow: var(--shadow-hover); }
.phonics-card:active { transform: scale(0.94); }
.phonics-card.highlight {
  border-color: var(--c6);
  background: #F3F0FF;
  transform: scale(1.05);
  box-shadow: 0 0 20px rgba(177,151,252,0.35);
}
.phonics-ipa {
  display: block;
  font-size: 40px;
  font-weight: 800;
  color: var(--c6);
  font-family: 'Segoe UI', 'Lucida Sans Unicode', 'Arial Unicode MS', sans-serif;
  line-height: 1.1;
}
.phonics-emoji { display: block; font-size: 34px; margin: 4px 0 2px; }
.phonics-word {
  display: block;
  font-size: var(--font-md);
  font-weight: 700;
  color: var(--text);
  margin-bottom: 8px;
}
.phonics-examples {
  display: flex;
  gap: 6px;
  justify-content: center;
  flex-wrap: wrap;
}
.phonics-example {
  padding: 6px 10px;
  border-radius: 50px;
  border: 2px solid #E0D7C6;
  background: #FFFDF7;
  font-size: 14px;
  font-weight: 600;
  color: var(--text-light);
  cursor: pointer;
}
.phonics-example:hover { border-color: var(--c6); color: var(--c6); }
.phonics-example:active { transform: scale(0.95); }
```

- [ ] **Step 4: 添加首页入口卡片**

在首页第三个 `home-card` 按钮之后、`</div>`（`home-cards` 的闭合标签）之前插入：

```html
      <button class="home-card" style="background:linear-gradient(135deg,var(--c7),var(--c1))" onclick="showPhonics()">
        <span class="home-card-icon">&#127925;</span>
        <span class="home-card-text">
          <span class="home-card-label">音标乐园</span>
          <span class="home-card-desc">认识元音和辅音，听音学发音</span>
        </span>
      </button>
```

- [ ] **Step 5: 添加音标页面结构**

在 `<!-- ====== QUIZ PAGE ====== -->` 之前插入：

```html
  <!-- ====== PHONICS PAGE ====== -->
  <div id="phonicsPage" class="page">
    <div class="category-tabs" id="phonicsTabs"></div>
    <div id="phonicsGrid" class="phonics-grid"></div>
  </div>
```

（逐个播放的工具栏在 Task 4 中加入，避免出现点击即报错的中间状态。）

- [ ] **Step 6: 添加状态变量**

在 `let quizScore = 0;` 之后插入：

```js
let currentPhonicsGroup = 'vowels';
let phonicsAutoTimer = null;
let phonicsAutoToken = 0;
```

- [ ] **Step 7: 添加音标页面函数**

在 `// ===== QUIZ =====` 之前插入：

```js
// ===== PHONICS =====
function shuffleArray(arr) {
  const out = arr.slice();
  for (let i = out.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [out[i], out[j]] = [out[j], out[i]];
  }
  return out;
}

function phonicsAllItems() {
  return [...PHONICS_DATA.vowels.items, ...PHONICS_DATA.consonants.items];
}

function wordHasPhoneme(wordIpa, item) {
  return wordIpa.replace(/\//g, '').includes(item.ipa.replace(/\//g, ''));
}

function showPhonics() {
  stopAutoPlay();
  stopWordsAuto();
  stopPhonicsAuto();
  hideAllPages();
  showNav('音标学习');
  document.getElementById('phonicsPage').classList.add('active');
  renderPhonics();
}

function switchPhonicsGroup(key) {
  currentPhonicsGroup = key;
  stopPhonicsAuto();
  renderPhonics();
}

function renderPhonics() {
  const data = PHONICS_DATA[currentPhonicsGroup];
  document.getElementById('phonicsTabs').innerHTML = Object.entries(PHONICS_DATA).map(([key, group]) =>
    `<button class="cat-tab${key === currentPhonicsGroup ? ' active' : ''}" onclick="switchPhonicsGroup('${key}')">${group.label}</button>`
  ).join('');
  document.getElementById('phonicsGrid').innerHTML = data.items.map((item, i) => `
    <div class="phonics-card" id="phonics-${i}" onclick="speakPhonics(${i})">
      <span class="phonics-ipa">${item.ipa}</span>
      <span class="phonics-emoji">${item.examples[0].emoji}</span>
      <span class="phonics-word">${item.examples[0].en}</span>
      <div class="phonics-examples">
        ${item.examples.map((ex, j) =>
          `<button class="phonics-example" onclick="event.stopPropagation(); speakExample(${i},${j})">${ex.en}</button>`
        ).join('')}
      </div>
    </div>
  `).join('');
}

function speakPhonics(idx) {
  stopPhonicsAuto();
  const item = PHONICS_DATA[currentPhonicsGroup].items[idx];
  highlightPhonics(idx);
  speak(item.say, () => setTimeout(() => speak(item.examples[0].en), 250));
}

function speakExample(idx, exampleIdx) {
  stopPhonicsAuto();
  const item = PHONICS_DATA[currentPhonicsGroup].items[idx];
  highlightPhonics(idx);
  speak(item.examples[exampleIdx].en);
}

function highlightPhonics(idx) {
  document.querySelectorAll('.phonics-card').forEach(c => c.classList.remove('highlight'));
  const el = document.getElementById('phonics-' + idx);
  if (el) el.classList.add('highlight');
}

function stopPhonicsAuto() {
  phonicsAutoToken++;
  if (phonicsAutoTimer) { clearTimeout(phonicsAutoTimer); phonicsAutoTimer = null; }
  synth.cancel();
  const playBtn = document.getElementById('autoPlayPhonicsBtn');
  const stopBtn = document.getElementById('stopPhonicsBtn');
  if (playBtn) playBtn.classList.remove('hidden');
  if (stopBtn) stopBtn.classList.add('hidden');
  document.querySelectorAll('.phonics-card').forEach(c => c.classList.remove('highlight'));
}
```

- [ ] **Step 8: 在 `goHome()` 中清理音标自动播放**

把：

```js
function goHome() {
  stopAutoPlay();
  stopWordsAuto();
  hideAllPages();
```

改为：

```js
function goHome() {
  stopAutoPlay();
  stopWordsAuto();
  stopPhonicsAuto();
  hideAllPages();
```

- [ ] **Step 9: 更新基线断言并运行渲染与发音测试**

把 `baseline.test.js` 中的：

```js
    assert.strictEqual(await page.locator('.home-card').count(), 3);
```

改为：

```js
    assert.strictEqual(await page.locator('.home-card').count(), 4);
```

Run:
```bash
cd /e/AI-work/english-study-e2e
node render.test.js
node audio.test.js
```
Expected:
```
PASS  home page shows the phonics entry
PASS  phonics page shows 11 vowel cards
PASS  switching to consonants shows 14 cards

3 passed, 0 failed
```
```
PASS  clicking a vowel card speaks approximation then example word
PASS  clicking an example word speaks only that word

2 passed, 0 failed
```

- [ ] **Step 10: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add phonics learning page with sound cards

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 4: 添加逐个播放

**Files:**
- Modify: `E:/AI-work/english-study/index.html` — 新增 `autoPlayPhonics()`
- Create: `E:/AI-work/english-study-e2e/autoplay.test.js`

- [ ] **Step 1: 写失败测试**

创建 `E:/AI-work/english-study-e2e/autoplay.test.js`：

```js
const { assert, openApp, runTest, finish } = require('./harness');

(async () => {
  const { browser, page } = await openApp();
  await page.evaluate(() => showPhonics());

  await runTest('autoplay starts and stop halts further speech', async () => {
    await page.evaluate(() => { window.__spoken.length = 0; });
    await page.locator('#autoPlayPhonicsBtn').click();
    await page.waitForTimeout(150);
    assert.ok((await page.evaluate(() => window.__spoken.length)) >= 1);
    await page.locator('#stopPhonicsBtn').click();
    const countAtStop = await page.evaluate(() => window.__spoken.length);
    await page.waitForTimeout(700);
    assert.strictEqual(await page.evaluate(() => window.__spoken.length), countAtStop);
    assert.strictEqual(await page.locator('.phonics-card.highlight').count(), 0);
    assert.strictEqual(
      await page.locator('#autoPlayPhonicsBtn').evaluate(el => el.classList.contains('hidden')),
      false
    );
  });

  await runTest('leaving the phonics page stops autoplay', async () => {
    await page.evaluate(() => { window.__spoken.length = 0; });
    await page.locator('#autoPlayPhonicsBtn').click();
    await page.waitForTimeout(100);
    await page.evaluate(() => goHome());
    const countAtHome = await page.evaluate(() => window.__spoken.length);
    await page.waitForTimeout(700);
    assert.strictEqual(await page.evaluate(() => window.__spoken.length), countAtHome);
  });

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node autoplay.test.js
```
Expected: 失败，控制台出现 `autoPlayPhonics is not defined`（点击按钮时）。

- [ ] **Step 3: 添加工具栏并实现逐个播放**

在音标页的 `<div class="category-tabs" id="phonicsTabs"></div>` 之后插入：

```html
    <div class="alphabet-toolbar">
      <button id="autoPlayPhonicsBtn" class="tool-btn" onclick="autoPlayPhonics()">&#9654; 逐个播放</button>
      <button id="stopPhonicsBtn" class="tool-btn hidden" onclick="stopPhonicsAuto()">&#9632; 停止</button>
    </div>
```

在 `stopPhonicsAuto()` 之后插入：

```js
function autoPlayPhonics() {
  stopPhonicsAuto();
  const token = phonicsAutoToken;
  const items = PHONICS_DATA[currentPhonicsGroup].items;
  let i = 0;
  document.getElementById('autoPlayPhonicsBtn').classList.add('hidden');
  document.getElementById('stopPhonicsBtn').classList.remove('hidden');

  function playNext() {
    if (token !== phonicsAutoToken) return;
    if (i >= items.length) { stopPhonicsAuto(); return; }
    highlightPhonics(i);
    const item = items[i];
    speak(item.say, () => {
      if (token !== phonicsAutoToken) return;
      setTimeout(() => {
        if (token !== phonicsAutoToken) return;
        speak(item.examples[0].en, () => {
          if (token !== phonicsAutoToken) return;
          i++;
          phonicsAutoTimer = setTimeout(playNext, 700);
        });
      }, 250);
    });
  }
  playNext();
}
```

- [ ] **Step 4: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node autoplay.test.js
```
Expected:
```
PASS  autoplay starts and stop halts further speech
PASS  leaving the phonics page stops autoplay

2 passed, 0 failed
```

- [ ] **Step 5: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add autoplay for phonics sound cards

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 5: 添加音标练习模式

**Files:**
- Modify: `E:/AI-work/english-study/index.html` — 练习设置卡片、出题、渲染、朗读、判题
- Create: `E:/AI-work/english-study-e2e/quiz.test.js`

- [ ] **Step 1: 写失败测试**

创建 `E:/AI-work/english-study-e2e/quiz.test.js`：

```js
const { assert, openApp, runTest, finish, APP_URL } = require('./harness');

(async () => {
  const { browser, page } = await openApp();

  await runTest('quiz setup shows three modes', async () => {
    await page.evaluate(() => showQuizSetup());
    assert.strictEqual(await page.locator('.quiz-mode-card').count(), 3);
    assert.ok((await page.locator('.quiz-mode-cards').innerText()).includes('音标练习'));
  });

  await runTest('easy phonics quiz shows two options and accepts the correct one', async () => {
    await page.goto(APP_URL);
    await page.evaluate(() => { setDifficulty('easy'); startQuiz('phonics'); });
    assert.strictEqual(await page.locator('.quiz-option').count(), 2);
    const q = await page.evaluate(() => ({
      correctIpa: quizQuestions[0].correct.ipa,
      optionIpas: quizQuestions[0].options.map(o => o.ipa)
    }));
    assert.ok(q.optionIpas.includes(q.correctIpa));
    await page.locator('.quiz-option').nth(q.optionIpas.indexOf(q.correctIpa)).click();
    assert.strictEqual(await page.evaluate(() => quizScore), 1);
  });

  await runTest('normal phonics quiz shows four options', async () => {
    await page.goto(APP_URL);
    await page.evaluate(() => { setDifficulty('normal'); startQuiz('phonics'); });
    assert.strictEqual(await page.locator('.quiz-option').count(), 4);
  });

  await runTest('distractors never occur in the spoken word', async () => {
    await page.goto(APP_URL);
    const bad = await page.evaluate(() => {
      let count = 0;
      for (let n = 0; n < 200; n++) {
        generateQuestions('phonics').forEach(q => {
          const word = q.spokenWord.ipa.replace(/\//g, '');
          q.options.forEach(opt => {
            if (opt === q.correct) return;
            if (word.includes(opt.ipa.replace(/\//g, ''))) count++;
          });
        });
      }
      return count;
    });
    assert.strictEqual(bad, 0);
  });

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

- [ ] **Step 2: 运行测试确认失败**

Run:
```bash
cd /e/AI-work/english-study-e2e
node quiz.test.js
```
Expected: 第一个测试失败（只有 2 个模式卡片），后续测试失败（`generateQuestions('phonics')` 返回空或没有音标题）。

- [ ] **Step 3: 添加练习设置入口**

在 `quiz-mode-cards` 中「单词练习」按钮之后插入：

```html
        <button class="quiz-mode-card" style="background:linear-gradient(135deg,var(--c7),var(--c1))" onclick="startQuiz('phonics')">
          <span class="quiz-mode-icon">&#127925;</span>音标练习
        </button>
```

- [ ] **Step 4: 添加音标出题函数**

在 `stopPhonicsAuto()` 之后、`autoPlayPhonics()` 之前插入：

```js
function generatePhonicsQuestions(total) {
  const all = phonicsAllItems();
  const optionCount = quizDifficulty === 'easy' ? 2 : 4;
  const questions = [];
  for (let i = 0; i < total; i++) {
    const correct = all[Math.floor(Math.random() * all.length)];
    const spokenWord = correct.examples[Math.floor(Math.random() * correct.examples.length)];
    const distractors = shuffleArray(all.filter(item =>
      item !== correct && !wordHasPhoneme(spokenWord.ipa, item)
    )).slice(0, optionCount - 1);
    questions.push({
      type: 'phonics',
      correct,
      spokenWord,
      options: shuffleArray([correct, ...distractors])
    });
  }
  return questions;
}
```

- [ ] **Step 5: 让 `generateQuestions` 支持音标模式**

在 `generateQuestions(mode)` 的 `const total = 10;` 之后插入：

```js
  if (mode === 'phonics') return generatePhonicsQuestions(total);
```

- [ ] **Step 6: 修改选项渲染**

把 `renderQuizQuestion()` 中的：

```js
  optsDiv.innerHTML = q.options.map((opt, i) => {
    const label = q.type === 'letter' ? opt : opt.en;
    const emojiPart = q.type === 'word' ? `<span style="font-size:34px">${opt.emoji}</span>` : '';
    return `<button class="quiz-option" id="qopt-${i}" onclick="answerQuiz(${i})">${emojiPart} ${label}</button>`;
  }).join('');
```

改为：

```js
  optsDiv.innerHTML = q.options.map((opt, i) => {
    const label = q.type === 'letter' ? opt
      : q.type === 'word' ? opt.en
      : opt.ipa;
    const emojiPart = q.type === 'word'
      ? `<span style="font-size:34px">${opt.emoji}</span>`
      : q.type === 'phonics'
        ? `<span style="font-size:34px">${opt.examples[0].emoji}</span>`
        : '';
    return `<button class="quiz-option" id="qopt-${i}" onclick="answerQuiz(${i})">${emojiPart} ${label}</button>`;
  }).join('');
```

- [ ] **Step 7: 修改题目朗读**

把 `speakQuizQuestion(q)` 中的：

```js
  const text = q.type === 'letter' ? q.correct : q.correct.en;
```

改为：

```js
  const text = q.type === 'letter' ? q.correct
    : q.type === 'word' ? q.correct.en
    : q.spokenWord.en;
```

- [ ] **Step 8: 修改判题逻辑**

把 `answerQuiz(idx)` 中的：

```js
  const isCorrect = q.type === 'letter'
    ? selected === q.correct
    : selected.en === q.correct.en;
```

改为：

```js
  const isCorrect = q.type === 'letter'
    ? selected === q.correct
    : q.type === 'word'
      ? selected.en === q.correct.en
      : selected.ipa === q.correct.ipa;
```

- [ ] **Step 9: 运行测试确认通过**

Run:
```bash
cd /e/AI-work/english-study-e2e
node quiz.test.js
```
Expected:
```
PASS  quiz setup shows three modes
PASS  easy phonics quiz shows two options and accepts the correct one
PASS  normal phonics quiz shows four options
PASS  distractors never occur in the spoken word

4 passed, 0 failed
```

- [ ] **Step 10: 提交**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "feat: add phonics quiz mode

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

---

## Task 6: 回归验证与最终检查

**Files:**
- Create: `E:/AI-work/english-study-e2e/regression.test.js`

- [ ] **Step 1: 写回归测试**

创建 `E:/AI-work/english-study-e2e/regression.test.js`：

```js
const { assert, openApp, runTest, finish, APP_URL } = require('./harness');

(async () => {
  const { browser, page } = await openApp();

  await runTest('alphabet and word learning still work', async () => {
    await page.evaluate(() => showAlphabet());
    assert.strictEqual(await page.locator('.letter-card').count(), 26);
    await page.evaluate(() => showWords());
    assert.strictEqual(await page.locator('.word-card').count(), 8);
    await page.evaluate(() => switchCategory('fruits'));
    assert.strictEqual(await page.locator('.word-card').count(), 8);
  });

  await runTest('letter and word quizzes still work', async () => {
    await page.goto(APP_URL);
    await page.evaluate(() => { setDifficulty('normal'); startQuiz('letter'); });
    assert.strictEqual(await page.locator('.quiz-option').count(), 4);
    await page.goto(APP_URL);
    await page.evaluate(() => { setDifficulty('normal'); startQuiz('word'); });
    assert.strictEqual(await page.locator('.quiz-option').count(), 4);
  });

  await runTest('navigation returns home from phonics', async () => {
    await page.evaluate(() => showPhonics());
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

  await finish(browser);
})().catch(err => { console.error(err); process.exit(1); });
```

- [ ] **Step 2: 运行全部测试**

Run:
```bash
cd /e/AI-work/english-study-e2e
node baseline.test.js
node data.test.js
node render.test.js
node audio.test.js
node autoplay.test.js
node quiz.test.js
node regression.test.js
```
Expected: 每个文件都输出 `N passed, 0 failed`。

- [ ] **Step 3: 真机手动验证发音**

用浏览器打开 `E:/AI-work/english-study/index.html`，逐项检查：

1. 首页 4 个入口，音标乐园可进入、可返回
2. 元音 11 张、辅音 14 张卡片，音标符号显示正常（/ɛ/ /æ/ /ɑ/ /ɔ/ /ʊ/ /ʌ/ /ɜr/ 无乱码）
3. 点击卡片先读近似音，再读例词；快速连点不会叠音
4. 点击例词小标签只读该例词
5. 逐个播放能连续朗读并自动停止；中途停止无残留高亮
6. 音标练习简单/普通模式各完成 10 题，答对/答错反馈正常
7. 字母、单词、字母练习、单词练习无回归

- [ ] **Step 4: 查看最终差异**

Run:
```bash
cd /e/AI-work/english-study
git diff --stat HEAD~4
git status --short
```
Expected: 只有 `index.html` 被修改；测试目录和 `openspec/` 不出现在仓库状态中（`openspec/` 原本就是未跟踪目录，保持原状）。

- [ ] **Step 5: 最终提交（如手动验证后有微调）**

```bash
cd /e/AI-work/english-study
git add index.html
git commit -m "fix: polish phonics module after manual verification

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

如果没有改动，跳过本步骤。

---

## 自检结果

- **规格覆盖**：数据、入口、页面、两段式发音、例词点读、逐个播放、练习模式、干扰项过滤、回归验证均有对应任务
- **占位符**：无 TBD/TODO；所有代码步骤均给出完整内容
- **命名一致性**：`PHONICS_DATA`、`currentPhonicsGroup`、`phonicsAutoToken`、`speakPhonics`、`speakExample`、`stopPhonicsAuto`、`generatePhonicsQuestions`、`wordHasPhoneme` 在各任务中命名一致
