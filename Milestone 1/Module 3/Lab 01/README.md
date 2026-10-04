# Availability Math

Imagine your URL shortener has a load balancer, an app server, and a database. Each component works **99.9% of the time**. Does that mean the whole system also works 99.9% of the time?

No. A request needs all three components. If any required component stops working, the user cannot complete the request. Adding a second app server can help—but the shared database can still stop the service.

In this lab, we will build a small Python calculator to see how these choices change **availability** and **expected downtime**.

![Lab overview: choose serial dependencies or parallel copies, calculate availability, then compare uptime and downtime](./images/availability.png)

**How to read the picture:**

- **Design:** decide which components are required and which ones are backup copies.
- **Calculate:** give the calculator each component's availability and how they connect.
- **Compare:** read the percentage of time the system works and the estimated time it is unavailable.

In the lower design, **A is still required**. Only the two B copies form a parallel group. If A fails, neither B copy can help the user.

## Objective

By the end of this lab, you will be able to:

- Explain availability and “nines” in plain words.
- Calculate availability when **all components must work** (serial).
- Calculate availability when **at least one copy must work** (parallel).
- Combine both designs and convert the result into expected downtime.

## Tools and Deliverable

You need **Python 3.8 or later**, a terminal, and a text editor. No extra Python packages are required.

You will create two files:

| File | What it does |
| --- | --- |
| `availability.py` | Calculates availability and prints a downtime report. |
| `components.csv` | Lists each component, its nines, and its group. |

A CSV is a plain-text table with commas between values. You can edit it in your text editor or a spreadsheet.

## 1. What Is Availability?

**Availability** means: what fraction of the time can the service do its job?

If a service works for 99 hours and is unavailable for 1 hour:

```text
Availability = uptime / (uptime + downtime)
             = 99 / (99 + 1)
             = 0.99
             = 99%
```

We use decimals in the formulas: **99.9% becomes 0.999**, not 99.9.

### What do “nines” mean?

“Three nines” is a short way to say **99.9% availability**. Count the 9s in the percentage.

| Nines | Availability | Expected downtime in a 30-day month | Expected downtime in a 365-day year |
| --- | --- | --- | --- |
| 2 | 99% | 432 minutes | 5,256 minutes |
| 3 | 99.9% | 43.2 minutes | 525.6 minutes |
| 4 | 99.99% | 4.32 minutes | 52.56 minutes |
| 5 | 99.999% | 0.432 minutes | 5.256 minutes |

Each extra nine makes the expected downtime **10 times smaller**.

```text
Downtime = (1 - availability) × time in the period

For three nines:
Yearly downtime = (1 - 0.999) × 525,600 minutes
                = 525.6 minutes
                = 8.76 hours
```

These are estimates over time. They do not tell us when an outage will happen or how long a particular outage will last.

## 2. Serial Architecture: Every Required Component Must Work

Think of the URL shortener's request path:

**User → Load balancer → App → Database**

The load balancer sends the request to the app. The app looks up the short code in the database. The user gets a redirect only if all required parts work.

![Serial request path: users reach the load balancer, but the app is down, so the request cannot reach the database](./images/serial.png)

**What is happening in the picture?**

1. The user's request reaches the load balancer.
2. The app is **down**.
3. The request cannot continue to the database, so the user does not get a successful result.

“Cannot connect” shows the broken request path. It does not necessarily mean the database itself has crashed.

For components that fail independently, multiply their availability:

```text
System availability = load balancer × app × database
```

Start with a smaller example: two required components, each with three nines.

```text
A = 0.999
B = 0.999

Serial availability = A × B
                    = 0.999 × 0.999
                    = 0.998001
                    = 99.8001%
```

**Why is the result lower?** Either A or B can stop the service. Adding a required dependency creates another way for the request to fail.

## 3. Parallel Architecture: One Working Copy Is Enough

A **replica** is another copy of a component that can do the same job. Here, two app servers can handle the same request.

![Parallel app group: the load balancer routes to App 1, which is up, while App 2 is down](./images/parallel.png)

**What is happening in the picture?**

1. The user sends a request to the load balancer.
2. App 2 is down, so the load balancer routes the request to App 1.
3. App 1 is up and handles the request. The app group can still work.

The app group fails only when **both copies fail**. That makes the calculation easier:

```text
Each app availability = 0.999
Each app failure probability = 1 - 0.999 = 0.001

Probability both apps fail = 0.001 × 0.001 = 0.000001

App group availability = 1 - probability both fail
                       = 1 - 0.000001
                       = 0.999999
                       = 99.9999%
```

For two independent copies:

```text
Parallel availability = 1 - (1 - A) × (1 - B)
```

This **99.9999% belongs to the app group only**. The load balancer is a shared dependency: if it fails, users cannot reach either app.

The model assumes each surviving app can handle the traffic and requests are successfully routed to it. Copies must fail independently; two apps on one failed machine would both go down together.

## 4. Compare the Two Designs

![Comparison: one 99.9% component has 8.76 hours of yearly downtime; two in series have 17.51 hours; two independent parallel copies have 31.54 seconds](./images/availability-calculation-result.png)

**Read the image from left to right:**

| Design | What must work? | Availability | Expected downtime/year |
| --- | --- | --- | --- |
| Single | One component | 99.9% | 8.76 hours |
| Serial | Both A and B | 99.8001% | 17.51 hours |
| Parallel | At least one of A or B | 99.9999% | 31.54 seconds |

All individual components have the same 99.9% availability. **How they connect changes the result.** The parallel column counts the two copies only; it does not include a shared load balancer or database.

## 5. Combined Architecture: Calculate the App Group First

Most systems combine required dependencies and parallel copies.

![Combined architecture: a required load balancer connects to two parallel apps, which share one required database](./images/combine.png)

**How a request travels:**

1. The load balancer receives the request.
2. It picks one working app; the request does not need both apps.
3. That app reads from the shared database.
4. The app returns the result to the user through the load balancer.

A **tier** is a component or group of components doing one job. This diagram has three required tiers: load balancer, app group, and database.

Use three nines for the load balancer and each app, and four nines for the database:

```text
Step A: Calculate the parallel app group.
Apps = 1 - (1 - 0.999) × (1 - 0.999)
     = 0.999999

Step B: Multiply the three required tiers.
System = load balancer × apps × database
       = 0.999 × 0.999999 × 0.9999
       = 0.9988991010999
       ≈ 99.889910%
```

The app group has six nines, but the **whole system has less than three nines**. The required load balancer and database still affect every request.

Now we will build a calculator that does these steps for us.

## Step 1: Create the Project

Check Python in your terminal:

```bash
python3 --version
```

**Expected:** Python 3.8 or newer. Then create the project folder and files:

```bash
mkdir -p m3-l1 && cd m3-l1
touch availability.py components.csv
```

Your folder should look like this:

```text
m3-l1/
├── availability.py
└── components.csv
```

Run all remaining commands from inside `m3-l1/`.

## Step 2: Write the Calculator

Open `availability.py` in your editor and paste the complete code below.

The calculator follows four steps: **read the inputs → convert nines to decimals → calculate availability → print downtime**.

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

**What this file does:**

- `probability(3)` converts three nines into `0.999`.
- `serial()` multiplies the availability of required components.
- `parallel()` calculates the chance that at least one copy works.
- `report()` converts availability into a percentage, equivalent nines, and downtime.
- `read_system()` combines copies in each CSV group, then multiplies the required groups.

You can run built-in examples with `--demo`, provide nines with `--nines`, or read a system from a file with `--csv`.

## Step 3: Write the Component Inputs

Open `components.csv` and paste:

```csv
component,nines,group
LB-1,3,lb
API-server-1,3,api
API-server-2,3,api
DB-primary,4,db
Cache,3,cache
```

**How to read each column:**

| Column | Meaning | Example |
| --- | --- | --- |
| `component` | A name for the component | `API-server-1` |
| `nines` | Its availability expressed as nines | `3` means 99.9% |
| `group` | The role it belongs to | Both app copies use `api` |

**Same group means alternatives:** `API-server-1` and `API-server-2` are in the `api` group. One working copy is enough.

**Different groups mean required dependencies:** `lb`, `api`, `db`, and `cache` must all work. A group with one component uses that component's availability.

This CSV extends the combined diagram by adding a **required cache**:

```text
Load balancer → (API-server-1 OR API-server-2) → Database → Cache
```

For this exercise, a failed cache makes the request fail. If your real app can fall back to the database when the cache is down, the cache is not a required dependency and this model would need to change.

