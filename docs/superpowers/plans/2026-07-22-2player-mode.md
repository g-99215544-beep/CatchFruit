# 2-Player Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tambah mode "👥 2 Pemain" pada game Tangkap Buah di mana P1 tangkap buah dengan tangan terbuka, P2 dengan tangan genggam, dan markah tertinggi dalam masa tetap (30/45/60s) menang.

**Architecture:** Semua perubahan dalam satu fail `index.html`. Guna semula pipeline pengesanan tangan sedia ada (`numHands: 2`, medan `hand.open`). Tambah state per-pemain, kembangkan logik tangkap supaya tangan genggam juga aktif (untuk P2), tambah butang mode + skrin keputusan baru.

**Tech Stack:** HTML/CSS/JS vanilla (module), MediaPipe HandLandmarker (CDN), Canvas 2D. Tiada build step, tiada test runner.

## Global Constraints

- Semua kerja dalam satu fail: `index.html`. Tiada dependency baru.
- Bahasa UI: Melayu (ikut teks sedia ada).
- Mode Normal, Survival, papan ranking, avatar kepala, kesan flash/shake survival **mesti kekal berfungsi** tanpa perubahan tingkah laku.
- Nilai mode baru: string `'2player'`.
- Warna pemain: P1 = hijau (`#5ae678` / `rgba(90,230,120,...)`), P2 = oren (`#ff9a3c` / `rgba(255,154,60,...)`).
- Tempoh: 30/45/60 saat (guna semula `#duration-row`). Bom kekal ~25%. Penalti bom = tolak 30 markah + kombo pemain reset.
- **Pengesahan manual sahaja**: tiada rangka ujian. Setiap task diuji dengan buka `index.html` (perlu host HTTPS + kamera untuk ujian penuh; ujian UI/logik boleh guna `python -m http.server` di localhost yang dibenarkan getUserMedia).

---

### Task 1: State per-pemain + nilai mode 2player

**Files:**
- Modify: `index.html` (objek `state`, ~baris 310-324)

**Interfaces:**
- Produces: medan `state.p1Score, state.p2Score, state.p1Combo, state.p2Combo, state.p1MaxCombo, state.p2MaxCombo` (semua number, mula 0). `state.mode` kini boleh `'normal' | 'survival' | '2player'`.

- [ ] **Step 1: Tambah medan per-pemain pada objek `state`**

Dalam objek `state`, selepas baris `score:0, combo:0, maxCombo:0, timeLeft:45,`, tambah:

```javascript
    // 2-player mode: markah & kombo berasingan (P1 = tangan terbuka, P2 = genggam)
    p1Score:0, p2Score:0, p1Combo:0, p2Combo:0, p1MaxCombo:0, p2MaxCombo:0,
```

Kemas kini komen mode:
```javascript
    mode:'normal', // 'normal' | 'survival' | '2player'
```

- [ ] **Step 2: Pengesahan manual**

Buka `index.html` di pelayar, buka DevTools console, tiada ralat sintaks JS (halaman skrin mula tetap papar seperti biasa). Mode masih default 'normal'.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add per-player state fields for 2-player mode"
```

---

### Task 2: Butang mode "2 Pemain" + kawalan skrin mula

**Files:**
- Modify: `index.html` (HTML `#mode-row` ~baris 188-191; nota peraturan; JS handler mode-row ~baris 368-376)

**Interfaces:**
- Consumes: `state.mode` dari Task 1.
- Produces: butang `data-mode="2player"` dalam `#mode-row`; blok nota `#twoplayer-options`; handler mode dikemas kini untuk tiga pilihan.

- [ ] **Step 1: Tambah butang mode ketiga**

Dalam `#mode-row` (selepas butang survival):
```html
      <button class="dur-btn" data-mode="2player">👥 2 Pemain</button>
```

- [ ] **Step 2: Tambah nota peraturan 2P**

