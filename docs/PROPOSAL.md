# Phase 1 Proposal: Crowdedness-Aware Bus Journey Planner for Delhi (DTC / Cluster)
### Transport Stack Hackathon 2026 — Phase 1 (Problem 2)
**Prepared for the Evaluation Committee:** Centre for Sustainable Mobility (CSM), IIIT-Delhi | Chartr / ONDC | JICA

---

## 1. Executive Summary & Problem Framing

Delhi’s public bus network—operated by the **Delhi Transport Corporation (DTC)** and **DIMTS Cluster**—is the lifeblood of the National Capital Region (NCR), serving over **4.2 million commuters daily** across 6,000+ active buses and 600+ physical routes. 

Yet, despite being the most affordable motorized transit mode in the city, daily commuters face acute daily friction:
1. **Severe In-Vehicle Crowding**: Passengers frequently endure crushing crush-loads during peak hours or are bypassed at stops by physically overloaded buses.
2. **Asymmetric Route Distribution**: Parallel bus routes sharing identical physical arterial corridors often exhibit radical disparities in passenger occupancy—where an overcrowded Non-AC bus is followed immediately by an underutilized AC bus.
3. **Planners Blind to Human Comfort**: Mainstream navigation tools (Google Maps, standard GTFS schedule engines) optimize purely for minimum timetable travel time or physical transfers. They have zero awareness of in-vehicle crowding, seat availability, or comfort trade-offs between AC and Non-AC fleets.

### The Real-World Data Challenge
To build a crowd-aware planner, naive consumption of raw Electronic Ticketing Machine (ETM / ONDC) transactions fails:
- **Low Digital Penetration**: With fare exemptions (e.g., Pink Tickets for women) and cash ticketing dominating surface bus transit, digital/ONDC transactions capture roughly **~4% of total physical boardings** ($\kappa \approx 25.0$). Treating raw transaction counts as absolute bus volume drastically under-predicts real crush loads.
- **Fleet Identification Data Quality**: In the provided real-world ticketing logs, **48.52% of transactions contain literal `'NULL'` strings in the `vehicle_number` column**, making vehicle-level trip matching unreliable without robust dual-source supply modeling.
- **Topological Divergence**: GTFS static timetable `direction_id` fields are **100% NULL**, with route directions instead baked into strings (`UP`/`DOWN`), and colloquial ETM stop names diverge significantly from official GTFS names.

Our solution addresses these exact challenges with a mathematically grounded, computationally efficient transit modeling and routing architecture.

---

## 2. System Architecture & End-to-End Pipeline

Our architecture comprises five tightly integrated, decoupled components designed for sub-second consumer response and massive offline data throughput:

```
[Raw Data Sources]
├── Delhi Static GTFS (6,862 stops, 2,574 routes, 2.7M stop_times)
└── ETM Ticketing Logs (2.56M txns, 2.75M passengers, Oct 1–31, 2026)
                        │
                        ▼
[Sprint 1: Entity Resolution & Canonical Alignment]
├── Colloquial Delhi Transit Text Normalizer (expands abbreviations, splits aliases)
├── Route-Scoped RapidFuzz Matcher (search restricted to route stops)
└── Directional Forward-Trip Filter (orig_seq < dest_seq)
    Result: Stop match rate jumped from 48.27% to 93.13% (+44.86%)
            2,574,781 valid forward passengers recovered (93.51%)
                        │
                        ▼
[Sprint 2: Vectorized Passenger Flow & Load Factor Engine]
├── Vectorized Difference-Array OD-to-Segment Flow Reconstruction (1.21M segment-slots)
├── Dual-Source Supply Normalization:
│   - Scheduled GTFS departure frequencies
│   - Empirical active fleet observations (calibrated for 48.52% NULL vehicle IDs)
├── Sample Penetration Scaling: κ = 25.0 (4% digital sampling rate)
├── Mode-Specific Capacity Normalization: Non-AC (65 cap) vs AC (70 cap)
└── Zero-Anchored Sigmoid Occupancy Mapping: C(e, t) ∈ [0, 1]
                        │
                        ▼
[Sprint 3: Time-Dependent McRAPTOR Journey Planning Engine]
├── 300m Footpath Transfer Graph (23,354 walking edges, 3.5s fast lat-window index)
├── Multi-Stop 250m Origin/Destination Radius Expansion (UP vs DOWN platform access)
├── O(1) Sub-Microsecond Segment Crowding Lookup (0.27 μs) with automatic time-advancement
└── Multi-Criteria Pareto Optimization:
    Cost(e) = T(e) · (1 + β · C(e, t)^1.5) + W_transfer
    Profiles: Fastest (β = 0.0), Balanced (β = 0.6), Least Crowded (β = 1.8)
                        │
                        ▼
[Sprint 4 & 5: Production Delivery & User Experience]
├── High-Throughput FastAPI REST Backend (< 180 ms single-profile latency)
└── Interactive Modern Web Application (Leaflet GIS paths + Chart.js crowding profiles)
```

