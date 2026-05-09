# HTML 可视化模板参考（动态 Canvas 版）

Claude 生成 HTML 可视化时，使用以下模板结构。将 `__CPN_DATA__` 替换为实际 JSON 数据对象。

## 核心规则

1. **渲染引擎**：Canvas（不用 SVG），绘制顺序：泳道 → 弧 → 变迁形状 → 库所形状 → 所有文字 → **依赖虚线最后画**（带背景色块），永不被节点遮挡
2. **动画**：粒子用线性进度值 `prog: 0→1`，每帧 `prog += dt / duration`，`prog >= 1` 时强制落地。**禁止**用指数逼近（`x += (target-x) * k`）——永远无法精确到达，必然卡死
3. **解锁条件**：`particles.every(pk => pk.done)` 全部落地才清除 `firingId`，**不能**用 `length <= 1`
4. **资源库所**：有初始 token 的库所（如"缴费待完成"）必须在 `places` 中设 `tokens: N`，否则多输入变迁永远无法触发
5. **主题**：提供4套宋式配色，CSS 变量驱动，切换时同步更新 canvas 背景

## 四套宋式主题

| 主题 | 灵感 | 背景 | 特点 |
|------|------|------|------|
| 天青 | 汝窑青瓷 | 暖米白 | 浅色，雅致清透 |
| 墨夜 | 极简黑白 | 纯黑 | 深色，高对比，荧光链色 |
| 石青 | 冷灰调 | 深蓝灰 | 深色，冷峻清晰 |
| 朱砂 | 暖白纸感 | 米白 | 浅色，饱和链色，对比强 |

## 模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>CPN 模型可视化</title>
<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
:root {
  --bg:#f0ece3; --cvs:#e8e3d8; --border:#cfc8bc;
  --text:#2a2418; --sub:#7a7060; --dim:#b0a898;
  --btn:#e4dfd4; --btn-t:#4a4030; --btn-b:#c8c0b0;
  --btn-h:#dad4c8; --act:#5a9080; --act-t:#f0ece3; --sep:#cfc8bc;
}
body { font-family:'PingFang SC','Noto Sans SC','Microsoft YaHei',sans-serif;
  background:var(--bg); color:var(--text); padding:28px;
  min-height:100vh; transition:background .35s,color .35s; }
h1 { font-size:16px; font-weight:600; letter-spacing:.06em; }
.sub { font-size:11px; color:var(--sub); margin-top:4px; letter-spacing:.04em; }
canvas { display:block; border-radius:16px; border:1px solid var(--border);
  background:var(--cvs); transition:background .35s,border-color .35s;
  box-shadow:0 4px 40px rgba(0,0,0,.08); margin-top:16px; }
.row { display:flex; gap:7px; margin-top:13px; align-items:center; flex-wrap:wrap; }
button { padding:5px 13px; border-radius:6px; font-size:11px; cursor:pointer;
  border:1px solid var(--btn-b); background:var(--btn); color:var(--btn-t);
  transition:all .15s; font-family:inherit; }
button:hover { background:var(--btn-h); }
button.active { background:var(--act); color:var(--act-t); border-color:var(--act); }
.sep { width:1px; height:18px; background:var(--sep); margin:0 3px; }
.canvas-wrap { position:relative; display:inline-block; }
#theme-sel { position:absolute; top:10px; right:10px; z-index:10;
  padding:4px 8px; border-radius:6px; font-size:11px; cursor:pointer;
  border:1px solid var(--btn-b); background:var(--btn); color:var(--btn-t);
  font-family:inherit; letter-spacing:.04em; outline:none;
  transition:background .2s,color .2s,border-color .2s; }
.legend { display:flex; gap:16px; margin-top:11px; font-size:11px; color:var(--dim); flex-wrap:wrap; align-items:center; }
.li { display:flex; align-items:center; gap:5px; }
</style>
</head>
<body>
<h1>CPN 模型：<span id="pid"></span></h1>
<div class="sub">着色 Petri 网</div>
<div class="canvas-wrap">
<canvas id="c"></canvas>
<select id="theme-sel" onchange="applyTheme(THEMES[this.selectedIndex])"></select>
</div>
<div class="row">
  <button id="btn-auto" onclick="toggleAuto()">▶ 自动运行</button>
  <button onclick="stepOnce()">单步</button>
  <button onclick="resetSim()">重置</button>
  <div class="sep"></div>
  <span style="font-size:11px;color:var(--dim)">速度</span>
  <button onclick="setSpeed(1400)" id="sp-slow">慢</button>
  <button onclick="setSpeed(800)"  id="sp-mid" class="active">中</button>
  <button onclick="setSpeed(360)"  id="sp-fast">快</button>