Selepas blok `#survival-options` (~baris 213), tambah:
```html
    <div id="twoplayer-options" class="rules hidden">
      <div>✋ <b>Player 1</b> tangkap buah dengan <b>tangan terbuka</b> (cincin hijau).</div>
      <div>✊ <b>Player 2</b> tangkap buah dengan <b>tangan genggam</b> (cincin oren).</div>
      <div>💣 Kena bom = tolak markah pemain berkenaan. ⏱️ Markah tertinggi menang.</div>
      <div class="distance-note" style="margin-top:6px;">Tempoh & kelajuan di atas digunakan untuk kedua-dua pemain.</div>
    </div>
```

- [ ] **Step 3: Kemas kini handler mode-row supaya kendali 3 mode**

Ganti handler `modeRow.querySelectorAll(...)` (~baris 368-376) dengan:
```javascript
  const twoplayerOptions = $('twoplayer-options');
  modeRow.querySelectorAll('.dur-btn').forEach(btn=>{
    btn.addEventListener('click', ()=>{
      modeRow.querySelectorAll('.dur-btn').forEach(b=>b.classList.remove('active'));
      btn.classList.add('active');
      state.mode = btn.dataset.mode;
      // Normal & 2-player kongsi kawalan tempoh + kelajuan.
      const showNormalControls = state.mode === 'normal' || state.mode === '2player';
      normalOptions.classList.toggle('hidden', !showNormalControls);
      survivalOptions.classList.toggle('hidden', state.mode !== 'survival');
      twoplayerOptions.classList.toggle('hidden', state.mode !== '2player');
    });
  });
```

- [ ] **Step 4: Pengesahan manual**

Buka `index.html`. Klik "👥 2 Pemain": butang jadi aktif, kawalan tempoh (30/45/60) + kelajuan kekal papar, dan nota peraturan 2P muncul. Klik "Normal": nota 2P hilang, kawalan tempoh kekal. Klik "Survival": kawalan tempoh hilang, nota survival papar.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add 2-player mode button and start-screen controls"
```

---

### Task 3: Reset state per-pemain di `beginGame()` + HUD toggle

**Files:**
- Modify: `index.html` (`beginGame()` ~baris 637-651; HTML HUD ~baris 223-231; CSS chip)

**Interfaces:**
- Consumes: medan state Task 1, nilai mode Task 2.
- Produces: elemen HUD `#hud-p1`, `#hud-p2` (span markah) & `#hud-p1combo`, `#hud-p2combo`; pemboleh ubah DOM `hudP1, hudP2, hudP1Combo, hudP2Combo` dalam blok `// ---------- dom ----------`.

- [ ] **Step 1: Tambah chip HUD untuk 2P**

Dalam `#hud` (selepas chip KOMBO sedia ada, ~baris 227), tambah dua chip tersembunyi:
```html
      <div class="chip hidden" id="hud-p1-chip" style="--pc:#5ae678;"><small style="color:#5ae678;">P1 ✋</small><span id="hud-p1">0</span><small id="hud-p1combo" style="opacity:.75;">kombo 0</small></div>
      <div class="chip hidden" id="hud-p2-chip" style="--pc:#ff9a3c;"><small style="color:#ff9a3c;">P2 ✊</small><span id="hud-p2">0</span><small id="hud-p2combo" style="opacity:.75;">kombo 0</small></div>
```

- [ ] **Step 2: Tambah rujukan DOM**

Selepas baris `const hudTimeChip = ...; hudLivesChip ...` (~baris 337), tambah:
```javascript
  const hudP1Chip = $('hud-p1-chip'), hudP2Chip = $('hud-p2-chip');
  const hudP1 = $('hud-p1'), hudP2 = $('hud-p2');
  const hudP1Combo = $('hud-p1combo'), hudP2Combo = $('hud-p2combo');
```

- [ ] **Step 3: Reset markah per-pemain & toggle HUD dalam `beginGame()`**

