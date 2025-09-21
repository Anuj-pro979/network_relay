**Excellent question!** You're thinking about the **automatic traffic routing** part. Once WiFi Direct connection is established, we need to monitor which apps need internet and route their traffic through the mesh when there's no direct signal. Here's how:

## Traffic Monitoring & Routing System

### 1. Network State Monitor (Java)
```java
// Add to your WiFiDirectActivity.java
public class NetworkMonitor {
    private ConnectivityManager connectivityManager;
    private boolean hasDirectInternet = false;
    private boolean wifiDirectConnected = false;
    
    public NetworkMonitor(Context context) {
        connectivityManager = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        startMonitoring();
    }
    
    private void startMonitoring() {
        // Monitor network state changes
        NetworkRequest.Builder builder = new NetworkRequest.Builder();
        connectivityManager.registerNetworkCallback(
            builder.build(),
            new ConnectivityManager.NetworkCallback() {
                @Override
                public void onAvailable(Network network) {
                    checkNetworkStatus();
                }
                
                @Override
                public void onLost(Network network) {
                    checkNetworkStatus();
                }
            }
        );
    }
    
    private void checkNetworkStatus() {
        NetworkInfo activeNetwork = connectivityManager.getActiveNetworkInfo();
        boolean hasInternet = (activeNetwork != null && activeNetwork.isConnected());
        
        if (!hasInternet && wifiDirectConnected) {
            // No direct internet but WiFi Direct available - route through mesh
            enableMeshRouting();
            notifyJavaScript("routeViaMesh", true);
        } else if (hasInternet) {
            // Direct internet available - use direct connection
            disableMeshRouting();
            notifyJavaScript("routeViaMesh", false);
        }
    }
    
    @JavascriptInterface
    public String getNetworkStatus() {
        return "{ \"hasDirectInternet\": " + hasDirectInternet + 
               ", \"meshAvailable\": " + wifiDirectConnected + " }";
    }
}
```

### 2. VPN Service for Traffic Interception (Java)
```java
// VPNProxyService.java - Captures ALL app traffic
public class VPNProxyService extends VpnService {
    private ParcelFileDescriptor vpnInterface;
    private Thread vpnThread;
    private boolean useMeshRoute = false;
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Create VPN tunnel to intercept all traffic
        Builder builder = new Builder();
        builder.setMtu(1500);
        builder.addAddress("10.0.0.2", 24);  // VPN IP
        builder.addRoute("0.0.0.0", 0);      // Route all traffic through VPN
        builder.addDnsServer("8.8.8.8");
        
        vpnInterface = builder.establish();
        
        // Start packet processing thread
        vpnThread = new Thread(() -> processPackets());
        vpnThread.start();
        
        return START_STICKY;
    }
    
    private void processPackets() {
        FileInputStream in = new FileInputStream(vpnInterface.getFileDescriptor());
        FileOutputStream out = new FileOutputStream(vpnInterface.getFileDescriptor());
        
        byte[] packet = new byte[32767];
        
        while (true) {
            try {
                int length = in.read(packet);
                if (length > 0) {
                    // Parse packet to get destination
                    PacketInfo info = parsePacket(packet, length);
                    
                    if (shouldRouteThroughMesh(info)) {
                        // Send through WiFi Direct mesh
                        routeViaMesh(packet, length, info);
                    } else {
                        // Send through direct internet
                        routeDirect(packet, length);
                    }
                }
            } catch (IOException e) {
                break;
            }
        }
    }
    
    private boolean shouldRouteThroughMesh(PacketInfo info) {
        // Check if direct internet is available
        if (hasDirectInternet()) {
            return false; // Use direct internet
        }
        
        // Check if destination is reachable via mesh
        if (isMeshAvailable() && isDestinationRoutable(info.destinationIP)) {
            return true; // Route through mesh
        }
        
        return false; // Drop packet or queue for later
    }
    
    private void routeViaMesh(byte[] packet, int length, PacketInfo info) {
        // Forward packet to WiFi Direct peer
        try {
            Socket meshSocket = getMeshConnection();
            OutputStream meshOut = meshSocket.getOutputStream();
            
            // Send packet through WiFi Direct connection
            meshOut.write(packet, 0, length);
            
            // Wait for response from mesh peer
            InputStream meshIn = meshSocket.getInputStream();
            byte[] response = new byte[32767];
            int responseLength = meshIn.read(response);
            
            // Send response back to original app
            if (responseLength > 0) {
                FileOutputStream out = new FileOutputStream(vpnInterface.getFileDescriptor());
                out.write(response, 0, responseLength);
            }
            
        } catch (IOException e) {
            // Mesh routing failed
            handleMeshFailure(packet, length);
        }
    }
}
```

