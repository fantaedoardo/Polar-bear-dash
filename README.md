# Polar-bear-dash
index.html
<!doctype html>
<html lang="it">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
  <title>Tap Dash — Orso Polare</title>
  <style>
    html, body { margin:0; padding:0; background:#06121f; height:100%; overflow:hidden; font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }
    canvas { display:block; width:100vw; height:100vh; touch-action: manipulation; }
    .hint {
      position: fixed; left: 12px; bottom: 10px; right: 12px;
      color: rgba(255,255,255,0.75); font-size: 12px; line-height: 1.25;
      user-select:none; pointer-events:none;
      text-shadow: 0 2px 12px rgba(0,0,0,0.5);
    }
  </style>
</head>
<body>
<canvas id="c"></canvas>
<div class="hint">Tap per saltare • 1 km = 100 punti • Più vai avanti, più accelera</div>

<script>
(() => {
  "use strict";

  // ---------- Canvas ----------
  const canvas = document.getElementById("c");
  const ctx = canvas.getContext("2d", { alpha: false });

  function resize() {
    const dpr = Math.max(1, Math.min(2, window.devicePixelRatio || 1));
    canvas.width  = Math.floor(innerWidth  * dpr);
    canvas.height = Math.floor(innerHeight * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0); // draw in CSS pixels
  }
  addEventListener("resize", resize, { passive: true });
  resize();

  // ---------- Helpers ----------
  const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
  const rand = (a, b) => a + Math.random() * (b - a);

  // ---------- Game constants ----------
  const GROUND_RATIO = 0.78;           // ground y as ratio of height
  const GRAVITY = 2200;                // px/s^2
  const JUMP_V = -820;                 // px/s
  const BASE_SPEED = 360;              // px/s world speed
  const SPEED_GROWTH = 18;             // +px/s every "difficulty step"
  const KM_PX = 1400;                  // px travelled == 1 km (tuning)
  const POINTS_PER_KM = 100;

  // Difficulty ramps up over time:
  // - speed increases
  // - spawn interval decreases
  // - min gap between obstacles decreases (but clamped)
  const BASE_SPAWN = 1.05;             // seconds
  const MIN_SPAWN  = 0.42;             // seconds
  const BASE_GAP   = 320;              // px between obstacles
  const MIN_GAP    = 170;              // px

  // ---------- State ----------
  const state = {
    started: false,
    over: false,
    t: 0,
    dt: 0,
    lastTs: 0,
    speed: BASE_SPEED,
    spawnTimer: 0,
    distancePx: 0,
    kmCount: 0,
    score: 0,
    best: Number(localStorage.getItem("tapDashBest") || 0),
    shake: 0
  };

  // ---------- Entities ----------
  const bear = {
    x: 0,
    y: 0,
    w: 54,
    h: 44,
    vy: 0,
    onGround: true,
    blink: 0
  };

  const obstacles = []; // {x, w, h, type}

  // Parallax layers (simple ice hills)
  const hills = [
    { x: 0, speed: 0.20 },
    { x: 0, speed: 0.35 },
    { x: 0, speed: 0.55 },
  ];

  function reset() {
    state.started = false;
    state.over = false;
    state.t = 0;
    state.lastTs = 0;
    state.speed = BASE_SPEED;
    state.spawnTimer = 0;
    state.distancePx = 0;
    state.kmCount = 0;
    state.score = 0;
    state.shake = 0;

    obstacles.length = 0;
    bear.vy = 0;
    bear.onGround = true;
    bear.blink = 0;

    layout();
  }

  function layout() {
    const W = innerWidth;
    const H = innerHeight;
    const groundY = H * GROUND_RATIO;

    bear.x = Math.max(70, W * 0.18);
    bear.y = groundY - bear.h;
  }

  // ---------- Input ----------
  function jump() {
    if (!state.started) state.started = true;
    if (state.over) return;

    if (bear.onGround) {
      bear.vy = JUMP_V;
      bear.onGround = false;
    }
  }

  function restartTap(x, y) {
    // If game over: tap on button area
    if (!state.over) return false;
    const W = innerWidth, H = innerHeight;
    const bw = Math.min(260, W * 0.62);
    const bh = 54;
    const bx = (W - bw) / 2;
    const by = H * 0.62;
    return x >= bx && x <= bx + bw && y >= by && y <= by + bh;
  }

  function onPointer(ev) {
    const rect = canvas.getBoundingClientRect();
    const x = (ev.clientX ?? (ev.touches?.[0]?.clientX)) - rect.left;
    const y = (ev.clientY ?? (ev.touches?.[0]?.clientY)) - rect.top;

    if (restartTap(x, y)) {
      reset();
      return;
    }
    jump();
  }

  canvas.addEventListener("pointerdown", onPointer, { passive: true });
  addEventListener("keydown", (e) => {
    if (e.code === "Space" || e.code === "ArrowUp") {
      e.preventDefault();
      if (state.over) { reset(); return; }
      jump();
    }
  });

  // Prevent double-tap zoom on iOS-ish
  let lastTouch = 0;
  addEventListener("touchend", (e) => {
    const now = Date.now();
    if (now - lastTouch <= 350) e.preventDefault();
    lastTouch = now;
  }, { passive: false });

  // ---------- Spawning ----------
  function currentDifficulty() {
    // Grows with km travelled (smooth)
    const km = state.distancePx / KM_PX;
    return km; // 1.0 diff per km
  }

  function spawnObstacle() {
    const W = innerWidth;
    const H = innerHeight;
    const groundY = H * GROUND_RATIO;

    const diff = currentDifficulty();

    // Tree sizes vary a bit; later diff spawns slightly taller trees
    const baseH = rand(52, 92) + diff * 3.5;
    const h = clamp(baseH, 54, 140);
    const w = clamp(h * rand(0.38, 0.52), 24, 70);

    // Mix of single and "chunky" trees later
    const type = (diff > 3 && Math.random() < 0.25) ? "fat" : "pine";

    obstacles.push({
      x: W + rand(20, 120),
      w, h,
      y: groundY - h,
      type
    });
  }

  // ---------- Collision ----------
  function aabb(ax, ay, aw, ah, bx, by, bw, bh) {
    return ax < bx + bw && ax + aw > bx && ay < by + bh && ay + ah > by;
  }

  function gameOver() {
    state.over = true;
    state.started = true;
    state.shake = 10;
    if (state.score > state.best) {
      state.best = state.score;
      localStorage.setItem("tapDashBest", String(state.best));
    }
  }

  // ---------- Drawing ----------
  function drawBackground(W, H) {
    // Sky gradient
    const g = ctx.createLinearGradient(0, 0, 0, H);
    g.addColorStop(0, "#6fe7ff");
    g.addColorStop(0.5, "#7aa8ff");
    g.addColorStop(1, "#0a1f3a");
    ctx.fillStyle = g;
    ctx.fillRect(0, 0, W, H);

    // Soft aurora bands
    ctx.globalAlpha = 0.16;
    for (let i = 0; i < 3; i++) {
      const y = H * (0.12 + i * 0.08);
      const gg = ctx.createLinearGradient(0, y, W, y);
      gg.addColorStop(0, "rgba(120,255,210,0)");
      gg.addColorStop(0.5, "rgba(120,255,210,1)");
      gg.addColorStop(1, "rgba(120,255,210,0)");
      ctx.fillStyle = gg;
      ctx.beginPath();
      ctx.ellipse(W/2, y, W*0.55, 34, 0, 0, Math.PI*2);
      ctx.fill();
    }
    ctx.globalAlpha = 1;

    // Distant ice hills (parallax)
    const groundY = H * GROUND_RATIO;
    hills.forEach((layer, idx) => {
      const speedK = layer.speed;
      const amp = 18 + idx * 10;
      const base = groundY - (120 + idx * 45);

      layer.x -= state.speed * speedK * state.dt;
      if (layer.x < -W) layer.x += W;

      ctx.fillStyle = idx === 0 ? "rgba(220,250,255,0.35)"
                   : idx === 1 ? "rgba(210,245,255,0.28)"
                               : "rgba(200,240,255,0.22)";

      for (let rep = -1; rep <= 1; rep++) {
        const ox = layer.x + rep * W;
        ctx.beginPath();
        ctx.moveTo(ox, groundY);
        for (let x = 0; x <= W; x += 40) {
          const yy = base + Math.sin((x + idx*60) * 0.010) * amp;
          ctx.lineTo(ox + x, yy);
        }
        ctx.lineTo(ox + W, groundY);
        ctx.closePath();
        ctx.fill();
      }
    });

    // Snow particles
    ctx.globalAlpha = 0.5;
    ctx.fillStyle = "rgba(255,255,255,0.9)";
    const n = Math.floor(W * H / 50000);
    for (let i = 0; i < n; i++) {
      const x = (Math.sin((state.t*0.7 + i) * 1.7) * 0.5 + 0.5) * W;
      const y = ((state.t*40 + i*120) % (H*0.9));
      const r = (i % 3) + 0.8;
      ctx.beginPath();
      ctx.arc(x, y, r, 0, Math.PI*2);
      ctx.fill();
    }
    ctx.globalAlpha = 1;

    // Ground (ice)
    const ice = ctx.createLinearGradient(0, groundY, 0, H);
    ice.addColorStop(0, "#e9fbff");
    ice.addColorStop(1, "#bfefff");
    ctx.fillStyle = ice;
    ctx.fillRect(0, groundY, W, H-groundY);

    // Ground shine line
    ctx.strokeStyle = "rgba(255,255,255,0.5)";
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(0, groundY + 2);
    ctx.lineTo(W, groundY + 2);
    ctx.stroke();
  }

  function drawTree(o) {
    // stylized icy pine
    const { x, y, w, h, type } = o;

    // trunk
    ctx.fillStyle = "rgba(110,80,60,0.9)";
    const tw = w * 0.22;
    ctx.fillRect(x + w*0.39, y + h*0.72, tw, h*0.28);

    // snow cap + leaves
    if (type === "fat") {
      ctx.fillStyle = "rgba(30,160,130,0.95)";
      ctx.beginPath();
      ctx.roundRect(x, y + h*0.22, w, h*0.55, 10);
      ctx.fill();
      ctx.fillStyle = "rgba(240,255,255,0.95)";
      ctx.beginPath();
      ctx.roundRect(x + w*0.06, y + h*0.16, w*0.88, h*0.20, 12);
      ctx.fill();
    } else {
      ctx.fillStyle = "rgba(20,170,140,0.95)";
      for (let i = 0; i < 3; i++) {
        const yy = y + h*(0.10 + i*0.18);
        const ww = w*(0.95 - i*0.18);
        const xx = x + (w-ww)/2;
        const hh = h*(0.30);
        ctx.beginPath();
        ctx.moveTo(x + w/2, yy);
        ctx.lineTo(xx, yy + hh);
        ctx.lineTo(xx + ww, yy + hh);
        ctx.closePath();
        ctx.fill();

        // snow sprinkle on each tier
        ctx.fillStyle = "rgba(240,255,255,0.9)";
        ctx.beginPath();
        ctx.ellipse(x + w/2, yy + hh*0.35, ww*0.38, hh*0.18, 0, 0, Math.PI*2);
        ctx.fill();
        ctx.fillStyle = "rgba(20,170,140,0.95)";
      }
    }

    // subtle shadow on ground
    ctx.fillStyle = "rgba(0,0,0,0.12)";
    ctx.beginPath();
    ctx.ellipse(x + w/2, y + h + 6, w*0.55, 10, 0, 0, Math.PI*2);
    ctx.fill();
  }

  function drawBear() {
    const x = bear.x, y = bear.y, w = bear.w, h = bear.h;

    // shadow
    ctx.fillStyle = "rgba(0,0,0,0.14)";
    ctx.beginPath();
    ctx.ellipse(x + w/2, y + h + 7, w*0.55, 12, 0, 0, Math.PI*2);
    ctx.fill();

    // body
    ctx.fillStyle = "#f6fbff";
    ctx.beginPath();
    ctx.roundRect(x, y + h*0.18, w, h*0.72, 18);
    ctx.fill();

    // head
    ctx.beginPath();
    ctx.roundRect(x + w*0.08, y, w*0.62, h*0.55, 18);
    ctx.fill();

    // ears
    ctx.beginPath();
    ctx.roundRect(x + w*0.10, y + h*0.02, w*0.18, h*0.18, 10);
    ctx.roundRect(x + w*0.46, y + h*0.02, w*0.18, h*0.18, 10);
    ctx.fill();

    // belly tint
    ctx.fillStyle = "rgba(190,240,255,0.45)";
    ctx.beginPath();
    ctx.ellipse(x + w*0.56, y + h*0.60, w*0.24, h*0.22, 0, 0, Math.PI*2);
    ctx.fill();

    // face details
    const blink = bear.blink > 0 ? 1 : 0;
    ctx.fillStyle = "#0d1b2a";
    // eyes
    if (!blink) {
      ctx.beginPath();
      ctx.arc(x + w*0.28, y + h*0.24, 3.2, 0, Math.PI*2);
      ctx.arc(x + w*0.44, y + h*0.24, 3.2, 0, Math.PI*2);
      ctx.fill();
    } else {
      ctx.strokeStyle = "#0d1b2a";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(x + w*0.25, y + h*0.24);
      ctx.lineTo(x + w*0.31, y + h*0.24);
      ctx.moveTo(x + w*0.41, y + h*0.24);
      ctx.lineTo(x + w*0.47, y + h*0.24);
      ctx.stroke();
    }
    // nose
    ctx.fillStyle = "#1a2b3d";
    ctx.beginPath();
    ctx.roundRect(x + w*0.40, y + h*0.32, w*0.12, h*0.09, 6);
    ctx.fill();

    // feet hint
    ctx.fillStyle = "rgba(220,245,255,0.95)";
    ctx.beginPath();
    ctx.roundRect(x + w*0.10, y + h*0.78, w*0.28, h*0.18, 12);
    ctx.roundRect(x + w*0.48, y + h*0.78, w*0.30, h*0.18, 12);
    ctx.fill();
  }

  function drawHUD(W, H) {
    const pad = 14;
    ctx.fillStyle = "rgba(0,0,0,0.25)";
    ctx.beginPath();
    ctx.roundRect(pad, pad, 190, 68, 14);
    ctx.fill();

    ctx.fillStyle = "rgba(255,255,255,0.92)";
    ctx.font = "700 18px system-ui, -apple-system, Segoe UI, Roboto";
    ctx.fillText(`Punti: ${state.score}`, pad + 14, pad + 28);

    const km = (state.distancePx / KM_PX);
    ctx.font = "600 14px system-ui, -apple-system, Segoe UI, Roboto";
    ctx.fillStyle = "rgba(255,255,255,0.80)";
    ctx.fillText(`Distanza: ${km.toFixed(2)} km`, pad + 14, pad + 50);

    ctx.textAlign = "right";
    ctx.fillStyle = "rgba(255,255,255,0.85)";
    ctx.fillText(`Best: ${state.best}`, W - pad, pad + 24);
    ctx.textAlign = "left";

    if (!state.started && !state.over) {
      ctx.fillStyle = "rgba(255,255,255,0.95)";
      ctx.font = "800 28px system-ui, -apple-system, Segoe UI, Roboto";
      ctx.fillText("TAP DASH", pad, H * 0.22);

      ctx.fillStyle = "rgba(255,255,255,0.85)";
      ctx.font = "600 16px system-ui, -apple-system, Segoe UI, Roboto";
      ctx.fillText("Tap per iniziare • Tap per saltare", pad, H * 0.22 + 32);
    }

    if (state.over) {
      ctx.fillStyle = "rgba(0,0,0,0.40)";
      ctx.fillRect(0, 0, W, H);

      ctx.fillStyle = "rgba(255,255,255,0.95)";
      ctx.textAlign = "center";
      ctx.font = "900 34px system-ui, -apple-system, Segoe UI, Roboto";
      ctx.fillText("GAME OVER", W/2, H * 0.34);

      ctx.font = "700 18px system-ui, -apple-system, Segoe UI, Roboto";
      ctx.fillStyle = "rgba(255,255,255,0.90)";
      ctx.fillText(`Punti: ${state.score} • Best: ${state.best}`, W/2, H * 0.34 + 34);

      // Restart button
      const bw = Math.min(260, W * 0.62);
      const bh = 54;
      const bx = (W - bw) / 2;
      const by = H * 0.62;

      ctx.fillStyle = "rgba(120,255,210,0.95)";
      ctx.beginPath();
      ctx.roundRect(bx, by, bw, bh, 16);
      ctx.fill();

      ctx.fillStyle = "rgba(6,18,31,0.95)";
      ctx.font = "900 18px system-ui, -apple-system, Segoe UI, Roboto";
      ctx.fillText("RESTART", W/2, by + 34);

      ctx.textAlign = "left";
    }
  }

  // Polyfill for roundRect on older Android WebViews
  if (!CanvasRenderingContext2D.prototype.roundRect) {
    CanvasRenderingContext2D.prototype.roundRect = function(x, y, w, h, r) {
      r = Math.min(r, w/2, h/2);
      this.beginPath();
      this.moveTo(x+r, y);
      this.arcTo(x+w, y, x+w, y+h, r);
      this.arcTo(x+w, y+h, x, y+h, r);
      this.arcTo(x, y+h, x, y, r);
      this.arcTo(x, y, x+w, y, r);
      this.closePath();
      return this;
    };
  }

  // ---------- Update loop ----------
  function step(ts) {
    if (!state.lastTs) state.lastTs = ts;
    state.dt = Math.min(0.033, (ts - state.lastTs) / 1000);
    state.lastTs = ts;
    state.t += state.dt;

    const W = innerWidth;
    const H = innerHeight;
    const groundY = H * GROUND_RATIO;

    // Blink randomly
    bear.blink -= state.dt;
    if (bear.blink <= 0 && Math.random() < 0.015) bear.blink = 0.12;

    // Difficulty -> speed/spawn/gap
    const diff = currentDifficulty();
    const targetSpeed = BASE_SPEED + diff * SPEED_GROWTH;
    state.speed += (targetSpeed - state.speed) * (1 - Math.pow(0.001, state.dt)); // smooth approach

    let spawnInterval = BASE_SPAWN - diff * 0.06;
    spawnInterval = clamp(spawnInterval, MIN_SPAWN, BASE_SPAWN);

    let minGap = BASE_GAP - diff * 18;
    minGap = clamp(minGap, MIN_GAP, BASE_GAP);

    if (state.started && !state.over) {
      // distance + score: every KM_PX => +100
      state.distancePx += state.speed * state.dt;

      const kmNow = Math.floor(state.distancePx / KM_PX);
      if (kmNow > state.kmCount) {
        const gained = (kmNow - state.kmCount) * POINTS_PER_KM;
        state.kmCount = kmNow;
        state.score += gained;
      }

      // spawn obstacles with timer, ensure spacing
      state.spawnTimer -= state.dt;
      const lastObs = obstacles[obstacles.length - 1];
      const okGap = !lastObs || (W + 80) - lastObs.x >= minGap;

      if (state.spawnTimer <= 0 && okGap) {
        spawnObstacle();
        state.spawnTimer = spawnInterval + rand(-0.12, 0.18);
      }

      // physics
      bear.vy += GRAVITY * state.dt;
      bear.y += bear.vy * state.dt;

      if (bear.y >= groundY - bear.h) {
        bear.y = groundY - bear.h;
        bear.vy = 0;
        bear.onGround = true;
      }

      // move obstacles
      for (let i = obstacles.length - 1; i >= 0; i--) {
        const o = obstacles[i];
        o.x -= state.speed * state.dt;
        if (o.x + o.w < -30) obstacles.splice(i, 1);
      }

      // collision (slightly forgiving hitbox)
      const bx = bear.x + 10;
      const by = bear.y + 6;
      const bw = bear.w - 18;
      const bh = bear.h - 10;

      for (const o of obstacles) {
        const ox = o.x + 4;
        const oy = o.y + 2;
        const ow = o.w - 8;
        const oh = o.h - 2;
        if (aabb(bx, by, bw, bh, ox, oy, ow, oh)) {
          gameOver();
          break;
        }
      }
    } else {
      // keep bear grounded when idle
      bear.y = groundY - bear.h;
      bear.vy = 0;
      bear.onGround = true;
    }

    // camera shake on gameover
    let shakeX = 0, shakeY = 0;
    if (state.shake > 0) {
      state.shake -= state.dt * 22;
      const s = Math.max(0, state.shake);
      shakeX = (Math.random() * 2 - 1) * s;
      shakeY = (Math.random() * 2 - 1) * s;
    }

    // Draw
    ctx.save();
    ctx.translate(shakeX, shakeY);

    drawBackground(W, H);

    // obstacles
    for (const o of obstacles) drawTree(o);

    // bear
    drawBear();

    // HUD
    drawHUD(W, H);

    ctx.restore();

    requestAnimationFrame(step);
  }

  // ---------- Boot ----------
  reset();
  requestAnimationFrame(step);

})();
</script>
</body>
</html>
