# App Performance Analytics Implementation Plans

## Goal

Implement LambdaTest-style App Performance Analytics for real-time monitoring during Appium test execution.

---

## 📊 What LambdaTest Offers

### Performance Metrics Tracked

| Category | Metrics | Platform |
|----------|---------|----------|
| **Startup Time** | Cold startup, Hot startup | iOS, Android |
| **CPU** | System CPU %, App CPU %, Max, Average | iOS, Android |
| **Memory** | System memory, App memory, Available memory (MB) | iOS, Android |
| **Disk** | System disk usage, App disk usage | iOS, Android |
| **Rendering** | FPS (Frames Per Second) throughout session | iOS, Android |
| **Network** | Download MB, Upload MB | iOS, Android |
| **Battery** | Battery drain rate, Temperature | Android only |
| **Stability** | ANR events, App crashes | Android only |

### How It Works

1. Enable with capability: `"appProfiling": true`
2. Metrics collected in real-time during test
3. Retrieve via API: `/sessions/SESSION_ID/log/appmetrics`
4. Returns comprehensive JSON with all metrics

---

## 🏗️ Implementation Architecture

### High-Level Flow

```
Test Execution
    ↓
Performance Collector (Background Thread)
    ↓ Collects every 1-2 seconds
    ├─ CPU usage (adb/instruments)
    ├─ Memory usage (adb/instruments)
    ├─ Battery stats (adb)
    ├─ Network stats (adb/instruments)
    └─ FPS data (dumpsys)
    ↓
Store metrics in time-series format
    ↓
Upload to S3 / Store in DB
    ↓
Expose via API: /execution/{id}/performance
    ↓
Frontend displays charts (CPU, Memory, FPS over time)
```

---

## 📝 Implementation Steps

### Phase 1: Data Collection (Android) - 2-3 days

#### Step 1.1: Create Performance Collector Service
**File**: `services/collectors/performance_collector.py`

**What to collect**:
```python
class PerformanceMetrics:
    timestamp: float

    # CPU metrics
    cpu_system_percent: float
    cpu_app_percent: float

    # Memory metrics (MB)
    memory_system_mb: float
    memory_app_mb: float
    memory_available_mb: float

    # Battery metrics
    battery_level: int  # 0-100
    battery_temperature: float  # Celsius

    # Network metrics (delta from last measurement)
    network_rx_bytes: int  # Received
    network_tx_bytes: int  # Transmitted

    # FPS metrics
    fps: Optional[float]

    # App state
    app_state: str  # foreground/background
```

**How to collect (Android)**:

```python
import subprocess
import re
import time

class AndroidPerformanceCollector:
    def __init__(self, device_serial: str, package_name: str):
        self.device_serial = device_serial
        self.package_name = package_name
        self.pid = None
        self.last_network_stats = None

    def get_app_pid(self) -> Optional[int]:
        """Get app process ID"""
        cmd = f"adb -s {self.device_serial} shell pidof {self.package_name}"
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
        if result.returncode == 0 and result.stdout.strip():
            return int(result.stdout.strip())
        return None

    def collect_cpu_usage(self) -> dict:
        """Collect CPU usage via adb shell top"""
        # Method 1: Using top
        cmd = f"adb -s {self.device_serial} shell top -n 1 | grep {self.package_name}"
        # Parse output: PID USER PR NI VIRT RES SHR S %CPU %MEM

        # Method 2: Using dumpsys cpuinfo
        cmd = f"adb -s {self.device_serial} shell dumpsys cpuinfo | grep {self.package_name}"

        return {
            "cpu_system_percent": 0.0,  # Total system CPU
            "cpu_app_percent": 0.0       # App CPU
        }

    def collect_memory_usage(self) -> dict:
        """Collect memory via dumpsys meminfo"""
        cmd = f"adb -s {self.device_serial} shell dumpsys meminfo {self.package_name}"
        # Parse: TOTAL PSS, Native Heap, Dalvik Heap, etc.

        # System memory
        cmd_sys = f"adb -s {self.device_serial} shell cat /proc/meminfo"
        # Parse: MemTotal, MemFree, MemAvailable

        return {
            "memory_system_mb": 0.0,
            "memory_app_mb": 0.0,
            "memory_available_mb": 0.0
        }

    def collect_battery_stats(self) -> dict:
        """Collect battery info"""
        cmd = f"adb -s {self.device_serial} shell dumpsys battery"
        # Parse: level, temperature, status

        return {
            "battery_level": 100,
            "battery_temperature": 0.0
        }

    def collect_network_stats(self) -> dict:
        """Collect network usage"""
        if not self.pid:
            return {"network_rx_bytes": 0, "network_tx_bytes": 0}

        # Method 1: Via /proc/net/xt_qtaguid/stats
        cmd = f"adb -s {self.device_serial} shell cat /proc/net/xt_qtaguid/stats | grep {self.pid}"

        # Method 2: Via dumpsys package
        cmd = f"adb -s {self.device_serial} shell dumpsys package {self.package_name}"

        current_stats = {"rx": 0, "tx": 0}
        if self.last_network_stats:
            delta_rx = current_stats["rx"] - self.last_network_stats["rx"]
            delta_tx = current_stats["tx"] - self.last_network_stats["tx"]
        else:
            delta_rx = delta_tx = 0

        self.last_network_stats = current_stats

        return {
            "network_rx_bytes": delta_rx,
            "network_tx_bytes": delta_tx
        }

    def collect_fps(self) -> Optional[float]:
        """Collect FPS via dumpsys gfxinfo"""
        cmd = f"adb -s {self.device_serial} shell dumpsys gfxinfo {self.package_name}"
        # Parse frame timing data
        # Calculate FPS from frame times

        return None  # Average FPS over last second

    def collect_all_metrics(self) -> PerformanceMetrics:
        """Collect all metrics in one call"""
        if not self.pid:
            self.pid = self.get_app_pid()

        cpu = self.collect_cpu_usage()
        memory = self.collect_memory_usage()
        battery = self.collect_battery_stats()
        network = self.collect_network_stats()
        fps = self.collect_fps()

        return PerformanceMetrics(
            timestamp=time.time(),
            **cpu,
            **memory,
            **battery,
            **network,
            fps=fps,
            app_state="foreground"
        )
```

