# SLO & Error Budget Definition

Imagine 1,000 customers trying to place orders in an online shop. Each order sends a request to the **Orders API**, the part of the shop's software that handles orders. We want at least **999 of those requests to work without a server error**. That is a **99.9% success target**, called a service level objective (**SLO**).

<p align="center">
  <img src="./images/budget.png" alt="Three steps: set a 99.9% success target, allow 0.1% failures, and alert when failures use the budget too quickly" width="800">
</p>

Read the three boxes in the image from left to right:

1. **Set SLO target:** aim for at least 999 successful requests out of every 1,000 over the measurement period.
2. **Error budget:** the remaining 1 request out of 1,000 may fail. This **0.1% allowance** is the error budget.
3. **Alert on burn:** if failed requests use up this allowance too quickly, raise an alert so the team can investigate.

In this lab, you will decide which requests count as good or bad, calculate the allowed failures over 30 days, and write a rule that alerts the team when failures happen too quickly.

## Objective

- Define a service level indicator (**SLI**) to measure good requests for the Orders API.
- Set a service level objective (**SLO**) of **99.9% good requests** over the most recent 30 days.
- Calculate the **error budget**: how many failed requests are allowed within that rolling 30-day window.

## Tools/Stack

- Markdown for the SLO document.
- Prometheus query sketches for budget burn-rate alerts.
- `promtool` for the optional syntax check in Step 5.

## Deliverable

You will create two files:

- **`slo-document.md`** defines which API requests count toward the SLO, the availability target, the error budget for a rolling 30-day window, and all three burn-rate thresholds (14.4x, 6x, and 1x).
- **`orders-api-rules.yaml`** provides a Prometheus alert rule for the **14.4x fast-burn threshold**. The alert fires when the error rate exceeds 1.44% in both the 1-hour and 5-minute windows continuously for two minutes.

<p align="center">
  <img src="./images/what-you-create.png" alt="Deliverables: an SLO document and a YAML rule file with dual-window alert conditions" width="800">
</p>

The diagram also shows a **6x burn-rate rule** as an optional extension. In this lab, you record all three thresholds in the SLO document and implement only the **14.4x rule** in the YAML file.

## Which Requests Count? (SLI and SLO)

<p align="center">
  <img src="./images/which-requests-orginal.png" alt="Which requests count: filter cancellations and health checks to separate good from bad requests" width="800">
</p>

When someone uses the online shop, their browser sends a **request** to the API. The API returns a response with a number called an **HTTP status code**. For example, `200` means the request succeeded, and `500` means the server had a problem.

Before measuring the API, we need two things:

- **SLI (service level indicator):** the percentage of counted requests that had no server error. This tells us how the API actually performed.
- **SLO (service level objective):** the target for that percentage. In this lab, the target is **at least 99.9% over the most recent 30 days**.

**Read the diagram from top to bottom:**

1. **Skip requests we do not measure.** Code `499` means the client disconnected before the server finished responding. Requests to `/healthz` are automatic checks that ask whether the service is running. This lab leaves both out of the calculation.
2. **Keep the remaining requests.** The diagram calls these **valid requests**, meaning requests included in our measurement.
3. **Split them into good and bad.** Codes `2xx`, `3xx`, and `4xx` count as good. Codes `5xx` count as bad. Here, `5xx` means any code from `500` to `599`.
4. **Calculate the good-request percentage:**

```text
SLI (%) = good requests / (good requests + bad requests) x 100
```

For example, if 999 counted requests are good and 1 is bad, the SLI is `999 / 1,000 x 100 = 99.9%`. That meets our 99.9% SLO.

In this lab, **good means no server error**. A `404` response counts as good because the server responded that the requested item was not found. It does not mean the customer successfully placed an order.

## When Do We Alert? (Error Budgets and Burn Rate)

<p align="center">
  <img src="./images/burn-rate-alerting.png" alt="When do we alert: multi-window burn rate thresholds and notification actions" width="800">
</p>

The **error budget** is the number of bad requests we can have while still meeting the SLO. Our target is 99.9% good requests, so we can allow **0.1% bad requests**:

```text
Allowed failure percentage = 100% - 99.9% = 0.1%
```

Each bad request uses part of this allowance. **Burn rate** tells us how quickly we are using it. Calculate it by dividing the measured failure percentage by the allowed failure percentage:

