# Simulating Failure Models

In a real distributed system, things break in three common ways: a node **crashes**, a node **drops messages**, or a node is **slow**. In this lab you build a 3-node service, inject all three failure types, and watch how each one shows up on the client side. Then you test which retry policy actually helps, and which one makes things worse.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/main-diagram.png" alt="Failure injection overview: client talks to a 3-node service through toxiproxy, where we add crash, omission, and temporal failures" width="100%" />

## Objective

By the end of this lab you will be able to:

- Inject **crash**, **omission**, and **temporal** failures using `toxiproxy`.
- Observe how each failure shows up in client logs.
- Match each failure to the right **retry policy**: no retry, bounded retry + timeout, fast retry without backoff, retry with longer timeout.

## Deliverable

A failure log that shows:

1. **Crash failure:** what the client sees when a node is killed.
2. **Omission failure:** what the client sees when some messages are dropped.
3. **Temporal failure:** what the client sees when a node gets slow.
4. **Retry table:** which retry policy helps for which failure and which one makes it worse.

## Failure models you will simulate

In a distributed system, faults come in three types. You will inject all three.

### Crash failure

A node stops responding forever. The process is dead, and no replies come back, even though the network itself is still alive. **Crash** is the simplest failure to understand and the hardest to handle, because a dead node never sends any signal that it is gone.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/crash_failure.png" alt="Crash failure timeline: the node process stops and no replies come back" width="100%" />

### Omission failure

The node is alive but **drops some messages**. Some requests get through, others vanish. The client sees a flaky connection: half the calls succeed, half fail. This is common on real networks because of packet loss, overloaded routers, or full socket buffers.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/omission_failure.png" alt="Omission failure: some requests reach the node and others do not, producing a flaky link" width="100%" />

### Temporal failure (delay)

The node is alive and accepts every message but is **very slow**. Every reply takes a long time. This is the trickiest case because the client has to guess: is the node dead (stop retrying) or just slow (keep waiting)?

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/temporal-failure.png" alt="Temporal failure: every reply is delayed, so round-trip time grows far above the client timeout" width="100%" />


## 1. Tools you will use

### 1.1 `toxiproxy`

`toxiproxy` is a TCP proxy that sits **between** the client and the server. You point the client at the proxy, and the proxy then forwards traffic to the real server. From there you can drop packets, reset connections, or add delay at runtime without touching the server itself.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/toxiproxy.png" alt="Toxiproxy sits between the client and the server, forwarding traffic and injecting faults at runtime through its admin API" width="100%" />

The proxy exposes an **admin API on port 8474**. We send `POST` requests to that API to add or remove faults at runtime. No restart of the client or server is needed.

### 1.2 `toxiproxy` toxic types we will use

| Mechanism                | What it does                                                              | Used for                |
| ------------------------ | ------------------------------------------------------------------------- | ----------------------- |
| `latency` toxic          | Adds a fixed delay (ms) to every byte that passes through                  | Temporal failure        |
| Listener toggle          | Sets `enabled: false` on a proxy, so new connections are refused (`ECONNREFUSED`) | Omission failure  |


## 2. File layout

```text
m2-l2/
├── docker-compose.yml
├── package.json
├── server.js
├── client.js
├── toxiproxy.json
└── README.md
```


## 3. Lab topology

Before you paste in any files, look at what you are building. The client container can reach the three app nodes **only through the toxiproxy container**. You never connect directly to a node.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/topology.png" alt="Lab topology: the client reaches the three app nodes only through the toxiproxy container on the labnet bridge network" width="100%" />

**Reading the map:**
- The client always calls `toxiproxy:30001`, `toxiproxy:30002`, `toxiproxy:30003`.
- Toxiproxy then opens a real TCP connection to the matching node on port 3000.
- To inject a fault, we POST a toxic rule to the toxiproxy admin API on port 8474.

### 3.1 Where each port lives

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/port-mapping.png" alt="Port mapping: the admin API on host port 8474 maps to toxiproxy, and data ports 30001 to 30003 are reached from inside the client container" width="100%" />

The admin API is reached from the host as `http://127.0.0.1:8474` (the compose file maps `8474:8474`). The data ports are reached **from inside the client container** as `toxiproxy:30001`, `toxiproxy:30002`, `toxiproxy:30003`.