### 3. App Usage Monitor (JavaScript Interface)
```html
<!-- Add to your index.html -->
<div class="card" id="app-monitor-section">
    <h2>📱 App Traffic Monitor</h2>
    <div class="toggle-switch">
        <label>
            <input type="checkbox" id="auto-routing" onchange="toggleAutoRouting()">
            Auto-route apps through mesh when no signal
        </label>
    </div>
    
    <div id="app-list"></div>
    <div id="traffic-stats"></div>
</div>

<script>
// Monitor which apps are trying to access internet
function startAppMonitoring() {
    // Get list of installed apps and their network usage
    WiFiDirect.getInstalledApps();
    
    // Start monitoring network requests
    setInterval(() => {
        WiFiDirect.getNetworkStats();
    }, 5000);
}

function toggleAutoRouting() {
    const enabled = document.getElementById('auto-routing').checked;
    WiFiDirect.setAutoRouting(enabled);
    
    if (enabled) {
        showStatus('Auto-routing enabled - Apps will use mesh when no signal', 'success');
    } else {
        showStatus('Auto-routing disabled - Direct internet only', 'info');
    }
}

// Called by Java when network status changes
function onNetworkStatusChange(status) {
    const stats = JSON.parse(status);
    updateNetworkDisplay(stats);
    
    if (!stats.hasDirectInternet && stats.meshAvailable) {
        showStatus('📶 No direct signal - Routing through mesh network', 'loading');
        enableMeshRouting();
    } else if (stats.hasDirectInternet) {
        showStatus('📶 Direct internet available', 'success');
        disableMeshRouting();
    }
}

// Display which apps are using which route
function updateAppTrafficDisplay(appTraffic) {
    const appList = document.getElementById('app-list');
    appList.innerHTML = '';
    
    JSON.parse(appTraffic).forEach(app => {
        const div = document.createElement('div');
        div.className = 'app-item';
        
        const routeIcon = app.routeType === 'mesh' ? '🔄' : '📶';
        const routeText = app.routeType === 'mesh' ? 'via Mesh' : 'Direct';
        
        div.innerHTML = `
            <div class="app-info">
                <strong>${app.name}</strong>
                <span class="route-indicator">${routeIcon} ${routeText}</span>
            </div>
            <div class="traffic-info">
                <small>↓ ${formatBytes(app.downloadBytes)} ↑ ${formatBytes(app.uploadBytes)}</small>
            </div>
        `;
        
        appList.appendChild(div);
    });
}
</script>
```

