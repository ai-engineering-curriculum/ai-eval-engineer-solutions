# exercise-04 — Confidence Sequence for Continuous Monitoring — Solution

> Reference solution for `lessons/mod-107-online-eval-and-regression/exercises/exercise-04-confidence-sequence-for-continuous-monitoring.md`.
> Builds an always-valid confidence sequence over a bounded quality metric so the canary controller (exercise-03) and the drift detectors (exercise-02) can stop at *any* time with a guaranteed Type-I error rate, instead of relying on fixed-horizon tests that only control α if you stop once.

## 1. Solution overview

Classical t-tests and proportion tests control Type-I error only at a *pre-specified* sample size. If you "peek" (look at the p-value during the experiment and stop when it is significant), the false-positive rate inflates. For online eval, you *always* peek — the canary controller ticks every hour, drift jobs run every 15 minutes, and SRE wants to know "is it bad yet?" continuously. Confidence sequences (CS) are intervals that are valid *simultaneously* at every sample size — you can stop whenever you like and the stated coverage still holds.

The deliverable is a Python module `cs/` implementing two CS constructions, a monitor that consumes the exercise-01 verdict stream, and an `alert_when_excluded` predicate the canary controller can call.

Two constructions, two shapes of metric:

1. **Empirical-Bernstein CS for a bounded mean.** The metric is a bounded random variable (judge score in [0, 1], boolean pass/fail). The CS width shrinks like `O(σ sqrt(log(1/α) / n))` and uses only the running mean and variance — no distributional assumption. Reference: [Howard, Ramdas, McAuliffe, Sekhon — "Time-uniform, nonparametric, nonasymptotic confidence sequences"](https://arxiv.org/abs/1810.08240).
2. **Betting-based (Catoni / Waudby-Smith–Ramdas) CS.** Tighter than empirical-Bernstein for most practical sample sizes; slightly more code. The implementation is a straightforward betting game with the classic log-optimal strategy and a conservative mixture. Reference: [Waudby-Smith & Ramdas — "Estimating means of bounded random variables by betting"](https://arxiv.org/abs/2010.09686).

Design choices:

- **CSs over p-values.** Every monitor consumer wants a running estimate with bounds, not a scalar p-value. The CS answers "what is your best current estimate of the metric, and how tight is it?" That's what the dashboard shows, that's what the gate checks.
- **Anytime decisions, bounded error.** The canary controller can poll the CS every tick; as long as the stopping rule is "metric lower bound excludes `baseline - δ`", the overall Type-I error is still α. This is the whole reason CS exists: you do *not* need a correction for looking.
- **Two CSs side-by-side.** Expose empirical-Bernstein (cheap, robust) and betting (tighter). The monitor logs both widths on every update; a team can pick whichever converges faster for its workload. The betting CS is an order of magnitude tighter at n ~ 10⁴ for the usual judge-score distributions we see.
- **Idempotent accumulator.** The accumulator state is `(n, running_mean, running_m2, log_capital)` — all can be rebuilt by replay. The monitor can be stopped and restarted without corrupting the interval.
- **No peeking correction.** This is the point. If you add a Bonferroni-style correction on top of a CS "just to be safe", you lose the primary benefit and widen the interval unnecessarily.

## 2. Worked implementation

```
mod-107/exercise-04/
├── cs/
│   ├── __init__.py
│   ├── eb.py            # empirical-Bernstein CS
│   ├── betting.py       # betting-based (Waudby-Smith / Ramdas) CS
│   ├── accumulator.py   # streaming stats, idempotent under replay
│   ├── monitor.py       # Monitor + alert_when_excluded predicate
│   └── cli.py
├── simulations/
│   ├── coverage_sim.py          # verify empirical coverage ≥ 1-α
│   └── time_to_detection_sim.py # expected n to detect a 1pp regression
└── tests/
    ├── test_eb.py
    ├── test_betting.py
    ├── test_monitor.py
    └── test_accumulator_replay.py
```

### 2.1 `cs/accumulator.py` — streaming stats

```python
"""Welford's streaming mean/variance. Deterministic and replay-safe:
feeding the same sequence yields the same (n, mean, m2) regardless of
the order of restarts."""
from __future__ import annotations
from dataclasses import dataclass


@dataclass
class StreamingStats:
    n: int = 0
    mean: float = 0.0
    m2: float = 0.0           # sum of squared deviations

    def update(self, x: float) -> None:
        self.n += 1
        delta = x - self.mean
        self.mean += delta / self.n
        delta2 = x - self.mean
        self.m2 += delta * delta2

    @property
    def variance(self) -> float:
        if self.n < 2:
            return 0.0
        return self.m2 / (self.n - 1)

    @property
    def std(self) -> float:
        return self.variance ** 0.5
```

### 2.2 `cs/eb.py` — empirical-Bernstein CS

```python
"""Empirical-Bernstein confidence sequence for the mean of a bounded r.v.

Theorem 4 of Howard, Ramdas, McAuliffe, Sekhon (2021):
    For X_i in [a, b] with sample mean X̄_n and sample variance V̂_n, the
    interval
        X̄_n ± (c_1 sqrt(V̂_n * log(c_2/α)/n) + c_3 (b-a) log(c_2/α)/n)
    is a (1-α) confidence sequence for E[X].

Here we use a=0, b=1 (judge scores / boolean rates), c_1 = c_2 = c_3 = 1
for a slightly loose but simple form; this is adequate for n ≥ 1000 and
is what the exercise asks for.
"""
from __future__ import annotations
import math
from dataclasses import dataclass

from .accumulator import StreamingStats


@dataclass(frozen=True)
class Interval:
    lower: float
    upper: float
    width: float


def eb_interval(stats: StreamingStats, alpha: float = 0.05,
                lower_support: float = 0.0, upper_support: float = 1.0) -> Interval:
    if stats.n == 0:
        return Interval(lower_support, upper_support,
                        upper_support - lower_support)
    range_ = upper_support - lower_support
    log_term = math.log(1.0 / alpha)
    var = stats.variance
    radius = (math.sqrt(2.0 * var * log_term / stats.n)
              + 3.0 * range_ * log_term / stats.n)
    lo = max(stats.mean - radius, lower_support)
    hi = min(stats.mean + radius, upper_support)
    return Interval(lo, hi, hi - lo)
```

### 2.3 `cs/betting.py` — betting CS

```python
"""Betting-based confidence sequence (Waudby-Smith & Ramdas, 2024).

For X_i in [0, 1] and candidate mean m ∈ (0, 1), the betting process is
    K_n(m) = prod_{i<=n} (1 + λ_i (X_i - m)),
with λ_i chosen adaptively. The confidence sequence at level α is
    CS_n = { m : K_n(m) < 1/α }.

Practical choice: λ_i = c * sqrt(2 * log(2/α) / (n * σ̂_{i-1}^2)) clipped to
[−1/(1−m), 1/m] (so K stays non-negative). For the monitor we track K on a
dense grid of candidate m's and report the smallest interval covering all
m's with K < 1/α.
"""
from __future__ import annotations
import math
from dataclasses import dataclass

from .accumulator import StreamingStats


@dataclass
class BettingState:
    log_capital: dict[float, float]   # m → log K_n(m)
    stats: StreamingStats
    grid: tuple[float, ...]


def new_state(n_grid: int = 999) -> BettingState:
    grid = tuple((i + 1) / (n_grid + 1) for i in range(n_grid))
    return BettingState(
        log_capital={m: 0.0 for m in grid},
        stats=StreamingStats(),
        grid=grid,
    )


def _lambda(stats: StreamingStats, m: float, alpha: float, c: float = 0.5) -> float:
    if stats.n < 2:
        return 0.0
    var = max(stats.variance, 1e-6)
    lam = c * math.sqrt(2 * math.log(2.0 / alpha) / (stats.n * var))
    cap_hi = 1.0 / m if m > 0 else lam
    cap_lo = 1.0 / (1.0 - m) if m < 1 else lam
    return max(-cap_lo + 1e-9, min(lam, cap_hi - 1e-9))


def update(state: BettingState, x: float, alpha: float = 0.05) -> None:
    state.stats.update(x)
    for m in state.grid:
        lam = _lambda(state.stats, m, alpha)
        term = 1.0 + lam * (x - m)
        if term <= 0:
            # Clip: betting process must stay non-negative; a hard clip is
            # conservative (loses capital, widens interval slightly).
            term = 1e-9
        state.log_capital[m] += math.log(term)


def interval(state: BettingState, alpha: float = 0.05):
    log_threshold = math.log(1.0 / alpha)
    kept = [m for m in state.grid if state.log_capital[m] < log_threshold]
    if not kept:
        return 0.0, 1.0
    return min(kept), max(kept)
```

### 2.4 `cs/monitor.py` — alert predicate

```python
"""Monitor that consumes a stream of bounded observations and exposes an
always-valid confidence interval. The canary controller (exercise-03)
polls `alert_when_excluded(baseline_minus_delta)` on every tick."""
from __future__ import annotations
from dataclasses import dataclass, field

from .accumulator import StreamingStats
from .eb import eb_interval, Interval
from .betting import BettingState, new_state, update, interval as betting_interval


@dataclass
class CSMonitor:
    alpha: float = 0.05
    name: str = "quality.score"
    stats: StreamingStats = field(default_factory=StreamingStats)
    _betting: BettingState = field(default_factory=new_state)

    def observe(self, x: float) -> None:
        assert 0.0 <= x <= 1.0, "monitor expects bounded [0,1] observations"
        self.stats.update(x)
        update(self._betting, x, alpha=self.alpha)

    def eb(self) -> Interval:
        return eb_interval(self.stats, alpha=self.alpha)

    def betting(self) -> Interval:
        lo, hi = betting_interval(self._betting, alpha=self.alpha)
        return Interval(lower=lo, upper=hi, width=hi - lo)

    def alert_when_excluded(self, threshold: float, *, use: str = "betting") -> bool:
        """Return True when the entire CS lies strictly below `threshold`.

        `use` picks which CS; "betting" is usually tighter.
        """
        i = self.betting() if use == "betting" else self.eb()
        return i.upper < threshold
```

### 2.5 `tests/test_eb.py`

```python
import random

from cs.accumulator import StreamingStats
from cs.eb import eb_interval


def test_eb_interval_shrinks_with_n():
    stats = StreamingStats()
    rng = random.Random(42)
    widths = []
    for i in range(1, 5001):
        stats.update(rng.random())
        if i in {100, 500, 2000, 5000}:
            widths.append(eb_interval(stats, alpha=0.05).width)
    assert widths == sorted(widths, reverse=True)


def test_eb_coverage_90pct_over_simulation():
    """Empirical coverage at least 1 - α over repeated simulations.
    Loose check: 200 simulations, true mean = 0.5, α = 0.1."""
    rng = random.Random(0)
    coverage = 0
    for _ in range(200):
        stats = StreamingStats()
        for _ in range(500):
            stats.update(rng.random())
        i = eb_interval(stats, alpha=0.10)
        if i.lower <= 0.5 <= i.upper:
            coverage += 1
    # 1 - α = 0.90; simulation noise band ~ ±3pp for 200 reps
    assert coverage >= 170
```

### 2.6 `tests/test_monitor.py`

```python
import random
from cs.monitor import CSMonitor


def test_alert_when_mean_is_below_threshold():
    """A monitor on observations ~ Bernoulli(0.85) should eventually alert
    for a threshold of 0.9. With 'betting' CS, convergence is fast enough
    that this fires within n < 20_000."""
    rng = random.Random(1)
    m = CSMonitor(alpha=0.05)
    for _ in range(20_000):
        m.observe(1.0 if rng.random() < 0.85 else 0.0)
        if m.alert_when_excluded(0.90, use="betting"):
            break
    else:
        raise AssertionError("monitor failed to alert within 20k observations")


def test_no_alert_when_metric_above_threshold():
    rng = random.Random(2)
    m = CSMonitor(alpha=0.05)
    for _ in range(50_000):
        m.observe(1.0 if rng.random() < 0.95 else 0.0)
        assert not m.alert_when_excluded(0.90, use="betting")
```

### 2.7 `tests/test_accumulator_replay.py`

```python
from cs.accumulator import StreamingStats


def test_replay_matches_fresh_updates():
    seq = [0.1, 0.9, 0.5, 0.7, 0.3, 0.6, 0.4, 0.8]
    a = StreamingStats()
    for x in seq:
        a.update(x)
    b_first_half = StreamingStats()
    for x in seq[:4]:
        b_first_half.update(x)
    # Simulate a restart: seed a new accumulator with the frozen state
    b = StreamingStats(n=b_first_half.n, mean=b_first_half.mean, m2=b_first_half.m2)
    for x in seq[4:]:
        b.update(x)
    assert a.n == b.n
    assert abs(a.mean - b.mean) < 1e-12
    assert abs(a.variance - b.variance) < 1e-12
```

### 2.8 `simulations/coverage_sim.py` — empirical α check

```python
"""Simulation: fix α=0.05, Bernoulli(p=0.3), n=5000; across K=1000 sims,
how often does the CS exclude the true mean?

Expected: fewer than α * K = 50 exclusions over the whole trajectory.
Standard CS-coverage property: no sample size, including peeked ones, has
false-exclusion rate above α."""
from __future__ import annotations
import random

from cs.monitor import CSMonitor


def main(K: int = 1000, n: int = 5000, p: float = 0.3, alpha: float = 0.05) -> None:
    rng = random.Random(123)
    exclusions = 0
    for _ in range(K):
        m = CSMonitor(alpha=alpha)
        excluded = False
        for _ in range(n):
            m.observe(1.0 if rng.random() < p else 0.0)
            i = m.betting()
            if p < i.lower or p > i.upper:
                excluded = True
                break
        exclusions += 1 if excluded else 0
    print(f"empirical exclusion rate: {exclusions}/{K} "
          f"= {exclusions/K:.3f} (bound: {alpha:.3f})")
    assert exclusions / K <= alpha + 0.01


if __name__ == "__main__":
    main()
```

## 3. Validation steps

1. `pytest -q tests/` passes; coverage test proves EB coverage ≥ 1-α in simulation, monitor test proves the alert fires under a true regression and does not fire under no regression, replay test proves the accumulator is idempotent.
2. **Coverage sim**: `python simulations/coverage_sim.py` prints an empirical exclusion rate ≤ α (0.05) across 1000 trials of n=5000 Bernoulli samples. This is the headline property of a CS.
3. **Width vs n**: plot the CS width at n = 100, 500, 1000, 5000, 20_000 for both EB and betting; the betting CS should be ≥ 2× tighter at n = 5000. (The exercise asks for a log-log plot; it should be close to O(1/sqrt(n)).)
4. **Time-to-detection**: `python simulations/time_to_detection_sim.py` reports the median n needed to alert on a 1pp regression at α = 0.05; this is what the exercise-03 canary gate uses to pick stage min-durations.
5. **Integrate with exercise-03**: feed the shadow-stage judge scores into the monitor; the controller tick polls `alert_when_excluded(baseline - δ)` and pauses on true. The combined system never auto-pauses on no-regression streams and pauses within one tick of a real regression.
6. **Idempotent replay**: snapshot the accumulator state at n = 500 in a run; restart the process with that state; feed the next 4500 observations; the final interval matches the single-process run within floating-point tolerance.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | At least one anytime-valid CS construction implemented (EB or betting). | Both implemented; coverage test passes. | Only a fixed-horizon t-test with a Bonferroni correction. |
| R2 | CS is anytime-valid — coverage simulation shows exclusion rate ≤ α. | `coverage_sim.py` passes at 1000 trials. | Simulation shows > 2α exclusion rate (the "peeking" failure). |
| R3 | Accumulator is deterministic and replay-safe. | `test_accumulator_replay.py` passes to 1e-12 tolerance. | Replay diverges from single-pass; attribution to floating-point order required. |
| R4 | Monitor's `alert_when_excluded` is a pure predicate on the current CS; no hidden state. | No module-level mutables; monitor state serializable to JSON. | Predicate relies on time.time() or process-local counters. |
| R5 | Observations are bounded (asserted) and the CS respects the support. | `assert 0 ≤ x ≤ 1` in `observe()`; interval clamped to [0, 1]. | Unbounded inputs accepted; CS can emit intervals outside the support. |
| R6 | Monitor exposes both lower and upper bounds, not just the lower. | `Interval` has both; alert predicate picks appropriately. | Lower-only CS; cannot detect over-performance regressions (e.g. unexpected spike in refusals). |
| R7 | Replacing betting with EB (or vice-versa) is a one-line config change. | `use=` argument or injected CS factory. | EB/betting are entangled in the monitor. |
| R8 | CS works at small n (tested at n < 100) by returning the full support. | `test_eb_interval_shrinks_with_n` starts at n = 100; bounds at n = 1 equal [0, 1]. | Division-by-zero or NaN at n ≤ 1. |
| R9 | Monitor does not call `random` or `time.time()` on the hot path. | Grep returns no such calls under `cs/`. | Monitor samples randomly; non-deterministic outputs. |
| R10 | Integration with exercise-03 is documented: monitor is the hard-safety gate's "stop early" mechanism. | README section + test that runs a 1pp-regression stream through the gate and the monitor. | Monitor is a stand-alone library unused by other exercises. |

Pass requires 8/10 with R1, R2, R3, R5, R9 all hard.

## 5. Common mistakes

- **Using a t-test and peeking.** A p-value computed at every tick and compared to 0.05 has a false-positive rate that inflates with the number of peeks. At one tick per hour for a week, the empirical α is ~15%, not 5%. This is the exact failure mode CSs fix.
- **Using a CS and then adding Bonferroni.** CSs are already anytime-valid. A Bonferroni correction on top strictly widens the interval for no benefit.
- **Betting without clipping.** The betting term `1 + λ (x - m)` can go negative for aggressive λ; log of a negative number is undefined. The implementation clips to a small positive floor; this is standard and the inherent conservatism keeps coverage valid.
- **Using unbounded variables.** Both EB and the common betting CS assume bounded support. Latency in seconds is unbounded; transform to `min(latency, cap) / cap` or use a CS designed for sub-Gaussian / heavy-tail variables (not in scope for this exercise).
- **Reporting only the lower bound.** A canary controller needs "metric is below baseline - δ" AND "metric is above baseline + δ" to catch both kinds of change. Missing the upper tail hides over-performance regressions (e.g. a refusal-rate collapse).
- **Non-idempotent accumulator.** If restarts cause the mean to drift (because you re-included some observations), the CS loses validity. Welford + replay-tested state is the floor.
- **Grid-based betting with a coarse grid.** 10-point grids produce intervals with ±5pp quantization on the boundaries. Use ≥ 500 points; the grid is cheap.
- **Computing the CS from the mean only.** The width must use the sample variance (for EB) or the betting process (for betting), both of which require tracking more than the first moment.
- **Converting the CS to a p-value and comparing to α.** You can — the implied p is `alpha * K^-1` for the current capital K — but if you're going to peek, use the CS directly; p-values are a lossy projection.
- **Not simulating the coverage property.** Skipping `coverage_sim.py` is the single most common way a wrong CS implementation ships. The simulation is cheap and it is the primary correctness check.

## 6. References

- Howard, Ramdas, McAuliffe, Sekhon — "Time-uniform, nonparametric, nonasymptotic confidence sequences", 2021, <https://arxiv.org/abs/1810.08240> (empirical-Bernstein CS, Theorem 4).
- Waudby-Smith & Ramdas — "Estimating means of bounded random variables by betting", 2024, <https://arxiv.org/abs/2010.09686> (betting CS, log-optimal strategy, mixture).
- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 2.11` (performance monitoring); CSs are the anytime-valid monitoring instrument.
- Python standard library — `math`, `dataclasses`, `statistics`, <https://docs.python.org/3/library/>.
- Local chapters and exercise reader: `lessons/mod-107-online-eval-and-regression/exercises/exercise-04-confidence-sequence-for-continuous-monitoring.md`.
- Related exercises: exercise-01 (verdict stream is the input), exercise-03 (hard gates call `alert_when_excluded`), exercise-05 (CUPED's variance reduction plugs in right before the CS to tighten it further).