## 4. File contents

### 4.1 `docker-compose.yml`

```yaml
# Three identical app nodes, one client, one toxiproxy.
# Each app node listens on 3000 inside its container.
# The client talks to a 3-node service through toxiproxy.

services:
  node-1:
    image: node:20-alpine
    container_name: node-1
    command: sh -c "node /app/server.js 1 3000"
    volumes:
      - ./:/app
    working_dir: /app
    networks:
      - labnet
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O- http://127.0.0.1:3000/ping >/dev/null 2>&1 || exit 1"]
      interval: 1s
      timeout: 1s
      retries: 10
      start_period: 1s

  node-2:
    image: node:20-alpine
    container_name: node-2
    command: sh -c "node /app/server.js 2 3000"
    volumes:
      - ./:/app
    working_dir: /app
    networks:
      - labnet
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O- http://127.0.0.1:3000/ping >/dev/null 2>&1 || exit 1"]
      interval: 1s
      timeout: 1s
      retries: 10
      start_period: 1s

  node-3:
    image: node:20-alpine
    container_name: node-3
    command: sh -c "node /app/server.js 3 3000"
    volumes:
      - ./:/app
    working_dir: /app
    networks:
      - labnet
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O- http://127.0.0.1:3000/ping >/dev/null 2>&1 || exit 1"]
      interval: 1s
      timeout: 1s
      retries: 10
      start_period: 1s

  toxiproxy:
    image: ghcr.io/shopify/toxiproxy:2.9.0
    container_name: toxiproxy
    command: -host 0.0.0.0 -config /config/toxiproxy.json
    ports:
      - "8474:8474"   # admin API on host
    volumes:
      - ./toxiproxy.json:/config/toxiproxy.json:ro
    networks:
      - labnet

  client:
    image: node:20-alpine
    container_name: client
    depends_on:
      node-1:
        condition: service_healthy
      node-2:
        condition: service_healthy
      node-3:
        condition: service_healthy
      toxiproxy:
        condition: service_started
    command: sh -c "node /app/client.js"
    volumes:
      - ./:/app
    working_dir: /app
    networks:
      - labnet

networks:
  labnet:
    driver: bridge
```

**What this file does:**
- Starts 3 app nodes (`node-1`, `node-2`, `node-3`), one `toxiproxy` proxy, and one `client`, all on the `labnet` network.
- App nodes wait for each other to be healthy, then the client waits for the app nodes and toxiproxy.

### 4.2 `toxiproxy.json`

```json
[
  {
    "name": "node-1",
    "listen": "0.0.0.0:30001",
    "upstream": "node-1:3000",
    "enabled": true
  },
  {
    "name": "node-2",
    "listen": "0.0.0.0:30002",
    "upstream": "node-2:3000",
    "enabled": true
  },
  {
    "name": "node-3",
    "listen": "0.0.0.0:30003",
    "upstream": "node-3:3000",
    "enabled": true
  }
]
```

**What this file does:**
- Defines 3 listeners, one per node. Each maps a public port on the proxy to the node's private port.
- Toxic rules get added per listener at runtime through the admin API.

### 4.3 `package.json`

```json
{
  "name": "failure-models-lab",
  "version": "1.0.0",
  "private": true,
  "description": "Lab 2 - Simulating Failure Models",
  "scripts": {
    "up": "docker compose up --build",
    "down": "docker compose down",
    "logs": "docker compose logs -f client"
  }
}
```


### 4.4 `server.js`

```javascript
"use strict";
const http = require("http");

// Args: <node-id> <port>
const NODE_ID = process.argv[2] || "0";
const PORT = parseInt(process.argv[3] || "3000", 10);

const server = http.createServer((req, res) => {
  const t0 = Date.now();
  // Reply with the same id every time so the client can tell nodes apart.
  const body = JSON.stringify({
    node: NODE_ID,
    pid: process.pid,
    path: req.url,
    served_at: new Date().toISOString(),
  });
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(body);
  console.log(
    `[node-${NODE_ID} pid=${process.pid}] ${req.method} ${req.url} in ${Date.now() - t0}ms`,
  );
});

server.listen(PORT, "0.0.0.0", () => {
  console.log(`[node-${NODE_ID} pid=${process.pid}] listening on 0.0.0.0:${PORT}`);
});
```

