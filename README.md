# -<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>像素猎手 · PIXEL HUNTER</title>
<style>
  html,body{margin:0;height:100%;background:#05070d;overflow:hidden;
    display:flex;align-items:center;justify-content:center;
    font-family:'Courier New',monospace;color:#8ab;}
  #game{image-rendering:pixelated;image-rendering:crisp-edges;
    box-shadow:0 0 0 2px #1b2340, 0 0 60px #0a1a3a; cursor:crosshair;
    display:block;}
  #hint{position:fixed;bottom:6px;left:0;right:0;text-align:center;
    font-size:11px;opacity:.4;pointer-events:none;letter-spacing:1px}
</style>
</head>
<body>
<canvas id="game"></canvas>
<div id="hint">WASD 移动 · 鼠标瞄准 · 左键射击 · 空格冲刺 · P 暂停</div>
<script>
(function(){
'use strict';

/* ==================== 基础 ==================== */
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = 480, H = 300;
canvas.width = W; canvas.height = H;
ctx.imageSmoothingEnabled = false;

function resize(){
  const s = Math.min((window.innerWidth-24)/W, (window.innerHeight-56)/H);
  const scale = Math.max(1, Math.floor(s));
  canvas.style.width  = W*scale + 'px';
  canvas.style.height = H*scale + 'px';
}
window.addEventListener('resize', resize); resize();

const rand = (a,b)=>a+Math.random()*(b-a);
const randi = (a,b)=>Math.floor(rand(a,b+1));
const clamp = (v,a,b)=>v<a?a:v>b?b:v;

/* ==================== 输入 ==================== */
const keys = Object.create(null);
const mouse = {x:W/2, y:H/2, down:false};

function toCanvas(e){
  const r = canvas.getBoundingClientRect();
  mouse.x = (e.clientX-r.left)/r.width  * W;
  mouse.y = (e.clientY-r.top )/r.height * H;
}
canvas.addEventListener('mousemove', toCanvas);
canvas.addEventListener('mousedown', e=>{
  e.preventDefault(); toCanvas(e); mouse.down = true; onClick();
});
window.addEventListener('mouseup', ()=> mouse.down = false);
canvas.addEventListener('contextmenu', e=>e.preventDefault());

window.addEventListener('keydown', e=>{
  if(['ArrowUp','ArrowDown','ArrowLeft','ArrowRight','Space'].includes(e.code)) e.preventDefault();
  if(keys[e.code]) return;
  keys[e.code] = true;
  onKeyPress(e.code);
});
window.addEventListener('keyup', e=> keys[e.code] = false);

/* ==================== 音效 ==================== */
let actx = null;
function initAudio(){
  if(actx) return;
  try { actx = new (window.AudioContext||window.webkitAudioContext)(); } catch(err){}
}
function sfx(freq, dur, type, vol, slide){
  if(!actx) return;
  const t = actx.currentTime;
  const o = actx.createOscillator();
  const g = actx.createGain();
  o.type = type || 'square';
  o.frequency.setValueAtTime(freq, t);
  if(slide) o.frequency.exponentialRampToValueAtTime(Math.max(30, freq+slide), t+dur);
  g.gain.setValueAtTime(vol||0.04, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t+dur);
  o.connect(g); g.connect(actx.destination);
  o.start(t); o.stop(t+dur+0.02);
}

/* ==================== 背景预渲染 ==================== */
const bgCanvas = document.createElement('canvas');
bgCanvas.width = W; bgCanvas.height = H;
(function(){
  const b = bgCanvas.getContext('2d');
  b.fillStyle = '#0a0c14'; b.fillRect(0,0,W,H);
  for(let y=8; y<H; y+=16)
    for(let x=8; x<W; x+=16){
      b.fillStyle = '#131a2b';
      b.fillRect(x, y, 1, 1);
    }
  for(let i=0;i<260;i++){
    b.fillStyle = 'rgba(70,100,170,'+rand(0.05,0.25).toFixed(2)+')';
    b.fillRect(Math.floor(rand(0,W)), Math.floor(rand(0,H)), 1, 1);
  }
})();

/* ==================== 数据 ==================== */
const ENEMY_TYPES = {
  slime  : {hp:2, speed:42, r:7,  color:'#5ad05a', xp:1, contact:1},
  bat    : {hp:1, speed:88, r:5,  color:'#b06cff', xp:1, contact:1},
  shooter: {hp:3, speed:42, r:8,  color:'#ff9a3c', xp:2, contact:1},
  tank   : {hp:9, speed:28, r:11, color:'#4a9eff', xp:3, contact:2},
};

const UPGRADES = [
  {name:'力 量',   desc:'子弹伤害 +1',            apply:p=>p.dmg+=1},
  {name:'速 射',   desc:'射速提升 20%',           apply:p=>p.fireRate=Math.max(0.055,p.fireRate*0.8)},
  {name:'散 射',   desc:'额外发射一颗子弹',        apply:p=>p.bullets+=1},
  {name:'疾 行',   desc:'移动速度 +15%',          apply:p=>p.speed*=1.15},
  {name:'体 质',   desc:'最大生命 +1，回复 2 点',  apply:p=>{p.maxHp+=1;p.hp=Math.min(p.maxHp,p.hp+2);}},
  {name:'穿 透',   desc:'子弹可多穿透 1 个敌人',   apply:p=>p.pierce+=1},
  {name:'影 步',   desc:'冲刺冷却 -25%',          apply:p=>p.dashMax*=0.75},
  {name:'磁 力',   desc:'经验拾取范围 +45%',       apply:p=>p.magnet*=1.45},
  {name:'高速弹',  desc:'子弹飞行速度 +25%',       apply:p=>p.bulletSpeed*=1.25},
  {name:'急 救',   desc:'立即回复 3 点生命',       apply:p=>p.hp=Math.min(p.maxHp,p.hp+3)},
  {name:'弹 匣',   desc:'射速提升 12%，伤害 +1',   apply:p=>{p.fireRate=Math.max(0.055,p.fireRate*0.88); p.dmg+=1;}},
];

/* ==================== 状态 ==================== */
let player = null;
let uid = 1;

const game = {
  state:'title', t:0,
  wave:0, phase:'break', phaseTimer:0, spawnQueue:[], spawnTimer:0,
  enemies:[], bullets:[], ebullets:[], gems:[], particles:[], texts:[],
  shake:0, flash:0,
  score:0, kills:0, combo:0, comboTimer:0,
  choices:[]
};

function newPlayer(){
  return {
    x:W/2, y:H/2, r:6,
    hp:5, maxHp:5,
    speed:105,
    fireRate:0.22, fireTimer:0,
    dmg:1, bulletSpeed:320, pierce:0, bullets:1,
    dashCd:0, dashMax:1.1, dashTime:0, dashDir:{x:1,y:0},
    invuln:0, angle:0,
    xp:0, level:1, xpNext:5,
    magnet:34
  };
}
player = newPlayer();

/* ==================== 粒子 / 文字 ==================== */
function spawnParticle(x,y,vx,vy,life,color,size){
  if(game.particles.length > 420) return;
  game.particles.push({x,y,vx,vy,life,maxLife:life,color,size:size||2});
}
function addText(x,y,text,color,size){
  game.texts.push({x,y,text,color,life:1.6,maxLife:1.6,size:size||11});
}
function updateParticles(dt){
  for(let i=game.particles.length-1;i>=0;i--){
    const p = game.particles[i];
    p.x += p.vx*dt; p.y += p.vy*dt;
    p.vx *= (1-dt*4); p.vy *= (1-dt*4);
    p.life -= dt;
    if(p.life <= 0) game.particles.splice(i,1);
  }
}
function updateTexts(dt){
  for(let i=game.texts.length-1;i>=0;i--){
    const t = game.texts[i];
    t.y -= dt*18; t.life -= dt;
    if(t.life <= 0) game.texts.splice(i,1);
  }
}

/* ==================== 波次 ==================== */
function pickEnemyType(w){
  const pool = [['slime', 10]];
  if(w >= 2) pool.push(['bat', 5 + w*0.8]);
  if(w >= 3) pool.push(['shooter', 3 + w*0.7]);
  if(w >= 5) pool.push(['tank', 2 + w*0.5]);
  let total = 0;
  for(const p of pool) total += p[1];
  let r = Math.random()*total;
  for(const p of pool){ r -= p[1]; if(r <= 0) return p[0]; }
  return 'slime';
}

function startWave(){
  game.wave++;
  game.phase = 'spawn';
  game.spawnTimer = 0.15;
  const w = game.wave;
  const count = Math.min(46, 4 + Math.floor(w*1.8));
  game.spawnQueue = [];
  for(let i=0;i<count;i++) game.spawnQueue.push(pickEnemyType(w));
  addText(W/2, 46, '第 ' + w + ' 波', '#ffffff', 16);
  sfx(440,0.18,'square',0.05,-120);
}

function spawnEnemy(type){
  const base = ENEMY_TYPES[type];
  const pad = 22;
  let x, y;
  const side = randi(0,3);
  if(side===0){ x = rand(0,W); y = -pad; }
  else if(side===1){ x = W+pad; y = rand(0,H); }
  else if(side===2){ x = rand(0,W); y = H+pad; }
  else { x = -pad; y = rand(0,H); }

  const hpMul = 1 + (game.wave-1)*0.24;
  const e = {
    id: uid++,
    type, x, y,
    r: base.r,
    color: base.color,
    speed: base.speed * (1 + (game.wave-1)*0.012),
    hp: Math.ceil(base.hp * hpMul),
    maxHp: Math.ceil(base.hp * hpMul),
    contact: base.contact,
    xp: base.xp,
    flash: 0, kx:0, ky:0,
    seed: rand(0, 10),
    shootTimer: rand(0.8, 1.8),
    strafe: Math.random()<0.5 ? 1 : -1
  };
  game.enemies.push(e);
}

function updateWave(dt){
  if(game.phase === 'break'){
    game.phaseTimer -= dt;
    if(game.phaseTimer <= 0) startWave();
    return;
  }
  if(game.phase === 'spawn'){
    game.spawnTimer -= dt;
    if(game.spawnTimer <= 0 && game.spawnQueue.length){
      spawnEnemy(game.spawnQueue.shift());
      game.spawnTimer = Math.max(0.16, 0.52 - game.wave*0.018);
    }
    if(game.spawnQueue.length === 0) game.phase = 'fight';
  }
  if(game.phase === 'fight' && game.enemies.length === 0){
    game.phase = 'break';
    game.phaseTimer = 2.2;
    game.score += 40 * game.wave;
    addText(W/2, H/2 - 26, '波次清除！ +' + (40*game.wave), '#7cf07c', 13);
    sfx(660,0.1,'square',0.04);
    setTimeout(()=>sfx(880,0.14,'square',0.04), 100);
  }
}

/* ==================== 玩家 ==================== */
function tryDash(){
  const p = player;
  if(!p || game.state !== 'playing') return;
  if(p.dashCd > 0 || p.dashTime > 0) return;
  let ix=0, iy=0;
  if(keys.KeyA || keys.ArrowLeft)  ix--;
  if(keys.KeyD || keys.ArrowRight) ix++;
  if(keys.KeyW || keys.ArrowUp)    iy--;
  if(keys.KeyS || keys.ArrowDown)  iy++;
  if(ix===0 && iy===0){ ix = Math.cos(p.angle); iy = Math.sin(p.angle); }
  const l = Math.hypot(ix,iy) || 1;
  p.dashDir = {x:ix/l, y:iy/l};
  p.dashTime = 0.14;
  p.dashCd = p.dashMax;
  p.invuln = Math.max(p.invuln, 0.22);
  game.shake = Math.max(game.shake, 3);
  sfx(240,0.14,'sawtooth',0.04,420);
}

function updatePlayer(dt){
  const p = player;
  let ix=0, iy=0;
  if(keys.KeyA || keys.ArrowLeft)  ix--;
  if(keys.KeyD || keys.ArrowRight) ix++;
  if(keys.KeyW || keys.ArrowUp)    iy--;
  if(keys.KeyS || keys.ArrowDown)  iy++;
  const len = Math.hypot(ix,iy);
  if(len > 0){ ix/=len; iy/=len; }

  if(p.dashTime > 0){
    p.dashTime -= dt;
    p.x += p.dashDir.x * p.speed * 3.4 * dt;
    p.y += p.dashDir.y * p.speed * 3.4 * dt;
    if(Math.random() < 0.7)
      spawnParticle(p.x, p.y, rand(-12,12), rand(-12,12), 0.25, '#6cf8ff', 2);
  } else {
    p.x += ix * p.speed * dt;
    p.y += iy * p.speed * dt;
  }

  p.x = clamp(p.x, 7, W-7);
  p.y = clamp(p.y, 10, H-7);

  if(p.dashCd > 0) p.dashCd -= dt;
  if(p.invuln > 0) p.invuln -= dt;
  p.fireTimer -= dt;
  p.angle = Math.atan2(mouse.y - p.y, mouse.x - p.x);

  if(mouse.down && p.fireTimer <= 0){
    p.fireTimer = p.fireRate;
    shoot();
  }
}

function shoot(){
  const p = player;
  const n = p.bullets;
  const spread = 0.11;
  for(let i=0;i<n;i++){
    const off = (i - (n-1)/2) * spread;
    const a = p.angle + off;
    game.bullets.push({
      x: p.x + Math.cos(a)*9,
      y: p.y + Math.sin(a)*9,
      vx: Math.cos(a)*p.bulletSpeed,
      vy: Math.sin(a)*p.bulletSpeed,
      r: 2, dmg: p.dmg, pierce: p.pierce,
      hit: new Set(), life: 1.5
    });
  }
  for(let i=0;i<3;i++){
    const a = p.angle + rand(-0.35,0.35);
    spawnParticle(p.x+Math.cos(a)*10, p.y+Math.sin(a)*10,
      Math.cos(a)*rand(30,80), Math.sin(a)*rand(30,80), 0.14, '#ffeeaa', 2);
  }
  game.shake = Math.max(game.shake, 1.1);
  sfx(540,0.05,'square',0.028,-320);
}

function hurtPlayer(dmg){
  const p = player;
  if(p.invuln > 0) return;
  p.hp -= dmg;
  p.invuln = 1.0;
  game.shake = 9;
  game.flash = 0.4;
  game.combo = 0; game.comboTimer = 0;
  sfx(130,0.2,'sawtooth',0.06,-70);
  for(let i=0;i<14;i++)
    spawnParticle(p.x, p.y, rand(-130,130), rand(-130,130), 0.5, '#ff4455', 2);
  if(p.hp <= 0){ p.hp = 0; gameOver(); }
}

/* ==================== 子弹 ==================== */
function updateBullets(dt){
  for(let i=game.bullets.length-1;i>=0;i--){
    const b = game.bullets[i];
    b.x += b.vx*dt; b.y += b.vy*dt; b.life -= dt;
    if(b.life <= 0 || b.x < -12 || b.x > W+12 || b.y < -12 || b.y > H+12){
      game.bullets.splice(i,1); continue;
    }
    let removed = false;
    for(let j=game.enemies.length-1;j>=0;j--){
      const e = game.enemies[j];
      if(b.hit.has(e.id)) continue;
      if(Math.hypot(b.x-e.x, b.y-e.y) < e.r + b.r){
        b.hit.add(e.id);
        damageEnemy(e, b.dmg, b.vx, b.vy);
        if(b.pierce > 0) b.pierce--;
        else { game.bullets.splice(i,1); removed = true; }
        break;
      }
    }
    if(removed) continue;
  }

  for(let i=game.ebullets.length-1;i>=0;i--){
    const b = game.ebullets[i];
    b.x += b.vx*dt; b.y += b.vy*dt; b.life -= dt;
    if(b.life <= 0 || b.x < -12 || b.x > W+12 || b.y < -12 || b.y > H+12){
      game.ebullets.splice(i,1); continue;
    }
    if(Math.hypot(b.x-player.x, b.y-player.y) < player.r + b.r){
      hurtPlayer(b.dmg);
      game.ebullets.splice(i,1);
    }
  }
}

function damageEnemy(e, dmg, vx, vy){
  e.hp -= dmg;
  e.flash = 0.1;
  const l = Math.hypot(vx,vy) || 1;
  e.kx += vx/l * 55;
  e.ky += vy/l * 55;
  for(let i=0;i<4;i++)
    spawnParticle(e.x, e.y, rand(-70,70), rand(-70,70), 0.3, e.color, 2);
  if(e.hp <= 0) killEnemy(e);
}

function killEnemy(e){
  const idx = game.enemies.indexOf(e);
  if(idx >= 0) game.enemies.splice(idx,1);
  game.kills++;
  game.combo++;
  game.comboTimer = 2.0;
  const mult = 1 + Math.floor(game.combo/6)*0.5;
  game.score += Math.round(e.xp * 10 * mult);
  for(let i=0;i<e.xp;i++){
    game.gems.push({
      x:e.x, y:e.y,
      vx: rand(-50,50), vy: rand(-50,50),
      value:1, life:22
    });
  }
  for(let i=0;i<11;i++)
    spawnParticle(e.x, e.y, rand(-120,120), rand(-120,120), 0.45, e.color, 2);
  game.shake = Math.max(game.shake, 2.6);
  sfx(rand(190,270), 0.1, 'square', 0.032, -110);
}

/* ==================== 敌人 ==================== */
function updateEnemies(dt){
  const p = player;
  for(let i=game.enemies.length-1;i>=0;i--){
    const e = game.enemies[i];
    e.flash = Math.max(0, e.flash - dt);

    const dx = p.x - e.x, dy = p.y - e.y;
    const d = Math.hypot(dx,dy) || 1;
    const nx = dx/d, ny = dy/d;

    if(e.type === 'shooter'){
      const want = 110;
      let mv = 0;
      if(d > want + 25) mv = 1;
      else if(d < want - 30) mv = -1;
      e.x += nx * e.speed * mv * dt;
      e.y += ny * e.speed * mv * dt;
      e.x += -ny * e.speed * 0.55 * dt * e.strafe;
      e.y +=  nx * e.speed * 0.55 * dt * e.strafe;
      e.shootTimer -= dt;
      if(e.shootTimer <= 0 && d < 230){
        e.shootTimer = Math.max(0.65, 1.9 - game.wave*0.075);
        const a = Math.atan2(dy,dx);
        game.ebullets.push({
          x:e.x, y:e.y,
          vx:Math.cos(a)*115, vy:Math.sin(a)*115,
          r:3, dmg:1, life:4
        });
        sfx(320,0.07,'sine',0.02,-120);
      }
    } else if(e.type === 'bat'){
      const t = game.t + e.seed;
      e.x += (nx*e.speed + Math.cos(t*6.5)*45) * dt;
      e.y += (ny*e.speed + Math.sin(t*6.5)*45) * dt;
    } else {
      e.x += nx * e.speed * dt;
      e.y += ny * e.speed * dt;
    }

    e.x += e.kx*dt; e.y += e.ky*dt;
    const damp = Math.max(0, 1 - dt*9);
    e.kx *= damp; e.ky *= damp;

    if(d < e.r + p.r && p.invuln <= 0){
      hurtPlayer(e.contact || 1);
    }
  }

  // 简单分离，避免完全重叠
  const list = game.enemies;
  for(let i=0;i<list.length;i++){
    const a = list[i];
    for(let j=i+1;j<list.length;j++){
      const b = list[j];
      const dx = b.x-a.x, dy = b.y-a.y;
      const d2 = dx*dx + dy*dy;
      const minD = a.r + b.r;
      if(d2 < minD*minD && d2 > 0.01){
        const dd = Math.sqrt(d2);
        const push = (minD - dd) * 0.5;
        const nx = dx/dd, ny = dy/dd;
        a.x -= nx*push; a.y -= ny*push;
        b.x += nx*push; b.y += ny*push;
      }
    }
  }
}

/* ==================== 经验 / 升级 ==================== */
function updateGems(dt){
  const p = player;
  for(let i=game.gems.length-1;i>=0;i--){
    const g = game.gems[i];
    g.life -= dt;
    const dx = p.x-g.x, dy = p.y-g.y;
    const d = Math.hypot(dx,dy) || 1;
    if(d < p.magnet){
      const pull = (1 - d/p.magnet) * 900 + 120;
      g.vx += dx/d * pull * dt;
      g.vy += dy/d * pull * dt;
    }
    g.vx *= (1 - dt*2.6);
    g.vy *= (1 - dt*2.6);
    g.x += g.vx*dt; g.y += g.vy*dt;

    if(d < 9){
      collectGem(g);
      game.gems.splice(i,1);
    } else if(g.life <= 0){
      game.gems.splice(i,1);
    }
  }
}

function collectGem(g){
  player.xp += g.value;
  sfx(680 + Math.random()*220, 0.045, 'square', 0.018);
  spawnParticle(g.x, g.y, 0, -40, 0.3, '#8fd8ff', 2);
  if(player.xp >= player.xpNext) levelUp();
}

function levelUp(){
  player.level++;
  player.xp -= player.xpNext;
  player.xpNext = Math.floor(player.xpNext*1.32 + 2);
  game.state = 'upgrade';
  game.choices = pickChoices(3);
  sfx(880,0.13,'square',0.05);
  setTimeout(()=>sfx(1180,0.16,'square',0.045), 90);
}

function pickChoices(n){
  const pool = UPGRADES.slice();
  const out = [];
  for(let i=0;i<n && pool.length;i++){
    out.push(pool.splice(Math.floor(Math.random()*pool.length),1)[0]);
  }
  return out;
}

function chooseUpgrade(i){
  const c = game.choices[i];
  if(!c) return;
  c.apply(player);
  sfx(720,0.08,'square',0.04,320);
  game.state = 'playing';
  game.choices = [];
  if(player.xp >= player.xpNext) levelUp();
}

/* ==================== 流程控制 ==================== */
function startGame(){
  player = newPlayer();
  uid = 1;
  game.state = 'playing';
  game.t = 0;
  game.wave = 0;
  game.phase = 'break';
  game.phaseTimer = 1.0;
  game.spawnQueue = [];
  game.spawnTimer = 0;
  game.enemies.length = 0;
  game.bullets.length = 0;
  game.ebullets.length = 0;
  game.gems.length = 0;
  game.particles.length = 0;
  game.texts.length = 0;
  game.score = 0; game.kills = 0;
  game.combo = 0; game.comboTimer = 0;
  game.shake = 0; game.flash = 0;
  game.choices = [];
}

function gameOver(){
  game.state = 'gameover';
  game.shake = 14;
  sfx(180,0.5,'sawtooth',0.07,-140);
  for(let i=0;i<30;i++)
    spawnParticle(player.x, player.y, rand(-180,180), rand(-180,180), rand(0.4,0.9), '#ff4455', 2);
}

function onClick(){
  initAudio();
  if(game.state === 'title'){ startGame(); return; }
  if(game.state === 'gameover'){ startGame(); return; }
  if(game.state === 'upgrade'){
    for(let i=0;i<game.choices.length;i++){
      const x = 25 + i*150, y = 85, w = 130, h = 140;
      if(mouse.x >= x && mouse.x <= x+w && mouse.y >= y && mouse.y <= y+h){
        chooseUpgrade(i); return;
      }
    }
  }
}

function onKeyPress(code){
  initAudio();
  if(code === 'Space') tryDash();
  if(code === 'KeyP' || code === 'Escape'){
    if(game.state === 'playing') game.state = 'paused';
    else if(game.state === 'paused') game.state = 'playing';
  }
  if(game.state === 'title' && (code === 'Enter' || code === 'Space')) startGame();
  if(game.state === 'gameover' && (code === 'Enter' || code === 'KeyR')) startGame();
  if(game.state === 'upgrade'){
    if(code === 'Digit1') chooseUpgrade(0);
    if(code === 'Digit2') chooseUpgrade(1);
    if(code === 'Digit3') chooseUpgrade(2);
  }
}

/* ==================== 更新 ==================== */
function update(dt){
  game.t += dt;
  game.shake = Math.max(0, game.shake - dt*38);
  game.flash = Math.max(0, game.flash - dt*2.2);
  if(game.comboTimer > 0){
    game.comboTimer -= dt;
    if(game.comboTimer <= 0) game.combo = 0;
  }

  updateParticles(dt);
  updateTexts(dt);

  if(game.state !== 'playing') return;

  updatePlayer(dt);
  updateBullets(dt);
  updateEnemies(dt);
  updateGems(dt);
  updateWave(dt);
}

/* ==================== 绘制 ==================== */
function drawPlayer(){
  const p = player;
  if(p.invuln > 0 && Math.floor(game.t*22) % 2 === 0) return;
  const x = Math.round(p.x), y = Math.round(p.y);

  ctx.fillStyle = 'rgba(0,0,0,0.35)';
  ctx.beginPath(); ctx.ellipse(x, y+9, 6, 2.5, 0, 0, 7); ctx.fill();

  // 枪口方向
  const a = p.angle;
  const gx = Math.round(x + Math.cos(a)*9);
  const gy = Math.round(y + Math.sin(a)*9);
  ctx.fillStyle = '#8aa0c0';
  ctx.fillRect(gx-2, gy-2, 4, 4);
  ctx.fillStyle = '#d8e8ff';
  ctx.fillRect(gx-1, gy-1, 2, 2);

  // 身体
  ctx.fillStyle = '#2b5fa8';
  ctx.fillRect(x-5, y-3, 10, 9);
  ctx.fillStyle = '#4a9eff';
  ctx.fillRect(x-4, y-2, 8, 6);
  // 头
  ctx.fillStyle = '#e2ecf8';
  ctx.fillRect(x-4, y-11, 8, 8);
  // 护目镜
  ctx.fillStyle = '#0b1428';
  ctx.fillRect(x-3, y-9, 6, 3);
  ctx.fillStyle = '#6cf8ff';
  ctx.fillRect(x-2, y-8, 2, 1);
  ctx.fillRect(x+1, y-8, 1, 1);

  // 冲刺冷却环
  if(p.dashCd > 0){
    const ratio = 1 - p.dashCd/p.dashMax;
    ctx.strokeStyle = 'rgba(110,220,255,0.55)';
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.arc(x, y, 12, -Math.PI/2, -Math.PI/2 + Math.PI*2*ratio);
    ctx.stroke();
  }
}

function drawEnemies(){
  for(const e of game.enemies){
    const x = Math.round(e.x), y = Math.round(e.y);
    const r = e.r;
    const col = e.flash > 0 ? '#ffffff' : e.color;

    ctx.fillStyle = 'rgba(0,0,0,0.3)';
    ctx.beginPath(); ctx.ellipse(x, y+r, r*0.8, r*0.3, 0, 0, 7); ctx.fill();

    if(e.type === 'slime'){
      ctx.fillStyle = col;
      ctx.fillRect(x-r+1, y-r+3, r*2-2, r*2-4);
      ctx.fillRect(x-r+2, y-r+1, r*2-4, r*2-1);
      ctx.fillStyle = '#06220c';
      ctx.fillRect(x-3, y-2, 2, 2);
      ctx.fillRect(x+1, y-2, 2, 2);
    } else if(e.type === 'bat'){
      const f = Math.sin(game.t*22 + e.seed) > 0 ? 1 : -1;
      ctx.fillStyle = col;
      ctx.fillRect(x-6, y-2-2*f, 4, 3);
      ctx.fillRect(x+2, y-2-2*f, 4, 3);
      ctx.fillRect(x-3, y-3, 6, 6);
      ctx.fillStyle = '#ffe0ff';
      ctx.fillRect(x-2, y-2, 1, 1);
      ctx.fillRect(x+1, y-2, 1, 1);
    } else if(e.type === 'shooter'){
      ctx.fillStyle = col;
      ctx.fillRect(x-r, y-r+2, r*2, r*2-2);
      ctx.fillStyle = '#3a1400';
      ctx.fillRect(x-4, y-2, 8, 4);
      const a = Math.atan2(player.y-e.y, player.x-e.x);
      ctx.fillStyle = '#ffe066';
      ctx.fillRect(Math.round(x+Math.cos(a)*r)-1, Math.round(y+Math.sin(a)*r)-1, 3, 3);
    } else if(e.type === 'tank'){
      ctx.fillStyle = col;
      ctx.fillRect(x-r, y-r, r*2, r*2);
      ctx.fillStyle = '#8ccfff';
      ctx.fillRect(x-r+2, y-r+2, r*2-4, r*2-4);
      ctx.fillStyle = '#03224a';
      ctx.fillRect(x-5, y-4, 10, 7);
      ctx.fillStyle = '#ffffff';
      ctx.fillRect(x-4, y-3, 2, 2);
      ctx.fillRect(x+2, y-3, 2, 2);
      ctx.fillStyle = '#aee4ff';
      ctx.fillRect(x-1, y-1, 2, 3);
    }

    if(e.hp < e.maxHp){
      const bw = e.r*2;
      ctx.fillStyle = '#2a0a10';
      ctx.fillRect(x-e.r, y-e.r-6, bw, 2);
      ctx.fillStyle = '#ff4d5e';
      ctx.fillRect(x-e.r, y-e.r-6, Math.max(1, bw*(e.hp/e.maxHp)), 2);
    }
  }
}

function drawBullets(){
  for(const b of game.bullets){
    const x = Math.round(b.x), y = Math.round(b.y);
    ctx.fillStyle = 'rgba(255,224,102,0.35)';
    ctx.fillRect(x-3, y-2, 6, 4);
    ctx.fillStyle = '#ffe066';
    ctx.fillRect(x-2, y-1, 4, 2);
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(x-1, y-1, 2, 2);
  }
  for(const b of game.ebullets){
    const x = Math.round(b.x), y = Math.round(b.y);
    ctx.fillStyle = 'rgba(255,80,80,0.3)';
    ctx.fillRect(x-3, y-3, 6, 6);
    ctx.fillStyle = '#ff5a5a';
    ctx.fillRect(x-2, y-2, 4, 4);
    ctx.fillStyle = '#ffc9c9';
    ctx.fillRect(x-1, y-1, 2, 2);
  }
}

function drawGems(){
  for(const g of game.gems){
    const x = Math.round(g.x), y = Math.round(g.y);
    const pulse = Math.sin(game.t*10 + g.x*0.3) * 1;
    ctx.fillStyle = 'rgba(90,200,255,0.35)';
    ctx.fillRect(x-3, y-3, 6, 6);
    ctx.fillStyle = '#5ad4ff';
    ctx.fillRect(x-2, y-2, 4, 4);
    ctx.fillStyle = '#d9f4ff';
    ctx.fillRect(x-1, y-1, 2, 2);
  }
}

function drawParticles(){
  for(const p of game.particles){
    const a = clamp(p.life/p.maxLife, 0, 1);
    ctx.globalAlpha = a;
    ctx.fillStyle = p.color;
    ctx.fillRect(Math.round(p.x), Math.round(p.y), p.size, p.size);
  }
  ctx.globalAlpha = 1;
}

function drawTexts(){
  ctx.textAlign = 'center';
  for(const t of game.texts){
    const a = clamp(t.life/t.maxLife, 0, 1);
    ctx.globalAlpha = a;
    ctx.fillStyle = t.color;
    ctx.font = 'bold ' + t.size + 'px "Courier New",monospace';
    ctx.fillText(t.text, t.x, t.y);
  }
  ctx.globalAlpha = 1;
  ctx.textAlign = 'left';
}

function drawHUD(){
  // 生命
  for(let i=0;i<player.maxHp;i++){
    const x = 8 + i*9, y = 12;
    const alive = i < player.hp;
    ctx.fillStyle = alive ? '#ff4d6d' : '#33202a';
    ctx.fillRect(x, y, 7, 5);
    ctx.fillRect(x+1, y-2, 5, 2);
    ctx.fillRect(x+1, y+5, 5, 2);
    if(alive){
      ctx.fillStyle = '#ff9db0';
      ctx.fillRect(x+1, y-1, 2, 2);
    }
  }

  // 经验条
  const xpRatio = clamp(player.xp / player.xpNext, 0, 1);
  ctx.fillStyle = '#0e1728';
  ctx.fillRect(0, 0, W, 3);
  ctx.fillStyle = '#3fa9f5';
  ctx.fillRect(0, 0, W*xpRatio, 3);
  ctx.fillStyle = '#8fd8ff';
  ctx.fillRect(0, 0, W*xpRatio, 1);

  // 右上信息
  ctx.textAlign = 'right';
  ctx.font = 'bold 10px "Courier New",monospace';
  ctx.fillStyle = '#e8f4ff';
  ctx.fillText('分数 ' + game.score, W-8, 14);
  ctx.fillStyle = '#7fa8d8';
  ctx.font = '9px "Courier New",monospace';
  ctx.fillText('波次 ' + Math.max(1,game.wave) + '  ·  Lv.' + player.level, W-8, 26);

  // 连击
  if(game.combo >= 3){
    ctx.textAlign = 'center';
    ctx.fillStyle = '#ffd166';
    ctx.font = 'bold 12px "Courier New",monospace';
    ctx.fillText('x' + game.combo + ' 连击', W/2, 22);
  }

  // 冲刺冷却指示
  ctx.textAlign = 'left';
  if(player.dashCd > 0){
    ctx.fillStyle = '#334a66';
    ctx.fillRect(8, H-12, 46, 4);
    ctx.fillStyle = '#6cf8ff';
    ctx.fillRect(8, H-12, 46*(1-player.dashCd/player.dashMax), 4);
  } else {
    ctx.fillStyle = '#6cf8ff';
    ctx.fillRect(8, H-12, 46, 4);
  }
  ctx.fillStyle = '#4a6a8a';
  ctx.font = '8px "Courier New",monospace';
  ctx.fillText('DASH', 58, H-8);

  // 波次间隙提示
  if(game.state === 'playing' && game.phase === 'break'){
    ctx.textAlign = 'center';
    ctx.fillStyle = '#8fb8e0';
    ctx.font = '10px "Courier New",monospace';
    ctx.fillText('下一波来袭… ' + Math.ceil(game.phaseTimer), W/2, H-26);
    ctx.textAlign = 'left';
  }
}

function wrapText(text, cx, y, maxW, lh){
  let line = '';
  let yy = y;
  for(const ch of text){
    const test = line + ch;
    if(ctx.measureText(test).width > maxW && line){
      ctx.fillText(line, cx, yy);
      line = ch; yy += lh;
    } else line = test;
  }
  if(line) ctx.fillText(line, cx, yy);
}

function drawUpgradeScreen(){
  ctx.fillStyle = 'rgba(5,8,16,0.9)';
  ctx.fillRect(0,0,W,H);

  ctx.textAlign = 'center';
  ctx.fillStyle = '#ffd166';
  ctx.font = 'bold 15px "Courier New",monospace';
  ctx.fillText('等级提升！选择一项强化', W/2, 48);
  ctx.fillStyle = '#4a6a8a';
  ctx.font = '9px "Courier New",monospace';
  ctx.fillText('点击卡片 或 按 1 / 2 / 3', W/2, 66);

  for(let i=0;i<game.choices.length;i++){
    const c = game.choices[i];
    const x = 25 + i*150, y = 85, w = 130, h = 140;
    const hover = mouse.x >= x && mouse.x <= x+w && mouse.y >= y && mouse.y <= y+h;

    ctx.fillStyle = hover ? '#1d3055' : '#111a2e';
    ctx.fillRect(x, y, w, h);
    ctx.strokeStyle = hover ? '#6cf8ff' : '#2a3a5a';
    ctx.lineWidth = 2;
    ctx.strokeRect(x+1, y+1, w-2, h-2);

    // 图标
    ctx.fillStyle = hover ? '#6cf8ff' : '#3a5a8a';
    ctx.fillRect(x+w/2-8, y+16, 16, 16);
    ctx.fillStyle = hover ? '#0b1428' : '#111a2e';
    ctx.fillRect(x+w/2-4, y+20, 8, 8);

    ctx.fillStyle = '#ffffff';
    ctx.font = 'bold 12px "Courier New",monospace';
    ctx.fillText(c.name, x+w/2, y+56);

    ctx.fillStyle = hover ? '#cfe6ff' : '#7f9ab8';
    ctx.font = '9px "Courier New",monospace';
    wrapText(c.desc, x+w/2, y+80, w-22, 13);

    ctx.fillStyle = '#3a5a8a';
    ctx.font = '10px "Courier New",monospace';
    ctx.fillText('[' + (i+1) + ']', x+w/2, y+h-12);
  }
  ctx.textAlign = 'left';
}

function drawGameOver(){
  ctx.fillStyle = 'rgba(22,4,12,0.85)';
  ctx.fillRect(0,0,W,H);
  ctx.textAlign = 'center';
  ctx.fillStyle = '#ff4d6d';
  ctx.font = 'bold 24px "Courier New",monospace';
  ctx.fillText('游戏结束', W/2, 108);

  ctx.fillStyle = '#dbe8f6';
  ctx.font = 'bold 13px "Courier New",monospace';
  ctx.fillText('得分  ' + game.score, W/2, 148);

  ctx.fillStyle = '#7f9ab8';
  ctx.font = '10px "Courier New",monospace';
  ctx.fillText('抵达波次 ' + game.wave + '   ·   击杀 ' + game.kills, W/2, 170);

  if(Math.floor(game.t*2) % 2 === 0){
    ctx.fillStyle = '#ffffff';
    ctx.font = '11px "Courier New",monospace';
    ctx.fillText('点击屏幕 或 按 R 重新开始', W/2, 214);
  }
  ctx.textAlign = 'left';
}

function drawPaused(){
  ctx.fillStyle = 'rgba(5,8,16,0.72)';
  ctx.fillRect(0,0,W,H);
  ctx.textAlign = 'center';
  ctx.fillStyle = '#8fd8ff';
  ctx.font = 'bold 18px "Courier New",monospace';
  ctx.fillText('暂 停', W/2, H/2 - 6);
  ctx.fillStyle = '#5a7a9a';
  ctx.font = '10px "Courier New",monospace';
  ctx.fillText('按 P 继续', W/2, H/2 + 18);
  ctx.textAlign = 'left';
}

function renderTitle(){
  ctx.fillStyle = 'rgba(5,8,16,0.78)';
  ctx.fillRect(0,0,W,H);

  ctx.textAlign = 'center';

  // 装饰像素方块
  for(let i=0;i<9;i++){
    const x = 60 + i*45;
    const y = 60 + Math.sin(game.t*2 + i)*6;
    ctx.fillStyle = ['#4a9eff','#6cf8ff','#ffd166'][i%3];
    ctx.globalAlpha = 0.25;
    ctx.fillRect(x, y, 4, 4);
  }
  ctx.globalAlpha = 1;

  ctx.fillStyle = '#4a9eff';
  ctx.font = 'bold 30px "Courier New",monospace';
  ctx.fillText('像素猎手', W/2, 120);

  ctx.fillStyle = '#2a4a7a';
  ctx.font = 'bold 11px "Courier New",monospace';
  ctx.fillText('P I X E L   H U N T E R', W/2, 142);

  if(Math.floor(game.t*2) % 2 === 0){
    ctx.fillStyle = '#ffffff';
    ctx.font = 'bold 13px "Courier New",monospace';
    ctx.fillText('点击屏幕开始', W/2, 192);
  }

  ctx.fillStyle = '#4a6a8a';
  ctx.font = '9px "Courier New",monospace';
  ctx.fillText('WASD 移动  ·  鼠标瞄准射击', W/2, 232);
  ctx.fillText('空格冲刺闪避  ·  击杀敌人升级强化', W/2, 248);
  ctx.fillText('在无尽的波次中活下去', W/2, 264);

  ctx.textAlign = 'left';
}

function render(){
  ctx.save();
  if(game.shake > 0.2){
    ctx.translate(rand(-game.shake,game.shake)|0, rand(-game.shake,game.shake)|0);
  }

  ctx.drawImage(bgCanvas, 0, 0);

  if(game.state === 'title'){
    ctx.restore();
    renderTitle();
    return;
  }

  drawGems();
  drawEnemies();
  drawBullets();
  if(game.state !== 'gameover') drawPlayer();
  drawParticles();
  drawTexts();

  ctx.restore();

  drawHUD();

  if(game.state === 'upgrade')  drawUpgradeScreen();
  if(game.state === 'gameover') drawGameOver();
  if(game.state === 'paused')   drawPaused();

  if(game.flash > 0){
    ctx.fillStyle = 'rgba(255,40,70,' + (game.flash*0.45).toFixed(3) + ')';
    ctx.fillRect(0,0,W,H);
  }
}

/* ==================== 主循环 ==================== */
let last = performance.now();
function loop(now){
  let dt = (now - last) / 1000;
  last = now;
  if(dt > 0.05) dt = 0.05;
  update(dt);
  render();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);

})();
</script>
</body>
</html>
