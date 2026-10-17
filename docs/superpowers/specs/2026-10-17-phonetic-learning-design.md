# 音标学习模块设计

日期：2026-10-17
状态：已与用户确认

## 背景

现有 `index.html` 是一个面向 3-6 岁幼儿的纯前端英语启蒙单页应用，包含字母学习、单词学习、听音选择练习三个模块。项目约定为单文件 HTML（内联 CSS/JS）、零构建、零网络依赖、使用 Web Speech API 发音。

本次在原有应用上新增音标学习，帮助 4-5 岁英语初学者建立「音标符号 → 声音 → 例词」的联系。用户已确认：采用美式音标符号、每个音标 3 个例词、新增独立页面并在练习模块中加入音标练习。

## 目标与非目标

**目标**

- 首页新增「音标乐园」入口，与现有三个入口并列
- 展示 25 个美式核心音标：11 个元音 + 14 个辅音
- 每个音标展示符号、代表 emoji、3 个例词
- 点击音标卡片，先播放 TTS 近似音，再播放代表例词
- 例词可单独点击听读
- 支持「逐个播放 / 停止」自动朗读
- 趣味练习新增「音标练习」模式：听单词，从 2 或 4 个音标中选出正确的一个
- 保持现有视觉风格、交互反馈和单文件架构

**非目标**

- 不实现完整的 48 个国际音标
- 不实现发音评分、录音或语音识别
- 不实现学习进度持久化
- 不新增外部音频、图片或第三方库
- 不改动现有字母、单词、练习题型的既有行为

## 音标数据（美式）

元音（11 个）：

| 音标 | TTS 近似拼写 | 例词 |
|---|---|---|
| /i/ | ee | bee 🐝、green 💚、tree 🌳 |
| /ɪ/ | ih | fish 🐟、pig 🐷、sit 🪑 |
| /ɛ/ | eh | red ❤️、egg 🥚、bed 🛏️ |
| /æ/ | ah | cat 🐱、apple 🍎、hat 🎩 |
| /ɑ/ | ahh | star ⭐、car 🚗、father 👨 |
| /ʌ/ | uh | duck 🦆、cup 🥤、sun ☀️ |
| /ɔ/ | aw | dog 🐶、ball ⚽、horse 🐴 |
| /ʊ/ | uu | book 📖、cook 👨‍🍳、foot 🦶 |
| /u/ | oo | moon 🌙、blue 💙、zoo 🦒 |
| /ɜr/ | er | bird 🐦、purple 💜、girl 👧 |
| /eɪ/ | ay | cake 🎂、game 🎮、name 🏷️ |

辅音（14 个）：

| 音标 | TTS 近似拼写 | 例词 |
|---|---|---|
| /b/ | buh | ball ⚽、bear 🐻、blue 💙 |
| /p/ | puh | pig 🐷、peach 🍑、pink 💗 |
| /m/ | mmm | moon 🌙、mango 🥭、monkey 🐒 |
| /f/ | fff | fish 🐟、five ✋、frog 🐸 |
| /d/ | duh | dog 🐶、duck 🦆、red ❤️ |
| /t/ | tuh | tiger 🐯、ten 🔟、cat 🐱 |
| /n/ | nnn | nose 👃、nine 9️⃣、sun ☀️ |
| /l/ | lll | lion 🦁、leg 🦵、yellow 💛 |
| /g/ | guh | goat 🐐、green 💚、grape 🍇 |
| /k/ | kuh | cat 🐱、kite 🪁、duck 🦆 |
| /h/ | huh | hat 🎩、horse 🐴、hello 👋 |
| /s/ | sss | sun ☀️、star ⭐、six 6️⃣ |
| /w/ | wuh | water 💧、watermelon 🍉、window 🪟 |
| /r/ | ruh | rabbit 🐰、red ❤️、rainbow 🌈 |

说明：

- 美式音标采用 /ɛ/、/ɑ/、/ɔ/、/ɜr/ 等写法，不采用英式的 /e/、/ɒ/、/ɜː/
- `/æ/` 与 `/ʊ/` 没有稳定的英文近似拼写，`say` 字段以近似值兜底，实际区分主要依靠例词；`say` 值可在数据中单独微调
- 例词优先选用现有单词库中的高频启蒙词，避免生僻词
- 每个例词额外记录完整美式音标 `ipa`（如 `red` → `/rɛd/`），仅用于练习出题的干扰项过滤，不在界面展示
- 语音近似拼写仅用于「听个大概」，真正的音值区分依赖例词

## 数据结构

```js
const PHONICS_DATA = {
  vowels: {
    label: '元音',
    items: [
      {
        ipa: '/i/',
        say: 'ee',
        examples: [
          { en: 'bee', cn: '蜜蜂', emoji: '🐝', ipa: '/bi/' },
          { en: 'green', cn: '绿色', emoji: '💚', ipa: '/ɡrin/' },
          { en: 'tree', cn: '树', emoji: '🌳', ipa: '/tri/' }
        ]
      }
      // ...
    ]
  },
  consonants: {
    label: '辅音',
    items: [ /* 同上 */ ]
  }
};
```

代表 emoji 取 `examples[0].emoji`，不额外存储，避免冗余。

## 页面结构

首页 `home-cards` 新增第 4 张卡片：

- 标签：音标乐园
- 描述：认识元音和辅音，听音学发音
- 图标：🎵
- 配色：沿用现有渐变色体系，使用未占用的珊瑚红/橙色组合

新增页面 `#phonicsPage`：

```html
<div id="phonicsPage" class="page">
  <div class="category-tabs" id="phonicsTabs"></div>
  <div class="alphabet-toolbar">
    <button id="autoPlayPhonicsBtn" class="tool-btn" onclick="autoPlayPhonics()">▶ 逐个播放</button>
    <button id="stopPhonicsBtn" class="tool-btn hidden" onclick="stopPhonicsAuto()">■ 停止</button>
  </div>
  <div id="phonicsGrid" class="phonics-grid"></div>
</div>
```

