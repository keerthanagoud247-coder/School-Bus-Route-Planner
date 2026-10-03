const $=s=>document.querySelector(s);
const cards=$('#cards');
const state={mode:'cluster',optimized:false};
const base=[
 {name:'North Loop',color:'r1',students:22,time:28,stops:6,safety:'11 min'},
 {name:'East Loop',color:'r2',students:21,time:34,stops:7,safety:'12 min'},
 {name:'South Loop',color:'r3',students:20,time:31,stops:6,safety:'13 min'},
 {name:'West Loop',color:'r4',students:21,time:30,stops:5,safety:'12 min'}
];
function render(list=base){
 cards.innerHTML=list.map((x,i)=>`<article class="route-card">
 <div class="route-top"><span class="bus-badge">🚌</span><span class="tag">BUS ${i+1}</span></div>
 <h3>${x.name}</h3><p>${x.students} students · ${x.stops} pickup stops</p>
 <div class="metric"><span>Estimated time</span><b>${x.time} min</b></div>
 <div class="metric"><span>Safety buffer</span><b>${x.safety}</b></div></article>`).join('');
}
render();
document.querySelectorAll('.chip').forEach(b=>b.onclick=()=>{document.querySelectorAll('.chip').forEach(x=>x.classList.remove('active'));b.classList.add('active');state.mode=b.dataset.mode});
$('#optimize').onclick=async()=>{
 const btn=$('#optimize'); btn.disabled=true; btn.innerHTML='✦ Optimizing routes…'; $('#aiBox p').textContent='AI is balancing capacity, stop clusters, travel time, and safety buffers…';
 try{
  const r=await fetch('/api/optimize',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({students:+$('#students').value,capacity:+$('#capacity').value,priority:$('#priority').value,mode:state.mode})});
  const data=await r.json();
  $('#busCount').textContent=data.summary.buses+' buses'; $('#routeTime').textContent=data.summary.averageTime+' min'; $('#buffer').textContent=data.summary.safetyBuffer+' min'; $('#stopCount').textContent=data.summary.stops;
  $('#aiBox p').textContent=data.explanation; $('#mapToast').textContent='AI route preview · '+data.summary.buses+' balanced clusters'; render(data.routes);
 }catch(e){$('#aiBox p').textContent='Demo optimization is ready, but the AI service could not be reached.'}
 btn.disabled=false; btn.innerHTML='<span>✦</span> Optimize with AI';
};
$('#reset').onclick=()=>location.reload();
$('#export').onclick=()=>{const rows=base.map((x,i)=>`Bus ${i+1},${x.name},${x.students},${x.stops},${x.time},${x.safety}`);const csv='Bus,Route,Students,Stops,Time,Safety Buffer\\n'+rows.join('\\n');const a=document.createElement('a');a.href=URL.createObjectURL(new Blob([csv],{type:'text/csv'}));a.download='routewise-plan.csv';a.click()};