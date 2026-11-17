# Technical Note: Mathematical Methodology, Calibration & Empirical Validation
### Transport Stack Hackathon 2026 — Phase 1 (Problem 2)
**Crowdedness-Aware Bus Journey Planner for Delhi (DTC / DIMTS Cluster)**

---

## 1. Dataset Profile, Schemas & Ground Truth Findings

### 1.1 GTFS Static Feed (`delhi_buses_static_gtfs_v1`)
The static schedule models the transit network topology published by DTC and DIMTS:
- **`routes.txt`**: 2,574 routes (`route_id`, `route_short_name`, `route_long_name`).
- **`stops.txt`**: 6,862 physical bus stops (`stop_id`, `stop_name`, `stop_lat`, `stop_lon`).
- **`trips.txt`**: 80,357 scheduled trips (`route_id`, `trip_id`, `shape_id`).
- **`stop_times.txt`**: 2,737,136 scheduled stop events (`trip_id`, `arrival_time`, `departure_time`, `stop_id`, `stop_sequence`).
- **`shapes.txt`**: 965,607 coordinate vertices mapping route alignments.

#### Critical GTFS Domain Findings
1. **Directional ID Missingness**: The standard GTFS column `direction_id` in `trips.txt` is **100% NULL**. 
2. **Direction Encoded in Strings**: Direction is explicitly encoded as alphanumeric suffixes in `route_short_name` and `route_long_name` (e.g., `8UP` = Outbound, `8DOWN` = Inbound, `8ADOWN` = Inbound Variant A).
3. **Trip Variations**: Each route features multiple scheduled trip variants (e.g., depot pull-ins, truncated runs). The canonical physical topology must be extracted by selecting the representative trip with `max(stop_count)` per `route_id`.

### 1.2 ETM Ticketing Transaction Log (`October-Delhi.csv`)
The operational dataset consists of all Electronic Ticketing Machine (ETM) and ONDC transactions recorded across Delhi between October 1, 2026 and October 31, 2026:
- **Total Transactions**: **2,559,899** records.
- **Total Passengers Logged**: **2,753,606** passengers (mean: 1.076 pax/ticket).
- **Date Range**: `2026-10-01 00:00:10` to `2026-10-31 23:59:58` (31 complete days).
- **Fleet Mode**: AC: **80.4%** of tickets (2,059,092), Non-AC: **19.6%** (500,807).
- **Ticket Type**: every record has `ticket_type_id = 1`; the file carries no pass/concession split, so pass holders are not visible in this data.

#### Critical Ticketing Findings
1. **Literal `'NULL'` Fleet Identifier**:
   - While standard programmatic checks report 0% NULLs in `vehicle_number`, an exact string audit revealed that **1,241,961 transactions (48.52%)** contain the literal string `'NULL'`.
   - **Implication**: Any pipeline attempting vehicle-level trip matching based strictly on `vehicle_number` will silently discard almost half of all passenger transactions.
2. **Colloquial Stop Divergence**:
   - Raw ticketing stop names diverge widely from GTFS stop names due to abbreviations (`sec` vs `sector`, `stn` vs `station`, `ph-ii` vs `phase 2`), landmark aliases, and slash-separated bilingual or dual-depot notations (`Ito Agcr / Ito`).
   - Raw exact match between ticketing stop names and GTFS stop names was only **48.27%**.

---

## 2. Entity Resolution & Canonical Stop Mapping

### 2.1 Representative Canonical Sequence Extraction
For each of the 2,574 GTFS routes, we extract the canonical ordered stop sequence by identifying the representative `trip_id` maximizing the stop count:
$$t^* = \arg\max_{t \in \text{Trips}(r)} |\text{StopTimes}(t)|$$

For each stop $s_i$ in sequence $1 \dots N$, we record:
$$\big(r, \text{seq}_i, s_i, \text{lat}_i, \text{lon}_i, \Delta T_{i-1 \to i}\big)$$
where scheduled inter-stop travel time $\Delta T_{i-1 \to i} = T_{\text{arr}}(s_i) - T_{\text{dep}}(s_{i-1})$ (defaulting to 60s if zero or missing). This produced **83,435 canonical stop entries** across all 2,574 routes.