**What this file does:**
- A small HTTP server that listens on the port passed in (3000 for every node).

### 4.5 `client.js`

```javascript
"use strict";
const http = require("http");

// One round of calls = one call per node.
// Runs N rounds so we get enough samples to see failure patterns.
const ROUNDS = parseInt(process.env.ROUNDS || "20", 10);
const TIMEOUT_MS = parseInt(process.env.TIMEOUT_MS || "1500", 10);
const TARGETS = [
  { name: "node-1", host: "toxiproxy", port: 30001 },
  { name: "node-2", host: "toxiproxy", port: 30002 },
  { name: "node-3", host: "toxiproxy", port: 30003 },
];

const stats = {};
for (const t of TARGETS) stats[t.name] = { ok: 0, fail: 0, ok_rtts: [], fail_rtts: [] };

function callOne(target) {
  return new Promise((resolve) => {
    const t0 = Date.now();
    const req = http.request(
      {
        host: target.host,
        port: target.port,
        path: "/ping",
        method: "GET",
        timeout: TIMEOUT_MS,
      },
      (res) => {
        let data = "";
        res.on("data", (c) => (data += c));
        res.on("end", () => {
          const rtt = Date.now() - t0;
          stats[target.name].ok++;
          stats[target.name].ok_rtts.push(rtt);
          resolve({ ok: true, rtt, body: data });
        });
      },
    );
    req.on("timeout", () => {
      const rtt = Date.now() - t0;
      stats[target.name].fail++;
      stats[target.name].fail_rtts.push(rtt);
      req.destroy();
      resolve({ ok: false, rtt, error: "timeout" });
    });
    req.on("error", (err) => {
      const rtt = Date.now() - t0;
      stats[target.name].fail++;
      stats[target.name].fail_rtts.push(rtt);
      resolve({ ok: false, rtt, error: err.code || err.message });
    });
    req.end();
  });
}

function summarize(arr) {
  if (!arr.length) return { avg: "-", max: "-" };
  const avg = (arr.reduce((a, b) => a + b, 0) / arr.length).toFixed(1);
  const max = Math.max(...arr);
  return { avg, max };
}

(async () => {
  console.log(
    `[client] starting ${ROUNDS} rounds, ${TIMEOUT_MS}ms timeout per call`,
  );

  for (let r = 0; r < ROUNDS; r++) {
    const results = await Promise.all(TARGETS.map(callOne));
    for (let i = 0; i < TARGETS.length; i++) {
      const t = TARGETS[i];
      const res = results[i];
      if (res.ok) {
        console.log(
          `round ${r + 1} ${t.name} OK rtt=${res.rtt}ms body=${res.body.slice(0, 60)}`,
        );
      } else {
        console.log(
          `round ${r + 1} ${t.name} FAIL rtt=${res.rtt}ms err=${res.error}`,
        );
      }
    }
    await new Promise((r) => setTimeout(r, 250));
  }

  console.log("\n=== summary ===");
  for (const t of TARGETS) {
    const s = stats[t.name];
    const total = s.ok + s.fail;
    const rate = total === 0 ? 0 : ((s.ok / total) * 100).toFixed(1);
    const ok = summarize(s.ok_rtts);
    const fail = summarize(s.fail_rtts);
    console.log(
      `${t.name}: ${s.ok}/${total} ok (${rate}%) ` +
        `ok_avg_rtt=${ok.avg}ms ok_max_rtt=${ok.max}ms ` +
        `fail_avg_rtt=${fail.avg}ms fail_max_rtt=${fail.max}ms`,
    );
  }
})();
```

**What this file does:**
- Runs `ROUNDS` rounds of calls, one per node per round, all through toxiproxy.
- Each call has a `TIMEOUT_MS` deadline so a slow or down node fails fast.
- Prints a summary at the end with per-node ok/fail counts and RTTs.


## 5. Run the following commands into the terminal

### 5.1 Install dependencies and bring up the lab

```bash
cd m2-l2
docker compose up --build
```

**Expected:** all 5 containers start and the client begins its 20 rounds.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/resources.png" alt="Docker Desktop resource panel after `docker compose up --build`: five containers listed as Running, all five on the labnet network" width="100%" />



### 5.2 Baseline: no failures injected

