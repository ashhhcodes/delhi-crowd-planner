# 5-Minute Demonstration Video Script & Walkthrough
### Transport Stack Hackathon 2026 — Phase 1 (Problem 2)
**Project:** Crowdedness-Aware Bus Journey Planner for Delhi (DTC / DIMTS Cluster)  
**Presenter / Narrator:** Antigravity Engineering Team  
**Total Target Duration:** 4 minutes 45 seconds (under 5-minute strict cap)

---

## 🎬 Video Overview & Timestamp Breakdown

| Timestamp | Duration | Section / Focus | Core Visual Action on Screen |
| :---: | :---: | :--- | :--- |
| **00:00 – 00:45** | 45s | **1. The Delhi Commuter Crisis & Problem 2** | Title slide $\to$ Delhi bus rush hour photos $\to$ Problem statement diagram. |
| **00:45 – 01:45** | 60s | **2. Data Pipeline & Mathematical Modeling** | Terminal / Architecture slides showing DuckDB + RapidFuzz + Difference-Array logic. |
| **01:45 – 03:00** | 75s | **3. Live Application Walkthrough & Journey Planning** | Live Web UI (`localhost:8000`) doing corridor searches (Nehru Place $\to$ Dhaula Kuan). |
| **03:00 – 04:00** | 60s | **4. Route Crowding Curves & Empirical Validation** | Route Crowding Explorer (`717AUP`) + Validation KPIs & Diurnal peak curves. |
| **04:00 – 04:45** | 45s | **5. Test Suite Verification & Phase 2 Roadmap** | Terminal running `pytest` (9/9 passed) $\to$ Real-time GTFS-RT streaming roadmap slide. |

---

## 🎙️ Detailed Script with On-Screen Cues

---

### Segment 1: The Delhi Commuter Crisis & Problem 2 (00:00 – 00:45)

**[Visual: Title Slide / Delhi transit map overlay]**
> **Narrator:**  
> "Hello evaluators and judges! Welcome to our demonstration of the **Crowdedness-Aware Bus Journey Planner for Delhi**, built for Phase 1, Problem 2 of the Transport Stack Hackathon 2026.
> 
> Over 4 million commuters rely on Delhi’s DTC and Cluster buses every day. But as anyone who travels across the Ring Road knows, the biggest question isn't just *'Which bus gets me there?'*—it's *'Can I physically board the bus, or will I be crushed inside?'*"

**[Visual: Comparison slide showing Google Maps vs. Our Planner]**
> **Narrator:**  
> "Standard navigation tools like Google Maps optimize purely for timetable travel time. They treat a packed, standing-only Non-AC bus identically to a comfortable, half-empty AC bus running right behind it.
> 
> Our mission in Problem 2 is to unlock **crowd-aware multi-criteria routing** that balances travel time, physical transfers, comfort tiers, and fleet mode preferences."

---

### Segment 2: Data Pipeline & Mathematical Modeling (00:45 – 01:45)

**[Visual: Architecture diagram: Ticketing + GTFS $\to$ Normalization $\to$ Difference Array $\to$ McRAPTOR]**
> **Narrator:**  
> "To solve this, we ingested two foundational datasets: the complete Delhi GTFS static feed with over 6,800 stops and 2,574 routes, and 2.56 million ETM ticketing records across October 2026.
> 
> But raw ticketing data is notoriously dirty. First, **48.52% of transaction records contain literal `'NULL'` strings** in the vehicle number field. Second, digital transactions account for only **~4% of total boardings**. Third, colloquial bus stop names diverge drastically from official GTFS names."

**[Visual: Show table highlighting stop resolution jumping from 48.27% to 93.13%]**
> **Narrator:**  
> "We tackled this through three core innovations:
> 1. **Route-Scoped RapidFuzz Resolution**: By restricting stop searches strictly to the stops on each specific line, we eliminated cross-city false matches and jumped our stop match rate from **48.27% to 93.13%**, recovering 2.57 million valid forward passengers.
> 2. **Vectorized Difference-Array Flow Expansion**: We reconstructed 1.2 million segment passenger flows in just 3.2 seconds without memory-heavy cross joins.
> 3. **Dual-Source Supply Calibration**: We blended scheduled GTFS departures with empirical vehicle observations corrected for the 48.5% NULL artifact, scaling with digital penetration $\kappa = 25.0$ and mapping through a zero-anchored sigmoid crowding function."

---