Dalam `beginGame()`, selepas `state.score = 0; state.combo = 0; state.maxCombo = 0;` tambah:
```javascript
    state.p1Score = 0; state.p2Score = 0;
    state.p1Combo = 0; state.p2Combo = 0;
    state.p1MaxCombo = 0; state.p2MaxCombo = 0;
```

Ganti blok toggle HUD sedia ada:
```javascript
    const survival = state.mode === 'survival';
    hudTimeChip.classList.toggle('hidden', survival);
    hudLivesChip.classList.toggle('hidden', !survival);
    if(survival) hudLives.textContent = '❤️'.repeat(state.lives);
```
dengan:
```javascript
    const survival = state.mode === 'survival';
    const twoP = state.mode === '2player';
    // Time chip tunjuk untuk Normal & 2P (kedua-dua ada had masa).
    hudTimeChip.classList.toggle('hidden', survival);
    hudLivesChip.classList.toggle('hidden', !survival);
    // Skor & kombo tunggal (chip pertama & KOMBO) disembunyikan dalam 2P.
    hudScore.parentElement.classList.toggle('hidden', twoP);
    hudCombo.parentElement.classList.toggle('hidden', twoP);
    hudP1Chip.classList.toggle('hidden', !twoP);
    hudP2Chip.classList.toggle('hidden', !twoP);
    if(survival) hudLives.textContent = '❤️'.repeat(state.lives);
```

Nota: `state.timeLeft = state.roundDuration;` sedia ada sudah pakai untuk 2P (tempoh dari `#duration-row`). Tiada perubahan diperlukan di situ.

- [ ] **Step 4: Pengesahan manual**

(Guna localhost dengan kamera atau host HTTPS.) Pilih "2 Pemain", mula main. HUD papar chip `P1 ✋ 0` dan `P2 ✊ 0` di kiri, `MASA` di tengah/kanan; chip SKOR & KOMBO tunggal tersembunyi. Pilih Normal/Survival: HUD kembali seperti asal. Console tiada ralat.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Reset per-player scores and toggle 2-player HUD in beginGame"
```

---

### Task 4: Logik tangkap 2-pemain di `update()`

**Files:**
- Modify: `index.html` (blok catch-check dalam `update()` ~baris 773-810; tambah fungsi `handleCatch2P`; kemas kini paparan HUD ~baris 819-826)

**Interfaces:**
- Consumes: `state.hands` (setiap satu ada `open`, `cx`, `cy`, `r`), medan skor per-pemain, `handleBomb` sedia ada.
- Produces: fungsi `handleCatch2P(player, f)` di mana `player` = `'p1'|'p2'`.

- [ ] **Step 1: Cabang logik tangkap untuk mode 2P**

Ganti blok catch-check sedia ada (dari `const openHands = state.hands.filter(h=>h.open);` sehingga penutup gelung `for(const hand of openHands){...}`) dengan versi bercabang:

```javascript
    // catch check
    const twoP = state.mode === '2player';
    // Mode single: hanya tangan terbuka menangkap. Mode 2P: SEMUA tangan menangkap
    // (terbuka = P1, genggam = P2).
    const catchingHands = twoP ? state.hands : state.hands.filter(h=>h.open);
    const caughtFruits = new Set();
    for(const hand of catchingHands){
      let best = null, bestDist = Infinity;
      for(const f of state.fruits){
        if(caughtFruits.has(f)) continue;
        const d = Math.hypot(f.x - hand.cx, f.y - hand.cy);
        if(d < (f.r + hand.r * 0.65) && d < bestDist){
          best = f; bestDist = d;
        }
      }
      if(best){
        caughtFruits.add(best);
        if(twoP){
          const player = hand.open ? 'p1' : 'p2';
          if(best.isBomb) handleBomb2P(player, best);
          else handleCatch2P(player, best);
        } else {
          if(best.isBomb) handleBomb(best);
          else handleCatch(best);
        }
      }
    }