#### Step 1.2: Integrate with Test Execution
**File**: `services/enhanced_appium_docker_service.py`

```python
class EnhancedAppiumDockerService:
    def __init__(self):
        # ... existing code ...
        self._performance_collector = None
        self._performance_buffer = []
        self._enable_performance_profiling = os.getenv("ENABLE_PERFORMANCE_PROFILING", "false").lower() == "true"

    async def run_appium_test_enhanced(self, test_request: Dict):
        # ... existing setup ...

        # Start performance collection
        if self._enable_performance_profiling:
            device_serial = test_request.get('device_udid')
            package_name = test_request.get('app_package')  # Need to extract from app

            self._performance_collector = AndroidPerformanceCollector(device_serial, package_name)

            # Start background thread for collection
            import threading
            self._perf_thread = threading.Thread(
                target=self._collect_performance_loop,
                daemon=True
            )
            self._perf_thread.start()

        # ... run test ...

        # Stop collection and upload
        if self._enable_performance_profiling:
            self._stop_performance_collection = True
            await self._upload_performance_metrics(test_execution_id)

    def _collect_performance_loop(self):
        """Background thread collecting metrics every 1 second"""
        while not getattr(self, '_stop_performance_collection', False):
            try:
                metrics = self._performance_collector.collect_all_metrics()
                self._performance_buffer.append(metrics)
                time.sleep(1)  # Collect every 1 second
            except Exception as e:
                logger.error(f"Performance collection error: {e}")

    async def _upload_performance_metrics(self, test_execution_id: str):
        """Upload collected metrics to S3"""
        if not self._performance_buffer:
            return

        # Convert to JSON
        metrics_json = [m.dict() for m in self._performance_buffer]

        # Save to file
        perf_file = f"./test_logs/performance_{test_execution_id}.json"
        with open(perf_file, 'w') as f:
            json.dump(metrics_json, f)

        # Upload to S3
        from utils.storage_utils import storage_manager
        remote_path = f"performance/{test_execution_id}/metrics.json"
        await storage_manager.upload_file(perf_file, remote_path, 'application/json')

        # Send URL to backend
        signed_url = storage_manager.generate_signed_url(remote_path, 24)
        await backend_callback_service.send_performance_metrics_callback(
            test_execution_id=test_execution_id,
            performance_url=signed_url
        )
```

### Phase 2: Data Storage & API - 1 day

#### Step 2.1: Backend API Endpoint
**File**: Backend `routes/performance_routes.py`