The first run is the **baseline**. All three nodes are healthy. Every call should succeed with a low RTT.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/all-node.png" alt="Client log output during the baseline run: 20 rounds of OK responses from node-1, node-2, and node-3 with RTTs around 5-15 ms" width="100%" />


**Baseline timeline** (one round, healthy):

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/baseline-timeline.png" alt="Baseline wire timeline: each healthy round-trip completes in around 5 to 15 ms across all three nodes" width="100%" />

### 5.3 About the retry policies

For each failure, you will see a one-line **Recommendation** at the end of the section. The full reasoning for every choice is collected in the [Retry policy reference](#10-retry-policy-reference) so you do not have to read it three times. The short version: each failure has its own wire signature, and the retry policy must be picked to match that signature, not the symptom you happened to notice.


## 6. Failure 1: Crash

Kill `node-1` mid-flight. The process is gone, so no replies come back.

Run the command in terminal:

```bash
docker compose kill node-1
```

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/kill-node-1.png" alt="Terminal screenshot showing `docker compose kill node-1` exiting with no error after node-1 is stopped" width="100%" />

You will see the crash log in the Docker logs.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/node1-crash-log.png" alt="Client logs after the kill: every call to node-1 prints FAIL err=ECONNRESET while node-2 and node-3 keep returning OK" width="100%" />

Now restart the client:

```bash
docker compose restart client
```

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/restart%20client.png" alt="Terminal screenshot showing `docker compose restart client` returning the client container to a fresh state for the next run" width="100%" />

See the logs:

```bash
docker compose logs client | grep -E "=== summary|node-[0-9]:"
```

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/node1-grep.png" alt="Grep output filtered to summary lines: node-1 at 0/20 ok with ECONNRESET, node-2 and node-3 at 20/20 ok" width="100%" />

### 6.1 Wire timeline

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/crash-failure2.png" alt="Crash failure wire timeline: node-1 stops, the client gets ECONNRESET on every connect through the proxy" width="100%" />

### 6.2 Recommendation

**No retry** for a node you know is dead; **bounded retry + timeout** if you do not yet know the state.

### 6.3 Reset

```bash
docker compose start node-1
docker compose restart client
```

## 7. Failure 2: Omission

Make `node-2`'s link flaky: every other round, the proxy is disabled so the client gets a fast `ECONNRESET`. The other rounds pass through normally. This models the packet-loss pattern where some requests vanish and others get through.

Toggle the `node-2` proxy off/on so the client sees a flaky link. Start the loop first, then restart the client so they overlap. Paste the whole block at once; the loop runs in the background while the client restarts:

```bash
(for i in $(seq 1 5); do
  curl -s -X POST http://127.0.0.1:8474/proxies/node-2 \
    -H "Content-Type: application/json" \
    -d '{"name":"node-2","listen":"0.0.0.0:30002","upstream":"node-2:3000","enabled":false}' > /dev/null
  sleep 0.6
  curl -s -X POST http://127.0.0.1:8474/proxies/node-2 \
    -H "Content-Type: application/json" \
    -d '{"name":"node-2","listen":"0.0.0.0:30002","upstream":"node-2:3000","enabled":true}' > /dev/null
  sleep 0.6
done) &
sleep 0.5
docker compose restart client
wait
docker compose logs client | grep -E "=== summary|node-[0-9]:"
```



**Expected:** `node-2: ~10/20 ok (50.0%) fail_avg_rtt=~2-12ms err=ECONNREFUSED`; `node-1`, `node-3`: 20/20 ok.

Record this in the failure log table (Section 9) alongside the symptom you actually saw.

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/failure2.png" alt="Omission failure summary: node-2 at roughly 10/20 ok with FAIL err=ECONNREFUSED on the missed rounds, node-1 and node-3 at 20/20 ok" width="100%" />

### 7.1 Wire timeline

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/omission2.png" alt="Omission failure wire timeline: half the requests to node-2 are refused by the disabled proxy and the other half succeed at around 5 ms" width="100%" />

### 7.2 Recommendation

**Fast retry, no backoff.** The link is healthy and the next attempt almost always gets through.

### 7.3 Reset

```bash
docker compose restart client
```


## 8. Failure 3: Temporal (Delay)

Add 2 s latency to `node-3`. The node is alive but every reply is late.

### 8.1 Inject the toxic, restart, and check logs

Inject the latency toxic first, then restart the client so it picks up the slow-down:

```bash
curl -s -X POST http://127.0.0.1:8474/proxies/node-3/toxics \
  -H "Content-Type: application/json" \
  -d '{"type":"latency","attributes":{"latency":2000},"name":"slow-down"}'
docker compose restart client
docker compose logs client | grep -E "=== summary|node-[0-9]:"
```

You will see:

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/failure3.png" alt="Temporal failure summary: node-3 at 0/20 ok with FAIL err=timeout at ~1500 ms, node-1 and node-2 at 20/20 ok" width="100%" />

### 8.2 Wire timeline

<img src="https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2002/images/temporal2.png" alt="Temporal failure wire timeline: every request to node-3 takes around 2000 ms, exceeding the 1500 ms client timeout" width="100%" />

### 8.3 Recommendation

**Retry with a longer timeout if the SLA allows, otherwise fail fast.** The server is consistently slow, not flaky, so backoff does not help.

### 8.4 Reset

Remove the latency toxic so the lab returns to baseline before the next run:

```bash
curl -s -X DELETE http://127.0.0.1:8474/proxies/node-3/toxics/slow-down
docker compose restart client
```


## 9. Failure log table

Run each failure, then compare what you observed against the expected values below. For the rationale behind each Recommendation, see the [Retry policy reference](#10-retry-policy-reference).

| Failure   | Node affected | Expected symptom | Expected ok/total | Observed symptom | Observed ok/total |
| --------- | ------------- | ---------------- | ----------------- | ---------------- | ----------------- |
| Crash     | node-1        | `err=ECONNRESET` from the proxy when it tries to forward to the dead upstream, ~40-150 ms | 0/20    | `err=ECONNRESET`, rtt ≈ 46-147 ms (avg 51.5 ms) | 0/20 |
| Omission  | node-2        | `FAIL err=ECONNREFUSED` on ~50% of rounds, ~2-12 ms         | ~10/20  | `err=ECONNREFUSED`, rtt ≈ 7-11 ms (avg 7.1 ms) | 11/20 (55.0%) |
| Temporal  | node-3        | `FAIL err=timeout` at ~1500 ms on every call                | 0/20    | `err=timeout`, rtt ≈ 1502-1518 ms (avg 1507 ms) | 0/20 |


## 10. Retry policy reference

A single table comparing the five common retry policies against the three failure types. Each cell shows what actually happens at the client.

| Policy | Crash (fast RST / refused) | Omission (fast error, ~50%) | Temporal (slow reply) |
|---|---|---|---|
| **No retry** | **Fastest** (~50 ms): kernel signal surfaces immediately | 50% user errors: link is otherwise healthy | Fast error in 1.5 s: explicit, user knows it failed |
| **Bounded retry + timeout** | **Principled default** (3 × 1.5 s ≈ 4.5 s): caps total wait, also covers transient restarts | Succeeds but wastes attempts | Fails 3×, ~4.5 s wasted: each retry hits the same wall |
| **Fast retry, no backoff** | Same as no retry in wall time | **Best** (succeeds on retry 2, ~50 ms): link healed in microseconds | Fails 3× at 1.5 s each: still too short |
| **Retry with exponential backoff** | Wastes ~10.6 s: ~50 ms × 3 plus 10.5 s of backoff on a node that will not return | Works but slower than needed | Fails 3×: backoff cannot speed up a server that is slow on every call |
| **Retry with longer timeout** | Same as no retry if the node is truly dead | Works but unnecessary | **Best if SLA allows** (5 s × 3 ≈ 3.5 s on first success): gives the slow server room to reply |

**The rule of thumb:** match the policy to the *cause*, not the symptom. A dead node is not coming back (cap and give up). A flaky link heals in microseconds (retry immediately). A slow server stays slow (raise the timeout, or fail fast).

## Conclusion

You saw three failures, each with its own wire signature: a **crash** surfaces as `ECONNRESET` within tens of milliseconds, an **omission** surfaces as `ECONNREFUSED` on roughly half of the calls, and a **temporal** failure surfaces as `timeout` because the 2 s latency always exceeds the 1.5 s client deadline. Match the retry policy to the signature.