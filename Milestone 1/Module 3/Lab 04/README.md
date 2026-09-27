# Reliability Baseline for a URL Shortener

![URL shortener reliability baseline](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2004/images/thumb.png)

Combine Labs 01, 02, and 03 into one reliability reference. Use it to check changes in later milestones.

## Objective

```text
Model -> Calculate -> Set targets
```

**Explanation:** Shows the workflow.

By the end, you can:

- Calculate availability and equivalent nines.
- Choose a service level objective (SLO), a target for successful requests.
- Record error budgets and alert thresholds.

## Deliverable

```text
L1 calculator + L2 SLO + L3 topology -> reliability-baseline.md
```

**Explanation:** Shows the workflow.

Create `reliability-baseline.md` using Markdown and Python 3. Keep the calculator beside it. AI tools may help; verify their output.

## Required dependencies

![Four required tiers in series](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2004/images/baseline-topology.png)

Every modeled request needs Gateway, Shortener Service, Cache, and Database. The client is excluded. Real cache hits may skip Database; this simplified chain is not a worst-case guarantee.

## Parallel copies

![Two copies each of Shortener Service and Cache](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2004/images/improved-topology.png)

Either copy must serve all requests alone, and traffic must reach a working copy. Assume independent failures, no retries, no switching delay, and no shared failures. Database is one logical managed multi-AZ service; its assumed 99.99% already includes internal redundancy.

## Step 1: Write the assumptions

```text
Components + request rules + traffic -> model inputs
```

**Explanation:** Shows the workflow.

Create `reliability-baseline.md` with these planning inputs. Values are practice assumptions, not measurements or provider guarantees.

| Tier | Availability | Baseline copies | Improved copies |
| :--- | ---: | ---: | ---: |
| Gateway | 99.95% | 1 | 1 |
| Shortener Service | 99.90% | 1 | 2 |
| Cache | 99.95% | 1 | 2 |
| Database | 99.99% | 1 logical | 1 logical |

Use a rolling 30-day window: 43,200 minutes and 5,000,000 assumed valid requests. 

Carry forward Lab 02: exclude 499 responses and `/healthz`. Count remaining completed 2xx/3xx/4xx as good and 5xx as bad. Count retries separately. The service level indicator (SLI) is `good / valid`; use external checks for failures that never reach the gateway.

---

## Step 2: Write the calculator

```text
Copies -> tier availability -> composite availability
```

**Explanation:** Shows the workflow.

Create `reliability_baseline_calc.py`. Multiply required tiers; parallel copies fail only when both fail.

```python
import math

WINDOW_DAYS = 30
VALID_REQUESTS = 5_000_000
SLO_TARGETS = (99.9, 99.95)
BASELINE = {
    "Gateway": 0.9995,
    "Shortener Service": 0.999,
    "Cache": 0.9995,
    "Database": 0.9999,
}


def parallel(*copies):
    return 1 - math.prod(1 - availability for availability in copies)


def summarize(tiers):
    availability = math.prod(tiers.values())
    failed_fraction = 1 - availability
    nines = math.inf if failed_fraction == 0 else -math.log10(failed_fraction)
    downtime = failed_fraction * WINDOW_DAYS * 24 * 60
    predicted_failed = failed_fraction * VALID_REQUESTS
    return availability, nines, downtime, predicted_failed


def main():
    improved = dict(BASELINE)
    for tier in ("Shortener Service", "Cache"):
        improved[tier] = parallel(BASELINE[tier], BASELINE[tier])

    for name, tiers in (("Baseline", BASELINE), ("Improved", improved)):
        availability, nines, downtime, failures = summarize(tiers)
        print(f"{name}: {availability * 100:.4f}%, {nines:.3f} nines, "
              f"{downtime:.2f} min, {failures:.0f} predicted failed requests")
        for target in SLO_TARGETS:
            verdict = "MODEL PASS" if availability >= target / 100 else "MODEL FAIL"
            print(f"  SLO {target}%: {verdict}")

    print("\nPlanning budgets:")
    for target in SLO_TARGETS:
        allowed_fraction = (100 - target) / 100
        budget = VALID_REQUESTS * allowed_fraction
        minutes = WINDOW_DAYS * 24 * 60 * allowed_fraction
        print(f"  {target}%: {budget:.0f} failed requests, {minutes:.2f} equivalent min")

    print("\nBaseline improvement if one tier becomes perfect:")
    original = math.prod(BASELINE.values())
    for tier in BASELINE:
        changed = dict(BASELINE)
        changed[tier] = 1.0
        gain = (math.prod(changed.values()) - original) * 100
        print(f"  {tier}: +{gain:.6f} percentage points")


if __name__ == "__main__":
    main()
```

