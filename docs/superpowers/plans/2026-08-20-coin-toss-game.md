# 佛前投币 · Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a pure-static, mobile-first H5 "佛前投币" (toss-a-coin-before-a-buddha) wish game as a single `index.html`, gated behind the GamePlatform login SDK, with manual settlement into a local 10-item history.

**Architecture:** One self-contained `index.html` (inline CSS + inline JS). State lives in JS objects/in-memory + `localStorage` (`coin-toss:history`). Buddha sprites are referenced from `assets/`. The coin uses the two dedicated coin sprites (`...T06-59-50.png` = one face, `...T07-00-05.png` = the other face) as its front/back, flipped with CSS `rotateY`. Login gating mirrors the existing 2048 pattern.

**Tech Stack:** Vanilla HTML/CSS/JS, localStorage, Touch/pointer events for the carousel, CSS 3D transforms for the coin, GamePlatform SDK (`https://api.w1lkin.site/sdk.js`).

## Global Constraints

- Max-width layout `~420px`, centered, mobile-first, safe-area insets (`env(safe-area-inset-bottom)`).
- Single file `coin-toss/index.html`; all CSS/JS inline; only external script is the SDK.
- Coin outcome: `Math.random() < buddha.rate` ⇒ 正面 (字, accepted). Same-buddha throws are independent.
- Settlement is manual only via 「结束本次」; record pushed to `coin-toss:history` even if never accepted.
- History array keeps only latest **10** records; each record `{ wish, buddha, throws, accepted, ts }`.
- GamePlatform gate: on failure or missing SDK, play anyway (graceful drop).
- No `submitScore` (娱乐向, `skip:true`, not on leaderboard).
- All user-facing strings in Simplified Chinese. Pixel sprites rendered with `image-rendering: pixelated`.
- Project is **not yet a git repo** — Task 1 initializes it. Deploy steps are manual (Cloudflare Pages), performed by the user.

---

### Task 1: Scaffold `index.html` shell + full CSS layout

**Files:**
- Create: `coin-toss/index.html`
- Create: `coin-toss/.gitignore`
- Test: manual browser open

**Interfaces:**
- Produces: `index.html` with `id` hooks consumed by every later task:
  `#app`, `#carousel`, `#carousel-track`, `#buddha-name`, `#buddha-words`, `#buddha-rate`, `#wish-input`, `#btn-toss`, `#btn-end`, `#coin`, `#coin-inner`, `#coin-head`, `#coin-tail`, `#result`, `#status`, `#status-throws`, `#status-miss`, `#history`, `#loading`.

- [ ] **Step 1: Initialize git repo**

```bash
cd /Users/welkin/Code/game/coin-toss && git init -b main
```
Expected: `Initialized empty Git repository...`

- [ ] **Step 2: Create `.gitignore`**

```
.DS_Store
*.log
```

- [ ] **Step 3: Write `index.html` shell (markup + full CSS)**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, viewport-fit=cover">
<meta name="api-base" content="https://api.w1lkin.site">
<title>佛前投币</title>
<meta property="og:title" content="佛前投币">
<meta property="og:description" content="写心愿，投币许愿，心诚则灵。">
<meta property="og:type" content="website">
<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html, body { height: 100%; }
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "PingFang SC", "Microsoft YaHei", sans-serif;
  background: linear-gradient(180deg, #2a1a0e 0%, #4a2c14 100%);
  color: #f5e6c8;
  display: flex;
  flex-direction: column;
  align-items: center;
  -webkit-tap-highlight-color: transparent;
  -webkit-user-select: none; user-select: none;
  overflow-x: hidden;
}
#app {
  width: 100%; max-width: 420px; margin: 0 auto; padding: 12px 16px calc(16px + env(safe-area-inset-bottom));
}
.gp-home-btn {
  position: fixed; top: 12px; right: 12px; z-index: 50;
  font-size: 18px; text-decoration: none; opacity: 0.4; color: #f5e6c8;
}
.gp-home-btn:active { opacity: 0.8; }

/* 顶部标题 */
.title { text-align: center; font-size: 30px; font-weight: 800; margin: 4px 0 12px; letter-spacing: 2px; }

