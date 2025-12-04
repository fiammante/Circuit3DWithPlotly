# Quick Reference Guide - Monaco Circuit Simulation

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         HTML DOCUMENT                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    LOADING SYSTEM                           │ │
│  │  • LoadingManager: Async CDN resource management            │ │
│  │  • Resource checkers with retry logic                      │ │
│  │  • Real-time UI feedback                                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                     DEBUG SYSTEM                            │ │
│  │  • Logger with levels (info/warning/error/success)         │ │
│  │  • Circular buffer (100 entries max)                       │ │
│  │  • Toggle UI for developers                                │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              REACT APPLICATION (JSX)                        │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │    CircuitRacingSimulation Component                 │  │ │
│  │  │                                                       │  │ │
│  │  │  ┌─────────────────┐    ┌──────────────────────┐   │  │ │
│  │  │  │  STATE HOOKS    │    │   REFS               │   │  │ │
│  │  │  │  • isRunning    │    │   • canvasRef        │   │  │ │
│  │  │  └─────────────────┘    │   • plotRef          │   │  │ │
│  │  │                          │   • simState         │   │  │ │
│  │  │                          │   • animationId      │   │  │ │
│  │  │                          └──────────────────────┘   │  │ │
│  │  │                                                       │  │ │
│  │  │  ┌─────────────────────────────────────────────┐   │  │ │
│  │  │  │         PHYSICS ENGINE                       │   │  │ │
│  │  │  │  ┌───────────────────────────────────────┐  │   │  │ │
│  │  │  │  │  updateSimulation()                   │  │   │  │ │
│  │  │  │  │  • Position interpolation             │  │   │  │ │
│  │  │  │  │  • Speed adjustment (accel/brake)     │  │   │  │ │
│  │  │  │  │  • Angle calculation                  │  │   │  │ │
│  │  │  │  │  • Lap detection                      │  │   │  │ │
│  │  │  │  │  • History tracking                   │  │   │  │ │
│  │  │  │  └───────────────────────────────────────┘  │   │  │ │
│  │  │  └─────────────────────────────────────────────┘   │  │ │
│  │  │                                                       │  │ │
│  │  │  ┌─────────────────────────────────────────────┐   │  │ │
│  │  │  │       2D CANVAS RENDERER                     │   │  │ │
│  │  │  │  ┌───────────────────────────────────────┐  │   │  │ │
│  │  │  │  │  drawCanvas()                         │  │   │  │ │
│  │  │  │  │  • Clear frame                        │  │   │  │ │
│  │  │  │  │  • Draw circuit waypoints             │  │   │  │ │
│  │  │  │  │  • Draw start/finish line             │  │   │  │ │
│  │  │  │  │  • Draw cars (player + 8 opponents)   │  │   │  │ │
│  │  │  │  │  • Draw HUD (speed, lap, time)        │  │   │  │ │
│  │  │  │  └───────────────────────────────────────┘  │   │  │ │
│  │  │  └─────────────────────────────────────────────┘   │  │ │
│  │  │                                                       │  │ │
│  │  │  ┌─────────────────────────────────────────────┐   │  │ │
│  │  │  │     3D VISUALIZATION (Plotly)                │   │  │ │
│  │  │  │  ┌───────────────────────────────────────┐  │   │  │ │
│  │  │  │  │  updatePlot()                         │  │   │  │ │
│  │  │  │  │  • Bucket speeds (10 km/h intervals)  │  │   │  │ │
│  │  │  │  │  • Compute avg acceleration           │  │   │  │ │
│  │  │  │  │  • Count occurrences                  │  │   │  │ │
│  │  │  │  │  • Preserve camera position           │  │   │  │ │
│  │  │  │  │  • Render 3D scatter plot             │  │   │  │ │
│  │  │  │  └───────────────────────────────────────┘  │   │  │ │
│  │  │  └─────────────────────────────────────────────┘   │  │ │
│  │  │                                                       │  │ │
│  │  │  ┌─────────────────────────────────────────────┐   │  │ │
│  │  │  │       ANIMATION LOOP                         │   │  │ │
│  │  │  │  requestAnimationFrame @ 60 FPS              │   │  │ │
│  │  │  │  ├─> updateSimulation()                      │   │  │ │
│  │  │  │  ├─> drawCanvas()                            │   │  │ │
│  │  │  │  └─> updatePlot() (throttled @ 0.5s)        │   │  │ │
│  │  │  └─────────────────────────────────────────────┘   │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 CIRCUIT DEFINITION                          │ │
│  │  • 36 waypoints with physical properties                   │ │
│  │  • 2D positions, target speeds, accel/brake factors        │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Flow