```text
Burn rate = measured failure percentage / 0.1%

Example: 1.44% / 0.1% = 14.4x
```

The `x` means “times.” A **14.4x burn rate** means failures are using the budget 14.4 times as fast as the rate that would use the full allowance over 30 days.

**Read each row of the diagram from left to right:**

| Burn-rate threshold | Check the failure percentage in both periods | What the team should do |
| :--- | :--- | :--- |
| **14.4x — fast** | Above **1.44%** over the last **1 hour** AND the last **5 minutes** | Send an urgent alert (“Page now”). |
| **6x — medium** | Above **0.60%** over the last **6 hours** AND the last **30 minutes** | Send an urgent alert (“Page now”). |
| **1x — slow** | Above **0.10%** over the last **3 days** AND the last **6 hours** | Create a task for the team to investigate (“Create ticket”). |

A **window** is the period we look back over. The long window checks whether errors have been happening for a while. The short window checks whether they are still happening recently. **Both must exceed the row's threshold.** A high error rate in only one window does not trigger that rule.

For example, the fast rule checks for more than 1.44% failures in both windows. In Step 4, you will also require both conditions to stay true for **two minutes** before the alert fires.

The diagram's “about 2 days” and “about 5 days” estimates assume steady request traffic and a failure rate that stays the same. They show how long it would take to use an allowance equal to the full 30-day budget: `30 / 14.4 ≈ 2 days`, `30 / 6 = 5 days`, and `30 / 1 = 30 days`. They are not a countdown for the budget currently remaining.

## Step 1: Define Good and Bad Requests

Start by writing down the rules for counting requests. Use the same rules every time you calculate the SLI or the error budget. You will save them in `slo-document.md` in Step 3.

```text
1. Skip cancelled requests (499) and health checks (/healthz).
2. Count the remaining requests:
   - 200-499: good, because there was no server error.
   - 500-599: bad, because the server failed to handle the request.
```

**Try the rules with these examples:**

| Request or response | How we count it | Why |
| :--- | :--- | :--- |
| `200` — request succeeded | Good | The server handled the request without a server error. |
| `302` — redirect to another address | Good | The server returned a redirect without a server error. |
| `404` — item not found | Good | This lab counts client errors as good responses. |
| `499` — client disconnected | Skip | Our rules exclude cancellations. |
| `500` — server error | Bad | The server failed to handle the request. |
| `504` — gateway timeout | Bad | The gateway did not receive a response in time. |
| Any request to `/healthz` | Skip | Our rules exclude health checks, whatever their status code. |

**Check your understanding:** Suppose there are 100 requests: 5 health checks, 2 cancellations, 90 good requests, and 3 server errors. We skip the first 7, so there are **93 valid requests**. The SLI is `90 / 93 x 100 ≈ 96.77%`.

---

## Step 2: Calculate the Error Budget

Now use the 99.9% target to calculate how many bad requests are allowed. For every 1,000 valid requests, at least 999 must be good, so at most 1 may be bad.

Here, “monthly” means the **most recent 30 days**, rather than a calendar month. This is a **rolling window**: when we check again tomorrow, we include the newest day and drop the oldest day.

Suppose the API received **10,000,000 valid requests** in that window, and **4,000** returned server errors. Health checks and cancellations have already been removed from these counts.

**First, calculate the allowed failures:**

```text
Allowed failure percentage = 100% - 99.9% = 0.1%
0.1% as a decimal          = 0.001
Allowed failures           = 10,000,000 x 0.001 = 10,000
```

**Next, subtract the failures that already happened:**

```text
Allowed failures: 10,000,000 x 0.1 / 100 = 10,000
Actual failures:                          4,000
Failures still allowed: 10,000 - 4,000  =  6,000
```

The API could have up to **10,000 bad requests** for this request total and still meet the target. It has had **4,000**, so **6,000** of the allowance remain.

**Finally, check how much budget was used and whether the API met its target:**

```text
Budget used      = 4,000 / 10,000 x 100 = 40%
Budget remaining = 6,000 / 10,000 x 100 = 60%

Good requests    = 10,000,000 - 4,000 = 9,996,000
Actual SLI       = 9,996,000 / 10,000,000 x 100 = 99.96%
SLO target       = 99.9%
```