</div>
<div class="legend">
  <div class="li"><svg width="14" height="14"><circle cx="7" cy="7" r="6" fill="none" stroke="currentColor" stroke-width="1.5"/></svg>库所</div>
  <div class="li"><svg width="16" height="10"><rect width="16" height="10" rx="2" fill="currentColor" opacity=".5"/></svg>变迁</div>
  <div class="li"><svg width="10" height="10"><circle cx="5" cy="5" r="4" fill="currentColor"/></svg>Token</div>
  <div class="li"><svg width="26" height="3"><line x1="0" y1="1.5" x2="26" y2="1.5" stroke="currentColor" stroke-width="1.5"/></svg>弧</div>
  <div class="li"><svg width="26" height="3"><line x1="0" y1="1.5" x2="26" y2="1.5" stroke="currentColor" stroke-width="1.5" stroke-dasharray="4,3"/></svg>依赖</div>
</div>
<script>
const data = __CPN_DATA__;
document.getElementById('pid').textContent = data.project_id;
```

```js
// ── 四套宋式主题 ──
const THEMES = [
  { id:'tiānqīng',name:'天青',sub:'汝窑',swatch:'#7aaa98',
    css:{'--bg':'#f0ece3','--cvs':'#e8e3d8','--border':'#cfc8bc','--text':'#2a2418','--sub':'#7a7060','--dim':'#b0a898','--btn':'#e4dfd4','--btn-t':'#4a4030','--btn-b':'#c8c0b0','--btn-h':'#dad4c8','--act':'#5a9080','--act-t':'#f0ece3','--sep':'#cfc8bc'},
    cvsBg:'#e8e3d8', dark:false, laneA:.09,
    chains:{"挂号链":{s:"#5a9a88",g:"#78b8a0",f:"#daeee8"},"诊疗链":{s:"#6888a0",g:"#88a8c0",f:"#d8e4f0"},"收费链":{s:"#9a7850",g:"#b89870",f:"#ecddd0"},"药房链":{s:"#8a7898",g:"#a898b8",f:"#e4dce8"}},
    dep:'#c04828' },
  { id:'mòyè',name:'墨夜',sub:'极简',swatch:'#e8e8e8',
    css:{'--bg':'#0a0a0a','--cvs':'#111111','--border':'#2a2a2a','--text':'#f0f0f0','--sub':'#888888','--dim':'#555555','--btn':'#1a1a1a','--btn-t':'#d0d0d0','--btn-b':'#333333','--btn-h':'#222222','--act':'#e8e8e8','--act-t':'#0a0a0a','--sep':'#333333'},
    cvsBg:'#111111', dark:true, laneA:.07,
    chains:{"挂号链":{s:"#60c8a0",g:"#80e8c0",f:"#0a2018"},"诊疗链":{s:"#60a8e0",g:"#80c8ff",f:"#0a1828"},"收费链":{s:"#e0a040",g:"#ffc060",f:"#281a08"},"药房链":{s:"#c060c0",g:"#e080e0",f:"#200820"}},
    dep:'#ff5040' },
  { id:'shíqīng',name:'石青',sub:'冷灰',swatch:'#7090b0',
    css:{'--bg':'#1a1e24','--cvs':'#20262e','--border':'#303840','--text':'#d8e4f0','--sub':'#6878a0','--dim':'#485870','--btn':'#252c38','--btn-t':'#a8c0d8','--btn-b':'#384858','--btn-h':'#2e3848','--act':'#4878b8','--act-t':'#e8f4ff','--sep':'#384858'},
    cvsBg:'#20262e', dark:true, laneA:.08,
    chains:{"挂号链":{s:"#48c8a0",g:"#68e8c0",f:"#0c2820"},"诊疗链":{s:"#4898e0",g:"#68b8ff",f:"#0c1c38"},"收费链":{s:"#e09848",g:"#ffb868",f:"#281c08"},"药房链":{s:"#a868e0",g:"#c888ff",f:"#1c0c30"}},
    dep:'#ff6050' },
  { id:'zhūshā',name:'朱砂',sub:'暖白',swatch:'#c83828',
    css:{'--bg':'#fafaf8','--cvs':'#f4f4f0','--border':'#d8d4cc','--text':'#1a1410','--sub':'#6a5a50','--dim':'#a89888','--btn':'#eeeae4','--btn-t':'#3a2a20','--btn-b':'#c8c0b4','--btn-h':'#e4e0d8','--act':'#c83828','--act-t':'#fff8f4','--sep':'#d8d4cc'},
    cvsBg:'#f4f4f0', dark:false, laneA:.07,
    chains:{"挂号链":{s:"#1a9870",g:"#28c890",f:"#d8f4ec"},"诊疗链":{s:"#1870c0",g:"#2890e8",f:"#d8ecf8"},"收费链":{s:"#c87820",g:"#f09830",f:"#f8ead8"},"药房链":{s:"#9828b8",g:"#c038e0",f:"#f0d8f8"}},
    dep:'#c83828' },
];

