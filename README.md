```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BagGuard • Live Location Checker</title>
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&amp;display=swap');
        
        :root {
            --red: #e63939;
        }
        
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Inter', system-ui, sans-serif;
            background: #0f172a;
            color: #e2e8f0;
            line-height: 1.6;
        }
        
        header {
            background: linear-gradient(90deg, #1e2937, #334155);
            padding: 1rem 0;
            border-bottom: 4px solid var(--red);
            box-shadow: 0 4px 12px rgba(230, 57, 57, 0.3);
        }
        
        .container {
            max-width: 1200px;
            margin: 0 auto;
            padding: 0 20px;
        }
        
        h1 {
            font-size: 2.2rem;
            font-weight: 600;
            display: flex;
            align-items: center;
            gap: 12px;
        }
        
        .badge {
            background: var(--red);
            color: white;
            font-size: 0.9rem;
            padding: 4px 12px;
            border-radius: 9999px;
            font-weight: 600;
        }
        
        .main {
            display: grid;
            grid-template-columns: 1fr 420px;
            gap: 25px;
            margin-top: 25px;
        }
        
        @media (max-width: 1024px) {
            .main {
                grid-template-columns: 1fr;
            }
        }
        
        .map-container {
            background: #1e2937;
            border-radius: 16px;
            overflow: hidden;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.4);
            height: 620px;
            position: relative;
        }
        
        #map {
            width: 100%;
            height: 100%;
        }
        
        .card {
            background: #1e2937;
            border-radius: 16px;
            padding: 24px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.4);
            height: fit-content;
        }
        
        .section {
            margin-bottom: 28px;
        }
        
        .section:last-child {
            margin-bottom: 0;
        }
        
        h2 {
            font-size: 1.4rem;
            margin-bottom: 14px;
            color: #f1f5f9;
            display: flex;
            align-items: center;
            gap: 8px;
        }
        
        button {
            background: var(--red);
            color: white;
            border: none;
            padding: 14px 24px;
            font-size: 1.05rem;
            font-weight: 600;
            border-radius: 12px;
            cursor: pointer;
            transition: all 0.2s;
            width: 100%;
            margin-top: 8px;
        }
        
        button:hover {
            background: #d12e2e;
            transform: translateY(-2px);
        }
        
        button.success {
            background: #10b981;
        }
        
        .input-group {
            margin-top: 12px;
        }
        
        label {
            display: block;
            margin-bottom: 6px;
            font-weight: 500;
            font-size: 0.95rem;
            color: #cbd5e1;
        }
        
        textarea {
            width: 100%;
            height: 110px;
            background: #334155;
            border: 2px solid #475569;
            border-radius: 12px;
            color: #e2e8f0;
            padding: 14px;
            font-family: monospace;
            font-size: 0.95rem;
            resize: vertical;
        }
        
        .result {
            background: #334155;
            border-radius: 12px;
            padding: 16px;
            margin-top: 16px;
            font-family: monospace;
            font-size: 1.1rem;
            display: none;
        }
        
        .coords {
            display: flex;
            gap: 20px;
            flex-wrap: wrap;
        }
        
        .coord-item {
            background: #1e2937;
            padding: 12px 18px;
            border-radius: 10px;
            flex: 1;
            min-width: 140px;
        }
        
        .coord-label {
            font-size: 0.8rem;
            color: #94a3b8;
            text-transform: uppercase;
        }
        
        .coord-value {
            font-size: 1.4rem;
            font-weight: 600;
            color: #60a5fa;
        }
        
        .status {
            padding: 10px 16px;
            border-radius: 9999px;
            font-size: 0.9rem;
            font-weight: 600;
            display: inline-flex;
            align-items: center;
            gap: 6px;
        }
        
        .footer {
            text-align: center;
            margin-top: 40px;
            padding: 20px;
            color: #64748b;
            font-size: 0.9rem;
        }
        
        .legend {
            position: absolute;
            bottom: 20px;
            left: 20px;
            background: rgba(30, 41, 55, 0.95);
            padding: 10px 16px;
            border-radius: 12px;
            font-size: 0.85rem;
            z-index: 1000;
            box-shadow: 0 4px 12px rgba(0,0,0,0.5);
        }
    </style>
</head>
<body>
    <header>
        <div class="container">
            <h1>
                🛡️ BagGuard
                <span class="badge">ANTI-THEFT</span>
                Live Location Checker
            </h1>
            <p style="opacity: 0.8; margin-top: 4px;">Your Arduino + GPRS device sends GPS via SMS • Paste it here to see exact location on map</p>
        </div>
    </header>

    <div class="container">
        <div class="main">
            
            <!-- MAP -->
            <div class="map-container">
                <div id="map"></div>
                
                <!-- Map Legend -->
                <div class="legend">
                    <div style="display:flex; gap:16px; align-items:center;">
                        <div style="display:flex; align-items:center; gap:6px;">
                            <span style="display:inline-block; width:12px; height:12px; background:#60a5fa; border-radius:50%;"></span>
                            <span><strong>Your Device</strong></span>
                        </div>
                        <div style="display:flex; align-items:center; gap:6px;">
                            <span style="display:inline-block; width:12px; height:12px; background:#f59e0b; border-radius:50%;"></span>
                            <span><strong>Bag Location</strong> (from SMS)</span>
                        </div>
                    </div>
                </div>
            </div>

            <!-- CONTROLS -->
            <div>

                <!-- 1. Browser Location -->
                <div class="card">
                    <div class="section">
                        <h2>📍 Check My Current Location</h2>
                        <p style="font-size:0.95rem; color:#cbd5e1; margin-bottom:16px;">
                            Uses your phone / laptop GPS (very accurate in Hong Kong)
                        </p>
                        <button onclick="getBrowserLocation()">📡 Get My Live Location Now</button>
                        
                        <div id="browser-result" class="result">
                            <div class="coords">
                                <div class="coord-item">
                                    <div class="coord-label">Latitude</div>
                                    <div id="browser-lat" class="coord-value">—</div>
                                </div>
                                <div class="coord-item">
                                    <div class="coord-label">Longitude</div>
                                    <div id="browser-lng" class="coord-value">—</div>
                                </div>
                            </div>
                            <p id="browser-status" style="margin-top:12px; font-size:0.9rem;"></p>
                        </div>
                    </div>
                </div>

                <!-- 2. Bag GPS from SMS -->
                <div class="card">
                    <div class="section">
                        <h2>🧳 Bag Location from SMS</h2>
                        <p style="font-size:0.95rem; color:#cbd5e1; margin-bottom:12px;">
                            Copy the GPS line from the SMS your device sent and paste it below
                        </p>
                        
                        <div class="input-group">
                            <label>📨 Paste full GPS response here (example: +CGPSINFO: 2235.1234,N,11355.1234,E,...)</label>
                            <textarea id="gps-input" placeholder="+CGPSINFO: 2235.XXXX,N,11355.XXXX,E,100420,123456.000,..."></textarea>
                        </div>
                        
                        <button onclick="parseAndShowBagLocation()">🔍 Parse &amp; Show Bag on Map</button>
                        
                        <div id="bag-result" class="result">
                            <div class="coords">
                                <div class="coord-item">
                                    <div class="coord-label">Latitude</div>
                                    <div id="bag-lat" class="coord-value">—</div>
                                </div>
                                <div class="coord-item">
                                    <div class="coord-label">Longitude</div>
                                    <div id="bag-lng" class="coord-value">—</div>
                                </div>
                            </div>
                            <div style="margin-top:14px; font-size:0.9rem; color:#86efac;">
                                ✅ Successfully parsed • Bag location plotted
                            </div>
                            <button onclick="clearBagMarker()" style="background:#64748b; margin-top:12px; font-size:0.9rem; padding:10px;">Clear Bag Marker</button>
                        </div>
                    </div>
                </div>

                <div style="background:#1e2937; border-radius:16px; padding:20px; font-size:0.85rem; color:#94a3b8; text-align:center;">
                    💡 Tip: Your device is currently in <strong>Tung Chung, Hong Kong</strong> area.<br>
                    The map is centered there by default.
                </div>
            </div>
        </div>
    </div>

    <div class="footer">
        Built for your ESP32 + MPU6050 + PN532 + GPRS anti-theft project • 
        Works offline after loading • Open this page on your phone for best GPS accuracy
    </div>

    <script>
        // Initialize Leaflet map centered on Tung Chung, Hong Kong
        let map = L.map('map', {
            zoomControl: true,
            attributionControl: false
        }).setView([22.2894, 113.9429], 15);  // Tung Chung coordinates
        
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            className: 'map-tiles'
        }).addTo(map);
        
        // Markers
        let browserMarker = null;
        let bagMarker = null;
        
        // Function to add / update marker
        function addMarker(lat, lng, label, color) {
            const icon = L.divIcon({
                className: 'custom-marker',
                html: `<div style="background:${color}; width:28px; height:28px; border-radius:50%; border:3px solid white; box-shadow:0 0 8px rgba(0,0,0,0.5); display:flex; align-items:center; justify-content:center; color:white; font-weight:bold; font-size:14px;">📍</div>`,
                iconSize: [28, 28],
                iconAnchor: [14, 14]
            });
            
            const popup = L.popup().setContent(`
                <b>${label}</b><br>
                <span style="font-size:0.9rem; color:#64748b;">
                    ${lat.toFixed(6)}, ${lng.toFixed(6)}
                </span>
            `);
            
            if (label === "Your Device" && browserMarker) browserMarker.remove();
            if (label === "Bag Location" && bagMarker) bagMarker.remove();
            
            const marker = L.marker([lat, lng], { icon: icon }).addTo(map).bindPopup(popup);
            
            if (label === "Your Device") browserMarker = marker;
            else bagMarker = marker;
            
            map.flyTo([lat, lng], 17, { duration: 1.5 });
            marker.openPopup();
        }
        
        // Get browser location
        window.getBrowserLocation = function() {
            const resultDiv = document.getElementById('browser-result');
            
            if (!navigator.geolocation) {
                alert("Your browser does not support geolocation.");
                return;
            }
            
            resultDiv.style.display = 'block';
            document.getElementById('browser-status').innerHTML = `
                <span class="status" style="background:#10b981; color:white;">🔄 Getting location...</span>
            `;
            
            navigator.geolocation.getCurrentPosition(
                (position) => {
                    const lat = position.coords.latitude;
                    const lng = position.coords.longitude;
                    
                    document.getElementById('browser-lat').textContent = lat.toFixed(6);
                    document.getElementById('browser-lng').textContent = lng.toFixed(6);
                    document.getElementById('browser-status').innerHTML = `
                        <span class="status" style="background:#10b981; color:white;">✅ Accurate • ${position.coords.accuracy.toFixed(0)}m</span>
                    `;
                    
                    addMarker(lat, lng, "Your Device", "#60a5fa");
                },
                (error) => {
                    let msg = "Unknown error";
                    if (error.code === 1) msg = "Location permission denied";
                    else if (error.code === 2) msg = "Position unavailable";
                    else if (error.code === 3) msg = "Timeout";
                    document.getElementById('browser-status').innerHTML = `
                        <span class="status" style="background:#ef4444; color:white;">❌ ${msg}</span>
                    `;
                },
                { enableHighAccuracy: true, maximumAge: 0, timeout: 10000 }
            );
        };
        
        // Parse SIMCom AT+CGPSINFO format
        function parseGPSString(str) {
            // Remove command prefix if present
            let clean = str.replace(/\+CGPSINFO:\s*/i, '').trim();
            
            // If empty or just commas → no fix
            if (!clean || clean === '' || clean === ',,,,,,,,') {
                return null;
            }
            
            const parts = clean.split(',').map(p => p.trim());
            
            // We need at least lat, NS, lon, EW
            if (parts.length < 4) return null;
            
            const latStr = parts[0];
            const ns = parts[1];
            const lonStr = parts[2];
            const ew = parts[3];
            
            if (!latStr || !ns || !lonStr || !ew) return null;
            
            // Convert ddmm.mmmm / dddmm.mmmm → decimal degrees
            const lat_raw = parseFloat(latStr);
            const lon_raw = parseFloat(lonStr);
            
            if (isNaN(lat_raw) || isNaN(lon_raw)) return null;
            
            let lat = Math.floor(lat_raw / 100) + (lat_raw % 100) / 60;
            let lon = Math.floor(lon_raw / 100) + (lon_raw % 100) / 60;
            
            // Apply hemisphere sign
            if (ns.toUpperCase() === 'S') lat = -lat;
            if (ew.toUpperCase() === 'W') lon = -lon;
            
            // Simple validation
            if (lat < -90 || lat > 90 || lon < -180 || lon > 180) return null;
            
            return { lat, lon };
        }
        
        // Parse and show bag location
        window.parseAndShowBagLocation = function() {
            const input = document.getElementById('gps-input').value.trim();
            const resultDiv = document.getElementById('bag-result');
            
            if (!input) {
                alert("Please paste the GPS data from your SMS first.");
                return;
            }
            
            const location = parseGPSString(input);
            
            if (!location) {
                alert("❌ Could not parse GPS data.\n\nMake sure you copied the full line starting with +CGPSINFO: or the numbers after it.");
                return;
            }
            
            // Show result
            resultDiv.style.display = 'block';
            document.getElementById('bag-lat').textContent = location.lat.toFixed(6);
            document.getElementById('bag-lng').textContent = location.lon.toFixed(6);
            
            // Plot on map
            addMarker(location.lat, location.lon, "Bag Location", "#f59e0b");
        };
        
        // Clear bag marker
        window.clearBagMarker = function() {
            if (bagMarker) {
                bagMarker.remove();
                bagMarker = null;
            }
            document.getElementById('bag-result').style.display = 'none';
        };
        
        // Auto-center map on Hong Kong when page loads
        console.log('%c✅ BagGuard Location Checker loaded successfully!', 'color:#10b981; font-size:14px; font-weight:600');
    </script>
</body>
</html>
