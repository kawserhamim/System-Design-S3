# Hello, Distributed System

A **distributed system** is a group of independent computers that talk over a network. They send messages to each other. From the outside, they look like one system. In this lab we mock a distributed system with a walkie-talkie. We use two Node.js programs. They share one TCP channel. Each has its own timer and memory. They talk until one walks away mid-sentence.

![Archetecture-Diagram](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/Archetecture-Diagram.gif)

## Objective

Feel a real network before the course adds abstractions on top. In this lab you will see three things: **latency**, **partial failure**, and **process independence**.

## Deliverable

- A minimal two-process toy (`server.js` + `client.js` over TCP)
- A written log of the first observed failure

## What is a Distributed System?

A **distributed system** is many independent computers joined by a wire. They pass messages to look like one system. Each node has its own memory, power, and clock.

The wire makes it distributed, not the number of machines. Once two processes use the network, messages take time and either side can vanish. That is why we mock two nodes here.

### Challenges of a Distributed System

Once two processes talk over a network, three challenges will always show up. You cannot remove them. Every later lab is about handling them.

1. **Latency** - Messages take time to travel. We call it delay **RTT (round-trip time)**: how long it takes to send a message and get the reply back.

![RTT](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/RTT.png)

2. **Partial Failure** - The other side can stop at any time. There is no warning. You only find out when you try to talk again. Your code keeps running. The other side is just gone.

![Partial Failure](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/Partial%20Failure.gif)

3. **Independence** - Neither side knows what the other is doing. Each has its own memory, timer, and process ID. The wire is all they share.

In plain words: messages are slow, partners can vanish, and no one shares memory.

### Let's Understand the Distributed System Though Walkie-Talkie Picture

Two kids. Two walkie-talkies. One channel. Anyone can walk away.

![Kids talking](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/Kids%20talking.png)

Now think of two kids as two node processes. They talk over TCP. It looks like this:

![Walkie-app-demo](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/Walkie-app-demo.png)

Now see how this picture mocks a distributed system.

| Distributed Concept | Walkie-Talkie Reality                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------- |
| **Latency** (RTT)   | Press the button. Wait a moment. Hear the reply. That wait is the wire.                        |
| **Partial Failure** | The other kid walks away mid-sentence. You find out when you press talk again and hear static. |
| **Independence**    | Each kid has its own battery, timer, and channel. Only the air connects them.                  |

In this section Kid A is the client and Kid B is the server. Let's try it hands-on.

## Step 1: Create the Lab Folder

Open Poridhi's terminal. Make the exercise folder:

```bash
mkdir sd-lab-02 && cd sd-lab-02
node --version
```

**Expected:** `v18.x.x` or newer.

## Step 2: Write `server.js` (Kid B Listens)

Create `server.js` and paste:

```javascript
"use strict";
const net = require("net");

const PORT = 4000;
const IDLE_TIMEOUT_MS = 5000;
let messageCounter = 0;

const server = net.createServer((socket) => {
  console.log(`[server pid=${process.pid}] client connected`);

  // Attach error handler FIRST. If a client RSTs during handshake, Node crashes
  // if no 'error' listener is attached before the first socket.write().
  socket.on("error", (err) =>
    console.error(
      `[server pid=${process.pid}] socket error: ${err.code || err.message}`,
    ),
  );
  socket.on("end", () => {
    clearTimeout(idleTimer);
    console.log(`[server pid=${process.pid}] client hung up`);
  });

  socket.write(
    JSON.stringify({
      type: "greeting",
      pid: process.pid,
      counter: ++messageCounter,
    }) + "\n",
  );

  let buffer = "";
  socket.setEncoding("utf8");

  let idleTimer = setTimeout(() => {
    console.log(
      `[server pid=${process.pid}] idle for ${IDLE_TIMEOUT_MS}ms, dropping the line`,
    );
    socket.end();
  }, IDLE_TIMEOUT_MS);

  socket.on("data", (chunk) => {
    clearTimeout(idleTimer);
    idleTimer = setTimeout(() => {
      console.log(
        `[server pid=${process.pid}] idle for ${IDLE_TIMEOUT_MS}ms, dropping the line`,
      );
      socket.end();
    }, IDLE_TIMEOUT_MS);

    buffer += chunk;
    let idx;
    while ((idx = buffer.indexOf("\n")) !== -1) {
      const line = buffer.slice(0, idx).trim();
      buffer = buffer.slice(idx + 1);
      if (!line) continue;
      console.log(`[server pid=${process.pid}] received "${line}"`);
      socket.write(
        JSON.stringify({
          type: "ack",
          pid: process.pid,
          counter: ++messageCounter,
          echo: line,
        }) + "\n",
      );
    }
  });
});

server.listen(PORT, "127.0.0.1", () =>
  console.log(`[server pid=${process.pid}] listening on 127.0.0.1:${PORT}`),
);

process.on("SIGINT", () => server.close(() => process.exit(0)));
```

