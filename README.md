<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BagGuard - 实时位置追踪</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        body { margin: 0; font-family: 'Segoe UI', sans-serif; background-color: #1a202c; color: white; display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
        header { background-color: #2d3748; padding: 15px; text-align: center; box-shadow: 0 4px 6px rgba(0,0,0,0.3); z-index: 100; }
        .container { display: flex; flex: 1; padding: 20px; gap: 20px; box-sizing: border-box; height: calc(100vh - 80px); }
        
        /* 修正点：确保地图容器有明确的宽高和背景色 */
        #map { 
            flex: 2; 
            background: #2d3748; 
            border-radius: 12px; 
            border: 2px solid #4a5568; 
            height: 100%; 
            min-height: 400px; /* 强制最小高度 */
            z-index: 1;
        }

        .sidebar { flex: 1; display: flex; flex-direction: column; gap: 20px; min-width: 300px; }
        .card { background-color: #2d3748; padding: 20px; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.3); }
        h2 { margin-top: 0; color: #63b3ed; font-size: 1.1rem; border-bottom: 1px solid #4a5568; padding-bottom: 10px; }
        .data-point { margin: 12px 0; font-size: 1rem; }
        .status-online { color: #48bb78; font-weight: bold; }
        .footer { text-align: center; padding: 5px; font-size: 0.7rem; color: #718096; }
    </style>
</head>
<body>

<header>
    <h1 style="margin:0; font-size: 1.5rem;">🛡️ BagGuard 实时追踪系统</h1>
    <p style="margin:5px 0 0; color: #a0aec0; font-size: 0.9rem;">自动从 ThingSpeak 读取数据 • 每 15 秒同步</p>
</header>

<div class="container">
    <div id="map"></div>

    <div class="sidebar">
        <div class="card">
            <h2>📍 设备状态</h2>
            <div class="data-point">状态: <span id="status" class="status-online">正在连接...</span></div>
            <div class="data-point">纬度: <span id="lat" style="color:#f6ad55">---</span></div>
            <div class="data-point">经度: <span id="lon" style="color:#f6ad55">---</span></div>
            <div class="data-point">更新: <span id="time" style="font-size: 0.8rem; color:#cbd5e0">---</span></div>
        </div>

        <div class="card">
            <h2>🛠️ 手动解析 (备用)</h2>
            <textarea id="manualInput" style="width: 100%; height: 50px; background: #1a202c; color: #cbd5e0; border: 1px solid #4a5568; border-radius: 5px; padding: 5px;" placeholder="粘贴 AT 指令返回内容..."></textarea>
            <button onclick="parseManual()" style="width: 100%; margin-top: 10px; padding: 10px; background: #e53e3e; color: white; border: none; border-radius: 5px; cursor: pointer; font-weight: bold;">解析并在地图显示</button>
        </div>
    </div>
</div>

<div class="footer">Powered by ESP32 + A7670X + Leaflet.js</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
    // --- 请在此处修改你的 API 信息 ---
    const channelID = "3336521"; // 示例 ID
    const readAPIKey = "M3EDCYNO6BNOICSA"; // 你的 Read API Key
    // ----------------------------

    var map, marker;

    // 初始化地图
    function initMap() {
        // 初始坐标设为香港 (根据你的截图)
        map = L.map('map').setView([22.3193, 114.1694], 15);
        
        // 使用 OpenStreetMap 图层
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            attribution: '© OpenStreetMap'
        }).addTo(map);

        marker = L.marker([22.3193, 114.1694]).addTo(map);
        
        // 关键步骤：解决容器大小初始化不准确的问题
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
                document.getElementById('status').innerText = "读取失败";
                document.getElementById('status').style.color = "#fc8181";
            });
    }

    function updateUI(lat, lon, time) {
        document.getElementById('lat').innerText = lat.toFixed(6);
        document.getElementById('lon').innerText = lon.toFixed(6);
        document.getElementById('time').innerText = time;
        document.getElementById('status').innerText = "在线";

        const pos = [lat, lon];
        marker.setLatLng(pos);
        map.panTo(pos);
        marker.bindPopup("最后更新时间:<br>" + time).openPopup();
    }

    function parseManual() {
        const input = document.getElementById('manualInput').value;
        const regex = /([0-9]+\.[0-9]+)/g;
        const matches = input.match(regex);
        if (matches && matches.length >= 2) {
            // 注意：ATD指令中的坐标格式可能需要转换，这里做简单处理
            updateUI(parseFloat(matches[0]), parseFloat(matches[1]), "手动触发");
        }
    }

    // 页面加载后初始化
    window.onload = () => {
        initMap();
        fetchData();
        setInterval(fetchData, 15000); // 每15秒同步一次
    };
</script>
</body>
</html>
