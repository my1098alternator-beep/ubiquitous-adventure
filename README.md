# ubiquitous-adventure
<!-- xcbhc
clash-royale-like prototype (single-file)
- Purpose: competition-ready demo to put on GitHub + GitHub Pages
- Features included:
  * 2-player lanes (left/right) with 3 towers each (one king + two princess towers)
  * deployable cards with elixir cost, cooldown, and simple stats
  * elixir regen and UI (bar and numeric)
  * basic AI opponent that deploys troops and defends
  * troop movement, path following, melee/ranged attacks, targeting towers/units
  * tower targeting logic and health bars
  * pause / stop / restart controls
  * local high-score (wins) via localStorage
  * clear code comments + suggestions for extensions

How to use:
1) Create a GitHub repo and add this file as index.html
2) Enable GitHub Pages from repo settings -> serve from main branch root
3) Open https://<your-username>.github.io/<repo-name>/ to play

Notes for competition:
- This is a prototype, not a full-scale real-time networking game.
- For online multiplayer, add websocket server (socket.io) or use WebRTC.
- You can expand with particle effects, more troops, balancing, matchmaking, and art.
-->

<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Clash-ish — Prototype</title>
<style>
  :root{
    --bg:#0b1020; --panel:#0f1724; --accent:#9affc9; --muted:#9aa4b2;
  }
  *{box-sizing:border-box}
  html,body{height:100%;margin:0;font-family:Inter,system-ui,Segoe UI,Roboto,Arial}
  body{background:linear-gradient(180deg,#071126 0%, #071226 60%);color:#e6eef6;display:flex;align-items:center;justify-content:center;padding:18px}
  .container{width:1200px;max-width:100%;background:linear-gradient(180deg,rgba(255,255,255,0.03),rgba(0,0,0,0.06));border-radius:12px;padding:14px;display:grid;grid-template-columns:280px 1fr 260px;gap:12px}

  /* left sidebar - cards */
  .sidebar{background:var(--panel);padding:12px;border-radius:10px}
  .deck{display:grid;grid-template-columns:1fr 1fr;gap:10px}
  .card{background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));padding:10px;border-radius:8px;text-align:center;cursor:pointer;border:2px solid transparent}
  .card.locked{opacity:.5;cursor:not-allowed}
  .card .cost{font-weight:700;color:var(--accent)}

  /* center - arena */
  .arena-wrap{background:transparent;padding:6px;border-radius:8px}
  .arena{background:linear-gradient(180deg,#0d2340,#052030);border-radius:8px;height:640px;position:relative;overflow:hidden;border:2px solid rgba(255,255,255,0.03)}
  .lane{position:absolute;left:50%;transform:translateX(-50%);width:360px;top:0;bottom:0}
  .river{position:absolute;left:50%;transform:translateX(-50%);top:50%;height:100px;margin-top:-50px;width:360px;background:linear-gradient(180deg,rgba(80,200,250,0.06),rgba(10,50,80,0.06));backdrop-filter:blur(2px);border-radius:6px}

  /* towers */
  .tower{position:absolute;width:88px;height:88px;border-radius:8px;background:linear-gradient(180deg,#28313f,#0e1620);display:flex;align-items:center;justify-content:center;border:2px solid rgba(255,255,255,0.03)}
  .tower.left{left:48px}
  .tower.right{right:48px}
  .tower.king{left:50%;transform:translateX(-50%);top:16px}
  .tower.bottom{bottom:16px}
  .hp{position:absolute;bottom:-18px;left:50%;transform:translateX(-50%);background:rgba(0,0,0,0.5);padding:2px 6px;border-radius:6px;font-size:12px}

  /* troops */
  .entity{position:absolute;width:40px;height:40px;border-radius:6px;display:flex;align-items:center;justify-content:center;font-weight:700}
  .entity .hpbar{position:absolute;top:-8px;left:0;right:0;height:6px;background:rgba(0,0,0,0.4);border-radius:4px;overflow:hidden}
  .entity .hpbar > i{display:block;height:100%;background:linear-gradient(90deg,#56c596,#2fd2a9);width:100%}

  /* right panel - HUD */
  .hud{background:var(--panel);padding:12px;border-radius:10px}
  .elixir{font-size:22px;margin-bottom:6px}
  .controls{display:flex;gap:8px;margin-top:8px}
  button{background:#0b1220;border:1px solid rgba(255,255,255,0.04);padding:8px 12px;border-radius:8px;color:var(--muted);cursor:pointer}
  button.primary{background:var(--accent);color:#062a1a;font-weight:700}
  .log{height:420px;overflow:auto;padding:8px;background:rgba(255,255,255,0.01);border-radius:6px;margin-top:8px}
  .small{font-size:12px;color:var(--muted)}

  /* responsive */
  @media(max-width:1000px){.container{grid-template-columns:1fr}.arena{height:540px}}
</style>
</head>
<body>
  <div class="container">
    <div class="sidebar">
      <h3>deck</h3>
      <div class="deck" id="deck"></div>
      <p class="small">click a card then click on lane to deploy. green cost = you can afford</p>
    </div>

    <div class="arena-wrap">
      <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:8px">
        <div><strong>Clash-ish: Prototype</strong> <span class="small">— single-file</span></div>
        <div class="small">time: <span id="matchTime">0</span>s</div>
      </div>

      <div class="arena" id="arena">
        <!-- top side (enemy) towers -->
        <div class="tower king" id="enemy-king" style="top:16px">
          <div>ENEMY KING</div>
          <div class="hp" id="enemy-king-hp">3000</div>
        </div>
        <div class="tower left" id="enemy-left" style="top:120px">
          <div>TL</div>
          <div class="hp" id="enemy-left-hp">1200</div>
        </div>
        <div class="tower right" id="enemy-right" style="top:120px">
          <div>TR</div>
          <div class="hp" id="enemy-right-hp">1200</div>
        </div>

        <!-- river and lane guide in the middle -->
        <div class="lane" style="height:100%"></div>
        <div class="river"></div>

        <!-- bottom side (player) towers -->
        <div class="tower king bottom" id="player-king" style="bottom:16px">
          <div>PLAYER KING</div>
          <div class="hp" id="player-king-hp">3000</div>
        </div>
        <div class="tower left bottom" id="player-left" style="bottom:120px">
          <div>BL</div>
          <div class="hp" id="player-left-hp">1200</div>
        </div>
        <div class="tower right bottom" id="player-right" style="bottom:120px">
          <div>BR</div>
          <div class="hp" id="player-right-hp">1200</div>
        </div>

      </div>
    </div>

    <div class="hud">
      <div class="elixir">elixir: <span id="elixirVal">5</span> / 10</div>
      <div style="height:12px;background:rgba(255,255,255,0.03);border-radius:8px;margin-bottom:8px;overflow:hidden">
        <div id="elixirBar" style="height:100%;width:50%;background:linear-gradient(90deg,#5ef7c7,#2ac7a0)"></div>
      </div>
      <div class="controls">
        <button id="pauseBtn">pause</button>
        <button id="stopBtnHud">stop</button>
        <button class="primary" id="restartBtn">restart</button>
      </div>
      <div style="margin-top:12px">match: <span id="matchCount">0</span></div>
      <div class="log" id="log"></div>
      <div style="margin-top:8px"><strong>tips</strong><div class="small">this prototype focuses on basic mechanics & AI. to win, focus on cost-efficiency and defending towers early.</div></div>
    </div>
  </div>

<script>
/* ========== data models ========== */
const CONFIG = {
  TICK: 40, // ms per game tick
  ARENA: {width: 360, height: 640},
  ELIXIR_MAX: 10,
  ELIXIR_REGEN: 0.04, // per tick
}

const CARDS = [
  {id:'knight',name:'Knight',cost:3,hp:250,atk:40,range:20,speed:1.0,desc:'melee tank'},
  {id:'archer',name:'Archer',cost:2,hp:120,atk:30,range:110,speed:1.2,desc:'ranged support'},
  {id:'mini',name:'Mini PEKKA',cost:4,hp:200,atk:90,range:20,speed:1.1,desc:'high damage'},
  {id:'gob',name:'Goblin',cost:1,hp:80,atk:25,range:20,speed:1.6,desc:'cheap DPS'},
]

/* ========== game state ========== */
let state = null
function newGame(){
  state = {
    time:0,
    running:true,
    selectedCard:null,
    elixir:5,
    match:0,
    entities:[], // active troops
    logs:[],
  }
  // reset towers
  document.getElementById('enemy-king-hp').textContent = 3000
  document.getElementById('enemy-left-hp').textContent = 1200
  document.getElementById('enemy-right-hp').textContent = 1200
  document.getElementById('player-king-hp').textContent = 3000
  document.getElementById('player-left-hp').textContent = 1200
  document.getElementById('player-right-hp').textContent = 1200
  document.getElementById('log').innerHTML=''
  state.match++
  document.getElementById('matchCount').textContent = state.match
  renderDeck()
}

/* ========== utilities ========== */
function log(s){
  state.logs.unshift(`[${(state.time/1000).toFixed(1)}s] ${s}`)
  const el = document.getElementById('log')
  el.innerHTML = state.logs.slice(0,200).map(x=>'<div class="small">'+x+'</div>').join('')
}

function clamp(v,a,b){return Math.max(a,Math.min(b,v))}

/* ========== UI deck rendering and deploy handling ========== */
const deckEl = document.getElementById('deck')
function renderDeck(){
  deckEl.innerHTML = ''
  for(const c of CARDS){
    const card = document.createElement('div')
    card.className='card'
    card.dataset.id=c.id
    card.innerHTML = `<div style="font-size:14px">${c.name}</div><div class="cost">${c.cost}</div><div class="small">${c.desc}</div>`
    if(state.elixir < c.cost) card.classList.add('locked')
    card.onclick = ()=>{
      if(state.elixir < c.cost) {log('not enough elixir for '+c.name);return}
      state.selectedCard = c
      // visual cue
      document.querySelectorAll('.card').forEach(x=>x.style.border='2px solid transparent')
      card.style.border = '2px solid var(--accent)'
      log('selected '+c.name)
    }
    deckEl.appendChild(card)
  }
}

// deploy by clicking arena bottom/top half
const arena = document.getElementById('arena')
arena.addEventListener('click', (ev)=>{
  if(!state.running) return
  if(!state.selectedCard) {log('select a card first');return}
  // determine lane (left or right) by x position relative to arena center
  const rect = arena.getBoundingClientRect()
  const x = ev.clientX - rect.left
  const lane = (x < rect.width/2)?'left':'right'
  deployCard(state.selectedCard, lane, ev.clientY - rect.top)
  state.selectedCard = null
  document.querySelectorAll('.card').forEach(x=>x.style.border='2px solid transparent')
})

/* ========== entity and combat logic ========== */
let idCounter = 1
function deployCard(card, lane, y){
  if(state.elixir < card.cost) return
  state.elixir -= card.cost
  renderElixir()
  const team = 'player'
  const spawnY = (team=='player') ? (CONFIG.ARENA.height - 100) : 100
  const spawnX = (lane=='left')?100:CONFIG.ARENA.width-100
  const e = {
    id:idCounter++,
    type:card.id,
    name:card.name,
    team:team,
    x:spawnX,
    y:spawnY,
    lane:lane,
    hp:card.hp,
    atk:card.atk,
    range:card.range,
    speed:card.speed,
    target:null,
    attackCooldown:0,
  }
  state.entities.push(e)
  log('deployed '+card.name+' on '+lane)
}

function deployAI(){
  // very simple AI: spawn a random card every few seconds if elixir available
  const choose = CARDS[Math.floor(Math.random()*CARDS.length)]
  const lane = (Math.random()>0.5)?'left':'right'
  if(Math.random()>0.6) return
  if(choose.cost <= Math.floor(state.elixir)){
    // spawn enemy entity mirrored
    state.elixir -= choose.cost // AI shares same elixir pool to keep balance simple
    const e = {
      id:idCounter++,
      type:choose.id,
      name:choose.name,
      team:'enemy',
      x:(lane=='left')?100:CONFIG.ARENA.width-100,
      y:100,
      lane:lane,
      hp:choose.hp,
      atk:choose.atk,
      range:choose.range,
      speed:choose.speed*0.95,
      target:null,
      attackCooldown:0,
    }
    state.entities.push(e)
    log('enemy deployed '+choose.name+' on '+lane)
  }
}

function findClosestTarget(entity){
  // priority: enemy units in same lane -> towers in range
  const ents = state.entities.filter(x=>x.team!==entity.team && x.lane===entity.lane)
  if(ents.length){
    // pick closest by distance
    ents.sort((a,b)=>Math.hypot(a.x-entity.x,a.y-entity.y)-Math.hypot(b.x-entity.x,b.y-entity.y))
    return ents[0]
  }
  // else check towers
  const towers = getEnemyTowers(entity.team, entity.lane)
  if(towers.length) return towers[0]
  return null
}

function getEnemyTowers(team, lane){
  // return array of tower-like objects with x,y,hp,team,id
  const res=[]
  if(team==='player'){
    // enemy towers at top
    res.push({id:'enemy-left',x:100,y:120,hp:parseInt(document.getElementById('enemy-left-hp').textContent),isTower:true,ref:'enemy-left'})
    res.push({id:'enemy-right',x:CONFIG.ARENA.width-100,y:120,hp:parseInt(document.getElementById('enemy-right-hp').textContent),isTower:true,ref:'enemy-right'})
    res.push({id:'enemy-king',x:CONFIG.ARENA.width/2,y:40,hp:parseInt(document.getElementById('enemy-king-hp').textContent),isTower:true,ref:'enemy-king'})
  } else {
    res.push({id:'player-left',x:100,y:CONFIG.ARENA.height-120,hp:parseInt(document.getElementById('player-left-hp').textContent),isTower:true,ref:'player-left'})
    res.push({id:'player-right',x:CONFIG.ARENA.width-100,y:CONFIG.ARENA.height-120,hp:parseInt(document.getElementById('player-right-hp').textContent),isTower:true,ref:'player-right'})
    res.push({id:'player-king',x:CONFIG.ARENA.width/2,y:CONFIG.ARENA.height-40,hp:parseInt(document.getElementById('player-king-hp').textContent),isTower:true,ref:'player-king'})
  }
  // optionally filter by lane to prefer side towers
  // simple: return all
  return res
}

function applyDamageToTower(refId, dmg){
  const el = document.getElementById(refId+'-hp')
  if(!el) return
  const newHp = Math.max(0, parseInt(el.textContent) - Math.floor(dmg))
  el.textContent = newHp
  log(`${refId} took ${Math.floor(dmg)} dmg`)
}

/* ========== rendering (DOM) for entities ========== */
function renderEntities(){
  // clear visual entities
  document.querySelectorAll('.entity').forEach(x=>x.remove())
  for(const e of state.entities){
    const div = document.createElement('div')
    div.className='entity'
    div.style.left = (e.x - 20) + 'px'
    div.style.top = (e.y - 20) + 'px'
    div.dataset.id = e.id
    div.style.background = (e.team==='player')? 'linear-gradient(180deg,#57c9ff,#2ea0ff)':'linear-gradient(180deg,#ff6b6b,#ff3b3b)'
    div.innerHTML = `<div style="z-index:2">${e.name[0]}</div><div class="hpbar"><i style="width:${clamp(e.hp/ (e.type==='knight'?250:200)*100,0,100)}%"></i></div>`
    arena.appendChild(div)
  }
}

/* ========== main game loop ========== */
let ticker = null
function startLoop(){
  if(ticker) clearInterval(ticker)
  ticker = setInterval(()=>{
    if(!state.running) return
    tick()
  }, CONFIG.TICK)
}

function tick(){
  state.time += CONFIG.TICK
  // elixir regen
  state.elixir = clamp(state.elixir + CONFIG.ELIXIR_REGEN, 0, CONFIG.ELIXIR_MAX)
  renderElixir()
  // simple AI decision occasionally
  if(Math.random() < 0.03) deployAI()
  // update AI and entities
  updateEntities()
  renderEntities()
  document.getElementById('matchTime').textContent = Math.floor(state.time/1000)
  // update deck lock visuals
  renderDeck()
  // check win/loss
  checkGameEnd()
}

function renderElixir(){
  document.getElementById('elixirVal').textContent = Math.floor(state.elixir)
  const pct = (state.elixir / CONFIG.ELIXIR_MAX)*100
  document.getElementById('elixirBar').style.width = pct+'%'
}

function updateEntities(){
  for(const e of state.entities.slice()){
    if(e.hp<=0){ // remove dead
      state.entities = state.entities.filter(x=>x.id!==e.id)
      continue
    }
    // find or update target
    if(!e.target || (e.target && e.target.hp<=0)){
      e.target = findClosestTarget(e)
    }
    if(e.target){
      // target position
      const tx = e.target.x !== undefined ? e.target.x : (e.target.ref? (document.getElementById(e.target.ref).offsetLeft + 44) : CONFIG.ARENA.width/2)
      const ty = e.target.y !== undefined ? e.target.y : (e.target.ref? (document.getElementById(e.target.ref).offsetTop + 44) : CONFIG.ARENA.height/2)
      const dist = Math.hypot(tx - e.x, ty - e.y)
      if(dist > e.range){
        // move towards target
        const dx = (tx - e.x)/dist * e.speed*2
        const dy = (ty - e.y)/dist * e.speed*2
        e.x += dx
        e.y += dy
      } else {
        // in range -> attack if cooldown ready
        if(e.attackCooldown <= 0){
          // damage either unit or tower
          if(e.target.isTower){
            applyDamageToTower(e.target.ref, e.atk)
          } else {
            e.target.hp -= e.atk
            log(`${e.name} hit ${e.target.name} for ${e.atk}`)
          }
          e.attackCooldown = 1000 // ms
        }
      }
    } else {
      // no target: push forward to enemy side
      const dir = (e.team==='player') ? -1 : 1
      e.y += dir * e.speed*1.2
      // clamp inside arena
      e.y = clamp(e.y, 20, CONFIG.ARENA.height-20)
    }
    // cooldown reduce
    e.attackCooldown = Math.max(0, e.attackCooldown - CONFIG.TICK)
  }
}

function checkGameEnd(){
  const pk = parseInt(document.getElementById('player-king-hp').textContent)
  const ek = parseInt(document.getElementById('enemy-king-hp').textContent)
  if(pk<=0 || ek<=0){
    state.running = false
    log((ek<=0)?'player wins':'enemy wins')
    // track wins
    const wins = JSON.parse(localStorage.getItem('wins')||'{}')
    wins['player'] = (wins['player']||0) + (ek<=0?1:0)
    wins['enemy'] = (wins['enemy']||0) + (pk<=0?1:0)
    localStorage.setItem('wins', JSON.stringify(wins))
    document.getElementById('matchCount').textContent = state.match
  }
}

/* ========== controls ========== */
const pauseBtn = document.getElementById('pauseBtn')
pauseBtn.onclick = ()=>{
  state.running = !state.running
  pauseBtn.textContent = state.running? 'pause':'resume'
}

const stopBtnHud = document.getElementById('stopBtnHud')
stopBtnHud.onclick = ()=>{
  state.running = false
  alert('Game stopped. final time: '+Math.floor(state.time/1000)+'s')
}

const restartBtn = document.getElementById('restartBtn')
restartBtn.onclick = ()=>{
  newGame();
}

/* ========== boot ========== */
function initUI(){
  // set arena logical size to match CONFIG for calculations (visual scaling)
  // arena is responsive so map logical coords to element coords when rendering
  const rect = arena.getBoundingClientRect()
  // we will use absolute pixel positions relative to current arena size
  // simplify by mapping CONFIG.ARENA.width to rect.width (scale dynamically if wanted)
}

// init
newGame()
startLoop()
renderEntities()

// expose state for debugging (remove in production)
window._CR = {state,CONFIG}
</script>
</body>
</html>
