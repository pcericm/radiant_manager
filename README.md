# Radiant Manager System Architecture

## Overview
This system is an intelligent orchestration layer sitting between **Versatile Thermostat** entities and the physical boiler/valves. 

Its primary purpose is to solve the **Short Cycling** and **Minimum Firing Rate** problems inherent in mixed lower mass and high mass radiant zones paired with high-efficiency condensing boilers, while providing knobs to tune the balance between **Efficiency** and **Comfort**.

## Origin & Motivation: The "Smart" Thermostat Problem
This system was built to overcome limitations in standard smart thermostats (like **Ecobee**, Nest, or others) which are often optimized for forced-air furnaces, not radiant floors using **Outdoor Reset (ODR)** curves. It also serves as a powerful mitigation layer for **improperly designed or implemented systems**, correcting for physical deficiencies (like micro-zoning or oversized boilers) via intelligent software orchestration.

*   **The ODR Conflict**: Consumer "Smart Recovery" algorithms are a "black box" that (in my experience) haven't worked properly. When a modern boiler varies its water temperature based on outdoor weather (ODR), these predictive models fail, leading to erratic behavior.
*   **The Inertia Issue**: Radiant floors have huge thermal inertia. Standard algorithms fail to account for the "braking distance" required to stop heating a slab hours before the target is reached.
*   **The Minimum Fire Math**: Most modern high-efficiency boilers have a "Minimum Fire Rate" (Turndown Ratio). A large boiler cannot just produce a tiny amount of heat for a single bathroom zone.
    *   *Generic Scenario*: A typical high-efficiency boiler might have a 10:1 turndown. If it's a 200k BTUh unit, the lowest it can go is 20k BTUh. If you try to heat a zone that only needs 2k BTUh, you are firing 10x too much energy into the pipe, potentially causing temperature spikes that trigger short cycles.
    *   *Our Specific Example*: We use a **399k BTUh Laars boiler** with a 10:1 turndown. At 7,000ft altitude (which requires a 25% derate), the absolute minimum flame it can sustain is **32,000 BTUh**.
    *   *The Mismatch*: The **Primary Bath** requires only **3,700 BTUh** at design temp. The **Mudroom** needs only **2,800 BTUh**.
    *   *Failure Mode*: If the Mudroom calls for heat alone, the boiler fires at 32k BTUh into a 2.8k BTUh load (**11x oversize**). The water quickly overheats, leading to **rapid short-cycling** on the high-limit safety.
*   **The Solution**: This system utilizes **Real-Time and Feed-Forward Data** to implement intelligent "Throttle and Brake" logic. By monitoring outdoor temps, calculated heat loss, and return metrics logic, the system can apply the brakes *before* the room overheats.

### Hardware Configuration Requirements
This system is hardware-agnostic. It works with **any thermostat or valve controller** exposed to Home Assistant (Ecobee, Nest, Z-Wave, Zigbee, ESPHome, Shellies, etc.), provided:
1.  **Dumb Mode**: Disable ALL learning, smart recovery, and "Eco+" features on the physical device. They must act as simple temperature probes/actuators.
2.  **Minimum Run Time**: Set the on-device minimum run time to **10 minutes** or higher to prevent hardware-level jitter.
3.  **Calibration**: Devices must be calibrated against a trusted thermometer.

The **Versatile Thermostat** integration handles the TPI physics; this system handles the **Orchestration**.

## Control Philosophy: Efficiency vs. Comfort
Unlike "black box" commercial controllers, this system exposes specific **Dials** that allow the homeowner to choose their position on the spectrum:

*   **Max Efficiency**: By increasing queue thresholds (`BOILER_MIN_FIRE`) and using aggressive batching, the system forces the boiler to maximize condensing mode, squeezing every BTU from the fuel.
    *   *Mechanism*: It forces zones to wait until a "critical mass" of load is gathered.
    *   *Trade-off*: Higher temperature variance (zones might cool slightly before the bus leaves).
