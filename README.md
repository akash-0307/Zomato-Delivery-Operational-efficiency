# Zomato Delivery Time Analysis: What Actually Drives a 54-Minute Delivery?

**An exploratory data analysis of 45,584 Zomato delivery orders to identify the operational levers that determine delivery time — and uncover a counter-intuitive finding about distance.**

> **TL;DR** — Delivery time at Zomato is not a geography problem. It's a congestion problem, amplified by weather. The single biggest controllable lever is how many orders are batched per trip. Distance and pickup lag — the two metrics most operations teams obsess over — have near-zero correlation with delivery time in this dataset.

---

## 📋 Project Overview

**Objective:** Identify the factors that drive delivery time at Zomato and separate signal from noise across operational, environmental, and agent-level variables.

**Dataset:** 45,584 delivery records with 20 features spanning agent profile, external conditions, order characteristics, and geolocation.

**Target Variable:** `Time_taken (min)` — ranges from 10 to 54 minutes, median 26 minutes.

**Tools:** Python (pandas, numpy, seaborn, matplotlib, scipy)

---

## 🎯 Approach — How I Structured the Analysis

Rather than plotting every column blindly, I grouped the 20 features into 4 logical categories based on **what they represent operationally**:

| Category | Features | Question Answered |
|---|---|---|
| **1. Delivery Agent Profile** | Age, Ratings, Vehicle Type, Vehicle Condition, Multiple Deliveries | Does *who* delivers affect speed? |
| **2. External Conditions** | Weather, Traffic, Festival, City | How much does environment affect speed? |
| **3. Order Characteristics** | Type of Order | Does *what* you order affect speed? |
| **4. Operational / Geo** | Lat/Long (→ distance), Order/Pickup timestamps (→ lag) | Does geography affect speed? |

I then ran three passes per category: **univariate** (understand the variable), **bivariate** (variable vs delivery time), and **interaction** (two variables combined).

---

## 🔍 Key Findings

### Finding 1: Batching Orders Is the Single Biggest Controllable Lever

| Batched Deliveries | Median Time | Delta |
|---|---|---|
| 0 (single order) | 22 min | baseline |
| 1 | 26 min | +18% |
| 2 | 40 min | +82% |
| 3 | 48 min | +118% |

Each additional batched delivery adds ~8 minutes of median time. Batching is Zomato's single largest **internal, controllable** variable.

### Finding 2: Traffic Is the Primary External Driver — But Not Independently

Road traffic density shows a clean monotonic relationship with delivery time:

| Traffic | Median Time |
|---|---|
| Low | 20 min |
| High | 27 min |
| Medium | 27 min |
| Jam | 31 min |

A ~11 minute swing. However — traffic's effect is **massively amplified by weather** (see Finding 3).

### Finding 3: Weather × Traffic Interaction — The Hidden Risk Compound

This was the most important finding of the analysis.

**Weather conditions alone show almost no effect on delivery time.** But when controlling for traffic, a striking pattern emerges:

| Weather | Low Traffic | Jam Traffic | Penalty from Jam |
|---|---|---|---|
| Sunny | 20 min | **21 min** | +1 min |
| Stormy / Windy / Sandstorms | 21 min | 29 min | +8 min |
| Fog / Cloudy | 20 min | **37 min** | +17 min |

**Interpretation:** Low-visibility weather (Fog, Cloudy) doesn't slow agents directly — it slows down the *entire road system*, which only manifests when roads are already congested. Sunny conditions are near-immune to traffic jams (20 → 21 min), while Foggy jams are catastrophic (20 → 37 min, +85%).

This is an **interaction effect**, not an additive one. It wouldn't show up in any single-variable analysis.

### Finding 4: Festivals Are High-Variance Shocks

Festival days show a median delivery time of **45 min vs 25 min** on normal days — an 80% increase. But more importantly, the variance during festivals is significantly wider, meaning festival days are not just slower but **less predictable**.