/* 佛像轮播 */
#carousel { position: relative; width: 100%; margin-bottom: 12px; }
#carousel-track {
  display: flex; overflow-x: auto; scroll-snap-type: x mandatory;
  -webkit-overflow-scrolling: touch; scrollbar-width: none; cursor: grab;
}
#carousel-track::-webkit-scrollbar { display: none; }
.buddha-slot {
  flex: 0 0 72%; scroll-snap-align: center;
  margin: 0 auto; padding: 14px; text-align: center;
  background: rgba(255,255,255,0.06); border-radius: 14px;
  border: 2px solid transparent; transition: border-color .2s, transform .2s;
}
.buddha-slot.selected { border-color: #ffd76a; transform: translateY(-3px); }
.buddha-slot img {
  width: 120px; height: 120px; object-fit: contain; image-rendering: pixelated;
  margin-bottom: 6px;
}
.buddha-slot .slot-emoji {
  width: 120px; height: 120px; display: flex; align-items: center; justify-content: center;
  font-size: 80px; color: #ffd76a; margin: 0 auto 6px; background: rgba(255,255,255,0.06); border-radius: 12px;
}
.buddha-slot .b-name { font-size: 18px; font-weight: 700; }
.buddha-slot .b-rate { font-size: 12px; color: #d8c39a; margin-top: 2px; }

.buddha-meta { text-align: center; margin-bottom: 10px; min-height: 44px; }
.buddha-meta .b-words { font-size: 15px; color: #ffd76a; font-style: italic; }

/* 许愿区 */
#wish-section { margin-bottom: 12px; }
#wish-input {
  width: 100%; padding: 12px 14px; border-radius: 10px; border: 1px solid #6b4a26;
  background: rgba(0,0,0,0.25); color: #f5e6c8; font-size: 16px;
  -webkit-user-select: text; user-select: text;
}
#wish-input::placeholder { color: #a98d5f; }

/* 硬币区 */
#coin-area { display: flex; flex-direction: column; align-items: center; margin-bottom: 14px; }
.coin-stage { perspective: 600px; width: 120px; height: 120px; margin-bottom: 12px; animation: coinBob 2s ease-in-out infinite; }
#coin {
  position: relative; width: 100%; height: 100%; transform-style: preserve-3d; transition: transform .7s cubic-bezier(.2,.8,.3,1);
}
.coin-face { position: absolute; inset: 0; border-radius: 50%; backface-visibility: hidden; background-size: cover; background-position: center; }
#coin-head {
  background-image: url('assets/pixel_art_sprite__retro_16_bit_2026-08-20T06-59-50.png');
}
#coin-tail {
  background-image: url('assets/pixel_art_sprite__retro_16_bit_2026-08-20T07-00-05.png');
  transform: rotateY(180deg);
}
#coin.still { animation: none; }
@keyframes coinBob { 0%,100% { transform: translateY(0); } 50% { transform: translateY(-6px); } }

