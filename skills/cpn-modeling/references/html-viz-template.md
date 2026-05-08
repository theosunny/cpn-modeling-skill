# HTML 可视化模板参考

Claude 生成 HTML 可视化时，使用以下模板结构。将 `__CPN_DATA__` 替换为实际的 JSON 数据对象（即 json-schema.md 中定义的完整 JSON）。

## 模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>CPN 模型可视化</title>
<style>
  body { font-family: sans-serif; background: #f8fafc; margin: 0; padding: 20px; }
  h1 { color: #1e293b; font-size: 18px; margin-bottom: 4px; }
  .subtitle { color: #64748b; font-size: 13px; margin-bottom: 20px; }
  svg { background: white; border: 1px solid #e2e8f0; border-radius: 8px; display: block; }
  .place { fill: white; stroke: #4f46e5; stroke-width: 2; }
  .transition { fill: #4f46e5; }
  .place-label { font-size: 11px; fill: #1e293b; text-anchor: middle; }
  .trans-label { font-size: 11px; fill: white; text-anchor: middle; }
  .arc { stroke: #64748b; stroke-width: 1.5; fill: none; marker-end: url(#arrow); }
  .fs-arc { stroke: #ef4444; stroke-width: 1.5; stroke-dasharray: 6,3; fill: none; marker-end: url(#arrow-red); }
  .legend { display: flex; gap: 20px; margin-top: 12px; font-size: 12px; color: #475569; }
  .legend-item { display: flex; align-items: center; gap: 6px; }
  .sp-新建 { stroke: #4f46e5; } .sp-水电 { stroke: #0891b2; }
  .sp-瓦工 { stroke: #059669; } .sp-木工 { stroke: #d97706; }
  .sp-油漆 { stroke: #dc2626; } .sp-安装 { stroke: #7c3aed; }
</style>
</head>
<body>
<h1>CPN 模型：<span id="project-id"></span></h1>
<p class="subtitle">着色 Petri 网 — 装修 BOM 业务流程</p>
<svg id="cpn-svg" width="900" height="600">
  <defs>
    <marker id="arrow" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#64748b"/>
    </marker>
    <marker id="arrow-red" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#ef4444"/>
    </marker>
  </defs>
</svg>
<div class="legend">
  <div class="legend-item"><svg width="20" height="20"><circle cx="10" cy="10" r="8" fill="white" stroke="#4f46e5" stroke-width="2"/></svg> 库所 (Place)</div>
  <div class="legend-item"><svg width="20" height="20"><rect x="2" y="4" width="16" height="12" fill="#4f46e5"/></svg> 变迁 (Transition)</div>
  <div class="legend-item"><svg width="30" height="20"><line x1="0" y1="10" x2="25" y2="10" stroke="#64748b" stroke-width="1.5"/></svg> 弧</div>
  <div class="legend-item"><svg width="30" height="20"><line x1="0" y1="10" x2="25" y2="10" stroke="#ef4444" stroke-width="1.5" stroke-dasharray="4,2"/></svg> FS 依赖</div>
</div>

<script>
const data = __CPN_DATA__;

const svg = document.getElementById('cpn-svg');
document.getElementById('project-id').textContent = data.project_id;

const subprojects = [...new Set([
  ...data.places.map(p => p.subproject),
  ...data.transitions.map(t => t.subproject)
].filter(Boolean))];

const colWidth = 160, rowHeight = 80, startX = 80, startY = 60;
const positions = {};

subprojects.forEach((sp, ci) => {
  const x = startX + ci * colWidth;
  const spPlaces = data.places.filter(p => p.subproject === sp);
  const spTrans = data.transitions.filter(t => t.subproject === sp);
  spPlaces.forEach((p, ri) => {
    positions[p.id] = { x, y: startY + ri * rowHeight * 2 };
  });
  spTrans.forEach((t, ri) => {
    positions[t.id] = { x, y: startY + rowHeight + ri * rowHeight * 2 };
  });
});

// 绘制弧
data.arcs.forEach(arc => {
  const from = positions[arc.from], to = positions[arc.to];
  if (!from || !to) return;
  const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
  line.setAttribute('x1', from.x); line.setAttribute('y1', from.y);
  line.setAttribute('x2', to.x); line.setAttribute('y2', to.y);
  line.setAttribute('class', 'arc');
  svg.appendChild(line);
});

// 绘制 FS 规则（红色虚线）
data.fs_rules.forEach(rule => {
  const [predSP, predDoc] = rule.predecessor.split('.');
  const [succSP, succDoc] = rule.successor.split('.');
  const fromPlace = data.places.find(p => p.subproject && p.subproject.includes(predSP) && p.name.includes(predDoc));
  const toTrans = data.transitions.find(t => t.subproject && t.subproject.includes(succSP) && t.name.includes(succDoc));
  if (!fromPlace || !toTrans) return;
  const from = positions[fromPlace.id], to = positions[toTrans.id];
  if (!from || !to) return;
  const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
  line.setAttribute('x1', from.x); line.setAttribute('y1', from.y);
  line.setAttribute('x2', to.x); line.setAttribute('y2', to.y);
  line.setAttribute('class', 'fs-arc');
  svg.appendChild(line);
});

// 绘制库所（圆形）
data.places.forEach(place => {
  const pos = positions[place.id];
  if (!pos) return;
  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  const circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
  circle.setAttribute('cx', pos.x); circle.setAttribute('cy', pos.y);
  circle.setAttribute('r', 24);
  circle.setAttribute('class', `place sp-${(place.subproject || '').replace('阶段','')}`);
  const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
  text.setAttribute('x', pos.x); text.setAttribute('y', pos.y + 4);
  text.setAttribute('class', 'place-label');
  text.textContent = place.name.length > 8 ? place.name.slice(0,8)+'…' : place.name;
  const title = document.createElementNS('http://www.w3.org/2000/svg', 'title');
  title.textContent = `${place.name}\n颜色: ${place.color}\n子项目: ${place.subproject}`;
  g.appendChild(circle); g.appendChild(text); g.appendChild(title);
  svg.appendChild(g);
});

// 绘制变迁（矩形）
data.transitions.forEach(trans => {
  const pos = positions[trans.id];
  if (!pos) return;
  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  const rect = document.createElementNS('http://www.w3.org/2000/svg', 'rect');
  rect.setAttribute('x', pos.x - 28); rect.setAttribute('y', pos.y - 16);
  rect.setAttribute('width', 56); rect.setAttribute('height', 32);
  rect.setAttribute('class', 'transition');
  const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
  text.setAttribute('x', pos.x); text.setAttribute('y', pos.y + 4);
  text.setAttribute('class', 'trans-label');
  text.textContent = trans.name.length > 7 ? trans.name.slice(0,7)+'…' : trans.name;
  const title = document.createElementNS('http://www.w3.org/2000/svg', 'title');
  title.textContent = `${trans.name}\n守卫: ${trans.guard || '无'}\n子项目: ${trans.subproject}`;
  g.appendChild(rect); g.appendChild(text); g.appendChild(title);
  svg.appendChild(g);
});
</script>
</body>
</html>
```

## 使用说明

1. 将 `__CPN_DATA__` 替换为 json-schema.md 中定义格式的完整 JSON 对象
2. 直接在浏览器打开 HTML 文件即可查看 Petri 网可视化
3. 悬停节点可查看详细信息（名称、颜色、子项目、守卫条件）
4. 圆形 = 库所，矩形 = 变迁，实线箭头 = 弧，红色虚线 = FS 依赖