### Segment 3: Live Application Walkthrough & Journey Planning (01:45 – 03:00)

**[Visual: Switch browser to `http://127.0.0.1:8000/`. The clean, responsive UI appears with Leaflet map on the left and control panel on the right.]**
> **Narrator:**  
> "Let’s jump into the live application running on our FastAPI service. 
> Notice that when the search fields are empty, the app immediately provides quick-select buttons for 10 major transit hubs across Delhi—like Nehru Place, Dhaula Kuan, and Kashmere Gate."

**[Visual: Click 'Nehru Place' as Origin, 'Dhaula Kuan' as Destination. Set departure time to 09:00 AM Weekday. Select 'Balanced Comfort'. Click 'Find Optimal Routes'.]**
> **Narrator:**  
> "Let’s simulate a morning peak commuter traveling from **Nehru Place to Dhaula Kuan** at 9:00 AM on a weekday. We click 'Find Optimal Routes'."

**[Visual: Results appear instantly (< 200 ms). Map renders true road-following bus polylines (derived from 965k GTFS shape vertices) along Ring Road with green/orange/red segment crowding colors. Itinerary card highlights 'Recommended — Balanced' alongside 'Fastest' and 'Least Crowded'.]**
> **Narrator:**  
> "In less than 200 milliseconds, our McRAPTOR engine computes the Pareto-optimal frontier. 
> 
> Look at this fascinating trade-off:
> - The **Fastest Route** suggests taking Non-AC line `567AUP`—duration is 38.1 minutes, but passenger crowding is **0.252** with heavy standing.
> - But our **Recommended Balanced Route** suggests AC line `724EXTDOWN`—duration is 38.2 minutes with a crowding score of **0.169**—a **33% reduction in crowding** and guaranteed air-conditioned comfort for **only 6 seconds extra travel time**!
> 
> This is the exact comfort intelligence standard journey planners fail to capture."

---

### Segment 4: Route Crowding Curves & Empirical Validation (03:00 – 04:00)

**[Visual: Switch to 'Route Crowding Explorer' tab. Type in `717AUP` for Weekday Slot 18 (09:00 AM). Click 'Load Profile'.]**
> **Narrator:**  
> "Now let's explore our **Route Crowding Explorer**. Let's inspect line `717AUP`, which connects Mehrauli to Badarpur."

**[Visual: Chart.js interactive line chart loads showing crowding score stop-by-stop across all 35 stops, color-coded green to red.]**
> **Narrator:**  
> "The engine dynamically loads the stop-by-stop crowding curve. Notice how crowding surges as the bus picks up passengers through Saket and Khanpur, reaching high-density levels of **0.802** before tapering off as workers alight at the Badarpur terminal."

**[Visual: Click on 'Model Validation & Analytics' tab. Scroll through the KPI cards and Diurnal Peak Windows table.]**
> **Narrator:**  
> "On the **Validation Dashboard**, we see citywide proof of our model:
> - **Morning Peak P90** hits **0.594** across radial corridors.
> - During the **Midday Lull**, crowding drops by **83% down to 0.102**, reflecting abundant seating.
> - And our directional asymmetry check shows the busier direction switching: `717AUP` is at 0.802 in the morning against 0.648 on `717ADOWN`, and in the evening `717ADOWN` leads at 0.799."

---

### Segment 5: Test Suite Verification & Phase 2 Roadmap (04:00 – 04:45)

**[Visual: Switch to terminal window. Run `python -m pytest src/api/test_api.py -v`.]**
> **Narrator:**  
> "All endpoints, data contracts, and routing algorithms are rigorously tested. Running our pytest suite shows all 9 end-to-end integration tests passing in under 10 seconds."

**[Visual: Terminal shows `9 passed in 9.18s`. Then cut to Phase 2 Roadmap slide.]**
> **Narrator:**  
> "Looking ahead to **Phase 2**:
> Because our router operates on modular segment-slot indices with sub-microsecond lookup times, transitioning to live operations requires zero architectural changes. 
> 
> We will connect streaming **GTFS-RT vehicle positions** and live **ONDC ticket issuance webhooks** directly into an in-memory Kalman filter, updating segment crowding scores every 60 seconds in real time.
> 
> Thank you to the Centre for Sustainable Mobility, IIIT-Delhi, Chartr, and the Hackathon committee. We look forward to advancing to Phase 2!"

**[Visual: End slide with Team Name, GitHub repository link, and Thank You.]**
