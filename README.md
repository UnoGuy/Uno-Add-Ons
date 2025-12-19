<!DOCTYPE html><html lang="en">
<head>
<meta charset="UTF-8" />
<title>UNO Advanced Ultimate PRO</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<style>
:root{--bg:#0b3d2e;--panel:#06261c}
body{margin:0;font-family:sans-serif;background:var(--bg);color:#fff}
header{padding:10px;text-align:center;background:var(--panel)}
#game{padding:10px}
.row{display:flex;gap:6px;flex-wrap:wrap}
.card{min-width:64px;padding:10px;border-radius:10px;text-align:center;cursor:pointer;font-size:12px;transition:.3s}
.card.played{animation:flip .4s}
@keyframes flip{0%{transform:rotateY(0)}50%{transform:rotateY(90deg)}100%{transform:rotateY(0)}}
.red{background:#c0392b}.blue{background:#2980b9}.green{background:#27ae60}.yellow{background:#f1c40f;color:#000}.wild{background:#555}
button{padding:10px 14px;margin:4px;border-radius:10px;border:none;font-size:14px}
.panel{background:#111;padding:8px;border-radius:10px;margin-top:8px}
#log{background:#000;padding:6px;height:120px;overflow:auto;font-size:11px;border-radius:8px}
.modal{position:fixed;inset:0;background:rgba(0,0,0,.7);display:none;align-items:center;justify-content:center}
.modal>div{background:#111;padding:16px;border-radius:14px;max-width:90%}
.stack{display:flex;gap:-20px}
.stack .card{transform:translateX(calc(var(--i)*-10px))}
</style>
</head>
<body>
<header>
<h2>UNO Advanced Ultimate PRO</h2>
<div id="status"></div>
<button onclick="openSettings()">Settings</button>
<button onclick="openDifficulty()">AI Difficulty</button>
<button onclick="callUNO()">UNO!</button>
</header><div id="game">
<div id="table" class="panel"></div>
<div id="stackPile" class="panel"></div>
<div id="players" class="panel"></div>
<h3>Your Hand</h3>
<div id="hand" class="row"></div>
<button onclick="drawCard()">Draw</button>
<button onclick="endTurn()">End Turn</button>
<h3>Log</h3>
<div id="log"></div>
</div><!-- SETTINGS --><div class="modal" id="settingsModal">
<div>
<h3>Settings</h3>
<label>Music <input type="range" min="0" max="1" step="0.1" onchange="music.volume=this.value"></label><br><br>
<label>SFX <input type="range" min="0" max="1" step="0.1" onchange="sfxVolume=this.value"></label><br><br>
<button onclick="resetStats()">Reset Stats</button><br><br>
<button onclick="closeSettings()">Close</button>
</div></div><!-- DIFFICULTY --><div class="modal" id="difficultyModal"><div>
<h3>AI Difficulty</h3>
<select id="aiLevel">
<option value="easy">Easy</option>
<option value="normal" selected>Normal</option>
<option value="hard">Hard</option>
</select><br><br>
<button onclick="applyDifficulty()">Apply</button>
<button onclick="closeDifficulty()">Close</button>
</div></div><script>
/******** AUDIO ********/
let sfxVolume=1;
const sndPlay=new Audio('https://actions.google.com/sounds/v1/cartoon/wood_plank_flicks.ogg');
const sndDraw=new Audio('https://actions.google.com/sounds/v1/cartoon/pop.ogg');
const sndUNO=new Audio('https://actions.google.com/sounds/v1/human_voices/human_grunt.ogg');
const music=new Audio('https://actions.google.com/sounds/v1/ambiences/game_room.ogg');
music.loop=true;music.volume=.4;music.play();

/******** STATS & ACHIEVEMENTS ********/
let stats=JSON.parse(localStorage.getItem('unoStats')||'{"wins":0,"unoCalls":0,"draws":0}');
let achievements=JSON.parse(localStorage.getItem('unoAch')||'{}');
function unlock(a){if(!achievements[a]){achievements[a]=true;log('🏆 Achievement unlocked: '+a);localStorage.setItem('unoAch',JSON.stringify(achievements))}}

/******** CORE STATE ********/
const COLORS=['red','yellow','green','blue'];
let players=[],current=0,direction=1,drawPile=[],discard=[],pendingDraw=0;
let unoCalled=false;

const log=m=>{let l=document.getElementById('log');l.innerHTML+=m+'<br>';l.scrollTop=l.scrollHeight}

/******** CARD FACTORY ********/
const num=(c,n)=>({t:'n',c,n})
const act=(n,c)=>({t:'a',n,c})
const wild=n=>({t:'w',n})

function buildDeck(){let d=[];COLORS.forEach(c=>{for(let i=0;i<=9;i++)d.push(num(c,i));d.push(act('skip',c),act('reverse',c),act('draw2',c))});d.push(wild('wild'),wild('wild_draw4'));return d.sort(()=>Math.random()-.5)}

/******** SETUP ********/
function start(){drawPile=buildDeck();discard=[];players=[{n:'You',h:[]},{n:'AI 1',h:[]},{n:'AI 2',h:[]}];for(let i=0;i<7;i++)players.forEach(p=>p.h.push(drawPile.pop()));discard.push(drawPile.pop());render();log('Game start')}

/******** RENDER ********/
function render(){document.getElementById('status').innerText=`Turn:${players[current].n}`;let top=discard.at(-1);document.getElementById('table').innerHTML=`<b>Top Card</b><div class='card ${top.c||'wild'}'>${label(top)}</div>`;renderStack();let pDiv=document.getElementById('players');pDiv.innerHTML='';players.forEach((p,i)=>{if(i!==0)pDiv.innerHTML+=`${p.n}: ${p.h.length} cards<br>`});let h=document.getElementById('hand');h.innerHTML='';players[0].h.forEach((c,i)=>{let d=document.createElement('div');d.className='card '+(c.c||'wild');d.innerText=label(c);d.onclick=()=>play(i,d);h.appendChild(d)})}

function renderStack(){let s=document.getElementById('stackPile');s.innerHTML='<b>Discard Stack</b><div class="stack">'+discard.slice(-5).map((c,i)=>`<div class='card ${c.c||'wild'}' style='--i:${i}'>${label(c)}</div>`).join('')+'</div>'}

const label=c=>c.t==='n'?c.n:c.n.toUpperCase().replace(/_/g,' ');

/******** GAME ********/
function canPlay(c,t){return c.t==='w'||c.c===t.c||c.n===t.n}

function play(i,el){if(current!==0)return;let c=players[0].h[i];if(!canPlay(c,discard.at(-1)))return;players[0].h.splice(i,1);el.classList.add('played');sndPlay.volume=sfxVolume;sndPlay.play();discard.push(c);checkUNO(0);endTurn()}

function endTurn(){current=(current+direction+players.length)%players.length;if(current!==0)setTimeout(ai,700);render()}

function ai(){let p=players[current];let t=discard.at(-1);let i=p.h.findIndex(c=>canPlay(c,t));if(i>=0){discard.push(p.h.splice(i,1)[0]);sndPlay.play();checkUNO(current)}else{p.h.push(drawPile.pop());sndDraw.volume=sfxVolume;sndDraw.play();stats.draws++}endTurn()}

/******** UNO ********/
function checkUNO(i){if(players[i].h.length===1){if(i===0&&!unoCalled){players[0].h.push(drawPile.pop(),drawPile.pop());log('UNO missed! +2');unlock('Oops!');}else{stats.unoCalls++;unlock('First UNO!')} }unoCalled=false;localStorage.setItem('unoStats',JSON.stringify(stats))}
function callUNO(){unoCalled=true;sndUNO.volume=sfxVolume;sndUNO.play();log('UNO!')}

/******** UI ********/
function drawCard(){players[0].h.push(drawPile.pop());sndDraw.volume=sfxVolume;sndDraw.play();stats.draws++;render()}
function openSettings(){document.getElementById('settingsModal').style.display='flex'}
function closeSettings(){document.getElementById('settingsModal').style.display='none'}
function resetStats(){stats={wins:0,unoCalls:0,draws:0};achievements={};localStorage.clear();log('Stats reset')}

start();
</script></body>
</html>
