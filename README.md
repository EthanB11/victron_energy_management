# Victron Multi-Tab Energy Automation System
### Production Deployment Manual

This repository contains an autonomous, weather-aware, multi-bank solar priority controller designed for the **Victron MultiPlus** platform. By integrating real-time API weather matrices with localized Net Energy Deficit calculators, this engine optimizes solar consumption runtime windows while actively protecting battery banks.

Devices in use
1. Cebr0GX running Venus large
2. BMV-712 battery monitor
3. MutliPlus II
4. 2x Victron solar controllers

---

## 📂 System Architecture Overview

The system is deployed across **two distinct Node-RED Workspace tabs** to cleanly isolate data gathering from master relay execution logic:

1. **Tab 1: Automated Shore Power & Grid Management** — Monitors battery SoC, voltage, and AC consumption loads to manipulate your physical AC Input transfer relay contacts.
2. **Tab 2: Weather Core Engine** — An autonomous astronomic scheduling loop that pulls forecast matrices to dynamically adjust your baseline battery safety limits throughout the seasons.

---

## 🛠️ Section 1: Weather Core Engine Setup (Tab 2)

Your background weather automation utilizes global variable passing to scale system thresholds on the fly.

```text
[ 30-Minute Inject Timer ] ───► [ Weather API / Fetch Node ] ───► [ Calculate Dynamic Floor ]
                                                                             │
                                                                             ▼
                                                               Writes: global.set("dynamicLowerSoC", X)
```

### How the Weather Logic Adjusts Your System:
Every 30 minutes, JavaScript inside the **Calculate Dynamic Floor** node checks the sky visibility arrays:
* **Clear / Sunny Skies:** The script writes `20%` into global memory. Your main controller runs its standard solar priority script down to your absolute minimum target.
* **Overcast / Rain / Heavy Clouds:** The script automatically scales your battery safety limits up to **`35%`**. This forces your main control block to re-engage grid charging prematurely, freezing your battery backup bank at a higher capacity before a severe storm strikes.

---

## 🔧 Section 2: Weather Integration & Code Customization (Action Required)

When sharing or importing this flow, you **must modify the topic filters** inside the JavaScript code to match your specific weather provider's output string format.

### Step 1: Identify Your Weather Node Output
Wire a green **Debug Node** to the output of your weather fetch block. Trigger it and look at the debug sidebar to see the format of the weather payload:
* Does it send a text string like `"Overcast"` or `"Rain"`?
* Does it send a numerical percentage value for cloud cover like `85`?

### Step 2: Modify the Code Filters
Open the **Advanced Logic & Alarms** node on Tab 1. Scroll to **Section 2 (Lines 37–44)** and customize the string keyword checks to match your data format:

```javascript
// CUSTOMIZE THIS BLOCK TO MATCH YOUR WEATHER PROVIDER:
if (lowerTopic.includes("weather") || lowerTopic.includes("cloud") || lowerTopic.includes("forecast")) {
    
    // Pattern A: Text string verification (e.g., OpenWeatherMap Node)
    if (msg.payload === "Overcast" || msg.payload === "Rain") { 
        isWeatherBad = true; 
    } 
    
    // Pattern B: Raw percentage threshold calculation (e.g., Cloud Cover > 70%)
    else if (typeof msg.payload === 'number' && msg.payload > 70) {
        isWeatherBad = true;
    }
    
    else { 
        isWeatherBad = false; 
    }
    context.set('isWeatherBad', isWeatherBad);
}
```

---

## ⚙️ Section 3: Master Logic Configuration (Tab 1)

All mathematical thresholds are positioned at the very top of the **Advanced Logic & Alarms** node for immediate adaptation:

```javascript
// ==========================================
// USER-CONFIGURED SYSTEM FIELDS (EDIT HERE)
// ==========================================
const HARD_FLOOR_SOC = 20;     // Minimum fallback State of Charge floor limit (%)
const RECOVERY_BUFFER = 4;     // SoC % required above floor before restarting solar priority

const CELL_CRITICAL_VOLT = 23.8; // Hard fallback low voltage limit (V)
const CELL_HEALTHY_VOLT = 24.5; // Healthy voltage required before restarting solar priority

const INVERTER_MAX_CEILING = 1200; // House AC load threshold that forces a safety shift to Grid (W)
const CRITICAL_AC_LOAD_LIMIT = 700; // Maximum allowed house load before switching back to Solar (W)

const RUN_SOLAR_SURPLUS_W = 50;  // Required net solar surplus before dropping the grid (W)
const FALLBACK_DEFICIT_W = -200; // Max allowed net solar deficit before engaging the grid (W)
const HIGH_SOC_DEFICIT_LIMIT = 50; // Maximum SoC (%) where deficit protection remains active
```

---

## 📦 Section 4: Cross-Tab Data Linking via Global Context

The two canvas tabs talk to each other seamlessly using Node-RED's core context-sharing layer:
* The weather script writes the target limit using **`global.set("dynamicLowerSoC", targetFloorSOC);`**.
* The master relay controller grabs that number on every input sweep using **`let dynamicLowerSoC = global.get("dynamicLowerSoC") || HARD_FLOOR_SOC;`**.

If your weather API tab fails or experiences a timeout error, the primary safety code handles it automatically—it falls back to your local `HARD_FLOOR_SOC` constant (`20%`) to keep your system fully running without interruptions.

---

## ⚠️ Section 5: Hardware Failsafe Configurations

Because hardware layers like Raspberry Pis, Venus OS Large, or Node-RED servers can freeze, crash, or lose power, **never rely on software alone for ultimate low-battery cutoff protection.**

1. **Open VEConfigure:** Navigate to your **Inverter** configuration tab.
2. **Set Hardware Cut-Off:** Configure your physical **Inverter shut-down on low pre-alarm** voltage boundaries to **24.0V** (for 24V installations). 
3. **The Failsafe:** If your Node-RED server ever locks up while a grid-lockout override command is active, the MultiPlus will ignore the software layer at this hardware boundary. It will drop its inverter circuit locally, forcing your physical **63A Geya ATS** to switch your house back to raw grid power automatically.