### 2.2 Colloquial Delhi Transit Text Normalizer
A rule-based text transformation pipeline standardizes raw ticketing stop strings prior to matching:
1. **Alias Splitting**: If a stop string contains slashes (e.g., `"Ito Agcr / Ito"`), it is parsed into the full string and individual constituent tokens (`["Ito Agcr", "Ito"]`).
2. **Abbreviation Expansion**:
   $$\begin{aligned}
   \text{sec} &\to \text{sector} & \text{ph-i / ph-ii / ph-iii} &\to \text{phase 1 / phase 2 / phase 3} \\
   \text{stn} &\to \text{station} & \text{t. / t-point} &\to \text{t point} \\
   \text{xing / x-ing} &\to \text{crossing} & \text{mkr / mkt} &\to \text{market} \\
   \text{hosp} &\to \text{hospital} & \text{vhr} &\to \text{vihar} \\
   \text{ng} &\to \text{nagar} & \text{extn} &\to \text{extension}
   \end{aligned}$$
3. **Punctuation & Whitespace Stripping**: Removal of non-alphanumeric noise and space normalization.

### 2.3 Route-Scoped RapidFuzz Resolution
Rather than executing a computationally prohibitive global search across all 6,862 stops, matching is strictly constrained to the ordered stops on the candidate route:
1. **Exact Canonical Match**: Direct lookup against canonical stop names on that route.
2. **Alias Constituent Match**: Checking individual slash-token constituents.
3. **Fuzzy String Alignment**: Using `RapidFuzz` `token_sort_ratio` and `WRatio`:
   $$\text{Score}(u, v) = \max\big(\text{token\_sort\_ratio}(u, v), \text{WRatio}(u, v)\big) \ge 78$$

### 2.4 Directional Sequence Validation
Once `start_stop_name` and `end_stop_name` resolve to canonical sequences $s_{\text{orig}}$ and $s_{\text{dest}}$ on route $r$:
- **Valid Forward Trip**: $s_{\text{orig}} < s_{\text{dest}}$ (Valid passenger boarding and alighting).
- **Reversed Sequence Anomaly**: $s_{\text{orig}} > s_{\text{dest}}$ (ETM conductor route direction toggle error).
- **Degenerate Single Stop**: $s_{\text{orig}} = s_{\text{dest}}$ (ETM entry error, 0 distance).

#### Sprint 1 Quantitative Results
| Metric | Value | Relative Impact |
| :--- | :---: | :--- |
| **Total Ticketing Rows Analyzed** | 2,559,899 | 100.0% |
| **Route Code Match Rate** | 91.19% | 2,334,310 rows (covering **98.71%** of pax volume) |
| **Stop Name Resolution Match Rate** | **93.13%** | Jumped from **48.27%** raw (+44.86% absolute gain) |
| **Valid Forward Tickets** ($s_{\text{orig}} < s_{\text{dest}}$) | **2,394,241** | **93.51% of total transaction volume** |
| **Valid Forward Passengers Recovered** | **2,574,781** | **93.51% of all passengers** |
| **Reversed Sequence Anomalies** | 60,378 | 2.36% (conductor direction toggle artifact) |
| **Identical OD Anomalies** | 2,130 | 0.08% |
| **Parquet Footprint** (`resolved_tickets.parquet`) | **24.99 MB** | ZSTD compressed (down from 290 MB CSV) |

---

## 3. Passenger Flow Reconstruction & Bus Load Factor Modeling

### 3.1 Temporal Discretization
To capture diurnal transit dynamics, transactions are partitioned into:
- **Day Types**: `weekday` (Mon–Fri, 22 days in Oct 2026), `saturday` (5 days), `sunday` (4 days).
- **Time Slots**: 48 discrete half-hour intervals:
  $$\tau = \left\lfloor \frac{\text{seconds since midnight}}{1800} \right\rfloor \in \{0, 1, \dots, 47\}$$

### 3.2 Vectorized Difference-Array Flow Expansion
Let route $r$ have canonical stop sequence $1, 2, \dots, N$. For each temporal bucket $(d, \tau)$, we aggregate total passenger demand $P(u, v)$ for all resolved OD pairs $(u, v)$ where $1 \le u < v \le N$.

Instead of expanding every ticket across intermediate stops, we populate a difference array $\Delta$ of size $N$:
$$\Delta[u] \mathrel{+}= P(u, v), \quad \Delta[v] \mathrel{-}= P(u, v)$$