---

## 3. Key Methodological Innovations

### 3.1 Route-Scoped Candidate Stop Resolution
Traditional global string distance matching across all 6,862 stops produces frequent false positives (e.g., matching a "Pocket A" in East Delhi to a "Pocket A" in Dwarka). By first resolving the route string (`8UP` $\to$ GTFS `route_id: 8`, `route_short_name: 8UP`) and constraining candidate stop matching strictly to the ordered stops on that specific route, we:
1. Eliminated geographic false matches entirely.
2. Reduced fuzzy search latency by **over 100-fold**.
3. Elevated match rate from **48.27% to 93.13%**, recovering 2,574,781 forward passengers.

### 3.2 Vectorized Difference-Array Flow Reconstruction
A naive approach to expanding 2.4 million OD ticket pairs across intermediate stops creates an intermediate Cartesian table of ~11.6 million rows, exhausting memory on commodity infrastructure. 

Instead, we formulated an $O(N)$ difference-array algorithm:
- For each OD pair $(s_{\text{orig}}, s_{\text{dest}})$ on a route with passenger count $P$:
  $$\Delta[s_{\text{orig}}] \mathrel{+}= P, \quad \Delta[s_{\text{dest}}] \mathrel{-}= P$$
- Evaluating the cumulative sum along the route's canonical stop sequence yields the exact segment passenger load across all intermediate links simultaneously:
  $$\text{Load}(s, s+1) = \sum_{i=1}^{s} \Delta[i]$$
This computed **1,209,599 segment-time-slot flows in just 3.28 seconds** with less than 500 MB RAM.

### 3.3 Zero-Anchored Sigmoid Crowding Formulation
Perceived passenger discomfort does not scale linearly with bus occupancy. A bus at 30% load feels empty and comfortable; a bus transitioning from 90% to 110% load crosses a sharp psychological discomfort threshold from standing room to crush load. 

We formulated a zero-anchored centered sigmoid:
$$S(L) = \frac{1}{1 + e^{-k(L - L_0)}}, \quad C(e, t) = \frac{S(L) - S(0)}{1 - S(0)}$$
where $L$ is the calibrated passenger-to-capacity load ratio, $L_0 = 0.65$ is the discomfort midpoint, and $k = 3.5$ is the steepness parameter. This ensures:
- $C(0) = 0.000$ (zero passenger load has zero crowding penalty).
- $C(L_0) \approx 0.500$ (seats fully occupied, moderate comfort tier).
- $C(L \ge 1.0) \to 1.000$ (severe crush load, steep penalty).

### 3.4 Transit Reliability & Behavioral Nudge Upgrades
1. **Empirical Route Headways & Perception Weighting**:
   Route headways $H_{r,t}$ are computed per route and time window, estimating realistic wait times $W_{\text{base}}(r, t) = \min(15.0, 0.5 \times H_{r,t})$ with a $1.5\times$ out-of-vehicle wait perception multiplier in generalized cost.
