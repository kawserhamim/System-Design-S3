# Availability Math

<p align="center">
  <a href="./images/availability.png">
    <img src="./images/availability.png" alt="Choose a design, calculate availability, and compare downtime" width="800">
  </a>
</p>

Availability measures how much of the time a service is working and accessible to users. In this lab, you will build a Node.js calculator to explore how a URL shortener’s load balancer, app servers, and database affect its overall availability. You will compare designs with required components and backup copies, then translate the results into expected monthly and yearly downtime. Through these comparisons, you will learn which design changes help keep a service available and why adding more servers alone may not solve the problem—all without deploying real servers.

## Objective

By the end, you can:

- Explain availability and nines.
- Calculate required parts and alternative copies.
- Estimate downtime and improve a shared part.

## Deliverable

A calculator in `availability.js` and a design in `components.csv`. A **CSV** is a text table with comma-separated columns.

## Understand Availability and Nines

**Availability** is working time divided by total time. **Nines** counts the 9s in the percentage. Three nines means 99.9%, or `0.999` in a formula: 999 working minutes out of 1,000.

| Nines | Availability | Downtime per 30-day month | Downtime per 365-day year |
| --- | --- | --- | --- |
| 2 | 99% | 432 minutes | 5,256 minutes |
| 3 | 99.9% | 43.2 minutes | 525.6 minutes |
| 4 | 99.99% | 4.32 minutes | 52.56 minutes |
| 5 | 99.999% | 0.432 minutes | 5.256 minutes |

Calculate yearly downtime with `(1 - availability) * 525600`. At three nines, this gives **525.6 minutes**. Each extra nine reduces downtime to one tenth. These totals do not predict when outages happen. A 30-day month has 43,200 minutes, so three nines gives `43,200 * 0.001 = 43.2` minutes of downtime.

## Serial: Every Part Must Work

<p align="center">
  <a href="./images/serial.png">
    <img src="./images/serial.png" alt="A failed app stops the request from reaching the database" width="800">
  </a>
</p>

The request needs the load balancer, app, and database. In the picture, the app is down, so a working database cannot finish the request alone. **Independent failures** means one part failing does not change another part's failure chance.

For independent required parts, multiply their availabilities. Two three-nine parts give `0.999 * 0.999 = 0.998001`, or **99.8001%**. Adding required parts adds ways for the request to fail. Both must work at the same time, so the combined result is lower than either part alone.

## Parallel: One Copy Is Enough

<p align="center">
  <a href="./images/parallel.png">
    <img src="./images/parallel.png" alt="One app handles requests while the other is down" width="800">
  </a>
</p>

A **replica** is another copy that does the same job. Two alternative apps work as a group if either works. Each fails with probability `0.001`, so both independent apps fail together with probability `0.001 * 0.001`.

Subtract that from one: `1 - 0.001 * 0.001 = 0.999999`, or **99.9999%**. This covers the app group only. The surviving app must handle the traffic alone. If App 2 stops, App 1 can still serve requests. The group fails only when both stop; that is why we subtract the chance of both failing from one.

## Compare the Designs

<p align="center">
  <a href="./images/availability-calculation-result.png">
    <img src="./images/availability-calculation-result.png" alt="Yearly downtime for single, serial, and parallel designs" width="800">
  </a>
</p>

All parts in this picture have 99.9% availability. The serial design needs both parts; the parallel group needs only one. Their connections change the expected downtime:

| Design | Availability | Yearly downtime |
| --- | --- | --- |
| Single part | 99.9% | 8.76 hours |
| Two required parts | 99.8001% | 17.51 hours |
| Two alternative copies | 99.9999% | 31.54 seconds |

## Combined: Calculate the Group First

<p align="center">
  <a href="./images/combine.png">
    <img src="./images/combine.png" alt="Required load balancer, alternative apps, and required database" width="800">
  </a>
</p>

A **tier** is a part or group doing one role. Read the picture as `LB -> (App 1 || App 2) -> DB`; `||` means alternatives. Calculate the app group first, then multiply the required tiers. Treat the two apps as one group with availability `0.999999`. The request still needs the load balancer and database.

With three nines for the load balancer and apps, and four for the database, `0.999 * 0.999999 * 0.9999` gives **99.889910%**. Both apps need the same load balancer and database, called **shared dependencies**. Either shared part can stop both apps. Two apps on one failed machine also fail together.

## Step 1: Create the Project

Use an Ubuntu/Debian terminal. Install Node.js, the runtime that executes JavaScript outside a browser:

```bash
sudo apt update
sudo apt install -y nodejs
node --version
```

**Explanation:** Installs Node.js and prints its version. This calculator uses built-in Node.js modules, so no framework or npm packages are required.

**What to expect:** Installation prints progress or says Node.js is already installed. `node --version` prints a version starting with `v`.

Create the folder and empty files:

```bash
mkdir -p m3-l1 && cd m3-l1
touch availability.js components.csv
ls
```

**Explanation:** Creates the project folder and lists its files.

**What to expect:** The first two commands normally print nothing. `ls` lists `availability.js` and `components.csv`; open this folder in your editor.

**In your editor:** Open `m3-l1`. The Explorer should show `availability.js` and `components.csv`. Both files are empty until you add their contents in the next steps.

---

## Step 2: Write the Calculator

```text
Nines -> availability -> serial or parallel formula -> downtime
```

**Explanation:** The calculator converts inputs, combines parts, and prints results.

Paste this complete code into `availability.js`. It uses JavaScript and Node.js’s built-in file system module:

```javascript
const fs = require("node:fs");

function probability(nines) {
  if (!Number.isFinite(nines) || nines < 1 || nines > 9) {
    throw new Error("nines must be between 1 and 9");
  }
  return 1 - 10 ** -nines;
}

function serial(values) {
  return values.reduce((product, value) => product * value, 1);
}

function parallel(values) {
  return 1 - serial(values.map((value) => 1 - value));
}

function report(label, value) {
  const downtime = 1 - value;
  const nines = downtime > 0
    ? (-Math.log10(downtime)).toFixed(3)
    : "beyond precision";
  console.log(label);
  console.log(`  availability: ${(value * 100).toFixed(6)}%`);
  console.log(`  nines:        ${nines}`);
  console.log(`  downtime:     ${(downtime * 43200).toFixed(2)} min/month`);
  console.log(`                ${(downtime * 525600).toFixed(2)} min/year`);
  console.log();
}

function readSystem(path) {
  const lines = fs.readFileSync(path, "utf8").trim().split(/\r?\n/);
  if (lines[0].trim() !== "component,nines,group") {
    throw new Error("CSV header must be component,nines,group");
  }
  const groups = new Map();
  for (const line of lines.slice(1)) {
    if (!line.trim()) continue;
    const cells = line.split(",").map((cell) => cell.trim());
    const [component, nines, group] = cells;
    if (cells.length !== 3 || !component || !nines || !group) {
      throw new Error("each row needs a component, nines, and group");
    }
    if (!groups.has(group)) groups.set(group, []);
    groups.get(group).push(probability(Number(nines)));
  }
  if (groups.size === 0) throw new Error("CSV must contain components");
  return serial([...groups.values()].map(parallel));
}

function main() {
  const args = process.argv.slice(2);
  if (args.length === 1 && args[0] === "--demo") {
    const a = probability(3);
    report("single 99.9%", a);
    report("serial 99.9% + 99.9%", serial([a, a]));
    report("parallel 99.9% || 99.9%", parallel([a, a]));
    report("combined LB -> (API || API) -> DB",
      serial([a, parallel([a, a]), probability(4)]));
  } else if (args.length === 2 && args[0] === "--csv") {
    report("SYSTEM", readSystem(args[1]));
  } else if (args[0] === "--nines") {
    const topologyIndex = args.indexOf("--topology");
    const inputs = args.slice(1, topologyIndex === -1 ? args.length : topologyIndex);
    const topology = topologyIndex === -1 ? "serial" : args[topologyIndex + 1];
    if (inputs.length === 0 || !["serial", "parallel"].includes(topology) ||
        (topologyIndex !== -1 && topologyIndex + 2 !== args.length)) {
      throw new Error("provide nines values and a serial or parallel topology");
    }
    const values = inputs.map((input) => probability(Number(input)));
    report(topology, topology === "serial" ? serial(values) : parallel(values));
  } else {
    throw new Error("Usage: node availability.js --demo | --csv FILE | --nines N [N ...] [--topology serial|parallel]");
  }
}

try {
  main();
} catch (error) {
  console.error(`Error: ${error.message}`);
  process.exitCode = 1;
}
```

**What this file does:** Calculates availability and expected downtime from nines or CSV groups.

---

## Step 3: Write the System CSV

For the CSV design below, `SYSTEM` has **99.889910%** availability and **578.63 minutes** of yearly downtime. The demo produces the same values under its combined-design heading.

Save the original design in `components.csv`:

```csv
component,nines,group
LB-1,3,lb
API-server-1,3,api
API-server-2,3,api
DB-primary,4,db
```

**What this file does:** Describes the load balancer, two alternative apps, and database.

Rows in the same group are alternatives. Different groups are required. Both `api` rows do the same job, so either can serve the request. The `lb` and `db` groups have separate required jobs. Use `3` for three nines, not `99.9`.

---

## Step 4: Run the Calculator

Run from `m3-l1` after saving `availability.js`:

```bash
node availability.js --demo
```

**Explanation:** Runs the four built-in availability examples.

**What to expect:** Four reports show single, serial, parallel, and combined designs. Each prints availability, nines, and downtime in minutes. The demo uses built-in values; it does not read `components.csv`.

| Design | Availability | Yearly downtime |
| --- | --- | --- |
| Single three-nine part | 99.9% | 525.60 minutes |
| Two required parts | 99.800100% | 1050.67 minutes |
| Two alternative apps | 99.999900% | 0.53 minutes |
| Original combined design | 99.889910% | 578.63 minutes |

Calculate the design saved in your CSV file:

```bash
node availability.js --csv components.csv
```

**What to expect:** The `SYSTEM` report shows **99.889910%** availability and **578.63 minutes/year** of downtime.

You can also compare components directly. Each number is a count of nines; the default topology is serial:

```bash
node availability.js --nines 3 3 --topology serial
node availability.js --nines 3 3 --topology parallel
```

**CSV format:** Keep the header exactly as shown and use simple values without commas or quotation marks inside fields. The calculator reads this lab’s simple comma-separated format.

### Improve the Load Balancer

In `components.csv`, change `LB-1,3,lb` to `LB-1,4,lb`, leaving the apps and database unchanged. Save the file and run the CSV calculation again:

```bash
node availability.js --csv components.csv
```

This run uses your updated CSV; `--demo` always uses its original built-in values.

**What to expect:** Availability becomes **99.979901%**, with **105.64 minutes** of yearly downtime. That is about **1.76 hours**, down from **9.64 hours**. Improving a real service needs engineering work.

---

## Conclusion

You learned that every required part adds a way for a request to fail, while alternative app copies let the service keep working when one app stops. Calculate each group of alternatives first, then multiply the required parts: backup apps still need a working load balancer and database. Next, use SLOs and error budgets to set a reliability target and decide how many failures are acceptable.