```

- [ ] **Step 2: Tambah `handleCatch2P` dan `handleBomb2P`**

Selepas fungsi `handleBomb` sedia ada (~baris 902), tambah:
```javascript
  // 2-player: catch/bomb per pemain ('p1' = tangan terbuka, 'p2' = genggam).
  function handleCatch2P(player, f){
    const combo = state[player + 'Combo'];
    const mult = 1 + Math.floor(combo / 5) * 0.5;
    const gained = Math.round(f.pts * mult);
    state[player + 'Score'] += gained;
    state[player + 'Combo'] = combo + 1;
    if(state[player + 'Combo'] > state[player + 'MaxCombo']) state[player + 'MaxCombo'] = state[player + 'Combo'];
    addEffect(f.x, f.y, '+' + gained, player === 'p1' ? '#5ae678' : '#ff9a3c');
    beep(500 + Math.min(state[player + 'Combo'], 20) * 20, 0.12, 'triangle');
  }
  function handleBomb2P(player, f){
    state[player + 'Score'] = Math.max(0, state[player + 'Score'] - 30);
    state[player + 'Combo'] = 0;
    addEffect(f.x, f.y, '-30', '#ff5a3c');
    state.effects.push({x: f.x, y: f.y, boom: true, life: 0.5, startLife: 0.5});
    beep(90, 0.35, 'sawtooth');
  }
```

- [ ] **Step 3: Kemas kini paparan HUD per-frame**

Dalam `update()`, ganti blok kemas kini HUD sedia ada:
```javascript
    hudScore.textContent = state.score;
    hudCombo.textContent = state.combo;
    if(survival){
      hudLives.textContent = '❤️'.repeat(Math.max(0, state.lives));
    } else {
      hudTime.textContent = Math.ceil(state.timeLeft);
    }
```
dengan:
```javascript
    if(state.mode === '2player'){
      hudP1.textContent = state.p1Score;
      hudP2.textContent = state.p2Score;
      hudP1Combo.textContent = 'kombo ' + state.p1Combo;
      hudP2Combo.textContent = 'kombo ' + state.p2Combo;
      hudTime.textContent = Math.ceil(state.timeLeft);
    } else {
      hudScore.textContent = state.score;
      hudCombo.textContent = state.combo;
      if(survival){
        hudLives.textContent = '❤️'.repeat(Math.max(0, state.lives));
      } else {
        hudTime.textContent = Math.ceil(state.timeLeft);
      }
    }
```

Nota: bahagian `state.fruits = state.fruits.filter(...)` untuk buah tersasar kekal. Semakan `if(survival) lifeLost = true;` hanya terpakai untuk survival, jadi dalam 2P kombo tidak diputuskan oleh buah tersasar — tepat seperti spec.

- [ ] **Step 4: Pengesahan manual**

Main mode 2P. Tangan terbuka sentuh buah → markah P1 naik + teks `+N` hijau. Tangan genggam sentuh buah → markah P2 naik + teks `+N` oren. Tangan terbuka kena bom → P1 tolak 30 + letupan. Tangan genggam kena bom → P2 tolak 30. Mode Normal/Survival: tingkah laku tangkap kekal sama (genggam tak tangkap apa-apa). MASA mengira turun.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add per-player catch and bomb handling for 2-player mode"
```

---

### Task 5: Warna cincin tangan ikut pemain (render)

**Files:**
- Modify: `index.html` (gelung hand indicators dalam `render()` ~baris 958-973)

**Interfaces:**
- Consumes: `state.mode`, `hand.open`.

- [ ] **Step 1: Warnakan cincin ikut mode**

