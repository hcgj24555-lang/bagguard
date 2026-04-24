<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BagGuard - 实时位置追踪</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        body { margin: 0; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #1a202c; color: white; display: flex; flex-direction: column; height: 100vh; }
        header { background-color: #2d3748; padding: 20px; text-align: center; box-shadow: 0 2px 10px rgba(0,0,0,0.5); }
        .container { display: flex; flex: 1; padding: 20px; gap: 20px; }
        #map { flex: 2; border-radius: 12px; border: 2px solid #4a5568; height: 100%; }
        .sidebar { flex: 1; display: flex; flex-direction: column; gap: 20px; }
        .card { background-color: #2d3748; padding: 20px; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.3); }
        h2 { margin-top: 0; color: #63b3ed; font-size: 1.2rem; }
        .data-point { margin: 10px 0; font-size: 1.1rem; }
        .status { color: #48bb78; font-weight: bold; }
        .footer { text-align: center; padding: 10px; font-size: 0.8rem; color: #a0aec0; }
    </style>
</head>
<body>

<header>
    <h1>🛡️ BagGuard 实时追踪系统</h1>
    <p>自动从 ThingSpeak 读取数据 • 每 15 秒更新一次</p>
</header>

<div class="container">
    <div id="map"></div>

    <div class="sidebar">
        <div class="card">
            <h2>📍 设备状态</h2>
            <div class="data-point">状态: <span id="status" class="status">正在连接...</span></div>
            <div class="data-point">纬度: <span id="lat">---</span></div>
            <div class="data-point">经度: <span id="lon">---</span></div>
            <div class="data-point">最后更新: <span id="time">---</span></div>
        </div>

        <div class="card">
            <h2>🛠️ 手动解析 (备用)</h2>
            <textarea id="manualInput" style="width: 100%; height: 60px; background: #1a202c; color: white; border: 1px solid #4a5568; border-radius: 5px;" placeholder="+CGPSINFO: 2235.1234,N,11355.1234,E..."></textarea>
            <button onclick="parseManual()" style="width: 100%; margin-top: 10px; padding: 10px; background: #e53e3e; color: white; border: none; border-radius: 5px; cursor: pointer;">解析并显示在地图</button>
        </div>
    </div>
</div>

<div class="footer">Powered by ESP32 + A7670X + Leaflet.js</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
    // --- 配置区 ---
    const channelID = "3336521"; 
    const readAPIKey = "M3EDCYNO6BNOICSA";
    // --------------

    // 初始化地图 (默认定位到香港/深圳附近)
    var map = L.map('map').setView([22.3193, 114.1694], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '© OpenStreetMap contributors'
    }).addTo(map);

    var marker = L.marker([22.3193, 114.1694]).addTo(map);

    // 从 ThingSpeak 获取数据
    function fetchData() {
        const url = `https://api.thingspeak.com/channels/${channelID}/feeds.json?api_key=${readAPIKey}&results=1`;

        fetch(url)
            .then(response => response.json())
            .then(data => {
                const feed = data.feeds[0];
                if (feed) {
                    const lat = parseFloat(feed.field1);
                    const lon = parseFloat(feed.field2);
                    const time = new Date(feed.created_at).toLocaleString();

                    if (!isNaN(lat) && !isNaN(lon)) {
                        updateMap(lat, lon, time);
                    }
                }
            })
            .catch(err => {
                console.error("读取失败:", err);
                document.getElementById('status').innerText = "读取失败";
                document.getElementById('status').style.color = "#f56565";
            });
    }

    function updateMap(lat, lon, time) {
        document.getElementById('lat').innerText = lat.toFixed(6);
        document.getElementById('lon').innerText = lon.toFixed(6);
        document.getElementById('time').innerText = time;
        document.getElementById('status').innerText = "在线";
        document.getElementById('status').style.color = "#48bb78";

        const newPos = [lat, lon];
        marker.setLatLng(newPos);
        map.panTo(newPos);
        marker.bindPopup("最后位置: " + time).openPopup();
    }

    // 手动解析备用逻辑
    function parseManual() {
        const input = document.getElementById('manualInput').value;
        // 简单的逻辑：寻找数字并解析 (根据 +CGPSINFO 格式)
        const parts = input.split(',');
        if (parts.length > 4) {
            let lat = parseFloat(parts[0].replace(/[^0-9.]/g, '')) / 100; // 粗略转换
            let lon = parseFloat(parts[2].replace(/[^0-9.]/g, '')) / 100;
            updateMap(lat, lon, "手动输入数据");
        } else {
            alert("格式不正确，请贴入完整的 AT 指令返回内容");
        }
    }

    // 轮询：每15秒更新一次 (ThingSpeak 免费版限制)
    setInterval(fetchData, 15000);
    fetchData(); // 初始加载
</script>
</body>
</html>
