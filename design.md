# ICS 3202: Artificial Intelligence — Practical Exercise 3
## Designing a Searching Agent: Autonomous Self-Driving Trailer

* **Student / Group Lead:** Rocliff Kadaji (`166133`)
* **Group Members:** *(Add remaining group member names and admission numbers here)*
* **Repository:** `PS-Agents`

---

## 1. Agent Design: PEAS Framework & Environment Definitions

### 1.1 PEAS Framework

| Component | Self-Driving Trailer Agent Specification |
| :--- | :--- |
| **Performance Measure** | • **Safety:** Zero collisions, strict adherence to safe braking distances for heavy articulated loads, and zero rollover incidents on escarpments.<br>• **Efficiency:** Minimizing cumulative route distance ($g(n)$) and transit time between Nairobi and Kisumu.<br>• **Compliance:** Obeying the 80 km/h commercial speed governor limit, axle-load weighbridge limits, and Kenyan traffic regulations.<br>• **Fuel Economy:** Smooth throttle modulation and avoiding steep/congested detours. |
| **Environment** | • Kenya's intercity highway network (A104/A8 Northern Corridor, A2, B5, C77, B1, A1) spanning urban traffic, Rift Valley escarpments (Limuru–Mai Mahiu), single-carriageway highways, weighbridges, roundabouts, pedestrians, livestock, matatus, and variable weather (fog, heavy rain). |
| **Actuators** | • Electronic steering actuator (articulated tractor-trailer control)<br>• Drive-by-wire throttle and Automated Manual Transmission (AMT) gear selector<br>• Pneumatic air-brake system and engine retarder (Jake brake)<br>• Headlights, fog lamps, turn signals, hazard lights, and horn<br>• Fleet telemetry / V2X communication module |
| **Sensors** | • Long-range and short-range LiDAR (3D obstacle and lane-boundary detection)<br>• Forward, side, and rear RADAR (relative speed and distance of surrounding traffic)<br>• Stereo RGB cameras (traffic signs, lane markings, traffic lights)<br>• GNSS/GPS receiver + Inertial Measurement Unit (localization and road gradient)<br>• Axle-load sensors, air-brake pressure sensors, and wheel speed encoders |

---

### 1.2 Environment Classification (5 Agent Environment Dimensions)

| Environment Dimension | Classification | Justification |
| :--- | :--- | :--- |
| **1. Observability** | **Partially Observable** | Onboard cameras and LiDAR cannot see around blind escarpment bends, through dense Rift Valley fog, or detect downstream accidents/roadblocks without external APIs. |
| **2. Agents** | **Multi-Agent** | The highway contains human-driven trucks, buses, matatus, motorcycles (boda bodas), pedestrians, and other vehicles whose trajectories interact with the trailer. |
| **3. Determinism** | **Stochastic** | Outcomes of driving actions are uncertain due to wet tarmac friction, sudden traffic stops, crosswinds, and unpredictable road users. |
| **4. Episodic vs. Sequential** | **Sequential** | Current steering, speed, and junction choices determine the trailer's future momentum, fuel state, and reachable highway corridors. |
| **5. Static vs. Dynamic** | **Dynamic** | Surrounding traffic moves continuously, weather shifts, and traffic signals change while the agent computes its decisions. |
| *(Additional)* **Discrete vs. Continuous** | **Continuous** | Vehicle position, velocity, and steering angles vary continuously (abstracted into discrete town nodes and road segments for high-level route planning). |

---

## 2. Directed Graph & Abstracted Search Tree (Nairobi to Kisumu)

### 2.1 Regional Highway Network & Google Maps Road Distances
The network spans all four specified regions—**Nairobi**, **Central Highlands**, **Rift Valley**, and **Lake Victoria Basin**—using Google Maps driving distances as edge costs $g(n)$:

| Region | Towns / Junction Nodes |
| :--- | :--- |
| **Nairobi** | `Nairobi` *(Start State)* |
| **Central Highlands** | `Thika`, `Nyeri`, `Nyahururu` |
| **Rift Valley** | `Naivasha`, `Gilgil`, `Nakuru`, `Mau Summit`, `Kericho`, `Eldoret` |
| **Lake Victoria Basin** | `Ahero`, `Kisumu` *(Goal State)* |

#### Directed Edges & Path Cost Table

| Origin | Destination | Highway Corridor | Distance Cost $g(n)$ (km) |
| :--- | :--- | :--- | :---: |
| `Nairobi` | `Naivasha` | A8 (Nairobi–Nakuru Highway via Mai Mahiu) | 90 |
| `Nairobi` | `Thika` | A2 (Thika Superhighway) | 45 |
| `Thika` | `Nyeri` | A2 (Kenol–Sagana–Karatina Highway) | 106 |
| `Nyeri` | `Nyahururu` | B5 (Nyeri–Nyahururu Road) | 101 |
| `Nyahururu` | `Gilgil` | C77 (Nyahururu–Gilgil Road) | 67 |
| `Naivasha` | `Gilgil` | A8 (Naivasha–Gilgil Highway) | 28 |
| `Gilgil` | `Nakuru` | A8 (Gilgil–Nakuru Highway) | 43 |
| `Nakuru` | `Mau Summit` | A8 (Nakuru–Mau Summit Highway) | 52 |
| `Mau Summit` | `Kericho` | B1 (Mau Summit–Kericho Highway) | 48 |
| `Mau Summit` | `Eldoret` | A8 / A104 (Mau Summit–Eldoret Highway) | 103 |
| `Kericho` | `Ahero` | B1 (Kericho–Awasi–Ahero Highway) | 62 |
| `Eldoret` | `Kisumu` | A1 / C39 (Eldoret–Kapsabet–Kisumu Highway) | 118 |
| `Ahero` | `Kisumu` | B1 / A1 (Ahero–Kisumu Highway) | 22 |

---

### 2.2 Directed Graph (Kenya Highway Network)

```mermaid
graph LR
    NBI((Nairobi)) -->|90 km| NVS((Naivasha))
    NBI -->|45 km| THK((Thika))
    THK -->|106 km| NYR((Nyeri))
    NYR -->|101 km| NYH((Nyahururu))
    NYH -->|67 km| GLG((Gilgil))
    NVS -->|28 km| GLG
    GLG -->|43 km| NKR((Nakuru))
    NKR -->|52 km| MAU((Mau Summit))
    MAU -->|48 km| KRC((Kericho))
    MAU -->|103 km| ELD((Eldoret))
    KRC -->|62 km| AHR((Ahero))
    ELD -->|118 km| KSM((Kisumu))
    AHR -->|22 km| KSM
    ```

---

### 2.3 Abstracted Search Tree (Nairobi $\rightarrow$ Kisumu)

**How Abstraction is Applied:**
* **State Abstraction:** Continuous GPS coordinates, road curvature, and intermediate trading centers are collapsed into discrete junction states (major towns where branching decisions occur).
* **Action Abstraction:** Low-level continuous actuator controls (steering, braking, gear changes) are abstracted into macro-transitions `Travel(Origin, Destination)` weighted by segment distance in kilometers.
* **Redundant-State Pruning:** Re-visiting `Gilgil` via the longer Central Highlands loop ($319\text{ km}$) is pruned because `Gilgil` is already reached via the Rift Valley corridor at a lower cost ($118\text{ km}$).