**Result:** The API meets the SLO because **99.96% is higher than 99.9%**. It has used **40%** of its error budget.

The 6,000 remaining failures apply to this example's request total and window. Recalculate the budget as the rolling window and number of requests change. This is a **request-based budget**, so it counts failed requests, not minutes of downtime.

### Run the calculation yourself

Create `calculate-budget.py` in your practice folder and paste the following code. It calculates the numbers above:

```python
"""Calculate the request-based error budget used in this lab."""

valid_requests = 10_000_000
bad_requests = 4_000
slo = 0.999
allowed_failures = round(valid_requests * (1 - slo))
remaining = allowed_failures - bad_requests
sli = (valid_requests - bad_requests) / valid_requests * 100

print(f"Valid requests:    {valid_requests:,}")
print(f"SLO target:        {slo * 100:.1f}%")
print(f"Allowed failures:  {allowed_failures:,}")
print(f"Actual failures:   {bad_requests:,}")
print(f"Failures remaining:{remaining:>10,}")
print(f"Budget used:       {bad_requests / allowed_failures * 100:.0f}%")
print(f"Budget remaining:  {remaining / allowed_failures * 100:.0f}%")
print(f"Actual SLI:        {sli:.2f}%")
print(f"SLO met:           {'Yes' if sli >= slo * 100 else 'No'}")
```

Open a terminal in your practice folder and run:

```bash
python3 calculate-budget.py
```

<p align="center">
  <img src="./images/budget-output.png" alt="Captured calculation output: 10,000 allowed failures, 6,000 remaining, 40% budget used, and 99.96% SLI" width="1000">
</p>

**Read the output:** `Allowed failures` is the full budget. `Actual failures` is what we used. `Failures remaining` is what is left. `SLO met: Yes` means the measured good-request percentage is at least the target.

---

## Step 3: Set the Burn-Rate Thresholds

Choose when to notify the team based on how quickly failures use the budget. A page is an urgent notification; a ticket is a task to review later.

```text
High error rate over 1 hour AND the last 5 minutes
Both conditions stay true for 2 minutes           --> urgent alert
High error rate in only one window                --> no fast alert
```

**Explanation:** Both windows must exceed the threshold. The fast-burn rule in Step 4 also requires this combined condition to remain true for two minutes, matching the alert diagram.

Create `slo-document.md` in your practice folder and paste this complete document:

```markdown
# Orders API SLO

## SLI
- Service: orders-api; owner: Orders API team.
- Source: production gateway request counts.
- Valid: completed requests, excluding code 499 and route /healthz.
- Good: valid requests with HTTP 2xx, 3xx, or 4xx.
- Bad: valid requests with HTTP 5xx, including gateway timeout 504.
- SLI = good requests / valid requests.
- Count each retry separately.
- Use checks from outside the system for requests that never reach the gateway.
## SLO
- At least 99.9% of valid requests must be good over the last 30 days.
- This is a rolling window: each day, we check the most recent 30 days.

## Monthly Error Budget
- Allowed failure percentage: 100% - 99.9% = 0.1%.

| Measure | Calculation or value |
| :--- | :--- |
| Valid requests in 30 days | 10,000,000 |
| Allowed failures | 10,000,000 x 0.1 / 100 = 10,000 |
| Actual failures | 4,000 |
| Failures still allowed | 10,000 - 4,000 = 6,000 |
| Budget used | 4,000 / 10,000 x 100 = 40% |
| Budget remaining | 6,000 / 10,000 x 100 = 60% |

- Good requests: 10,000,000 - 4,000 = 9,996,000.
- Actual SLI: 9,996,000 / 10,000,000 x 100 = 99.96%.
- The API meets the SLO because 99.96% is above the 99.9% target.
## Alerts

| Speed | Long + short window | Burn threshold | Error ratio | Action |
| :--- | :--- | :--- | :--- | :--- |
| Fast | 1h + 5m | >14.4x | >1.44% | Page now |
| Medium | 6h + 30m | >6x | >0.60% | Page now |
| Slow | 3d + 6h | >1x | >0.10% | Create ticket |
```

**What this file does:** Records the request rules, availability target, error budget, and alert thresholds.

