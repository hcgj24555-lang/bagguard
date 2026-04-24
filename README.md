<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>BagGuard - 行李安全衛士</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        :root { --primary: #60a5fa; --bg: #0f172a; --card: #1e293b; }
        body { font-family: system-ui; background: var(--bg); color: white; margin: 0; padding: 20px; }
        .main { display: grid; grid-template-columns: 1fr 350px; gap: 20px; max-width: 1200px; margin: 0 auto; }
        #map { height: 600px; border-radius: 20px; border: 1px solid #334155; }
        .card { background: var(--card); padding: 20px; border-radius: 20px; border: 1px solid #334155; }
        .status { padding: 10px; border-radius: 10px; background: #064e3b; color: #4ade80; text-align: center; font-weight: bold; }
        .coord-box { background: #0f172a; padding: 15px; border-radius: 10px; margin-top: 10px; }
        .label { color: #94a3b8; font-size: 0.8rem; }
        .value { font-family: monospace; font-size: 1.2rem; color: var(--primary); }
    </style>
</head>
<body>

<div class="main">
    <div id="map"></div>
    <div class="side">
        <div class="card">
            <h2 style="margin-top:0">🛡️ BagGuard 狀態</h2>
            <div id="live-status" class="status">🔄 正在同步數據...</div>
            
            <div class="coord-box">
                <div class="label">緯度 Latitude (Field 1)</div>
                <div id="live-lat" class="value">—</div>
            </div>
            <div class="coord-box">
                <div class="label">經度 Longitude (Field 2)</div>
                <div id="live-lng" class="value">—</div>
            </div>
            
            <p id="last-update" style="font-size: 0.8rem; color: #94a3b8; margin-top: 20px;"></p>
            <button onclick="getBrowserLocation()" style="width:100%; padding:10px; border-radius:10px; cursor:pointer;">顯示我的位置</button>
        </div>
    </div>
</div>

<script>
    const CHANNEL_ID = "3336521"; // 你的 Channel ID
    let map = L.map('map').setView([22.2894, 113.9429], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
    let bagMarker = null;

    function fetchUpdate() {
        // 关键点：使用 feeds/last.json 获取包含所有 Field 的最新记录
        fetch(`https://api.thingspeak.com/channels/${CHANNEL_ID}/feeds/last.json`)
            .then(res => res.json())
            .then(data => {
                const lat = parseFloat(data.field1);
                const lng = parseFloat(data.field2);
                
                if (!isNaN(lat) && !isNaN(lng)) {
                    if (bagMarker) bagMarker.remove();
                    bagMarker = L.marker([lat, lng]).addTo(map)
                        .bindPopup("行李在此處").openPopup();
                    
                    map.panTo([lat, lng]);
                    
                    document.getElementById('live-lat').innerText = lat.toFixed(6);
                    document.getElementById('live-lng').innerText = lng.toFixed(6);
                    document.getElementById('last-update').innerText = "最後同步: " + new Date(data.created_at).toLocaleString();
                    document.getElementById('live-status').innerText = "✅ 設備在線";
                }
            })
            .catch(() => {
                document.getElementById('live-status').innerText = "⚠️ 連線失敗";
            });
    }

    setInterval(fetchUpdate, 5000);
    window.onload = fetchUpdate;

    function getBrowserLocation() {
        navigator.geolocation.getCurrentPosition(p => {
            L.circle([p.coords.latitude, p.coords.longitude], {radius: 50, color: 'red'}).addTo(map).bindPopup("我的位置");
        });
    }
</script>
</body>
</html>