```python
@router.get("/api/v1/execution/{test_execution_id}/performance")
async def get_performance_metrics(test_execution_id: str):
    """
    Get performance metrics for test execution

    Returns:
    {
        "metrics": [
            {
                "timestamp": 1697000000.123,
                "cpu_app_percent": 45.2,
                "memory_app_mb": 256.5,
                "fps": 58.3,
                ...
            },
            ...
        ],
        "summary": {
            "startup_time_ms": 1234,
            "avg_cpu_percent": 42.1,
            "max_cpu_percent": 78.5,
            "avg_memory_mb": 245.3,
            "max_memory_mb": 312.7,
            "avg_fps": 57.8,
            "min_fps": 45.2,
            "network_rx_total_mb": 12.5,
            "network_tx_total_mb": 3.2,
            "battery_drain_percent": 5,
            "anr_count": 0,
            "crash_count": 0
        }
    }
    """
    execution = get_test_execution_by_id(test_execution_id)
    performance_url = execution.get("performance_metrics_url")

    if not performance_url:
        raise HTTPException(404, "Performance metrics not available")

    # Fetch from S3
    import requests
    response = requests.get(performance_url)
    metrics = response.json()

    # Calculate summary
    summary = calculate_performance_summary(metrics)

    return {
        "metrics": metrics,
        "summary": summary
    }

def calculate_performance_summary(metrics: list) -> dict:
    """Calculate summary statistics from time-series metrics"""
    if not metrics:
        return {}

    cpu_values = [m["cpu_app_percent"] for m in metrics if m.get("cpu_app_percent")]
    memory_values = [m["memory_app_mb"] for m in metrics if m.get("memory_app_mb")]
    fps_values = [m["fps"] for m in metrics if m.get("fps") and m["fps"] > 0]

    return {
        "avg_cpu_percent": sum(cpu_values) / len(cpu_values) if cpu_values else 0,
        "max_cpu_percent": max(cpu_values) if cpu_values else 0,
        "avg_memory_mb": sum(memory_values) / len(memory_values) if memory_values else 0,
        "max_memory_mb": max(memory_values) if memory_values else 0,
        "avg_fps": sum(fps_values) / len(fps_values) if fps_values else 0,
        "min_fps": min(fps_values) if fps_values else 0,
        # ... more calculations
    }
```

### Phase 3: Frontend Visualization - 2-3 days

#### Step 3.1: Performance Dashboard Component
**File**: `components/PerformanceDashboard.tsx`

```typescript
import { Line } from 'react-chartjs-2';

interface PerformanceMetric {
  timestamp: number;
  cpu_app_percent: number;
  memory_app_mb: number;
  fps: number;
  battery_level: number;
}

export function PerformanceDashboard({ testExecutionId }: { testExecutionId: string }) {
  const [metrics, setMetrics] = useState<PerformanceMetric[]>([]);
  const [summary, setSummary] = useState<any>(null);

  useEffect(() => {
    fetch(`/api/v1/execution/${testExecutionId}/performance`)
      .then(res => res.json())
      .then(data => {
        setMetrics(data.metrics);
        setSummary(data.summary);
      });
  }, [testExecutionId]);

  // Prepare chart data
  const cpuChartData = {
    labels: metrics.map(m => new Date(m.timestamp * 1000).toLocaleTimeString()),
    datasets: [{
      label: 'CPU Usage (%)',
      data: metrics.map(m => m.cpu_app_percent),
      borderColor: 'rgb(255, 99, 132)',
      tension: 0.1
    }]
  };

  const memoryChartData = {
    labels: metrics.map(m => new Date(m.timestamp * 1000).toLocaleTimeString()),
    datasets: [{
      label: 'Memory (MB)',
      data: metrics.map(m => m.memory_app_mb),
      borderColor: 'rgb(54, 162, 235)',
      tension: 0.1
    }]
  };

  const fpsChartData = {
    labels: metrics.map(m => new Date(m.timestamp * 1000).toLocaleTimeString()),
    datasets: [{
      label: 'FPS',
      data: metrics.map(m => m.fps),
      borderColor: 'rgb(75, 192, 192)',
      tension: 0.1
    }]
  };

  return (
    <div className="performance-dashboard">
      {/* Summary Cards */}
      <div className="summary-cards">
        <Card title="Avg CPU" value={`${summary?.avg_cpu_percent?.toFixed(1)}%`} />
        <Card title="Max Memory" value={`${summary?.max_memory_mb?.toFixed(0)} MB`} />
        <Card title="Avg FPS" value={summary?.avg_fps?.toFixed(1)} />
        <Card title="Battery Drain" value={`${summary?.battery_drain_percent}%`} />
      </div>

      {/* Charts */}
      <div className="charts">
        <div className="chart">
          <h3>CPU Usage Over Time</h3>
          <Line data={cpuChartData} />
        </div>

        <div className="chart">
          <h3>Memory Usage Over Time</h3>
          <Line data={memoryChartData} />
        </div>

        <div className="chart">
          <h3>FPS Over Time</h3>
          <Line data={fpsChartData} />
        </div>
      </div>
    </div>
  );
}
```

---

## 🔧 Complexity Assessment

### Overall Complexity: **Medium to High**