The expected segment passenger volume per 30-minute slot is the prefix sum divided by operational days $D(d) \in \{22, 5, 4\}$:
$$Q(s, s+1 \mid r, d, \tau) = \frac{1}{D(d)} \sum_{k=1}^{s} \Delta[k] \quad \text{for } s \in \{1, \dots, N-1\}$$
This algorithm executes in $O(N)$ vector operations, processing all **1,409,093 OD combinations across 1,209,599 segment slots in 3.28 seconds**.

### 3.3 Dual-Source Supply Normalization
Because 48.52% of transactions lack physical vehicle IDs, relying solely on empirical vehicle tracking underestimates capacity, while relying solely on static GTFS timetables misses operational cancellations. We formulate a dual-source supply estimator:

1. **GTFS Scheduled Departures ($S_{\text{GTFS}}$)**:
   Count of trips in `stop_times.txt` originating on route $r$ within 30-min window $\tau$:
   $$S_{\text{GTFS}}(r, \tau) = \sum_{t \in \text{Trips}(r)} \mathbb{I}\left(\tau \le \frac{T_{\text{dep}}(t, \text{first\_stop})}{1800} < \tau+1\right)$$
2. **Empirical Active Fleet ($S_{\text{Emp}}$)**:
   Distinct non-null vehicle IDs observed, scaled by the missingness factor:
   $$S_{\text{Emp}}(r, d, \tau) = \frac{|\text{Distinct Non-Null Vehicles}|}{D(d) \cdot (1 - 0.4852)}$$
3. **Calibrated Supply ($S_{\text{eff}}$)**:
   $$S_{\text{eff}}(r, d, \tau) = \max\Big(1.0, \, \max\big(S_{\text{GTFS}}(r, \tau), \, S_{\text{Emp}}(r, d, \tau)\big)\Big)$$

### 3.4 Penetration Scaling & Sigmoid Crowding Formulation
1. **Calibrated Bus Passenger Demand**:
   Accounting for digital market penetration $\kappa = 25.0$ (~4% digital share):
   $$P_{\text{bus}}(s, s+1) = \frac{Q(s, s+1) \cdot \kappa}{S_{\text{eff}}(r, d, \tau)}$$
2. **Load Ratio ($L$)**:
   Normalized by fleet capacity ($\text{Cap} = 70$ for AC, $65$ for Non-AC):
   $$L = \frac{P_{\text{bus}}(s, s+1)}{\text{Cap}(\text{mode}_r)}$$
3. **Zero-Anchored Sigmoid Crowding Score ($C \in [0, 1]$)**:
   $$S(L) = \frac{1}{1 + e^{-k(L - L_0)}}, \quad C = \frac{S(L) - S(0)}{1 - S(0)}$$
   where $L_0 = 0.65$ (discomfort inflection point) and $k = 3.5$ (steepness).

#### Comfort Tier Classification
$$C(e, t) < 0.35 \implies \textbf{Low (Comfortable Seats)}, \quad 0.35 \le C(e, t) \le 0.70 \implies \textbf{Moderate (Standing Room)}, \quad C(e, t) > 0.70 \implies \textbf{High (Crush Load)}$$

---

## 4. Empirical Validation & Citywide Commuting Patterns

### 4.1 Network-Wide Diurnal Peak/Off-Peak Distribution
Weekday segment-time-slots, taken directly from `data/processed/sprint2_validation_report.json` (regenerated by `sprint2_crowding_model.py`):

| Time Window | Time Slots ($\tau$) | Mean Crowding Score | P90 Crowding Score | % Low / Moderate / High |
| :--- | :---: | :---: | :---: | :---: |
| **Early Morning** | 05:00–07:30 ($\tau = 10 \dots 14$) | 0.0992 | 0.2670 | 92.4 / 4.7 / 2.9 |
| **Morning Peak** | 08:30–10:30 ($\tau = 17 \dots 20$) | **0.1889** | **0.5939** | 82.2 / 10.2 / 7.6 |
| **Afternoon Lull** | 13:00–15:30 ($\tau = 26 \dots 30$) | **0.0518** | **0.1018** | 98.1 / 1.4 / 0.5 |
| **Evening Peak** | 17:30–20:00 ($\tau = 35 \dots 39$) | **0.1326** | **0.4089** | 88.2 / 7.2 / 4.6 |
| **Late Night** | 21:30–23:30 ($\tau = 43 \dots 46$) | 0.0466 | 0.1214 | 97.1 / 2.1 / 0.7 |