```
User Action (Click "Start")
    ↓
toggleSimulation()
    ↓
setIsRunning(true)
    ↓
useEffect([isRunning]) triggered
    ↓
┌─────────────────────────────────────────────┐
│        ANIMATION LOOP STARTS                │
│  requestAnimationFrame(animate)             │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │  1. updateSimulation()                │ │
│  │     ├─> Calculate deltaTime           │ │
│  │     ├─> Update player car             │ │
│  │     │   ├─> Interpolate position      │ │
│  │     │   ├─> Calculate angle           │ │
│  │     │   ├─> Adjust speed              │ │
│  │     │   └─> Update position           │ │
│  │     ├─> Update 8 opponent cars        │ │
│  │     ├─> Increment time                │ │
│  │     └─> Track speed history           │ │
│  └───────────────────────────────────────┘ │
│            ↓                                │
│  ┌───────────────────────────────────────┐ │
│  │  2. drawCanvas()                      │ │
│  │     ├─> Clear canvas                  │ │
│  │     ├─> Draw circuit                  │ │
│  │     ├─> Draw finish line              │ │
│  │     ├─> Draw all cars                 │ │
│  │     └─> Draw HUD info                 │ │
│  └───────────────────────────────────────┘ │
│            ↓                                │
│  ┌───────────────────────────────────────┐ │
│  │  3. updatePlot() (every 0.5s)         │ │
│  │     ├─> Bucket speed data             │ │
│  │     ├─> Calculate statistics          │ │
│  │     ├─> Preserve camera               │ │
│  │     └─> Update Plotly 3D              │ │
│  └───────────────────────────────────────┘ │
│            ↓                                │
│  requestAnimationFrame(animate) → Loop     │
└─────────────────────────────────────────────┘
```

---

## Quick API Reference for Main Functions

### LoadingManager

```javascript
// Construction
const loader = new LoadingManager();

// Check a resource
await loader.checkResource(
    'react',                          // name
    () => typeof React !== 'undefined', // test function
    50,                               // max retries
    100                               // delay (ms)
);

// Check all resources
await loader.checkAllResources();

// Hide loading screen
loader.hideLoadingScreen();
```

### Debug Logger

```javascript
// Log a message
log('Message', 'info');      // Types: info, warning, error, success
log('Warning!', 'warning');
log('Critical error', 'error');
log('Operation successful', 'success');

// Toggle debug panel
toggleDebugPanel();  // Called by UI button
```

### Simulation State

```javascript
// Structure of simState.current
{
    playerCar: {
        position: 0,        // [0, 1]
        speed: 0,           // km/h
        targetSpeed: 180,   // km/h
        x: 100,             // px
        y: 280,             // px
        angle: 0            // radians
    },
    otherCars: Array(8),    // Same structure as playerCar
    speedHistory: [{
        speed: number,
        accel: number
    }],
    time: 0,                // seconds
    lap: 0                  // lap count
}
```

### Physics Functions

```javascript
// Update simulation (called every frame)
updateSimulation()
    → Calculates deltaTime
    → Updates positions/speeds
    → Handles lap detection
    → Records history

// Key formulas
const metersPerSecond = (speedKmh * 1000) / 3600;
const distance = metersPerSecond * deltaTime;
const newPosition = (oldPosition + distance / totalLength) % 1.0;

// Position interpolation
const segmentT = (progress * numSegments) % 1;
const x = startX + (endX - startX) * segmentT;
const y = startY + (endY - startY) * segmentT;

// Angle calculation
const angle = Math.atan2(dy, dx);
```

### Canvas Rendering

