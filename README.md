<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Domain & Range Calculator — Improved (∞ support)</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <script src="https://cdn.jsdelivr.net/npm/mathjs@11.8.0/lib/browser/math.js"></script>
  <script src="https://cdn.plot.ly/plotly-2.25.2.min.js"></script>
  <style>
    body { font-family: Inter, Arial, sans-serif; margin: 16px; background:#f7f9fc; color:#111 }
    .card { background:#fff; border-radius:10px; padding:18px; box-shadow:0 8px 28px rgba(20,30,60,0.08); max-width:1100px; margin:auto }
    h1{margin:0 0 8px 0}
    label{display:block;margin-top:12px}
    input[type=text]{width:100%; padding:10px; font-size:16px; border-radius:8px; border:1px solid #ddd}
    button{margin-top:12px;padding:10px 16px; font-size:15px;border-radius:8px;border:none;background:#2d6cdf;color:white;cursor:pointer}
    button.secondary{background:#6c757d}
    pre{background:#fbfdff;padding:12px;border-radius:8px;overflow:auto;border:1px solid #eef3ff}
    #plot{height:480px}
    .row{display:flex;gap:18px;flex-wrap:wrap}
    .col{flex:1;min-width:320px}
    .muted{color:#556;font-size:14px}
    .warn{color:#b33}
    .good{color:green}
  </style>
</head>
<body>
  <div class="card">
    <h1>Domain & Range Calculator — Improved (∞ support)</h1>
    <p class="muted">Enter a function f(x). This version detects unbounded behavior (±∞), marks likely vertical asymptotes and shows approximate domain & range using sampling + analytic hints.</p>

    <label for="expr">f(x) =</label>
    <input id="expr" type="text" placeholder="e.g. (x^2-4)/(x^2+1)   or   sqrt(x-2)   or   1/x" value="(x^2-4)/(x^2+1)">
    <div style="display:flex;gap:8px;flex-wrap:wrap;margin-top:8px;">
      <button id="analyzeBtn">Analyze & Plot</button>
      <button id="resetBtn" class="secondary">Reset</button>
      <button id="examplesBtn" class="secondary">Examples</button>
    </div>

    <div class="row" style="margin-top:16px">
      <div class="col">
        <h3>Explanation & Hints</h3>
        <div id="explanation"><em>Press <strong>Analyze & Plot</strong> to start.</em></div>

        <h4 style="margin-top:12px">Domain (approx)</h4>
        <pre id="domainOut">—</pre>

        <h4>Range (approx)</h4>
        <pre id="rangeOut">—</pre>

        <div id="notes" style="margin-top:8px" class="muted"></div>
      </div>

      <div class="col">
        <h3>Graph</h3>
        <div id="plot"></div>
      </div>
    </div>
  </div>

<script>
/* Improved numeric prototype that reports ±∞ and asymptotes.
   - Sampling over multi-scale grid
   - Detects finite domain intervals, marks -∞/+∞ when sampling hits bounds
   - Detects unbounded range via large magnitudes / infinity evaluations
   - Marks vertical asymptotes (where evaluation fails or returns Infinity)
   Limitations: numeric sampling only — exact symbolic holes (like missing y=0 for 1/x) may not be detected precisely.
*/

const exprInput = document.getElementById('expr');
const analyzeBtn = document.getElementById('analyzeBtn');
const resetBtn = document.getElementById('resetBtn');
const examplesBtn = document.getElementById('examplesBtn');
const domainOut = document.getElementById('domainOut');
const rangeOut = document.getElementById('rangeOut');
const explanation = document.getElementById('explanation');
const notes = document.getElementById('notes');

function cleanExpression(s){ return s.replace(/÷/g,'/').replace(/×/g,'*').trim(); }

function sampleFunction(compiled, x){
  try{
    const v = compiled.evaluate({x: x});
    // Accept numeric, boolean? convert booleans to numbers
    if (typeof v === 'number'){
      if (Number.isFinite(v)) return {status:'ok', val: v};
      if (v === Infinity) return {status:'inf', val: Infinity};
      if (v === -Infinity) return {status:'ninf', val: -Infinity};
      return {status:'nan', err:'non-finite'};
    }
    // complex value
    if (typeof v === 'object' && v !== null && v.re !== undefined){
      if (Number.isFinite(v.re)) return {status:'ok', val: v.re};
      return {status:'nan', err:'complex'};
    }
    return {status:'nan', err:'unsupported'};
  } catch(e){
    return {status:'err', err: e.toString()};
  }
}

function detectAnalyticHints(expr){
  const hints = [];
  const lower = expr.toLowerCase();
  if (lower.includes('sqrt(') || lower.includes('root(')) hints.push('Contains √ — inside must be ≥ 0.');
  if (lower.includes('log(') || /(^|[^a-zA-Z])ln\(/.test(lower)) hints.push('Contains log/ln — argument must be > 0.');
  if (lower.includes('/') ) hints.push('Contains division — denominators may be 0 (vertical asymptotes).');
  if (lower.includes('^')) hints.push('Power ^ present — check even/odd powers for sign/negativity constraints.');
  return hints;
}

function contiguousIntervals(sortedXs, stepTolerance){
  if (!sortedXs.length) return [];
  const intervals = [];
  let start = sortedXs[0], prev = sortedXs[0];
  for (let i=1;i<sortedXs.length;i++){
    const x = sortedXs[i];
    if (Math.abs(x - prev) <= stepTolerance) { prev = x; continue; }
    intervals.push([start, prev]); start = x; prev = x;
  }
  intervals.push([start, prev]);
  return intervals;
}

function approximateDomainAndRange(expression){
  const expr = cleanExpression(expression);
  let compiled;
  try { compiled = math.compile(expr); } catch(e){ return {error: 'Parse error: ' + e.toString()}; }

  // Multi-scale sampling ranges
  const samples = [];
  // coarse
  for (let x=-2000; x<=2000; x+=20) samples.push(Number(x));
  // medium
  for (let x=-400; x<=400; x+=1) samples.push(Number(x));
  // fine
  for (let x=-10; x<=10; x+=0.05) samples.push(Number(x));
  // near zero super fine
  for (let x=-1; x<=1; x+=0.005) samples.push(Number(x));

  const uniq = Array.from(new Set(samples)).sort((a,b)=>a-b);

  const finiteXs = [], finiteYs = [];
  const infPoints = []; // {x, sign}
  const failedPoints = [];

  for (const x of uniq){
    const r = sampleFunction(compiled, x);
    if (r.status === 'ok'){
      finiteXs.push(x); finiteYs.push(r.val);
    } else if (r.status === 'inf' || r.status === 'ninf'){
      infPoints.push({x:x, sign: r.status === 'inf' ? 1 : -1});
      // treat as failure for domain but record asymptote indicator
      failedPoints.push({x, err: 'infinite'});
    } else {
      // err or nan
      failedPoints.push({x, err: r.err || r.status});
    }
  }

  // Build domain intervals from finiteXs
  let domainIntervals = [];
  if (finiteXs.length){
    // Determine a representative step tolerance by looking at smallest positive diff
    const diffs = [];
    for (let i=1;i<finiteXs.length;i++) diffs.push(Math.abs(finiteXs[i]-finiteXs[i-1]));
    const minStep = diffs.length ? Math.min(...diffs) : 0.05;
    domainIntervals = contiguousIntervals(finiteXs, minStep*1.5);
  }

  const minSample = uniq[0], maxSample = uniq[uniq.length-1];
  const domainReadable = domainIntervals.map(it=>{
    const start = it[0], end = it[1];
    const leftInfinite = Math.abs(start - minSample) < 1e-9;
    const rightInfinite = Math.abs(end - maxSample) < 1e-9;
    const leftStr = leftInfinite ? '(-∞' : '[' + start.toFixed(6);
    const rightStr = rightInfinite ? '∞)' : end.toFixed(6) + ']';
    return leftStr + ', ' + rightStr;
  }).join(' ∪ ');

  // Range analysis
  let rangeReadable = '—';
  const notes = [];
  const finiteExist = finiteYs.length > 0;

  // detect unboundedness by sampled magnitudes or tail behavior
  const LARGE_MAG = 1e8; // threshold for treating as ±∞ in sampling
  let unboundedPos = false, unboundedNeg = false;

  // check explicit Infinity samples
  if (infPoints.length){
    // if we have sign info, mark accordingly
    for (const p of infPoints){ if (p.sign>0) unboundedPos = true; else unboundedNeg = true; }
  }

  // check finiteYs magnitudes
  if (finiteExist){
    const minY = Math.min(...finiteYs), maxY = Math.max(...finiteYs);
    if (maxY > LARGE_MAG) unboundedPos = true;
    if (minY < -LARGE_MAG) unboundedNeg = true;
  }

  // tail checks: evaluate at large |x|
  const tailXs = [1e3, -1e3, 1e4, -1e4];
  for (const tx of tailXs){
    const r = sampleFunction(compiled, tx);
    if (r.status === 'ok'){
      if (r.val > 1e7) unboundedPos = true;
      if (r.val < -1e7) unboundedNeg = true;
    } else if (r.status === 'inf') unboundedPos = true;
    else if (r.status === 'ninf') unboundedNeg = true;
  }

  // Build approximate range string
  if (!finiteExist){
    if (unboundedPos && unboundedNeg) rangeReadable = '(-∞, ∞)';
    else if (unboundedPos) rangeReadable = '[some finite, ∞) or (−∞, ∞) (no finite samples)';
    else if (unboundedNeg) rangeReadable = '(-∞, some finite] (no finite samples)';
    else rangeReadable = 'No finite values detected on sampled range';
  } else {
    const minY = Math.min(...finiteYs), maxY = Math.max(...finiteYs);
    if (unboundedPos && unboundedNeg) {
      rangeReadable = '(-∞, ∞)';
    } else if (unboundedPos) {
      rangeReadable = `[${minY.toFixed(6)}, ∞)`;
    } else if (unboundedNeg) {
      rangeReadable = `(-∞, ${maxY.toFixed(6)}]`;
    } else {
      rangeReadable = `[${minY.toFixed(6)}, ${maxY.toFixed(6)}]`;
    }
  }

  // Identify candidate vertical asymptotes from failedPoints clusters
  const failedXs = failedPoints.map(f=>f.x).sort((a,b)=>a-b);
  const asymClusters = [];
  if (failedXs.length){
    // cluster nearby failed xs
    let s = failedXs[0], p = failedXs[0];
    for (let i=1;i<failedXs.length;i++){
      const x = failedXs[i];
      if (Math.abs(x - p) <= 1e-6 + 1.5) { p = x; continue; }
      asymClusters.push([s,p]); s = x; p = x;
    }
    asymClusters.push([s,p]);
  }

  // Build readable warning lines
  const hints = detectAnalyticHints(expr);
  const warnings = [];
  if (asymClusters.length){
    asymClusters.forEach(c=>{
      const mid = ((c[0]+c[1])/2).toFixed(6);
      warnings.push(`Possible vertical asymptote near x ≈ ${mid}`);
    });
  }
  if (unboundedPos) warnings.push('Range appears unbounded above (→ +∞)');
  if (unboundedNeg) warnings.push('Range appears unbounded below (→ -∞)');

  return {
    expr: expr,
    compiled,
    sampleX: uniq,
    finiteX: finiteXs,
    finiteY: finiteYs,
    domainIntervals,
    domainReadable: domainReadable || 'No finite domain intervals detected',
    rangeReadable,
    hints,
    warnings,
    asymClusters
  };
}

function producePlot(result){
  const expr = result.expr;
  const xs = result.finiteX;
  const ys = result.finiteY;
  // Clip plotted values to keep plot legible
  const PLOT_CLIP = 1e4;
  const yClipped = ys.map(v => (Math.abs(v) > PLOT_CLIP ? (v>0?PLOT_CLIP:-PLOT_CLIP) : v));

  const trace = { x: xs, y: yClipped, mode: 'lines', name: `f(x) = ${expr}`, line:{width:2} };
  const shapes = [];
  // vertical asymptote markers
  result.asymClusters.forEach(c=>{
    const xmid = (c[0]+c[1])/2;
    shapes.push({type:'line', x0:xmid, x1:xmid, y0:-1e4, y1:1e4, line:{dash:'dashdot', color:'crimson', width:1}});
  });

  const layout = {
    margin:{t:30,b:40,l:50,r:20},
    xaxis:{title:'x'},
    yaxis:{title:'f(x)'},
    shapes: shapes,
    annotations: []
  };

  // if clipping happened, add note
  const clipped = ys.some(v=>Math.abs(v) > PLOT_CLIP);
  if (clipped){
    layout.annotations.push({ xref:'paper', yref:'paper', x:0.99, xanchor:'right', y:0.98, text:`Values clipped to ±${PLOT_CLIP} for plotting`, showarrow:false, font:{size:12,color:'#555'} });
  }

  Plotly.newPlot('plot', [trace], layout, {responsive:true});
}

analyzeBtn.addEventListener('click', ()=>{
  domainOut.textContent = 'Working...'; rangeOut.textContent = 'Working...'; explanation.innerHTML = '';
  notes.textContent = '';
  const raw = exprInput.value.trim();
  if(!raw){ explanation.innerHTML = '<span class="warn">Please enter a function.</span>'; return; }

  const result = approximateDomainAndRange(raw);
  if (result.error){ explanation.innerHTML = `<span class="warn">${result.error}</span>`; domainOut.textContent='—'; rangeOut.textContent='—'; return; }

  const lines = [];
  lines.push(`<div><strong>Parsed as:</strong> <code>f(x) = ${result.expr}</code></div>`);
  if (result.hints.length){ lines.push('<div><strong>Hints:</strong></div>'); result.hints.forEach(h=>lines.push(`<div>• ${h}</div>`)); }
  if (result.warnings.length){ lines.push('<div class="warn"><strong>Warnings:</strong></div>'); result.warnings.forEach(w=>lines.push(`<div>• ${w}</div>`)); }

  explanation.innerHTML = lines.join('\n');

  // Domain and range
  domainOut.textContent = result.domainReadable;
  rangeOut.textContent = result.rangeReadable;
  if (result.warnings.length) notes.innerHTML = '<strong>Notes:</strong> Numeric sampling — for exact symbolic holes (single missing y-values) a symbolic CAS is needed.';

  producePlot(result);
});

resetBtn.addEventListener('click', ()=>{
  exprInput.value = '';
  domainOut.textContent = '—'; rangeOut.textContent = '—'; explanation.innerHTML = '<em>Press <strong>Analyze & Plot</strong> to start.</em>'; notes.textContent = '';
  document.getElementById('plot').innerHTML = '';
});

examplesBtn.addEventListener('click', ()=>{
  const examples = [ '1/x', 'sqrt(x-2)', 'log(x-3)', '(x^2-4)/(x^2+1)', 'e^(1/x)', '1/(x^2-4)', ' (x+1)/(x-1) ' ];
  const next = examples[Math.floor(Math.random()*examples.length)];
  exprInput.value = next; analyzeBtn.click();
});
</script>
</body>
</html>