P90 crowding in the afternoon lull is **83% lower** than in the morning peak (0.102 vs 0.594).

### 4.2 Directional Asymmetry Check
Mean weekday crowding score per direction, morning peak (08:30–10:30) vs evening peak (17:30–20:00):

| Route pair | Morning UP | Morning DOWN | Evening UP | Evening DOWN | Reading |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **`717A`** | **0.802** | 0.648 | 0.632 | **0.799** | UP busier in the morning, DOWN busier in the evening; both directions are loaded at both peaks. |
| **`729`** | 0.302 | **0.459** | **0.413** | 0.264 | DOWN busier in the morning, UP in the evening. |
| **`511A`** | **0.565** | 0.327 | 0.195 | **0.281** | UP clearly dominant in the morning. |
| **`578`** | 0.201 | **0.512** | **0.301** | 0.192 | DOWN dominant in the morning, UP in the evening. |

All four pairs show the busier direction switching between morning and evening peaks. The contrast is moderate (roughly 1.2–2.5×), not a full reversal, and these are 4 hand-picked routes; a network-wide test is still to be done.

---

## 5. Multi-Criteria McRAPTOR Routing Engine

### 5.1 Generalized Discomfort Objective Function & Reliability Modeling
For transit edge $e$ on route $r$ traversed at time $t$ with scheduled transit duration $T(e)$ and crowding score $C(e, t) \in [0, 1]$, the generalized travel cost incorporates in-vehicle travel time, crowd discomfort, empirical headway waiting time, and left-behind risk:
$$\text{Cost}(e) = T(e) \cdot \left(1 + \beta \cdot C(e, t)^{1.5}\right) + \text{Cost}_{\text{wait}}(r, t) + \text{Cost}_{\text{deny}}(e, t) + W_{\text{transfer}}$$
where:
- $\beta \ge 0$ is the user discomfort weight parameter.
- $W_{\text{transfer}} = 300\text{ s}$ (5-minute discomfort penalty per transfer) prevents trivial one-stop transfers.
- **Empirical Headway Waiting Time**:
  Route headway $H_{r,t}$ (clamped to $[4.0, 30.0]$ min) is derived per canonical route across 5 time-of-day windows (`Early Morning`, `Morning Peak`, `Afternoon Lull`, `Evening Peak`, `Late Night`). Expected wait is $W_{\text{base}}(r, t) = \min(15.0, 0.5 \times H_{r,t})$, weighted by a **$1.5\times$ out-of-vehicle wait perception multiplier**:
  $$\text{Cost}_{\text{wait}}(r, t) = 1.5 \times (W_{\text{base}}(r, t) \times 60\text{s})$$
- **Left-Behind (Boarding-Denial) Risk ($C > 0.80$)**:
  When boarding segments operate at crush capacity ($C(e,t) > 0.80$), commuters face significant left-behind probability:
  $$p_{\text{deny}} = \min\left(0.75,\ \max\left(0,\ \frac{C(e,t) - 0.80}{0.20}\right) \times 0.75\right)$$
  The expected missed-bus delay $E[\Delta T_{\text{deny}}] = p_{\text{deny}} \times (H_{r,t} \times 60\text{s})$ is penalized at a **$1.7\times$ reliability multiplier** in generalized cost. Boarding risk is classified as `"High"` ($C > 0.85$), `"Moderate"` ($0.75 < C \le 0.85$), or `"Low"`.

### 5.2 Footpath Transfer Topology & Opposite-Platform Access
1. **Latitude-Window Footpath Precomputation**:
   Using binary search (`np.searchsorted`) over sorted stop latitudes ($\pm 300\text{m} \approx \pm 0.0027^\circ$), the router indexes all **23,354 walking transfer edges** across 6,862 stops in **3.5 seconds** ($v_{\text{walk}} = 1.2\text{ m/s}$, minimum transfer buffer $= 120\text{ s}$).
2. **400m Origin / Destination Walk-Access Radius Expansion**:
   Dual-carriageway boulevards in Delhi (e.g., Ring Road, Outer Ring Road) place `UP` and `DOWN` bus platforms on opposite sides of wide avenues (50–150m apart). Radius expansion automatically evaluates all physical platforms within **400m (~5 min walk)** of the selected origin/destination, ensuring parallel corridor access and metro-feeder stop integration.