```javascript
// Draw canvas (called every frame)
drawCanvas()
    → ctx.clearRect(0, 0, width, height)
    → drawCircuitWaypoints()
    → drawStartFinishLine()
    → drawCars()
    → drawHUD()

// Draw a car
function drawCar(car, color, isPlayer) {
    ctx.save();
    ctx.translate(car.x, car.y);
    ctx.rotate(car.angle);
    ctx.fillStyle = color;
    ctx.fillRect(-w/2, -h/2, w, h);
    ctx.restore();
}
```

### 3D Visualization

```javascript
// Update 3D graph
updatePlot()
    → Bucket by speed (10 km/h)
    → Calculate average acceleration
    → Count occurrences
    → Preserve camera position
    → Plotly.react()

// Plotly trace structure
{
    x: speeds,        // Array of speeds
    y: avgAccels,     // Array of accelerations
    z: counts,        // Array of frequencies
    mode: 'markers',
    type: 'scatter3d',
    marker: {
        size: 6,
        color: speeds,
        colorscale: 'Viridis'
    }
}
```

---

## Circuit Waypoints - Key Points

| ID | Name | Coordinates | Speed | Type |
|----|------|-------------|-------|------|
| 1 | Start/Finish | (100, 280) | 180 | Acceleration |
| 6 | Sainte-Dévote | (240, 100) | 100 | Heavy braking |
| 9 | Casino Climb | (450, 70) | 140 | Fast turn |
| 15 | Grand Hotel | (650, 220) | 60 | Slow hairpin |
| 19 | Tunnel | (450, 380) | 280 | Max speed |
| 22 | Nouvelle Chicane | (250, 330) | 140 | Technical |
| 27 | Tabac | (120, 180) | 110 | Braking |
| 30 | Swimming Pool | (150, 50) | 120 | Chicane |
| 33 | Rascasse | (180, 130) | 80 | Braking |

---

## Important Constants

```javascript
// Simulation
const NUM_OTHER_CARS = 8;
const SPEED_HISTORY_LIMIT = 1000;
const PLOT_UPDATE_INTERVAL = 0.5;  // seconds
const SPEED_BUCKET_SIZE = 10;      // km/h

// Canvas
const CANVAS_WIDTH = 800;
const CANVAS_HEIGHT = 400;

// Physics
const MAX_ACCEL_CHANGE = 20;       // km/h per frame
const SPEED_UPDATE_RATE = 0.1;     // smoothing factor

// Performance
const TARGET_FPS = 60;
const FRAME_TIME = 1000 / 60;      // ~16.67 ms
```

---

## React Events and Hooks

```javascript
// Hooks used
const [isRunning, setIsRunning] = useState(false);
const canvasRef = useRef(null);
const plotRef = useRef(null);
const simState = useRef({...});
const animationId = useRef(null);

// Effects
useEffect(() => {
    // Start/stop animation based on isRunning
    if (isRunning) {
        animate();
    } else {
        cancelAnimationFrame(animationId.current);
    }
    
    return () => cancelAnimationFrame(animationId.current);
}, [isRunning]);

useEffect(() => {
    // Initialize 3D graph on mount
    updatePlot();
}, []);

// Event handler
const toggleSimulation = () => {
    // Check canvas
    // Reset if necessary
    // Toggle isRunning
};
```

---

## Performance Tips

### Applied Optimizations

1. **RequestAnimationFrame**: Native sync with screen refresh
2. **3D Throttling**: Update every 0.5s instead of every frame
3. **History limiting**: FIFO with max 1000 entries
4. **Camera preservation**: Avoids expensive 3D recalculations
5. **Canvas buffering**: Native browser double buffering
6. **React Production**: Minified build without DevTools

### Expected Metrics

- **Target FPS**: 60 FPS constant
- **CPU usage**: 15-25% (single core)
- **Memory**: ~50-80 MB
- **3D rendering**: <5ms per update
- **Canvas draw**: <10ms per frame

---

## Debugging Checklist

### Initial Checks

```
✓ Loading screen displayed
✓ All resources marked "loaded" (green)
✓ "Application loaded" message
✓ Canvas visible and sized correctly
✓ 3D graph initialized (axes visible)
```

### During Simulation