let T = THEMES[0];
const canvas = document.getElementById('c');

function applyTheme(theme) {
  T = theme;
  Object.entries(theme.css).forEach(([k,v]) => document.documentElement.style.setProperty(k,v));
  canvas.style.background = theme.cvsBg;
  const sel = document.getElementById('theme-sel');
  if (sel) sel.selectedIndex = THEMES.indexOf(theme);
}

const sel = document.getElementById('theme-sel');
THEMES.forEach((t,i) => {
  const opt = document.createElement('option');
  opt.value = i;
  opt.textContent = `${t.name} · ${t.sub}`;
  sel.appendChild(opt);
});
applyTheme(THEMES[0]);
```

```js
// ── 自动布局（BFS 拓扑排序 x；subproject 索引 y）──
const allNodes = [...data.places, ...data.transitions];
const spList = [...new Set(allNodes.map(n => n.subproject).filter(Boolean))];
const adj = {};
data.arcs.forEach(a => { if (!adj[a.from]) adj[a.from]=[]; adj[a.from].push(a.to); });
const hasIn = new Set(data.arcs.map(a => a.to));
const depths = {};
const bfsQ = allNodes.map(n=>n.id).filter(id => !hasIn.has(id));
bfsQ.forEach(id => depths[id]=0);
let qi=0;
while (qi<bfsQ.length) {
  const id=bfsQ[qi++];
  (adj[id]||[]).forEach(nx => { if (depths[nx]===undefined) { depths[nx]=depths[id]+1; bfsQ.push(nx); }});
}
allNodes.forEach(n => { if (depths[n.id]===undefined) depths[n.id]=0; });

const COL=240, ROW=170, PX=100, PY=180;
const spDepCnt={}, positions={};
allNodes.forEach(n => {
  const sp=n.subproject||'_', d=depths[n.id]||0, key=`${sp}:${d}`;
  if (!spDepCnt[key]) spDepCnt[key]=0;
  const slot=spDepCnt[key]++;
  positions[n.id]={ x:PX+d*COL, y:PY+spList.indexOf(sp)*ROW+slot*55 };
});

// 资源库所位置修正：放在其主要消费变迁正上方
data.places.forEach(pl => {
  if (!pl.initial_marking || !pl.initial_marking.length) return;
  const firstConsumer = data.arcs.find(a => a.from === pl.id && a.to.startsWith('T'));
  if (firstConsumer && positions[firstConsumer.to]) {
    positions[pl.id] = {
      x: positions[firstConsumer.to].x,
      y: positions[firstConsumer.to].y - 100
    };
  }
});