**Explanation:** `net.createServer` is the kid holding the radio. The idle timer is like the kid getting bored. The error handler is like the kid noticing the channel went wrong. Without it, Node crashes on the first error.

---

## Step 3: Write `client.js` (Kid A Talks First)

Create `client.js` and paste:

```javascript
"use strict";
const net = require("net");

const HOST = "127.0.0.1";
const PORT = 4000;
const MESSAGES = process.argv[2]
  ? [process.argv[2]]
  : ["hello-1", "hello-2", "hello-3"];
const HARD_TIMEOUT_MS = 8000;
const IDLE_TIMEOUT_MS = 5000;

const sendTimes = new Map();
const rtts = [];
const socket = net.createConnection({ host: HOST, port: PORT }, () => {
  console.log(`[client pid=${process.pid}] connected to ${HOST}:${PORT}`);
});

let buffer = "";
socket.setEncoding("utf8");

let idleTimer = setTimeout(() => {
  console.log(`[client pid=${process.pid}] idle ${IDLE_TIMEOUT_MS}ms, ending`);
  socket.end();
}, IDLE_TIMEOUT_MS);

socket.on("data", (chunk) => {
  clearTimeout(idleTimer);
  idleTimer = setTimeout(() => {
    console.log(
      `[client pid=${process.pid}] idle ${IDLE_TIMEOUT_MS}ms, ending`,
    );
    socket.end();
  }, IDLE_TIMEOUT_MS);

  buffer += chunk;
  let idx;
  while ((idx = buffer.indexOf("\n")) !== -1) {
    const line = buffer.slice(0, idx).trim();
    buffer = buffer.slice(idx + 1);
    if (!line) continue;
    let msg;
    try {
      msg = JSON.parse(line);
    } catch {
      continue;
    }
    if (msg.type === "greeting") {
      console.log(
        `[client pid=${process.pid}] heard greet from server pid=${msg.pid}`,
      );
      setTimeout(sendNext, 250);
    } else if (msg.type === "ack") {
      const sent = sendTimes.get(msg.echo);
      const rtt = sent ? Date.now() - sent : -1;
      rtts.push(rtt);
      console.log(
        `[client pid=${process.pid}] heard ack for "${msg.echo}", RTT=${rtt}ms`,
      );
    }
  }
});

let sentIdx = 0;
function sendNext() {
  if (sentIdx >= MESSAGES.length) return;
  const text = MESSAGES[sentIdx++];
  sendTimes.set(text, Date.now());
  console.log(`[client pid=${process.pid}] -> saying "${text}"`);
  socket.write(text + "\n");
  if (sentIdx < MESSAGES.length) setTimeout(sendNext, 1000); // 1s between sends; kill window is ~1 second wide after the first ack
}

const hardTimeout = setTimeout(() => {
  const msg = `[client pid=${process.pid}] hard timeout ${HARD_TIMEOUT_MS}ms, forcing exit\n`;
  try {
    require("fs").writeSync(2, msg);
  } catch (_) {}
  socket.destroy();
  process.exit(1);
}, HARD_TIMEOUT_MS);

socket.on("end", () => {
  clearTimeout(idleTimer);
  clearTimeout(hardTimeout);
  console.log(
    `[client pid=${process.pid}] line closed. RTT samples: [${rtts.join(", ")}] ms`,
  );
  process.exit(0);
});
socket.on("error", (err) => {
  clearTimeout(idleTimer);
  clearTimeout(hardTimeout);
  // Use stderr (unbuffered) and fs.writeSync (sync I/O) to make sure the
  // error message survives process.exit(). console.error to a redirected file
  // can be lost because Node's stdout/stderr are buffered.
  const msg = `[client pid=${process.pid}] socket error: ${err.code || err.message}\n`;
  try {
    require("fs").writeSync(2, msg);
  } catch (_) {}
  process.exit(1);
});
```

**Explanation:** `sendTimes` lets the client compute `Date.now() - sent` for each reply. That number is the **RTT**. The 5-second idle timer matches the server's. The 8-second hard timeout is a backup that makes sure the client never hangs forever.

---

## Step 4: Run Both Kids Talking

Open **two terminals** in your vscode.

**Terminal A: start the server:**

```bash
node server.js
```

**Expected:**

![server-log](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/server-log.png)


**Terminal B: start the client:**

```bash
node client.js
```

**Expected on the client:**

![client-log](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/client-log.png)

### Process Map While Talking

![Process mapping](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/Process%20mapping.png)

### Check That They Are Two Separate Processes

Open **Terminal C**. Run a filter that matches only the lab's two scripts:

```bash
ps -ef | grep -E 'node (server|client)\.js' | grep -v grep
```

