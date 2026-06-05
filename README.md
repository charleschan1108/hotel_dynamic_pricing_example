# Hotel Dynamic Pricing — AI Training for Melco Analysts

## Overview

This training teaches analysts how to build a **dynamic pricing system** for hotel rooms using open-source data. The same framework applies to Melco's ADT system — just swap in your own data.

**Dataset**: [Hotel Booking Demand](https://www.kaggle.com/datasets/jessemostipak/hotel-booking-demand) — 119K bookings from a Resort Hotel and a City Hotel (2015–2017), including ADR (Average Daily Rate = room price/night), booking lead time, market segment, and more.

---

## How to Run

1. Download `hotel_bookings.csv` from Kaggle and place it in this folder
2. Open `hotel_dynamic_pricing_old.ipynb` in Jupyter
3. Run the first cell (`%pip install ...`) to install packages
4. Run all cells top-to-bottom (~2 min total)

Or use `uv`:
```bash
uv run jupyter lab
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

```

---

## Notebook Structure

| Section | Cells | What Analysts Learn |
|--------|-------|---------------------|
| 1. Load & Clean | 3 | Data wrangling: date parsing, filtering cancellations & outliers |
| 2. EDA | 2 | Seasonality patterns, demand curves, price drivers (segment, lead time) |
| 3. Data Prep | 1 | Daily aggregation + pick-up curve + quarterly×segment panel for elasticity |
| 4. Pick-Up + SARIMAX | 4 | Forecast demand (pick-up) + **evaluation vs naive baseline** |
| 5. Elasticity | 5 | Log-log regression for price sensitivity + **endogeneity diagnosis** + literature benchmarks |
| 6. Dynamic Pricing | 4 | Combining forecast + elasticity → revenue-optimal price schedule |
| 7. Summary | 1 | Key takeaways for Melco |

---

## Key Insights for Delivery

### 1. Modeling Approach: Pick-Up Forecasting → ADR Derived from Elasticity

**The correct causal chain for dynamic pricing:**
1. **SARIMAX** forecasts demand (pick-up) → D₀ rooms/day
2. **Elasticity** tells us how demand responds to price → ε
3. **Revenue optimization**: find price P* that maximizes P × D₀ × (P/P₀)^ε
4. **P* is the derived ADR** — no need for a separate ADR forecast

**Why this order?** Demand drives pricing, not the other way around. You don't forecast ADR independently — you derive it from the demand forecast and elasticity.

**How we reconstruct pick-up:**
- `lead_time` field tells us how far in advance each booking was made
- Aggregate bookings by arrival date × lead-time bucket → pick-up curve
- 7 lead windows: 0–7d (last-minute), 8–14d, 15–30d, 31–60d, 61–90d, 91–180d, 181d+ (early planners)

**Average pick-up curve (% of final bookings in hand):**

| Lead Window | Resort | City |
|------------|--------|------|
| 181d+ (earliest) | 15% | 11% |
| 91–180d | 34% | 31% |
| 61–90d | 43% | 43% |
| 31–60d | 57% | 60% |
| 15–30d | 69% | 73% |
| 8–14d | 77% | 82% |
| 0–7d (last-minute) | 100% | 100% |

**Model**: SARIMAX(1,1,1)(1,0,1,7) with `is_weekend` + 4 Fourier pairs, forecasting demand only.

**Weekend premium**: Resort +9% (€91→€99), City +3% (€103→€106)

---

### 2. Demand Forecast Accuracy

**Baseline**: Naive forecast — predict last observed demand. Any useful model must beat this.

| Hotel | Metric | SARIMAX | Naive | Beats Naive? |
|-------|--------|---------|-------|-------------|
| Resort | MAE | 4.5 rooms | 9.2 rooms | ✅ YES (2× better) |
| Resort | MAPE | 13.4% | 24.5% | ✅ YES |
| City | MAE | 14.4 rooms | 19.7 rooms | ✅ YES |
| City | MAPE | 25.9% | 36.9% | ✅ YES |

- **Demand forecasting works well** for both hotels — SARIMAX beats naive by ~50%
- **Teaching point**: Always compare vs a naive baseline

### 3. Price Elasticity Is Hard to Estimate from Observational Data

- **OLS gives POSITIVE elasticity** (+1.9 to +2.0) — economically nonsensical
- Root cause: **Endogeneity** — in high season, hotels raise prices AND demand rises. OLS confuses the two
- Even with quarter fixed effects and segment-level data, the bias persists
- **This is the #1 pitfall in pricing analytics** and the most important lesson for analysts

### 4. Practical Solution: Literature Benchmarks + A/B Testing

Since OLS is biased, we use published elasticity estimates:
- **Resort Hotel: ε = -1.4** (elastic — leisure travelers are price-sensitive)
- **City Hotel: ε = -0.8** (inelastic — business travelers are less price-sensitive)

| Source | Hotel Type | Elasticity Range |
|--------|-----------|-----------------|
| Abrate et al. (2012) | Resort | -1.2 to -1.8 |
| Masiero et al. (2015) | Business/City | -0.6 to -1.1 |
| Schramm-Schulz (2019) | Mixed | -0.8 to -1.4 |

**For Melco**: Run A/B price tests to estimate YOUR OWN elasticity cleanly. Random price variation eliminates the endogeneity problem.

### 4. Dynamic Pricing = Forecast × Elasticity Adjustment

| Hotel | Baseline ADR | Optimal ADR | Revenue Impact | Strategy |
|-------|-------------|-------------|---------------|----------|
| Resort (ε=-1.4) | €163 | €136 | **+7.7%** | Lower price → fill more rooms → more revenue |
| City (ε=-0.8) | €138 | €165 | **+3.7%** | Raise price → inelastic demand barely drops → more revenue |

Key constraints for realism:
- **Capacity constraint**: Can't sell more rooms than you have
- **Price cap**: Limit markup to +20% over forecast (business prudence)

### 5. Translating to Melco/ADT

| This Notebook | Melco/ADT Equivalent |
|--------------|---------------------|
| `adr` (Average Daily Rate) | Room rate from ADT |
| `market_segment` | Channel / player tier |
| `lead_time` | Advance purchase window |
| `arrival_date_month` | Season / event calendar |
| Literature elasticity | A/B test elasticity from ADT experiments |

---

## Suggested Training Agenda (90 min)

| Time | Topic | Activity |
|------|-------|----------|
| 0–10 min | Intro & dataset overview | Walk through data dictionary |
| 10–25 min | EDA | Run cells 4–8, discuss seasonal patterns |
| 25–45 min | SARIMA modeling | Run cells 16–23, interpret forecast & evaluation |
| 45–65 min | Elasticity modeling | Run cells 24–32, discuss endogeneity pitfall |
| 65–80 min | Dynamic pricing | Run cells 33–37, review pricing schedule |
| 80–90 min | Discussion | How to apply to Melco data, A/B test design |

---

## Dependencies

```
pandas
numpy
matplotlib
seaborn
statsmodels
scikit-learn
```

Managed via `uv` — see `pyproject.toml`.