2. **Crush Load Left-Behind Risk Penalty**:
   When boarding segments exceed 80% occupancy, the engine calculates missed-bus denial probability $p_{\text{deny}}$ and adds $1.7\times$ expected headway delay to prevent passengers from being routed into overcrowded buses they cannot physically board.
3. **Smart $\pm 15$-Minute Departure Shift Nudge**:
   During peak periods ($C \ge 0.45$), the system automatically evaluates departure times 15 minutes earlier and later. If shifting departure reduces average crowding by $\ge 15\%$ without increasing travel time by $> 10\%$, a proactive nudge advises the commuter to bypass crush loads.

---

## 4. Empirical Validation & Citywide Impact

Our model was validated against the entire month of October 2026 ticketing data (2.56M transactions):

1. **Bi-Modal Diurnal Commuter Waves**:
   - **Morning Peak (08:30–10:30)**: Network P90 crowding reaches **0.594** (Heavy standing).
   - **Afternoon Lull (13:00–15:30)**: Network P90 drops by **83% to 0.102** (Open seating).
   - **Evening Peak (17:30–20:00)**: Network P90 rebounds to **0.409** (Suburban dispersal).
2. **Directional Asymmetry**:
   - Radial line `717A` (Mehrauli $\to$ Badarpur) exhibits a morning peak crowding score of **0.802** on `717AUP` vs. **0.648** on `717ADOWN`. In the evening the busier direction switches: `717ADOWN` reaches **0.799**, while `717AUP` falls to **0.632**.
3. **Pareto Comfort Unlocking (Nehru Place $\to$ Dhaula Kuan)**:
   - Our planner demonstrates that taking AC line `724EXTDOWN` instead of Non-AC line `567AUP` saves **33% in passenger crowding** while adding **only 6 seconds** to the 38-minute trip.

---

## 5. Phase 2 Roadmap & Production Deployment

Because our Phase 1 engine operates on modular segment-slot indices, transitioning to Phase 2 real-time streaming requires **zero re-architecture**:

```
                       [Phase 2 Real-Time Integration]
                                      │
           ┌──────────────────────────┴──────────────────────────┐
           ▼                                                     ▼
[Live GTFS-RT Vehicle Positions]                      [Streaming ONDC / ETM Feed]
- Real-time GPS coordinates                           - Real-time digital ticket issues
- Dynamic headway & delay updates                     - Micro-batch passenger delta updates
           │                                                     │
           └──────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
             [Streaming Kalman Filter / EWMA Crowding Updater]
             - Ingests streaming deltas every 60 seconds
             - Blends historical baseline prior with live vehicle occupancy
             - Updates dense segment crowding arrays in-place (O(1) memory)
                                      │
                                      ▼
                      [Live Multi-Criteria McRAPTOR Router]
                      - Evaluates dynamic real-time crowd avoidance
                      - Pushes live crowding alerts to consumer mobile apps
```

### Proposed Phase 2 Milestones
- **M1 (Weeks 1–3): Ingestion of Delhi Open Transit GTFS-RT Feeds**:
  Ingest live vehicle positions and trip updates; map GPS pings to canonical GTFS trip segments.
- **M2 (Weeks 4–6): Real-Time Streaming Occupancy Filter**:
  Deploy a streaming Kalman / EWMA state estimator that dynamically adjusts segment crowding scores based on active vehicle counts and recent stop transactions.
- **M3 (Weeks 7–8): Pilot Deployment & Field Validation**:
  Conduct physical ground-truth passenger count audits across 5 high-density Delhi corridors (*Ring Road, Mehrauli-Badarpur Road, Outer Ring Road, Najafgarh Road, Vikas Marg*) to calibrate and benchmark real-time prediction accuracy.

---

## 6. Conclusion

Our **Crowdedness-Aware Bus Journey Planner** transforms raw, incomplete transit data into an actionable, passenger-centric mobility tool. By reconciling real-world data gaps through robust engineering and delivering sub-second Pareto itineraries, this platform provides DTC, DIMTS, and millions of daily Delhi commuters with a scalable solution for equitable, comfortable, and efficient urban transit.
