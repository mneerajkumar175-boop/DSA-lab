# Emergency Route Planner

**Intelligent Emergency Vehicle Routing and Path Optimisation** (Roll No 56–60)

This app routes ambulances through a simulated city. It finds the fastest route using predicted traffic, turns traffic signals green ahead of the ambulance, and re-routes immediately when a road is blocked. It also saves every emergency call in a database.

- **Backend:** Python + FastAPI. It does all the work: the city graph, the traffic model, Dijkstra / A* / D* Lite, dispatch and the simulation clock.
- **Database:** SQLite, which is built into Python. It stores every call and every log message.
- **Frontend:** HTML, CSS and JavaScript. It only draws the map and sends clicks to the API.

---

## 1. How to run it (Windows)

You need **Python 3.10 or newer**. To check, open Command Prompt and run `python --version`.

**Step 1: open a terminal in the project folder**

```bat
cd path\to\emergency-route-planner
```

**Step 2: create a virtual environment and install the libraries (first time only)**

```bat
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

**Step 3: start the server**

```bat
uvicorn backend.main:app --reload
```

**Step 4: open the app in your browser**

- App: http://127.0.0.1:8000
- Interactive API docs: http://127.0.0.1:8000/docs

To stop the server, press `Ctrl + C`. The next time you only need `venv\Scripts\activate` followed by Step 3.

On **Mac or Linux**, use `python3 -m venv venv` and `source venv/bin/activate` instead.

**Run the tests:**

```bat
python -m pytest -q
```

---

## 2. Architecture

```
 ┌──────────────── Browser (frontend/) ────────────────┐
 │  index.html + style.css + app.js                     │
 │  - draws the map on a <canvas>                       │
 │  - asks GET /api/state about 7 times a second        │
 │  - sends clicks as POST requests                     │
 └───────────────▲──────────────────────┬───────────────┘
                 │ JSON                 │ HTTP (REST)
 ┌───────────────┴──────────────────────▼───────────────┐
 │  main.py        FastAPI: routes, input validation    │
 │       │                                              │
 │  simulation.py  ambulances, calls, clock, re-routing │
 │       │                 │                    │       │
 │  algorithms.py    traffic.py           database.py   │
 │  Dijkstra, A*,    congestion +         SQLite:       │
 │  D* Lite          travel times         incidents,    │
 │       │                 │              events        │
 │       └──── city.py: the road graph ───┘             │
 └──────────────────────── Backend (Python) ────────────┘