Ganti gelung hand indicators:
```javascript
    for(const hand of state.hands){
      ctx.save();
      ctx.beginPath();
      ctx.arc(hand.cx, hand.cy, hand.r * 0.65, 0, Math.PI * 2);
      if(hand.open){
        ctx.strokeStyle = 'rgba(90, 230, 120, 0.9)';
        ctx.lineWidth = 4;
      } else {
        ctx.strokeStyle = 'rgba(200, 200, 200, 0.7)';
        ctx.lineWidth = 3;
        ctx.setLineDash([6,6]);
      }
      ctx.stroke();
      ctx.restore();
    }
```
dengan:
```javascript
    const twoPRender = state.mode === '2player';
    for(const hand of state.hands){
      ctx.save();
      ctx.beginPath();
      ctx.arc(hand.cx, hand.cy, hand.r * 0.65, 0, Math.PI * 2);
      if(hand.open){
        // Terbuka = P1 (hijau), aktif dalam semua mode.
        ctx.strokeStyle = 'rgba(90, 230, 120, 0.9)';
        ctx.lineWidth = 4;
      } else if(twoPRender){
        // 2P: genggam = P2 (oren), aktif.
        ctx.strokeStyle = 'rgba(255, 154, 60, 0.95)';
        ctx.lineWidth = 4;
      } else {
        // Mode single: genggam = selamat/tak aktif (kelabu putus-putus).
        ctx.strokeStyle = 'rgba(200, 200, 200, 0.7)';
        ctx.lineWidth = 3;
        ctx.setLineDash([6,6]);
      }
      ctx.stroke();
      ctx.restore();
    }
```

- [ ] **Step 2: Pengesahan manual**

Mode 2P: tangan terbuka = cincin hijau tebal, tangan genggam = cincin oren tebal. Mode Normal: tangan genggam = cincin kelabu putus-putus (tidak berubah).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Color hand rings per player in 2-player mode"
```

---

### Task 6: Skrin keputusan 2-pemain + cabang `endGame()`

**Files:**
- Modify: `index.html` (tambah HTML `#screen-result-2p` selepas `#screen-gameover` ~baris 264; kemas kini `endGame()`; tambah rujukan DOM & handler butang; kemas kini `resetToStart`)

**Interfaces:**
- Consumes: `state.p1Score`, `state.p2Score`, `state.mode`, fungsi `resetToStart` & `stopGame`/aliran `endGame` sedia ada.
- Produces: skrin `#screen-result-2p`, fungsi `showResult2P()`.

- [ ] **Step 1: Tambah HTML skrin keputusan**

Selepas `#screen-gameover` (~baris 264, sebelum `#screen-leaderboard`), tambah:
```html
  <!-- 2-PLAYER RESULT SCREEN -->
  <div id="screen-result-2p" class="card hidden">
    <h1 id="result2p-title">🏆 Player 1 Menang!</h1>
    <div class="subtitle">Keputusan perlawanan</div>
    <div style="display:flex;gap:14px;justify-content:center;margin:18px 0;flex-wrap:wrap;">
      <div style="flex:1;min-width:120px;">
        <div style="font-weight:900;color:#3a9c4d;">P1 ✋</div>
        <div class="score-final" id="result2p-p1" style="color:#3a9c4d;">0</div>
      </div>
      <div style="flex:1;min-width:120px;">
        <div style="font-weight:900;color:#e07a1f;">P2 ✊</div>
        <div class="score-final" id="result2p-p2" style="color:#e07a1f;">0</div>
      </div>
    </div>
    <button class="big-btn" id="btn-result-again">🔁 Main Lagi</button>
  </div>
```

- [ ] **Step 2: Tambah rujukan DOM & pemboleh ubah skrin**

Selepas `const screenRank = $('screen-leaderboard');` (~baris 331), tambah:
```javascript
  const screenResult2P = $('screen-result-2p');
```

- [ ] **Step 3: Cabang `endGame()` untuk 2P**