每个音标卡片：

```html
<div class="phonics-card" id="phonics-0" onclick="speakPhonics(0)">
  <span class="phonics-ipa">/i/</span>
  <span class="phonics-emoji">🐝</span>
  <span class="phonics-word">bee</span>
  <div class="phonics-examples">
    <button onclick="event.stopPropagation(); speakExample(0,1)">green</button>
    <!-- ... -->
  </div>
</div>
```

- `phonics-ipa`：大号符号，使用等宽或字体支持良好的字体栈
- `phonics-emoji` + `phonics-word`：代表例词
- `phonics-examples`：3 个例词小标签，可单独点击
- 卡片点击区域大于 64×64px，符合幼儿操作习惯

样式复用现有变量 `--c1` 至 `--c7`、`--radius-sm`、`--shadow`，与 `letter-card` / `word-card` 保持一致的悬停、缩放、高亮、抖动反馈。

## 交互与状态

新增状态：

```js
let currentPhonicsGroup = 'vowels'; // 'vowels' | 'consonants'
let phonicsAutoTimer = null;
```

- `showPhonics()`：停止其它自动播放，切换到音标页，渲染标签与卡片
- `switchPhonicsGroup(key)`：切换元音/辅音，停止自动播放，重新渲染
- `renderPhonics()`：按当前分组渲染卡片和标签
- `speakPhonics(idx)`：先停止自动播放，再播放「近似音 → 代表单词」两段式发音，并高亮卡片
- `speakExample(groupKey, idx, exampleIdx)`：单独播放某个例词
- `autoPlayPhonics()`：从当前分组第一个音标开始逐个朗读，读完自动停止
- `stopPhonicsAuto()`：清除定时器、取消语音、恢复按钮状态、清除高亮

两段式发音使用 `speak(text, callback)` 的 `onend` 链式调用：

```js
speak(item.say, () => {
  setTimeout(() => speak(item.examples[0].en), 250);
});
```

不使用并行 `speak`，避免 `synth.cancel()` 打断前一段。切换分组、离开页面、点击停止时统一调用 `stopPhonicsAuto()`。

## 练习模式

练习设置页新增第三个模式卡片：

- 标签：音标练习
- 图标：🎵
- `startQuiz('phonics')`

出题逻辑：

- 从元音和辅音中随机抽取一个音标作为正确答案
- 从该音标的 3 个例词中随机选一个作为朗读词
- 干扰项从其它音标中随机抽取，简单模式共 2 个选项，普通模式共 4 个
- **干扰项过滤**：去除音标符号（`/` 后）后，若该符号出现在朗读词的 `ipa` 中，则不能作为干扰项。例如朗读 `red` `/rɛd/` 时，/r/、/ɛ/、/d/ 都不能出现在选项里，确保只有一个正确答案
- 题目对象：`{ type: 'phonics', correct: item, spokenWord: example, options: [...] }`

渲染：

- 题面继续显示「听一听，选一选！」
- 喇叭按钮播放 `spokenWord.en`
- 选项显示 `音标 + 代表 emoji`，例如 `/b/ ⚽`
- 判题按 `ipa` 比较
- 答对：加分、绿色高亮、庆祝动画，自动进入下一题
- 答错：抖动、重新播放单词、允许再次作答
- 结算沿用现有星星规则

`generateQuestions(mode)` 增加 `phonics` 分支；`speakQuizQuestion(q)`、`answerQuiz(idx)` 增加类型判断。现有 `letter` / `word` 分支保持不变。

## 语音方案

- 复用现有 `speak()`：`lang = 'en-US'`、`rate = 0.85`、`pitch = 1.1`
- 近似音使用数据中的 `say` 字段，例词使用原词
- 每次播放前由现有 `speak()` 调用 `synth.cancel()`，保证不重叠
- 保留现有首次点击解锁语音的 prime 逻辑
- 无新增网络请求或音频资源

## 改动范围

仅修改 `index.html`：

1. `<style>` 新增音标页与卡片样式（约 80 行）
2. 首页新增第 4 张入口卡片
3. 新增 `#phonicsPage` 页面结构
4. 练习设置页新增音标模式卡片
5. `<script>` 新增 `PHONICS_DATA`、状态、渲染、播放、自动播放函数
6. `<script>` 扩展 `generateQuestions`、`speakQuizQuestion`、`answerQuiz` 的音标分支
7. `goHome()` 增加一行 `stopPhonicsAuto()`，与现有两个自动播放的清理方式一致；`hideAllPages()` 无需修改

不新增文件、不引入依赖、不改动 `pinyin.html`。

## 错误处理与降级

- 若浏览器无可用英文语音，`speak()` 静默失败，页面文字和 emoji 仍可正常学习
- 若 `voices` 尚未加载，沿用现有延迟重载逻辑
- 快速连续点击时，`synth.cancel()` 保证只播放最后一次
- 切换分组或返回首页时清理定时器与高亮，避免残留

## 验证标准

- 首页显示 4 个入口，音标入口可进入、可返回
- 元音标签渲染 11 张卡片，辅音标签渲染 14 张卡片
- 每张卡片显示正确音标、代表 emoji、3 个例词
- 点击卡片听到「近似音 + 代表单词」，无重叠
- 点击例词小标签只播放该例词，不触发卡片两段式播放
- 逐个播放可正常读完并自动停止；中途停止无残留高亮和声音
- 音标练习简单/普通难度各能完整完成 10 题并结算
- 原有字母学习、单词学习、字母/单词练习功能无回归