```
✓ Button shows "⏸ Pause" when running
✓ Green car moves on circuit
✓ 8 opponent cars visible
✓ HUD displays: speed, lap, time
✓ 3D graph populates with points
✓ 3D camera preserves orientation on updates
```

### If Problems Occur

1. Open debug panel (bottom-right button)
2. Check logs for errors (red)
3. Confirm sequence:
   - "All resources loaded"
   - "3D graph init"
   - "Toggle simulation: START"
   - "Updating simulation..."
4. Test in different browser
5. Check JavaScript console (F12)

---

## Possible Extensions

### Short Term (Easy)

- [ ] Player car color choice
- [ ] Best lap time display
- [ ] Pause/resume without reset
- [ ] CSV data export
- [ ] Canvas screenshots

### Medium Term (Moderate)

- [ ] Multiple circuits (Monza, Spa, etc.)
- [ ] Weather conditions (rain → grip)
- [ ] Tire wear (degradation)
- [ ] Pit stops
- [ ] Simplified collisions

### Long Term (Advanced)

- [ ] Real-time multiplayer (WebSockets)
- [ ] Intelligent AI opponents
- [ ] Advanced physics (suspension, aerodynamics)
- [ ] VR/3D immersive mode
- [ ] Online leaderboard with authentication

---

## Mathematical Formulas

### Speed Conversion

```
v_ms = (v_kmh × 1000) / 3600

Examples:
  180 km/h = 50 m/s
  100 km/h = 27.78 m/s
  280 km/h = 77.78 m/s
```

### Acceleration

```
a = Δv / Δt

where:
  Δv = v_target - v_current (km/h)
  Δt = deltaTime (s)
  
Limitation:
  |a| ≤ MAX_ACCEL_CHANGE × accel_factor
```

### Normalized Position

```
position_new = position_old + (distance / total_length)

where:
  distance = v_ms × deltaTime
  total_length = Σ(sqrt(dx² + dy²)) for all segments
  
Looping:
  if position >= 1.0: position -= 1.0
```

### Linear Interpolation

```
x(t) = x₀ + (x₁ - x₀) × t
y(t) = y₀ + (y₁ - y₀) × t

where:
  t ∈ [0, 1] (local segment parameter)
  (x₀, y₀) = start point
  (x₁, y₁) = end point
```

### Rotation Angle

```
θ = atan2(dy, dx)

where:
  dx = x₁ - x₀
  dy = y₁ - y₀
  
Result in radians ∈ [-π, π]
```

---

## File Structure if Extracted

If you extract components into separate files:

```
project/
├── index.html
├── styles/
│   ├── main.css
│   ├── loading.css
│   ├── debug.css
│   └── circuit.css
├── scripts/
│   ├── loadingManager.js
│   ├── debugLogger.js
│   ├── circuitData.js
│   ├── physics.js
│   └── plotUtils.js
├── components/
│   ├── CircuitRacingSimulation.jsx
│   ├── CircuitPanel.jsx
│   ├── PlotPanel.jsx
│   └── Legend.jsx
└── utils/
    ├── constants.js
    └── helpers.js
```

---

## Technology Summary

| Technology | Version | Usage | Criticality |
|------------|---------|-------|-------------|
| React | 18.x | UI Framework | ★★★★★ |
| ReactDOM | 18.x | DOM Rendering | ★★★★★ |
| Plotly.js | 2.27.0 | 3D Graphs | ★★★★☆ |
| Babel Standalone | Latest | JSX Transform | ★★★★★ |
| Canvas 2D API | Native | Graphics Render | ★★★★★ |
| RequestAnimationFrame | Native | Animation Loop | ★★★★★ |

**Legend**: ★★★★★ Essential | ★★★★☆ Very Important | ★★★☆☆ Important

---

## Contact and Support

For technical questions or bug reports:

1. Check main documentation
2. Consult debug panel
3. Examine browser console logs
4. Test in recent browser (Chrome 90+, Firefox 88+, Safari 14+)

**Code Complexity**: ~1500 lines  
**Required Level**: Intermediate to Advanced  
**Domains**: WebGL, Physics, DataViz, React Hooks
