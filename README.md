<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>EchoSOS — Real-Time Safety System</title>

<!-- Fonts -->
<link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@600;700&family=Poppins:wght@300;500&display=swap" rel="stylesheet">

<!-- Leaflet -->
<link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>

<style>
  :root{
    --neon:#00f6ff;
    --accent:#00b4d8;
    --danger:#ff3b30;
    --bg1:#020617;
    --bg2:#071428;
  }
  *{box-sizing:border-box}
  body{
    margin:0;
    font-family: 'Poppins', sans-serif;
    background: linear-gradient(135deg,var(--bg1),var(--bg2));
    color:#e6f7ff;
  }
  header{
    padding:24px;
    text-align:center;
    border-bottom: 1px solid rgba(0,180,216,0.06);
  }
  header h1{
    margin:0;
    font-family:'Orbitron',sans-serif;
    font-weight:700;
    color:var(--neon);
    letter-spacing:1px;
  }
  header p { margin:6px 0 0; color: #bfefff; opacity:0.9; }

  .wrap{ display:flex; gap:24px; padding:20px; align-items:flex-start; justify-content:center; flex-wrap:wrap; }
  .left{ width:66%; min-width:360px; }
  .right{ width:30%; min-width:300px; }

  @media (max-width:768px){
    .wrap{ flex-direction:column; align-items:center; }
    .left, .right{ width:100%; }
    #map{ height:50vh; }
  }

  #map{
    width:100%;
    height:62vh;
    border-radius:14px;
    border:2px solid rgba(0,180,216,0.12);
    box-shadow: 0 8px 40px rgba(0,0,0,0.6);
    overflow:hidden;
  }

  .controls{
    display:flex; gap:12px; margin-top:16px; flex-wrap:wrap; justify-content:center;
  }

  .btn{
    background:linear-gradient(45deg,var(--accent),var(--neon));
    color:#001;
    border:none;
    padding:12px 18px;
    border-radius:999px;
    font-weight:700;
    cursor:pointer;
    transition:transform .23s ease, box-shadow .23s ease;
  }
  .btn:hover{ transform:translateY(-4px); }

  .btn--sos{
    background: linear-gradient(45deg, var(--danger), #ff6b6b);
    color:#fff;
    animation: neonPulse 2s infinite;
  }
  @keyframes neonPulse{
    0% { box-shadow: 0 0 8px rgba(255,60,60,0.14); transform:scale(1); }
    50%{ box-shadow: 0 0 28px rgba(255,60,60,0.26); transform:scale(1.04); }
    100%{ box-shadow: 0 0 8px rgba(255,60,60,0.14); transform:scale(1); }
  }

  .panel{
    background: rgba(255,255,255,0.02);
    padding:18px;
    border-radius:12px;
    border:1px solid rgba(0,180,216,0.06);
    box-shadow: 0 8px 30px rgba(0,0,0,0.5);
  }
  .panel h3{ margin:6px 0 12px; color:var(--neon); font-family:'Orbitron',sans-serif; }

  .alert-list{ max-height:36vh; overflow:auto; margin-top:12px; padding-right:6px; }
  .alert-item{ background: rgba(0,0,0,0.2); border-radius:8px; padding:10px; margin-bottom:8px; border-left:4px solid rgba(0,180,216,0.4); }
  .danger-item{ border-left-color: var(--danger); }

  footer{ text-align:center; opacity:0.6; margin:18px 0; font-size:13px; }

  /* RED MARKER */
  .blink-marker{
    width:18px;height:18px;
    background:red;
    border-radius:50%;
    box-shadow:0 0 10px rgba(255,0,0,0.8);
    animation:blinkPulse 1s infinite;
  }
  @keyframes blinkPulse{
    0%{transform:scale(1);box-shadow:0 0 8px rgba(255,0,0,0.6);}
    50%{transform:scale(1.4);box-shadow:0 0 16px rgba(255,0,0,1);}
    100%{transform:scale(1);box-shadow:0 0 8px rgba(255,0,0,0.6);}
  }
</style>
</head>
<body>

<header>
  <h1>EchoSOS</h1>
  <p>Real-Time Location-Based Safety System</p>
</header>

<div class="wrap">
  <div class="left">
    <div id="map"></div>

    <div class="controls">
      <button class="btn btn--sos" onclick="triggerEmergency()">🚨 EMERGENCY</button>
      <button class="btn" onclick="showMyLocation()">📍 My Location</button>
      <button class="btn" onclick="voiceAlert()">🗣️ Voice Alert</button>
      <button class="btn" onclick="triggerFakeCall()">📞 Fake Call</button>
    </div>
  </div>

  <div class="right">
    <div class="panel">
      <h3>Alert Log</h3>
      <div id="alerts" class="alert-list">
        <div id="empty-msg" style="opacity:0.6">No alerts yet...</div>
      </div>
    </div>
  </div>
</div>

<footer>EchoSOS Prototype — 2025</footer>

<audio id="alarm" src="https://actions.google.com/sounds/v1/alarms/alarm_clock.ogg"></audio>

<script>
var map = L.map('map').setView([10.732, 106.721], 14);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19 }).addTo(map);

