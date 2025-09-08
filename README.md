<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Domain & Range Calculator + Graph</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <script src="https://cdn.jsdelivr.net/npm/mathjs@11.8.0/lib/browser/math.js"></script>
  <script src="https://cdn.plot.ly/plotly-2.25.2.min.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 16px; background:#f7f9fc; color:#111; }
    .card { background:white; border-radius:10px; padding:16px; box-shadow:0 6px 18px rgba(20,30,60,0.08); max-width:980px; margin:auto; }
    h1{margin:0 0 8px 0}
    label{display:block;margin-top:12px}
    input[type=text]{width:100%; padding:10px; font-size:16px; border-radius:6px; border:1px solid #ddd}
    button{margin-top:12px;padding:10px 16px; font-size:16px;border-radius:8px;border:none;background:#2d6cdf;color:white;cursor:pointer}
    pre{background:#f3f6ff;padding:12px;border-radius:8px;overflow:auto}
    #plot{height:420px}
    .row{display:flex;gap:16px;flex-wrap:wrap}
    .col{flex:1;min-width:280px}
    .muted{color:#555;font-size:14px}
  </style>
</head>
<body>
  <div class="card">
    <h1>Domain & Range Calculator + Graph</h1>
    <p class="muted">Type a function in terms of <code>x</code>. Examples: <code>(x^2-4)/(x^2+1)</code>, <code>sqrt(x-2)</code>, <code>log(x-3)</code>, <code>exp(1/x)</code></p>

    <label for="expr">f(x) =</label>
    <input id="expr" type="text" placeholder="e.g. (x^2-4)/(x^2+1) or sqrt(x-2)" value="(x^2-4)/(x^2+1)">
    <div style="display:flex;gap:8px;">
      <button id="analyzeBtn">Find Domain & Range + Plot</button>
      <button id="resetBtn" style="background:#6c757d">Reset</button>
    </div>

    <div class="row" style="margin-top:16px">
      <div class="col">
        <h3>Results</h3>
        <div id="explanation"><em>Press <strong>Find Domain & Range + Plot</strong> to start.</em></div>
        <h4>Domain (approx)</h4>
        <pre id="domainOut">—</pre>
        <h4>Range (approx)</h4>
        <pre id="rangeOut">—</pre>
      </div>

      <div class="col">
        <h3>Graph</h3>
        <div id="plot"></div>
      </div>
    </div>
  </div>

<script>
/*
  Prototype domain & range calculator:
  - Uses math.js for parsing/evaluation
  - Uses numeric sampling to detect domain and estimate range
  - Provides analytic hints for sqrt() and log()
  - Not a complete symbolic solver but works well for most standard functions
*/

const exprInput = document.getElementById('expr');
const analyzeBtn = document.getElementById('analyzeBtn');
const resetBtn = document.getElementById('resetBtn');
const domainOut = document.getElementById('domainOut');
const rangeOut = document.getElementById('rangeOut');
const explanation = document.getElementById('explanation');

function cleanExpression(s){
  // math.js supports ^ and sqrt, log, exp. Replace any common user variants:
  return s.replace(/÷/g,'/').replace(/×/g,'*').trim();
}

function sampleFunction(compiled, x){
  // Evaluate compiled expression at x; return {ok:bool, val:number, err:optional}
  try{
    const v = compiled.evaluate({x: x});
    if (typeof v === 'number' && isFinite(v)) return {ok:true, val:v};
    // sometimes mathjs returns complex or objects
    if (typeof v === 'object' && v.re !== undefined && isFinite(v.re)) return {ok:true, val:v.re};
    return {ok:false, err:'non-finite'};
  } catch(e){
    return {ok:false, err: e.toString()};
  }
}

function detectAnalyticHints(expr){
  const hints = [];
  const lower = expr.toLowerCase();
  if (lower.includes('sqrt(') || lower.includes('root(')){
    hints.push('Contains √ (square root) — require expression inside √ to be ≥ 0.');
  }
  if (lower.includes('log(') || /(^|[^a-zA-Z])ln\(/.test(lower)){
    hints.push('Contains log/ln — require argument of log to be > 0.');
  }
  if (lower.includes('(') && lower.includes('/') ){
    hints.push('There is a division — watch for values that make denominators 0 (vertical asymptotes).');
  }
  return hints;
}

function contiguousIntervals(validXs, tol=1e-6){
  // validXs sorted ascending. Return contiguous ranges where adjacent gaps <= step*1.5
  if (validXs.length===0) return [];
  const intervals = [];
  let start = validXs[0], prev = validXs[0];
  for (let i=1;i<validXs.length;i++){
    const x = validXs[i];
    if (Math.abs(x - prev) <= (validXs[1]-validXs[0])*1.6 + tol){ // still contiguous
      prev = x;
      continue;
    } else {
      intervals.push([start, prev]);
      start = x; prev = x;
    }
  }
  intervals.push([start, prev]);
  return intervals;
}

function approximateDomainAndRange(expression){
  const expr = cleanExpression(expression);
  let compiled;
  try {
    compiled = math.compile(expr);
  } catch(e){
    return {error:'Parse error: ' + e.toString()};
  }

  // We'll sample a wide x-range to detect where the function is defined.
  // Reasonable window: [-100,100], but denser near origin.
  const sampleXS = [];
  // combine coarse and fine sampling
  for(let x=-100; x<=100; x+=1) sampleXS.push(x);
  for(let x=-10; x<=10; x+=0.25) sampleXS.push(x);
  for(let x=-2; x<=2; x+=0.05) sampleXS.push(x);
  // unique & sort
  const uniq = Array.from(new Set(sampleXS)).sort((a,b)=>a-b);

  const validX = [];
  const yvals = [];
  const failedPoints = [];
  for(const x of uniq){
    // skip exactly x=0 for expressions that blow up numerically; we still test though
    const r = sampleFunction(compiled, x);
    if (r.ok) { validX.push(x); yvals.push(r.val); }
    else { failedPoints.push({x, err: r.err}); }
  }

  // Domain intervals (approx)
  const intervals = contiguousIntervals(validX);
  const domainStr = intervals.map(it=>{
    if (Math.abs(it[0] - it[1]) < 1e-9) return it[0].toFixed(3);
    return `[${it[0].toFixed(3)}, ${it[1].toFixed(3)}]`;
  }).join(' ∪ ');
  // Range approximate
  let rangeStr = 'Could not determine';
  if (yvals.length > 0){
    const min = Math.min(...yvals);
    const max = Math.max(...yvals);
    // check if extreme appears unbounded: look at tails if |y| grows with |x|
    const tailLeft = yvals.slice(0,10);
    const tailRight = yvals.slice(-10);
    const tailGrows = (arr)=> {
      const av = arr.reduce((s,v)=>s+Math.abs(v),0)/arr.length;
      return av > 1e6; // arbitrary huge
    };
    const unbounded = tailGrows(tailLeft) || tailGrows(tailRight);
    rangeStr = (unbounded ? 'Unbounded' : `[${min.toFixed(6)}, ${max.toFixed(6)}]`);
  }

  // Provide analytic hints (sqrt/log/denominator)
  const hints = detectAnalyticHints(expr);

  // For better messages: try to analyze denominator 0 points by looking for x where eval returns Infinity or throws for points near them
  const denomWarnings = [];
  // check failedPoints: if many failed clustered at certain x, flag holes/asymptotes
  const failedXs = failedPoints.map(f=>f.x);
  if (failedXs.length){
    // find clusters
    const clusters = contiguousIntervals(failedXs);
    clusters.forEach(c=>{
      denomWarnings.push(`Function failed or is undefined around x ≈ ${c[0].toFixed(3)} to ${c[1].toFixed(3)} (possible vertical asymptote or denominator = 0).`);
    });
  }

  // Build a cleaned domain representation: if intervals cover a wide range and form single continuous all real, say all real
  let domainReadable = domainStr;
  // If first interval starts near -100 and last ends near 100 then treat as all real with holes maybe
  if (intervals.length>0 && Math.abs(intervals[0][0]+100) < 1e-6 && Math.abs(intervals[intervals.length-1][1]-100) < 1e-6){
    domainReadable = 'All real numbers (except possible points where function failed - see warnings)';
  }

  return {
    expr: expr,
    compiled: compiled,
    sampleX: uniq,
    validX: validX,
    domainIntervals: intervals,
    domainReadable: domainReadable || 'None found',
    domainNumeric: domainStr || '—',
    rangeNumeric: rangeStr,
    hints: hints,
    denomWarnings: denomWarnings,
    yvals: yvals
  };
}

function producePlot(compiled, validX, yvals, expr){
  // Create plotly trace from sampled points. ValidX & yvals correspond.
  const trace = { x: validX, y: yvals, mode: 'lines', name: `f(x) = ${expr}`, line:{width:2} };
  const layout = {
    margin:{t:30,b:40,l:40,r:20},
    xaxis:{title:'x'},
    yaxis:{title:'f(x)'},
    hovermode:'closest'
  };
  Plotly.newPlot('plot', [trace], layout, {responsive:true});
}

analyzeBtn.addEventListener('click', ()=> {
  domainOut.textContent = 'Working...';
  rangeOut.textContent = 'Working...';
  explanation.innerHTML = '';
  const raw = exprInput.value.trim();
  if(!raw){ explanation.innerHTML = '<span style="color:#a00">Please enter a function.</span>'; return; }

  let result;
  try {
    result = approximateDomainAndRange(raw);
  } catch(e){
    explanation.innerHTML = `<span style="color:#a00">Error: ${e.toString()}</span>`;
    return;
  }
  if (result.error){
    explanation.innerHTML = `<span style="color:#a00">${result.error}</span>`;
    domainOut.textContent = '—';
    rangeOut.textContent = '—';
    return;
  }

  // Explanation text
  const lines = [];
  lines.push(`<strong>Function parsed as:</strong> <code>f(x) = ${result.expr}</code>`);
  if(result.hints.length) {
    lines.push(`<strong>Analytic hints:</strong>`);
    result.hints.forEach(h=>lines.push(`<div>• ${h}</div>`));
  } else {
    lines.push(`<div>• No immediate sqrt/log/division patterns found.</div>`);
  }
  if(result.denomWarnings.length){
    lines.push(`<strong>Numerical warnings:</strong>`);
    result.denomWarnings.forEach(w=>lines.push(`<div style="color:#b33">• ${w}</div>`));
  }

  explanation.innerHTML = lines.join('\n');

  // Domain & Range outputs
  domainOut.textContent = result.domainNumeric || result.domainReadable;
  rangeOut.textContent = result.rangeNumeric;

  // Plot
  if (result.validX.length && result.yvals.length){
    producePlot(result.compiled, result.validX, result.yvals, result.expr);
  } else {
    document.getElementById('plot').innerHTML = '<div style="padding:40px;color:#555">No valid points to plot (function may be undefined for tested range).</div>';
  }
});

resetBtn.addEventListener('click', ()=>{
  exprInput.value = '';
  domainOut.textContent = '—';
  rangeOut.textContent = '—';
  explanation.innerHTML = '<em>Press <strong>Find Domain & Range + Plot</strong> to start.</em>';
  document.getElementById('plot').innerHTML = '';
});
</script>
</body>
</html>
