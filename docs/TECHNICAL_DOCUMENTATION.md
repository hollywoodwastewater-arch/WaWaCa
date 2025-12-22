# ThewastewaterCalc.app (TWWC) - Technical Documentation

> **Version:** Stable v8-ADEQ
> **Last Updated:** December 2024
> **Purpose:** Professional Offline Wastewater Operator Calculator for Grades 1-4

---

## Table of Contents

1. [Architecture & Tech Stack](#1-architecture--tech-stack)
2. [Global Utilities & UX Patterns](#2-global-utilities--ux-patterns)
3. [Mathematical Modules (Deep Dive)](#3-mathematical-modules-deep-dive)
4. [Component Hierarchy](#4-component-hierarchy)

---

## 1. Architecture & Tech Stack

### 1.1 Core Framework: React 18 + Babel (CDN-Based)

The application is a **single-file SPA** that uses CDN-delivered dependencies with in-browser JSX compilation:

```html
<!-- React & ReactDOM -->
<script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
<script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>

<!-- Babel for in-browser JSX compilation -->
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
```

**Implementation Details:**
- All React code is written inside a `<script type="text/babel">` block
- Babel transforms JSX to JavaScript at runtime in the browser
- Uses React 18's production build for performance
- Leverages `ReactDOM.createRoot()` API (React 18 concurrent rendering)

**React Hooks Used:**
- `useState` - Component state management
- `useEffect` - Side effects and lifecycle
- `useMemo` - Memoized calculations (prevents recalculation on every render)
- `useRef` - DOM references for Lucide icon injection

---

### 1.2 Styling: Tailwind CSS

**CDN Implementation:**
```html
<script src="https://cdn.tailwindcss.com"></script>
```

**Custom Color Palette (Cyan/Slate Theme):**
```javascript
tailwind.config = {
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#ecfeff',   // Lightest cyan
          100: '#cffafe',
          500: '#06b6d4',  // Primary cyan
          600: '#0891b2',
          700: '#0e7490',  // Theme color (meta tag)
          900: '#164e63'   // Darkest cyan
        }
      }
    }
  }
};
```

**Design System Characteristics:**
- High-contrast colors for field visibility (important for treatment plant environments)
- Large touch targets for glove-friendly operation
- Slate gray (`slate-*`) for neutral UI elements
- Cyan (`primary-*`) for interactive/highlight elements

---

### 1.3 PWA Features

#### 1.3.1 Web App Manifest

The manifest is embedded inline as a data URI:

```json
{
  "name": "ThewastewaterCalc.app",
  "short_name": "TWWC",
  "start_url": ".",
  "display": "standalone",
  "background_color": "#ecfeff",
  "theme_color": "#0e7490",
  "orientation": "portrait-primary",
  "icons": [{
    "src": "https://cdn.jsdelivr.net/npm/lucide-static@latest/icons/waves.svg",
    "sizes": "192x192",
    "type": "image/svg+xml"
  }]
}
```

**Key PWA Meta Tags:**
```html
<meta name="theme-color" content="#0e7490" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<meta name="apple-mobile-web-app-title" content="TWWC" />
```

#### 1.3.2 Service Worker Logic (`swCode`)

The Service Worker is dynamically created via Blob URL:

```javascript
const swCode = `
    const CACHE_NAME = 'twwc-stable-v8-adeq';
    const urlsToCache = [
        './',
        'https://cdn.tailwindcss.com',
        'https://unpkg.com/react@18/umd/react.production.min.js',
        'https://unpkg.com/react-dom@18/umd/react-dom.production.min.js',
        'https://unpkg.com/@babel/standalone/babel.min.js',
        'https://unpkg.com/lucide@latest'
    ];

    // Install: Cache all critical resources
    self.addEventListener('install', event => {
        self.skipWaiting();
        event.waitUntil(
            caches.open(CACHE_NAME).then(cache => cache.addAll(urlsToCache))
        );
    });

    // Fetch: Cache-First Strategy
    self.addEventListener('fetch', event => {
        event.respondWith(
            caches.match(event.request).then(response =>
                response || fetch(event.request)
            )
        );
    });

    // Activate: Clean up old caches
    self.addEventListener('activate', event => {
        event.waitUntil(
            caches.keys().then(names =>
                Promise.all(names.map(name => {
                    if(name !== CACHE_NAME) return caches.delete(name);
                }))
            )
        );
    });
`;
```

**Caching Strategy: Cache-First**
1. On `install`: Pre-caches all CDN dependencies and the root document
2. On `fetch`: Checks cache first, falls back to network if not found
3. On `activate`: Purges old cache versions (any cache not matching `twwc-stable-v8-adeq`)

**Registration Method:**
```javascript
const blob = new Blob([swCode], {type: 'text/javascript'});
const swUrl = URL.createObjectURL(blob);
window.addEventListener('load', () =>
    navigator.serviceWorker.register(swUrl).catch(console.error)
);
```

---

### 1.4 Persistence: `useLocalStorage` Hook

Custom hook for persisting state across sessions:

```javascript
const useLocalStorage = (key, initialValue) => {
  const [value, setValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch { return initialValue; }
  });

  const setStoredValue = (newValue) => {
    try {
      const nextVal = typeof newValue === 'function'
        ? newValue(value)
        : newValue;
      setValue(nextVal);
      window.localStorage.setItem(key, JSON.stringify(nextVal));
    } catch (e) { console.error(e); }
  };

  return [value, setStoredValue];
};
```

**Usage in App:**
| Key | Purpose |
|-----|---------|
| `twwc_vol` | Stores last calculated tank volume (MG, area, ft³) for cross-tool use |
| `twwc_conv_val` | Unit converter input value |
| `twwc_conv_cat` | Unit converter category (flow/vol/len) |
| `twwc_conv_u1` | Unit converter source unit |
| `twwc_conv_u2` | Unit converter target unit |

---

### 1.5 Security & Hardening Vectors

The code contains explicit hardening comments marking defensive patterns:

#### **Vector I: Debounce**
**Purpose:** Prevent excessive recalculations during rapid input

```javascript
// HARDENING: Vector I - Debounce inputs
const useDebounce = (value, delay) => {
  const [debouncedValue, setDebouncedValue] = useState(value);
  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);
  return debouncedValue;
};
```

**Implementation:** All calculator inputs use 300ms debounce before triggering calculations.

---

#### **Vector II: Text Wrapping**
**Purpose:** Prevent layout issues from unwrapped text nodes

```javascript
// HARDENING: Vector II - Wrap text in span
<label className="...">
  <span>{label}</span>  {/* Text wrapped in span */}
  {help && <span title={help}>...</span>}
</label>
```

---

#### **Vector III: Iconography Conflict (Safe Render)**
**Purpose:** Prevent conflicts with Lucide icon library's DOM manipulation

```javascript
// HARDENING: Vector III - Iconography Conflict (Safe Render)
const Icon = ({ name, className = "w-4 h-4" }) => {
    const ref = useRef(null);
    useEffect(() => {
        if (ref.current && window.lucide) {
           const temp = document.createElement('div');
           const iconName = name.replace(/([A-Z])/g, "-$1")
                               .toLowerCase().replace(/^-/, "");
           temp.innerHTML = `<i data-lucide="${iconName}"></i>`;
           window.lucide.createIcons({ root: temp });
           const svg = temp.querySelector('svg');
           if (svg) {
               svg.setAttribute('class', className);
               ref.current.innerHTML = '';
               ref.current.appendChild(svg);
           }
        }
    }, [name, className]);
    return <span ref={ref} className="inline-flex items-center justify-center" />;
};
```

**Why This Exists:**
- Lucide's `createIcons()` mutates the DOM directly
- React's virtual DOM can conflict with direct DOM manipulation
- Solution: Render icons in a temporary detached element, then inject the SVG

---

#### **Vector IV: Browser Quirks**
**Purpose:** Handle input behavior inconsistencies across browsers

**CSS Hardening:**
```css
/* HARDENING: Vector IV - Browser Quirks */
html, body {
    overscroll-behavior: none;      /* Prevent pull-to-refresh */
    touch-action: pan-y;            /* Limit touch gestures */
    height: 100%;
    overflow: hidden;
    -webkit-user-select: none;      /* Prevent text selection */
    user-select: none;
    -webkit-tap-highlight-color: transparent;  /* Remove tap highlight */
}

input, textarea {
    -webkit-user-select: text;      /* Re-enable for inputs */
    user-select: text;
}

/* Hide number input spinners */
input[type="number"] { -moz-appearance: textfield; }
input[type="number"]::-webkit-outer-spin-button,
input[type="number"]::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
}
```

**JavaScript Hardening (InputGroup):**
```javascript
/* HARDENING: Vector IV - Input Specifications */
onKeyDown={(e) => {
    // Block scientific notation and sign characters
    if (["e", "E", "+", "-"].includes(e.key)) e.preventDefault();
    // Blur on Enter to dismiss keyboard
    if (e.key === 'Enter') e.target.blur();
}}
// Safari wheel event fix: blur input when scrolling
onWheel={(e) => e.target.blur()}
```

---

## 2. Global Utilities & UX Patterns

### 2.1 Input Handling: `InputGroup` Component

```javascript
const InputGroup = ({ label, value, onChange, unit, help, placeholder, type="decimal" }) => (
  <div className="bg-white p-4 rounded-xl border border-slate-200 shadow-sm
                  focus-within:ring-2 focus-within:ring-primary-500 transition-all">
    <label className="text-[10px] font-bold text-slate-500 uppercase tracking-wider mb-1
                      flex items-center justify-between">
      <span>{label}</span>
      {help && <span title={help}>
        <Icon name="info" className="w-3 h-3 text-slate-400" />
      </span>}
    </label>
    <div className="flex items-center gap-2">
      <input
        inputMode={type}           // "decimal" for numeric keyboard on mobile
        type="number"
        className="w-full text-xl font-mono text-slate-800 outline-none
                   bg-transparent placeholder:text-slate-300"
        value={value}
        placeholder={placeholder || "0"}
        onChange={(e) => onChange(e.target.value)}
        onKeyDown={(e) => {
            if (["e", "E", "+", "-"].includes(e.key)) e.preventDefault();
            if (e.key === 'Enter') e.target.blur();
        }}
        onWheel={(e) => e.target.blur()}
      />
      {unit && (
        <span className="text-xs font-bold text-slate-500 bg-slate-100
                         px-2 py-1 rounded whitespace-nowrap">
          {unit}
        </span>
      )}
    </div>
  </div>
);
```

**Sanitization Features:**
| Input | Action |
|-------|--------|
| `e`, `E` | Blocked (prevents scientific notation entry) |
| `+`, `-` | Blocked (prevents sign entry) |
| `Enter` | Blurs input (dismisses mobile keyboard) |
| Mouse wheel | Blurs input (prevents accidental value changes in Safari) |

---

### 2.2 Visual Feedback: `ResultCard` Component

**Tone System:**
```javascript
const tones = {
  success: "bg-emerald-50 border-emerald-200 text-emerald-800",  // Green
  warn: "bg-amber-50 border-amber-200 text-amber-800",           // Yellow/Orange
  error: "bg-red-50 border-red-200 text-red-800",                // Red
  info: "bg-primary-50 border-primary-200 text-primary-800"      // Cyan
};
```

**Calculation Result Object Structure:**
```javascript
{
  result: number,           // Numeric result
  resultText: string,       // Formatted result string
  unit: string,             // Unit label
  sub: string,              // Secondary info line
  steps: string[],          // Array of calculation step descriptions
  insight: string,          // Contextual message
  insight2: string,         // Optional secondary insight
  tone: "success"|"warn"|"error"|"info",
  extra: ReactNode          // Optional additional UI (e.g., navigation buttons)
}
```

**Step Rendering (Collapsible):**
```javascript
{calc.steps && (
  <div>
    <button onClick={() => setShowSteps(!showSteps)} className="...">
      <span>Calculation Steps</span>
      <Icon name={showSteps ? "chevronUp" : "chevronDown"} />
    </button>
    {showSteps && (
      <div className="mt-2 space-y-1 bg-slate-50 p-3 rounded-lg">
        {calc.steps.map((s, i) => (
          <div key={i} className="font-mono text-xs text-slate-600">
            {s}
          </div>
        ))}
      </div>
    )}
  </div>
)}
```

---

### 2.3 Boot Process & Error Trapping

**Boot Overlay Mechanism:**
```html
<div id="bootOverlay">
  <div id="bootCard">
    <div id="bootTitle">Loading WWCalc…</div>
    <div id="bootMsg">If this stays here: make sure you're opening this in
         Safari/Chrome (not the ChatGPT preview), and that you're online
         the first time so the CDN scripts can load.</div>
    <div id="bootHint">Tip: If you just updated the site, do a hard refresh
         and/or clear site data to replace the old service worker cache.</div>
    <div id="bootBtn">
      <button class="bootBtn" onclick="location.reload(true)">Reload</button>
    </div>
    <div id="bootErr"></div>
  </div>
</div>
```

**Error Trapping (iOS/Safari Debugging):**
```javascript
(function(){
  const overlay = document.getElementById('bootOverlay');
  const errBox = document.getElementById('bootErr');
  const btn = document.getElementById('bootBtn');

  function showErr(msg){
    if(!overlay) return;
    errBox.style.display = 'block';
    btn.style.display = 'flex';
    errBox.textContent = msg;
  }

  // Catch synchronous errors
  window.addEventListener('error', (e) => showErr(
    'JS Error: ' + (e.message||'') + '\n' +
    (e.filename||'') + ':' + (e.lineno||'') + ':' + (e.colno||'') + '\n' +
    (e.error && e.error.stack ? e.error.stack : '')
  ));

  // Catch async/Promise rejections
  window.addEventListener('unhandledrejection', (e) => showErr(
    'Unhandled Promise Rejection:\n' +
    (e.reason && e.reason.stack ? e.reason.stack : String(e.reason))
  ));

  // Remove overlay once React mounts successfully
  window.__hideBootOverlay = () => { if(overlay) overlay.remove(); };
})();
```

**Overlay Removal:**
```javascript
// Called at end of App render
root.render(<App />);
window.__hideBootOverlay && window.__hideBootOverlay();
```

---

### 2.4 Number Formatting Utility

```javascript
const formatNum = (n, decimals = 2) => {
  if (n === null || n === undefined || isNaN(n) || !isFinite(n)) return '-';
  if (n === 0) return '0';
  const abs = Math.abs(n);
  // Use scientific notation for very small or very large numbers
  if (abs < 0.001 || abs > 1e7) return n.toExponential(2);
  return n.toLocaleString('en-US', { maximumFractionDigits: decimals });
};
```

---

### 2.5 Constants Object

```javascript
const C = {
  MGD_TO_GPM: 694.44,        // 1 MGD = 694.44 GPM
  LBS_PER_GAL: 8.34,         // Weight of water (lb/gal)
  MGD_TO_GPD: 1000000,       // 1 MGD = 1,000,000 GPD
  HP_CONSTANT: 3960,         // Hydraulic horsepower constant
  LPS_PER_MGD: 43.8126,      // 1 MGD = 43.81 L/s
  MLD_PER_MGD: 3.785,        // 1 MGD = 3.785 MLD
  ML_PER_MG: 3.785411784,    // 1 MG = 3.785 ML
  FT3_TO_GAL: 7.48,          // 1 ft³ = 7.48 gallons
  PI: Math.PI                // 3.14159...
};
```

---

## 3. Mathematical Modules (Deep Dive)

### 3.1 Loading Rate

**Formula:**
```
Loading Rate (lb/day) = Flow (MGD) × Concentration (mg/L) × 8.34
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Flow | MGD (Million Gallons per Day) |
| Concentration | mg/L |

**Output Units:** `lbs/day`

**Code Implementation:**
```javascript
const lbs = Q * mgL * C.LBS_PER_GAL;
// Where C.LBS_PER_GAL = 8.34
```

---

### 3.2 RAS Rate

**Modes:**
1. **Standard Mode** - Calculate RAS flow from known concentrations
2. **Inverse/Target Mode** - Calculate RAS percentage

**Formula:**
```
RAS% = MLSS / (RAS_TSS - MLSS)
RAS_Flow (MGD) = Influent_Flow × RAS%
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Flow (Q) | MGD |
| MLSS | mg/L |
| RAS TSS | mg/L |

**Output Units:** `MGD` and `%`

**Validation:**
- RAS TSS must be greater than MLSS (otherwise division error)

**Insights:**
- Typical range: 30% - 150%
- Outside range triggers `warn` tone

**Code Implementation:**
```javascript
if (R <= M) return { resultText: "Error", insight: "RAS TSS must be > MLSS", tone: "error" };
const rasFlow = mode === "std" ? Q * (M / (R - M)) : null;
const rasPct = mode === "std" ? (rasFlow/Q)*100 : (M / (R - M)) * 100;
```

---

### 3.3 Tank Volume

**Shapes:**

#### Cylinder (Circular Tank)
```
Area = 0.785 × Diameter²
Volume (ft³) = Area × Depth
```
*Note: 0.785 ≈ π/4*

#### Rectangular
```
Area = Length × Width
Volume (ft³) = Area × Depth
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Diameter (circular) | ft |
| Length (rectangular) | ft |
| Width (rectangular) | ft |
| Depth | ft |

**Output Units:**
| Result | Unit |
|--------|------|
| Primary | MG (Million Gallons) |
| Secondary | gallons, ft³ |

**Conversion:**
```javascript
const gal = ft3 * C.FT3_TO_GAL;  // ft³ × 7.48 = gallons
const mg = gal / 1000000;        // gallons / 1,000,000 = MG
```

---

### 3.4 Unit Converter

**Conversion Factor Tables:**

#### Flow
| Unit | Factor (relative to MGD) |
|------|--------------------------|
| MGD | 1 |
| GPM | 694.44 |
| CFS | 1.547 |
| LPS | 43.81 |

#### Volume
| Unit | Factor (relative to MG) |
|------|-------------------------|
| MG | 1 |
| Gallons | 1,000,000 |
| ft³ | 133,680.56 |
| ML | 3.785 |

#### Mass
| Unit | Factor (relative to lbs) |
|------|--------------------------|
| lbs | 1 |
| kg | 0.453592 |

#### Length
| Unit | Factor (relative to ft) |
|------|-------------------------|
| ft | 1 |
| m | 0.3048 |
| in | 12 |

**Conversion Formula:**
```javascript
const res = inputValue * (targetUnitFactor / sourceUnitFactor);
```

---

### 3.5 Pipe Volume

**Formula:**
```
Radius (ft) = (Diameter_inches / 2) / 12
Volume (ft³) = π × Radius² × Length
Volume (gal) = Volume_ft³ × 7.48
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Pipe Diameter | inches |
| Pipe Length | ft |

**Output Units:** `gallons` (primary), `ft³` (secondary)

**Code Implementation:**
```javascript
const vol_ft3 = Math.PI * Math.pow((diam/2)/12, 2) * len;
const gals = vol_ft3 * C.FT3_TO_GAL;
```

---

### 3.6 Detention Time

**Formula:**
```
Detention Time (days) = Volume (MG) / Flow (MGD)
Detention Time (hours) = (Volume / Flow) × 24
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Volume | MG |
| Flow | MGD |

**Output Units:** `hours` (primary), `days` (secondary)

**Insights:**
- Detention < 2 hours triggers `warn` tone

**Code Implementation:**
```javascript
const hrs = (V/Q) * 24;
```

---

### 3.7 Clarifier (Surface Overflow Rate)

**Formula:**
```
SOR (gpd/ft²) = (Flow_MGD × 1,000,000) / Surface_Area_ft²
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Flow | MGD |
| Surface Area | ft² |

**Output Units:** `gpd/ft²` (gallons per day per square foot)

**Insights:**
- SOR > 1000 gpd/ft² triggers `warn` tone ("High SOR - check solids loss")

**Code Implementation:**
```javascript
const sor = (Q * 1000000) / A;
```

---

### 3.8 Digester (Van Kleeck Formula)

**Formula:**
```
VS Reduction (%) = ((VS_in - VS_out) / (VS_in - (VS_in × VS_out))) × 100
```

*Where VS values are expressed as decimals (e.g., 70% = 0.70)*

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Volatile Solids In | % |
| Volatile Solids Out | % |

**Output Units:** `%` (VS Reduction percentage)

**Insights:**
- ≥ 38% reduction = "Meets EPA 38% Requirement" (`success` tone)
- < 38% = "Below 38% Requirement" (`warn` tone)

**Code Implementation:**
```javascript
const inDec = i / 100;      // Convert % to decimal
const outDec = o / 100;
const num = inDec - outDec;
const den = inDec - (inDec * outDec);
const red = (num / den) * 100;
```

**Step-by-Step Output:**
```
In = 0.70, Out = 0.50
Num = 0.70 - 0.50 = 0.2000
Den = 0.70 - (0.70 × 0.50) = 0.3500
Red = (0.2000 / 0.3500) × 100 = 57.1%
```

---

### 3.9 F/M Ratio (Food-to-Microorganism)

**Formula:**
```
F (lb BOD/day) = BOD (mg/L) × Flow (MGD) × 8.34
MLVSS (mg/L) = MLSS × (Volatile% / 100)    [if Volatile% provided]
M (lb MLVSS) = MLVSS (mg/L) × Volume (MG) × 8.34
F/M = F / M
```

**Input Units:**
| Parameter | Unit | Required |
|-----------|------|----------|
| BOD | mg/L | Yes |
| Flow | MGD | Yes |
| Volume | MG | Yes |
| MLSS | mg/L | Yes |
| Volatile % | % | Optional |

**Output Units:** `lb BOD / lb MLVSS`

**Insights:**
| F/M Range | Condition | Tone |
|-----------|-----------|------|
| < 0.2 | Low Loading (Old Sludge) | `info` |
| 0.2 - 0.5 | Optimal Range | `success` |
| > 0.5 | High Loading (Young Sludge) | `info` |

**Code Implementation:**
```javascript
const bodLbs = v.bod * v.q * 8.34;
const mlvssConc = v.pct ? v.mlss * (v.pct/100) : v.mlss;
const mlvssLbs = mlvssConc * v.vol * 8.34;
const fm = bodLbs / mlvssLbs;
```

---

### 3.10 MCRT (Mean Cell Residence Time / SRT)

**Formula:**
```
Inventory (lbs) = Volume (MG) × MLSS (mg/L) × 8.34
WAS Loss (lbs/day) = WAS_Flow (MGD) × WAS_TSS (mg/L) × 8.34
Effluent Loss (lbs/day) = Eff_Flow (MGD) × Eff_TSS (mg/L) × 8.34
Total Loss = WAS Loss + Effluent Loss
MCRT (days) = Inventory / Total Loss
```

**Input Units:**
| Parameter | Unit | Required |
|-----------|------|----------|
| Volume | MG | Yes |
| MLSS | mg/L | Yes |
| WAS Flow | MGD | Yes |
| WAS TSS | mg/L | Yes |
| Effluent Flow | MGD | Yes |
| Effluent TSS | mg/L | Optional |

**Output Units:** `days`

**Insights:**
- Target range: 5-15 days

**Code Implementation:**
```javascript
const inv = v.vol * v.mlss * 8.34;
const loss = (v.qw * v.xw * 8.34) + (v.qe * v.xe * 8.34);
return inv / loss;
```

---

### 3.11 Disinfection (Chlorine Dosage)

**Formula:**
```
Chlorine (lbs/day) = Flow (MGD) × Dose (mg/L) × 8.34
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Flow | MGD |
| Target Dose | mg/L |

**Output Units:** `lbs/day` (pure Cl₂ required)

**Code Implementation:**
```javascript
const lbs = v.q * v.dose * 8.34;
```

---

### 3.12 Pump Efficiency

**Formula:**
```
Water Horsepower (WHP) = (GPM × Head_ft) / 3960
Pump Efficiency (%) = (WHP / Motor_HP) × 100
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Flow | GPM |
| Head | ft |
| Motor HP | hp |

**Output Units:** `%` (efficiency)

**Insights:**
| Efficiency | Condition | Tone |
|------------|-----------|------|
| < 60% | Low Efficiency | `warn` |
| ≥ 60% | Good Efficiency | `success` |

**Code Implementation:**
```javascript
const whp = (v.gpm * v.head) / 3960;
const eff = (whp / v.hp) * 100;
```

---

### 3.13 Population & Design

#### 3.13.1 Water Use (gpcd)
```
gpcd = Volume_Produced (gpd) / Population
```
- **Input:** gpd, population (people)
- **Output:** gpcd (gallons per capita per day)
- **Typical range:** 50-200 gpcd

#### 3.13.2 Weir Overflow Rate
```
gpd = MGD × 1,000,000
Weir Overflow Rate (gpd/ft) = gpd / Weir_Length (ft)
```
- **Input:** MGD, weir length (ft)
- **Output:** gpd/ft

#### 3.13.3 Population Equivalent - Hydraulic
```
gpd = MGD × 1,000,000
PE (hydraulic) = gpd / gpcd
```
- **Input:** MGD, gpcd (default: 100 per ADEQ)
- **Output:** people

#### 3.13.4 Population Equivalent - Organic
```
BOD Load (lb/day) = MGD × BOD (mg/L) × 8.34
PE (organic) = BOD Load / BOD_per_person (lb/person·day)
```
- **Input:** MGD, BOD (mg/L), BOD per person (default: 0.17 lb/person·day per ADEQ)
- **Output:** people

---

### 3.14 Advanced Calculations

#### 3.14.1 Two-Normal Equation (C₁V₁ = C₂V₂)

**Formula:**
```
C₁ × V₁ = C₂ × V₂
```

**Solve for any variable:**
- `V₁ = (C₂ × V₂) / C₁`
- `V₂ = (C₁ × V₁) / C₂`
- `C₁ = (C₂ × V₂) / V₁`
- `C₂ = (C₁ × V₁) / V₂`

**Input/Output Units:** User-defined (must be consistent)

**Validation:** Denominator cannot be zero

---

#### 3.14.2 Three-Normal Equation (Blending)

**Formula:**
```
C₁V₁ + C₂V₂ = C₃(V₁ + V₂)
```

**Solve for:**
- `C₃ (blend) = (C₁V₁ + C₂V₂) / (V₁ + V₂)`
- `C₁ = (C₃(V₁+V₂) - C₂V₂) / V₁`
- `V₁ = ((C₃ - C₂)V₂) / (C₁ - C₃)`
- `V₂ = ((C₃ - C₁)V₁) / (C₂ - C₃)`

**Validation:**
- Negative results indicate impossible blend with given streams

---

#### 3.14.3 Specific Gravity

**Formula:**
```
US Mode:   SG = Specific_Weight (lb/gal) / 8.34
Metric:    SG = Specific_Weight (kg/L) / 1.0
```

**Output:** Dimensionless ratio

---

#### 3.14.4 Recirculation Ratio

**Formula:**
```
R = Qr / Q
```
Where:
- `Qr` = Recirculated flow
- `Q` = Primary flow

**Output:** Ratio (multiply by 100 for percentage)

---

#### 3.14.5 Milliequivalents (mEq)

**Formula:**
```
mEq = mL × N (Normality)
```

**Solve for:**
- `mEq = mL × N`
- `mL = mEq / N`
- `N = mEq / mL`

---

#### 3.14.6 Composite Sampling (Flow-Weighted Portions)

**Formula:**
```
Base Portion (mL) = Total_Composite_Volume / Number_of_Portions
Flow Factor = Instantaneous_Flow / Average_Flow
Portion (mL) = Base Portion × Flow Factor
```

**Input Units:**
| Parameter | Unit |
|-----------|------|
| Total composite volume | mL |
| Number of portions | count |
| Average flow | any consistent unit |
| Instantaneous flow | same unit as average |

**Output Units:** `mL`

---

## 4. Component Hierarchy

```
App (Shell)
├── State Management
│   ├── active (current screen ID)
│   ├── mobileOpen (sidebar visibility)
│   ├── lastVol (useLocalStorage - cross-tool volume data)
│   └── isOffline (network status)
│
├── Sidebar (Navigation)
│   ├── Logo & Branding
│   ├── Menu Items (14 calculators)
│   │   ├── tank → TankVolume
│   │   ├── pipe → PipeVolume
│   │   ├── detention → Detention
│   │   ├── loading → LoadingRate
│   │   ├── clarifier → Clarifier
│   │   ├── ras → RASRate
│   │   ├── fm → SimpleCalc (F/M)
│   │   ├── mcrt → SimpleCalc (MCRT)
│   │   ├── chlorine → SimpleCalc (Disinfection)
│   │   ├── digester → Digester
│   │   ├── pump → SimpleCalc (Pump)
│   │   ├── convert → UnitConverter
│   │   ├── popdesign → PopulationDesign
│   │   └── advanced → AdvancedCalcs
│   └── Offline Indicator
│
└── Main Content Area
    ├── Header (title + mobile menu button)
    └── renderScreen() → Active Calculator Component
```

### Component Types

#### Dedicated Components (Custom Logic)
- `TankVolume` - Shape switching, cross-tool navigation
- `RASRate` - Mode switching (Standard/Inverse)
- `UnitConverter` - Category switching, bidirectional conversion
- `Detention` - Pre-fills from lastVol
- `Clarifier` - Pre-fills from lastVol
- `Digester` - Van Kleeck formula with EPA threshold
- `PopulationDesign` - Tabbed sub-calculators
- `AdvancedCalcs` - Tabbed sub-calculators with solve-for dropdowns

#### SimpleCalc Wrapper Components
Used for calculators with straightforward input→formula→output:
- F/M Ratio
- MCRT / SRT
- Disinfection (Chlorine)
- Pump Efficiency

### Shared UI Components
- `InputGroup` - Standardized number input with label, unit, help tooltip
- `ResultCard` - Standardized result display with tones, steps, copy button
- `Segmented` - Toggle button group for mode selection
- `Icon` - Safe Lucide icon wrapper

### Data Flow Pattern
```
User Input → useState → useDebounce (300ms) → useMemo (calculation) → ResultCard
                                                      ↓
                                              { result, steps, insight, tone }
```

---

## Appendix: Quick Reference

### Key Constants
| Constant | Value | Purpose |
|----------|-------|---------|
| 8.34 | lb/gal | Weight of water |
| 7.48 | gal/ft³ | Volume conversion |
| 694.44 | GPM/MGD | Flow conversion |
| 3960 | - | Hydraulic HP constant |
| 0.785 | π/4 | Area from diameter |

### Tone Usage Guide
| Tone | Color | Use Case |
|------|-------|----------|
| `success` | Green | Value in optimal range |
| `warn` | Amber | Value outside typical range |
| `error` | Red | Invalid input or calculation error |
| `info` | Cyan | Neutral informational result |

### LocalStorage Keys
| Key | Type | Purpose |
|-----|------|---------|
| `twwc_vol` | Object | `{volumeMG, area, volumeFt3}` from Tank Volume |
| `twwc_conv_*` | String | Unit converter state persistence |

---

*Documentation generated for ThewastewaterCalc.app v8-ADEQ*
