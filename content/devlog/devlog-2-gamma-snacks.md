+++
title = "Devlog #2: Gamma Snacks at Midnight"
date = 2025-12-12
description = "CQF done, still learning daily—let’s unpack vanilla options, Black–Scholes math, code, and how the trade-pricer-platform hangs together."
[taxonomies]
tags = ["options", "black-scholes", "pricing", "greeks", "python", "panel", "devlog"]
+++

**Catchy start:** CQF graduation badge unlocked—still very much a student every day, so here’s a brain-dump while it’s fresh.

## Vanilla options, quick intuition
- **Call**: right (not obligation) to **buy** the underlying at strike $K$ on expiry $T$. Payoff: $\max(S_T - K, 0)$.
- **Put**: right to **sell** at $K$. Payoff: $\max(K - S_T, 0)$.
- European style: exercise **only at expiry**; no early exercise, no path dependence.

## Black–Scholes refresher (continuous carry \(q\))
- Pricing a European call:
  $$
  C = S_0 e^{-qT}\Phi(d_1) - K e^{-rT}\Phi(d_2)
  $$
  $$
  d_{1,2} = \frac{\ln(S_0/K) + (r - q \pm \tfrac{1}{2}\sigma^2)T}{\sigma\sqrt{T}}
  $$
  Put via put-call parity: $P = C - S_0 e^{-qT} + K e^{-rT}$.
- Key Greeks (per unit):
  - Delta (call): \(e^{-qT}\Phi(d_1)\)
  - Gamma: $\frac{e^{-qT}\phi(d_1)}{S_0 \sigma \sqrt{T}}$
  - Vega: $S_0 e^{-qT} \phi(d_1) \sqrt{T}$
  - Theta, Rho similarly from BS differentials.

### Parameters at a glance
- $S_0$: spot, $K$: strike, $T$: years to expiry, $r$: risk-free (cont.), $q$: dividend/foreign yield, $\sigma$: implied vol.
- All outputs scale by notional; without notional they’re per-unit.

## Tiny code slice (from trade-pricer-platform)
```python
from trade_pricer_platform import EuropeanOption, price_and_greeks

opt = EuropeanOption(
    option_type="call",
    spot=1.10,
    strike=1.09,
    maturity=1.0,
    rate=0.01,
    volatility=0.20,
    dividend_yield=0.0,
    notional=1_000_000,
)
metrics = price_and_greeks(opt)
print(metrics)
```

And if you want the pure-formula numbers (without the library), here’s a minimal Black–Scholes call in Python:

```python
import math
from math import log, sqrt, exp
from mpmath import quad

def phi(x):  # standard normal PDF
    return math.exp(-0.5 * x * x) / math.sqrt(2 * math.pi)

def Phi(x):  # standard normal CDF via numeric integration
    return 0.5 * (1 + math.erf(x / math.sqrt(2)))

def bs_call(S, K, T, r, q, sigma):
    d1 = (log(S / K) + (r - q + 0.5 * sigma * sigma) * T) / (sigma * sqrt(T))
    d2 = d1 - sigma * sqrt(T)
    price = S * exp(-q * T) * Phi(d1) - K * exp(-r * T) * Phi(d2)
    delta = exp(-q * T) * Phi(d1)
    gamma = exp(-q * T) * phi(d1) / (S * sigma * sqrt(T))
    vega = S * exp(-q * T) * phi(d1) * sqrt(T)
    theta = (
        -S * exp(-q * T) * phi(d1) * sigma / (2 * sqrt(T))
        + q * S * exp(-q * T) * Phi(d1)
        - r * K * exp(-r * T) * Phi(d2)
    )
    rho = K * T * exp(-r * T) * Phi(d2)
    return price, delta, gamma, vega, theta, rho

print(bs_call(1.10, 1.09, 1.0, 0.01, 0.0, 0.20))
```

That yields the same numbers you see in the UI/CLI, making the mapping between formulas and code explicit.

