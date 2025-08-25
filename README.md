<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Nanoseconds Countdown — VSS Widget</title>
  <style>
    :root{--bg:#ffffff;--card:#f9f9f9;--text:#000000;--muted:#555555;--accent:#0078d4;--danger:#d13438}
    *{box-sizing:border-box}html,body{height:100%}body{margin:0;font:16px/1.45 'Helvetica Neue', Helvetica, Arial, sans-serif;font-weight:bold;background:var(--bg);color:var(--text);display:grid;place-items:center;padding:18px}
    .wrap{width:min(980px,95vw);background:var(--card);border:1px solid #ddd;border-radius:16px;padding:16px}
    header{display:flex;justify-content:space-between;align-items:center;gap:12px}
    h1{font-size:18px;margin:0}
    .controls{display:flex;gap:8px;align-items:center}
    input[type=datetime-local]{background:#fff;border:1px solid #ccc;color:var(--text);padding:8px;border-radius:8px;font-family:'Helvetica Neue', Helvetica, Arial, sans-serif;font-weight:bold}
    .btn{appearance:none;background:var(--accent);border:1px solid var(--accent);color:white;padding:8px 10px;border-radius:8px;cursor:pointer;font-family:'Helvetica Neue', Helvetica, Arial, sans-serif;font-weight:bold}
    .screen{margin-top:12px;border-radius:12px;background:#fff;padding:18px;border:1px solid #ddd}
    .ns{font-family:'Helvetica Neue', Helvetica, Arial, sans-serif;font-weight:bold;font-size:clamp(20px,5vw,48px);white-space:nowrap;overflow:auto}
    .label{color:var(--muted);margin-top:8px}
    footer{margin-top:12px;color:var(--muted);font-size:13px;display:flex;justify-content:space-between}
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Nanoseconds Countdown — Azure DevOps Widget (VSS)</h1>
      <div class="controls">
        <input id="target" type="datetime-local" step="1" aria-label="Target date" />
        <button class="btn" id="apply">Start</button>
      </div>
    </header>

    <div class="screen">
      <div id="output" class="ns" aria-live="polite">—</div>
      <div id="status" class="label">Waiting for target…</div>
    </div>

    <footer>
      <div>Use as an Azure DevOps dashboard tile — integrates with VSS SDK.</div>
      <div>Shows <strong>total remaining nanoseconds</strong> (plain number, no commas).</div>
    </footer>
  </div>

  <!-- VSS SDK (required when loaded inside Azure DevOps / VSTS) -->
  <script src="https://unpkg.com/vss-web-extension-sdk@1.165.0/lib/VSS.SDK.min.js"></script>

  <script>
    const $ = (s)=>document.querySelector(s);
    const out = $('#output');
    const statusEl = $('#status');
    const input = $('#target');
    const btn = $('#apply');

    function nowEpochNs(){
      const originMsInt = Math.floor(performance.timeOrigin);
      const nowNs = Math.floor(performance.now()*1e6);
      return BigInt(originMsInt)*1_000_000n + BigInt(nowNs);
    }
    function toEpochNs(date){ return BigInt(date.getTime())*1_000_000n; }

    let targetNs = null; let rafId = null; let paused=false;
    function tick(){
      if(paused||targetNs===null) return; const cur = nowEpochNs(); let rem = targetNs - cur;
      if(rem<=0n){ out.textContent='0'; statusEl.textContent='Reached target.'; cancelAnimationFrame(rafId); rafId=null; return; }
      out.textContent = rem.toString(); statusEl.textContent='Counting down…'; rafId = requestAnimationFrame(tick);
    }
    function startCountdown(date){ targetNs = toEpochNs(date); paused=false; if(rafId) cancelAnimationFrame(rafId); rafId = requestAnimationFrame(tick); }

    btn.addEventListener('click', ()=>{
      const val = input.value; if(!val){ alert('Pick a target date & time first.'); return; }
      const localDate = new Date(val.replace(' ','T')); if(Number.isNaN(localDate.getTime())){ alert('Invalid date/time.'); return; }
      startCountdown(localDate);
    });

    if(window.VSS && VSS.init){
      VSS.init({ explicitNotifyLoaded: true, usePlatformStyles: true });

      VSS.register("nanoseconds.countdown.widget", function(context){
        try{
          const cfg = (context && context.customSettings && context.customSettings.data) ? JSON.parse(context.customSettings.data) : null;
          if(cfg && cfg.target){
            const d = new Date(cfg.target);
            if(!Number.isNaN(d.getTime())){
              startCountdown(d);
              input.value = new Date(d.getTime() - d.getTimezoneOffset()*60000).toISOString().slice(0,19);
            }
          }
        }catch(e){}

        VSS.notifyLoadSucceeded();

        return {
          onConfigurationChange: function(newConfig){
            try{
              const cfg = newConfig && newConfig.data ? JSON.parse(newConfig.data) : null;
              if(cfg && cfg.target){ const d = new Date(cfg.target); if(!Number.isNaN(d.getTime())) startCountdown(d); }
            }catch(e){}
          }
        };
      });

      try{ VSS.notifyLoadSucceeded(); }catch(e){}
    }

    (function initFromQuery(){ const p = new URLSearchParams(location.search); const t = p.get('target'); if(t){ const d = new Date(t); if(!Number.isNaN(d.getTime())){ startCountdown(d); input.value = new Date(d.getTime() - d.getTimezoneOffset()*60000).toISOString().slice(0,19); } } })();
  </script>
</body>
</html>
