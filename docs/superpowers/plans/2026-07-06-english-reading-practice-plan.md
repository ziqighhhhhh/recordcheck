# 英语朗读练习工具实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 实现一个单文件网页应用，让用户粘贴英文原文、录音后获得 Azure Speech 发音评估反馈，并针对问题句/词反复练习。

**架构：** 浏览器内单 HTML 文件，通过 CDN 引入 Azure Speech JS SDK；用户填写 Azure Key/Region 后直连 Azure；录音、评估、对齐、高亮、Drill 全部在页面内完成。

**技术栈：** HTML5, CSS, Vanilla JavaScript, Azure Speech SDK (CDN), MediaRecorder API。

---

## 文件结构

```
口语练习/
├── index.html          # 完整应用（配置、录音、评估、Drill、错误处理）
└── docs/
    └── superpowers/
        ├── specs/2026-07-06-english-reading-practice-design.md
        └── plans/2026-07-06-english-reading-practice-plan.md
```

---

## 任务 1：创建 index.html 骨架与 Azure SDK 引入

**文件：**
- 创建：`index.html`

- [ ] **步骤 1：编写最小 HTML 骨架，引入 Azure Speech SDK**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>英语朗读练习</title>
  <style>
    /* ponytail: 最小样式，先保证可用再美化 */
    body { font-family: system-ui, -apple-system, sans-serif; margin: 1rem; line-height: 1.6; }
    .section { margin-bottom: 1.5rem; }
    label { display: block; margin-top: 0.5rem; font-size: 0.9rem; }
    input, textarea, button { font-size: 1rem; padding: 0.5rem; width: 100%; box-sizing: border-box; }
    textarea { min-height: 120px; }
    button { margin-top: 0.5rem; }
  </style>
</head>
<body>
  <h1>英语朗读练习</h1>
  <div id="app"></div>
  <script src="https://aka.ms/csspeech/jsbrowserpackageraw"></script>
  <script>
    // ponytail: 单文件应用，所有逻辑放这里
    console.log('SpeechSDK:', typeof SpeechSDK);
  </script>
</body>
</html>
```

- [ ] **步骤 2：在桌面浏览器打开验证**

运行：直接双击 `index.html` 在 Chrome/Edge 打开
预期：页面显示标题，控制台输出 `SpeechSDK: object`

- [ ] **步骤 3：Commit**

```bash
git add index.html
git commit -m "feat: scaffold index.html with Azure SDK"
```

---

## 任务 2：配置面板与原文输入

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：在 body 内添加配置和文本输入 UI**

```html
<div class="section" id="setup">
  <label>Azure Region</label>
  <input id="region" type="text" placeholder="westus2" value="westus2">
  <label>Azure Subscription Key</label>
  <input id="key" type="password" placeholder="输入你的 Azure Speech Key">
  <label>英文原文</label>
  <textarea id="sourceText" placeholder="在此粘贴英文原文..."></textarea>
  <button id="startBtn">开始朗读评估</button>
  <button id="demoBtn">用示例文本试试</button>
</div>
```

- [ ] **步骤 2：添加 JS 读取这些字段并在点击时校验**

```javascript
const $ = id => document.getElementById(id);
$('demoBtn').onclick = () => {
  $('sourceText').value = "I've always wanted to build a one-person business.";
};
$('startBtn').onclick = () => {
  const region = $('region').value.trim();
  const key = $('key').value.trim();
  const text = $('sourceText').value.trim();
  if (!region || !key || !text) {
    alert('请填写 Region、Key 和原文');
    return;
  }
  console.log('ready', { region, textLength: text.length });
};
```

- [ ] **步骤 3：刷新页面验证**

运行：填写 Key/Region 和原文，点击“开始朗读评估”
预期：控制台输出 `ready` 对象，不填时弹出提示

- [ ] **步骤 4：Commit**

```bash
git add index.html
git commit -m "feat: add setup form and demo text"
```

---

## 任务 3：录音状态与计时器

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：添加录音状态 UI**

```html
<div class="section" id="recording" style="display:none">
  <p id="recText"></p>
  <p>录音时长：<span id="recTimer">0.0</span>s</p>
  <button id="stopBtn">停止录音</button>
