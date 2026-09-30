# Option Pricing: Black-Scholes vs Monte Carlo Methods

Pricing a European call and a binary cash-or-nothing call under geometric Brownian motion, and checking Monte Carlo estimates against the Black-Scholes closed form. The notebook goes beyond "MC ≈ BS". It measures discretisation bias separately from sampling noise and shows which of the two actually limits accuracy.

## What's in the repo

| File | Contents |
|---|---|
| `Option Pricing with Monte Carlo Simulation and BS Model.ipynb` | The main study: BS benchmarks, Euler-Maruyama / Milstein / exact simulation, convergence analysis, antithetic variates, sensitivity analysis |
| `Options & Greeks Interview Prep.pdf` | 110 interview questions on options and Greeks for quant, valuation and market-risk roles. 
| `Practical Solutions` | Notes on validating FDM prices and Greeks for American and exotic options when there is no closed form or liquid market price |

## Set-up

| Option | Payoff at $T$ | Black-Scholes price |
|---|---|---|
| European call | $\max(S_T-K,\,0)$ | $S_0N(d_1)-Ke^{-rT}N(d_2)$ |
| Binary cash-or-nothing call (pays $K$) | $K\cdot\mathbb{1}_{\{S_T>K\}}$ | $Ke^{-rT}N(d_2)$ |

Base case: $S_0=100$, $K=105$, $r=5\%$, $\sigma=20\%$, $T=1$. BS prices: **call 8.0214**, **binary 46.2015**.

**Methods compared**

- **Euler-Maruyama** (strong order 0.5, weak order 1)
- **Milstein** (strong order 1, weak order 1)
- **Exact log-normal simulation**, which has no discretisation error and serves as the reference
- **Antithetic variates** for variance reduction, at equal path budget

## Approach

1. **Fair comparison.** Every method gets the same budget (200,000 paths × 252 steps). Each estimate is reported with its standard error and a z-score, `z = (MC − BS) / SE`.
2. **Common random numbers for the bias.** A fresh simulation at each step count mixes discretisation bias with MC noise (SE ≈ 0.03), which swamps a bias of order 10⁻³. The notebook generates one fine Brownian path (256 steps), sums it into coarser grids, and drives every scheme and the exact solution with the same path. The noise mostly cancels, so the weak and strong error curves are clean.
3. **Memory-efficient engine.** Only the current price vector is kept, not the full path matrix (about 600 MB at 150k × 500 would be needed otherwise). Each run has its own seeded `numpy.random.Generator`, so results don't depend on cell order.

## Key results

**All methods agree with Black-Scholes within sampling error.** All 12 base-case estimates have $|z|<2$, and so do all 68 points of the sensitivity grid.

**Discretisation bias behaves as theory predicts.**

| | Euler-Maruyama | Milstein |
|---|---|---|
| Strong order (fitted) | ≈ 0.5 | ≈ 1.0 |
| Weak order, European call (fitted) | ≈ 1.0 | ≈ 1.0 |
| Weak error, call, 256 steps | ≈ −0.001 | ≈ −0.002 |
| Weak error, binary, 256 steps | ≈ +0.03 | below noise |

Milstein's pathwise error is about 40× smaller at 256 steps. It gives no price improvement for the smooth call payoff, but a large one for the digital, whose value depends on which side of $K$ the path ends.

**Sampling error, not discretisation, is the binding constraint.** At 252 steps the bias is ≤ 0.002 for the call, against an SE of about 0.024–0.030. Reaching SE = 0.01 with antithetic sampling needs about **1.1M paths for the call** and **3.5M for the binary**. More time steps beyond ~50 don't help the price.

**Antithetic variates help unevenly.**

| Option | SE reduction | Variance reduction factor |
|---|---|---|
| European call | ≈ 20% | ≈ 1.6 |
| Binary CoN | ≈ 62% | ≈ 7.0 |

The binary payoff is close to monotone in $Z$ near the money, so the $Z$ and $-Z$ payoffs are strongly negatively correlated.

**Sensitivities match the Greeks.** The call is convex in $S_0$ and increasing in $T$, $r$ and $\sigma$. With $S_0<K$, the binary's value *falls* as volatility rises (49.9 at σ = 5% down to 40.2 at σ = 50%), because the sign of a digital's vega depends on moneyness.

## Practical takeaways

- Under GBM, simulate $S_T$ exactly. Time-stepping is only needed for path-dependent payoffs or models without a closed-form solution.
- When a scheme is needed, use Milstein for discontinuous payoffs (digitals, barriers). For smooth payoffs Euler's weak accuracy is equivalent.
- Always report the standard error, and size the number of paths from a target SE rather than a rule of thumb.

## Validating prices without a closed form

`Practical Solutions` covers the harder case: American and exotic options priced by FDM, often in illiquid markets with no observable prices.

- **Numerical-method cross-validation:** compare binomial tree, FDM and Monte Carlo (including Longstaff-Schwartz) results.
- **Finite-difference bumping for Greeks:** test stability across bump sizes and against algorithmic differentiation.
- **Degeneracy testing:** reduce the structure to vanilla limits, in the spirit of Carr's static hedging, and compare values.

## Running the notebook

```bash
pip install numpy pandas matplotlib scipy jupyter
jupyter notebook "Option Pricing with Monte Carlo Simulation and BS Model.ipynb"
```

Developed with Python 3.11. A full run takes a few minutes, mostly in the sensitivity grid.

## References

- Black, F. & Scholes, M. (1973). The pricing of options and corporate liabilities. *Journal of Political Economy*, 81(3), 637–654.
- Merton, R. C. (1973). Theory of rational option pricing. *Bell Journal of Economics and Management Science*, 4(1), 141–183.
- Boyle, P. P. (1977). Options: A Monte Carlo approach. *Journal of Financial Economics*, 4(3), 323–338.
- Glasserman, P. (2004). *Monte Carlo Methods in Financial Engineering*. Springer.
- Kloeden, P. E. & Platen, E. (1992). *Numerical Solution of Stochastic Differential Equations*. Springer.
- Rubinstein, M. & Reiner, E. (1991). Unscrambling the binary code. *Risk*, 4(9), 75–83.
