<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Boss Hunt Scheduler</title>
<style>
  body { font-family: Arial, sans-serif; background: #f5f5f5; color: #000; padding: 20px; transition: background 0.3s, color 0.3s; }
  h1 { color: #ff6f61; font-weight: bold; }
  h2, h3 { margin-top: 20px; font-weight: bold; }
  table { width: 100%; border-collapse: collapse; margin-top: 10px; }
  th, td { border: 1px solid #555; padding: 8px; text-align: center; font-weight: bold; }
  input, button, select { padding: 5px; margin: 5px; }
  #current-boss { font-size: 2.5em; font-weight: bold; text-align: center; margin-bottom: 20px; padding: 5px 10px; border-radius: 5px; transition: color 0.5s; }
  #options { margin-bottom: 20px; }
  @keyframes flash-bg {
    0%,100% { background-color: yellow; color: black; }
    50% { background-color: orange; color: black; }
  }
  .flashing { animation: flash-bg 1s infinite; }
</style>
</head>
<body>

<h1>Boss Hunt Scheduler</h1>
<h2 id="current-boss">No boss respawning now</h2>

<!-- Options -->
<div id="options">
  <h3>Theme:</h3>
  <select id="theme-toggle" onchange="toggleTheme()">
    <option value="light">Light Mode</option>
    <option value="night">Night Mode</option>
  </select>
</div>

<!-- Admin Login -->
<div id="admin-login">
  <h3>Admin Login</h3>
  <label>Username: <input type="text" id="admin-username" /></label>
  <label>Password: <input type="password" id="admin-password" /></label>
  <button onclick="checkAdmin()">Login</button>
</div>

<!-- Admin Panels -->
<div id="admin-panels" style="display:none;">
  <div class="admin">
    <h3>Add Boss</h3>
    <label>Boss Type:
      <select id="boss-type" onchange="toggleRespawnDays()">
        <option value="">Select Type</option>
        <option value="Fixed">Fixed</option>
        <option value="Dynamic">Dynamic</option>
      </select>
    </label>
    <label>Boss Name: <input type="text" id="boss-name" /></label>
    <label>Respawn Duration (hours): <input type="number" id="respawn-hours" /></label>
    <div id="respawn-days-container" style="display:none;">
      <label>Respawn Days:</label><br>
      <label><input type="checkbox" value="0" class="respawn-day"> Sunday</label>
      <label><input type="checkbox" value="1" class="respawn-day"> Monday</label>
      <label><input type="checkbox" value="2" class="respawn-day"> Tuesday</label>
      <label><input type="checkbox" value="3" class="respawn-day"> Wednesday</label>
      <label><input type="checkbox" value="4" class="respawn-day"> Thursday</label>
      <label><input type="checkbox" value="5" class="respawn-day"> Friday</label>
      <label><input type="checkbox" value="6" class="respawn-day"> Saturday</label>
    </div>
    <button onclick="addBoss()">Add Boss</button>
  </div>

  <div class="admin">
    <h3>Log Boss Kill</h3>
    <label>Select Boss:
      <select id="boss-select"><option value="">Select Boss</option></select>
    </label>
    <label>Killed Date: <input type="date" id="killed-date" /></label>
    <label>Killed Time: <input type="time" id="killed-time" /></label>
    <button onclick="logBossKill()">Log Kill</button>
  </div>
</div>

<h2>🕒 Soon to Respawn (&lt;12h)</h2>
<table id="soon-table"><thead><tr><th>Logo</th><th>Boss Name</th><th>Type</th><th>Next Respawn</th><th>Time Remaining</th></tr></thead><tbody></tbody></table>

<h2>⏳ Upcoming (12–24h)</h2>
<table id="upcoming-table"><thead><tr><th>Logo</th><th>Boss Name</th><th>Type</th><th>Next Respawn</th><th>Time Remaining</th></tr></thead><tbody></tbody></table>

<h2>📊 Dynamic Bosses</h2>
<table id="dynamic-table"><thead><tr><th>Logo</th><th>Boss Name</th><th>Next Respawn</th><th>Time Remaining</th></tr></thead><tbody></tbody></table>

<h2>📌 Fixed Bosses</h2>
<table id="fixed-table"><thead><tr><th>Logo</th><th>Boss Name</th><th>Next Respawn</th><th>Time Remaining</th></tr></thead><tbody></tbody></table>

<h2>🟡 Still Alive</h2>
<table id="still-alive-table"><thead><tr><th>Logo</th><th>Boss Name</th><th>Type</th></tr></thead><tbody></tbody></table>

<script>
const ADMIN_CREDENTIALS = { username: "RonJayvee", password: "mypassword123" };
let bosses = {}; // {name: {type, respawnHours, respawnDays, lastKilled}}

// Toggle respawn days for Fixed bosses
function toggleRespawnDays(){
  document.getElementById('respawn-days-container').style.display =
    document.getElementById('boss-type').value === 'Fixed' ? 'block' : 'none';
}

// Admin login
function checkAdmin(){
  const username = document.getElementById('admin-username').value;
  const password = document.getElementById('admin-password').value;
  if(username === ADMIN_CREDENTIALS.username && password === ADMIN_CREDENTIALS.password){
    document.getElementById('admin-login').style.display = 'none';
    document.getElementById('admin-panels').style.display = 'block';
    setDefaultKilledDate();
  } else alert("Incorrect admin credentials!");
}

// Add boss
function addBoss(){
  const type = document.getElementById('boss-type').value;
  const name = document.getElementById('boss-name').value.trim();
  const hours = parseFloat(document.getElementById('respawn-hours').value);
  if(!type || !name || isNaN(hours) || hours <= 0){ alert('Enter valid type, name, and hours!'); return; }

  let respawnDays = [];
  if(type === "Fixed"){
    const checkedBoxes = document.querySelectorAll('.respawn-day:checked');
    respawnDays = Array.from(checkedBoxes).map(cb => parseInt(cb.value));
    if(respawnDays.length === 0){ alert('Select at least one day for Fixed boss'); return; }
  }

  bosses[name] = { type, respawnHours: hours, respawnDays, lastKilled: null };
  updateBossDropdown();
  updateTables();
}

// Update boss dropdown for Admin 2
function updateBossDropdown(){
  const select = document.getElementById('boss-select');
  select.innerHTML = '<option value="">Select Boss</option>';
  Object.keys(bosses).forEach(name=>{
    const opt = document.createElement('option'); opt.value=name; opt.textContent=name;
    select.appendChild(opt);
  });
}

// Log boss kill
function logBossKill(){
  const name = document.getElementById('boss-select').value;
  const dateInput = document.getElementById('killed-date').value;
  const timeInput = document.getElementById('killed-time').value;
  if(!name || !dateInput || !timeInput){ alert('Select boss, date, and time!'); return; }
  const [y,m,d] = dateInput.split('-').map(Number);
  const [hh,mm] = timeInput.split(':').map(Number);
  bosses[name].lastKilled = new Date(y,m-1,d,hh,mm,0);
  updateTables();
}

// Default killed date today
function setDefaultKilledDate(){
  const now = new Date();
  const y = now.getFullYear();
  const m = String(now.getMonth()+1).padStart(2,'0');
  const d = String(now.getDate()).padStart(2,'0');
  document.getElementById('killed-date').value = `${y}-${m}-${d}`;
}

// Calculate next spawn
function getNextSpawn(boss){
  if(!boss.lastKilled) return null;
  let next = new Date(boss.lastKilled.getTime() + boss.respawnHours*3600000);
  if(boss.type==="Fixed" && boss.respawnDays.length){
    const today = next.getDay();
    let minDiff = 7;
    boss.respawnDays.forEach(day=>{
      let diff = (day-today+7)%7;
      if(diff===0 && next>new Date()) diff=0;
      minDiff=Math.min(minDiff,diff);
    });
    next.setDate(next.getDate()+minDiff);
  }
  return next;
}

// Format remaining time
function formatTime(ms){
  ms = Math.max(ms,0);
  const sec = Math.floor(ms/1000);
  const h = Math.floor(sec/3600);
  const m = Math.floor((sec%3600)/60);
  const s = sec%60;
  return `${String(h).padStart(2,'0')}:${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`;
}

// Toggle theme
function toggleTheme(){
  const theme = document.getElementById('theme-toggle').value;
  document.body.style.background = theme==='night' ? '#121212' : '#f5f5f5';
  document.body.style.color = theme==='night' ? '#fff' : '#000';
  updateCurrentBoss();
}

// Update all tables
function updateTables(){
  const soonBody = document.getElementById('soon-table').querySelector('tbody');
  const upcomingBody = document.getElementById('upcoming-table').querySelector('tbody');
  const fixedBody = document.getElementById('fixed-table').querySelector('tbody');
  const dynamicBody = document.getElementById('dynamic-table').querySelector('tbody');
  const stillBody = document.getElementById('still-alive-table').querySelector('tbody');

  [soonBody, upcomingBody, fixedBody, dynamicBody, stillBody].forEach(tb=>tb.innerHTML='');

  const nowPH = new Date();

  Object.keys(bosses).forEach(name=>{
    const boss = bosses[name];
    const nextSpawn = getNextSpawn(boss);
    const remainingMs = nextSpawn ? nextSpawn-nowPH : null;
    const remainingHrs = remainingMs ? remainingMs/3600000 : null;

    if(remainingHrs!==null && remainingHrs<=12 && remainingHrs>0){
      const row=document.createElement('tr');
      row.innerHTML=`<td>🕒</td><td>${name}</td><td>${boss.type}</td><td>${nextSpawn.toLocaleString('en-US',{timeZone:'Asia/Manila'})}</td><td>${formatTime(remainingMs)}</td>`;
      soonBody.appendChild(row);
    } else if(remainingHrs!==null && remainingHrs>12 && remainingHrs<=24){
      const row=document.createElement('tr');
      row.innerHTML=`<td>⏳</td><td>${name}</td><td>${boss.type}</td><td>${nextSpawn.toLocaleString('en-US',{timeZone:'Asia/Manila'})}</td><td>${formatTime(remainingMs)}</td>`;
      upcomingBody.appendChild(row);
    } else if(boss.lastKilled){
      const row=document.createElement('tr');
      const icon=boss.type==='Fixed'?'📌':'📊';
      row.innerHTML=`<td>${icon}</td><td>${name}</td><td>${nextSpawn?nextSpawn.toLocaleString('en-US',{timeZone:'Asia/Manila'}):'-'}</td><td>${remainingMs?formatTime(remainingMs):'-'}</td>`;
      boss.type==='Fixed'?fixedBody.appendChild(row):dynamicBody.appendChild(row);
    } else {
      const row=document.createElement('tr');
      row.innerHTML=`<td>🟡</td><td>${name}</td><td>${boss.type}</td>`;
      stillBody.appendChild(row);
    }
  });
  updateCurrentBoss();
}

// Update current boss
function updateCurrentBoss(){
  const nowPH=new Date();
  const respawning=Object.keys(bosses).find(name=>{
    const boss=bosses[name];
    const nextSpawn=getNextSpawn(boss);
    return nextSpawn && nextSpawn<=nowPH && nextSpawn>new Date(nowPH-1000*60*5);
  });
  const header=document.getElementById('current-boss');
  if(respawning){
    header.textContent=`Boss Respawning Now: ${respawning}`;
    header.classList.add('flashing');
  } else {
    header.textContent='No boss respawning now';
    header.classList.remove('flashing');
  }
}

setInterval(updateTables,1000);
</script>
</body>
</html>