### Finding 5: Agent Ratings Matter — Age Matters Less Than Expected

| Variable | Correlation with Time | Interpretation |
|---|---|---|
| Delivery_person_Ratings | **-0.338** | Higher-rated agents are genuinely faster |
| Delivery_person_Age | +0.298 | Mild positive, non-linear |
| Vehicle_condition | -0.243 | Poor-condition vehicles add ~4 min |

### Finding 6 (Counter-Intuitive): Distance Does NOT Predict Delivery Time

After feature-engineering straight-line distance from lat/long pairs using the Haversine formula:

```
Correlation(distance_km, Time_taken) = -0.002
```

**Distance has zero predictive value in this dataset.** A 2 km delivery takes the same time as a 15 km delivery. The road state completely dominates the road length.

This is the most surprising finding — and the most strategically important. It suggests Zomato's delivery bottleneck is **not a last-mile geography problem.** It's a congestion and batching problem.

---

## ⚠️ Data Quality Notes

A good analyst flags what the data *cannot* tell you:

1. **`City = Semi-Urban`** has only 156 rows out of 45K (0.3%). Any finding about Semi-Urban delivery time (including the 49-min median) is statistically unreliable.
2. **`pickup_lag`** was discretised to only three values (5, 10, 15 min) before the dataset was shared. This kills any correlation analysis. The near-zero correlation (r = -0.002) is a data granularity artifact, not a real finding.
3. **Time-of-day information** is not cleanly available as a derived feature. This is a meaningful gap — weather's effect might be partially explained by time-of-day confounding that cannot be tested here.

---

## 💡 Business Recommendations

Framed as actions Zomato's product and ops teams could take:

| # | Recommendation | Impact Lever |
|---|---|---|
| 1 | Cap batching at 2 deliveries in adverse weather + high traffic zones | Cuts worst-case median from 48 → ~35 min |
| 2 | Build a "Fog + Jam" alert flag in the dispatch system; pre-notify customers of ETA buffer | Manages expectations during the 37-min scenarios |
| 3 | Create festival-day operational playbook — higher agent density, dynamic surge pricing, proactive customer communication | Contains the 45-min festival median variance |
| 4 | Tie agent ratings to batch eligibility — highest-rated agents get priority on batched orders | Leverages the -0.338 rating-speed correlation |
| 5 | Stop optimising routes for distance minimisation alone — it's the wrong objective function | Re-focuses ops on congestion avoidance |

---

## 🧠 What I Learned About EDA From This Project

1. **Never trust a bivariate finding without interaction analysis.** Weather looked meaningless until I controlled for traffic.
2. **Sample size checks should come before conclusions.** Semi-Urban's 49-min median almost made it into my findings.
3. **r ≈ 0 is a finding, not a failure.** The distance result is the most actionable insight in the whole analysis — it tells Zomato what *not* to optimise for.
4. **The best insights often come from engineered features.** Distance, while ultimately null, was the right question to ask.
5. **Document what the data can't tell you.** The pickup lag discretisation issue is the kind of thing that separates a junior analyst from someone PMs want to work with.

---

## 📊 Analysis Artifacts

- `01_univariate.png` — Distribution of agent profile variables
- `02_bivariate.png` — Each feature vs. delivery time
- `03_interaction.png` — Agent × Batch and Rating × Vehicle heatmaps
- `04_correlation.png` — Profile numerics correlation matrix
- `05_weather_traffic_interaction.png` — The key finding (Weather × Traffic heatmap)
- `delivery_person_eda.py` — Full analysis script

---

## 📬 Contact

**Akash Prasad** · B.Tech, IIT Bombay (2025) · Currently at Qure.ai  
📧 akashprasad0307@gmail.com · 🔗 [linkedin.com/in/akash-0307](https://linkedin.com/in/akash-0307)

*Open to Product Analyst / Growth Analyst / Data Analyst roles in consumer tech, healthtech, and fintech.*
