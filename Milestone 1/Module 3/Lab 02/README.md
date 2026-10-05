# SLO & Error Budget Definition

<p align="center">
  <img src="./images/budget.png" alt="SLOs and error budgets: set an SLO target, allow an error budget, and watch for fast burn" width="800">
</p>

An application programming interface (API) lets programs request data. Define an Orders API target, calculate its error budget, and write an alert rule.

## Objective

Define service level indicators (SLIs) and service level objectives (SLOs) for a sample API, then derive its monthly error budget. This lab uses the Orders API and a rolling 30-day month.

## Tools/Stack

- Markdown for the SLO document.
- Prometheus query sketches for budget burn-rate alerts.
- `promtool` for the optional syntax check in Step 5.

## Deliverable

An SLO document, `slo-document.md`, with the API's request rules, availability target, monthly error budget, and budget burn-rate thresholds. A supporting file, `orders-api-rules.yaml`, shows the fast-burn threshold as a Prometheus query sketch.

<p align="center">
  <img src="./images/what-you-create.png" alt="Deliverables: an SLO document and a YAML rule file with dual-window alert conditions" width="800">
</p>

The SLO document records all three burn-rate thresholds. In this lab, the YAML file implements the 14.4x fast-burn rule; the diagram also lists the 6x threshold as a possible extension.

## Which Requests Count? (SLI and SLO)

<p align="center">
  <img src="./images/which-requests-orginal.png" alt="Which requests count: filter cancellations and health checks to separate good from bad requests" width="800">
</p>

A service level indicator (SLI) measures performance; a service level objective (SLO) sets its target. Client cancellations (code 499) and health checks (`/healthz`) are filtered out first. From the remaining valid requests, responses with 2xx, 3xx, and 4xx count as good, while 5xx server errors count as bad.

## When Do We Alert? (Error Budgets and Burn Rate)

<p align="center">
  <img src="./images/burn-rate-alerting.png" alt="When do we alert: multi-window burn rate thresholds and notification actions" width="800">
</p>

An error budget is the allowed bad fraction: `1 - SLO`. A 99.9% target over rolling 30 days permits 0.1% failures. Burn rate measures how fast your service spends this budget. At steady traffic, a 1x burn rate uses the budget in 30 days; 14.4x uses it in about two days.

## Step 1: Define Good and Bad Requests

First, decide which requests count when measuring availability. You will save these rules in `slo-document.md` in Step 3.

```text
1. Skip cancelled requests (499) and health checks (/healthz).
2. Count the remaining requests:
   - 200-499: good, because there was no server error.
   - 500-599: bad, because the server failed to handle the request.
```

**Explanation:** This lab measures server availability, so client errors such as 404 count as good requests.


---

## Step 2: Calculate the Error Budget

An error budget is the number of failed requests allowed while still meeting the SLO. A 99.9% target means at least 999 out of every 1,000 valid requests must be good, so 1 may fail.

Example: 10,000,000 valid requests in 30 days:

```text
Allowed failures: 10,000,000 x 0.1 / 100 = 10,000
Actual failures:                          4,000
Failures still allowed: 10,000 - 4,000  =  6,000
```

**Explanation:** The API has used 4,000 of its 10,000 allowed failures, leaving 6,000 for this example.

The API used 40% of its budget. Its 99.96% good-request rate exceeds the 99.9% target.

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

This optional step checks the supporting query sketch. On Linux with an x86-64 processor, download Prometheus and use its `promtool` checker to validate the rule file.

```bash
# paste this as one line
curl -fLO https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
```

**Explanation:** Downloads the official Prometheus archive containing `promtool`.

```bash
tar -xzf prometheus-3.5.0.linux-amd64.tar.gz
```

**Explanation:** Extracts the Prometheus binaries to your practice folder.

```bash
./prometheus-3.5.0.linux-amd64/promtool check rules orders-api-rules.yaml
```

**Explanation:** Checks the rule syntax without starting a server.

<p align="center">
  <img src="./images/run-output.png" alt="Check your rule file: promtool check rules outputs SUCCESS" width="800">
</p>

**What to expect:** The output reports `SUCCESS` and one rule found, confirming valid syntax. The diagram uses `promtool` directly; the command above uses the binary from the extracted folder. Sending notifications also requires a running Prometheus setup and notification configuration.

---

## Conclusion

You documented the Orders API target, calculated its error budget, and checked a fast-burn alert rule. Next, explore load balancing and automated failover.