```text
                                     [Nairobi (g = 0)]
                                    /                  \
                     (90 km)       /                    \       (45 km)
                                  v                      v
                       [Naivasha (g = 90)]        [Thika (g = 45)]
                                  |                      |
                     (28 km)      |                      |      (106 km)
                                  v                      v
                        [Gilgil (g = 118)]        [Nyeri (g = 151)]
                                  |                      |
                     (43 km)      |                      |      (101 km)
                                  v                      v
                        [Nakuru (g = 161)]      [Nyahururu (g = 252)]
                                  |                      |
                     (52 km)      |                      |      (67 km)
                                  v                      v
                      [Mau Summit (g = 213)]     [Gilgil (g = 319)]  <-- Pruned (319 > 118)
                      /                    \
         (48 km)     /                      \     (103 km)
                    v                        v
          [Kericho (g = 261)]          [Eldoret (g = 316)]
                    |                        |
         (62 km)    |                        |    (118 km)
                    v                        v
          [Ahero (g = 323)]           [Kisumu (g = 434)]  (Goal - Route 2)
                    |
         (22 km)    |
                    v
          [Kisumu (g = 345)]  (Goal - Route 1: OPTIMAL)
```

---

## 3. Optimal Path Determination (Nairobi to Kisumu)

Applying **Uniform Cost Search (UCS)** with path cost $g(n)$ equal to cumulative Google Maps driving distance:

| Route | Town Sequence | Distance Summation (km) | Total Cost $g(n)$ |
| :--- | :--- | :--- | :---: |
| **Route 1 (Optimal)** | `Nairobi` $\rightarrow$ `Naivasha` $\rightarrow$ `Gilgil` $\rightarrow$ `Nakuru` $\rightarrow$ `Mau Summit` $\rightarrow$ `Kericho` $\rightarrow$ `Ahero` $\rightarrow$ `Kisumu` | $90 + 28 + 43 + 52 + 48 + 62 + 22$ | **345 km** |
| **Route 2** | `Nairobi` $\rightarrow$ `Naivasha` $\rightarrow$ `Gilgil` $\rightarrow$ `Nakuru` $\rightarrow$ `Mau Summit` $\rightarrow$ `Eldoret` $\rightarrow$ `Kisumu` | $90 + 28 + 43 + 52 + 103 + 118$ | **434 km** |
| **Route 3** | `Nairobi` $\rightarrow$ `Thika` $\rightarrow$ `Nyeri` $\rightarrow$ `Nyahururu` $\rightarrow$ `Gilgil` $\rightarrow$ `Nakuru` $\rightarrow$ `Mau Summit` $\rightarrow$ `Kericho` $\rightarrow$ `Ahero` $\rightarrow$ `Kisumu` | $45 + 106 + 101 + 67 + 43 + 52 + 48 + 62 + 22$ | **546 km** |
| **Route 4** | `Nairobi` $\rightarrow$ `Thika` $\rightarrow$ `Nyeri` $\rightarrow$ `Nyahururu` $\rightarrow$ `Gilgil` $\rightarrow$ `Nakuru` $\rightarrow$ `Mau Summit` $\rightarrow$ `Eldoret` $\rightarrow$ `Kisumu` | $45 + 106 + 101 + 67 + 43 + 52 + 103 + 118$ | **635 km** |

* **Selected Optimal Path:** `Nairobi -> Naivasha -> Gilgil -> Nakuru -> Mau Summit -> Kericho -> Ahero -> Kisumu`
* **Optimal Path Cost:** **345 km** (saves **89 km** over the Eldoret corridor and **201 km** over the Central Highlands detour).

---

## 4. Evidence of Groupwork

### Group Meeting Minutes
* **Date:** *(Insert Date)*
* **Mode:** Online (Google Meet) / Physical Discussion
* **Participants:**
  1. **Rocliff Kadaji (`166133`)** — Defined PEAS architecture, 5 environment classifications, and constructed the Mermaid directed graph & search tree.
  2. *(Member 2 Name & Admission Number)* — Extracted and verified Google Maps distances across Nairobi, Central Highlands, Rift Valley, and Lake Victoria Basin.
  3. *(Member 3 Name & Admission Number)* — Verified Uniform Cost Search path expansions and Markdown formatting.
* **Summary of Discussion:**
  * Agreed on selecting major highway junctions across all four regions so both primary (A8/B1) and alternative (A2/B5/C77 and A104/A1) corridors are represented.
  * Confirmed all calculations and diagrams are rendered natively in Markdown without handwritten uploads.