3. **Road-Hugging Map Polylines (GTFS `shapes.txt` Integration)**:
   A compact serialized lookup cache (`data/processed/route_segment_shapes.pkl`, 80,859 segments) maps each canonical stop pair `(from_stop, to_stop)` to its ordered sub-slice of shape vertices, enabling true road-following polyline rendering on Leaflet maps rather than straight-line approximations.

### 5.3 Microsecond Lookups & Time-Advancement
Segment crowding scores and travel durations are indexed into dense NumPy matrices:
$$\text{CS}[r_{\text{idx}}, \text{seg}_{\text{idx}}, d_{\text{code}}, \tau] \in \mathbb{R}^{32}$$
Fetching segment cost takes **$0.27\,\mu\text{s}$ ($O(1)$)**. As the journey traverses successive segments and footpaths, the current arrival time $t_{\text{arr}}$ automatically updates the active half-hour time slot $\tau = \lfloor (t_{\text{arr}} \pmod{86400}) / 1800 \rfloor$.

### 5.4 Smart $\pm 15$-Minute Departure Shift Nudge
During peak shoulder periods where the primary itinerary experiences severe crowding ($C \ge 0.45$ or `boarding_risk != "Low"`), the router conducts lightweight forward/backward evaluations at $t - 15\text{ min}$ and $t + 15\text{ min}$.
If shifting departure time yields:
1. Average crowding reduction $\ge 15\%$ (absolute drop $\ge 0.08$), and
2. Travel time penalty $\le 10\%$,
the engine attaches an actionable `departure_nudge` payload, prompting commuters to travel slightly earlier or later to bypass crush loads.

### 5.4 Corridor Benchmark Comparisons

#### Benchmark Corridor 1: Nehru Place $\to$ Dhaula Kuan (Weekday 09:00 AM Peak)
- **Fastest ($\beta = 0.0$)**: Non-AC route `567AUP` (38.1 min, 0 transfers, Mean Crowding: **0.252**, Fare: ₹15).
- **Balanced / Least Crowded ($\beta = 0.6, 1.8$)**: AC route `724EXTDOWN` (38.2 min, 0 transfers, Mean Crowding: **0.169**, Fare: ₹25).
- **Pareto Insight**: The commuter trades **only 6 seconds of travel time** to achieve a **33% reduction in crowding** inside an air-conditioned bus.

#### Benchmark Corridor 2: Ali Village $\to$ ITO (Weekday 09:00 AM Peak)
- Direct bus connection does not exist.
- **Planner Output**: Multimodal itinerary via `422AUP` $\to$ transfer at Nehru Place $\to$ `DS-12DOWN` $\to$ transfer at Pragati Maidan $\to$ `317UP` (24.4 min, 2 transfers, Mean Crowding: **0.072**, Fare: ₹45).
- **Pareto Insight**: Confirms the router's capability to discover efficient, low-crowding multi-transfer combinations across complex network gaps.

---

## 6. Verification & Quality Assurance Summary

The entire pipeline and API service are covered by automated unit and integration tests (`src/api/test_api.py`):
1. `test_get_frontend_html`: Confirms root HTTP 200 and static dashboard bundle delivery.
2. `test_api_health`: Validates router memory state (6,862 stops, 2,574 routes, 23,354 transfer edges).
3. `test_stops_search_default_hubs`: Validates default 10 transit hubs (*Nehru Place, Dhaula Kuan, Kashmere Gate, etc.*).
4. `test_stops_search_query`: Validates RapidFuzz substring and prefix stop matching.
5. `test_plan_journey_peak_balanced`: Validates peak hour routing, itinerary schemas, and latency guarantees.
6. `test_plan_journey_offpeak_least_crowded`: Confirms crowding score drops below 0.35 in off-peak routing.
7. `test_validation_metrics`: Validates pipeline reporting metrics and diurnal peak JSON payloads.
8. `test_route_profile`: Validates stop-by-stop GPS coordinate geometry and crowding curves for `717AUP`.
9. `test_invalid_stop_plan`: Validates 404 error handling on non-existent stops.

**Test Suite Status**: **9 PASSED in 9.18 seconds**.
