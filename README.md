<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>BagGuard - 行李實時定位系統</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        :root { --primary: #60a5fa; --bg: #0f172a; --card: #1e293b; --accent: #f59e0b; }
        body { font-family: system-ui; background: var(--bg); color: white; margin: 0; padding: 20px; }
        .main { display: grid; grid-template-columns: 1fr 380px; gap: 20px; max-width: 1300px; margin: 0 auto; }
        @media (max-width: 900px) { .main { grid-template-columns: 1fr; } }
        #map { height: 650px; border-radius: 24px; border: 1px solid #334155; }
        .card { background: var(--card); padding: 25px; border-radius: 24px; border: 1px solid #334155; }
        .status { padding: 12px; border-radius: 12px; background: #334155; color: #94a3b8; text-align: center; font-weight: bold; margin-bottom: 20px; }
        .online { background: #064e3b; color: #4ade80; border: 1px solid #059669; }
        .coord-box { background: #0f172a; padding: 15px; border-radius: 15px; margin-bottom: 15px; border-left: 4px solid var(--primary); }
        .label { color: #94a3b8; font-size: 0.75rem; margin-bottom: 5px; }
        .value { font-family: monospace; font-size: 1.3rem; color: var(--primary); }
        .btn { display: block; width: 100%; padding: 15px; border-radius: 12px; border: none; font-weight: bold; cursor: pointer; text-align: center; text-decoration: none; margin-top: 10px; transition: 0.2s; }
        .btn-google { background: var(--accent); color: #0f172a; }
        .btn:hover { transform: translateY(-2px); opacity: 0.9; }
    </style>
</head>
<body>

<div class="main">
    <div id="map"></div>

    <div class="side">
        <div class="card">
            <h2 style="margin-top:0">🛡️ BagGuard 監控中</h2>
            <div id="live-status" class="status">🔄 同步數據中...</div>
            
            <div class="coord-box">
                <div class="label">緯度 LATITUDE</div>
                <div id="live-lat" class="value">—</div>
            </div>
            <div class="coord-box">
                <div class="label">經度 LONGITUDE</div>
                <div id="live-lng" class="value">—</div>
            </div>
            
            <p id="last-update" style="font-size: 0.85rem; color: #94a3b8; margin: 15px 0;"></p>
            
            <a id="google-link" href="#" target="_blank" class="btn btn-google" style="display:none">
                🌐 在 Google Maps 查看詳細地址
            </a>
            
            <button onclick="getBrowserLocation()" class="btn" style="background:#475569; color:white;">顯示我的位置</button>
        </div>
    </div>
</div>

<script>
    const CHANNEL_ID = "3336521"; // 你的 ThingSpeak 頻道
    let map = L.map('map').setView([22.2894, 113.9429], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
    
    let bagMarker = null;

    function fetchUpdate() {
        // 使用 feeds/last.json 同時獲取 field1(纬度) 和 field2(经度)
        fetch(`https://api.thingspeak.com/channels/${CHANNEL_ID}/feeds/last.json`)
            .then(res => res.json())
            .then(data => {
                const lat = parseFloat(data.field1);
                const lng = parseFloat(data.field2);
                
                if (!isNaN(lat) && !isNaN(lng)) {
                    // 更新地圖標記
                    if (bagMarker) bagMarker.remove();
                    bagMarker = L.marker([lat, lng]).addTo(map)
                        .bindPopup("<b>行李位置</b><br>最後同步: " + new Date(data.created_at).toLocaleTimeString())
                        .openPopup();
                    
                    map.panTo([lat, lng]);

                    // 更新 UI 文字
                    document.getElementById('live-lat').innerText = lat.toFixed(6);
                    document.getElementById('live-lng').innerText = lng.toFixed(6);
                    document.getElementById('last-update').innerText = "最後通訊: " + new Date(data.created_at).toLocaleString();
                    
                    const statusBox = document.getElementById('live-status');
                    statusBox.innerText = "✅ 設備在線";
                    statusBox.className = "status online";

                    // 更新 Google 地圖按鈕連結
                    const link = document.getElementById('google-link');
                    link.href = `https://www.google.com/maps?q=${lat},${lng}`;
                    link.style.display = "block";
                }
            })
            .catch(() => {
                document.getElementById('live-status').innerText = "⚠️ 數據連接異常";
            });
    }

    // 每 5 秒刷新一次
    setInterval(fetchUpdate, 5000);
    window.onload = fetchUpdate;

    function getBrowserLocation() {
        navigator.geolocation.getCurrentPosition(p => {
            L.circle([p.coords.latitude, p.coords.longitude], {radius: 80, color: 'red'}).addTo(map).bindPopup("我的位置");
        });
    }
</script>
</body>
</html>