**What this file does:** Calculates both designs, budgets, and the gain from making each baseline tier perfect.

---

## Step 3: Write the SLO decision

```text
Composite availability -> compare target -> record decision
```

**Explanation:** Shows the workflow.

Keep Lab 02's 99.9% target conditional on deploying both replicated tiers. Add 99.95% as a new stretch target.

| Target | Allowed failures at 5 million requests | Equivalent minutes | Baseline | Improved |
| :--- | ---: | ---: | :--- | :--- |
| 99.9% | 5,000 | 43.20 | Fail | Pass |
| 99.95% | 2,500 | 21.60 | Fail | Fail |

The request budget is `valid requests × (1 - target / 100)`. Equivalent downtime assumes uniform traffic; busy-period outages can cause more failures. Recalculate budgets from actual traffic.

Gateway's 99.95% caps the chain. Reaching 99.95% requires improving or bypassing that dependency, then checking the whole model. Improving Database alone is insufficient.

---

## Step 4: Write the alert policy

```text
Bad / valid requests -> burn rate -> paired windows -> alert
```

**Explanation:** Shows the workflow.

Burn rate means `observed bad fraction / allowed bad fraction`. Keep Lab 02's paired windows: both must exceed the threshold.

| Alert | Long + short window | Burn rate | Bad requests | Action |
| :--- | :--- | ---: | ---: | :--- |
| Fast | 1h + 5m | >14.4x | >1.44% | Page |
| Medium | 6h + 30m | >6x | >0.60% | Page |
| Slow | 3d + 6h | >1x | >0.10% | Ticket |

These are lookback windows, not waiting periods. With steady traffic, 14.4x for one hour spends 2% of the monthly budget. If the SLO changes, recalculate request budgets and error-percentage thresholds. Review tickets the next business day; prioritize reliability fixes when the budget runs out.

---

## 5. Run the following commands in the terminal

```text
Calculator -> terminal results -> baseline document
```

**Explanation:** Shows the workflow.

Run from the folder containing your calculator.

```bash
python3 reliability_baseline_calc.py
```

**Explanation:** Prints modeled results and compares them with both targets.

![Verified Python calculator output](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2004/images/run-output.png)

**What to expect:** Baseline: 99.7901%, 2.678 nines, 90.66 minutes. Improved: 99.9399%, 3.221 nines, 25.97 minutes. Only the improved design passes 99.9%. Predicted failures are 10,493 and 3,006 under uniform traffic. These estimates are not measured SLO results.

---

## Step 6: Complete the baseline

```text
Inputs + results + policy + reflection -> reliability baseline
```

**Explanation:** Shows the workflow.

Organize `reliability-baseline.md` into scope, assumptions, topology results, SLO decision, budgets, alerts, future requirements, and reflection. Record the calculator command so others can reproduce the results.

Require later milestones to preserve the replica assumptions and recalculate after dependency changes. Track Gateway and Database as required dependencies. Record the date and reason for revisions. Compare production SLIs with this model.

Write a 3-5-line reflection: Which component assumption most affected availability? Did Lab 02's target survive, or did the architecture need revision? Use the calculator's sensitivity results as evidence.

---

## Verification checklist

```text
Recalculate -> compare -> submit
```

**Explanation:** Shows the workflow.

- [ ] Both designs match the calculator's rounded results.
- [ ] The selected SLO has a matching request budget.
- [ ] All three alerts include both windows and actions.
- [ ] Assumptions and future requirements are explicit.
- [ ] The reflection answers both questions in 3-5 lines.

## Conclusion

```text
Reliability baseline -> later milestone reviews
```

**Explanation:** Shows the workflow.

Use this baseline to review architecture changes in later milestones. Compare its predictions with measured request reliability as the service develops.
