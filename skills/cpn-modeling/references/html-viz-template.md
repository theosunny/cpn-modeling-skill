# HTML 可视化模板参考（动态 Canvas 版）

Claude 生成 HTML 可视化时，使用以下模板结构。将 `__CPN_DATA__` 替换为实际的 JSON 数据对象。

## 关键规则

- 使用 Canvas 渲染，**绘制顺序**：泳道背景 → 弧 → 变迁 → 库所 → 文字（文字最后画，永不被遮挡）
- 自动布局：BFS 拓扑排序决定 x 坐标，subproject 索引决定 y 坐标
- Token 动画：飞行粒子经过变迁节点再落入目标库所，带发光拖尾
- P6 等"资源库所"若有初始 token，务必在 `places` 数组中设置 `tokens:N`
- `firingTrans` 清除条件：`animTokens.filter(t=>!t.done).length === 0`（必须全部落地）

## 模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>CPN 模型可视化</title>
<style>
* { box-sizing: border-box; }
body { font-family: 'PingFang SC','Microsoft YaHei',sans-serif; background:#0f172a; margin:0; padding:20px; color:#e2e8f0; }
h1 { font-size:18px; margin-bottom:2px; color:#f1f5f9; }
.subtitle { color:#64748b; font-size:13px; margin-bottom:16px; }
canvas { border-radius:12px; display:block; background:#1e293b; border:1px solid #334155; }
.controls { display:flex; gap:10px; margin-top:12px; align-items:center; flex-wrap:wrap; }
button { padding:6px 16px; border-radius:6px; border:1px solid #475569; background:#1e293b; color:#cbd5e1; cursor:pointer; font-size:13px; transition:all .15s; }
button:hover { background:#334155; color:#f1f5f9; }
button.active { background:#6366f1; border-color:#6366f1; color:white; }
.legend { display:flex; gap:14px; margin-top:10px; font-size:12px; color:#94a3b8; flex-wrap:wrap; align-items:center; }
.legend-item { display:flex; align-items:center; gap:5px; }
</style>
</head>
<body>
<h1>CPN 模型：<span id="pid"></span></h1>
<p class="subtitle">着色 Petri 网</p>
<canvas id="c"></canvas>
<div class="controls">
  <button id="btn-auto" onclick="toggleAuto()">▶ 自动运行</button>
  <button onclick="stepOnce()">单步执行</button>
  <button onclick="resetSim()">重置</button>
  <span style="color:#64748b;font-size:12px">速度：</span>
  <button onclick="setSpeed(1200)" id="sp-slow">慢</button>
  <button onclick="setSpeed(700)"  id="sp-mid" class="active">中</button>
  <button onclick="setSpeed(350)"  id="sp-fast">快</button>
</div>
<div class="legend">
  <div class="legend-item"><div style="width:12px;height:12px;border-radius:50%;border:2px solid #94a3b8;background:transparent"></div>库所</div>
  <div class="legend-item"><div style="width:14px;height:9px;border-radius:2px;background:#94a3b8"></div>变迁</div>
  <div class="legend-item"><div style="width:10px;height:10px;border-radius:50%;background:#fbbf24"></div>Token</div>
  <div class="legend-item"><div style="width:28px;height:2px;background:#94a3b8"></div>弧</div>
  <div class="legend-item"><div style="width:28px;height:0;border-top:2px dashed #ef4444"></div>依赖关系</div>
</div>
<script>
const data = __CPN_DATA__;
document.getElementById('pid').textContent = data.project_id;
```

```js
// ── 颜色配置 ──
const PALETTE = ['#6366f1','#0891b2','#d97706','#059669','#dc2626','#7c3aed','#0d9488','#b45309'];
const GLOW    = ['#818cf8','#38bdf8','#fbbf24','#34d399','#f87171','#a78bfa','#2dd4bf','#fcd34d'];
const allNodes = [...data.places, ...data.transitions];
const spList   = [...new Set(allNodes.map(n => n.subproject).filter(Boolean))];
const chList   = [...new Set(allNodes.map(n => n.chain).filter(Boolean))];
const chainColor = {}, chainGlow = {};
chList.forEach((ch, i) => { chainColor[ch] = PALETTE[i % PALETTE.length]; chainGlow[ch] = GLOW[i % GLOW.length]; });

// ── 自动布局（BFS 拓扑排序 → x；subproject 索引 → y）──
const adj = {};
data.arcs.forEach(a => { if (!adj[a.from]) adj[a.from] = []; adj[a.from].push(a.to); });
const hasIncoming = new Set(data.arcs.map(a => a.to));
const allIds = allNodes.map(n => n.id);
const depths = {};
const bfsQ = allIds.filter(id => !hasIncoming.has(id));
bfsQ.forEach(id => depths[id] = 0);
let qi = 0;
while (qi < bfsQ.length) {
  const id = bfsQ[qi++];
  (adj[id] || []).forEach(next => { if (depths[next] === undefined) { depths[next] = depths[id] + 1; bfsQ.push(next); } });
}
allIds.forEach(id => { if (depths[id] === undefined) depths[id] = 0; });

// 同 subproject 内按 depth 分列，不同 subproject 按行排
const COL_W = 150, ROW_H = 130, PAD_X = 80, PAD_Y = 80;
const spRowY = {};
spList.forEach((sp, i) => { spRowY[sp] = PAD_Y + i * ROW_H; });
// 每个 subproject 内各 depth 的节点按顺序排列（避免重叠）
const spDepthCount = {};
const positions = {};
allNodes.forEach(n => {
  const sp = n.subproject || '_';
  const d  = depths[n.id] || 0;
  const key = `${sp}:${d}`;
  if (!spDepthCount[key]) spDepthCount[key] = 0;
  const slot = spDepthCount[key]++;
  positions[n.id] = { x: PAD_X + d * COL_W, y: (spRowY[n.subproject] || PAD_Y) + slot * 50 };
});

// Canvas 尺寸自适应
const maxX = Math.max(...Object.values(positions).map(p => p.x)) + 100;
const maxY = Math.max(...Object.values(positions).map(p => p.y)) + 80;
const canvas = document.getElementById('c');
canvas.width = Math.max(maxX, 600); canvas.height = Math.max(maxY, 400);
const ctx = canvas.getContext('2d');

// ── 构建变迁输入输出表（从 arcs 推导）──
const transInputs = {}, transOutputs = {};
data.transitions.forEach(t => { transInputs[t.id] = []; transOutputs[t.id] = []; });
data.arcs.forEach(a => {
  if (a.from.startsWith('T') && transOutputs[a.from]) transOutputs[a.from].push(a.to);
  if (a.to.startsWith('T')   && transInputs[a.to])    transInputs[a.to].push(a.from);
});

// ── 模拟状态 ──
let tokenMap = {}, animTokens = [], firingTrans = null, autoMode = false, autoTimer = null, stepInterval = 700, tick = 0;

function resetSim() {
  tokenMap = {};
  data.places.forEach(p => { tokenMap[p.id] = (p.initial_marking && p.initial_marking.length) ? p.initial_marking.length : 0; });
  animTokens = []; firingTrans = null;
}
resetSim();

function getEnabled() {
  return data.transitions.filter(t => (transInputs[t.id] || []).length > 0 && (transInputs[t.id] || []).every(pid => tokenMap[pid] > 0));
}

function fire(t) {
  if (firingTrans) return;
  (transInputs[t.id] || []).forEach(pid => tokenMap[pid]--);
  firingTrans = t.id;
  const tp = positions[t.id];
  (transInputs[t.id] || []).forEach(fromId => {
    (transOutputs[t.id] || []).forEach(toId => {
      const fp = positions[fromId], dp = positions[toId];
      animTokens.push({ x: fp.x, y: fp.y, tx: tp.x, ty: tp.y, tx2: dp.x, ty2: dp.y, phase: 0, color: chainGlow[t.chain] || '#fbbf24', toId, done: false });
    });
  });
}

function stepOnce() {
  if (firingTrans) return;
  const en = getEnabled();
  if (!en.length) return;
  fire(en[0]);
}

function toggleAuto() {
  autoMode = !autoMode;
  const btn = document.getElementById('btn-auto');
  if (autoMode) { btn.textContent = '⏸ 暂停'; btn.classList.add('active'); scheduleNext(); }
  else { btn.textContent = '▶ 自动运行'; btn.classList.remove('active'); clearTimeout(autoTimer); }
}

function scheduleNext() {
  if (!autoMode) return;
  autoTimer = setTimeout(() => {
    if (!firingTrans) {
      const en = getEnabled();
      if (!en.length) {
        if (autoMode) { setTimeout(() => { resetSim(); scheduleNext(); }, 1600); return; }
      } else { fire(en[0]); }
    }
    scheduleNext();
  }, stepInterval);
}

function setSpeed(ms) {
  stepInterval = ms;
  ['sp-slow','sp-mid','sp-fast'].forEach(id => document.getElementById(id).classList.remove('active'));
  document.getElementById(ms===1200?'sp-slow':ms===700?'sp-mid':'sp-fast').classList.add('active');
}
```

```js
// ── 绘制工具 ──
const PR = 26, TW = 52, TH = 32;

function roundRect(x, y, w, h, r) {
  ctx.beginPath();
  ctx.moveTo(x+r, y); ctx.lineTo(x+w-r, y); ctx.arcTo(x+w,y,x+w,y+r,r);
  ctx.lineTo(x+w, y+h-r); ctx.arcTo(x+w,y+h,x+w-r,y+h,r);
  ctx.lineTo(x+r, y+h); ctx.arcTo(x,y+h,x,y+h-r,r);
  ctx.lineTo(x, y+r); ctx.arcTo(x,y,x+r,y,r); ctx.closePath();
}

function edgePts(fromId, toId) {
  const f = positions[fromId], t = positions[toId];
  const dx = t.x-f.x, dy = t.y-f.y, len = Math.sqrt(dx*dx+dy*dy)||1;
  const ux = dx/len, uy = dy/len;
  let x1=f.x, y1=f.y, x2=t.x, y2=t.y;
  if (fromId.startsWith('P')) { x1=f.x+ux*PR; y1=f.y+uy*PR; }
  else { const s=Math.min(Math.abs((TW/2)/(ux||1e-9)), Math.abs((TH/2)/(uy||1e-9))); x1=f.x+ux*s; y1=f.y+uy*s; }
  if (toId.startsWith('P'))   { x2=t.x-ux*PR; y2=t.y-uy*PR; }
  else { const s=Math.min(Math.abs((TW/2)/(ux||1e-9)), Math.abs((TH/2)/(uy||1e-9))); x2=t.x-ux*s; y2=t.y-uy*s; }
  return {x1,y1,x2,y2};
}

function drawArrow(x1,y1,x2,y2,color,dashed,lw) {
  ctx.save(); ctx.strokeStyle=color; ctx.lineWidth=lw||1.5;
  if (dashed) ctx.setLineDash([6,4]);
  ctx.beginPath(); ctx.moveTo(x1,y1); ctx.lineTo(x2,y2); ctx.stroke(); ctx.setLineDash([]);
  const dx=x2-x1,dy=y2-y1,len=Math.sqrt(dx*dx+dy*dy)||1,ux=dx/len,uy=dy/len;
  const ax=x2-ux*9,ay=y2-uy*9;
  ctx.fillStyle=color; ctx.beginPath(); ctx.moveTo(x2,y2); ctx.lineTo(ax-uy*5,ay+ux*5); ctx.lineTo(ax+uy*5,ay-ux*5); ctx.closePath(); ctx.fill();
  ctx.restore();
}

// ── 主绘制循环 ──
function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  tick++;

  // 1. 泳道背景
  spList.forEach((sp, i) => {
    const color = PALETTE[i % PALETTE.length];
    const spNodes = allNodes.filter(n => n.subproject === sp);
    if (!spNodes.length) return;
    const xs = spNodes.map(n => positions[n.id].x), ys = spNodes.map(n => positions[n.id].y);
    const lx=Math.min(...xs)-40, ly=Math.min(...ys)-30, lw=Math.max(...xs)-lx+80, lh=Math.max(...ys)-ly+60;
    ctx.save(); ctx.globalAlpha=0.07; ctx.fillStyle=color; roundRect(lx,ly,lw,lh,10); ctx.fill();
    ctx.globalAlpha=1; ctx.strokeStyle=color; ctx.lineWidth=1.2; ctx.setLineDash([5,4]); roundRect(lx,ly,lw,lh,10); ctx.stroke(); ctx.setLineDash([]);
    ctx.fillStyle=color; ctx.font='bold 11px PingFang SC,sans-serif'; ctx.fillText(sp, lx+10, ly+18);
    ctx.restore();
  });

  // 2. 普通弧（节点下层）
  data.arcs.forEach(a => {
    const {x1,y1,x2,y2} = edgePts(a.from, a.to);
    const node = data.places.find(p=>p.id===a.from) || data.transitions.find(t=>t.id===a.from);
    drawArrow(x1,y1,x2,y2,(chainColor[node?.chain]||'#475569')+'99',false,1.5);
  });

  // 3. 依赖关系虚线（dependency_rules）
  (data.dependency_rules || []).forEach(dep => {
    const fromP = data.places.find(p => dep.predecessor.includes(p.name.split('_')[0]) || dep.predecessor.includes(p.id));
    const toT   = data.transitions.find(t => dep.successor.includes(t.name) || dep.successor.includes(t.id));
    if (!fromP || !toT || !positions[fromP.id] || !positions[toT.id]) return;
    const {x1,y1,x2,y2} = edgePts(fromP.id, toT.id);
    drawArrow(x1,y1,x2,y2,'#ef4444cc',true,1.5);
    ctx.save(); ctx.fillStyle='#ef4444'; ctx.font='bold 9px monospace';
    ctx.fillText(dep.id, (x1+x2)/2+4, (y1+y2)/2-4); ctx.restore();
  });

  // 4. 变迁（矩形）
  data.transitions.forEach(t => {
    const p = positions[t.id]; if (!p) return;
    const color = chainColor[t.chain]||'#475569', glow = chainGlow[t.chain]||'#fbbf24';
    const isFiring = firingTrans===t.id, isEnabled = !firingTrans && getEnabled().some(e=>e.id===t.id);
    ctx.save();
    if (isFiring) { ctx.shadowColor=glow; ctx.shadowBlur=18+Math.sin(tick*0.2)*6; }
    ctx.fillStyle = isFiring ? glow : (isEnabled ? color : color+'88');
    roundRect(p.x-TW/2, p.y-TH/2, TW, TH, 5); ctx.fill();
    if (isEnabled && !isFiring) { ctx.strokeStyle=glow; ctx.lineWidth=1.5; ctx.shadowColor=glow; ctx.shadowBlur=8; roundRect(p.x-TW/2,p.y-TH/2,TW,TH,5); ctx.stroke(); }
    ctx.restore();
    ctx.save(); ctx.fillStyle='#f1f5f9'; ctx.font='10px PingFang SC,sans-serif'; ctx.textAlign='center'; ctx.textBaseline='middle';
    ctx.fillText(t.name.length>6?t.name.slice(0,6)+'…':t.name, p.x, p.y); ctx.restore();
  });

  // 5. 库所（圆形）
  data.places.forEach(pl => {
    const p = positions[pl.id]; if (!p) return;
    const color = chainColor[pl.chain]||'#475569', glow = chainGlow[pl.chain]||'#94a3b8';
    const hasToken = tokenMap[pl.id] > 0;
    ctx.save();
    ctx.beginPath(); ctx.arc(p.x,p.y,PR,0,Math.PI*2);
    ctx.fillStyle = hasToken ? color+'22' : '#1e293b'; ctx.fill();
    ctx.strokeStyle = hasToken ? glow : color+'66'; ctx.lineWidth = hasToken ? 2.5 : 1.5;
    if (hasToken) { ctx.shadowColor=glow; ctx.shadowBlur=10; }
    ctx.stroke(); ctx.restore();
    if (hasToken) {
      const cnt = tokenMap[pl.id];
      for (let i=0; i<Math.min(cnt,5); i++) {
        const angle = cnt===1 ? -Math.PI/2 : (i/cnt)*Math.PI*2-Math.PI/2;
        const r = cnt===1 ? 0 : 10;
        ctx.save(); ctx.beginPath(); ctx.arc(p.x+Math.cos(angle)*r, p.y+Math.sin(angle)*r, 5, 0, Math.PI*2);
        ctx.fillStyle=glow; ctx.shadowColor=glow; ctx.shadowBlur=12; ctx.fill(); ctx.restore();
      }
    }
    // 文字最后画（最上层，永不被遮挡）
    ctx.save(); ctx.fillStyle=hasToken?'#f1f5f9':'#94a3b8'; ctx.font='10px PingFang SC,sans-serif'; ctx.textAlign='center'; ctx.textBaseline='middle';
    const lines = pl.name.split('_'); const lh = 13;
    lines.forEach((line, i) => ctx.fillText(line, p.x, p.y - (lines.length-1)*lh/2 + i*lh));
    ctx.restore();
  });

  // 6. 飞行 token 粒子
  animTokens = animTokens.filter(tk => !tk.done);
  animTokens.forEach(tk => {
    const spd = 0.1;
    if (tk.phase===0) {
      tk.x+=(tk.tx-tk.x)*spd*2; tk.y+=(tk.ty-tk.y)*spd*2;
      if (Math.abs(tk.x-tk.tx)<3 && Math.abs(tk.y-tk.ty)<3) tk.phase=1;
    } else {
      tk.x+=(tk.tx2-tk.x)*spd*2; tk.y+=(tk.ty2-tk.y)*spd*2;
      if (Math.abs(tk.x-tk.tx2)<3 && Math.abs(tk.y-tk.ty2)<3) {
        tokenMap[tk.toId]++; tk.done=true;
        if (animTokens.filter(t=>!t.done).length === 0) firingTrans=null;
      }
    }
    ctx.save(); ctx.beginPath(); ctx.arc(tk.x,tk.y,6,0,Math.PI*2);
    ctx.fillStyle=tk.color; ctx.shadowColor=tk.color; ctx.shadowBlur=16; ctx.fill();
    ctx.globalAlpha=0.3; ctx.beginPath(); ctx.arc(tk.x-(tk.tx2-tk.x)*0.15, tk.y-(tk.ty2-tk.y)*0.15, 4, 0, Math.PI*2); ctx.fill();
    ctx.restore();
  });

  requestAnimationFrame(draw);
}
draw();
</script>
</body>
</html>
```

## 使用说明

1. 将 `__CPN_DATA__` 替换为 json-schema.md 中定义格式的完整 JSON 对象
2. `places` 数组中每个节点需包含 `initial_marking` 字段；资源库所（如"缴费待完成"）若有初始 token，务必在 `initial_marking` 中列出
3. 直接在浏览器打开 HTML 文件即可查看动态 Petri 网
4. 圆形=库所，矩形=变迁，实线=弧，红色虚线=依赖关系；可触发变迁会发光提示
5. 支持自动运行（循环）、单步执行、速度调节