`endGame()` bermula di baris ~1281:
```javascript
  function endGame(){
    state.running = false;
    distanceWarning.classList.add('hidden');
    leaveFullscreen();
    const survival = state.mode === 'survival';
    $('final-score').textContent = state.score;
    ...
```

Sisip cabang 2P SELEPAS `leaveFullscreen();` dan SEBELUM `const survival = ...` (supaya aliran gameover single-player tidak dijalankan langsung untuk 2P):
```javascript
    leaveFullscreen();
    if(state.mode === '2player'){ showResult2P(); return; }
    const survival = state.mode === 'survival';
```

Tambah fungsi `showResult2P` selepas fungsi `endGame` (selepas baris `}` penutupnya, ~baris 1298):
```javascript
  function showResult2P(){
    const p1 = state.p1Score, p2 = state.p2Score;
    const title = p1 > p2 ? '🏆 Player 1 Menang!'
                : p2 > p1 ? '🏆 Player 2 Menang!'
                : '🤝 Seri!';
    $('result2p-title').textContent = title;
    $('result2p-p1').textContent = p1;
    $('result2p-p2').textContent = p2;
    gameWrap.classList.add('hidden');
    screenResult2P.classList.remove('hidden');
  }
```

- [ ] **Step 4: Handler butang Main Lagi + pastikan `resetToStart` sembunyikan skrin baru**

Selepas `$('btn-play-again').addEventListener('click', resetToStart);` (~baris 461), tambah:
```javascript
  $('btn-result-again').addEventListener('click', resetToStart);
```

Dalam `resetToStart` (~baris 1373-1388), tambah baris menyembunyikan skrin keputusan baru bersama skrin lain. Cari:
```javascript
    screenRank.classList.add('hidden');
    screenOver.classList.add('hidden');
    gameWrap.classList.add('hidden');
    screenStart.classList.remove('hidden');
```
dan tambah satu baris:
```javascript
    screenRank.classList.add('hidden');
    screenOver.classList.add('hidden');
    screenResult2P.classList.add('hidden');
    gameWrap.classList.add('hidden');
    screenStart.classList.remove('hidden');
```

- [ ] **Step 5: Pengesahan manual**

Main mode 2P sehingga MASA = 0. Skrin keputusan muncul: tajuk betul (P1/P2 menang atau Seri ikut markah), papar kedua-dua markah dengan warna. Klik "Main Lagi" → kembali ke skrin mula, tiada baki skrin bertindih. Main mode Normal sehingga tamat → skrin gameover + input nama kekal seperti asal (tidak terjejas).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add 2-player result screen and endGame branch"
```

---

### Task 7: Semakan integrasi penuh + regresi mode sedia ada

**Files:**
- Modify: `index.html` (pembetulan kecil jika ujian dedah isu)

- [ ] **Step 1: Regresi mode Normal**

Main penuh mode Normal (45s): tangkap buah tangan terbuka tambah markah, genggam tak buat apa-apa, bom tolak markah, kombo & papan ranking berfungsi, skrin gameover + simpan nama berfungsi.

- [ ] **Step 2: Regresi mode Survival**

Main mode Survival: nyawa berkurang bila buah tersasar/kena bom, kesan flash+shake+bunyi muncul, tamat bila 0 nyawa.

- [ ] **Step 3: Ujian penuh mode 2P**

Uji ketiga-tiga tempoh (30/45/60). Dua orang (atau satu orang guna dua tangan: satu buka, satu genggam): markah masing-masing naik betul, HUD kemas kini, cincin warna betul, bom tolak pemain betul, skrin keputusan papar pemenang tepat, Main Lagi berfungsi. Kalibrasi jarak & amaran "terlalu dekat" masih berfungsi.

- [ ] **Step 4: Baiki sebarang isu dijumpai, kemudian commit**

```bash
git add index.html
git commit -m "Fix integration issues found during 2-player mode testing"
```

(Langkau commit jika tiada isu dijumpai.)