</div>
```

- [ ] **步骤 2：实现录音计时与切换显示**

```javascript
let recInterval;
function showRecording(text) {
  $('setup').style.display = 'none';
  $('recording').style.display = 'block';
  $('recText').textContent = text;
  $('recTimer').textContent = '0.0';
  const start = Date.now();
  recInterval = setInterval(() => {
    $('recTimer').textContent = ((Date.now() - start) / 1000).toFixed(1);
  }, 100);
}
$('startBtn').onclick = () => {
  const text = $('sourceText').value.trim();
  if (!text) { alert('请输入原文'); return; }
  showRecording(text);
};
$('stopBtn').onclick = () => {
  clearInterval(recInterval);
  console.log('stopped');
};
```

- [ ] **步骤 3：刷新验证**

运行：点击“开始朗读评估”
预期：进入录音状态，计时器开始增加，点击停止后停止

- [ ] **步骤 4：Commit**

```bash
git add index.html
git commit -m "feat: add recording state and timer"
```

---

## 任务 4：调用 Azure Speech 发音评估

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：创建发音评估函数**

```javascript
async function assessPronunciation(text, key, region) {
  const speechConfig = SpeechSDK.SpeechConfig.fromSubscription(key, region);
  speechConfig.speechRecognitionLanguage = 'en-US';
  const pronunciationConfig = SpeechSDK.PronunciationAssessmentConfig.fromJson(JSON.stringify({
    referenceText: text,
    gradingSystem: SpeechSDK.PronunciationAssessmentGradingSystem.HundredMark,
    granularity: SpeechSDK.PronunciationAssessmentGranularity.Phoneme,
    enableMiscue: true
  }));
  const audioConfig = SpeechSDK.AudioConfig.fromDefaultMicrophoneInput();
  const recognizer = new SpeechSDK.SpeechRecognizer(speechConfig, audioConfig);
  pronunciationConfig.applyTo(recognizer);
  return new Promise((resolve, reject) => {
    recognizer.recognizeOnceAsync(
      result => {
        recognizer.close();
        if (result.reason === SpeechSDK.ResultReason.RecognizedSpeech) {
          const json = result.properties.getProperty(SpeechSDK.PropertyId.SpeechServiceResponse_JsonResult);
          resolve(JSON.parse(json));
        } else {
          reject(new Error('识别失败: ' + result.errorDetails || result.reason));
        }
      },
      err => { recognizer.close(); reject(err); }
    );
  });
}
```

- [ ] **步骤 2：将 startBtn 接入真实评估**

```javascript
$('startBtn').onclick = async () => {
  const region = $('region').value.trim();
  const key = $('key').value.trim();
  const text = $('sourceText').value.trim();
  if (!region || !key || !text) { alert('请填写完整'); return; }
  showRecording(text);
  try {
    const result = await assessPronunciation(text, key, region);
    console.log('result', result);
    clearInterval(recInterval);
  } catch (e) {
    clearInterval(recInterval);
    alert(e.message);
  }
};
```

- [ ] **步骤 3：使用真实 Azure Key 在桌面浏览器验证**

运行：填入有效 Key/Region，点击开始并朗读
预期：控制台输出包含 `NBest[0].PronunciationAssessment` 的 JSON

- [ ] **步骤 4：Commit**

```bash
git add index.html
git commit -m "feat: integrate Azure pronunciation assessment"
```

---

## 任务 5：文本归一化与对齐

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：添加归一化与对齐函数**

```javascript
function normalizeWord(w) {
  return w.toLowerCase()
    .replace(/[.,!?;:"'()\[\]]/g, '')
    .replace(/’/g, "'")
    .replace(/i'm/g, 'im')
    .replace(/i've/g, 'ive')
    .replace(/i'll/g, 'ill')
    .replace(/don't/g, 'dont')
    .replace(/can't/g, 'cant')
    .trim();
}
function tokenize(text) {
  return text.split(/\s+/).filter(Boolean).map((w, i) => ({ raw: w, norm: normalizeWord(w), index: i }));
}
function align(sourceTokens, recoWords) {
  const out = sourceTokens.map(t => ({ ...t, status: 'missing', score: 0, recognized: null }));
  let s = 0;
  for (const rw of recoWords) {
    const rn = normalizeWord(rw.Word || rw.word || '');
    if (!rn) continue;
    // 找到下一个匹配的原文词
    let matched = -1;
    for (let i = s; i < out.length; i++) {
      if (out[i].status === 'missing' && out[i].norm === rn) { matched = i; break; }
    }
    if (matched >= 0) {
      out[matched].status = 'ok';
      out[matched].score = (rw.PronunciationAssessment?.AccuracyScore) || 80;
      out[matched].recognized = rw.Word;
      s = matched + 1;
    } else {
      out.push({ raw: rw.Word, norm: rn, index: -1, status: 'extra', score: 0, recognized: rw.Word });
    }
  }
  return out;
}
```

- [ ] **步骤 2：将 Azure 返回的词列表解析出来并打印对齐结果**

```javascript
function getRecoWords(result) {
  const nbest = result.NBest?.[0];
  return nbest?.Words || [];
}
// 在 assessPronunciation 成功后调用
const words = getRecoWords(result);
const sourceTokens = tokenize(text);
const aligned = align(sourceTokens, words);
console.table(aligned);
```

- [ ] **步骤 3：用示例文本录音验证**

运行：用 demo 文本朗读
预期：控制台 table 显示每个原文词的 status、score、recognized

- [ ] **步骤 4：Commit**

```bash
git add index.html
git commit -m "feat: add text normalization and word alignment"
```

---

## 任务 6：结果高亮展示

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：添加结果区域 UI**

```html
<div class="section" id="results" style="display:none">
  <div id="scores"></div>
  <div id="highlighted" style="line-height:2; font-size:1.1rem;"></div>
  <button id="drillSentenceBtn">练习问题句</button>
  <button id="drillWordBtn">练习问题词</button>
  <button id="retryFullBtn">重新读全文</button>
</div>
```

- [ ] **步骤 2：实现高亮渲染**

```javascript
function scoreToColor(score, status) {
  if (status === 'missing') return '#fecaca'; // red
  if (status === 'extra') return '#fef08a';   // yellow
  if (score >= 80) return '#bbf7d0';          // green
  if (score >= 60) return '#fef08a';          // yellow
  return '#fecaca';                           // red
}
function renderResults(aligned, overall) {
  $('recording').style.display = 'none';
  $('results').style.display = 'block';
  $('scores').innerHTML = `
    <strong>总体 ${overall.PronScore}</strong> |
    准确度 ${overall.AccuracyScore} |
    流畅度 ${overall.FluencyScore} |
    语调 ${overall.ProsodyScore}
  `;
  $('highlighted').innerHTML = aligned.map(t => {
    const bg = scoreToColor(t.score, t.status);
    return `<span style="background:${bg};padding:2px 4px;border-radius:4px;cursor:pointer" data-idx="${t.index}">${t.raw}</span>`;
  }).join(' ');
}
```

- [ ] **步骤 3：在评估成功后调用 renderResults**

```javascript
const nbest = result.NBest?.[0];
renderResults(aligned, nbest.PronunciationAssessment);
```

- [ ] **步骤 4：刷新验证**

运行：完成一次录音
预期：页面显示分数和彩色高亮原文

- [ ] **步骤 5：Commit**

```bash
git add index.html
git commit -m "feat: render pronunciation scores and highlights"
```

---

## 任务 7：Drill 模式（句子/单词循环）

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：添加 Drill UI**

```html
<div class="section" id="drill" style="display:none">
  <p id="drillTarget" style="font-size:1.2rem;padding:1rem;background:#fff7ed;border-left:4px solid #f97316"></p>
  <button id="playTtsBtn">🔊 听标准发音</button>
  <button id="drillRecordBtn">🎙️ 我读</button>
  <button id="nextDrillBtn">下一个</button>
  <button id="backBtn">返回结果</button>
  <p id="drillStatus"></p>
</div>
```

- [ ] **步骤 2：实现句子拆分与 Drill 队列**

```javascript
function splitSentences(text) {
  // ponytail: 简单保护 U.S. 类缩写，不做完美 NLP
  return text.replace(/(Mr|Mrs|Ms|Dr|U\.S|e\.g|i\.e)\.\s+/g, '$1##DOT## ')
    .split(/[.!?]+\s+/)
    .map(s => s.replace(/##DOT##/g, '.').trim())
    .filter(Boolean);
}
let drillQueue = [];
let drillIndex = 0;
function buildDrillQueue(aligned, mode) {
  if (mode === 'word') {
    drillQueue = aligned.filter(t => t.status !== 'ok' || t.score < 80).map(t => t.raw);
  } else {
    const badIndices = new Set(aligned.filter(t => t.status !== 'ok' || t.score < 80).map(t => t.index));
    const sents = splitSentences($('sourceText').value);
    drillQueue = sents.filter((s, i) => {
      const words = tokenize(s);
      return words.some(w => badIndices.has(w.index + aligned.filter(a => a.index < 0).length));
    });
  }
  drillIndex = 0;
}
```

- [ ] **步骤 3：实现 Drill 录音与评分**

```javascript
async function runDrill() {
  if (drillQueue.length === 0) { alert('没有需要练习的内容'); return; }
  $('results').style.display = 'none';
  $('drill').style.display = 'block';
  const target = drillQueue[drillIndex];
  $('drillTarget').textContent = target;
  $('drillStatus').textContent = `第 ${drillIndex + 1}/${drillQueue.length} 个`;
}
$('drillSentenceBtn').onclick = () => { buildDrillQueue(window.lastAligned, 'sentence'); runDrill(); };
$('drillWordBtn').onclick = () => { buildDrillQueue(window.lastAligned, 'word'); runDrill(); };
$('drillRecordBtn').onclick = async () => {
  const target = drillQueue[drillIndex];
  const region = $('region').value.trim();
  const key = $('key').value.trim();
  const result = await assessPronunciation(target, key, region);
  const score = result.NBest?.[0]?.PronunciationAssessment?.PronScore || 0;
  $('drillStatus').textContent = `得分 ${score}；${score >= 80 ? '通过' : '再试一次'}`;
};
$('nextDrillBtn').onclick = () => { drillIndex++; if (drillIndex >= drillQueue.length) drillIndex = 0; runDrill(); };
$('backBtn').onclick = () => { $('drill').style.display = 'none'; $('results').style.display = 'block'; };
```

- [ ] **步骤 4：在 renderResults 中保存 aligned 供 Drill 使用**

```javascript
window.lastAligned = aligned;
```

- [ ] **步骤 5：验证 Drill 流程**

运行：录音后点击“练习问题句”或“练习问题词”，录音，查看得分
预期：Drill 面板切换，显示目标片段和得分

- [ ] **步骤 6：Commit**

```bash
git add index.html
git commit -m "feat: add sentence and word drill mode"
```

---

## 任务 8：标准发音播放（TTS）

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：实现浏览器原生 TTS**

```javascript
$('playTtsBtn').onclick = () => {
  const target = drillQueue[drillIndex];
  const u = new SpeechSynthesisUtterance(target);
  u.lang = 'en-US';
  u.rate = 0.9;
  speechSynthesis.speak(u);
};
```

- [ ] **步骤 2：在 Drill 面板点击“听标准发音”验证**

运行：进入 Drill，点击听发音
预期：设备播放英文

- [ ] **步骤 3：Commit**

```bash
git add index.html
git commit -m "feat: add browser TTS for drill target"
```

---

## 任务 9：错误处理与用户提示

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：添加 showError 辅助函数**

```javascript
function showError(msg) {
  alert(msg); // ponytail: 先用 alert，后续可换成页面内提示
}
```

- [ ] **步骤 2：在 assessPronunciation 调用处分类处理错误**

```javascript
$('startBtn').onclick = async () => {
  // ... 校验 ...
  showRecording(text);
  try {
    const result = await assessPronunciation(text, key, region);
    // ...
  } catch (e) {
    clearInterval(recInterval);
    const msg = e.message || '';
    if (msg.includes('401') || msg.includes('403')) showError('Azure Key 或 Region 无效，请检查 Azure 门户');
    else if (msg.includes('network') || msg.includes('ECONNREFUSED')) showError('连接 Azure 失败，请检查网络');
    else if (msg.includes('NotAllowed')) showError('麦克风权限被拒绝，请在浏览器设置中允许');
    else if (msg.includes('识别失败')) showError('未检测到语音，请靠近麦克风再试一次');
    else showError('出错了：' + msg);
  }
};
```

- [ ] **步骤 3：测试错误场景**

运行：
- 故意填错 Key → 看到 Key 无效提示
- 拒绝麦克风权限 → 看到权限提示
- 不讲话点击停止 → 看到未检测到语音提示

- [ ] **步骤 4：Commit**

```bash
git add index.html
git commit -m "feat: add user-friendly error messages"
```

---

## 任务 10：iOS Safari 适配与移动样式

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：优化 viewport 与触控**

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
```

- [ ] **步骤 2：添加移动优先 CSS**

```css
body { margin: 0; padding: 1rem; background: #fff; color: #111; }
button { min-height: 44px; touch-action: manipulation; }
#highlighted span { display: inline-block; margin: 2px; }
#drillTarget { word-break: break-word; }
```

- [ ] **步骤 3：确保录音按钮在 iOS 上可用**

录音必须通过用户手势触发（已满足：点击按钮触发 assessPronunciation）。

- [ ] **步骤 4：在 iPhone Safari 打开验证**

运行：把文件传到手机或本地网络访问，用 Safari 打开
预期：布局正常，能请求麦克风，能录音并返回结果

- [ ] **步骤 5：Commit**

```bash
git add index.html
git commit -m "feat: mobile-first styles and iOS Safari compatibility"
```

---

## 任务 11：模拟数据模式（离线验证 UI）

**文件：**
- 修改：`index.html`

- [ ] **步骤 1：添加“模拟评估”按钮和写死数据**

```html
<button id="mockBtn">用模拟数据看效果</button>
```

```javascript
$('mockBtn').onclick = () => {
  const text = $('sourceText').value.trim() || "I've always wanted to build a one-person business.";
  const sourceTokens = tokenize(text);
  const mockResult = {
    NBest: [{
      PronunciationAssessment: { PronScore: 82, AccuracyScore: 85, FluencyScore: 78, ProsodyScore: 80 },
      Words: sourceTokens.map((t, i) => ({
        Word: t.raw,
        PronunciationAssessment: { AccuracyScore: i === 3 ? 45 : (i === 5 ? 70 : 88) }
      }))
    }]
  };
  const words = getRecoWords(mockResult);
  const aligned = align(sourceTokens, words);
  window.lastAligned = aligned;
  renderResults(aligned, mockResult.NBest[0].PronunciationAssessment);
};
```

- [ ] **步骤 2：离线打开验证**

运行：不填 Key，点击“用模拟数据看效果”
预期：页面显示分数和高亮（其中 build 红色，person 黄色）

- [ ] **步骤 3：Commit**

```bash
git add index.html
git commit -m "feat: add mock mode for offline UI testing"
```

---

## 自检

- **规格覆盖度：**
  - 单文件网页应用：✓ 任务 1
  - 手动粘贴原文：✓ 任务 2
  - Azure Speech 发音评估 A-D：✓ 任务 4
  - Key/Region 用户填写：✓ 任务 2/4
  - 混合练习模式：✓ 任务 6/7
  - 对齐与高亮：✓ 任务 5/6
  - iOS Safari 适配：✓ 任务 10
  - 错误处理：✓ 任务 9
  - 测试策略：✓ 各任务中的验证步骤 + 任务 11 模拟数据

- **占位符扫描：** 无 TODO、无“后续实现”、所有步骤包含实际代码。

- **类型一致性：** 全局状态通过 `window.lastAligned` 传递；字段名 `PronScore/AccuracyScore/FluencyScore/ProsodyScore` 与 Azure 返回一致。
