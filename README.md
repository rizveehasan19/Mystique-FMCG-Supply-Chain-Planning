# Mystique-FMCG-Supply-Chain-Analysis
End-to-end supply chain analytics for an FMCG company operating in Bangladesh. Covers warehouse network design, vehicle routing, inventory optimisation, and demand forecasting across a 2-warehouse, 50-distribution-house network.

---

## Problem Context

Mystique distributes consumer goods from 2 warehouses (Northern and Southern) to 50 distribution houses spread across Bangladesh. The business needed answers to four operational questions:

1. Which warehouse should serve each distribution house?
2. How should daily delivery routes be structured given fleet, distance, and shift constraints?
3. How much inventory should each distribution house hold, and when should they reorder?
4. What will demand look like next week?

---

## Repository Structure

```
├── Mystique_SCP.ipynb          # Main analysis notebook
├── outputs/
│   ├── route_summary.csv       # Tier 1 daily route plan
│   ├── inventory_parameters.csv # Safety stock, ROP, EOQ per DH
│   └── 7day_forecast.csv       # Forward demand forecast
└── README.md
```

---

## Dataset

Data lives in a Google Sheets workbook (`Mystique_Data`) with four tabs. The notebook authenticates via Google Colab and pulls all four directly.

| Sheet | Rows | Description |
|---|---|---|
| `Warehouses` | 2 | Location, capacity (35,000 units/day each) |
| `Distribution_Houses` | 50 | Location, storage capacity, avg daily demand |
| `Historical_Demand` | 1,500 | 30 days × 50 DHs, daily demand in units |
| `Warehouse_Distances` | 100 | Road distances (km) from each DH to each warehouse |

---

## Methodology

### 1. Warehouse Assignment

Nearest-warehouse allocation using pre-computed road distances. Flags outliers — DHs that are unusually far from their assigned warehouse or sit equidistant from both.

**Result:** W1 serves 22 DHs (1,575 units/day, 43%); W2 serves 28 DHs (2,095 units/day, 57%). Both warehouses are significantly under-utilised at current demand, leaving a capacity buffer for growth.


<img width="1590" height="1181" alt="image" src="https://github.com/user-attachments/assets/0ef418b0-67f0-4028-9129-d89b96718299" />




---

### 2. Route Optimisation

Greedy nearest-neighbour heuristic for the Capacitated VRP. Three iterations were required to reach a workable solution:

| Version | Seed strategy | Scope | Distance cap | Shift | Outcome |
|---|---|---|---|---|---|
| V1 | Farthest-first | All 50 DHs | 320 km | 8 h | ❌ 35 routes, 13% utilisation, 16.5h max |
| V2 | Closest-first | All 50 DHs | 320 km | 8 h | ⚠️ Better clustering; remote DHs still infeasible |
| V3 | Closest-first | Tier 1 only (≤200 km) | 400 km | 9 h | ✅ Deployable route plan |

**V1 failure:** Seeding from the farthest DH commits the full distance budget on the outbound leg, leaving no room for additional stops — nearly every route becomes a single-stop run.

**V2 failure:** Closest-first clustering works well for nearby DHs, but DHs beyond 200 km still produce near-empty solo routes. The round-trip distance alone exhausts the 320 km cap. The routing model itself is wrong for that geography.

**V3 solution:** Split the network by distance. DHs ≤200 km (Tier 1, 32 DHs) get daily multi-stop routes with relaxed constraints reflecting Bangladesh's road reality. DHs >200 km (Tier 2, 18 DHs) move to a scheduled bulk delivery model — 3× per week, each run carrying 3 days of stock.

**Tier 1 final metrics:** ~9 daily routes across both warehouses, avg ~55% truck utilisation, daily cost ~৳45,000.

**Fleet gap identified:** Current fleet of 6 trucks is insufficient. Tier 1 alone requires ~9; peak days (with Tier 2 runs) require ~14.

---