| Component | Complexity | Time | Difficulty |
|-----------|-----------|------|------------|
| **Android Data Collection** | Medium | 2-3 days | Parsing adb output, handling edge cases |
| **iOS Data Collection** | High | 3-4 days | Requires instruments, more complex |
| **Background Collection Thread** | Low | 4 hours | Standard threading |
| **Data Storage & Upload** | Low | 4 hours | Reuse existing S3 code |
| **Backend API** | Low | 4 hours | Standard REST endpoint |
| **Frontend Charts** | Medium | 2-3 days | Chart.js/Recharts integration |
| **Startup Time Detection** | Medium | 1 day | Need to detect app launch |
| **ANR/Crash Detection** | Medium | 1 day | Parse logcat for ANR/crashes |
| **FPS Calculation** | High | 1-2 days | Complex frame timing analysis |

**Total Estimated Time**:
- **Android only**: 1-2 weeks
- **Android + iOS**: 2-3 weeks

---

## 🎯 Recommended Approach

### Phase 1 (MVP) - 1 week
✅ Start with **Android only**
✅ Collect **basic metrics**: CPU, Memory, Battery
✅ **Time-series** storage (S3 JSON)
✅ **Simple API** endpoint
✅ **Basic charts** (CPU, Memory over time)

### Phase 2 - 1 week
✅ Add **advanced metrics**: FPS, Network
✅ **Startup time** detection
✅ **Summary statistics**
✅ **Better charts** with zoom/pan

### Phase 3 - 1 week
✅ **iOS support**
✅ **ANR/Crash detection**
✅ **Performance alerts** (CPU > 80%, Memory > threshold)
✅ **Comparison** with previous runs

---

## 💡 Shortcuts & Tools

### Use Existing Tools

1. **Appium Profiler Plugin**
   - `appium-device-farm` has built-in profiling
   - May be easier than building from scratch

2. **py-android-profiler**
   - Python library for Android profiling
   - Handles parsing adb output

3. **Chart Libraries**
   - Chart.js - Simple, lightweight
   - Recharts - React-native
   - ApexCharts - Beautiful, feature-rich

4. **Battery Stats**
   ```bash
   adb shell dumpsys batterystats --reset  # Reset before test
   adb shell dumpsys batterystats > battery.txt  # After test
   ```

---

## 📦 Dependencies Needed

### Python (Mobile Service)
```bash
pip install psutil  # For system metrics
pip install py-android-profiler  # If using existing library
```

### Frontend
```bash
npm install chart.js react-chartjs-2
npm install recharts  # Alternative
npm install date-fns  # For time formatting
```

### Backend
```bash
# No new dependencies needed
# Uses existing S3, database, REST framework
```

---

## 🚨 Challenges

### 1. Device Compatibility
- Different Android versions have different adb outputs
- Need to handle multiple formats

### 2. Performance Overhead
- Collecting metrics every second adds overhead
- Need to ensure it doesn't affect test execution

### 3. iOS Complexity
- Requires Xcode instruments
- More restrictive than Android
- May need Mac environment

### 4. Accurate FPS
- Hard to get accurate FPS from adb
- May need to parse gfxinfo carefully
- Alternative: Screenshot-based frame detection (slow)

### 5. Startup Time
- Need to detect exactly when app starts
- Cold vs Hot startup different

---

## 🎯 Recommendation

### Option 1: Build From Scratch (2-3 weeks)
✅ Full control
✅ Customizable
❌ More time
❌ More bugs to fix

### Option 2: Use Existing Library (1-2 weeks)
✅ Faster
✅ Tested
❌ Less flexibility
❌ May not fit exact needs

### Option 3: Hybrid (1.5-2 weeks) **RECOMMENDED**
✅ Use library for data collection (py-android-profiler)
✅ Build custom storage/API/frontend
✅ Best balance of speed and control

---

## 📋 Implementation Checklist

### Week 1
- [ ] Research and choose data collection method
- [ ] Implement basic Android collector (CPU, Memory)
- [ ] Add background thread for collection
- [ ] Upload to S3 in JSON format
- [ ] Create backend API endpoint
- [ ] Test with sample data

### Week 2
- [ ] Add battery, network metrics
- [ ] Implement FPS collection
- [ ] Add startup time detection
- [ ] Create frontend dashboard
- [ ] Integrate Chart.js
- [ ] Display real metrics

### Week 3 (Optional)
- [ ] Add iOS support
- [ ] ANR/Crash detection
- [ ] Performance alerts
- [ ] Comparison feature

---

## 💰 Estimated Effort

| Role | Time | Notes |
|------|------|-------|
| **Backend Developer** | 1 week | Data collection, API, storage |
| **Frontend Developer** | 1 week | Charts, dashboard UI |
| **QA/Testing** | 3-4 days | Test across devices, validate metrics |
| **Total** | **2-3 weeks** | For Android MVP |

---

**Complexity Rating**: 6/10 (Medium-High)
**Feasibility**: ✅ Definitely doable
**Value**: ⭐⭐⭐⭐⭐ High value for users

**Recommendation**: Start with Android MVP (Phase 1) - 1 week, then expand based on user feedback.