You will see only one process now (the client exits after the 5-second idle timer, the server keeps running). Run the same check a few seconds later - only the server should remain:

![process-log](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/process-log.png)

## Step 5: Feel the Latency

Goal: watch the round-trip number grow when the network adds delay. RTT is almost zero on localhost. Let's make it real. In `server.js`, wrap the ack write in a `setTimeout`:

```javascript
setTimeout(() => {
  socket.write(
    JSON.stringify({
      type: "ack",
      pid: process.pid,
      counter: ++messageCounter,
      echo: line,
    }) + "\n",
  );
}, 200);
```

Restart the server. Press Ctrl+C in Terminal A. Then run `node server.js` again. Run the client again.

**Expected:** RTT now reads about **200 ms** per message.

![enhancedRTT](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/enhancedRTT.png)

## Step 6: Kid B Walks Away (Chaos Test)

This is the heart of the lab. Goal: see the kernel make the call when one side vanishes while the other keeps running.

Now make a file called `chaos.sh`:

```bash
#!/usr/bin/env bash
set -u
cd "$(dirname "${BASH_SOURCE[0]}")"
echo "=== chaos.sh starting ==="

# Remove stale logs from any previous run. Otherwise the grep loops below
# may match a leftover "saying" or "heard ack" from a prior chaos.sh run.
rm -f server.log client.log

node server.js > server.log 2>&1 &
SERVER_PID=$!
echo "Server started, pid=$SERVER_PID"

# Use `nc -z` instead of `echo > /dev/tcp/...`. The echo form opens a TCP
# socket and never closes it, leaking one file descriptor per retry. nc -z
# just probes the port and exits cleanly.
for i in $(seq 1 20); do
  nc -z 127.0.0.1 4000 2>/dev/null && { echo "Server is up"; break; }
  sleep 0.1
done

# Optional: pass one arg to send a single custom message, e.g. ./chaos.sh slow-payload
if [ "$#" -gt 0 ]; then
  node client.js "$1" > client.log 2>&1 &
else
  node client.js > client.log 2>&1 &
fi
CLIENT_PID=$!
echo "Client started, pid=$CLIENT_PID"

for i in $(seq 1 50); do grep -q 'saying' client.log 2>/dev/null && break; sleep 0.1; done

# Wait for hello-1's ack. With 200ms server delay + localhost, hello-1 ack
# is logged at ~450ms after connect. After this, the client sits idle for
# ~1 second before sending hello-2. Killing NOW means the server is dead
# before hello-2 fires. The fail sound depends on what the client does next:
#   - Client is idle (within 1s): server process dies, kernel sends FIN.
#     Client gets `line closed`, exit 0.
#   - Client writes hello-2 at a slow pace (5s+): that write hits the dead
#     socket before FIN, kernel sends RST. Client gets `ECONNRESET`, exit 1.
for i in $(seq 1 30); do grep -q 'heard ack for "hello-1"' client.log 2>/dev/null && break; sleep 0.1; done

# Brief pause so the server's ack has fully flushed to the client before we kill.
sleep 0.1

echo ">>> Killing server pid=$SERVER_PID with SIGKILL ..."
kill -9 "$SERVER_PID" 2>/dev/null || echo "(server already gone)"

wait "$CLIENT_PID" 2>/dev/null
CLIENT_EXIT=$?
echo "Client exited with code $CLIENT_EXIT"

echo; echo "=== server.log ==="; cat server.log
echo; echo "=== client.log ==="; cat client.log
```

> **Before running:** kill any leftover server from earlier steps. If port 4000 is busy, `chaos.sh` fails with `EADDRINUSE`. Your "chaos" run hits a stale server:
>
> ```bash
> pkill -9 -f 'node server.js' 2>/dev/null
> ```
>
> Verify the port is free:
>
> ```bash
> (echo > /dev/tcp/127.0.0.1/4000) 2>&1 | grep -q 'refused' && echo "port free" || echo "port still busy"
>
> ```

![verify-free](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/verify-free.png)

Make it executable and run it:

```bash
chmod +x chaos.sh && ./chaos.sh
```

**Expected shape:**

![chaos](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2002/images/chaos.png)

There are no **client hung up** or **idle** logs in `client.log`. That means the server died mid-chat. The client stayed alive (`line closed`, exit 0). That is a partial failure.


| Concept | What you saw |
|---|---|
| **Latency** | You measured RTT twice. First about 1 to 3 ms. Then, after you added the delay, about 200 ms. That number is the network delay. It is not Node or Linux. |
| **Partial Failure** | The server was killed mid-chat with SIGKILL. One side vanished, the other kept running. |
| **Independence** | You saw 2 PIDs, 2 timers, 2 memories. The moment you crossed the wire, you gave up the easy way to share data between programs. |


## Conclusion
In this lab you turned one program into two programs on a wire. That is a distributed system.
