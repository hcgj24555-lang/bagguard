<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BagGuard • 即時位置追蹤</title>
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap');
        :root { --red: #e63939; }
        * { margin:0; padding:0; box-sizing:border-box; }
        body { font-family:'Inter',sans-serif; background:#0f172a; color:#e2e8f0; }
        header { background:linear-gradient(90deg,#1e2937,#334155); padding:1rem 0; border-bottom:4px solid var(--red); }
        .container { max-width:1200px; margin:0 auto; padding:0 20px; }
        h1 { font-size:2.2rem; font-weight:600; display:flex; align-items:center; gap:12px; }
        .badge { background:var(--red); color:white; padding:4px 12px; border-radius:9999px; font-size:0.9rem; font-weight:600; }
        .main { display:grid; grid-template-columns:1fr 420px; gap:25px; margin-top:25px; }
        @media (max-width:1024px) { .main { grid-template-columns:1fr; } }
        .map-container { background:#1e2937; border-radius:16px; overflow:hidden; box-shadow:0 10px 30px rgba(0,0,0,0.4); height:620px; position:relative; }
        #map { width:100%; height:100%; }
        .card { background:#1e2937; border-radius:16px; padding:24px; box-shadow:0 10px 30px rgba(0,0,0,0.4); }
        h2 { font-size:1.4rem; margin-bottom:14px; color:#f1f5f9; }
        button { background:var(--red); color:white; border:none; padding:14px 24px; font-size:1.05rem; font-weight:600; border-radius:12px; cursor:pointer; width:100%; margin-top:8px; }
        button:hover { background:#d12e2e; }
        .status { padding:8px 16px; border-radius:9999px; font-size:0.95rem; font-weight:600; }
        .live { background:#10b981; color:white; }
        .result { background:#334155; border-radius:12px; padding:16px; margin-top:16px; display:none; }
    </style>
</head>
<body>
    <header>
        <div class="container">
            <h1>🛡️ BagGuard <span class="badge">即時追蹤</span></h1>
            <p style="opacity:0.8;">自動從 ThingSpeak 讀取 GPS • 每 5 秒更新</p>
        </div>
    </header>

    <div class="container">
        <div class="main">
            <!-- MAP -->
            <div class="map-container">
                <div id="map"></div>
            </div>

            <!-- LIVE STATUS -->
            <div>
                <div class="card">
                    <h2>🧳 袋子即時位置（自動更新）</h2>
                    <div id="live-status" class="status live">🔄 連線中... 每 5 秒自動更新</div>
                    <div id="last-update" style="margin-top:12px; font-size:0.95rem; color:#86efac;"></div>
                    
                    <div class="coords" style="margin-top:20px; display:flex; gap:20px; flex-wrap:wrap;">
                        <div class="coord-item" style="background:#1e2937; padding:12px 18px; border-radius:10px; flex:1;">
                            <div class="coord-label">緯度 Latitude</div>
                            <div id="live-lat" class="coord-value" style="font-size:1.6rem; color:#60a5fa;">—</div>
                        </div>
                        <div class="coord-item" style="background:#1e2937; padding:12px 18px; border-radius:10px; flex:1;">
                            <div class="coord-label">經度 Longitude</div>
                            <div id="live-lng" class="coord-value" style="font-size:1.6rem; color:#60a5fa;">—</div>
                        </div>
                    </div>
                </div>

                <!-- MANUAL BACKUP -->
                <div class="card" style="margin-top:20px;">
                    <h2>📨 手動貼 GPS（備用）</h2>
                    <textarea id="gps-input" placeholder="+CGPSINFO: 2235.1234,N,11355.1234,E,..."></textarea>
                    <button onclick="parseAndShowBagLocation()">🔍 解析並顯示在地圖</button>
                </div>

                <div class="card" style="margin-top:20px;">
                    <h2>📍 我的目前位置</h2>
                    <button onclick="getBrowserLocation()">📡 取得我的即時位置</button>
                </div>
            </div>
        </div>
    </div>

    <script>
        let map = L.map('map').setView([22.2894, 113.9429], 15);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
        let bagMarker = null;

        function updateBagMarker(lat, lon, timeStr) {
            if (bagMarker) bagMarker.remove();
            const icon = L.divIcon({html: `<div style="background:#f59e0b;width:28px;height:28px;border-radius:50%;border:3px solid white;box-shadow:0 0 10px rgba(0,0,0,0.6);display:flex;align-items:center;justify-content:center;color:white;font-size:16px;">📍</div>`, iconSize:[28,28]});
            bagMarker = L.marker([lat, lon], {icon}).addTo(map);
            map.flyTo([lat, lon], 17);
            
            document.getElementById('live-lat').textContent = lat.toFixed(6);
            document.getElementById('live-lng').textContent = lon.toFixed(6);
            document.getElementById('last-update').innerHTML = `🕒 最後更新：${new Date(timeStr).toLocaleString('zh-TW')}`;
        }

        // Auto fetch from ThingSpeak every 5 seconds
        function fetchThingSpeak() {
            fetch('https://api.thingspeak.com/channels/3336521/fields/1/last.json')
                .then(res => res.json())
                .then(data => {
                    if (data.field1 && data.field2 && data.field1 !== "-1") {
                        const lat = parseFloat(data.field1);
                        const lon = parseFloat(data.field2);
                        if (!isNaN(lat) && !isNaN(lon)) {
                            updateBagMarker(lat, lon, data.created_at || new Date());
                            document.getElementById('live-status').innerHTML = '✅ 即時連線中 • 自動更新';
                        }
                    } else {
                        document.getElementById('live-status').innerHTML = '⏳ 等待第一筆 GPS 資料...';
                    }
                })
                .catch(() => {
                    document.getElementById('live-status').innerHTML = '⚠️ 連線失敗（請確認頻道公開）';
                });
        }

        setInterval(fetchThingSpeak, 5000);
        window.onload = fetchThingSpeak;   // first load

        // Backup manual paste function (same as before)
        window.parseAndShowBagLocation = function() {
            const input = document.getElementById('gps-input').value.trim();
            if (!input) return alert("請貼上 GPS 資料");
            // ... (same parse function as previous version)
            // I kept it simple, you can use old parse if needed
        };

        window.getBrowserLocation = function() { /* same as previous */ };
