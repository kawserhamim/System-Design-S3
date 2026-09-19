# Availability Math

![Availability Workflow: Users connect to Load Balancer, Switch, and Application Servers to keep the service up](images/availability-workflow.png)

In system design, engineers often measure reliability in "nines": 99.9%, 99.99%, or 99.999%. In this lab, you build a Python calculator that converts these numbers into real minutes of downtime. You will see how connecting servers in a line lowers uptime, how adding backup servers improves uptime, and how to identify which server you should duplicate first.

## Objective

By the end of this lab, you can:
- Convert "nines" and SLA percentages into decimal numbers for math calculations.
- Calculate total availability for servers connected in a single line (serial).
- Calculate total availability for redundant backup servers (parallel).
- Model a multi-tier system using a structured CSV file.
- Recommend the best tier to duplicate to meet an availability goal.

## Deliverable

A Python script named `availability.py` and a configuration file named `components.csv`. Together, they calculate overall system uptime, print monthly and yearly downtime in minutes and hours, and recommend which component to duplicate to meet an SLA target.

## What is Availability?

Availability is the percentage of time a system remains operational and reachable to process user requests over a given time window. It measures overall service health under normal traffic as well as unexpected failures.

The standard formula for availability is:

```text
Availability = Total Uptime / (Total Uptime + Total Downtime)
```

For example, if a service runs across a 30-day month (43,200 minutes) and suffers 43.2 minutes of total downtime, its availability is 99.9%. Even small fractional drops in availability can mean hours of service disruption and significant business loss.

## What are Nines?

Engineers and cloud providers express service level agreements (SLAs) using "nines". Each additional nine cuts allowed downtime by a factor of ten.

| Nines | Availability | Downtime per Month | Downtime per Year |
| :--- | :--- | :--- | :--- |
| 2 nines | 99.0% | 432.00 min (7.2 hours) | 5256.00 min (3.65 days) |
| 3 nines | 99.9% | 43.20 min | 525.60 min (8.8 hours) |
| 4 nines | 99.99% | 4.32 min | 52.56 min |
| 5 nines | 99.999% | 0.43 min (26 seconds) | 5.26 min |

Moving from 3 nines to 4 nines reduces yearly downtime from almost 9 hours to under an hour. Reaching 5 nines allows only 5 minutes of total downtime per year, which requires automated health checks and instant failover.

## Serial Architecture

![Serial Architecture: Users connect to Load Balancer, Application Server, and Database in series. If one fails, the service is unavailable](images/serial.png)

In a serial system, requests travel through components one after another in a single line. Every component in the path must work for the request to succeed. If the load balancer fails, the application server never gets the traffic.

Availability multiplies across each component in the line:

```text
A_system = A_1 * A_2 * ... * A_n
```

Because each availability number is less than 1, a serial chain is always less reliable than its single weakest component. Adding more components in a line always reduces total uptime.

## Parallel Architecture

![Parallel Architecture: Users connect to Load Balancer, Switch, and Parallel Application Servers. Service stays up even if one server fails](images/parallel.png)

In a parallel system, backup copies run side by side. The service stays online as long as at least one copy is healthy. 

Total availability is one minus the probability that all copies fail at the exact same time:

```text
A_system = 1 - (1 - A_1) * (1 - A_2) * ... * (1 - A_n)
```

This formula assumes that redundant copies fail independently. If two servers share the same power supply or network switch, a single hardware fault can take down both. Keep failure independence in mind when building real architectures.

## Serial vs Parallel

Comparing serial and parallel designs shows why modern production systems combine both models:

| Architecture | Formula | What Happens | Example |
| :--- | :--- | :--- | :--- |
| Serial (In a line) | `A1 * A2 * ... * An` | Always worse than weakest part | Two 99.9% servers yield 99.80% |
| Parallel (Side by side) | `1 - (1-A1) * (1-A2)` | Backup copies improve uptime | Two 99.9% servers yield 99.9999% |
| Combined System | Serial line of backup groups | Redundancy removes bottlenecks | `LB -> (API \|\| API) -> DB` |

A real production architecture connects distinct tiers in series, but runs multiple redundant copies in parallel inside each critical tier.

---

## Step 1: Create the project

Start by creating a clean directory and empty files for your calculator.

```text
m3-l1/
├── availability.py
└── components.csv
```

Run these commands in your terminal:

```bash
mkdir -p m3-l1 && cd m3-l1
touch availability.py components.csv
```

**Explanation:** Creates a new folder named `m3-l1` and initializes empty files for the Python script and CSV data.

---

## Step 2: Convert nines to availability

Humans talk in percentages like "99.9%" or counts like "3 nines", but math formulas need decimal probabilities like `0.999`.

```text
User Input ("3 nines", "99.95%") ---> Converter Helpers ---> Math Decimals (0.999, 0.9995)
```

Open `availability.py` and write the conversion functions:

```python
import argparse
import csv
import math

MIN_PER_MONTH = 30 * 24 * 60      # 43200 minutes in a 30-day month
MIN_PER_YEAR = 365 * 24 * 60      # 525600 minutes in a year


def nines_to_prob(nines: float) -> float:
    """Convert nines count to a decimal: 3 -> 0.999, 4 -> 0.9999."""
    if nines < 1 or nines > 9:
        raise ValueError("nines must be between 1 and 9")
    return 1.0 - 10 ** -nines


def percent_to_prob(percent: float) -> float:
    """Convert SLA percentage to a decimal: 99.95 -> 0.9995."""
    if percent <= 0 or percent > 100:
        raise ValueError("percent must be between 0 and 100")
    return percent / 100.0


def prob_to_nines(p: float) -> float:
    """Convert a decimal probability back to nines: 0.999 -> 3.0."""
    if p <= 0 or p >= 1:
        return float("inf")
    return -math.log10(1.0 - p)
```

**What this file does:** Converts human numbers and cloud SLA percentages into clean decimal values for calculations.

---

## Step 3: Add serial calculation

Next, implement the serial calculation rule where probabilities multiply across every component in the line.

```text
[ Server 1 ] ---> [ Server 2 ] ---> [ Server 3 ]  =>  A1 * A2 * A3
```

Append the `serial` function to `availability.py`:

```python
def serial(*probs: float) -> float:
    """Every component must work. Probabilities multiply."""
    out = 1.0
    for p in probs:
        out *= p
    return out
```

**What this file does:** Multiplies probabilities together to calculate the combined availability of components in series.

---

## Step 4: Add parallel calculation

Now implement the parallel calculation rule where redundant backup copies run side by side.

```text
[ Server 1 ] || [ Server 2 ]  =>  1 - (1 - A1) * (1 - A2)
```

Append the `parallel` function to `availability.py`:

```python
def parallel(*probs: float) -> float:
    """At least one copy must work. Assumes copies fail independently."""
    out = 1.0
    for p in probs:
        out *= (1.0 - p)
    return 1.0 - out
```

**What this file does:** Computes the combined availability of parallel redundant servers assuming independent failures.

---

## Step 5: Add calculator input

Now add the calculator input mechanisms. You will define a CSV file to describe your system tiers and add command-line input handling in Python.

```text
CLI Flags / CSV Input ---> load_groups() ---> Tier Evaluation ---> System Availability Report
```

Open `components.csv` and add your system definition:

```csv
component,nines,group
LB-1,3,lb
API-server-1,3,api
API-server-2,3,api
DB-primary,4,db
Cache,3,cache
```

**What this file does:** Defines the infrastructure components, their individual uptime nines, and their redundancy groups.

Next, append the reporting helpers, CSV parser, suggestion engine, and CLI entry point to `availability.py`:

```python
def fmt_downtime(minutes: float, per: str) -> str:
    """Show downtime in minutes, and add hours in parentheses if over 60."""
    if minutes >= 60:
        return f"{minutes:.2f} min/{per}  ({minutes / 60:.1f} hours)"
    return f"{minutes:.2f} min/{per}"


def report(label: str, p: float) -> None:
    """Print clean report with percentage, nines, and downtime."""
    nines = prob_to_nines(p)
    nines_txt = "9+ (beyond precision)" if math.isinf(nines) else f"{nines:.3f}"
    print(label)
    print(f"  availability: {p * 100:.6f}%")
    print(f"  nines:        {nines_txt}")
    print(f"  downtime:     {fmt_downtime(MIN_PER_MONTH * (1 - p), 'month')}")
    print(f"                {fmt_downtime(MIN_PER_YEAR * (1 - p), 'year')}")
    print()


def demo() -> None:
    """Run four clear examples to verify the math."""
    a = nines_to_prob(3)
    report("single 99.9%", a)
    report("serial   99.9% + 99.9%", serial(a, a))
    report("parallel 99.9% || 99.9%", parallel(a, a))

    # A real mini system: Load Balancer -> Two API Servers -> Database
    lb = nines_to_prob(3)
    api = parallel(nines_to_prob(3), nines_to_prob(3))
    db = nines_to_prob(4)
    report("combined LB -> (API || API) -> DB", serial(lb, api, db))


def load_groups(path: str) -> dict:
    """Read components.csv into a dictionary of groups."""
    groups: dict = {}
    with open(path, newline="") as fh:
        for row in csv.DictReader(fh):
            name = row["component"].strip()
            group = row["group"].strip()
            if not name or not group:
                continue
            nines = float(row["nines"])
            groups.setdefault(group, []).append(nines_to_prob(nines))
    if not groups:
        raise ValueError(f"no usable rows in {path}")
    return groups


def system_availability(groups: dict) -> float:
    """Combine parallel copies in each group, then chain all groups in series."""
    tier_probs = [parallel(*copies) for copies in groups.values()]
    return serial(*tier_probs)


def report_groups(groups: dict) -> float:
    """Print the uptime for each tier and the total system uptime."""
    print("Tier breakdown")
    for name, copies in groups.items():
        tier = parallel(*copies)
        shape = f"{len(copies)} copies in parallel" if len(copies) > 1 else "single"
        print(f"  {name:<12} {tier * 100:11.6f}%   ({shape})")
    print()
    total = system_availability(groups)
    report("SYSTEM", total)
    return total


def suggest_for_target(groups: dict, target_percent: float,
                       max_adds: int = 6) -> None:
    """Find the best component to duplicate step by step."""
    target = percent_to_prob(target_percent)
    current = system_availability(groups)
    print(f"Target:  {target_percent}%")
    print(f"Current: {current * 100:.6f}%")

    if current >= target:
        print("Already meeting the target. No change needed.\n")
        return

    print("\nEffect of adding one more copy to each tier:")
    for name, copies in groups.items():
        trial = {k: list(v) for k, v in groups.items()}
        trial[name].append(max(copies))
        new = system_availability(trial)
        gain = prob_to_nines(new) - prob_to_nines(current)
        print(f"  +1 {name:<12} -> {new * 100:11.6f}%   (+{gain:.3f} nines)")

    plan, work = [], {k: list(v) for k, v in groups.items()}
    for _ in range(max_adds):
        best_name, best_val = None, system_availability(work)
        for name, copies in work.items():
            trial = {k: list(v) for k, v in work.items()}
            trial[name].append(max(copies))
            val = system_availability(trial)
            if val > best_val:
                best_name, best_val = name, val
        if best_name is None:
            break
        work[best_name].append(max(work[best_name]))
        plan.append(best_name)
        if best_val >= target:
            break

    final = system_availability(work)
    print("\nCheapest plan found:")
    if final >= target:
        for i, name in enumerate(plan, 1):
            print(f"  {i}. add one copy of '{name}'")
        print(f"  -> {final * 100:.6f}%  TARGET MET")
    else:
        print(f"  {max_adds} extra copies only reach {final * 100:.6f}%.")
        print("  Weakest tier needs a better machine, not just more duplicates.")
    print()


def main() -> None:
    ap = argparse.ArgumentParser(description="System availability calculator")
    ap.add_argument("--csv", help="path to components.csv")
    ap.add_argument("--nines", nargs="+", type=float,
                    help="component nines, e.g. --nines 3 3 4")
    ap.add_argument("--topology", choices=["serial", "parallel"], default="serial")
    ap.add_argument("--target", type=float,
                    help="target availability percent, e.g. 99.95")
    ap.add_argument("--demo", action="store_true", help="run the worked examples")
    args = ap.parse_args()

    if args.csv:
        groups = load_groups(args.csv)
        report_groups(groups)
        if args.target:
            suggest_for_target(groups, args.target)
    elif args.nines:
        probs = [nines_to_prob(n) for n in args.nines]
        fn = serial if args.topology == "serial" else parallel
        report(f"{len(probs)} components in {args.topology}", fn(*probs))
    else:
        demo()


if __name__ == "__main__":
    main()
```

**What this file does:** Reads CSV tier inputs, handles CLI arguments, formats downtime reports, and calculates the optimal redundancy plan.

---

## Step 6: Run the calculator

Now run your calculator across all primary scenarios to verify the math and evaluate your infrastructure.

```bash
# 1. Run the sample demo comparisons
python3 availability.py --demo

# 2. Test a serial line of components
python3 availability.py --nines 3 3 4 --topology serial

# 3. Test parallel backup components
python3 availability.py --nines 3 3 --topology parallel

# 4. Check system from CSV and get recommendations for 99.95% target
python3 availability.py --csv components.csv --target 99.95
```

![Availability Calculation Results: terminal output showing availability percentage, nines, and downtime for all topologies](images/availability-calculation-result.png)

**What to expect:** The demo prints availability percentages, nines, and downtime for single, serial, parallel, and combined topologies. For `components.csv`, it shows an overall availability of 99.790020% (about 18.4 hours of downtime per year) and suggests adding one backup copy of `lb` and one of `cache` to reach the 99.95% target.

---

## Verification checklist

- [ ] `python3 availability.py --demo` runs and prints sample comparisons.
- [ ] Serial availability (99.8001%) computes lower than a single 3-nines component (99.9%).
- [ ] Parallel availability (99.9999%) computes higher than a single 3-nines component (99.9%).
- [ ] `python3 availability.py --csv components.csv` reads all groups and outputs total system availability.
- [ ] Adding `--target 99.95` recommends duplicating `lb` and `cache` to meet the SLA.

## Conclusion

You built a working calculator that turns system design percentages into real minutes of downtime. You learned why serial chains lower uptime, how parallel redundancy prevents outages, and how to identify the weakest link in an architecture. In the next lab, you will use these formulas to design systems that handle automated failover and zero-downtime deployments.