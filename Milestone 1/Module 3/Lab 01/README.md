# Availability Math

![Availability lab: design a topology, calculate availability, and compare uptime and downtime](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2001/images/availability.png)

Build a calculator that shows how component reliability affects a whole system. Compare required components in a serial chain with backup copies in a parallel group, then estimate downtime.

## Objective

```text
Component nines + topology ---> System availability
```

**Explanation:** Topology means how components connect.

By the end of this lab, you can:

- Compute availability for serial and parallel component chains.
- Combine parallel groups into a serial system.
- Read availability and downtime from calculator output.

## Tools/Stack

```text
Spreadsheet ---> components.csv ---> Python 3 ---> Report
```

**Explanation:** A spreadsheet helps edit inputs, and Python calculates the result.

Use Python 3 and a terminal. Node.js is an alternative implementation option; this lab provides Python code. Edit comma-separated values (CSV) in a spreadsheet.

## Deliverable

```text
availability.py + component nines + topology ---> Availability report
```

**Explanation:** One calculator handles direct inputs and a CSV system definition.

A working `availability.py` calculator that outputs system availability given per-component nines and topology.

## Availability and Nines

```text
Availability = uptime / (uptime + downtime)
3 nines = 99.9% = 0.999
```

**Explanation:** Availability is the share of time a service works; nines describe its percentage.

Each extra nine reduces expected downtime tenfold. The calculator uses a 30-day month and a 365-day year.

| Nines | Availability | Downtime/month | Downtime/year |
| :--- | :--- | :--- | :--- |
| 2 | 99% | 432 min | 5,256 min |
| 3 | 99.9% | 43.2 min | 525.6 min |
| 4 | 99.99% | 4.32 min | 52.56 min |
| 5 | 99.999% | 0.432 min | 5.256 min |

## Serial Architecture

![Serial dependencies: the app is down and cannot connect to the database](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2001/images/serial.png)

Every required component must work. The diagram shows an app failure breaking the request path to the database. For independent failures, multiply component availability: `A_system = A_1 * A_2`. Adding a required component below 100% availability lowers the total.

## Parallel Architecture

![Parallel app replicas: App 1 is up, App 2 is down, and both share a load balancer](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2001/images/parallel.png)

A replica is another copy of a component. App 1 can serve requests when App 2 fails, assuming successful routing and enough capacity. For independent failures, `A_group = 1 - (1 - A_1) * (1 - A_2)`. This covers the app group only; the shared load balancer remains a required dependency.

## Compare the Designs

![Comparison of single, serial, and parallel components with 99.9% availability each](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2001/images/availability-comparision.png)

The parallel result excludes shared dependencies and assumes one copy can handle all traffic.

| Design | Availability | Expected downtime/year |
| :--- | :--- | :--- |
| Single component | 99.9% | 8.76 hours |
| Two in series | 99.8001% | 17.51 hours |
| Two in parallel | 99.9999% | 31.54 seconds |

## Combined Architecture

![Load balancer, two parallel apps, and database connected as required serial tiers](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2001/images/combine.png)

A tier is a group that performs one role. Calculate the parallel app tier first, then multiply by load balancer and database availability: `A_system = A_lb * A_apps * A_db`. The shared load balancer and database can each stop the service. These estimates assume independent failures; a shared power or network failure can take down multiple copies together.

## Step 1: Create the project

```text
m3-l1/
|-- availability.py
`-- components.csv
```

**Explanation:** Keep the calculator and its input file together.

Create the project files.

```bash
mkdir -p m3-l1 && cd m3-l1
touch availability.py components.csv
```

**Explanation:** Creates the project and moves into its directory.

---

## Step 2: Write the calculator

```text
Nines ---> Probabilities ---> Serial or parallel calculation ---> Report
```

**Explanation:** Convert nines into decimals before applying the formulas.

Paste this complete file into `availability.py`. CSV groups combine both topologies.

```python
import argparse
import csv
import math