### 3. Inventory Optimisation

Classical inventory theory applied per distribution house. Parameters vary by delivery tier due to different lead times.

**Assumptions:**

| Parameter | Value |
|---|---|
| Service level | 95% (Z = 1.65) |
| Holding cost | ৳0.50 / unit / day |
| Order cost | ৳800 / order |
| Lead time — Tier 1 | 1 day |
| Lead time — Tier 2 | 2 days |

**Outputs per DH:** safety stock, reorder point (ROP), economic order quantity (EOQ), average inventory level, annual holding and ordering cost.

**ABC classification** by cumulative demand contribution (A = top 70%, B = next 20%, C = bottom 10%) to prioritise monitoring frequency and tightness of safety stock.

**System-wide:** ~2,100 units total safety stock; ~৳2.2M annual inventory cost; 0 DHs exceeding storage capacity.


<img width="1789" height="1180" alt="image" src="https://github.com/user-attachments/assets/69754325-f76e-4260-bdd3-ef822e8cffcf" />




---

### 4. Demand Forecasting

Three time-series models were evaluated on a 25-day training / 5-day test split across all 50 DHs.

| Model | Description |
|---|---|
| Moving Average (MA) | 7-day simple average |
| Weighted MA (WMA) | 7-day linearly weighted, more weight to recent days |
| Exponential Smoothing (ES) | α = 0.3, exponential decay |

Each DH is assigned its individually best-performing model (by MAE). The best model per DH is then used to generate a **7-day forward forecast**, which feeds directly into the inventory reorder schedule.

All three models produce comparable MAPE (~8–9%) on this dataset. A longer history would enable seasonal decomposition; flagged as a follow-up.


<img width="1790" height="1181" alt="image" src="https://github.com/user-attachments/assets/210a3cc1-f955-4afe-b9b0-5baabe5e591e" />



---

## Key Findings

- **Warehouse capacity is not the constraint** — demand is ~10% of combined capacity. Growth headroom is large; fleet and logistics are the binding constraints.
- **Geography drives everything** — the 200 km tier boundary isn't arbitrary. The distance distribution has a natural break there, and it's also the point where a daily round-trip becomes infeasible within a standard shift.
- **The fleet needs to nearly double** — this is the most actionable output for operations.
- **18 DHs need a fundamentally different delivery model** — attempting to force them into daily routing produces either infeasible routes or near-empty trucks.

---

## Stack

- **Python 3.10** — Pandas, NumPy, Matplotlib, Seaborn
- **Google Colab** — runtime and Sheets authentication
- **Google Sheets API** (`gspread`) — data source

No external routing libraries used — heuristic implemented from scratch to keep the decision logic transparent and auditable.

---

## Running the Notebook

1. Upload or open `Mystique_SCP.ipynb` in Google Colab
2. Share the `Mystique_Data` Google Sheet with your Colab account
3. Run all cells — the authentication prompt will appear at cell 2
4. Outputs are saved as CSV in the working directory

---

## Limitations & Next Steps

- **Tier 2 routing** is only planned at a strategic level here. Detailed route optimisation for the 18 remote DHs (including clustering and sequencing) is a natural extension.
- **Forecasting models** are simple baselines. With 12+ months of history, Holt-Winters or SARIMA would capture seasonality.
- **Road distances** are pre-computed and static. Dynamic routing accounting for traffic or road closures would require a live maps API.
- **Fleet sizing** is flagged but not optimised. A formal fleet expansion cost-benefit analysis (lease vs buy, utilisation targets) would be the logical follow-on.


Check Colab for updated analysis: https://colab.research.google.com/drive/1ME_nt-OxGp0JGC0m1P3J4DgBZeyZN7dz?usp=sharing


---

## Author

| Field | |
|---|---|
| **Md Rizvee Hasan** | |
| **https://www.linkedin.com/in/rizveehasan19/** | |
| **rizvee.hasan19@gmail.com** | |
