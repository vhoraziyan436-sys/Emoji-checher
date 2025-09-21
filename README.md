# Emoji-chechervictory..  
Hi my own game
="hi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Emoji Catcher — Funny Game</title>
  <style>
    :root{
      --bg:#0f172a;
      --panel:#0b1220;
      --accent:#ffd166;
      --text:#e6eef8;
    }
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;}
    body{
      background: linear-gradient(180deg, #08101f 0%, #071326 60%);
      color:var(--text);
      display:flex;
      align-items:center;
      justify-content:center;
      padding:20px;
    }
    .game {
      width:360px;
      max-width:calc(100vw - 40px);
      background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
      border:1px solid rgba(255,255,255,0.04);
      border-radius:12px;
      padding:12px;
      box-shadow: 0 8px 30px rgba(2,6,23,0.6);
    }
    .header{
      display:flex;justify-content:space-between;align-items:center;margin-bottom:8px;
    }
    .title{font-weight:700;font-size:16px;color:var(--accent)}
    .score{font-weight:600}
    canvas{
      width:100%;
      height:420px;
      background: linear-gradient(#061322, #0b1b2a);
      display:block;border-radius:8px;touch-action:none;
    }
    .controls{display:flex;gap:8px;margin-top:8px;}
    button{
      flex:1;padding:8px;border-radius:8px;border:0;background:var(--panel);color:var(--text);
      cursor:pointer;font-weight:600;
      box-shadow: 0 4px 10px rgba(2,6,23,0.6);
    }
    button.primary{background:var(--accent);color:#071326}
    .hint{font-size:12px;color:#9fb0cc;margin-top:8px;text-align:center}
    .footer{font-size:12px;color:#9fb0cc;margin-top:6px;text-align:center}
  </style>
</head>
<body>
  <div class="game" role="application" aria-label="Emoji Catcher Game">
    <div class="header">
      <div class="title">😂 Emoji Catcher</div>
      <div class="score">Score: <span id="score">0</span></div>
    </div>

    <canvas id="c" width="720" height="840" aria-label="Game canvas"></canvas>

    <div class="controls" aria-hidden="false">
      <button id="startBtn" class="primary">Start / Restart</button>
      <button id="pauseBtn">Pause</button>
    </div>

    <div class="hint">Use ← → or A / D to move. Tap left/right on mobile.</div>
    <div class="footer">Catch the good emojis, avoid the rotten ones! 🍎💥</div>
  </div>

<script>
/*
  Emoji Catcher
  - Move the catcher left/right to catch falling emojis.
  - Good emojis (+1), rotten emojis (-2).
  - Difficulty increases with score.
*/

(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d', { alpha: false });
  const startBtn = document.getElementById('startBtn');
  const pauseBtn = document.getElementById('pauseBtn');
  const scoreEl = document.getElementById('score');

  // High-res scaling
  function resizeCanvas() {
    const dpr = Math.min(window.devicePixelRatio || 1, 2);
    canvas.width = Math.floor(canvas.clientWidth * dpr);
    canvas.height = Math.floor(canvas.clientHeight * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);

  // Game state
  let running = false;
  let paused = false;
  let score = 0;
  let lastTime = 0;
  let spawnTimer = 0;
  let spawnInterval = 1200; // ms
  let entities = [];

  // Player / catcher
  const player = {
    x: 0.5, // normalized (0..1)
    width: 0.18, // normalized
    y: 0.92, // normalized
    speed: 1.8 // normalized per second
  };

  // Emojis list — good and rotten
  const EMOJIS = [
    { char: '🍎', good: true },
    { char: '🍌', good: true },
    { char: '🍕', good: true },
    { char: '🍩', good: true },
    { char: '🍇', good: true },
    { char: '💣', good: false },
    { char: '💀', good: false },
    { char: '🪲', good: false },
    { char: '🥴', good: false }
  ];

  // Input
  const input = { left:false, right:false };
  window.addEventListener('keydown', e => {
    if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') input.left = true;
    if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') input.right = true;
  });
  window.addEventListener('keyup', e => {
    if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') input.left = false;
    if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') input.right = false;
  });

  // Touch: split canvas left/right
  canvas.addEventListener('touchstart', e => {
    const t = e.touches[0];
    const r = canvas.getBoundingClientRect();
    const cx = (t.clientX - r.left) / r.width;
    if (cx < 0.5) input.left = true;
    else input.right = true;
  }, { passive: true });
  canvas.addEventListener('touchend', e => {
    input.left = false; input.right = false;
  });

  // UI buttons
  startBtn.addEventListener('click', startGame);
  pauseBtn.addEventListener('click', () => {
    if (!running) return;
    paused = !paused;
    pauseBtn.textContent = paused ? 'Resume' : 'Pause';
    if (!paused) lastTime = performance.now(); // avoid big dt
  });

  function startGame() {
    running = true;
    paused = false;
    pauseBtn.textContent = 'Pause';
    score = 0;
    entities = [];
    spawnInterval = 1200;
    player.x = 0.5;
    lastTime = performance.now();
    spawnTimer = 0;
    scoreEl.textContent = score;
    requestAnimationFrame(loop);
  }

  function rand(min,max){ return Math.random()*(max-min)+min; }

  function spawnEmoji() {
    const pick = EMOJIS[Math.floor(Math.random()*EMOJIS.length)];
    const speedBase = 0.16 + Math.min(0.5, score * 0.01); // increase with score
    const e = {
      char: pick.char,
      good: pick.good,
      x: Math.random() * 0.9 + 0.05, // normalized x
      y: -0.05,
      vy: speedBase + Math.random()*0.08,
      rot: rand(-0.8,0.8),
      rotSpeed: rand(-1,1)
    };
    entities.push(e);
  }

  // Collision: player is rectangle; emoji is small circle-ish area
  function checkCollision(e) {
    const px = player.x;
    const pw = player.width;
    const cx = e.x;
    const cy = e.y;
    const catcherTop = player.y - 0.04;
    const catcherLeft = px - pw/2;
    const catcherRight = px + pw/2;
    // If emoji y within catcher area and x overlaps
    return (cy >= catcherTop && cy <= player.y + 0.06) &&
           (cx >= catcherLeft && cx <= catcherRight);
  }

  function update(dt) {
    // player movement
    let dir = 0;
    if (input.left) dir -= 1;
    if (input.right) dir += 1;
    player.x += dir * player.speed * dt;
    player.x = Math.max(0 + player.width/2, Math.min(1 - player.width/2, player.x));

    // spawn logic
    spawnTimer += dt*1000;
    const interval = Math.max(400, spawnInterval - Math.floor(score/5)*40);
    if (spawnTimer >= interval) {
      spawnTimer = 0;
      spawnEmoji();
    }

    // update entities
    for (let i = entities.length-1; i >= 0; i--) {
      const ent = entities[i];
      ent.y += ent.vy * dt;
      ent.rot += ent.rotSpeed * dt;

      // caught?
      if (checkCollision(ent)) {
        if (ent.good) score += 1;
        else score -= 2;
        // small clamp
        score = Math.max(0, score);
        entities.splice(i,1);
        scoreEl.textContent = score;
        continue;
      }

      // missed and fell off bottom
      if (ent.y > 1.15) {
        // penalty for missing good ones
        if (ent.good) {
          score = Math.max(0, score - 1);
          scoreEl.textContent = score;
        }
        entities.splice(i,1);
      }
    }
  }

  function draw() {
    // clear
    const w = canvas.clientWidth;
    const h = canvas.clientHeight;
    ctx.clearRect(0,0,w,h);

    // background subtle grid
    ctx.fillStyle = '#071727';
    ctx.fillRect(0,0,w,h);

    // draw entities
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    const baseFontSize = Math.min(w, h) * 0.06;
    for (const ent of entities) {
      const ex = ent.x * w;
      const ey = ent.y * h;
      ctx.save();
      ctx.translate(ex, ey);
      ctx.rotate(ent.rot);
      ctx.font = `${baseFontSize}px serif`;
      // shadow
      ctx.shadowColor = 'rgba(0,0,0,0.5)';
      ctx.shadowBlur = 8;
      ctx.fillStyle = ent.good ? '#fff' : '#fff';
      ctx.fillText(ent.char, 0, 0);
      ctx.restore();
    }

    // draw catcher (rounded rectangle with emoji face)
    const px = player.x * w;
    const pw = player.width * w;
    const py = player.y * h;
    // strip
    ctx.save();
    // body
    const radius = 12;
    ctx.fillStyle = '#1f6f8b';
    roundRect(ctx, px - pw/2, py - 16, pw, 32, radius);
    ctx.fill();
    // stripe highlight
    ctx.fillStyle = 'rgba(255,255,255,0.08)';
    roundRect(ctx, px - pw/2 + 6, py - 10, pw - 12, 8, 6);
    ctx.fill();
    // face emoji on catcher
    ctx.font = `${Math.floor(14 + pw*0.05)}px serif`;
    ctx.fillStyle = '#fff';
    ctx.fillText('😎', px, py);
    ctx.restore();

    // bottom floor
    ctx.fillStyle = 'rgba(255,255,255,0.03)';
    ctx.fillRect(0, h - 6, w, 6);
  }

  function roundRect(ctx, x, y, w, h, r) {
    const rr = Math.min(r, w/2, h/2);
    ctx.beginPath();
    ctx.moveTo(x+rr, y);
    ctx.arcTo(x+w, y, x+w, y+h, rr);
    ctx.arcTo(x+w, y+h, x, y+h, rr);
    ctx.arcTo(x, y+h, x, y, rr);
    ctx.arcTo(x, y, x+w, y, rr);
    ctx.closePath();
  }

  function loop(t) {
    if (!running) return;
    if (paused) {
      lastTime = t;
      requestAnimationFrame(loop);
      return;
    }
    const dt = Math.min(0.05, (t - lastTime) / 1000); // cap dt to avoid jump
    lastTime = t;

    // update difficultly slowly
    spawnInterval = Math.max(400, 1200 - score*8);

    update(dt);
    draw();

    // small game-over idea: if score is big and you want victory...
    requestAnimationFrame(loop);
  }

  // initial draw (before start)
  draw();

  // Friendly tips on long-press mobile
  canvas.addEventListener('contextmenu', e => e.preventDefault());
})();
</script>
</body>
</html><!doctype html>
<html lang="hi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Emoji Catcher — Funny Game</title>
  <style>
    :root{
      --bg:#0f172a;
      --panel:#0b1220;
      --accent:#ffd166;
      --text:#e6eef8;
    }
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;}
    body{
      background: linear-gradient(180deg, #08101f 0%, #071326 60%);
      color:var(--text);
      display:flex;
      align-items:center;
      justify-content:center;
      padding:20px;
    }
    .game {
      width:360px;
      max-width:calc(100vw - 40px);
      background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
      border:1px solid rgba(255,255,255,0.04);
      border-radius:12px;
      padding:12px;
      box-shadow: 0 8px 30px rgba(2,6,23,0.6);
    }
    .header{
      display:flex;justify-content:space-between;align-items:center;margin-bottom:8px;
    }
    .title{font-weight:700;font-size:16px;color:var(--accent)}
    .score{font-weight:600}
    canvas{
      width:100%;
      height:420px;
      background: linear-gradient(#061322, #0b1b2a);
      display:block;border-radius:8px;touch-action:none;
    }
    .controls{display:flex;gap:8px;margin-top:8px;}
    button{
      flex:1;padding:8px;border-radius:8px;border:0;background:var(--panel);color:var(--text);
      cursor:pointer;font-weight:600;
      box-shadow: 0 4px 10px rgba(2,6,23,0.6);
    }
    button.primary{background:var(--accent);color:#071326}
    .hint{font-size:12px;color:#9fb0cc;margin-top:8px;text-align:center}
    .footer{font-size:12px;color:#9fb0cc;margin-top:6px;text-align:center}
  </style>
</head>
<body>
  <div class="game" role="application" aria-label="Emoji Catcher Game">
    <div class="header">
      <div class="title">😂 Emoji Catcher</div>
      <div class="score">Score: <span id="score">0</span></div>
    </div>

    <canvas id="c" width="720" height="840" aria-label="Game canvas"></canvas>

    <div class="controls" aria-hidden="false">
      <button id="startBtn" class="primary">Start / Restart</button>
      <button id="pauseBtn">Pause</button>
    </div>

    <div class="hint">Use ← → or A / D to move. Tap left/right on mobile.</div>
    <div class="footer">Catch the good emojis, avoid the rotten ones! 🍎💥</div>
  </div>

<script>
/*
  Emoji Catcher
  - Move the catcher left/right to catch falling emojis.
  - Good emojis (+1), rotten emojis (-2).
  - Difficulty increases with score.
*/

(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d', { alpha: false });
  const startBtn = document.getElementById('startBtn');
  const pauseBtn = document.getElementById('pauseBtn');
  const scoreEl = document.getElementById('score');

  // High-res scaling
  function resizeCanvas() {
    const dpr = Math.min(window.devicePixelRatio || 1, 2);
    canvas.width = Math.floor(canvas.clientWidth * dpr);
    canvas.height = Math.floor(canvas.clientHeight * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);

  // Game state
  let running = false;
  let paused = false;
  let score = 0;
  let lastTime = 0;
  let spawnTimer = 0;
  let spawnInterval = 1200; // ms
  let entities = [];

  // Player / catcher
  const player = {
    x: 0.5, // normalized (0..1)
    width: 0.18, // normalized
    y: 0.92, // normalized
    speed: 1.8 // normalized per second
  };

  // Emojis list — good and rotten
  const EMOJIS = [
    { char: '🍎', good: true },
    { char: '🍌', good: true },
    { char: '🍕', good: true },
    { char: '🍩', good: true },
    { char: '🍇', good: true },
    { char: '💣', good: false },
    { char: '💀', good: false },
    { char: '🪲', good: false },
    { char: '🥴', good: false }
  ];

  // Input
  const input = { left:false, right:false };
  window.addEventListener('keydown', e => {
    if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') input.left = true;
    if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') input.right = true;
  });
  window.addEventListener('keyup', e => {
    if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') input.left = false;
    if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') input.right = false;
  });

  // Touch: split canvas left/right
  canvas.addEventListener('touchstart', e => {
    const t = e.touches[0];
    const r = canvas.getBoundingClientRect();
    const cx = (t.clientX - r.left) / r.width;
    if (cx < 0.5) input.left = true;
    else input.right = true;
  }, { passive: true });
  canvas.addEventListener('touchend', e => {
    input.left = false; input.right = false;
  });

  // UI buttons
  startBtn.addEventListener('click', startGame);
  pauseBtn.addEventListener('click', () => {
    if (!running) return;
    paused = !paused;
    pauseBtn.textContent = paused ? 'Resume' : 'Pause';
    if (!paused) lastTime = performance.now(); // avoid big dt
  });

  function startGame() {
    running = true;
    paused = false;
    pauseBtn.textContent = 'Pause';
    score = 0;
    entities = [];
    spawnInterval = 1200;
    player.x = 0.5;
    lastTime = performance.now();
    spawnTimer = 0;
    scoreEl.textContent = score;
    requestAnimationFrame(loop);
  }

  function rand(min,max){ return Math.random()*(max-min)+min; }

  function spawnEmoji() {
    const pick = EMOJIS[Math.floor(Math.random()*EMOJIS.length)];
    const speedBase = 0.16 + Math.min(0.5, score * 0.01); // increase with score
    const e = {
      char: pick.char,
      good: pick.good,
      x: Math.random() * 0.9 + 0.05, // normalized x
      y: -0.05,
      vy: speedBase + Math.random()*0.08,
      rot: rand(-0.8,0.8),
      rotSpeed: rand(-1,1)
    };
    entities.push(e);
  }

  // Collision: player is rectangle; emoji is small circle-ish area
  function checkCollision(e) {
    const px = player.x;
    const pw = player.width;
    const cx = e.x;
    const cy = e.y;
    const catcherTop = player.y - 0.04;
    const catcherLeft = px - pw/2;
    const catcherRight = px + pw/2;
    // If emoji y within catcher area and x overlaps
    return (cy >= catcherTop && cy <= player.y + 0.06) &&
           (cx >= catcherLeft && cx <= catcherRight);
  }

  function update(dt) {
    // player movement
    let dir = 0;
    if (input.left) dir -= 1;
    if (input.right) dir += 1;
    player.x += dir * player.speed * dt;
    player.x = Math.max(0 + player.width/2, Math.min(1 - player.width/2, player.x));

    // spawn logic
    spawnTimer += dt*1000;
    const interval = Math.max(400, spawnInterval - Math.floor(score/5)*40);
    if (spawnTimer >= interval) {
      spawnTimer = 0;
      spawnEmoji();
    }

    // update entities
    for (let i = entities.length-1; i >= 0; i--) {
      const ent = entities[i];
      ent.y += ent.vy * dt;
      ent.rot += ent.rotSpeed * dt;

      // caught?
      if (checkCollision(ent)) {
        if (ent.good) score += 1;
        else score -= 2;
        // small clamp
        score = Math.max(0, score);
        entities.splice(i,1);
        scoreEl.textContent = score;
        continue;
      }

      // missed and fell off bottom
      if (ent.y > 1.15) {
        // penalty for missing good ones
        if (ent.good) {
          score = Math.max(0, score - 1);
          scoreEl.textContent = score;
        }
        entities.splice(i,1);
      }
    }
  }

  function draw() {
    // clear
    const w = canvas.clientWidth;
    const h = canvas.clientHeight;
    ctx.clearRect(0,0,w,h);

    // background subtle grid
    ctx.fillStyle = '#071727';
    ctx.fillRect(0,0,w,h);

    // draw entities
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    const baseFontSize = Math.min(w, h) * 0.06;
    for (const ent of entities) {
      const ex = ent.x * w;
      const ey = ent.y * h;
      ctx.save();
      cursor:pointer;font-weight:600;
      box-shadow: 0 4px 10px rgba(2,6,23,0.6);
    }
    button.primary{background:var(--accent);color:#071326}
    .hint{font-size:12px;color:#9fb0cc;margin-top:8px;text-align:center}
    .footer{font-size:12px;color:#9fb0cc;margin-top:6px;text-align:center}
  </style>
</head>
<body>
  <div class="game" role="application" aria-label="Emoji Catcher Game">
    <div class="header">
      <div class="title">😂 Emoji Catcher</div>
      <div class="score">Score: <span id="score">0</span></div>
    </div>

    <canvas id="c" width="720" height="840" aria-label="Game canvas"></canvas>

    <div class="controls" aria-hidden="false">
      <button id="startBtn" class="primary">Start / Restart</button>
      <button id="pauseBtn">Pause</button>
    </div>

    <div class="hint">Use ← → or A / D to move. Tap left/right on mobile.</div>
    <div class="footer">Catch the good emojis, avoid the rotten ones! 🍎💥</div>
  </div>

<script>
/*
  Emoji Catcher
  - Move the catcher left/right to catch falling emojis.
  - Good emojis (+1), rotten emojis (-2).
  - Difficulty increases with score.
*/

(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d', { alpha: false });
  const startBtn = document.getElementById('startBtn');
  const pauseBtn = document.getElementById('pauseBtn');
  const scoreEl = document.getElementById('score');

  // High-res scaling
  function resizeCanvas() {
    const dpr = Math.min(window.devicePixelRatio || 1, 2);
    canvas.width = Math.floor(canvas.clientWidth * dpr);
    canvas.height = Math.floor(canvas.clientHeight * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);

  // Game state
  let running = false;
  let paused = false;
  let score = 0;
  let lastTime = 0;
  let spawnTimer = 0;
  let spawnInterval = 1200; // ms
  let entities = [];

  // Player / catcher
  const player = {
    x: 0.5, // normalized (0..1)
    width: 0.18, // normalized
    y: 0.92, // normalized
    speed: 1.8 // normalized per second
  };

  // Emojis list — good and rotten
  const EMOJIS = [
    { char: '🍎', good: true },
    { char: '🍌', good: true },
    { char: '🍕', good: true },
    { char: '🍩', good: true },
    { char: '🍇', good: true },
    { char: '💣', good: false },
    { char: '💀', good: false },
    { char: '🪲', good: false },
    { char: '🥴', good: false }
  ];

  // Input
  const input = { left:false, right:false };
  window.addEventListener('keydown', e => {
    if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') input.left = true;
    if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') input.right = true;
  });
  window.addEventListener('keyup', e => {
    if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') input.left = false;
    if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') input.right = false;
  });

  // Touch: split canvas left/right
  canvas.addEventListener('touchstart', e => {
    const t = e.touches[0];
    const r = canvas.getBoundingClientRect();
    const cx = (t.clientX - r.left) / r.width;
    if (cx < 0.5) input.left = true;
    else input.right = true;
  }, { passive: true });
  canvas.addEventListener('touchend', e => {
    input.left = false; input.right = false;
  });

  // UI buttons
  startBtn.addEventListener('click', startGame);
  pauseBtn.addEventListener('click', () => {
    if (!running) return;
    paused = !paused;
    pauseBtn.textContent = paused ? 'Resume' : 'Pause';
    if (!paused) lastTime = performance.now(); // avoid big dt
  });

  function startGame() {
    running = true;
    paused = false;
    pauseBtn.textContent = 'Pause';
    score = 0;
    entities = [];
    spawnInterval = 1200;
    player.x = 0.5;
    lastTime = performance.now();
    spawnTimer = 0;
    scoreEl.textContent = score;
    requestAnimationFrame(loop);
  }

  function rand(min,max){ return Math.random()*(max-min)+min; }

  function spawnEmoji() {
    const pick = EMOJIS[Math.floor(Math.random()*EMOJIS.length)];
    const speedBase = 0.16 + Math.min(0.5, score * 0.01); // increase with score
    const e = {
      char: pick.char,
      good: pick.good,
      x: Math.random() * 0.9 + 0.05, // normalized x
      y: -0.05,
      vy: speedBase + Math.random()*0.08,
      rot: rand(-0.8,0.8),
      rotSpeed: rand(-1,1)
    };
    entities.push(e);
  }

  // Collision: player is rectangle; emoji is small circle-ish area
  function checkCollision(e) {
    const px = player.x;
    const pw = player.width;
    const cx = e.x;
    const cy = e.y;
    const catcherTop = player.y - 0.04;
    const catcherLeft = px - pw/2;
    const catcherRight = px + pw/2;
    // If emoji y within catcher area and x overlaps
    return (cy >= catcherTop && cy <= player.y + 0.06) &&
           (cx >= catcherLeft && cx <= catcherRight);
  }

  function update(dt) {
    // player movement
    let dir = 0;
    if (input.left) dir -= 1;
    if (input.right) dir += 1;
    player.x += dir * player.speed * dt;
    player.x = Math.max(0 + player.width/2, Math.min(1 - player.width/2, player.x));

    // spawn logic
    spawnTimer += dt*1000;
    const interval = Math.max(400, spawnInterval - Math.floor(score/5)*40);
    if (spawnTimer >= interval) {
      spawnTimer = 0;
      spawnEmoji();
    }

    // update entities
    for (let i = entities.length-1; i >= 0; i--) {
      const ent = entities[i];
      ent.y += ent.vy * dt;
      ent.rot += ent.rotSpeed * dt;

      // caught?
      if (checkCollision(ent)) {
        if (ent.good) score += 1;
        else score -= 2;
        // small clamp
        score = Math.max(0, score);
        entities.splice(i,1);
        scoreEl.textContent = score;
        continue;
      }

      // missed and fell off bottom
      if (ent.y > 1.15) {
        // penalty for missing good ones
        if (ent.good) {
          score = Math.max(0, score - 1);
          scoreEl.textContent = score;
        }
        entities.splice(i,1);
      }
    }
  }

  function draw() {
    // clear
    const w = canvas.clientWidth;
    const h = canvas.clientHeight;
    ctx.clearRect(0,0,w,h);

    // background subtle grid
    ctx.fillStyle = '#071727';
    ctx.fillRect(0,0,w,h);

    // draw entities
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    const baseFontSize = Math.min(w, h) * 0.06;
    for (const ent of entities) {
      const ex = ent.x * w;
      const ey = ent.y * h;
      ctx.save();
      ctx.translate(ex, ey);
      ctx.rotate(ent.rot);
      ctx.font = `${baseFontSize}px serif`;
      // shadow
      ctx.shadowColor = 'rgba(0,0,0,0.5)';
      ctx.shadowBlur = 8;
      ctx.fillStyle = ent.good ? '#fff' : '#fff';
      ctx.fillText(ent.char, 0, 0);
      ctx.restore();
    }

    // draw catcher (rounded rectangle with emoji face)
    const px = player.x * w;
    const pw = player.width * w;
    const py = player.y * h;
    // strip
    ctx.save();
    // body
    const radius = 12;
    ctx.fillStyle = '#1f6f8b';
    roundRect(ctx, px - pw/2, py - 16, pw, 32, radius);
    ctx.fill();
    // stripe highlight
    ctx.fillStyle = 'rgba(255,255,255,0.08)';
    roundRect(ctx, px - pw/2 + 6, py - 10, pw - 12, 8, 6);
    ctx.fill();
    // face emoji on catcher
    ctx.font = `${Math.floor(14 + pw*0.05)}px serif`;
    ctx.fillStyle = '#fff';
    ctx.fillText('😎', px, py);
    ctx.restore();

    // bottom floor
    ctx.fillStyle = 'rgba(255,255,255,0.03)';
    ctx.fillRect(0, h - 6, w, 6);
  }

  function roundRect(ctx, x, y, w, h, r) {
    const rr = Math.min(r, w/2, h/2);
    ctx.beginPath();
    ctx.moveTo(x+rr, y);
    ctx.arcTo(x+w, y, x+w, y+h, rr);
    ctx.arcTo(x+w, y+h, x, y+h, rr);
    ctx.arcTo(x, y+h, x, y, rr);
    ctx.arcTo(x, y, x+w, y, rr);
    ctx.closePath();
  }

  function loop(t) {
    if (!running) return;
    if (paused) {
      lastTime = t;
      requestAnimationFrame(loop);
      return;
    }
    const dt = Math.min(0.05, (t - lastTime) / 1000); // cap dt to avoid jump
    lastTime = t;

    // update difficultly slowly
    spawnInterval = Math.max(400, 1200 - score*8);

    update(dt);
    draw();

    // small game-over idea: if score is big and you want victory...
    requestAnimationFrame(loop);
  }

  // initial draw (before start)
  draw();

  // Friendly tips on long-press mobile
  canvas.addEventListener('contextmenu', e => e.preventDefault());
})();
</script>
</body>
</html><!doctype html>
<html lang="hi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Emoji Catcher — Funny Game</title>
  <style>
    :root{
      --bg:#0f172a;
      --panel:#0b1220;
      --accent:#ffd166;
      --text:#e6eef8;
    }
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;}
    body{
      background: linear-gradient(180deg, #08101f 0%, #071326 60%);
      color:var(--text);
      display:flex;
      align-items:center;
      justify-content:center;
      padding:20px;
    }
    .game {
      width:360px;
      max-width:calc(100vw - 40px);
      background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
      border:1px solid rgba(255,255,255,0.04);
      border-radius:12px;
      padding:12px;
      box-shadow: 0 8px 30px rgba(2,6,23,0.6);
    }
    .header{
      display:flex;justify-content:space-between;align-items:center;margin-bottom:8px;
    }
    .title{font-weight:700;font-size:16px;color:var(--accent)}
    .score{font-weight:600}
    canvas{
      width:100%;
      height:420px;
      background: linear-gradient(#061322, #0b1b2a);
      display:block;border-radius:8px;touch-action:none;
    }
    .controls{display:flex;gap:8px;margin-top:8px;}
    button{
      flex:1;padding:8px;border-radius:8px;border:0;background:var(--panel);color:var(--text);
      cursor:pointer;font-weight:600;
      box-shadow: 0 4px 10px rgba(2,6,23,0.6);
    }
    button.primary{background:var(--accent);color:#071326}
    .hint{font-size:12px;color:#9fb0cc;margin-top:8px;text-align:center}
    .footer{font-size:12px;color:#9fb0cc;margin-top:6px;text-align:center}
  </style>
</head>
<body>
  <div class="game" role="application" aria-label="Emoji Catcher Game">
    <div class="header">
      <div class="title">😂 Emoji Catcher</div>
      <div class="score">Score: <span id="score">0</span></div>
    </div>

    <canvas id="c" width="720" height="840" aria-label="Game canvas"></canvas>

    <div class="controls" aria-hidden="false">
      <button id="startBtn" class="primary">Start / Restart</button>
      <button id="pauseBtn">Pause</button>
    </div>

    <div class="hint">Use ← → or A / D to move. Tap left/right on mobile.</div>
    <div class="footer">Catch the good emojis, avoid the rotten ones! 🍎💥</div>
  </div>

<script>
/*
  Emoji Catcher
  - Move the catcher left/right to catch falling emojis.
  - Good emojis (+1), rotten emojis (-2).
  - Difficulty increases with score.
*/

(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d', { alpha: false });
  const startBtn = document.getElementById('startBtn');
  const pauseBtn = document.getElementById('pauseBtn');
  const scoreEl = document.getElementById('score');

  // High-res scaling
  function resizeCanvas() {
    const dpr = Math.min(window.devicePixelRatio || 1, 2);
    canvas.width = Math.floor(canvas.clientWidth * dpr);
    canvas.height = Math.floor(canvas.clientHeight * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);

  // Game state
  let running = false;
  let paused = false;
  let score = 0;
  let lastTime = 0;
  let spawnTimer = 0;
  let spawnInterval = 1200; // ms
  let entities = [];

  // Player / catcher
  const player = {
    x: 0.5, // normalized (0..1)
    width: 0.18, // normalized
    y: 0.92, // normalized
    speed: 1.8 // normalized per second
  };

  // Emojis list — good and rotten
  const EMOJIS = [
    { char: '🍎', good: true },
    { char: '🍌', good: true },
    { char: '🍕', good: true },
    { char: '🍩', good: true },
    { char: '🍇', good: true },
    { char: '💣', good: false },
    { char: '💀', good: false },
    { char: '🪲', good: false },
    { char: '🥴', good: false }
  ];

  // Input
  const input = { left:false, right:false };
  window.addEventListener('keydown', e => {
    if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') input.left = true;
    if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') input.right = true;
  });
  window.addEventListener('keyup', e => {
    if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') input.left = false;
    if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') input.right = false;
  });

  // Touch: split canvas left/right
  canvas.addEventListener('touchstart', e => {
    const t = e.touches[0];
    const r = canvas.getBoundingClientRect();
    const cx = (t.clientX - r.left) / r.width;
    if (cx < 0.5) input.left = true;
    else input.right = true;
  }, { passive: true });
  canvas.addEventListener('touchend', e => {
    input.left = false; input.right = false;
  });

  // UI buttons
  startBtn.addEventListener('click', startGame);
  pauseBtn.addEventListener('click', () => {
    if (!running) return;
    paused = !paused;
    pauseBtn.textContent = paused ? 'Resume' : 'Pause';
    if (!paused) lastTime = performance.now(); // avoid big dt
  });

  function startGame() {
    running = true;
    paused = false;
    pauseBtn.textContent = 'Pause';
    score = 0;
    entities = [];
    spawnInterval = 1200;
    player.x = 0.5;
    lastTime = performance.now();
    spawnTimer = 0;
    scoreEl.textContent = score;
    requestAnimationFrame(loop);
  }

  function rand(min,max){ return Math.random()*(max-min)+min; }

  function spawnEmoji() {
    const pick = EMOJIS[Math.floor(Math.random()*EMOJIS.length)];
    const speedBase = 0.16 + Math.min(0.5, score * 0.01); // increase with score
    const e = {
      char: pick.char,
      good: pick.good,
      x: Math.random() * 0.9 + 0.05, // normalized x
      y: -0.05,
      vy: speedBase + Math.random()*0.08,
      rot: rand(-0.8,0.8),
      rotSpeed: rand(-1,1)
    };
    entities.push(e);
  }

  // Collision: player is rectangle; emoji is small circle-ish area
  function checkCollision(e) {
    const px = player.x;
    const pw = player.width;
    const cx = e.x;
    const cy = e.y;
    const catcherTop = player.y - 0.04;
    const catcherLeft = px - pw/2;
    const catcherRight = px + pw/2;
    // If emoji y within catcher area and x overlaps
    return (cy >= catcherTop && cy <= player.y + 0.06) &&
           (cx >= catcherLeft && cx <= catcherRight);
  }

  function update(dt) {
    // player movement
    let dir = 0;
    if (input.left) dir -= 1;
    if (input.right) dir += 1;
    player.x += dir * player.speed * dt;
    player.x = Math.max(0 + player.width/2, Math.min(1 - player.width/2, player.x));

    // spawn logic
    spawnTimer += dt*1000;
    const interval = Math.max(400, spawnInterval - Math.floor(score/5)*40);
    if (spawnTimer >= interval) {
      spawnTimer = 0;
      spawnEmoji();
    }

    // update entities
    for (let i = entities.length-1; i >= 0; i--) {
      const ent = entities[i];
      ent.y += ent.vy * dt;
      ent.rot += ent.rotSpeed * dt;

      // caught?
      if (checkCollision(ent)) {
        if (ent.good) score += 1;
        else score -= 2;
        // small clamp
        score = Math.max(0, score);
        entities.splice(i,1);
        scoreEl.textContent = score;
        continue;
      }

      // missed and fell off bottom
      if (ent.y > 1.15) {
        // penalty for missing good ones
        if (ent.good) {
          score = Math.max(0, score - 1);
          scoreEl.textContent = score;
        }
        entities.splice(i,1);
      }
    }
  }

  function draw() {
    // clear
    const w = canvas.clientWidth;
    const h = canvas.clientHeight;
    ctx.clearRect(0,0,w,h);

    // background subtle grid
    ctx.fillStyle = '#071727';
    ctx.fillRect(0,0,w,h);

    // draw entities
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    const baseFontSize = Math.min(w, h) * 0.06;
    for (const ent of entities) {
      const ex = ent.x * w;
      const ey = ent.y * h;
      ctx.save();
      ctx.translate(ex, ey);
      ctx.rotate(ent.rot);
      ctx.font = `${baseFontSize}px serif`;
      // shadow
      ctx.shadowColor = 'rgba(0,0,0,0.5)';
      ctx.shadowBlur = 8;
      ctx.fillStyle = ent.good ? '#fff' : '#fff';
      ctx.fillText(ent.char, 0, 0);
      ctx.restore();
    }

    // draw catcher (rounded rectangle with emoji face)
    const px = player.x * w;
    const pw = player.width * w;
    const py = player.y * h;
    // strip
    ctx.save();
    // body
    const radius = 12;
    ctx.fillStyle = '#1f6f8b';
    roundRect(ctx, px - pw/2, py - 16, pw, 32, radius);
    ctx.fill();
    // stripe highlight
    ctx.fillStyle = 'rgba(255,255,255,0.08)';
    roundRect(ctx, px - pw/2 + 6, py - 10, pw - 12, 8, 6);
    ctx.fill();
    // face emoji on catcher
    ctx.font = `${Math.floor(14 + pw*0.05)}px serif`;
    ctx.fillStyle = '#fff';
    ctx.fillText('😎', px, py);
    ctx.restore();

    // bottom floor
    ctx.fillStyle = 'rgba(255,255,255,0.03)';
    ctx.fillRect(0, h - 6, w, 6);
  }
f             ctx.translate(ex, ey);
      ctx.rotate(ent.rot);
      ctx.font = `${baseFontSize}px serif`;
      // shadow
      ctx.shadowColor = 'rgba(0,0,0,0.5)';
      ctx.shadowBlur = 8;
      ctx.fillStyle = ent.good ? '#fff' : '#fff';
      ctx.fillText(ent.char, 0, 0);
      ctx.restore();
    }

    // draw catcher (rounded rectangle with emoji face)
    const px = player.x * w;
    const pw = player.width * w;
    const py = player.y * h;
    // strip
    ctx.save();
    // body
    const radius = 12;
    ctx.fillStyle = '#1f6f8b';
    roundRect(ctx, px - pw/2, py - 16, pw, 32, radius);
    ctx.fill();
    // stripe highlight
    ctx.fillStyle = 'rgba(255,255,255,0.08)';
    roundRect(ctx, px - pw/2 + 6, py - 10, pw - 12, 8, 6);
    ctx.fill();
    // face emoji on catcher
    ctx.font = `${Math.floor(14 + pw*0.05)}px serif`;
    ctx.fillStyle = '#fff';
    ctx.fillText('😎', px, py);
    ctx.restore();

    // bottom floor
    ctx.fillStyle = 'rgba(255,255,255,0.03)';
    ctx.fillRect(0, h - 6, w, 6);
  }

  function roundRect(ctx, x, y, w, h, r) {
    const rr = Math.min(r, w/2, h/2);
    ctx.beginPath();
    ctx.moveTo(x+rr, y);
    ctx.arcTo(x+w, y, x+w, y+h, rr);
    ctx.arcTo(x+w, y+h, x, y+h, rr);
    ctx.arcTo(x, y+h, x, y, rr);
    ctx.arcTo(x, y, x+w, y, rr);
    ctx.closePath();
  }

  function loop(t) {
    if (!running) return;
    if (paused) {
      lastTime = t;
      requestAnimationFrame(loop);
      return;
    }
    const dt = Math.min(0.05, (t - lastTime) / 1000); // cap dt to avoid jump
    lastTime = t;

    // update difficultly slowly
    spawnInterval = Math.max(400, 1200 - score*8);

    update(dt);
    draw();

    // small game-over idea: if score is big and you want victory...
    requestAnimationFrame(loop);
  }

  // initial draw (before start)
  draw();

  // Friendly tips on long-press mobile
  canvas.addEventListener('contextmenu', e => e.preventDefault());
})();
</script>
</body>
</html><!doctype html>
<html lang="hi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Emoji Catcher — Funny Game</title>
  <style>
    :root{
      --bg:#0f172a;
      --panel:#0b1220;
      --accent:#ffd166;
      --text:#e6eef8;
    }
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;}
    body{
      background: linear-gradient(180deg, #08101f 0%, #071326 60%);
      color:var(--text);
      display:flex;
      align-items:center;
      justify-content:center;
      padding:20px;
    }
    .game {
      width:360px;
      max-width:calc(100vw - 40px);
      background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
      border:1px solid rgba(255,255,255,0.04);
      border-radius:12px;
      padding:12px;
      box-shadow: 0 8p