const maxX=Math.max(...Object.values(positions).map(p=>p.x))+130;
const maxY=Math.max(...Object.values(positions).map(p=>p.y))+110;
// 自适应窗口，不出现滚动条
const availW = Math.min(window.innerWidth - 56, 960);
const availH = Math.min(window.innerHeight - 220, 600);
const scale = Math.min(availW / maxX, availH / maxY, 1);
const cW = Math.round(maxX * scale);
const cH = Math.round(maxY * scale);
const dpr=window.devicePixelRatio||1;
canvas.width=cW*dpr; canvas.height=cH*dpr;
canvas.style.width=cW+'px'; canvas.style.height=cH+'px';
const ctx=canvas.getContext('2d');
ctx.scale(dpr * scale, dpr * scale);

// ── 变迁输入输出表（从 arcs 推导）──
const tIn={}, tOut={};
data.transitions.forEach(t => { tIn[t.id]=[]; tOut[t.id]=[]; });
data.arcs.forEach(a => {
  if (a.from.startsWith('T') && tOut[a.from]) tOut[a.from].push(a.to);
  if (a.to.startsWith('T')   && tIn[a.to])    tIn[a.to].push(a.from);
});

// ── 模拟状态 ──
let tokenMap={}, particles=[], firingId=null, autoMode=false, autoTimer=null, stepMs=800;

function resetSim() {
  tokenMap={};
  data.places.forEach(p => { tokenMap[p.id]=(p.initial_marking&&p.initial_marking.length)?p.initial_marking.length:0; });
  particles=[]; firingId=null;
}
resetSim();

function getEnabled() {
  return data.transitions.filter(t => (tIn[t.id]||[]).length>0 && (tIn[t.id]||[]).every(pid=>(tokenMap[pid]||0)>0));
}

function fire(t) {
  if (firingId) return;
  (tIn[t.id]||[]).forEach(pid => tokenMap[pid]--);
  firingId=t.id;
  (tIn[t.id]||[]).forEach(fromId => {
    (tOut[t.id]||[]).forEach(toId => {
      const tp=positions[t.id];
      particles.push({ path:[positions[fromId],tp,positions[toId]], prog:0, chain:t.chain, toId, done:false });
    });
  });
}

function stepOnce() { if (!firingId) { const en=getEnabled(); if(en.length) fire(en[0]); } }

function toggleAuto() {
  autoMode=!autoMode;
  const btn=document.getElementById('btn-auto');
  if (autoMode) { btn.textContent='⏸ 暂停'; btn.classList.add('active'); scheduleNext(); }
  else { btn.textContent='▶ 自动运行'; btn.classList.remove('active'); clearTimeout(autoTimer); }
}

function scheduleNext() {
  if (!autoMode) return;
  autoTimer=setTimeout(() => {
    if (!firingId) {
      const en=getEnabled();
      if (!en.length) { setTimeout(()=>{ resetSim(); if(autoMode) scheduleNext(); },1400); return; }
      fire(en[Math.floor(Math.random()*en.length)]);
    }
    scheduleNext();
  }, 60);
}

function setSpeed(ms) {
  stepMs=ms;
  ['sp-slow','sp-mid','sp-fast'].forEach(id=>document.getElementById(id).classList.remove('active'));
  document.getElementById(ms===1400?'sp-slow':ms===800?'sp-mid':'sp-fast').classList.add('active');
}
```

```js
// ── 绘制工具 ──
const PR=26, TW=52, TH=30;

function roundRect(x,y,w,h,r) {
  ctx.beginPath();
  ctx.moveTo(x+r,y); ctx.lineTo(x+w-r,y); ctx.arcTo(x+w,y,x+w,y+r,r);
  ctx.lineTo(x+w,y+h-r); ctx.arcTo(x+w,y+h,x+w-r,y+h,r);
  ctx.lineTo(x+r,y+h); ctx.arcTo(x,y+h,x,y+h-r,r);
  ctx.lineTo(x,y+r); ctx.arcTo(x,y,x+r,y,r); ctx.closePath();
}

function edgePt(id, toward, isStart) {
  const f=positions[id], t=positions[toward];
  const dx=t.x-f.x, dy=t.y-f.y, len=Math.sqrt(dx*dx+dy*dy)||1;
  const ux=dx/len, uy=dy/len, sign=isStart?1:-1;
  if (id.startsWith('P')) return {x:f.x+ux*sign*PR, y:f.y+uy*sign*PR};
  const sx=Math.abs(ux)<1e-6?1e9:(TW/2)/Math.abs(ux);
  const sy=Math.abs(uy)<1e-6?1e9:(TH/2)/Math.abs(uy);
  const s=Math.min(sx,sy);
  return {x:f.x+ux*sign*s, y:f.y+uy*sign*s};
}

