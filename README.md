<!doctype html>
<html lang="id">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Game Joy - Clone (Customizable) + Login & Realtime Leaderboard</title>
  <style>
    :root{ --bg-color:#f7f7f7; }
    html,body{height:100%;margin:0;font-family:Inter,system-ui,Arial;background:var(--bg-color);display:flex;align-items:center;justify-content:center}
    .wrap{max-width:960px;width:95%;}
    canvas{display:block;width:100%;height:auto;border:1px solid #ddd;background:transparent}
    .info{margin-top:10px;color:#333;font-size:14px}
    .controls{margin-top:8px}
    .controls small{display:block;color:#666}

    /* simple UI boxes */
    .ui-row{display:flex;gap:8px;align-items:center;margin-top:10px}
    .card{background:#fff;padding:10px;border-radius:8px;box-shadow:0 6px 18px rgba(0,0,0,0.06)}
    .login{display:flex;gap:8px;align-items:center}
    input[type=text]{padding:6px 8px;border-radius:4px;border:1px solid #ccc}
    button{padding:6px 10px;border-radius:6px;border:none;cursor:pointer;background:#2d8cff;color:#fff}
    .leaderboard{margin-top:12px}
    .leaderboard ol{padding-left:18px}
    .muted{color:#666;font-size:13px}
  </style>
</head>
<body>
  <div class="wrap">
    <canvas id="game" width="800" height="200" aria-label="game-joy clone game"></canvas>

    <div class="info card">
      <div style="display:flex;justify-content:space-between;align-items:center">
        <div>
          <strong>Controls:</strong> Space / ArrowUp to jump, ArrowDown to duck, Click/Tap to jump.
        </div>
        <div id="userStatus" class="muted">Not signed</div>
      </div>

      <div class="ui-row">
        <div class="login">
          <input id="usernameInput" type="text" placeholder="Nama (public)" />
          <button id="btnLogin">Login</button>
          <button id="btnLogout" style="display:none;background:#e14a4a">Logout</button>
          <button id="btnWebAuth" title="Use device passkey/webauthn" style="background:#6b6bd6">Passkey</button>
        </div>
      </div>

      <div class="muted">Auto-login: browser will remember your display name on this device (localStorage). To implement secure account linking, integrate a backend identity provider (Firebase Auth, OAuth, etc.).</div>

      <div class="leaderboard card" style="margin-top:10px">
        <strong>Global Leaderboard (Top 10, realtime)</strong>
        <ol id="leaderboardList"><li class="muted">Connecting...</li></ol>
      </div>

      <div class="controls">
        <small>Customize images in the <code>images</code> object inside the script (look for `IMAGES TO REPLACE`).</small>
      </div>
    </div>
  </div>

<!--
  Notes for developers:
  - This single-file demo adds simple client-side login (localStorage) with optional WebAuthn placeholder.
  - Realtime leaderboard uses Firebase Firestore listeners. You MUST create a Firebase project and replace the CONFIG below with your project's config. Firestore rules should allow secure writes only from authenticated clients in production.
  - For production: do server-side validation of submitted scores to prevent cheating.
-->

<script>
/* ---------------------------
   Configuration & sizes (same as before)
*/
const CONFIG = {
  baseWidth: 800,
  baseHeight: 200,
  groundY: 150,
  gravity: 0.6,
  jumpVelocity: -12,
  speedStart: 6,
  speedIncreaseRate: 0.002
};

// --- IMAGES TO REPLACE ---
const images = {
  background: 'Artboard 1.png',
  player: 'Artboard 5.png',
  obstacleSmall: 'Artboard 2.png',
  obstacleLarge: 'Artboard 3.png',
  flyer: 'Artboard 4.png',
  ground: 'images/ground.png'
};

/* ---------------------------
   Firebase (Realtime leaderboard)
   ---------------------------
   - Replace the firebaseConfig object with your project's config.
   - You can use Firebase Firestore with a collection `scores` where each doc has {name: string, score: number, ts: timestamp}
   - Firestore will be used only from client in this demo. For production you should secure writes using Firebase Auth or a server.
*/

// --- FIREBASE CONFIG (REPLACE with your project's settings) ---
const FIREBASE_CONFIG = {
  apiKey: "REPLACE_WITH_YOUR_API_KEY",
  authDomain: "REPLACE_WITH_YOUR_AUTH_DOMAIN",
  projectId: "REPLACE_WITH_YOUR_PROJECT_ID",
  storageBucket: "REPLACE_WITH_YOUR_STORAGE_BUCKET",
  messagingSenderId: "REPLACE_WITH_MESSAGING_SENDER_ID",
  appId: "REPLACE_WITH_APP_ID"
};

// Firestore variables (initialized later if config set)
let firestore = null;
let firebaseApp = null;

// If user supplies real Firebase config, we'll dynamically load Firebase SDK and init.
async function initFirebase(){
  if(!FIREBASE_CONFIG.apiKey || FIREBASE_CONFIG.apiKey.startsWith('REPLACE')) return false;
  // load compat SDKs
  await loadScript('https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js');
  await loadScript('https://www.gstatic.com/firebasejs/9.22.2/firebase-firestore-compat.js');
  firebaseApp = firebase.initializeApp(FIREBASE_CONFIG);
  firestore = firebase.firestore();
  console.log('Firebase initialized');
  return true;
}
function loadScript(src){
  return new Promise((res,rej)=>{
    const s = document.createElement('script'); s.src = src; s.onload = res; s.onerror = rej; document.head.appendChild(s);
  });
}

/* ---------------------------
   Simple client 'login' using localStorage + optional WebAuthn placeholder
   - This stores only a public display name locally (not secure account linking).
   - For real accounts, integrate Firebase Auth / OAuth and require user verification.
*/
const UI = {
  usernameInput: null, btnLogin: null, btnLogout: null, btnWebAuth: null, userStatus: null, leaderboardList: null
};

function setupUI(){
  UI.usernameInput = document.getElementById('usernameInput');
  UI.btnLogin = document.getElementById('btnLogin');
  UI.btnLogout = document.getElementById('btnLogout');
  UI.btnWebAuth = document.getElementById('btnWebAuth');
  UI.userStatus = document.getElementById('userStatus');
  UI.leaderboardList = document.getElementById('leaderboardList');

  UI.btnLogin.addEventListener('click', ()=>{
    const name = (UI.usernameInput.value || '').trim();
    if(!name) return alert('Masukkan nama untuk tampil di leaderboard');
    localStorage.setItem('dino_display_name', name);
    setUser(name);
  });
  UI.btnLogout.addEventListener('click', ()=>{
    localStorage.removeItem('dino_display_name');
    setUser(null);
  });
  UI.btnWebAuth.addEventListener('click', async ()=>{
    // WebAuthn placeholder: in real app you'd call navigator.credentials.create/get with a server challenge.
    alert('WebAuthn/passkey flow: perlu integrasi server. This demo shows a placeholder.');
  });

  // auto-login if name stored
  const saved = localStorage.getItem('dino_display_name');
  if(saved) { setUser(saved); UI.usernameInput.value = saved; }
  else setUser(null);
}

function setUser(name){
  if(name){ UI.userStatus.textContent = 'Signed as: ' + name; UI.btnLogout.style.display = 'inline-block'; }
  else { UI.userStatus.textContent = 'Not signed'; UI.btnLogout.style.display = 'none'; }
}

/* ---------------------------
   Leaderboard functions (uses firestore)
*/
async function submitScoreToLeaderboard(name, score){
  if(!firestore){ console.warn('Firestore not initialized. Score not sent.'); return; }
  try{
    await firestore.collection('scores').add({ name: name || 'anonymous', score: score, ts: firebase.firestore.FieldValue.serverTimestamp() });
    console.log('Score submitted');
  } catch(e){ console.error('Failed to submit score', e); }
}

function listenLeaderboard(){
  if(!firestore){ UI.leaderboardList.innerHTML = '<li class="muted">Enable Firebase config to see leaderboard.</li>'; return; }
  // Query top 10 scores descending; if same score, latest ts
  firestore.collection('scores').orderBy('score','desc').limit(10).onSnapshot((snap)=>{
    const items = [];
    snap.forEach(doc=>{ const d = doc.data(); items.push({name: d.name, score: d.score}); });
    renderLeaderboard(items);
  }, (err)=>{ console.error(err); UI.leaderboardList.innerHTML = '<li class="muted">Error loading leaderboard</li>'; });
}

function renderLeaderboard(items){
  if(!items || items.length===0) { UI.leaderboardList.innerHTML = '<li class="muted">No scores yet</li>'; return; }
  UI.leaderboardList.innerHTML = items.map(i=>`<li>${escapeHtml(i.name)} — ${i.score}</li>`).join('');
}
function escapeHtml(s){ return String(s).replace(/[&<>"]/g, (m)=>({ '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;' }[m])); }

/* ---------------------------
   Game code (same logic as before but emitting score to leaderboard on game over)
*/
let canvas = document.getElementById('game');
let ctx = canvas.getContext('2d');
const W = CONFIG.baseWidth; const H = CONFIG.baseHeight; canvas.width = W; canvas.height = H;
function clear() { ctx.clearRect(0,0,W,H); }
function loadImage(src){ return new Promise((resolve)=>{ const img = new Image(); img.onload = ()=>resolve(img); img.onerror = ()=>resolve(null); img.src = src; }); }

(async function main(){
  setupUI();
  const firebaseOk = await initFirebase();
  if(firebaseOk) listenLeaderboard(); else { listenLeaderboard(); }

  const [bgImg, playerImg, obsSmallImg, obsLargeImg, flyerImg, groundImg] = await Promise.all([
    loadImage(images.background), loadImage(images.player), loadImage(images.obstacleSmall), loadImage(images.obstacleLarge), loadImage(images.flyer), loadImage(images.ground)
  ]);
  const assets = { bgImg, playerImg, obsSmallImg, obsLargeImg, flyerImg, groundImg };
  startGame(assets);
})();

function startGame(assets){
  let speed = CONFIG.speedStart; let frame = 0; let score = 0; let running = true;
  const player = { x:50, y:CONFIG.groundY-47, w:64, h:67, vy:0, onGround:true, ducking:false };
  const obstacles = [];
  let spawnTimer=0; let spawnInterval=90;

  const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  function playJump(){ const o = audioCtx.createOscillator(), g = audioCtx.createGain(); o.type='sine'; o.frequency.value=450; g.gain.setValueAtTime(0.0001,audioCtx.currentTime); g.gain.exponentialRampToValueAtTime(0.15,audioCtx.currentTime+0.01); g.gain.exponentialRampToValueAtTime(0.0001,audioCtx.currentTime+0.25); o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime+0.3); }
  function playDie(){ const o = audioCtx.createOscillator(), g = audioCtx.createGain(); o.type='square'; o.frequency.value=120; g.gain.setValueAtTime(0.0001,audioCtx.currentTime); g.gain.linearRampToValueAtTime(0.2,audioCtx.currentTime+0.02); g.gain.exponentialRampToValueAtTime(0.0001,audioCtx.currentTime+0.5); o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime+0.6); }
  function playPoint(){ const o = audioCtx.createOscillator(), g = audioCtx.createGain(); o.type='sine'; o.frequency.value=800; g.gain.setValueAtTime(0.0001,audioCtx.currentTime); g.gain.exponentialRampToValueAtTime(0.12,audioCtx.currentTime+0.01); g.gain.exponentialRampToValueAtTime(0.0001,audioCtx.currentTime+0.12); o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime+0.16); }

  function spawnObstacle(){ const kind = Math.random()<0.2?'flyer':(Math.random()<0.5?'large':'small'); if(kind==='small'){ obstacles.push({x:W+10,kind:'small',w:assets.obsSmallImg?assets.obsSmallImg.width:25,h:assets.obsSmallImg?assets.obsSmallImg.height:50,y:CONFIG.groundY-(assets.obsSmallImg?assets.obsSmallImg.height:50)}); } else if(kind==='large'){ obstacles.push({x:W+10,kind:'large',w:assets.obsLargeImg?assets.obsLargeImg.width:50,h:assets.obsLargeImg?assets.obsLargeImg.height:70,y:CONFIG.groundY-(assets.obsLargeImg?assets.obsLargeImg.height:70)}); } else { const flyY = CONFIG.groundY - (assets.flyerImg?assets.flyerImg.height+60:90); obstacles.push({x:W+10,kind:'flyer',w:assets.flyerImg?assets.flyerImg.width:46,h:assets.flyerImg?assets.flyerImg.height:40,y:flyY}); } }

  function update(){ if(!running) return; frame++; speed += CONFIG.speedIncreaseRate; player.vy += CONFIG.gravity; player.y += player.vy; if(player.y > CONFIG.groundY - (player.ducking ? player.h/2 : player.h)){ player.y = CONFIG.groundY - (player.ducking ? player.h/2 : player.h); player.vy = 0; player.onGround = true; } else player.onGround = false; spawnTimer++; if(spawnTimer>spawnInterval){ spawnTimer=0; spawnInterval = 60 + Math.floor(Math.random()*120); spawnObstacle(); } for(let i=obstacles.length-1;i>=0;i--){ const o = obstacles[i]; o.x -= speed; if(o.x + o.w < -50) obstacles.splice(i,1); if(rectIntersect(player.x, player.y, player.w*(player.ducking?0.8:1), player.ducking?player.h/2:player.h, o.x, o.y, o.w, o.h)){ running=false; playDie(); onGameOver(); } } if(frame%6===0){ score++; if(score%100===0) playPoint(); } }
  function rectIntersect(ax,ay,aw,ah,bx,by,bw,bh){ return ax < bx + bw && ax + aw > bx && ay < by + bh && ay + ah > by; }

  function draw(){ clear(); if(assets.bgImg){ const tileW = assets.bgImg.width || W; for(let x=0;x<W;x+=tileW){ ctx.drawImage(assets.bgImg, x, 0, tileW, H); } } else { const g = ctx.createLinearGradient(0,0,0,H); g.addColorStop(0,'#e9e9e9'); g.addColorStop(1,'#f7f7f7'); ctx.fillStyle = g; ctx.fillRect(0,0,W,H); } ctx.fillStyle='rgba(0,0,0,0.06)'; ctx.fillRect(0, CONFIG.groundY, W, 2); for(const o of obstacles){ if(o.kind==='small' && assets.obsSmallImg){ ctx.drawImage(assets.obsSmallImg, o.x, o.y, o.w, o.h); } else if(o.kind==='large' && assets.obsLargeImg){ ctx.drawImage(assets.obsLargeImg, o.x, o.y, o.w, o.h); } else if(o.kind==='flyer' && assets.flyerImg){ ctx.drawImage(assets.flyerImg, o.x, o.y, o.w, o.h); } else { ctx.fillStyle='#444'; ctx.fillRect(o.x, o.y, o.w, o.h); } }
    if(assets.playerImg){ const drawH = player.ducking ? player.h/2 : player.h; ctx.drawImage(assets.playerImg, player.x, player.y + (player.ducking?player.h/2:0), player.w, drawH); } else { ctx.fillStyle='#222'; const drawH = player.ducking ? player.h/2 : player.h; ctx.fillRect(player.x, player.y + (player.ducking?player.h/2:0), player.w, drawH); }
    ctx.fillStyle='#111'; ctx.font='16px monospace'; ctx.fillText('Score: ' + score, W - 140, 30);
    if(!running){ ctx.fillStyle='rgba(0,0,0,0.6)'; ctx.fillRect(W/2 - 120, H/2 - 30, 240, 60); ctx.fillStyle='#fff'; ctx.font='18px sans-serif'; ctx.textAlign='center'; ctx.fillText('Game Over — Click/Space to restart', W/2, H/2 + 6); ctx.textAlign='start'; }
  }

  function jump(){ if(!running){ restart(); return; } if(player.onGround){ player.vy = CONFIG.jumpVelocity; player.onGround = false; playJump(); } }
  function duck(state){ player.ducking = state && player.onGround; }
  document.addEventListener('keydown',(e)=>{ if(e.code==='Space' || e.code==='ArrowUp'){ e.preventDefault(); jump(); } if(e.code==='ArrowDown'){ e.preventDefault(); duck(true); } });
  document.addEventListener('keyup',(e)=>{ if(e.code==='ArrowDown'){ duck(false); } });
  canvas.addEventListener('mousedown',()=>{ jump(); });
  canvas.addEventListener('touchstart',(e)=>{ e.preventDefault(); jump(); }, {passive:false});

  function restart(){ obstacles.length=0; score=0; speed=CONFIG.speedStart; frame=0; spawnTimer=0; spawnInterval=90; running=true; player.y = CONFIG.groundY - player.h; player.vy=0; player.ducking=false; playPoint(); }

  // when game over, push score to leaderboard (if user signed and firebase enabled)
  function onGameOver(){
    const name = localStorage.getItem('dino_display_name') || 'anonymous';
    if(firestore){ submitScoreToLeaderboard(name, score); }
    // also update leaderboard UI locally while waiting for realtime update
    // show a little message
    UI.leaderboardList.insertAdjacentHTML('afterbegin', `<li><em>Submitted: ${escapeHtml(name)} — ${score}</em></li>`);
  }

  function loop(){ update(); draw(); requestAnimationFrame(loop); }
  loop();
}
</script>
</body>
</html>