/* 投币按钮 */
.btn {
  width: 100%; padding: 13px; border: none; border-radius: 12px;
  font-size: 17px; font-weight: 700; cursor: pointer; transition: opacity .15s, transform .05s;
}
.btn:active { transform: scale(.98); }
.btn-toss { background: linear-gradient(180deg, #ffd76a, #e8a838); color: #5a3206; }
.btn-toss:disabled { opacity: .45; }
.btn-end { background: rgba(255,255,255,0.12); color: #f5e6c8; margin-top: 8px; }

/* 结果反馈 */
#result { min-height: 30px; text-align: center; font-size: 15px; margin: 10px 0 4px; font-weight: 600; }
#result.gold { color: #ffd76a; text-shadow: 0 0 12px rgba(255,215,106,.7); }

/* 状态区 */
#status { text-align: center; font-size: 13px; color: #d8c39a; margin-bottom: 14px; }

/* 历史 */
#history { margin-top: 4px; }
#history h3 { font-size: 15px; color: #ffd76a; margin-bottom: 6px; }
.h-item { background: rgba(255,255,255,0.05); border-radius: 8px; padding: 8px 10px; margin-bottom: 6px; font-size: 12px; line-height: 1.5; color: #ead9b3; }
.h-item .h-wish { display: -webkit-box; -webkit-line-clamp: 1; -webkit-box-orient: vertical; overflow: hidden; }
.h-item.accepted { border-left: 3px solid #ffd76a; }
.h-item .h-time { color: #a98d5f; }

#loading { position: fixed; inset: 0; display: flex; align-items: center; justify-content: center; background: #2a1a0e; color: #ffd76a; z-index: 99; }
</style>
</head>
<body>
<div id="loading">加载中…</div>
<div id="app" style="display:none">
  <div class="title">🙏 佛前投币</div>

  <div id="carousel">
    <div id="carousel-track"></div>
  </div>
  <div id="buddha-meta" class="buddha-meta">
    <div id="buddha-name"></div>
    <div id="buddha-words"></div>
    <div id="buddha-rate"></div>
  </div>

  <div id="wish-section">
    <input id="wish-input" type="text" maxlength="30" placeholder="写下你的心愿…" inputmode="default">
  </div>

  <div id="coin-area">
    <div class="coin-stage">
      <div id="coin" class="still">
        <div id="coin-head" class="coin-face"></div>
        <div id="coin-tail" class="coin-face"></div>
      </div>
    </div>
    <button id="btn-toss" class="btn btn-toss">投币</button>
    <div id="result"></div>
    <button id="btn-end" class="btn btn-end">结束本次</button>
  </div>

  <div id="status">本次已投 <span id="status-throws">0</span> 次 / 当前连续未中 <span id="status-miss">0</span> 次</div>

  <div id="history">
    <h3>最近 10 次</h3>
  </div>
</div>
<a href="https://game-list-arv.pages.dev" class="gp-home-btn" title="主页">🏠</a>
<script src="https://api.w1lkin.site/sdk.js"></script>
</body>
</html>
```

- [ ] **Step 4: Manual smoke test — layout renders**

Open `coin-toss/index.html` in a browser. Expected: loading screen then the full app shell (title, empty carousel, wish input, static coin showing the front coin sprite, two buttons, status line, history heading). Coin bob animates.

- [ ] **Step 5: Commit**

```bash
cd /Users/welkin/Code/game/coin-toss && git add .gitignore index.html && git commit -m "chore: scaffold 佛前投币 shell and layout"
```

---

### Task 2: Buddha data + horizontal carousel selection

**Files:**
- Modify: `coin-toss/index.html`
- Test: manual browser check that swiping selects a buddha and updates meta

**Interfaces:**
- Produces global array `BUDDHAS` — order matters; later tasks read `BUDDHAS[i]`.
  `{ id, name, rate, words, img }` where `img` is the asset path, or `null` for 信自己.
- Produces `initCarousel()` and `selectBuddha(index)`.
- `selectBuddha(i)` sets `currentBuddhaIndex`, toggles `.selected`, and updates `#buddha-name`, `#buddha-rate`, `#buddha-words`.

- [ ] **Step 1: Define `BUDDHAS` and carousel logic (insert a `<script>` before `</body>`, after the SDK script tag)**

Assign the 7 buddha sprites (`...06-57-47` … `...06-59-34`) to: 观音, 弥勒, 如来, 文殊, 普贤, 地藏, 药师. `信自己` is the `img:null` "self-belief" slot (static, no sprite). The two later sprites `...06-59-50` and `...07-00-05` are the **coin front/back** — do NOT use them as buddhas.

> ⚠️ Sprite→name mapping is a **visual guess** (AI outputs are unlabeled). The implementer must open each PNG to confirm which file looks like which buddha, and swap the `img` paths below accordingly.

```html
<script>
(function () {
  'use strict';

  var BUDDHAS = [
    { id:'self',  name:'信自己', rate:0.50, words:'心诚则灵，你自己最灵',           img:null },
    { id:'guanyin', name:'观音', rate:0.55, words:'慈悲加持，所愿皆成',           img:'assets/pixel_art_sprite__retro_16_bit_2026-08-20T06-57-47.png' },
    { id:'mile',  name:'弥勒', rate:0.60, words:'笑口常开，好运自然来',           img:'assets/pixel_art_sprite__retro_16_bit_2026-08-20T06-58-05.png' },
    { id:'rulai', name:'如来', rate:0.50, words:'如来见证，静待花开',             img:'assets/pixel_art_sprite__retro_16_bit_2026-08-20T06-58-20.png' },
    { id:'wenshu',name:'文殊', rate:0.53, words:'智慧光明，迷障尽除',             img:'assets/pixel_art_sprite__retro_16_bit_2026-08-20T06-58-42.png' },
    { id:'puxian',name:'普贤', rate:0.52, words:'行愿圆满，前路坦途',             img:'assets/pixel_art_sprite__retro_16_bit_2026-08-20T06-58-59.png' },
    { id:'dizang',name:'地藏', rate:0.54, words:'誓愿深重，苦难得度',             img:'assets/pixel_art_sprite__retro_16_bit_2026-08-20T06-59-16.png' },
    { id:'yaoshi',name:'药师', rate:0.56, words:'消灾延寿，身心安康',             img:'assets/pixel_art_sprite__retro_16_bit_2026-08-20T06-59-34.png' }
  ];

  var track = document.getElementById('carousel-track');
  var currentBuddhaIndex = 0;

  function renderBuddhas() {
    track.innerHTML = '';
    BUDDHAS.forEach(function (b, i) {
      var slot = document.createElement('div');
      slot.className = 'buddha-slot';
      if (i === currentBuddhaIndex) slot.classList.add('selected');
      if (b.img) {
        var img = document.createElement('img');
        img.src = b.img; img.alt = b.name;
        slot.appendChild(img);
      } else {
        var e = document.createElement('div');
        e.className = 'slot-emoji'; e.textContent = '✨';
        slot.appendChild(e);
      }
      var name = document.createElement('div');
      name.className = 'b-name'; name.textContent = b.name;
      var rate = document.createElement('div');
      rate.className = 'b-rate'; rate.textContent = '应验率 ' + Math.round(b.rate * 100) + '%';
      slot.appendChild(name); slot.appendChild(rate);
      track.appendChild(slot);
    });
    updateMeta();
  }

  function updateMeta() {
    var b = BUDDHAS[currentBuddhaIndex];
    document.getElementById('buddha-name').textContent = '「' + b.name + '」';
    document.getElementById('buddha-words').textContent = b.words;
    document.getElementById('buddha-rate').textContent = '应验率 ' + Math.round(b.rate * 100) + '%';
  }

  function selectBuddha(i) {
    currentBuddhaIndex = i;
    var slots = track.querySelectorAll('.buddha-slot');
    slots.forEach(function (s, idx) { s.classList.toggle('selected', idx === i); });
    // snap selected slot into view
    var slot = slots[i];
    if (slot) track.scrollTo({ left: slot.offsetLeft - (track.clientWidth - slot.clientWidth) / 2, behavior: 'smooth' });
    updateMeta();
  }

  // 滑动结束时，以中心位佛为准选择
  function pickCentered() {
    var centerX = track.scrollLeft + track.clientWidth / 2;
    var slots = track.querySelectorAll('.buddha-slot');
    var best = 0, bestDist = Infinity;
    slots.forEach(function (s, idx) {
      var c = s.offsetLeft + s.clientWidth / 2;
      var d = Math.abs(c - centerX);
      if (d < bestDist) { bestDist = d; best = idx; }
    });
    selectBuddha(best);
  }

  track.addEventListener('scroll', function () {
    clearTimeout(track._t);
    track._t = setTimeout(pickCentered, 120);
  });

  window.BUDDHAS = BUDDHAS;
  window.selectBuddha = selectBuddha;
  window.renderBuddhas = renderBuddhas;
})();
</script>
```

- [ ] **Step 2: Manual test — carousel works**

Open in browser. Expected: 8 slots (信自己 first, shown as ✨, then 7 buddhas with sprites). Swiping horizontally scroll-snaps and, 120ms after scroll ends, highlights the centered buddha with gold border and updates 「…」/吉祥语/应验率 in the meta area. Validate sprite faces match their assigned names; swap `img` paths if any file looks wrong.

- [ ] **Step 3: Commit**

```bash
cd /Users/welkin/Code/game/coin-toss && git add index.html && git commit -m "feat: add buddha carousel selection"
```

---

### Task 3: Wish input + 投币 flip + accept/reject feedback

**Files:**
- Modify: `coin-toss/index.html`
- Test: manual — tossing flips coin 540°, result text updates correctly

**Interfaces:**
- Produces `tossCoin()` (call on 投币), reads `currentBuddhaIndex`.
- Produces `BUDDHAS[currentBuddhaIndex].rate` use for acceptance.
- Produces a global current-round state consumed by Task 4:
  `state = { throws, missStreak, wish, accepted, buddhaName }`.
- Exposes `window.__setBusy(bool)` / re-enables the button after flip. (Simplest: inline in this task.)

- [ ] **Step 1: Add round state + toss logic (append into the SAME IIFE from Task 2 — add above `window.BUDDHAS` lines)**

```html
<script>
  // ... continue inside the existing IIFE, before the window.* exports ...

  var coin = document.getElementById('coin');
  var resultEl = document.getElementById('result');
  var btnToss = document.getElementById('btn-toss');
  var wishInput = document.getElementById('wish-input');
  var flipping = false;

  var state = { throws: 0, missStreak: 0, wish: '', accepted: false, buddhaName: BUDDHAS[0].name };

  function tossCoin() {
    if (flipping) return; // animating
    state.wish = (wishInput.value || '').trim();
    state.buddhaName = BUDDHAS[currentBuddhaIndex].name;
    state.throws += 1;

    var b = BUDDHAS[currentBuddhaIndex];
    var ok = Math.random() < b.rate;

    flipping = true;
    coin.classList.remove('still');
    btnToss.disabled = true;
    // 720°(2 turns)→正面(字)在上；540°(1.5 turns)→反面(花)在上。让结果面朝上。
    coin.style.transform = ok ? 'rotateY(720deg)' : 'rotateY(540deg)';

    setTimeout(function () {
      coin.classList.add('still');
      btnToss.disabled = false;
      flipping = false;

      if (ok) {
        state.accepted = true;
        state.missStreak = 0;
        resultEl.className = 'gold';
        resultEl.innerHTML = '✨ 佛已应允 · ' + b.words;
      } else {
        state.missStreak += 1;
        resultEl.className = '';
        resultEl.textContent = '🪙 再试一次';
      }
      updateStatus();
    }, 700);
  }

  function updateStatus() {
    document.getElementById('status-throws').textContent = state.throws;
    document.getElementById('status-miss').textContent = state.missStreak;
  }

  btnToss.addEventListener('click', tossCoin);
</script>
```

- [ ] **Step 2: Add `updateStatus` export + wire renderBuddhas to reset everything if needed (no-op here)**

Add to the existing `window.*` exports:
```html
<script>
  window.updateStatus = updateStatus;
</script>
```
(Adding `window.updateStatus` makes Task 4 able to reuse it.)

- [ ] **Step 3: Manual test — flip & verdict**

Open in browser. Enter a wish, tap 投币. Expected: stage bob continues while the coin itself rotates (720° on accept, 540° on reject) over 0.7s, ending with the result face (head sprite on accept, tail sprite on reject) on top; button disabled during flip. On accept: gold "✨ 佛已应允 · 吉祥语"; on reject: "🪙 再试一次". Status line counts up. Multiple throws accumulate; a later accept resets 连续未中. Confirm which sprite is head(字) vs tail(花) matches the outcome face shown.

- [ ] **Step 4: Commit**

```bash
cd /Users/welkin/Code/game/coin-toss && git add index.html && git commit -m "feat: implement coin toss flip and verdict"
```

---

### Task 4: Manual settlement + localStorage history (recent 10)

**Files:**
- Modify: `coin-toss/index.html`
- Test: manual — 结束本次 pushes a record, history renders, survives reload, caps at 10

**Interfaces:**
- Consumes: `state`, `window.updateStatus`, `BUDDHAS`, `currentBuddhaIndex`.
- Produces `settle()`, `loadHistory()` → `HISTORY`, `saveHistory()`, `renderHistory()`, `resetRound()`.
- Reads/writes `localStorage['coin-toss:history']`.

- [ ] **Step 1: Add settlement + history logic (inside the same IIFE)**

```html
<script>
  var HISTORY_KEY = 'coin-toss:history';
  var HISTORY = loadHistory();

  function loadHistory() {
    try {
      var arr = JSON.parse(localStorage.getItem(HISTORY_KEY) || '[]');
      return Array.isArray(arr) ? arr : [];
    } catch (e) { return []; }
  }

  function saveHistory() {
    try { localStorage.setItem(HISTORY_KEY, JSON.stringify(HISTORY)); } catch (e) { /* ignore */ }
  }

  function resetRound() {
    state = { throws: 0, missStreak: 0, wish: '', accepted: false, buddhaName: BUDDHAS[0].name };
    wishInput.value = '';
    resultEl.textContent = '';
    resultEl.className = '';
    updateStatus();
  }

  function settle() {
    HISTORY.push({
      wish: state.wish || '（未写心愿）',
      buddha: state.buddhaName,
      throws: state.throws,
      accepted: state.accepted,
      ts: Date.now()
    });
    if (HISTORY.length > 10) HISTORY = HISTORY.slice(HISTORY.length - 10);
    saveHistory();
    renderHistory();
    resetRound();
  }

  function renderHistory() {
    var box = document.getElementById('history');
    box.querySelectorAll('.h-item').forEach(function (n) { n.remove(); });
    var h = box.querySelector('h3');
    HISTORY.slice().reverse().forEach(function (r) {
      var div = document.createElement('div');
      div.className = 'h-item' + (r.accepted ? ' accepted' : '');
      var time = new Date(r.ts);
      var pad = function (n) { return String(n).padStart(2, '0'); };
      var timeStr = pad(time.getMonth() + 1) + '-' + pad(time.getDate()) + ' ' + pad(time.getHours()) + ':' + pad(time.getMinutes());
      div.innerHTML =
        '<div class="h-wish">' + escapeHtml(r.wish) + '</div>' +
        '<div>' + escapeHtml(r.buddha) + ' · 投了' + r.throws + '回 · ' + (r.accepted ? '已应允 ✅' : '未中') +
        ' <span class="h-time">' + timeStr + '</span></div>';
      box.insertBefore(div, h.nextSibling);
    });
  }

  function escapeHtml(s) {
    return String(s).replace(/[&<>"']/g, function (c) {
      return { '&':'&amp;', '<':'&lt;', '>':'&gt;', '"':'&quot;', "'":'&#39;' }[c];
    });
  }

  document.getElementById('btn-end').addEventListener('click', settle);
  window.settle = settle;
</script>
```

- [ ] **Step 2: Call `renderHistory()` + `initCarousel()`/`renderBuddhas()` on startup, hidden until bootstrap**

Append at end of the IIFE (and called from the gate in Task 5):
```html
<script>
  renderHistory();
</script>
```

- [ ] **Step 3: Manual test — settlement & persistence**

Open in browser. Toss a few times → tap 结束本次. Expected: a new history card appears (wish, buddha, throws count, 已应允/未中, time), input clears, status resets to 0/0. Reload page → history persists. Toss + settle across 11 rounds → only latest 10 cards shown.

- [ ] **Step 4: Commit**

```bash
cd /Users/welkin/Code/game/coin-toss && git add index.html && git commit -m "feat: add manual settlement and local history"
```

---

### Task 5: GamePlatform login gate

**Files:**
- Modify: `coin-toss/index.html`
- Test: manual — app only begins after gate resolves; still plays if SDK fails

**Interfaces:**
- Consumes: `window.renderBuddhas`, `window.renderHistory`, `window.updateStatus`.
- Produces addEventListener setup triggering `tossCoin`/`settle`/carousel (buttons/listeners are added across Tasks 2–4 inside the IIFE; this task just gates the reveal).

- [ ] **Step 1: Add reveal helper that calls render + shows app**

Insert a new small IIFE **at the very end of the body** (after all existing scripts), mirroring the 2048 gate:

```html
<script>
(function () {
  function startGame() {
    if (window.renderBuddhas) window.renderBuddhas();
    if (window.renderHistory) window.renderHistory();
    if (window.updateStatus) window.updateStatus();
    document.getElementById('loading').style.display = 'none';
    document.getElementById('app').style.display = '';
  }
  if (typeof GamePlatform !== 'undefined') {
    GamePlatform.init();
    GamePlatform.mountGate({ gameId: 'coin-toss' }).then(startGame).catch(startGame);
  } else {
    startGame();
  }
})();
</script>
```

- [ ] **Step 2: Manual test — gate & offline**

Open in browser with network. Expected: loading shown → login gate (if not logged in) → on success, app reveals with carousel + history rendered, coin animating. Block/disable network and reload: app still reveals and is fully playable (gate catches and drops).

- [ ] **Step 3: Commit**

```bash
cd /Users/welkin/Code/game/coin-toss && git add index.html && git commit -m "feat: gate coin-toss behind GamePlatform login"
```

---

### Task 6: Final cross-checks + game-api registration (manual ops)

**Files:**
- Modify: `coin-toss/index.html` (only if a bug appears)
- Create (game-api side, separate repo): `coin-toss` registration via a new migration
- Test: full playthrough

**Interfaces:**
- Consumes: everything above.

- [ ] **Step 1: Full manual regression**

In a browser, run a full loop: pick 观音 → write a wish → toss until accept (verify 连续未中 resets) → switch to 弥勒 mid-round (verify throws keep counting across the round) → 结束本次 → verify history. Then repeat to push past 10 records and verify the 10-cap and ordering (newest first). Check the 信自己 slot (✨, 50%) in the carousel. Verify all pixel sprites appear crisp (pixelated), no broken `img`.

- [ ] **Step 2: Register `coin-toss` in game-api `games` table (dry run)**

In `/Users/welkin/Code/game/game-api`, add migration `0011_coin_toss.sql` mirroring `0007_2048.sql` with `skip=1`:

```sql
INSERT OR IGNORE INTO games (game_id, label, metric, direction, skip, enabled, sort_order, category, emoji, url, desc, diffs)
VALUES ('coin-toss', '佛前投币', '距上次应允(次)', 'asc', 1, 1, 2, '玄学趣玩', '🪙', 'https://coin-toss-<slug>.pages.dev/', '写心愿投币许愿，心诚则灵', NULL);
```

> `metric`/`direction` here are display-only because `skip=1` excludes it from any leaderboard ranking. The `<slug>` is the Cloudflare Pages project subdomain you choose in Step 4.

- [ ] **Step 3: Confirm game-list respects `skip`**

In the game-list frontend, verify a `skip=1` game renders as a jumgable card but is excluded from the leaderboard/tian-ti list (do not modify leaderboard sort logic — only the portal list visibility path). If game-list already filters correctly, no change needed.

- [ ] **Step 4: Deploy (manual, user-run)**

1. Create a new Cloudflare Pages project (e.g. `coin-toss-xxx`), root = `coin-toss/`.
2. Deploy the branch so `index.html` + `assets/` are hosted.
3. Update the migration `url` to the real Pages domain, then apply `0011_coin_toss.sql` to D1 (`wrangler d1 execute ... --file=0011_coin_toss.sql`).
4. Manual smoke on the live URL (login gate + full loop).

- [ ] **Step 5: Final commit (game code)**

```bash
cd /Users/welkin/Code/game/coin-toss && git add -A && git commit -m "chore: finalize 佛前投币"
```

---

## Self-Review

- **Spec coverage:** 玩法循环 (T2/T3), 佛像轮播+信自己占位 (T2), 心愿输入 (T3), 硬币翻转+判定 (T3), 手动结算 (T4), 状态区计数 (T3/T4), 最近10历史 (T4), localStorage (T4), GamePlatform gate (T5), games 表 skip 登记 (T6), 素材本地引用离线可用 (T2 sprites + T3 CSS coin), 部署 (T6). 全覆盖。
- **Placeholders:** none — all syntax/commands concrete. The only "guess" is sprite→name mapping, explicitly flagged with a visual-verification step (`⚠️`), not a silent placeholder.
- **Type consistency:** `state`, `BUDDHAS`, `currentBuddhaIndex`, `updateStatus`, `renderHistory`, `renderBuddhas`, `selectBuddha`, `tossCoin`, `settle` used consistently across tasks; same file references throughout.
- **Scope:** correctly scoped to a single static game + one registration migration; deployment left as explicit manual ops (not build tooling).