*   **Max Comfort**: By lowering thresholds, the system reacts faster to small zones (like a single cold bathroom).
    *   *Mechanism*: It lowers the entry criteria, allowing the boiler to run more often for smaller loads.
    *   *Trade-off*: More frequent boiler cycles, lower fuel efficiency, but rock-steady temperatures.
*   **The Balanced Middle**: Most users (like us) settle here—using **Active Thresholds (~23%)** that prevent short-cycling but are sensitive enough to catch temperature drops before they become noticeable.

## Core Logic Flow

### 1. The Input: Versatile Thermostat (TPI)
The system does *not* decide the room temperature. It relies on the [Versatile Thermostat](https://github.com/jmcollin78/versatile_thermostat) integration to handle the physics of the room.
*   **TPI (Time Proportional & Integral)**: Versatile Thermostat calculates a duty cycle (0-100%) required to maintain temperature.
*   **Key Tuning Parameters**:
    *   `coef_int` (Integral): Controls how aggressively the system corrects long-term errors.
        *   **0.3**: Recommended for High Mass Radiant. Slower, smoother accumulation.
        *   **0.6**: Typical for radiators. Can cause "hunting" in slab floors.
    *   `coef_ext` (External): Feed-forward component based on outdoor temperature.
        *   **0.002**: Recommended to push the duty cycle higher on cold days *before* the room drops temperature. This helps overcome the "Inertia Lag".
*   **Global Orchestration**: All VTherm "Proxy Thermostats" are logically slaved to a central **"Boiler Master"** bus. This allows the entire heating system to be armed or disarmed (turned 'Heat' or 'Off') via a single toggle, ensuring positive shutdown when needed.
*   **Signal**: We read this duty cycle via `sensor.{zone}_power_percent`.

### 2. The Processor: Radiant Manager (Queueing)
Instead of firing the boiler immediately when a zone calls for heat (which would cause short cycling if the zone is small), the manager acts as a **Queueing System**.
*   **Consolidation**: It watches all zones. If a zone is calling for heat (high duty cycle), it enters the queue.
*   **Load Calculation**: It calculates the BTU load of the queue based on the current water temperature (Outdoor Reset) and flow rates. **Zone BTU loads used in this calculation are derived directly from the home's original Manual J documentation.**
*   **The Gate**: The boiler fires ONLY when the total queued load exceeds `BOILER_MIN_FIRE`. This is set to **32,000 BTUh**, which accounts for the 25% minimum fire rate deration of the 399k BTUh Laars boiler required at **7,000ft altitude**.

### 3. Queue Management Features
To make this work comfortably, several advanced logic gates are applied:
*   **Recruitment**: If the queue is *almost* full enough to fire, the system "recruits" available **large high-mass zones** (like the Living Room) or the **Garage** to add load. These zones act as thermal batteries/ballast to efficiently absorb excess capacity.
*   **Penalty Box**: If a zone successfully heats up and turns off, it enters a "Penalty Box" (10-minute cooldown) where it cannot request heat again, preventing jitter.
*   **Scavenging**: If the boiler is already running, smaller zones are allowed to "piggyback" (scavenge) onto the active cycle even if they wouldn't satisfy the minimum load on their own. This feature can be toggled **per zone** via the `allow_scavenge: true/false` configuration parameter.

### 4. Phase 4: Logic Hardening & Pulse Support (Jan 2026)
To address "Flywheel Overshoot" in high-mass slabs and improve boiler safety, new logic layers were added:

#### A. Native Pulse Support (Anti-Flywheel)
The system now respects the native **PWM Cycle** of the Versatile Thermostat.
*   **Old Way**: If `power_percent > 23%`, run continuously. Result: Overshoot.
*   **New Way**: If VTherm switches to `Idle` (Pulse OFF), the zone stops immediately (subject to safety timers). This allows the system to "feather" heat input even when demand is high.

#### B. Smart Buffer Strategy
Buffer zones are no longer held hostage blindly.
*   **Starvation Mode**: If Load < 32k or Boiler is Latched, Buffer is held to **+1.0°F** (Strict).
*   **Healthy Mode**: If Load > 32k, Buffer releases at **+0.5°F** (Normal).
*   **Latch Override**: If Boiler is in Minimum Run (Latch), Buffers are **forced active** to prevent self-imposed starvation.

#### C. Safety Hierarchy
1.  **Manual Override**: Supreme priority. Bypasses Penalty Box and Pulse Veto.
2.  **Boiler Min Run**: Hard floor. If Boiler < 15m, zones CANNOT stop.
3.  **Physical Off**: Hard veto. If User Tstat = Off, zone stays Off.
4.  **Pulse Logic**: The "Soft" control layer.

## Solar Manager Integration
The system includes a dedicated `solar_manager.py` module that provides feed-forward thermal inputs based on weather forecasts.
*   **Purpose**: To prevent overheating on sunny days by "braking" the slab charging early in the morning.
*   **Rocket Logic**: If a rapid temperature rise (>10°F delta) and high solar gain (>4 hours) is predicted, the system shifts thermostats to `Eco` mode before sunrise.
    *   **Variable Timing**: The "braking distance" is configurable. We use **90 minutes** (`OFFSET_MINUTES_BEFORE_SUNRISE`) to allow the slab to cool before the sun hits, but this can be adjusted (60m, 120m) based on your specific building envelope and glass exposure.
*   **Catching the Knife (Dusk Boost)**: The inverse of Rocket Logic.
    *   **Trigger**: If the sun is setting (last 20° elevation) and a cold night is predicted (`feels_like < 30°F` or `min < 25°F`).
    *   **Action**: Shifts to `Boost` mode (+1°F) to pre-charge the slab, preventing the temperature "crash" that often happens at sunset.
*   **Mechanism**: The Solar Manager switches the Versatile Thermostat "Preset Mode". Crucially, **these presets are configured individually per zone based on their specific characteristics**:
    *   **Eco Mode**: Drops the target temperature based on that room's specific solar exposure. For example, a **Guest Room** (North facing) may only drop **1°F**, while a **Living Room** (South facing) may drop **3°F**.
    *   **Comfort**: Normal TPI operation.
    *   **Boost**: Increases target temp during dusk/damp conditions.

## Material Profiles
The system applies different logic based on floor material:
*   **GYPCRETE (Medium Mass)**: Moderate response. Faster than slab but still holds heat. Higher priority for immediate heating.
*   **CONCRETE (High Mass)**: Slow response. Used primarily as thermal battery/ballast to smooth out boiler cycles.

## Financial Granularity
Because the system calculates real-time energy usage based on **BTU Load** rather than simple runtime:
*   **Per-Room Costing**: Costs are broken down by individual zone, not just the whole house.
*   **Tuning**: This allows the homeowner to see exactly how much the "Master Bedroom at 72°F" costs vs "70°F", enabling precise, data-driven tuning of the budget.

## Safety Mechanisms & Failure Modes
The system includes multiple layers of protection to prevent damage:
1.  **Physical Off Veto**: A hard-stop. If a user sets a wall thermostat to 'OFF', the system will never override it, even if the house is freezing. This ensures maintenance safety.
2.  **Fail-Safe Setpoints**: The software operates by modifying the setpoints on the physical thermostats.
    *   To call for heat, it sets the physical thermostat **+1°F** above target.
    *   To stop heat, it sets it **-2°F** below target.
    *   **Failure Mode**: If Home Assistant crashes, the physical thermostat remains at its last setpoint. The house will essentially default to maintaining a temperature either slightly above or slightly below the target, ensuring pipes don't freeze and the house doesn't bake.
3.  **Emergency Stop**: A virtual input boolean (`input_boolean.radiant_emergency_stop`) that cuts all power to pumps and valves instantly.
4.  **Valve Watchdog**: Monitors for "Stuck Valves" where the physical temperature rises significantly above target (>5°F overshoot), signaling a mechanical failure.
5.  **Pump Cycling**: The system's "Pulse Maintenance" strategy ensures that even low-demand zones get flow regularly during the heating season.
    *   *Natural Exercise*: Because of the daily ODR load matching, typically no zone sits idle for more than 24 hours in winter, preventing pump seizure.
    *   *Off-Season*: (Note: This software does not currently enforce a summer exercise routine; we rely on the physical zone controller hardware or manual runs for off-season maintenance).6.  **Boiler Cooldown**: Enforces a strict minimum "OFF" time for the boiler to purge heat and prevent stress cracks from rapid thermal cycling.

7.  **Smart Buffer Selection**: When the system needs to "recruit" a buffer zone to meet minimum boiler flow, it doesn't just dumbly pick the largest zone.
    *   **Logic**: It sorts all available candidates by their **Duty Cycle (Demand)**.
    *   **Action**: It recruits the zone that is "furthest away" from its target temperature first.
    *   **Result**: This ensures we are putting the excess heat into the coldest available slab, rather than overheating the Living Room just because it's big.

8.  **Panic Dump (Load Shedding)**: While currently disabled, the system has the capability to activate a "Last Resort Dump Zone" (e.g., Garage slab).
    *   **Trigger**: If the boiler cannot modulate down low enough and water temps spike dangerously, or if a specific zone overheats critically.
    *   **Action**: It opens the Garage valve to create a massive heat sink, "dumping" the excess energy into the largest slab available to protect the boiler from tripping its High Limit switch.
    *   **Status**: Currently disabled as the primary logic handles load management effectively.



### Real World Example: The "Feed Forward" Effect
How does this math translate to your house? Let’s assume your **Target Temp is 67°F** and your **Active Threshold is 23%**.

**Scenario A: Mild Winter Day (35°F Outdoor)**
*   The "External" component (Feed Forward) contributes a small amount (say, **5%**) because it's not that cold.
*   To reach the **23%** trigger to start the boiler, the "Proportional" component (Room Temp Deficit) must do the heavy lifting.
*   *Result*: The room must drop to **66.5°F** (0.5°F deficit) to generate enough signal intensity to hit 23%.
*   *Outcome*: The system waits for a real drop before firing.

**Scenario B: Freezing Night (15°F Outdoor)**
*   The "External" component is screaming because it knows heat loss is rapid. It contributes a huge chunk (say, **15%**) immediately.
*   Now, the Proportional component only needs to provide **8%** to cross the line.
*   *Result*: The room only needs to drop to **66.9°F** (0.1°F deficit)—or sometimes even *at* target—to trigger the 23%.
*   *Outcome*: The system anticipates the cold and fires **much earlier**, preventing the slab from getting cold effectively "Catching the Knife" before you feel it.

This is why `coef_ext` is the magic variable: it shifts the "Start Line" closer to the target as the weather gets colder.

## Control Dashboard (BMS)
A comprehensive Building Management System interface is available at `homeassistant/dashboards/radiant_bms.yaml`. This dashboard provides deep visibility into the system's "Brain":

*   **Cockpit**: High-level status of the Boiler Plant, alerts, and vital signs (Load Potential vs Actual Output).
*   **System Brain**: A transparency layer showing the exact state of the decision engine (Latched, Cooldown, Firing) and a real-time log of logic decisions (e.g., "Why is the boiler off right now?").
*   **Controls**: Individual interfaces for every zone, organized by physical Manifold (M1-M6), allowing for granular tuning.
*   **Financials**: Real-time ticker showing the exact dollar cost-per-hour of the system, broken down by individual room.
*   **Diagnostics**: Advanced engineering views including TPI Duty Cycle matrices, Valve Synchronicity graphs, and Thermal Response trends for debugging.

## Metrics & Dashboard Bridge
To bridge the Python backend with the frontend dashboard, a Home Assistant **Package** file is used.
*   **File Path**: `homeassistant/packages/radiant_metrics.yaml`
*   **Function**:
    *   **Translation**: Converts Python state attributes (like `hvac_action`) into simple Binary Sensors for the Cockpit view.
    *   **Power Estimation**: Calculates instantaneous Pump Watts based on which valves are open.
    *   **Accounting**: Uses `utility_meter` entities to track Daily and Monthly costs/energy, handling the auto-reset logic at midnight.
*   **Requirements**: Your `configuration.yaml` must include the packages directory:
    ```yaml
    homeassistant:
      packages: !include_dir_named packages
    ```
