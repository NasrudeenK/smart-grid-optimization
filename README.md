# Smart Grid Optimization — Forecast-Driven Battery Dispatch

Day-ahead electricity-load forecasting, Time-of-Use cost simulation, and rolling
battery dispatch on the **PJM East (PJME)** hourly load dataset. The project
quantifies a question that matters economically but is usually skipped: **how much
of a battery's theoretical savings does a real forecast actually capture?**

**Headline result:** a day-ahead load forecast (7.6% MAPE) captures **75%** of the
battery-dispatch savings available under perfect foresight. The remaining 25% —
about **$87K over the 60-day test period (~$10K/week)** for an 18.6 MW battery —
is the cost of forecast error.

---

## Why this framing

A battery scheduled against a *perfect* forecast is a fiction: in operation you
commit each day's charge/discharge schedule against a **prediction**, then live
with whatever load actually shows up. Reporting perfect-foresight savings alone
overstates what any real system achieves.

This project separates the two explicitly, day by day:

- **Rolling day-ahead schedule** — each day, the battery is optimized against that
  day's load *forecast*. That fixed schedule is then executed against *actual*
  demand through a physical simulation that enforces real state-of-charge limits:
  if the forecast was wrong and the plan calls for discharging energy the battery
  doesn't have, the realized dispatch falls short. This is where forecast error
  turns into lost savings.
- **Perfect-foresight benchmark** — each day re-optimized directly on actual
  demand. Unattainable by construction; it's the ceiling.

The gap between them is the economic value lost to forecast error. Realized
savings as a fraction of ideal savings is the schedule's **capture rate** — the
single number that connects model accuracy to dollars.

---

## Pipeline

```
EDA  →  day-ahead forecast (XGBoost)  →  ToU cost sim  →  rolling battery dispatch (LP)  →  capture-rate benchmark
```

1. **Data & EDA** — load, sort, drop duplicate DST-boundary timestamps, enforce an
   hourly calendar; daily/weekly/seasonal profiles and an additive decomposition.
2. **Forecasting** — XGBoost on calendar, cyclical, and lagged-load features. The
   feature set **excludes prior-hour load and rolling statistics**, so every
   feature is observable when a next-day hourly schedule is committed. A genuine
   day-ahead model, not a one-hour-ahead operational forecast.
3. **Cost simulation** — Time-of-Use tariff with peak-shifting under per-day
   energy conservation.
4. **Rolling battery dispatch** — for each day, a linear program (PuLP/CBC)
   minimizes energy cost **plus a daily demand charge** on peak grid draw, subject
   to power/capacity limits, √-split round-trip efficiency, and a cyclic
   state-of-charge constraint. Planned schedules are then run through a physical
   realization step against actual demand.
5. **Benchmark** — rolling day-ahead vs perfect-foresight, reported as a capture rate.

---

## Results

The final **60 days** are held out untouched; the prior 30 days tune early
stopping only. The battery is evaluated over the same full 60-day test period.

### Forecast (day-ahead, 60-day holdout)

| Metric | Value |
|---|---|
| Test RMSE | 3,452.9 MW |
| Test MAPE | 7.59 % |
| 5-fold CV mean RMSE | 2,347.0 MW (± 686.2) |

### Battery dispatch (60-day test period, stylized C&I / DER actor peaking ~57 MW)

| Metric | Value |
|---|---|
| Battery power / energy | 18.6 MW / 74.4 MWh |
| Baseline cost (no battery) | $4,667,262 |
| Rolling day-ahead realized savings | $261,965 (5.6% of bill) |
| Perfect-foresight savings | $349,356 (7.5% of bill) |
| Forecast-error opportunity gap | $87,390 |
| **Capture rate** | **75.0 %** |

The actor is modeled at **0.1% of PJME zone load**. The battery is sized at **30%
of the actor's training-period peak** (4-hour duration), so no test-period data
influences the asset design. Scaling to a single actor rather than the full 50+ GW
zone is what lets the battery be large enough relative to its host for demand to
actually bind — otherwise the schedule degenerates into pure price arbitrage and
the forecast plays no role.

---

## Key modeling decisions

- **No leakage.** Lagged features are timestamp-based (a missing hour maps to
  NaN, never a shifted wrong row), and no feature depends on the immediately
  preceding hour — required for an honest day-ahead claim.
- **Rolling per-day horizon.** Each day is scheduled independently, matching how
  day-ahead commitment actually works — the battery can't plan Monday's charging
  around Friday's prices.
- **Realized-dispatch physics.** The planned schedule is executed against actual
  demand with real SOC limits enforced hour by hour, so a mis-forecast schedule
  genuinely underperforms rather than being assumed executable.
- **Realistic demand charge.** ~$20/kW-month, applied as ≈$667/MW on each day's
  peak draw. Demand charges make up ~36% of the baseline bill, a typical share for
  a commercial/industrial customer. They make *when* the battery discharges depend
  on *when the actual peak lands* — precisely what the forecast gets wrong.
- **One cost formula everywhere.** Baseline, realized, and perfect-foresight costs
  all use `energy + demand_charge × daily_peak`, so the comparison is apples-to-apples.
- **√-split round-trip efficiency.** Loss is split across charge and discharge legs
  (`η = √0.92`), so a round trip loses 8% total — the standard convention.
- **Cyclic SOC.** Each planned day ends at its starting charge, preventing phantom
  savings from spending down initial stored energy.

---

## Stack

Python · XGBoost · PuLP (CBC) · pandas · scikit-learn · statsmodels · matplotlib / seaborn

## Data

PJM East hourly load (`PJME_hourly.csv`) — [Hourly Energy Consumption, Kaggle](https://www.kaggle.com/datasets/robikscube/hourly-energy-consumption).
Place the CSV at `data/PJME_hourly.csv` (or edit the `DATA` path in the first cell).

## Run

```bash
pip install -r requirements.txt
jupyter notebook Analysis.ipynb   # Kernel → Restart & Run All
```

---

## Scope & limitations

- This is an **operating-cost simulation on public data**, not a deployed system
  or an investment case. A real business case would add capex, battery
  degradation, interconnection limits, and formal asset sizing.
- The realization step restarts each day at 50% state of charge rather than
  carrying the realized end-of-day charge forward; since realized charge usually
  ends at or above plan, this slightly understates achievable savings.
- Forecast accuracy (~7.6% day-ahead MAPE) would improve materially with weather
  features — the main unmodeled driver of load — which is the clearest next step
  and would raise the capture rate further.

The capture rate is the transferable finding: it prices forecast error against a
storage asset as a *method*, independent of these refinements.
