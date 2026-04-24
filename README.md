<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BagGuard - LBS 实时定位 (大范围模式)</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        body { margin: 0; font-family: 'Segoe UI', sans-serif; background-color: #1a202c; color: white; display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
        header { background-color: #2d3748; padding: 15px; text-align: center; box-shadow: 0 4px 6px rgba(0,0,0,0.3); z-index: 100; }
        .container { display: flex; flex: 1; padding: 20px; gap: 20px; box-sizing: border-box; height: calc(100vh - 80px); }
        #map { flex: 2; background: #2d3748; border-radius: 12px; border: 2px solid #4a5568; height: 100%; z-index: 1; }
        .sidebar { flex: 1; display: flex; flex-direction: column; gap: 20px; min-width: 300px; }
        .card { background-color: #2d3748; padding: 20px; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.3); }
        h2 { margin-top: 0; color: #63b3ed; font-size: 1.1rem; border-bottom: 1px solid #4a5568; padding-bottom: 10px; }
        .data-point { margin: 12px 0; font-size: 1rem; }
        .hint-text { font-size: 0.8rem; color: #a0aec0; font-style: italic; margin-top: 10px; line-height: 1.4; }
        .status-online { color: #48bb78; font-weight: bold; }
        .footer { text-align: center; padding: 5px; font-size: 0.7rem; color: #718096; }
    </style>
</head>
<body>

<header>
    <h1 style="margin:0; font-size: 1.5rem;">🛡️ BagGuard 实时追踪系统</h1>
    <p style="margin:5px 0 0; color: #a0aec0; font-size: 0.9rem;">基站定位 (LBS) 大范围模式</p>
</header>

<div class="container">
    <div id="map"></div>

    <div class="sidebar">
        <div class="card">
            <h2>📍 设备状态</h2>
            <div class="data-point">状态: <span id="status" class="status-online">正在连接...</span></div>
            <div class="data-point">纬度: <span id="lat" style="color:#f6ad55">---</span></div>
            <div class="data-point">经度: <span id="lon" style="color:#f6ad55">---</span></div>
            <div class="data-point">搜索半径: <span id="accText" style="color:#63b3ed">---</span></div>
            <div class="data-point">更新: <span id="time" style="font-size: 0.8rem; color:#cbd5e0">---</span></div>
            
            <p class="hint-text">
                ⚠️ <b>提示：</b><br>
                当前使用 LBS 基站定位，圆圈代表设备可能存在的范围（通常误差为 500-1500米）。
            </p>
        </div>

        <div class="card">
            <h2>🛠️ 手动调试</h2>
            <textarea id="manualInput" style="width: 100%; height: 50px; background: #1a202c; color: #cbd5e0; border: 1px solid #4a5568; border-radius: 5px; padding: 5px;" placeholder="粘贴 AT 指令返回内容..."></textarea>
            <button onclick="parseManual()" style="width: 100%; margin-top: 10px; padding: 10px; background: #e53e3e; color: white; border: none; border-radius: 5px; cursor: pointer; font-weight: bold;">立即在地图显示</button>
        </div>
    </div>
</div>

<div class="footer">Powered by ESP32 + A7670X + Leaflet.js</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
    // --- 配置区 ---
    // 请替换为你自己的 ThingSpeak Channel ID 和 Read API Key
    const channelID = "3336521"; 
    const readAPIKey = "M3EDCYNO6BNOICSA";
    
    // 【修改点：设置默认的大半径 (单位：米)】
    // LBS 误差通常在 500m-2000m。我们将默认半径设大，例如 1500米 (1.5公里)。
    const LBS_DEFAULT_RADIUS = 1500; 
    // --------------

    var map, marker, accuracyCircle;

    function initMap() {
        // 由于圆圈变大了，初始缩放层级从 14 改为 13，以便看到完整的圆。
        map = L.map('map').setView([22.3193, 114.1694], 13);
        
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 18,
            attribution: '© OpenStreetMap'
        }).addTo(map);

        marker = L.marker([22.3193, 114.1694]).addTo(map);

        // 【新增/修改】初始化更粗、更明显的透明圆圈
        accuracyCircle = L.circle([22.3193, 114.1694], {
            color: '#3182ce',      // 边框颜色稍深一些
            weight: 2,             // 边框粗细
            fillColor: '#63b3ed',  // 填充颜色
            fillOpacity: 0.15,     // 稍微降低透明度，避免大范围覆盖遮挡地图细节
            radius: LBS_DEFAULT_RADIUS // 使用设定的大半径
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
                    
                    // 如果你不在 ThingSpeak 上传精度，就强制使用设定的大半径
                    // const acc = feed.field3 ? parseFloat(feed.field3) : LBS_DEFAULT_RADIUS; 
                    const acc = LBS_DEFAULT_RADIUS; // 强制大半径

                    const time = new Date(feed.created_at).toLocaleString('zh-CN');

                    if (!isNaN(lat) && !isNaN(lon)) {
                        updateUI(lat, lon, time, acc);
                    }
                }
            })
            .catch(err => console.log("数据获取失败"));
    }

    function updateUI(lat, lon, time, acc) {
        document.getElementById('lat').innerText = lat.toFixed(6);
        document.getElementById('lon').innerText = lon.toFixed(6);
        document.getElementById('accText').innerText = (acc/1000).toFixed(1) + " 公里";
        document.getElementById('time').innerText = time;
        document.getElementById('status').innerText = "在线";

        const pos = [lat, lon];
        
        marker.setLatLng(pos);
        accuracyCircle.setLatLng(pos);
        accuracyCircle.setRadius(acc); // 确保半径正确更新
        
        map.panTo(pos);
        marker.bindPopup("<b>搜索区域中心</b><br>更新时间: " + time).openPopup();
    }

    function parseManual() {
        const input = document.getElementById('manualInput').value;
        const regex = /([0-9]+\.[0-9]+)/g;
        const matches = input.match(regex);
        if (matches && matches.length >= 2) {
            updateUI(parseFloat(matches[0]), parseFloat(matches[1]), "手动调试", LBS_DEFAULT_RADIUS);
        }
    }

    window.onload = () => {
        initMap();
        fetchData();
        setInterval(fetchData, 15000); // 每 15 秒同步一次
    };
</script>
</body>
</html>