```

### Folder structure

| File | What it does |
|---|---|
| `backend/city.py` | Builds the city graph: 216 junctions, about 360 roads, a river with 4 bridges, 4 hospitals and 5 ambulance bases |
| `backend/traffic.py` | Predicts congestion for every road and converts it into travel time (the edge weight) |
| `backend/algorithms.py` | Dijkstra, A*, shortest-distance search and D* Lite |
| `backend/simulation.py` | Dispatch, moving ambulances, re-routing, green corridor and the state sent to the frontend |
| `backend/database.py` | SQLite tables `incidents` and `events` |
| `backend/main.py` | FastAPI app: all the API endpoints |
| `frontend/` | The web page |
| `tests/` | Automated tests for the algorithms and the API |
| `data/dispatch.db` | Created automatically on first run |

---

## 3. API endpoints

| Method | URL | What it does |
|---|---|---|
| GET | `/api/map` | City layout (sent once) |
| GET | `/api/state` | Live state: ambulances, routes, traffic levels, signals, log |
| GET | `/api/history` | Past calls from the database, with averages |
| POST | `/api/emergencies` | Report an emergency `{x, y}` or `{random_location: true}` |
| POST | `/api/roads/{id}/toggle` | Block a road, or reopen it if it's blocked |
| POST | `/api/roads/accident` | Block a road ahead of a moving ambulance |
| POST | `/api/roads/clear` | Reopen all roads |
| POST | `/api/settings` | Change `hour`, `weekend`, `algorithm`, `corridor`, `speed`, `paused` |
| POST | `/api/reset` | Restart the simulation (the database keeps its history) |

You can try every endpoint from the browser at **/docs**, which is a good thing to show the teacher.

---

## 4. Demo script (about 3 minutes)

1. Click the map to report an emergency. The nearest free ambulance is dispatched, and its route appears in black. The dashed grey line is the plain shortest-distance route.
2. Point out **"Why this route"**: shortest distance vs. traffic-aware vs. traffic-aware + green corridor.
3. Watch the green dots: the signals ahead of the ambulance turn green.
4. Click **Accident on route**. The ambulance re-routes, and the algorithm table shows D* Lite expanding fewer junctions than A* from scratch.
5. Move the time slider to **18:00**. Roads turn red, and every route is re-planned for evening traffic.
6. Switch the route search between **A\*** and **Dijkstra**. Both find the same route, but A* explores fewer junctions.
7. Scroll down to **Call history**. These rows come from the SQLite database and stay there after a restart.
8. Open **/docs** and run `GET /api/history` to show the backend on its own.

---

## 5. Explaining it to the teacher

### The one-minute version

> "The city is a weighted graph. Junctions are nodes and roads are edges. The weight of each road is the predicted travel time, which depends on time of day, road type and how central the road is. When an emergency is reported, we run Dijkstra once from the emergency to find the nearest free ambulance and the fastest hospital. The route itself is found with A*. While the ambulance is driving, D* Lite handles road closures by repairing only the affected part of the search instead of starting again. The green corridor turns signals green ahead of the ambulance. Everything runs in a FastAPI backend, the browser only displays it, and every call is stored in SQLite."

### Key formulas

- **Congestion** = rush(hour, weekday) × road type factor × (0.35 + centrality) × hotspot
- **Ambulance speed** = speed limit ÷ (1 + 0.55 × congestion). Sirens let it push through part of the traffic.
- **Edge weight (seconds)** = road length ÷ ambulance speed + signal wait at the next junction
- **Signal wait** = 5 + 30 × rush × junction size. With the green corridor it is about 2 s.
- **A\* heuristic** = straight-line distance ÷ top speed (50 km/h). It never overestimates, so A* is still optimal.

### Likely viva questions

**Why A\* instead of Dijkstra?**
Both give the same optimal route. A* adds a heuristic that pulls the search towards the goal, so it expands fewer nodes. The algorithm table shows this live.

**Why is the A\* heuristic admissible?**
No ambulance can go faster than the top speed limit in a straight line. So straight-line distance ÷ top speed is always ≤ the real travel time.

**What does D\* Lite do that A\* doesn't?**
D* Lite keeps its search results (the `g` and `rhs` values) between runs. When a road closes, it only updates the two junctions at the ends of that road, and changes spread only where needed. A* would have to search everything again from scratch.

**Why does D\* Lite search backwards from the goal?**
The goal (the hospital or the patient) stays fixed while the start (the ambulance) keeps moving. Searching from the fixed end means the saved results stay valid. The `km` value corrects the queue keys as the start moves.

**How is the hospital chosen?**
One Dijkstra from the emergency gives the travel time to every junction. From that, we pick the free ambulance with the lowest time and the hospital with the lowest time.

**What data structure is the priority queue?**
A binary heap (`heapq`). Push and pop are O(log n), so Dijkstra and A* run in O((V + E) log V).

**Is the traffic real?**
No. The traffic is a model, built from rush-hour curves and road properties. In a real system the same `traffic.py` interface would be fed by live data (Google or TomTom traffic APIs) or by a trained model, for example XGBoost on historical speeds.

**Why have a backend at all?**
It separates the logic from the display. Several users or devices could share one simulation, the data is saved in a database, and the API could be used by a mobile app or a traffic-control system.

**How do you know the algorithms are correct?**
`tests/test_algorithms.py` checks that A* always matches Dijkstra's cost, and that D* Lite's re-planned route matches a fresh Dijkstra after random road closures.

---

## 6. Limitations and future work

- The city, traffic and hospitals are simulated. A real map could come from OpenStreetMap using the `osmnx` library.
- The traffic model is a formula, not trained on data. Possible upgrade: an XGBoost or LSTM model on historical traffic.
- Hospital capacity and how serious each patient is are not considered.
- There is only one simulation per server. Multiple control rooms would need user accounts.
