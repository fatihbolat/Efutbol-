# Efutbol-

<!doctype html>
<html lang="tr">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" />
<title>e-Futbol - Tarayıcı</title>
<style>
  :root{--bg:#0f1724;--panel:#ffffffdd;--accent:#10b981;--danger:#ef4444}
  html,body{height:100%;margin:0;background:var(--bg);font-family:Inter,ui-sans-serif,system-ui,Arial;}
  .wrap{min-height:100%;display:flex;flex-direction:column;align-items:center;padding:14px 10px 90px;}
  .topbar{width:100%;max-width:920px;color:#fff;display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
  .title{font-weight:600}
  .controls{display:flex;gap:8px}
  button{background:#1f2937;color:#fff;border:0;padding:8px 10px;border-radius:10px;font-weight:600}
  .stage-wrap{width:100%;max-width:920px;background:#000;border-radius:16px;overflow:hidden;box-shadow:0 10px 30px rgba(0,0,0,.6)}
  canvas{display:block;width:100%;height:auto;background:#000}
  .info{width:100%;max-width:920px;color:#cbd5e1;margin-top:10px;text-align:center;font-size:13px}
  /* Mobile controls */
  .ui-fixed{position:fixed;inset:0;pointer-events:none}
  .joystick-area{position:absolute;bottom:22px;left:18px;width:120px;height:120px;border-radius:999px;background:rgba(255,255,255,0.04);backdrop-filter:blur(2px);pointer-events:auto;touch-action:none}
  .joy-knob{position:absolute;left:50%;top:50%;width:48px;height:48px;margin-left:-24px;margin-top:-24px;border-radius:50%;background:rgba(255,255,255,0.12);border:1px solid rgba(255,255,255,0.18);pointer-events:none;transform:translate(0,0);transition:transform 0s}
  .right-buttons{position:absolute;bottom:22px;right:18px;display:flex;flex-direction:column;gap:12px;pointer-events:auto}
  .btn-big{width:86px;height:86px;border-radius:999px;background:var(--accent);color:#042018;font-weight:800;border:0;font-size:16px;box-shadow:0 8px 18px rgba(16,185,129,.15)}
  .btn-small{width:64px;height:64px;border-radius:999px;background:#6366f1;color:#fff;border:0;font-weight:800;font-size:14px;box-shadow:0 8px 18px rgba(99,102,241,.12)}
  /* overlay for scoreboard inside canvas fallback */
  .desktop-hint{display:none}
  @media(min-width:680px){ .joystick-area{display:none} .right-buttons{display:none} .desktop-hint{display:block;color:#94a3b8;margin-top:8px} }
</style>
</head>
<body>
<div class="wrap">
  <div class="topbar">
    <div class="title">⚽ e-Futbol — Tarayıcı</div>
    <div class="controls">
      <button id="pauseBtn">Duraklat</button>
      <button id="restartBtn" style="background:var(--accent);">Yeniden Başlat</button>
    </div>
  </div>

  <div class="stage-wrap" id="stageWrap">
    <canvas id="gameCanvas" width="900" height="520"></canvas>
  </div>

  <div class="info">
    PC: WASD hareket, J=Şut, K=Sprint. Mobil: sol joystick, sağ Şut & Sprint butonları. 5 gol veya 90 sn bittiğinde maç sona erer.
    <div class="desktop-hint">Tarayıcıda tam ekran (F11) daha iyi deneyim verir.</div>
  </div>
</div>

<!-- Mobil UI -->
<div class="ui-fixed" aria-hidden="true">
  <div class="joystick-area" id="joyArea">
    <div class="joy-knob" id="joyKnob"></div>
  </div>
  <div class="right-buttons">
    <button id="shootBtn" class="btn-big">Şut</button>
    <button id="sprintBtn" class="btn-small">Sprint</button>
  </div>
</div>

<script>
/* -------------- Tam tek dosya e-Futbol oyunu -------------- */
/* Dünya boyutu (logical) */
const WORLD_W = 900, WORLD_H = 520;

const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

let DPR = Math.max(1, window.devicePixelRatio || 1);
function resizeCanvas(){
  const parent = document.getElementById('stageWrap');
  const cssW = Math.min(parent.clientWidth, 1000);
  const cssH = cssW * (WORLD_H / WORLD_W);
  canvas.style.width = cssW + 'px';
  canvas.style.height = cssH + 'px';
  canvas.width = Math.round(cssW * DPR);
  canvas.height = Math.round(cssH * DPR);
  ctx.setTransform(canvas.width / WORLD_W, 0, 0, canvas.height / WORLD_H, 0, 0);
}
window.addEventListener('resize', ()=>{ DPR = Math.max(1, window.devicePixelRatio || 1); resizeCanvas(); });
resizeCanvas();

/* Oyun durumları */
let running = true, paused = false, gameOver = false;
let score = { you:0, cpu:0 }, timeLeft = 90;
let message = '';
const goal = { leftX: 20, rightX: WORLD_W - 20, y1: 180, y2: 340 };

/* Oyuncular ve top */
const player = { x:150, y:260, r:16, base:2.35, sprint:1.0, sprintMax:1.8 };
const cpu = { x: WORLD_W - 150, y:260, r:16, speed:2.2 };
const ball = { x: WORLD_W/2, y: WORLD_H/2, r:9, vx:0, vy:0 };

/* Kontroller */
const keys = {};
let joy = { active:false, x:0, y:0 }; // -1..1
let isTouchActive = false;

/* Helper'lar */
const clamp = (v,a,b)=>Math.max(a,Math.min(b,v));
const dist = (ax,ay,bx,by)=>Math.hypot(ax-bx, ay-by);
const vib = (ms)=>{ try{ if(navigator.vibrate) navigator.vibrate(ms); }catch(e){} };
const fmtTime = t => `${Math.floor(t/60)}:${String(t%60).padStart(2,'0')}`;

/* Zamanlayıcı */
let timerId = null;
function startTimer(){
  if(timerId) clearInterval(timerId);
  if(!running || paused || gameOver) return;
  timerId = setInterval(()=>{
    if(!running || paused || gameOver) return;
    timeLeft = Math.max(0, timeLeft - 1);
    if(timeLeft === 0) finish(score.you === score.cpu ? 'Berabere!' : (score.you > score.cpu ? 'Kazandın!' : 'Kaybettin!'));
  }, 1000);
}
startTimer();

/* Oyun başlangıcı / reset */
function resetBall(dir = 1){
  ball.x = WORLD_W/2; ball.y = WORLD_H/2;
  ball.vx = 2.7 * dir; ball.vy = (Math.random()*2-1)*1.6;
  player.x = 150; player.y = WORLD_H/2; player.sprint = 1.0;
  cpu.x = WORLD_W - 150; cpu.y = WORLD_H/2;
}
function restart(){
  score = { you:0, cpu:0 }; timeLeft = 90; message = 'Yeni maç!'; gameOver = false; paused = false; running = true;
  resetBall(Math.random()<0.5?-1:1); startTimer();
}
function finish(text){
  message = text; running = false; paused = false; gameOver = true; vib(60);
  if(timerId) clearInterval(timerId);
}

/* Giriş: klavye */
window.addEventListener('keydown', (e)=>{
  const k = e.key.toLowerCase();
  keys[k] = true;
  if(['w','a','s','d','j','k'].includes(k)) e.preventDefault();
});
window.addEventListener('keyup', (e)=>{ keys[e.key?.toLowerCase()] = false; });

/* Mobil joystick */
const joyArea = document.getElementById('joyArea');
const joyKnob = document.getElementById('joyKnob');
let joyState = { dragging:false, ox:0, oy:0, dx:0, dy:0 };

function toLocal(e, el){
  const rect = el.getBoundingClientRect();
  const pt = e.touches ? e.touches[0] : e;
  return { x: pt.clientX - rect.left, y: pt.clientY - rect.top };
}

joyArea.addEventListener('touchstart', (e)=>{ e.preventDefault(); const p = toLocal(e, joyArea); joyState.dragging=true; joyState.ox=p.x; joyState.oy=p.y; joyState.dx=0; joyState.dy=0; isTouchActive=true; joy = {active:true,x:0,y:0}; }, {passive:false});
joyArea.addEventListener('touchmove', (e)=>{ e.preventDefault(); if(!joyState.dragging) return; const p = toLocal(e, joyArea); let dx = p.x - joyState.ox, dy = p.y - joyState.oy; const max = 44; const mag = Math.hypot(dx,dy); if(mag>max){ dx = dx/mag*max; dy = dy/mag*max; } joyState.dx = dx; joyState.dy = dy; joy = { active:true, x: dx/max, y: dy/max }; joyKnob.style.transform = `translate(${joyState.dx}px, ${joyState.dy}px)`; }, {passive:false});
joyArea.addEventListener('touchend', (e)=>{ joyState.dragging=false; joyState.dx=0; joyState.dy=0; joy = {active:false,x:0,y:0}; joyKnob.style.transform = 'translate(0,0)'; isTouchActive=false; });

/* mouse support for desktop joystick */
joyArea.addEventListener('mousedown', (e)=>{ e.preventDefault(); const p = toLocal(e, joyArea); joyState.dragging=true; joyState.ox=p.x; joyState.oy=p.y; joyState.dx=0; joyState.dy=0; isTouchActive=true; joy = {active:true,x:0,y:0}; });
window.addEventListener('mousemove', (e)=>{ if(!joyState.dragging) return; const p = toLocal(e, joyArea); let dx = p.x - joyState.ox, dy = p.y - joyState.oy; const max = 44; const mag = Math.hypot(dx,dy); if(mag>max){ dx = dx/mag*max; dy = dy/mag*max; } joyState.dx = dx; joyState.dy = dy; joy = { active:true, x: dx/max, y: dy/max }; joyKnob.style.transform = `translate(${joyState.dx}px, ${joyState.dy}px)`; });
window.addEventListener('mouseup', ()=>{ if(!joyState.dragging) return; joyState.dragging=false; joyState.dx=0; joyState.dy=0; joy = {active:false,x:0,y:0}; joyKnob.style.transform = 'translate(0,0)'; isTouchActive=false; });

/* Butonlar */
document.getElementById('shootBtn').addEventListener('touchstart', (e)=>{ e.preventDefault(); keys['j'] = true; vib(8); }, {passive:false});
document.getElementById('shootBtn').addEventListener('touchend', (e)=>{ keys['j'] = false; }, {passive:false});
document.getElementById('shootBtn').addEventListener('mousedown', ()=>{ keys['j'] = true; vib(8); });
document.getElementById('shootBtn').addEventListener('mouseup', ()=>{ keys['j'] = false; });

document.getElementById('sprintBtn').addEventListener('touchstart', (e)=>{ e.preventDefault(); keys['k'] = true; }, {passive:false});
document.getElementById('sprintBtn').addEventListener('touchend', (e)=>{ keys['k'] = false; }, {passive:false});
document.getElementById('sprintBtn').addEventListener('mousedown', ()=>{ keys['k'] = true; });
document.getElementById('sprintBtn').addEventListener('mouseup', ()=>{ keys['k'] = false; });

/* Üst butonlar */
document.getElementById('pauseBtn').addEventListener('click', ()=>{
  if(gameOver) return;
  paused = !paused;
  document.getElementById('pauseBtn').textContent = paused ? 'Devam' : 'Duraklat';
  if(!paused) startTimer(); else if(timerId) clearInterval(timerId);
});
document.getElementById('restartBtn').addEventListener('click', ()=> restart());

/* Oyun mantığı: update ve çarpışmalar */
function update(dt){
  // oyuncu input
  let ix = 0, iy = 0;
  if(keys['w']) iy -= 1; if(keys['s']) iy += 1; if(keys['a']) ix -= 1; if(keys['d']) ix += 1;
  if(joy.active){ ix += joy.x; iy += joy.y; }
  const len = Math.hypot(ix,iy) || 1;

  // sprint
  if(keys['k']) player.sprint = clamp((player.sprint || 1.0) + dt*1.4, 1.0, player.sprintMax);
  else player.sprint = clamp((player.sprint || 1.0) - dt*1.0, 1.0, player.sprintMax);
  const speed = player.base * (1 + (player.sprint - 1));

  player.x += (ix/len) * speed * 120 * dt;
  player.y += (iy/len) * speed * 120 * dt;
  player.x = clamp(player.x, 40, WORLD_W - 40);
  player.y = clamp(player.y, 30, WORLD_H - 30);

  // CPU AI basit
  const defendX = WORLD_W - 150;
  const targetX = (Math.abs(ball.x - cpu.x) + Math.abs(ball.y - cpu.y) < 280) ? ball.x : defendX;
  const targetY = clamp(ball.y, 100, WORLD_H - 100);
  const cdx = targetX - cpu.x, cdy = targetY - cpu.y; const clen = Math.hypot(cdx,cdy) || 1;
  cpu.x += (cdx/clen) * cpu.speed * 110 * dt;
  cpu.y += (cdy/clen) * cpu.speed * 110 * dt;
  cpu.x = clamp(cpu.x, 40, WORLD_W - 40); cpu.y = clamp(cpu.y, 30, WORLD_H - 30);

  // top hareketi
  ball.x += ball.vx; ball.y += ball.vy; ball.vx *= 0.992; ball.vy *= 0.992;

  // üst-alt çarpma
  if(ball.y - ball.r < 20){ ball.y = 20 + ball.r; ball.vy = Math.abs(ball.vy); }
  if(ball.y + ball.r > WORLD_H - 20){ ball.y = WORLD_H - 20 - ball.r; ball.vy = -Math.abs(ball.vy); }

  // yan dışarı -> gol
  if(ball.x - ball.r < 0){ // sol dışarı => cpu gol
    score.cpu++; onGoal('cpu'); return;
  }
  if(ball.x + ball.r > WORLD_W){ score.you++; onGoal('you'); return; }

  // yan duvar sekme (kalelerin dışında)
  if(ball.x - ball.r < 40 && (ball.y < goal.y1 || ball.y > goal.y2)){ ball.x = 40 + ball.r; ball.vx = Math.abs(ball.vx); }
  if(ball.x + ball.r > WORLD_W - 40 && (ball.y < goal.y1 || ball.y > goal.y2)){ ball.x = WORLD_W - 40 - ball.r; ball.vx = -Math.abs(ball.vx); }

  // oyuncu-top etkileşim ve şut
  interactKick(player, 3.9, goal.rightX);
  // cpu-top etkileşim ve şut
  const dc = dist(cpu.x, cpu.y, ball.x, ball.y);
  if(dc < cpu.r + 18){
    const nx = (ball.x - cpu.x)/(dc||1), ny = (ball.y - cpu.y)/(dc||1);
    ball.x = cpu.x + (cpu.r + ball.r + 0.5)*nx; ball.y = cpu.y + (cpu.r + ball.r + 0.5)*ny;
    ball.vx += nx*0.5; ball.vy += ny*0.5;
    // şut
    const tx = goal.leftX, ty = ball.y + (Math.random()*34 - 17);
    const kdx = tx - cpu.x, kdy = ty - cpu.y; const klen = Math.hypot(kdx,kdy)||1;
    ball.vx += (kdx/klen) * 3.5; ball.vy += (kdy/klen) * 3.0;
  }
}

function interactKick(actor, power, targetX){
  const d = dist(actor.x, actor.y, ball.x, ball.y);
  if(d < actor.r + ball.r){
    const nx = (ball.x - actor.x)/(d||1), ny = (ball.y - actor.y)/(d||1);
    ball.x = actor.x + (actor.r + ball.r + 0.5)*nx; ball.y = actor.y + (actor.r + ball.r + 0.5)*ny;
    ball.vx += nx * 0.6; ball.vy += ny * 0.6;
  }
  if(keys['j'] && d < actor.r + 18){
    vib(10);
    const ty = ball.y + (Math.random()*30 - 15);
    const kdx = targetX - actor.x, kdy = ty - actor.y; const klen = Math.hypot(kdx,kdy)||1;
    ball.vx += (kdx/klen) * power; ball.vy += (kdy/klen) * (power*0.85);
    if(!isTouchActive) keys['j'] = false; // tek atış için tuşu temizle
  }
}

function onGoal(side){
  message = (side === 'you') ? 'GOOOOL!' : 'Gol yedik!';
  vib(25);
  if(score.you >= 5 || score.cpu >= 5){ finish(score.you > score.cpu ? 'Maç bitti: Kazandın!' : 'Maç bitti: Kaybettin!'); return; }
  setTimeout(()=> message = '', 1200);
  resetBall(side === 'you' ? -1 : 1);
}

/* Render */
function render(){
  // temizle (logical coords)
  ctx.clearRect(0,0,WORLD_W,WORLD_H);

  // zemin
  const grd = ctx.createLinearGradient(0,0,0,WORLD_H); grd.addColorStop(0,'#18703a'); grd.addColorStop(1,'#146c2e');
  ctx.fillStyle = grd; ctx.fillRect(0,0,WORLD_W,WORLD_H);

  // saha çizgileri
  ctx.strokeStyle = '#ffffff'; ctx.lineWidth = 2;
  ctx.strokeRect(40,20,WORLD_W-80,WORLD_H-40);
  ctx.beginPath(); ctx.moveTo(WORLD_W/2,20); ctx.lineTo(WORLD_W/2,WORLD_H-20); ctx.stroke();
  ctx.beginPath(); ctx.arc(WORLD_W/2, WORLD_H/2, 60, 0, Math.PI*2); ctx.stroke();

  // ceza sahaları
  ctx.strokeRect(40, WORLD_H/2 - 110, 110, 220);
  ctx.strokeRect(WORLD_W-150, WORLD_H/2 - 110, 110, 220);

  // kaleler (çerçeve)
  ctx.strokeRect(30, goal.y1, 10, goal.y2 - goal.y1);
  ctx.strokeRect(WORLD_W-40, goal.y1, 10, goal.y2 - goal.y1);

  // top
  ctx.fillStyle = '#f9fafb'; ctx.beginPath(); ctx.arc(ball.x, ball.y, ball.r, 0, Math.PI*2); ctx.fill();
  ctx.strokeStyle = '#111827'; ctx.beginPath(); ctx.arc(ball.x, ball.y, ball.r, 0, Math.PI*2); ctx.stroke();

  // oyuncular
  drawPlayer(player.x, player.y, player.r, '#2563eb', 'SEN');
  drawPlayer(cpu.x, cpu.y, cpu.r, '#ef4444', 'CPU');

  // UI üst panel (beyaz kutu)
  ctx.setTransform(1,0,0,1,0,0); // reset transform to canvas pixel space
  const panelPad = 12 * DPR;
  const panelH = 40 * DPR;
  const wpx = canvas.width, hpx = canvas.height;
  // draw a translucent panel as rectangle in pixel space
  ctx.fillStyle = 'rgba(255,255,255,0.9)';
  roundRect(ctx, panelPad, panelPad, wpx - panelPad*2, panelH, 8*DPR);
  ctx.fill();

  // text
  ctx.fillStyle = '#0b1220';
  ctx.font = `${14*DPR}px sans-serif`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(`SEN ${score.you} - ${score.cpu} CPU`, wpx/2, panelPad + panelH/2);

  ctx.fillStyle = '#0b1220';
  ctx.font = `${12*DPR}px sans-serif`;
  ctx.textAlign = 'left';
  ctx.fillText(`Süre: ${fmtTime(timeLeft)}`, panelPad + 16*DPR, panelPad + panelH/2 + 2*DPR);
  ctx.textAlign = 'right';
  ctx.fillText('WASD, J=Şut, K=Sprint', wpx - panelPad - 14*DPR, panelPad + panelH/2 + 2*DPR);

  // mesage alt
  if(message){
    ctx.textAlign = 'center'; ctx.fillStyle = 'rgba(255,255,255,0.95)'; ctx.font = `${26*DPR}px sans-serif`;
    ctx.fillText(message, wpx/2, hpx - 30*DPR);
  }

  // restore logical transform for next frame
  ctx.setTransform(canvas.width / WORLD_W, 0, 0, canvas.height / WORLD_H, 0, 0);
}

function drawPlayer(x,y,r,color,label){
  ctx.fillStyle = color; ctx.beginPath(); ctx.arc(x,y,r,0,Math.PI*2); ctx.fill();
  ctx.fillStyle = '#fff'; ctx.font = '12px sans-serif'; ctx.textAlign='center'; ctx.textBaseline='middle';
  ctx.fillText(label, x, y - r - 12);
}

function roundRect(ctx,x,y,w,h,r){
  // pixel-space rectangle (expects transform reset)
  ctx.beginPath(); ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r); ctx.arcTo(x+w,y+h,x,y+h,r); ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r); ctx.closePath();
}

/* Main loop */
let last = performance.now();
function loop(now){
  const dt = Math.min(0.033, (now - last) / 1000); last = now;
  if(running && !paused && !gameOver) update(dt);
  render();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);

/* İlk reset */
resetBall(1);

/* Kullanıcıya indirme notu (isteğe bağlı) */
console.log('e-Futbol yüklendi. Mobilde joystick ile, PCde WASD ile oynayın.');

/* ----- Son ----- */
</script>
</body>
</html>
