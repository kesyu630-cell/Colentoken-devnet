<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Sinyal RSI + Stochastic</title>
<style>
:root{--bg:#0f1115;--card:#181b22;--tx:#e8eaed;--mut:#8b93a1;--g:#22c55e;--r:#ef4444}
@media (prefers-color-scheme:light){:root{--bg:#f4f5f7;--card:#fff;--tx:#111;--mut:#667}}
*{box-sizing:border-box}
body{margin:0;padding:12px;background:var(--bg);color:var(--tx);font:14px system-ui,sans-serif}
h1{font-size:18px;margin:0 0 8px}
.bar{display:flex;gap:6px;flex-wrap:wrap;margin-bottom:10px}
input,button{padding:8px;border-radius:8px;border:1px solid #444;background:var(--card);color:var(--tx);font-size:13px}
button{cursor:pointer}
.wrap{overflow-x:auto}
table{border-collapse:collapse;width:100%;background:var(--card);border-radius:10px}
th,td{padding:8px 6px;text-align:center;border-bottom:1px solid #2a2e37;white-space:nowrap}
.buy{background:rgba(34,197,94,.2);color:var(--g);font-weight:700}
.sell{background:rgba(239,68,68,.2);color:var(--r);font-weight:700}
small{color:var(--mut)}
#log{margin-top:10px;font-size:13px}
</style>
</head>
<body>
<h1>Sinyal RSI + Stochastic</h1>
<div class="bar">
<input id="pairs" value="BTCUSDT,ETHUSDT,SOLUSDT" style="flex:1;min-width:180px">
<button onclick="save()">Simpan &amp; Muat</button>
</div>
<div class="bar">
<input id="tok" placeholder="Token Telegram (opsional)" style="flex:1;min-width:180px">
<input id="cid" placeholder="Chat ID" style="width:120px">
</div>
<small>BUY: RSI&lt;30 &amp; Stoch K&lt;20 · SELL: RSI&gt;70 &amp; Stoch K&gt;80 · update tiap 30 detik · <span id="upd"></span></small>
<div class="wrap" style="margin-top:8px"><table id="t"></table></div>
<div id="log"></div>
<script>
const TF=["1m","5m","15m","30m","1h","4h"];
const $=id=>document.getElementById(id);
const sent={};
function ld(k,d){try{return localStorage.getItem(k)||d}catch(e){return d}}
function st(k,v){try{localStorage.setItem(k,v)}catch(e){}}
$("pairs").value=ld("pairs",$("pairs").value);$("tok").value=ld("tok","");$("cid").value=ld("cid","");
function save(){st("pairs",$("pairs").value);st("tok",$("tok").value);st("cid",$("cid").value);run()}
function rsi(c,p=14){let g=0,l=0;for(let i=1;i<=p;i++){const d=c[i]-c[i-1];d>0?g+=d:l-=d}
g/=p;l/=p;for(let i=p+1;i<c.length;i++){const d=c[i]-c[i-1];g=(g*(p-1)+Math.max(d,0))/p;l=(l*(p-1)+Math.max(-d,0))/p}
return l==0?100:100-100/(1+g/l)}
function stoch(h,l,c,n=14,s=3){const raw=[];for(let i=n-1;i<c.length;i++){const hh=Math.max(...h.slice(i-n+1,i+1)),ll=Math.min(...l.slice(i-n+1,i+1));raw.push(100*(c[i]-ll)/(hh-ll||1))}
const k=[];for(let i=s-1;i<raw.length;i++)k.push(raw.slice(i-s+1,i+1).reduce((a,b)=>a+b)/s);return k[k.length-1]}
async function cek(sym,tf){
const r=await fetch(`https://api.binance.com/api/v3/klines?symbol=${sym}&interval=${tf}&limit=100`);
const d=(await r.json()).slice(0,-1);
const h=d.map(x=>+x[2]),l=d.map(x=>+x[3]),c=d.map(x=>+x[4]);
const R=rsi(c),K=stoch(h,l,c);
let sig="";if(R<30&&K<20)sig="BUY";else if(R>70&&K>80)sig="SELL";
return{R,K,sig,t:d[d.length-1][0],p:c[c.length-1]}}
function notif(m){
const li=document.createElement("div");li.textContent=new Date().toLocaleTimeString()+" — "+m;$("log").prepend(li);
const tok=$("tok").value.trim(),cid=$("cid").value.trim();
if(tok&&cid)fetch(`https://api.telegram.org/bot${tok}/sendMessage`,{method:"POST",headers:{"Content-Type":"application/x-www-form-urlencoded"},body:`chat_id=${cid}&text=${encodeURIComponent(m)}`}).catch(()=>{})}
async function run(){
const pairs=$("pairs").value.split(",").map(s=>s.trim().toUpperCase()).filter(Boolean);
let html="<tr><th>Pair</th>"+TF.map(t=>`<th>${t}</th>`).join("")+"</tr>";
for(const p of pairs){
const res=await Promise.all(TF.map(t=>cek(p,t).catch(()=>null)));
html+=`<tr><td><b>${p.replace("USDT","")}</b></td>`;
res.forEach((x,i)=>{
if(!x){html+="<td>-</td>";return}
html+=`<td class="${x.sig.toLowerCase()}">${x.sig||"·"}<br><small>${x.R.toFixed(0)}/${x.K.toFixed(0)}</small></td>`;
const key=p+TF[i];
if(x.sig&&sent[key]!=x.sig+x.t){sent[key]=x.sig+x.t;notif(`${x.sig} ${p} [${TF[i]}] harga ${x.p} | RSI ${x.R.toFixed(1)} | Stoch ${x.K.toFixed(1)}`)}
});
html+="</tr>"}
$("t").innerHTML=html;$("upd").textContent="update "+new Date().toLocaleTimeString()}
run();setInterval(run,30000);
</script>
</body>
</html>
