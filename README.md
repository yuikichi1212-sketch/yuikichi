<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>リアルタイム位置共有（Room）</title>
  
  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <style>
    html, body, #map { height: 100%; margin:0; padding:0; }
    body { font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Hiragino Kaku Gothic ProN','Noto Sans JP', sans-serif; }
    .ui { position: absolute; z-index:1000; left:12px; top:12px; background:rgba(255,255,255,0.95); padding:12px; border-radius:8px; box-shadow:0 4px 20px rgba(0,0,0,0.12); width:320px; }
    .ui h2 { margin:0 0 8px 0; font-size:16px; }
    .ui label { display:block; margin-top:8px; font-size:13px; }
    .ui input[type=text] { width:100%; padding:8px; border-radius:6px; border:1px solid #ccc; }
    .ui button { margin-top:8px; width:100%; padding:8px; border-radius:6px; border:1px solid #ddd; background:#f7f7f7; cursor:pointer; }
    .small { font-size:12px; color:#555; margin-top:6px; }
  </style>
</head>
<body>
  <div id="map"></div>

  <div class="ui" id="ui">
    <h2>ルームで位置共有（5秒更新）</h2>
    <label>表示名（任意）</label>
    <input id="displayName" type="text" placeholder="例: ゆいきち" />

    <label>ルーム名</label>
    <input id="roomInput" type="text" placeholder="例: friends" />

    <button id="joinBtn">ルームに参加</button>
    <button id="leaveBtn" disabled>退出</button>

    <div class="small" id="status">未参加</div>
  </div>

  <!-- Leaflet JS -->
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

  <!-- Firebase SDK（モジュール版不要で簡易読み込み） -->
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-database-compat.js"></script>

  <script>
    // Firebase config
    const firebaseConfig = {
      apiKey: "AIzaSyC5gLzMBuoHCO9_CgqnUTOLAC1BIzZb7Sw",
      authDomain: "location-share-bc69b.firebaseapp.com",
      databaseURL: "https://location-share-bc69b-default-rtdb.asia-southeast1.firebasedatabase.app",
      projectId: "location-share-bc69b",
      storageBucket: "location-share-bc69b.firebasestorage.app",
      messagingSenderId: "589367892503",
      appId: "1:589367892503:web:a37b7cc6b4edd96b65da52",
      measurementId: "G-QVZN8MS60K"
    };
    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();

    // Leaflet map
    const map = L.map('map').setView([35.68,139.76], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{
      maxZoom:19, attribution:'© OpenStreetMap contributors'
    }).addTo(map);

    // UI elements
    const joinBtn = document.getElementById('joinBtn');
    const leaveBtn = document.getElementById('leaveBtn');
    const roomInput = document.getElementById('roomInput');
    const displayNameInput = document.getElementById('displayName');
    const statusEl = document.getElementById('status');

    let clientId = 'c_' + Math.random().toString(36).slice(2,9);
    let roomName = null;
    let sendInterval = null;
    let markers = {};
    let firstPositionSet = false;

    function setStatus(t){ statusEl.textContent = t; }

    function startSharing(){
      if(!roomName) return alert('先にルーム名を入れてください');
      if(!navigator.geolocation) return alert('このブラウザは Geolocation に未対応です');

      // 初回位置取得して自動センター
      navigator.geolocation.getCurrentPosition(pos=>{
        const lat = pos.coords.latitude, lon = pos.coords.longitude;
        if(!firstPositionSet){ map.setView([lat,lon],15); firstPositionSet=true; }
      }, err=>console.warn('初回位置取得失敗',err), { enableHighAccuracy:true, maximumAge:5000, timeout:10000 });

      sendMyPos();
      sendInterval = setInterval(sendMyPos,5000);

      const roomRef = db.ref(`rooms/${roomName}`);
      roomRef.on('value', snapshot=>{
        const data = snapshot.val() || {};
        const now = Date.now();
        // 消えた人のマーカー削除
        const existingIds = new Set(Object.keys(data));
        for(const id of Object.keys(markers)){
          if(!existingIds.has(id)){ map.removeLayer(markers[id]); delete markers[id]; }
        }
        for(const [id,obj] of Object.entries(data)){
          if(!obj || !obj.lat || !obj.lon) continue;
          if(obj.ts && (now-obj.ts)>30000) continue;
          if(!markers[id]){
            markers[id] = L.marker([obj.lat,obj.lon]).addTo(map).bindPopup(obj.name||id);
          }else{
            markers[id].setLatLng([obj.lat,obj.lon]);
            if(markers[id].getPopup()) markers[id].getPopup().setContent(obj.name||id);
          }
        }
      });

      setStatus(`参加中 — ルーム: ${roomName}（5秒更新）`);
      joinBtn.disabled=true; leaveBtn.disabled=false; roomInput.disabled=true; displayNameInput.disabled=true;
    }

    function stopSharing(){
      if(sendInterval){ clearInterval(sendInterval); sendInterval=null; }
      if(roomName){
        db.ref(`rooms/${roomName}/${clientId}`).remove().catch(()=>{});
      }
      for(const id of Object.keys(markers)){ map.removeLayer(markers[id]); }
      markers={};
      setStatus('未参加');
      joinBtn.disabled=false; leaveBtn.disabled=true; roomInput.disabled=false; displayNameInput.disabled=false;
      roomName=null;
    }

    function sendMyPos(){
      if(!roomName) return;
      navigator.geolocation.getCurrentPosition(pos=>{
        const lat = pos.coords.latitude, lon = pos.coords.longitude;
        const name = (displayNameInput.value || '名無し');
        db.ref(`rooms/${roomName}/${clientId}`).set({ lat, lon, name, ts:Date.now() }).catch(err=>console.warn('書き込み失敗',err));
      }, err=>console.warn('位置取得エラー',err), { enableHighAccuracy:true, maximumAge:3000, timeout:10000 });
    }

    joinBtn.addEventListener('click', ()=>{
      const r = (roomInput.value||'').trim();
      if(!r) return alert('ルーム名を入力してください');
      roomName=r; startSharing();
    });
    leaveBtn.addEventListener('click', ()=>{
      if(!confirm('退出しますか？')) return;
      stopSharing();
    });
    window.addEventListener('beforeunload', ()=>{
      if(roomName) db.ref(`rooms/${roomName}/${clientId}`).remove().catch(()=>{});
    });
  </script>
</body>
</html>