def probability(nines):
    if not math.isfinite(nines) or not 1 <= nines <= 9:
        raise ValueError("nines must be between 1 and 9")
    return 1 - 10 ** -nines


def serial(values):
    return math.prod(values)


def parallel(values):
    return 1 - math.prod(1 - value for value in values)


def report(label, value):
    downtime = 1 - value
    nines = f"{-math.log10(downtime):.3f}" if downtime else "beyond precision"
    print(label)
    print(f"  availability: {value * 100:.6f}%")
    print(f"  nines:        {nines}")
    print(f"  downtime:     {downtime * 43200:.2f} min/month")
    print(f"                {downtime * 525600:.2f} min/year")
    print()


def read_system(path):
    groups = {}
    with open(path, newline="") as source:
        for row in csv.DictReader(source):
            group = row["group"].strip()
            if not group or not row["component"].strip():
                raise ValueError("component and group must not be empty")
            groups.setdefault(group, []).append(probability(float(row["nines"])))
    if not groups:
        raise ValueError("CSV must contain components")
    return serial(parallel(copies) for copies in groups.values())


def main():
    parser = argparse.ArgumentParser(description="Calculate system availability")
    inputs = parser.add_mutually_exclusive_group(required=True)
    inputs.add_argument("--nines", nargs="+", type=float)
    inputs.add_argument("--csv")
    inputs.add_argument("--demo", action="store_true")
    parser.add_argument("--topology", choices=["serial", "parallel"], default="serial")
    args = parser.parse_args()
    try:
        if args.demo:
            a = probability(3)
            report("single 99.9%", a)
            report("serial 99.9% + 99.9%", serial([a, a]))
            report("parallel 99.9% || 99.9%", parallel([a, a]))
            report("combined LB -> (API || API) -> DB",
                   serial([a, parallel([a, a]), probability(4)]))
        elif args.csv:
            report("SYSTEM", read_system(args.csv))
        else:
            values = [probability(n) for n in args.nines]
            calculate = serial if args.topology == "serial" else parallel
            report(args.topology, calculate(values))
    except (ValueError, KeyError, OSError) as error:
        parser.error(str(error))


if __name__ == "__main__":
    main()
```

**What this file does:** Calculates availability from nines or CSV groups and prints expected downtime.

---

## Step 3: Write the component inputs

```text
LB ---> (API-server-1 OR API-server-2) ---> Database ---> Cache
```

**Explanation:** The CSV extends the combined diagram with a required cache tier.

Paste this into `components.csv`. Matching group names identify parallel copies; different groups are required serial dependencies. Keep the header when exporting CSV.

```csv
component,nines,group
LB-1,3,lb
API-server-1,3,api
API-server-2,3,api
DB-primary,4,db
Cache,3,cache
```

**What this file does:** Defines component nines and their redundancy groups.

---

## 4. Run the following commands in the terminal

```text
Demo ---> Serial chain ---> Parallel copies ---> CSV system
```

**Explanation:** Run each input type from the project directory.

```bash
python3 availability.py --demo
python3 availability.py --nines 3 3 4 --topology serial
python3 availability.py --nines 3 3 --topology parallel
python3 availability.py --csv components.csv
```

**Explanation:** Compares the built-in examples, direct inputs, and CSV system.

![Demo results showing availability, equivalent nines, and downtime for four designs](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%203/Lab%2001/images/availability-calculation-result.png)

**What to expect:** The image shows the demo's values; this script prints minutes without the extra hour labels. The combined demo gives 99.889910%. The CSV adds a required cache, lowering availability to 99.790020%, about 18.4 hours of downtime per year.

---

## Conclusion

```text
Availability estimates ---> Service targets ---> Error budgets
```

**Explanation:** Use the results to set measurable reliability targets.

You can now turn component nines and topology into system availability and expected downtime. Next, use service level objectives, or reliability targets, to calculate how much failure a service can tolerate.