let userMarker;

/* Reverse Geocoding */
async function getAddress(lat,lng){
  try{
    const res = await fetch(`https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`);
    const data = await res.json();
    return data.display_name || "Address unavailable";
  }catch{
    return "Address lookup failed";
  }
}

/* Alert Log */
function logAlert(text,danger=false){
  const empty = document.getElementById("empty-msg");
  if(empty) empty.remove();
  const box=document.getElementById("alerts");
  const item=document.createElement("div");
  item.className="alert-item"+(danger?" danger-item":"");
  item.innerHTML=text;
  box.prepend(item);
}

/* Safe Zones */
var safeZones = [
  {lat:10.728, lng:106.710}, {lat:10.740, lng:106.720}, {lat:10.730, lng:106.735},
  {lat:10.725, lng:106.740}, {lat:10.745, lng:106.715}, {lat:10.735, lng:106.725},
  {lat:10.738, lng:106.730}, {lat:10.742, lng:106.722}, {lat:10.733, lng:106.718},
  {lat:10.747, lng:106.728}, {lat:10.729, lng:106.732}
];
safeZones.forEach(async z=>{
  const adr = await getAddress(z.lat,z.lng);
  L.circle([z.lat,z.lng],{color:"green",fillColor:"#69ff69",fillOpacity:0.4,radius:220}).addTo(map).bindPopup("SAFE ZONE:<br>"+adr);
});

/* Danger Zones */
var dangerZones = [
  {lat:10.732, lng:106.712}, {lat:10.748, lng:106.735}, {lat:10.737, lng:106.720},
  {lat:10.740, lng:106.725}, {lat:10.726, lng:106.738}, {lat:10.743, lng:106.730},
  {lat:10.735, lng:106.742}, {lat:10.739, lng:106.718}, {lat:10.744, lng:106.723},
  {lat:10.731, lng:106.727}, {lat:10.746, lng:106.729}
];
dangerZones.forEach(async d=>{
  const adr = await getAddress(d.lat,d.lng);
  L.circle([d.lat,d.lng],{color:"red",fillColor:"#ff6969",fillOpacity:0.4,radius:220}).addTo(map).bindPopup("DANGER ZONE:<br>"+adr);
});

/* Emergency */
async function triggerEmergency(){
  document.getElementById("alarm").play();
  navigator.vibrate?.([500,300,500]);
  navigator.geolocation.getCurrentPosition(async pos=>{
    const lat=pos.coords.latitude,lng=pos.coords.longitude;
    const adr=await getAddress(lat,lng);
    logAlert("🚨 EMERGENCY triggered at:<br>"+adr,true);
u.lang = 'vi-VN';
  u.volume = 100;
  u.rate = 1;
  u.pitch = 1.4; //
  speechSynthesis.speak(u)
  });
}

/* My Location */
async function showMyLocation(){
  navigator.geolocation.getCurrentPosition(async pos=>{
    const lat=pos.coords.latitude,lng=pos.coords.longitude;
    const adr=await getAddress(lat,lng);
    if(userMarker) map.removeLayer(userMarker);
    userMarker = L.marker([lat,lng]).addTo(map).bindPopup("You are here").openPopup();
    map.setView([lat,lng],16);
    logAlert("📍 My Location:<br>"+adr);
  });
}

/* Voice Alert */
function voiceAlert(){
  const u = new SpeechSynthesisUtterance("Help me!!! Giúp đỡ! Xin hãy cứu tôi! Help me!  Giúp đỡ! Xin hãy cứu tôi!  Help me! Giúp đỡ! Xin hãy cứu tôi!");
  u.lang = 'vi-VN';
  u.volume = 1.0;
  u.rate = 1;
  u.pitch = 1.4; //
  speechSynthesis.speak(u);
}

/* Fake Call */
function triggerFakeCall(){
  //
  const callSound = new Audio("https://actions.google.com/sounds/v1/alarms/phone_alerts_and_rings.ogg");
  callSound.volume = 1.0; // 
  callSound.addEventListener('loadedmetadata', () => {
    callSound.currentTime = 14; //
    callSound.play();
const duration = 20 - 14; //
    setTimeout(() => {
      callSound.pause();
      callSound.currentTime = 0; // 
    }, duration * 1000);
  });
}
</script>

</body>
</html>