function arrow(x1,y1,x2,y2,color,dashed,lw) {
  ctx.save(); ctx.strokeStyle=color; ctx.lineWidth=lw||1.3;
  if (dashed) ctx.setLineDash([5,4]);
  ctx.beginPath(); ctx.moveTo(x1,y1); ctx.lineTo(x2,y2); ctx.stroke(); ctx.setLineDash([]);
  const dx=x2-x1,dy=y2-y1,len=Math.sqrt(dx*dx+dy*dy)||1,ux=dx/len,uy=dy/len;
  const ax=x2-ux*8,ay=y2-uy*8;
  ctx.fillStyle=color; ctx.beginPath();
  ctx.moveTo(x2,y2); ctx.lineTo(ax-uy*4,ay+ux*4); ctx.lineTo(ax+uy*4,ay-ux*4);
  ctx.closePath(); ctx.fill(); ctx.restore();
}

function pathLerp(path, prog) {
  let segs=[], total=0;
  for (let i=1;i<path.length;i++) {
    const d=Math.sqrt((path[i].x-path[i-1].x)**2+(path[i].y-path[i-1].y)**2);
    segs.push(d); total+=d;
  }
  if (!total) return path[0];
  let dist=prog*total;
  for (let i=0;i<segs.length;i++) {
    if (dist<=segs[i]) { const f=segs[i]>0?dist/segs[i]:0; return {x:path[i].x+(path[i+1].x-path[i].x)*f,y:path[i].y+(path[i+1].y-path[i].y)*f}; }
    dist-=segs[i];
  }
  return path[path.length-1];
}

// ── 主绘制循环 ──
let lastT=0, frame=0;

