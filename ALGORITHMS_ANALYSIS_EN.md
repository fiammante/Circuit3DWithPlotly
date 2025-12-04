# Algorithm and Critical Code Analysis

## Table of Contents

1. [Asynchronous Loading Algorithm](#1-asynchronous-loading-algorithm)
2. [Physics Simulation Engine](#2-physics-simulation-engine)
3. [Circuit Position Interpolation](#3-circuit-position-interpolation)
4. [Data Bucketing for 3D Visualization](#4-data-bucketing-for-3d-visualization)
5. [Circular Buffer Logging System](#5-circular-buffer-logging-system)
6. [State Management with React Refs](#6-state-management-with-react-refs)
7. [Performance Optimizations](#7-performance-optimizations)

---

## 1. Asynchronous Loading Algorithm

### Context
Loading external resources (React, Plotly) from CDN with timeout and retry management.

### Source Code (lines 221-339)

```javascript
class LoadingManager {
    constructor() {
        this.resources = {
            react: { loaded: false, error: null },
            reactDOM: { loaded: false, error: null },
            plotly: { loaded: false, error: null },
            babel: { loaded: false, error: null }
        };
        this.loadingElement = document.getElementById('loading-screen');
        this.statusElement = document.getElementById('loading-status');
    }

    async checkResource(name, checkFn, maxRetries = 50, retryDelay = 100) {
        log(`Checking ${name}...`, 'info');
        
        try {
            // Retry strategy with implicit exponential backoff
            for (let i = 0; i < maxRetries; i++) {
                if (checkFn()) {
                    this.resources[name].loaded = true;
                    log(`✓ ${name} loaded`, 'success');
                    this.updateUI();
                    return true;
                }
                
                // Non-blocking async wait
                await new Promise(resolve => setTimeout(resolve, retryDelay));
            }
            
            throw new Error(`Timeout after ${maxRetries} attempts`);
            
        } catch (error) {
            this.resources[name].error = error.message;
            log(`✗ Error ${name}: ${error.message}`, 'error');
            this.updateUI();
            throw error;
        }
    }

    async checkAllResources() {
        log('Starting resource verification...', 'info');
        
        try {
            // Parallel loading with Promise.all
            await Promise.all([
                this.checkResource('react', () => typeof React !== 'undefined'),
                this.checkResource('reactDOM', () => typeof ReactDOM !== 'undefined'),
                this.checkResource('plotly', () => typeof Plotly !== 'undefined'),
                this.checkResource('babel', () => typeof Babel !== 'undefined')
            ]);
            
            log('All resources loaded!', 'success');
            
            // Visual delay before hiding
            setTimeout(() => this.hideLoadingScreen(), 500);
            
        } catch (error) {
            log('Loading failed', 'error');
            this.statusElement.innerHTML = `
                <span style="color: #ff4444;">❌ Loading Error</span><br/>
                <span style="color: #888; font-size: 12px;">${error.message}</span>
            `;
        }
    }
}
```

### Algorithmic Analysis

**Time Complexity**:
- Best case: O(1) if resource immediately available
- Average case: O(n) where n = number of retries
- Worst case: O(n × k) where k = number of resources

**Space Complexity**: O(1) - constant state

**Patterns Used**:
1. **Retry with exponential backoff** (implicit via retryDelay)
2. **Promise.all for parallel execution** 
3. **Async/await for readability**

### Possible Improvements

```javascript
// Explicit exponential backoff
async checkResourceWithBackoff(name, checkFn, maxRetries = 10) {
    for (let i = 0; i < maxRetries; i++) {
        if (checkFn()) return true;
        
        const delay = Math.min(100 * Math.pow(2, i), 5000);
        await new Promise(resolve => setTimeout(resolve, delay));
    }
    throw new Error('Timeout');
}

// Circuit breaker pattern
class CircuitBreaker {
    constructor(threshold = 3) {
        this.failures = 0;
        this.threshold = threshold;
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    }
    
    async execute(fn) {
        if (this.state === 'OPEN') {
            throw new Error('Circuit breaker is OPEN');
        }
        
        try {
            const result = await fn();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }
    
    onSuccess() {
        this.failures = 0;
        this.state = 'CLOSED';
    }
    
    onFailure() {
        this.failures++;
        if (this.failures >= this.threshold) {
            this.state = 'OPEN';
            setTimeout(() => this.state = 'HALF_OPEN', 10000);
        }
    }
}
```

---

## 2. Physics Simulation Engine

### Context
Real-time simulation of 9 vehicles on circuit with realistic physics.

### Source Code (lines 1007-1162)

```javascript
function updateSimulation() {
    const now = performance.now();
    const deltaTime = (now - (lastUpdateTime || now)) / 1000; // Convert ms → s
    lastUpdateTime = now;
    
    simState.current.time += deltaTime;
    const { playerCar, otherCars, speedHistory } = simState.current;
    const numSegments = circuitData.length - 1;
    
    // Helper function: update a vehicle
    const updateCar = (car) => {
        // 1. CALCULATE POSITION IN SEGMENT
        const progress = car.position % 1.0;
        const segmentIndex = Math.floor(progress * numSegments);
        const nextIndex = (segmentIndex + 1) % circuitData.length;
        
        const start = circuitData[segmentIndex];
        const end = circuitData[nextIndex];
        const localT = (progress * numSegments) % 1;
        
        // 2. LINEAR POSITION INTERPOLATION
        car.x = start.x + (end.x - start.x) * localT;
        car.y = start.y + (end.y - start.y) * localT;
        
        // 3. ROTATION ANGLE CALCULATION
        const dx = end.x - start.x;
        const dy = end.y - start.y;
        car.angle = Math.atan2(dy, dx);
        
        // 4. ACCELERATION/BRAKING PHYSICS
        const waypoint = start;
        car.targetSpeed = waypoint.targetSpeed;
        
        const speedDiff = car.targetSpeed - car.speed;
        const accelFactor = speedDiff > 0 ? waypoint.accel : waypoint.brake;
        
        // Speed change limitation (realism)
        const maxChange = accelFactor * 20; // km/h per frame max
        const speedChange = Math.max(-maxChange, Math.min(maxChange, speedDiff * 0.1));
        car.speed += speedChange;
        
        // 5. SPEED → DISPLACEMENT CONVERSION
        const metersPerSecond = (car.speed * 1000) / 3600;
        const distanceThisFrame = metersPerSecond * deltaTime;
        
        // Normalization by total circuit length
        car.position += distanceThisFrame / totalCircuitLength;
        
        // 6. LAP DETECTION
        if (car.position >= 1.0) {
            car.position -= 1.0;
            if (car === playerCar) {
                simState.current.lap++;
                log(`Lap ${simState.current.lap} completed!`, 'success');
            }
        }
    };
    
    // Update player
    updateCar(playerCar);
    
    // Calculate acceleration for history
    const acceleration = deltaTime > 0 
        ? ((playerCar.speed - (speedHistory[speedHistory.length - 1]?.speed || 0)) 
           * 1000 / 3600) / deltaTime 
        : 0;
    
    // Record history with FIFO limitation
    speedHistory.push({ speed: playerCar.speed, accel: acceleration });
    if (speedHistory.length > 1000) {
        speedHistory.shift();
    }
    
    // Update opponents
    otherCars.forEach(updateCar);
}
```

### Detailed Analysis

#### 1. Temporal Calculation

```javascript
const deltaTime = (now - lastUpdateTime) / 1000;
```

**Justification**: Allows frame-rate independence. If the system runs at 30 FPS instead of 60, vehicles move 2× more per frame to maintain constant speed.

**Formula**:
```
Δt = t_now - t_last

If FPS = 60 → Δt ≈ 0.0167s
If FPS = 30 → Δt ≈ 0.0333s
```

#### 2. Position Interpolation

```javascript
const progress = car.position % 1.0;  // [0, 1]
const segmentIndex = Math.floor(progress * numSegments);
const localT = (progress * numSegments) % 1;

car.x = start.x + (end.x - start.x) * localT;
car.y = start.y + (end.y - start.y) * localT;
```

**Mathematics**:

Global position → Segment index:
```
segment_index = ⌊position × num_segments⌋
```

Local position in segment:
```
t_local = (position × num_segments) mod 1
```

Linear interpolation (LERP):
```
P(t) = P₀ + (P₁ - P₀) × t
```

**Complexity**: O(1) - direct waypoint access

#### 3. Acceleration Physics

```javascript
const speedDiff = targetSpeed - currentSpeed;
const accelFactor = speedDiff > 0 ? waypoint.accel : waypoint.brake;
const maxChange = accelFactor * 20;
const speedChange = Math.max(-maxChange, Math.min(maxChange, speedDiff * 0.1));
car.speed += speedChange;
```

**Simplified Physical Model**:

```
a(t) = k × (v_target - v_current)

where:
  k = damping factor (0.1)
  |Δv| ≤ accelFactor × 20 km/h

First-order system with saturation
```

**Graphically**:
```
v_current
    ↑
    │      Asymptote → v_target
    │    ╱
    │   ╱
    │  ╱
    │ ╱_______________
    └────────────────→ t
```

#### 4. Speed-Distance Conversion

```javascript
const metersPerSecond = (car.speed * 1000) / 3600;
const distanceThisFrame = metersPerSecond * deltaTime;
car.position += distanceThisFrame / totalCircuitLength;
```

**Conversions**:

```
v_ms = v_kmh × (1000 m / 1 km) × (1 h / 3600 s)
     = v_kmh × 1000 / 3600
     = v_kmh / 3.6

distance = v × Δt  (basic kinematic equation)

normalized_position = distance / circuit_length
```

**Numerical Example**:
```
v = 180 km/h = 50 m/s
Δt = 0.0167s (60 FPS)
distance = 50 × 0.0167 = 0.835 m

If circuit = 3337 m:
Δposition = 0.835 / 3337 ≈ 0.00025 (0.025%)
```

### Possible Optimizations

```javascript
// Cache trigonometric calculations
const angleCache = new Map();

function getCachedAngle(dx, dy) {
    const key = `${dx.toFixed(2)},${dy.toFixed(2)}`;
    if (!angleCache.has(key)) {
        angleCache.set(key, Math.atan2(dy, dx));
    }
    return angleCache.get(key);
}

// Vectorization (if WebAssembly)
// Process all vehicles in parallel with SIMD
```

---

## 3. Circuit Position Interpolation

### Source Code

```javascript
// Linear interpolation (LERP)
const interpolateLinear = (start, end, t) => {
    return start + (end - start) * t;
};

// Catmull-Rom interpolation for smooth curves
const interpolateCatmullRom = (p0, p1, p2, p3, t) => {
    const t2 = t * t;
    const t3 = t2 * t;
    
    return 0.5 * (
        (2 * p1) +
        (-p0 + p2) * t +
        (2*p0 - 5*p1 + 4*p2 - p3) * t2 +
        (-p0 + 3*p1 - 3*p2 + p3) * t3
    );
};

// Cubic Bézier interpolation
const interpolateBezier = (p0, p1, p2, p3, t) => {
    const u = 1 - t;
    const tt = t * t;
    const uu = u * u;
    const uuu = uu * u;
    const ttt = tt * t;
    
    return uuu * p0 + 
           3 * uu * t * p1 + 
           3 * u * tt * p2 + 
           ttt * p3;
};
```

### Method Comparison

| Method | Continuity | Complexity | Usage |
|---------|------------|------------|-------|
| Linear | C⁰ | O(1) | Current implementation, simple |
| Catmull-Rom | C¹ | O(1) | Smooth curves, passes through points |
| Bézier | C¹ | O(1) | Precise curvature control |
| Spline | C² | O(n) | Maximum smoothness |

### Improved Implementation

```javascript
function updateCarWithSmoothPath(car) {
    const numSegments = circuitData.length - 1;
    const progress = car.position % 1.0;
    const segmentIndex = Math.floor(progress * numSegments);
    const localT = (progress * numSegments) % 1;
    
    // Get 4 points for Catmull-Rom
    const i0 = (segmentIndex - 1 + circuitData.length) % circuitData.length;
    const i1 = segmentIndex;
    const i2 = (segmentIndex + 1) % circuitData.length;
    const i3 = (segmentIndex + 2) % circuitData.length;
    
    const p0 = circuitData[i0];
    const p1 = circuitData[i1];
    const p2 = circuitData[i2];
    const p3 = circuitData[i3];
    
    // Smooth interpolation
    car.x = interpolateCatmullRom(p0.x, p1.x, p2.x, p3.x, localT);
    car.y = interpolateCatmullRom(p0.y, p1.y, p2.y, p3.y, localT);
    
    // Calculate tangent for more precise angle
    const tx = 0.5 * (-p0.x + p2.x + 2*localT*(-2*p0.x + 5*p1.x - 4*p2.x + p3.x));
    const ty = 0.5 * (-p0.y + p2.y + 2*localT*(-2*p0.y + 5*p1.y - 4*p2.y + p3.y));
    car.angle = Math.atan2(ty, tx);
}
```

---

## 4. Data Bucketing for 3D Visualization

### Source Code (lines 659-836)

```javascript
function updatePlot() {
    const { speedHistory } = simState.current;
    
    if (speedHistory.length < 10) {
        log('Not enough data for graph', 'warning');
        return;
    }
    
    // 1. BUCKETING BY SPEED (10 km/h intervals)
    const speedBuckets = {};
    
    speedHistory.forEach(point => {
        const bucket = Math.floor(point.speed / 10) * 10;
        
        if (!speedBuckets[bucket]) {
            speedBuckets[bucket] = [];
        }
        
        speedBuckets[bucket].push(point.accel);
    });
    
    // 2. CALCULATE STATISTICS PER BUCKET
    const speeds = [];
    const avgAccels = [];
    const counts = [];
    
    Object.keys(speedBuckets).sort((a, b) => a - b).forEach(bucket => {
        const accels = speedBuckets[bucket];
        
        speeds.push(parseFloat(bucket));
        
        // Arithmetic mean
        const avgAccel = accels.reduce((sum, a) => sum + a, 0) / accels.length;
        avgAccels.push(avgAccel);
        
        counts.push(accels.length);
    });
    
    // 3. CAMERA PRESERVATION
    let preservedCamera = null;
    if (plotRef.current?.layout?.scene?.camera) {
        preservedCamera = JSON.parse(
            JSON.stringify(plotRef.current.layout.scene.camera)
        );
    }
    
    // 4. CREATE 3D TRACE
    const trace = {
        x: speeds,
        y: avgAccels,
        z: counts,
        mode: 'markers',
        type: 'scatter3d',
        marker: {
            size: 6,
            color: speeds,
            colorscale: 'Viridis',
            colorbar: {
                title: 'Speed (km/h)',
                thickness: 15,
                len: 0.7
            },
            showscale: true
        },
        text: speeds.map((s, i) => 
            `Speed: ${s} km/h<br>` +
            `Accel: ${avgAccels[i].toFixed(2)} m/s²<br>` +
            `Frequency: ${counts[i]}`
        ),
        hovertemplate: '%{text}<extra></extra>'
    };
    
    // 5. LAYOUT CONFIGURATION
    const layout = {
        scene: {
            xaxis: { 
                title: 'Speed (km/h)',
                gridcolor: 'rgba(255,255,255,0.1)'
            },
            yaxis: { 
                title: 'Avg. Acceleration (m/s²)',
                gridcolor: 'rgba(255,255,255,0.1)'
            },
            zaxis: { 
                title: 'Frequency',
                gridcolor: 'rgba(255,255,255,0.1)'
            },
            camera: preservedCamera || {
                eye: { x: 1.5, y: 1.5, z: 1.5 },
                center: { x: 0, y: 0, z: 0 },
                up: { x: 0, y: 0, z: 1 }
            },
            bgcolor: 'rgba(0,0,0,0)'
        },
        paper_bgcolor: 'rgba(0,0,0,0)',
        plot_bgcolor: 'rgba(0,0,0,0)',
        font: { color: '#ffffff' },
        margin: { l: 0, r: 0, t: 0, b: 0 }
    };
    
    // 6. RENDER WITH PLOTLY
    Plotly.react('plot3d', [trace], layout, {
        responsive: true,
        displayModeBar: false
    });
}
```

### Bucketing Algorithm Analysis

**Complexity**:
- Bucketing: O(n) where n = speedHistory size
- Key sorting: O(k log k) where k = number of buckets
- Mean calculation: O(n) total
- **Total**: O(n + k log k) ≈ O(n) since k << n

**Space**: O(n) for temporary bucket storage

### Bucketing Visualization

```
speedHistory: [
    {speed: 92, accel: 1.2},
    {speed: 95, accel: 0.8},
    {speed: 103, accel: -0.5},
    {speed: 98, accel: 1.0},
    ...
]
        ↓
    Bucketing (10 km/h)
        ↓
speedBuckets: {
    90: [1.2, 0.8, 1.0],     → avg: 1.0, count: 3
    100: [-0.5],              → avg: -0.5, count: 1
    ...
}
        ↓
    Aggregation
        ↓
trace: {
    x: [90, 100, ...],        // Speeds
    y: [1.0, -0.5, ...],      // Average accelerations
    z: [3, 1, ...]            // Frequencies
}
```

### Improvement: Adaptive Bucketing

```javascript
function adaptiveBucketing(data, targetBuckets = 20) {
    // Calculate speed range
    const speeds = data.map(d => d.speed);
    const minSpeed = Math.min(...speeds);
    const maxSpeed = Math.max(...speeds);
    const range = maxSpeed - minSpeed;
    
    // Adaptive bucket size
    const bucketSize = Math.ceil(range / targetBuckets);
    
    const buckets = {};
    
    data.forEach(point => {
        const bucket = Math.floor((point.speed - minSpeed) / bucketSize) * bucketSize + minSpeed;
        
        if (!buckets[bucket]) {
            buckets[bucket] = [];
        }
        
        buckets[bucket].push(point.accel);
    });
    
    return buckets;
}
```

---

## 5. Circular Buffer Logging System

### Source Code (lines 341-404)

```javascript
const debugLogs = [];
const MAX_DEBUG_LOGS = 100;

function log(message, type = 'info') {
    const timestamp = new Date().toLocaleTimeString();
    const logEntry = {
        timestamp,
        message,
        type
    };
    
    // Add to end
    debugLogs.push(logEntry);
    
    // Remove oldest if overflow (FIFO)
    if (debugLogs.length > MAX_DEBUG_LOGS) {
        debugLogs.shift();
    }
    
    // Update UI
    updateDebugPanel();
    
    // Console log for debugging
    const consoleMethod = type === 'error' ? 'error' : 
                         type === 'warning' ? 'warn' : 'log';
    console[consoleMethod](`[${timestamp}] ${message}`);
}

function updateDebugPanel() {
    const panel = document.getElementById('debug-logs');
    if (!panel) return;
    
    // Generate HTML logs (reverse order = newest on top)
    panel.innerHTML = debugLogs
        .slice()
        .reverse()
        .map(log => `
            <div class="debug-log ${log.type}">
                <span style="color: #888;">[${log.timestamp}]</span>
                ${log.message}
            </div>
        `)
        .join('');
    
    // Auto-scroll to bottom if panel visible
    if (panel.parentElement.classList.contains('show')) {
        panel.scrollTop = 0; // Top because reversed order
    }
}
```

### Data Structure: Circular Buffer

**Concept**:

```
Capacity: 100
Size: 100 (full)

[0][1][2]...[97][98][99]
 ↑                    ↑
head                 tail

Add new element:
1. push(new_element)    → [0][1][2]...[98][99][new]
2. shift()             → [1][2][3]...[99][new]
                          ↑                   ↑
                         head                tail
```

**Advantages**:
- Constant memory O(1)
- Amortized O(1) insertion
- O(1) deletion
- Always keeps last N logs

**Alternative: Classic Ring Buffer**

```javascript
class RingBuffer {
    constructor(capacity) {
        this.buffer = new Array(capacity);
        this.capacity = capacity;
        this.size = 0;
        this.head = 0;
        this.tail = 0;
    }
    
    push(item) {
        this.buffer[this.tail] = item;
        this.tail = (this.tail + 1) % this.capacity;
        
        if (this.size < this.capacity) {
            this.size++;
        } else {
            // Overwrite oldest
            this.head = (this.head + 1) % this.capacity;
        }
    }
    
    get(index) {
        if (index >= this.size) return null;
        return this.buffer[(this.head + index) % this.capacity];
    }
    
    toArray() {
        const result = [];
        for (let i = 0; i < this.size; i++) {
            result.push(this.get(i));
        }
        return result;
    }
}
```

**Performance**:
- Array.shift(): O(n) due to element shifting
- Ring buffer: Pure O(1) for all operations
- For 100 elements: negligible
- For 10000+: ring buffer preferable

---

## 6. State Management with React Refs

### Source Code (lines 847-1006)

```javascript
const CircuitRacingSimulation = () => {
    // State for UI reactivity
    const [isRunning, setIsRunning] = useState(false);
    
    // Refs for DOM references
    const canvasRef = useRef(null);
    const plotRef = useRef(null);
    
    // Ref for non-reactive mutable state
    const simState = useRef({
        playerCar: {
            position: 0,
            speed: 0,
            targetSpeed: 180,
            x: 100,
            y: 280,
            angle: 0
        },
        otherCars: Array(8).fill(0).map((_, i) => ({
            position: (i + 1) * 0.11,
            speed: 120 + Math.random() * 100,
            x: undefined,
            y: undefined,
            angle: undefined,
            color: `hsl(${i * 45}, 70%, 50%)`
        })),
        speedHistory: [],
        time: 0,
        lap: 0
    });
    
    const animationId = useRef(null);
    
    // Effect for animation lifecycle management
    useEffect(() => {
        if (isRunning) {
            const animate = () => {
                updateSimulation();
                drawCanvas();
                
                if (simState.current.time % 0.5 < 0.016) {
                    updatePlot();
                }
                
                animationId.current = requestAnimationFrame(animate);
            };
            
            animate();
        } else {
            if (animationId.current) {
                cancelAnimationFrame(animationId.current);
            }
        }
        
        // Cleanup on unmount or when isRunning changes
        return () => {
            if (animationId.current) {
                cancelAnimationFrame(animationId.current);
            }
        };
    }, [isRunning]);
    
    // ...
};
```

### Analysis: useState vs useRef

| Aspect | useState | useRef |
|--------|----------|--------|
| **Reactivity** | Triggers re-render | No re-render |
| **Persistence** | Between renders | Between renders |
| **Usage** | UI state | Mutable values, DOM refs |
| **Performance** | Re-render cost | O(1) mutation |
| **Example** | isRunning (button) | simState (high frequency) |

### Why useRef for simState?

```javascript
// ❌ BAD - with useState
const [simState, setSimState] = useState({...});

function updateSimulation() {
    // This would trigger a re-render at 60 FPS!
    setSimState(prev => ({
        ...prev,
        time: prev.time + deltaTime
    }));
}
// Result: 60 re-renders/second = lag

// ✅ GOOD - with useRef
const simState = useRef({...});

function updateSimulation() {
    // Direct mutation, no re-render
    simState.current.time += deltaTime;
}
// Result: 0 re-render for simulation updates
```

### Pattern: Refs for High Frequency

```javascript
// General principle
const highFreqData = useRef(initialValue);

function highFreqUpdate() {
    // O(1) mutation without re-render
    highFreqData.current = newValue;
}

// Only UI changes trigger re-render
const [uiState, setUiState] = useState(initialUI);
```

---

## 7. Performance Optimizations

### Applied Techniques

#### 1. RequestAnimationFrame for Synchronization

```javascript
function animate() {
    // Sync with refresh rate (typically 60 Hz)
    updateSimulation();
    drawCanvas();
    
    animationId.current = requestAnimationFrame(animate);
}
```

**Advantages**:
- Sync with monitor V-Sync
- No calculations when tab inactive (CPU savings)
- Precise and stable timing

**Alternative to Avoid**:
```javascript
// ❌ Bad
setInterval(() => {
    updateSimulation();
    drawCanvas();
}, 16.67); // Imprecise, no V-Sync
```

#### 2. 3D Graph Throttling

```javascript
if (simState.current.time % 0.5 < 0.016) {
    updatePlot();  // Update only 2×/second
}
```

**Calculation**:
```
Canvas FPS: 60
Plotly FPS: 2
Ratio: 30:1

CPU savings: ~97% on 3D rendering
```

#### 3. History Limitation (FIFO)

```javascript
speedHistory.push(newPoint);

if (speedHistory.length > 1000) {
    speedHistory.shift();  // O(n) but n=1000 acceptable
}
```

**Memory**:
```
1 point = 2 × 8 bytes (float64) = 16 bytes
1000 points = 16 KB
vs infinity = eventual crash
```

#### 4. Canvas Double Buffering (Native)

The browser automatically does:

```
Buffer 1 (display) ←→ Buffer 2 (draw)

While Buffer 1 displays:
  - clear Buffer 2
  - draw in Buffer 2

Next frame:
  - swap(Buffer 1, Buffer 2)  // Atomic, no flickering
```

#### 5. Plotly Camera Preservation

```javascript
// Save deep copy
let preservedCamera = null;
if (plotRef.current?.layout?.scene?.camera) {
    preservedCamera = JSON.parse(
        JSON.stringify(plotRef.current.layout.scene.camera)
    );
}

// Restore
const layout = {
    scene: {
        camera: preservedCamera || defaultCamera
    }
};
```

**Gain**: Avoids 3D matrix recalculation (expensive in WebGL)

### Profiling Suggestions

```javascript
// Add in updateSimulation
console.time('updateSimulation');
// ... code ...
console.timeEnd('updateSimulation');

// Add in drawCanvas
console.time('drawCanvas');
// ... code ...
console.timeEnd('drawCanvas');

// Add in updatePlot
console.time('updatePlot');
// ... code ...
console.timeEnd('updatePlot');
```

**Performance Targets**:
- updateSimulation: < 5ms
- drawCanvas: < 10ms
- updatePlot: < 50ms (throttled)

---

## Conclusion

This code demonstrates several advanced patterns:

1. **Async/await with retry logic** for robustness
2. **Real-time physics** with frame-rate independence
3. **Mathematical interpolation** for smooth movement
4. **Data bucketing** for efficient aggregation
5. **Circular buffer** for constant memory
6. **useRef for performance** avoiding re-renders
7. **Throttling and optimizations** for stable 60 FPS

These techniques are transferable to other simulation, data visualization, or real-time application projects.