### Quick numeric example (per unit; multiply by notional to scale)
- Inputs: $S_0=1.10, K=1.09, T=1.0, r=1\%, q=0, \sigma=20\%$.
- Approx outputs: price $\approx 0.098$, delta $\approx 0.58$, gamma $\approx 1.78$, vega $\approx 0.43$ (theta/rho from the code above).
- With a 1,000,000 notional, price is about $98{,}000$ and vega about $430{,}000$ (per vol point).

### Spicy math derivation (note to self)
- Start with risk-neutral GBM with carry: $dS_t = (r - q) S_t\,dt + \sigma S_t\,dW_t$.
- Apply Ito to $V(S,t)$, delta-hedge with $\Delta = V_S$, enforce drift = $r$ to kill arbitrage → BS PDE:
  $$V_t + \tfrac12 \sigma^2 S^2 V_{SS} + (r - q) S V_S - r V = 0.$$
- Boundary conditions for a European call: $V(S,T)=\max(S-K,0)$, $V(0,t)=0$, $V\to S$ as $S\to\infty$.
- Solve PDE / Feynman–Kac → closed-form:
  $$C = S_0 e^{-qT}\Phi(d_1) - K e^{-rT}\Phi(d_2),$$
  $$d_{1,2} = \frac{\ln(S_0/K) + (r - q \pm \tfrac12 \sigma^2)T}{\sigma \sqrt{T}}.$$
- Put via put–call parity: $P = C - S_0 e^{-qT} + K e^{-rT}$.

### Gamma and hedging flows (dealer dynamics)
- Gamma measures how fast delta changes per unit move in spot. Large magnitude gamma means deltas swing quickly, forcing frequent hedge adjustments.
- Dealers long gamma (e.g., long options) buy dips/sell rips when hedging, dampening moves; short gamma hedging does the opposite, amplifying moves.
- Around big strikes/expiries, aggregate dealer gamma can pin spot (long gamma) or fuel breakouts (short gamma) as hedges are rebalanced.
- Vol spikes often coincide with dealers flipping from long to short gamma; the hedging flow becomes pro-cyclical, reinforcing the underlying move.

## How vanilla European options work (cash-settled intuition)
- Pay premium up front; at expiry, payoff depends only on terminal $S_T$.
- No early exercise; continuous rates/discounting simplify to closed-form BS.
- For FX, \(q\) acts like foreign yield; for equities, \(q\) can represent dividends.

## Building the project (tech + assumptions)
- **Tech stack**: Python 3.12, `panel` + `bokeh` UI, `holoviews` plots, `httpx`/`yfinance`/FRED for data, `uv` for project management.
- **Assumptions**: flat vol, lognormal diffusion, continuous rates/dividends, European exercise only, no funding/credit/quanto.
- **UI bits**: scenario sliders for spot/vol/rate; heatmaps (spot × vol); payoff curves; synthetic smile/uploadable; history fetch; dark/light toggle.
- **Limitations**: no smile surface, no early exercise (so no American/barriers), no stochastic vol/rates, live data is best-effort via Yahoo/FRED.
- **Safety/UX**: caching on data fetches, dark/light theme toggle, CSS fixes for readability.
- **Tests**: unit tests for pricing, mocked market-data tests (Yahoo/FRED) so no network is needed when running the suite.

## Plots you can see in the app
- Spot–vol heatmaps (price/delta/vega), payoff overlays, smile (uploadable), term ladders, basic history curve.

## How to enhance next
- Add smile/skew support (surface input), greeks under smile.
- Add Greeks ladders per shock bucket and CSV export.
- Plug in alternative data providers; cache/async fetches; retry/backoff.
- More risk views: P/L vs shocks, scenario sets, portfolio aggregation.
 - Add embedded numeric outputs next to plots, e.g., showing price/delta/vega at the hovered point on the heatmap.

Thanks for reading; more to come once I ship the next set of improvements.***

## Project link
- Code: [Trade Pricer Platform](https://github.com/Sachin-Bhat/trade-pricer-platform)