function draw(now) {
  const dt=Math.min((now-lastT)/1000,.05); lastT=now; frame++;
  ctx.clearRect(0,0,canvas.width,canvas.height);
  const ch=T.chains;

  // 1. 泳道背景
  spList.forEach((sp,i) => {
    const spNodes=allNodes.filter(n=>n.subproject===sp); if(!spNodes.length) return;
    const xs=spNodes.map(n=>positions[n.id].x), ys=spNodes.map(n=>positions[n.id].y);
    const lx=Math.min(...xs)-40, ly=Math.min(...ys)-30, lw=Math.max(...xs)-lx+80, lh=Math.max(...ys)-ly+60;
    const c=Object.values(ch)[i%Object.keys(ch).length];
    ctx.save(); ctx.globalAlpha=T.laneA; ctx.fillStyle=c.s; roundRect(lx,ly,lw,lh,10); ctx.fill();
    ctx.globalAlpha=1; ctx.strokeStyle=c.s+(T.dark?'55':'44'); ctx.lineWidth=1; ctx.setLineDash([4,4]);
    roundRect(lx,ly,lw,lh,10); ctx.stroke(); ctx.setLineDash([]);
    ctx.fillStyle=c.s+(T.dark?'99':'bb'); ctx.font='bold 10px PingFang SC,sans-serif';
    ctx.fillText(sp,lx+10,ly+16); ctx.restore();
  });

  // 2. 普通弧（节点下层）
  data.arcs.forEach(a => {
    const node=data.places.find(p=>p.id===a.from)||data.transitions.find(t=>t.id===a.from);
    const c=ch[node?.chain]||Object.values(ch)[0];
    const s=edgePt(a.from,a.to,true), e=edgePt(a.to,a.from,false);
    arrow(s.x,s.y,e.x,e.y,c.s+(T.dark?'77':'88'),false,1.2);
  });

  // 3. 变迁形状
  data.transitions.forEach(t => {
    const p=positions[t.id]; if(!p) return;
    const c=ch[t.chain]||Object.values(ch)[0];
    const isFiring=firingId===t.id, isEnabled=!firingId&&getEnabled().some(e=>e.id===t.id);
    ctx.save();
    if (isFiring) { ctx.shadowColor=c.g; ctx.shadowBlur=22+Math.sin(frame*.15)*6; }
    else if (isEnabled) { ctx.shadowColor=c.g; ctx.shadowBlur=10; }
    ctx.fillStyle=isFiring?c.g+'ee':(isEnabled?c.s+'cc':(T.dark?c.f:c.f+'cc'));
    roundRect(p.x-TW/2,p.y-TH/2,TW,TH,4); ctx.fill();
    ctx.strokeStyle=isFiring?c.g:(isEnabled?c.g+'cc':c.s+(T.dark?'44':'55'));
    ctx.lineWidth=isEnabled||isFiring?1.5:1;
    roundRect(p.x-TW/2,p.y-TH/2,TW,TH,4); ctx.stroke(); ctx.restore();
  });

  // 5. 库所形状
  data.places.forEach(pl => {
    const p=positions[pl.id]; if(!p) return;
    const c=ch[pl.chain]||Object.values(ch)[0];
    const has=(tokenMap[pl.id]||0)>0;
    ctx.save(); ctx.beginPath(); ctx.arc(p.x,p.y,PR,0,Math.PI*2);
    ctx.fillStyle=has?c.f+(T.dark?'dd':'cc'):(T.dark?'#0d0c0a':'#f8f5f0'); ctx.fill();
    ctx.strokeStyle=has?c.s:c.s+(T.dark?'44':'55'); ctx.lineWidth=has?2:1.2;
    if (has) { ctx.shadowColor=c.g; ctx.shadowBlur=12; }
    ctx.stroke(); ctx.restore();
    if (has) {
      const cnt=tokenMap[pl.id];
      for (let i=0;i<Math.min(cnt,5);i++) {
        const angle=cnt===1?-Math.PI/2:(i/cnt)*Math.PI*2-Math.PI/2, r=cnt===1?0:9;
        ctx.save(); ctx.beginPath(); ctx.arc(p.x+Math.cos(angle)*r,p.y+Math.sin(angle)*r,4.5,0,Math.PI*2);
        ctx.fillStyle=c.g; ctx.shadowColor=c.g; ctx.shadowBlur=14; ctx.fill(); ctx.restore();
      }
    }
  });

  // 6. 所有文字最后画（永不被遮挡）
  data.transitions.forEach(t => {
    const p=positions[t.id]; if(!p) return;
    const c=ch[t.chain]||Object.values(ch)[0];
    const isFiring=firingId===t.id, isEnabled=!firingId&&getEnabled().some(e=>e.id===t.id);
    ctx.save(); ctx.textAlign='center'; ctx.textBaseline='middle'; ctx.font='10px PingFang SC,sans-serif';
    ctx.fillStyle=isFiring?(T.dark?'#1a1000':'#fff8e8'):(isEnabled?(T.dark?'#e8d090':c.s):(T.dark?c.s+'cc':'#6a5a48'));
    ctx.fillText(t.name.length>6?t.name.slice(0,6)+'…':t.name,p.x,p.y); ctx.restore();
  });

  data.places.forEach(pl => {
    const p=positions[pl.id]; if(!p) return;
    const c=ch[pl.chain]||Object.values(ch)[0];
    const has=(tokenMap[pl.id]||0)>0;
    ctx.save(); ctx.textAlign='center'; ctx.textBaseline='middle'; ctx.font='10px PingFang SC,sans-serif';
    ctx.fillStyle=has?(T.dark?'#d4b870':c.s):(T.dark?c.s+'88':'#9a8e80');
    const lines=pl.name.split('_'), lh=13;
    lines.forEach((ln,i)=>ctx.fillText(ln,p.x,p.y-(lines.length-1)*lh/2+i*lh));
    ctx.restore();
  });

  // 6. 依赖关系虚线（最后画，永远在最上层）
  (data.dependency_rules||[]).forEach(dep => {
    const fromP=data.places.find(p=>dep.predecessor.includes(p.name.replace('_',''))||dep.predecessor.includes(p.id));
    const toTr=data.transitions.find(t=>dep.successor.includes(t.name)||dep.successor.includes(t.id));
    if (!fromP||!toTr||!positions[fromP.id]||!positions[toTr.id]) return;
    const fp=positions[fromP.id], tp=positions[toTr.id];
    const dx=tp.x-fp.x, dy=tp.y-fp.y, len=Math.sqrt(dx*dx+dy*dy)||1;
    const ux=dx/len, uy=dy/len, nx=-uy, ny=ux;
    const off=dep.offset||-50;
    const sx=fp.x+ux*PR, sy=fp.y+uy*PR;
    const tsx=Math.abs(ux)<1e-6?1e9:(TW/2)/Math.abs(ux);
    const tsy=Math.abs(uy)<1e-6?1e9:(TH/2)/Math.abs(uy);
    const ts=Math.min(tsx,tsy);
    const ex=tp.x-ux*ts, ey=tp.y-uy*ts;
    const cpx=(sx+ex)/2+nx*off, cpy=(sy+ey)/2+ny*off;
    ctx.save();
    ctx.strokeStyle=T.dep+'cc'; ctx.lineWidth=1.5; ctx.setLineDash([5,4]);
    ctx.beginPath(); ctx.moveTo(sx,sy); ctx.quadraticCurveTo(cpx,cpy,ex,ey); ctx.stroke();
    ctx.setLineDash([]);
    const tax2=ex-cpx, tay2=ey-cpy, tl=Math.sqrt(tax2*tax2+tay2*tay2)||1;
    const tax=tax2/tl, tay=tay2/tl;
    ctx.fillStyle=T.dep+'cc'; ctx.beginPath();
    ctx.moveTo(ex,ey); ctx.lineTo(ex-tax*8-tay*4,ey-tay*8+tax*4); ctx.lineTo(ex-tax*8+tay*4,ey-tay*8-tax*4);
    ctx.closePath(); ctx.fill();
    const lx=0.25*sx+0.5*cpx+0.25*ex, ly=0.25*sy+0.5*cpy+0.25*ey;
    ctx.font='bold 9px monospace';
    const tw=ctx.measureText(dep.id).width;
    ctx.fillStyle=T.dark?'rgba(0,0,0,.75)':'rgba(255,255,255,.85)';
    ctx.fillRect(lx-tw/2-4,ly-8,tw+8,14);
    ctx.fillStyle=T.dep; ctx.textAlign='center'; ctx.textBaseline='middle';
    ctx.fillText(dep.id,lx,ly); ctx.restore();
  });

  // 7. 粒子（线性进度，不会卡死）
  const dur=stepMs*0.001*0.65;
  particles.forEach(pk => {
    if (pk.done) return;
    pk.prog+=dt/dur;
    if (pk.prog>=1) { pk.prog=1; pk.done=true; tokenMap[pk.toId]=(tokenMap[pk.toId]||0)+1; }
    const pt=pathLerp(pk.path,pk.prog), pt0=pathLerp(pk.path,Math.max(0,pk.prog-.1));
    const c=ch[pk.chain]||Object.values(ch)[0];
    ctx.save();
    ctx.strokeStyle=c.g+'55'; ctx.lineWidth=3;
    ctx.beginPath(); ctx.moveTo(pt0.x,pt0.y); ctx.lineTo(pt.x,pt.y); ctx.stroke();
    ctx.beginPath(); ctx.arc(pt.x,pt.y,6,0,Math.PI*2);
    ctx.fillStyle=c.g; ctx.shadowColor=c.g; ctx.shadowBlur=20; ctx.fill(); ctx.restore();
  });

  // 全部落地才解锁（必须 every，不能用 length <= 1）
  if (particles.length>0 && particles.every(pk=>pk.done)) { firingId=null; particles=[]; }

  requestAnimationFrame(draw);
}
requestAnimationFrame(draw);
</script>
</body>
</html>
```

## 使用说明

1. 将 `__CPN_DATA__` 替换为 json-schema.md 格式的完整 JSON 对象
2. `places` 中有初始 token 的库所必须在 `initial_marking` 中列出（如 `["1\`收费链"]`），否则多输入变迁永远无法触发
3. 浏览器打开即可看到动态 Petri 网，支持自动运行、单步、重置、速度调节
4. 右上角4套宋式主题可切换：天青（汝窑）/ 墨夜（极简）/ 石青（冷灰）/ 朱砂（暖白）
5. 圆形=库所，矩形=变迁，实线=弧，红色虚线=依赖关系；可触发变迁发光提示
