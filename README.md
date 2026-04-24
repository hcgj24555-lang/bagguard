<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BagGuard - 1km 范围追踪</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        body { margin: 0; font-family: 'Segoe UI', sans-serif; background-color: #1a202c; color: white; display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
        header { background-color: #2d3748; padding: 15px; text-align: center; box-shadow: 0 4px 6px rgba(0,0,0,0.3); z-index: 100; border-bottom: 3px solid #3182ce; }
        .container { display: flex; flex: 1; padding: 20px; gap: 20px; box-sizing: border-box; height: calc(100vh - 80px); }
        #map { flex: 2; background: #2d3748; border-radius: 15px; border: 2px solid #4a5568; height: 100%; z-index: 1; }
        .sidebar { flex: 1; display: flex; flex-direction: column; gap: 20px; min-width: 320px; }
        .card { background-color: #2d3748; padding: 20px; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.3); border: 1px solid #4a5568; }
        h2 { margin-top: 0; color: #63b3ed; font-size: 1.1rem; border-bottom: 1px solid #4a5568; padding-bottom: 10px; display: flex; justify-content: space-between; }
        .data-point { margin: 12px 0; font-size: 1.05rem; display: flex; justify-content: space-between; }
        .hint-text { font-size: 0.8rem; color: #a0aec0; font-style: italic; margin-top: 15px; line-height: 1.5; background: #1a202c; padding: 10px; border-radius: 8px; }
        .status-online { color: #48bb78; font-weight: bold; animation: blink 2s infinite; }
        @keyframes blink { 0% { opacity: 1; } 50% { opacity: 0.5; } 100% { opacity: 1; } }
        .footer { text-align: center; padding: 5px; font-size: 0.7rem; color: #718096; }
    </style>
</head>
<body>

<header>
    <h1 style="margin:0; font-size: 1.6rem;">🛡️ BagGuard 实时追踪</h1>
    <p style="margin:5px 0 0; color: #a0aec0; font-size: 0.9rem;">基站定位 (LBS) • 1公里模糊搜索模式</p>
</header>

<div class="container">
    <div id="map"></div>

    <div class="sidebar">
        <div class="card">
            <h2>📍 设备状态 <span id="status" class="status-online">在线</span></h2>
            <div class="data-point"><span>经度:</span> <span id="lon" style="color:#f6ad55">---</span></div>
            <div class="data-point"><span>纬度:</span> <span id="lat" style="color:#f6ad55">---</span></div>
            <div class="data-point"><span>搜索半径:</span> <span id="accText" style="color:#63b3ed">1.0 公里</span></div>
            <div class="data-point"><span>最后同步:</span> <span id="time" style="font-size: 0.85rem; color:#cbd5e0">---</span></div>
            
            <div class="hint-text">
                蓝色圆圈代表 1 公里的搜索范围。由于使用的是基站信号而非 GPS，设备位置可能偏离中心点。
            </div>
        </div>

        <div class="card">
            <h2>🛠️ 原始数据解析</h2>
            <textarea id="manualInput" style="width: 100%; height: 50px; background: #1a202c; color: #cbd5e0; border: 1px solid #4a5568; border-radius: 5px; padding: 8px; font-family: monospace; resize: none;" placeholder="粘贴 +CLBS 或 +CGPSINFO 数据..."></textarea>
            <button onclick="parseManual()" style="width: 100%; margin-top: 12px; padding: 12px; background: #3182ce; color: white; border: none; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.3s;">更新地图位置</button>
        </div>
    </div>
</div>

<div class="footer">ESP32 + A7670X Module | Leaflet Map Engine</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
    // ================= 配置区 =================
    const channelID = "3336521"; 
    const readAPIKey = "M3EDCYNO6BNOICSA";
    const LBS_RADIUS = 1000; // 设定为 1 公里 (1000米)
    // ==========================================

    var map, marker, accuracyCircle;

    function initMap() {
        // 设置缩放层级为 14，刚好可以看到 1km 的圆圈范围
        map = L.map('map').setView([22.3193, 114.1694], 14);
        
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 18,
            attribution: '© OpenStreetMap'
        }).addTo(map);

        // 初始化标记
        marker = L.marker([22.3193, 114.1694]).addTo(map);

        // 初始化 1km 蓝色覆盖层
        accuracyCircle = L.circle([22.3193, 114.1694], {
            color: '#3182ce',      // 边框深蓝色
            weight: 2,             // 边框粗细
            fillColor: '#3182ce',  // 填充蓝色
            fillOpacity: 0.2,      // 20% 透明度
            radius: LBS_RADIUS     // 1000米
        }).addTo(map);
        
        setTimeout(() => { map.invalidateSize(); }, 500);
    }

    function fetchData() {
        const url = `https://api.thingspeak.com/channels/${channelID}/feeds.json?api_key=${readAPIKey}&results=1`;

        fetch(url)
            .then(res => res.json())
            .then(data => {
                if (data.feeds && data.feeds.length > 0) {
                    const feed = data.feeds[0];
                    const lat = parseFloat(feed.field1);
                    const lon = parseFloat(feed.field2);
                    const time = new Date(feed.created_at).toLocaleString('zh-CN');

                    if (!isNaN(lat) && !isNaN(lon)) {
                        updateUI(lat, lon, time);
                    }
                }
            })
            .catch(err => {
                console.error("Fetch Error");
                document.getElementById('status').innerText = "离线";
                document.getElementById('status').