Fast-alert threshold: `14.4 x 0.1% = 1.44%`. Review tickets the next business day; prioritize reliability fixes when the budget runs out.

---

## Step 4: Write the Fast-Burn Alert Rule

Create `orders-api-rules.yaml` in your practice folder. Prometheus monitors metrics, and its query language, PromQL, calculates error rates from request counts.

<p align="center">
  <img src="./images/fast-alert.png" alt="Fast alert: both 1-hour and 5-minute error rates exceed 1.44% continuously for two minutes" width="800">
</p>

```yaml
groups:
  - name: orders-api-slo
    rules:
      - alert: OrdersAPIErrorBudgetBurnFast
        for: 2m
        expr: |
          (
            1 - sum(rate(http_requests_total{job="orders-api",code!~"5..|499",route!="/healthz"}[1h]))
            / sum(rate(http_requests_total{job="orders-api",code!="499",route!="/healthz"}[1h]))
            > 0.0144
          )
          and
          (
            1 - sum(rate(http_requests_total{job="orders-api",code!~"5..|499",route!="/healthz"}[5m]))
            / sum(rate(http_requests_total{job="orders-api",code!="499",route!="/healthz"}[5m]))
            > 0.0144
          )
        labels:
          severity: page
        annotations:
          summary: "orders-api is burning its error budget fast"
```

**What this file does:** Defines an alert that fires once both error rates have remained above 1.44% for two minutes across rule evaluations.

`rate()` handles counter resets. The 5-minute lookback smooths recent measurements and confirms that errors are still occurring. `for: 2m` adds a two-minute pending period after both thresholds are crossed. If either condition stops being true during that period, the alert does not fire.

A lookback window is not a fixed waiting period: a large spike can cross both thresholds before five minutes have passed. Detection time depends on traffic, the error rate, and scrape and rule evaluation intervals. Notification delivery can add further delay. See the [Prometheus alerting rules documentation](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/).

---

## Step 5: Check the Rule File

This step checks the YAML rule file. The Markdown file is a document you read; the YAML file contains instructions for Prometheus. Use `promtool` to check those instructions without starting a Prometheus server.

Create `slo-document.md` and `orders-api-rules.yaml` in your practice folder using the examples in Steps 3 and 4. Run the following commands from that folder. These download instructions are for Linux with an x86-64 processor.

```bash
# paste this as one line
curl -fLO https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
```

**Explanation:** Downloads the official Prometheus archive containing `promtool`.

<p align="center">
  <img src="./images/download-output.png" alt="Captured curl output showing the Prometheus archive download reaching 100 percent" width="1000">
</p>

**Read the output:** `100` in the progress line means the download finished. The speed and time will differ on your computer.

```bash
tar -xzf prometheus-3.5.0.linux-amd64.tar.gz prometheus-3.5.0.linux-amd64/promtool
ls prometheus-3.5.0.linux-amd64
```

**Explanation:** Extracts only `promtool`, saving space because we do not need the Prometheus server binary. `tar` normally prints nothing on success. The next command, `ls`, lists the extracted folder so you can confirm that `promtool` exists.

<p align="center">
  <img src="./images/extract-output.png" alt="Captured extraction output: tar completes silently and ls lists promtool" width="1000">
</p>

**Read the output:** Seeing `promtool` under the `ls` command means the checker is present.

```bash
./prometheus-3.5.0.linux-amd64/promtool check rules orders-api-rules.yaml
```

**Explanation:** Checks the rule syntax without starting a server.

<p align="center">
  <img src="./images/run-output.png" alt="Check your rule file: promtool check rules outputs SUCCESS" width="800">
</p>

**Read the output:** `Checking orders-api-rules.yaml` identifies the file being checked. `SUCCESS: 1 rules found` means the checker accepted the syntax and found our one fast-burn alert rule.

These images render output captured from real commands, rather than example results. During capture, the local disk was full, so the downloaded checker was run from memory; the rule-check output comes from the official Prometheus 3.5.0 binary. The displayed checker command uses the normal extracted-folder path for students.

Passing this check does not send an alert or prove that live metrics exist. Sending notifications requires a running Prometheus setup and notification configuration.

---

## Conclusion

You documented the Orders API target, calculated its error budget, and checked a fast-burn alert rule. Next, explore load balancing and automated failover.