Keep the header `component,nines,group`. When using a spreadsheet, save/export as CSV.

## Step 4: Run the Built-in Examples

```bash
python3 availability.py --demo
```

**Expected output:**

```text
single 99.9%
  availability: 99.900000%
  nines:        3.000
  downtime:     43.20 min/month
                525.60 min/year

serial 99.9% + 99.9%
  availability: 99.800100%
  nines:        2.699
  downtime:     86.36 min/month
                1050.67 min/year

parallel 99.9% || 99.9%
  availability: 99.999900%
  nines:        6.000
  downtime:     0.04 min/month
                0.53 min/year

combined LB -> (API || API) -> DB
  availability: 99.889910%
  nines:        2.958
  downtime:     47.56 min/month
                578.63 min/year
```

**What to notice:**

- The first three results match the comparison image. The program prints downtime in **minutes**, while the image uses hours or seconds.
- `||` means parallel alternatives. `->` shows required tiers in sequence.
- `nines: 2.958` means the combined result is slightly below three nines; it does not mean 99.958%.
- The last result matches the combined diagram **without the cache**.

## Step 5: Try Your Own Serial and Parallel Inputs

### 5.1 Three required components

```bash
python3 availability.py --nines 3 3 4 --topology serial
```

`3 3 4` gives the three components' nines. `--topology serial` tells the calculator that all three are required.

**Expected output:**

```text
serial
  availability: 99.790120%
  nines:        2.678
  downtime:     90.67 min/month
                1103.13 min/year
```

This is one load balancer, one app, and one database. Compare its availability with the combined demo: the demo's second app copy improves the result.

### 5.2 Two interchangeable app copies

```bash
python3 availability.py --nines 3 3 --topology parallel
```

**Expected:** availability is `99.999900%`, with about `0.53 min/year` of downtime for the app group.

**Try this:** change `3 3` to `3 3 3` and run again. You are adding a third independent app copy. Does the group's expected downtime increase or decrease? The very small downtime may round to `0.00` in this report; that does not mean outages are impossible.

## Step 6: Calculate the CSV System

```bash
python3 availability.py --csv components.csv
```

**Expected output:**

```text
SYSTEM
  availability: 99.790020%
  nines:        2.678
  downtime:     90.71 min/month
                1103.65 min/year
```

**What happened?** The calculator combined the two `api` copies first, then multiplied the four required groups:

```text
System = LB × parallel API group × DB × Cache
       = 0.999 × 0.999999 × 0.9999 × 0.999
       ≈ 99.790020%
```

That is about **18.4 hours of expected downtime per year**. It is worse than the combined demo because the CSV adds one more required dependency: the cache.

### Experiment: Improve the cache

1. Change the cache row from `Cache,3,cache` to `Cache,4,cache`.
2. Save `components.csv`.
3. Run the same CSV command again.

**Expected:** availability rises to about `99.879921%`. The design has the same groups, but a more available required cache improves the total.

Restore `Cache,3,cache` if you want your file to match the original example.

## Step 7: Verification Checklist

Before finishing, check that you can explain the results:

- [ ] Both files exist and the commands run without errors.
- [ ] I can convert three nines into `99.9%` and `0.999`.
- [ ] I can explain why serial components reduce total availability.
- [ ] I can explain why a parallel group works when one copy is down.
- [ ] I include the shared load balancer and database when calculating the whole system.
- [ ] I understand that matching CSV group names mean interchangeable copies.
- [ ] I can explain why the CSV result differs from the combined demo.

**If something goes wrong:**

| Message or problem | What to do |
| --- | --- |
| `python3: command not found` | Install Python 3.8 or newer; if your system uses `python`, check its version and use that command instead. |
| `can't open file 'availability.py'` | Change into `m3-l1/` and confirm you saved the file there. |
| Missing CSV file or a column-name error | Confirm the file is named `components.csv` and its first line is exactly `component,nines,group`. |
| `nines must be between 1 and 9` | Enter a nines value such as `3`, rather than the percentage `99.9`. |

## Conclusion

You built a calculator that turns component availability and system design into an expected downtime report. Required components must all work; parallel copies need at least one working member. Calculate each parallel group first, then include every required shared dependency.

Use these estimates to compare designs. Real results also depend on shared failures, routing, recovery time, and available capacity. In the next lab, we will use reliability targets to calculate how much downtime a service can tolerate.
