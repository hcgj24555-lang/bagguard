<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BagGuard - LBS Real-time Tracking (Wide Range Mode)</title>
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
    <h1 style="margin:0; font-size: 1.5rem;">🛡️ BagGuard Real-time Tracking System</h1>
    <p style="margin:5px 0 0; color: #a0aec0; font-size: 0.9rem;">Base Station Positioning (LBS) - Wide Range Mode</p>
</header>
<div class="container">
    <div id="map"></div>
    <div class="sidebar">
        <div class="card">
            <h2>📍 Device Status</h2>
            <div class="data-point">Status: <span id="status" class="status-online">Connecting...</span></div>
            <div class="data-point">Latitude: <span id="lat" style="color:#f6ad55">---</span></div>
            <div class="data-point">Longitude: <span id="lon" style="color:#f6ad55">---</span></div>
            <div class="data-point">Search Radius: <span id="accText" style="color:#63b3ed">---</span></div>
            <div class="data-point">Last Update: <span id="time" style="font-size: 0.8rem; color:#cbd5e0">---</span></div>
           
            <p class="hint-text">
                ⚠️ <b>Note:</b><br>
                This system uses LBS (Base Station) positioning. The circle represents the possible area where the device may be located (typical error: 500–1500 meters).
            </p>
        </div>
        <div class="card">
            <h2>🛠️ Manual Debug</h2>
            <textarea id="manualInput" style="width: 100%; height: 50px; background: #1a202c; color: #cbd5e0; border: 1px solid #4a5568; border-radius: 5px; padding: 5px;" placeholder="Paste AT command response here..."></textarea>
            <button onclick="parseManual()" style="width: 100%; margin-top: 10px; padding: 10px; background: #e53e3e; color: white; border: none; border-radius: 5px; cursor: pointer; font-weight: bold;">Show on Map</button>
        </div>
    </div>
</div>
<div class="footer">Powered by ESP32 + A7670X + Leaflet.js</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
    // --- Configuration ---
    const channelID = "3336521";
    const readAPIKey = "M3EDCYNO6BNOICSA";
   
    // Default large radius for LBS (in meters)
    const LBS_DEFAULT_RADIUS = 1500;

    var map, marker, accuracyCircle;

    function initMap() {
        map = L.map('map').setView([22.3193, 114.1694], 13);
       
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 18,
            attribution: '© OpenStreetMap'
        }).addTo(map);

        marker = L.marker([22.3193, 114.1694]).addTo(map);
        
        accuracyCircle = L.circle([22.3193, 114.1694], {
            color: '#3182ce',
            weight: 2,
            fillColor: '#63b3ed',
            fillOpacity: 0.15,
            radius: LBS_DEFAULT_RADIUS
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
                    const acc = LBS_DEFAULT_RADIUS;
                    const time = new Date(feed.created_at).toLocaleString('en-US');
                    
                    if (!isNaN(lat) && !isNaN(lon)) {
                        updateUI(lat, lon, time, acc);
                    }
                }
            })
            .catch(err => console.log("Failed to fetch data"));
    }

    function updateUI(lat, lon, time, acc) {
        document.getElementById('lat').innerText = lat.toFixed(6);
        document.getElementById('lon').innerText = lon.toFixed(6);
        document.getElementById('accText').innerText = (acc/1000).toFixed(1) + " km";
        document.getElementById('time').innerText = time;
        document.getElementById('status').innerText = "Online";
        const pos = [lat, lon];
       
        marker.setLatLng(pos);
        accuracyCircle.setLatLng(pos);
        accuracyCircle.setRadius(acc);
       
        map.panTo(pos);
        marker.bindPopup("<b>Search Area Center</b><br>Last Update: " + time).openPopup();
    }

    function parseManual() {
        const input = document.getElementById('manualInput').value;
        const regex = /([0-9]+\.[0-9]+)/g;
        const matches = input.match(regex);
        if (matches && matches.length >= 2) {
            updateUI(parseFloat(matches[0]), parseFloat(matches[1]), "Manual Debug", LBS_DEFAULT_RADIUS);
        }
    }

    window.onload = () => {
        initMap();
        fetchData();
        setInterval(fetchData, 15000);
    };
</script>
</body>
</html>