### 4. Intelligent Routing Logic (Java)
```java
// Add to WiFiBridge class
@JavascriptInterface
public void setAutoRouting(boolean enabled) {
    this.autoRoutingEnabled = enabled;
    
    if (enabled) {
        // Start VPN service to intercept traffic
        Intent vpnIntent = new Intent(this, VPNProxyService.class);
        startService(vpnIntent);
        
        // Start app monitoring
        startAppTrafficMonitoring();
    } else {
        // Stop VPN service
        Intent vpnIntent = new Intent(this, VPNProxyService.class);
        stopService(vpnIntent);
    }
}

@JavascriptInterface
public String getNetworkStats() {
    NetworkStatsManager statsManager = (NetworkStatsManager) getSystemService(Context.NETWORK_STATS_SERVICE);
    
    JSONArray appStats = new JSONArray();
    
    // Get network usage for each app
    List<ApplicationInfo> apps = getPackageManager().getInstalledApplications(0);
    for (ApplicationInfo app : apps) {
        try {
            NetworkStats.Bucket bucket = new NetworkStats.Bucket();
            NetworkStats stats = statsManager.queryUidDetails(
                ConnectivityManager.TYPE_MOBILE, 
                null, 
                System.currentTimeMillis() - 3600000, // Last hour
                System.currentTimeMillis(),
                app.uid
            );
            
            if (stats.hasNextBucket()) {
                stats.getNextBucket(bucket);
                
                JSONObject appStat = new JSONObject();
                appStat.put("name", getAppName(app));
                appStat.put("packageName", app.packageName);
                appStat.put("downloadBytes", bucket.getRxBytes());
                appStat.put("uploadBytes", bucket.getTxBytes());
                appStat.put("routeType", getRouteType(app.uid));
                
                appStats.put(appStat);
            }
        } catch (Exception e) {
            // Handle permission issues
        }
    }
    
    return appStats.toString();
}

private String getRouteType(int appUid) {
    // Determine if app traffic is going through mesh or direct
    if (!hasDirectInternet() && isMeshAvailable()) {
        return "mesh";
    }
    return "direct";
}
```

### 5. Mesh Peer Communication
```java
// Handle requests from mesh peers
private void handleMeshProxyRequest(Socket peerSocket) {
    try {
        InputStream in = peerSocket.getInputStream();
        OutputStream out = peerSocket.getOutputStream();
        
        // Read packet from peer
        byte[] packet = new byte[32767];
        int length = in.read(packet);
        
        if (length > 0) {
            // Parse destination from packet
            PacketInfo info = parsePacket(packet, length);
            
            // Forward to internet using our connection
            byte[] response = forwardToInternet(packet, length, info);
            
            // Send response back to peer
            if (response != null) {
                out.write(response);
            }
        }
        
    } catch (IOException e) {
        // Handle mesh communication error
    }
}

private byte[] forwardToInternet(byte[] packet, int length, PacketInfo info) {
    try {
        // Create connection to actual destination
        Socket internetSocket = new Socket(info.destinationIP, info.destinationPort);
        
        // Forward original packet
        OutputStream internetOut = internetSocket.getOutputStream();
        internetOut.write(packet, 0, length);
        
        // Get response
        InputStream internetIn = internetSocket.getInputStream();
        byte[] response = new byte[32767];
        int responseLength = internetIn.read(response);
        
        internetSocket.close();
        
        if (responseLength > 0) {
            return Arrays.copyOf(response, responseLength);
        }
        
    } catch (IOException e) {
        // Internet forwarding failed
    }
    
    return null;
}
```

## How It All Works Together

```
1. App makes network request → VPN Service intercepts
2. VPN Service checks: Do we have direct internet?
3. If NO direct internet:
   - Check: Is mesh available?
   - If YES: Route packet through WiFi Direct peer
   - Peer forwards to internet using their connection
   - Response comes back through mesh
4. If YES direct internet:
   - Route directly to internet
5. JavaScript interface shows real-time routing status
```

## Visual Flow
```
WhatsApp → VPN Service → Network Check
                            ↓
                    No Direct Signal?
                    ↙            ↘
            Route via Mesh    Route Direct
                 ↓                ↓
            WiFi Direct      Mobile Data
            Peer Device         ↓
                 ↓           Internet
            Internet            ↓
                 ↓           Response
            Response            ↓
                 ↓           WhatsApp
            WiFi Direct
                 ↓
            Your Device
                 ↓
            WhatsApp
```

This system automatically detects when apps need internet, checks signal availability, and intelligently routes traffic through the most reliable path. The user sees real-time status of which apps are using which route!

Would you like me to show you how to implement any specific part of this monitoring system?
