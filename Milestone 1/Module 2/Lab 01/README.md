# RPC vs. REST in Practice

In this lab we will understand the difference between REST and gRPC. We will build the same operation two ways: once as a REST API and once as a gRPC service. Then we will compare how big each message is on the wire and how long each round-trip takes.

![REST vs gRPC side-by-side comparison](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/Rest-vs-grpc.png)

## Objective

By the end of this lab you will be able to:
1. Understand rest and grpc
1. Implement the **same** `GetUser` as REST and gRPC.who
2. Benchmark both with `wrk` and `ghz` under the same load.
3. Compare **wire-format size** with `measure-size.js`.

### What is REST?

REST is an API architectural style that uses HTTP to communicate with resources, commonly exchanging data in JSON format (also XML, plain text, HTML, or others).

It is simple and human-readable, but JSON messages are typically larger than binary formats like Protobuf.

![REST request-response flow over HTTP/JSON](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/rest.png)

### What is gRPC?

gRPC is a framework that lets one service call a function on another service over a network, like calling a local function.

It uses **Protobuf**, a compact binary format that makes messages smaller and faster than JSON. It commonly runs over **HTTP/2** (the newer HTTP version that lets many requests share one connection, which speeds things up under traffic).

![gRPC request-response flow over HTTP/2 with Protobuf](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/grpc.png)

In short, **REST = readable JSON over HTTP and gRPC = compact binary over HTTP/2.** Both can return the same data; we will measure which one is smaller and faster.

## 1. File layout

Let's create the file layout and copy each content from the following file:

```text
m2-l1/
├── package.json
├── proto/
│   └── user.proto
├── rest-server.js
├── grpc-server.js
└── measure-size.js
```

## 2. File contents

Create the directories and copy each block below into the named file.

### 2.1 `package.json`

```json
{
  "name": "rpc-vs-rest",
  "version": "1.0.0",
  "private": true,
  "description": "Lab 1 - RPC vs REST comparison",
  "scripts": {
    "rest": "node rest-server.js",
    "grpc": "node grpc-server.js",
    "size": "node measure-size.js"
  },
  "dependencies": {
    "@grpc/grpc-js": "^1.10.0",
    "@grpc/proto-loader": "^0.7.10",
    "express": "^4.19.2",
    "protobufjs": "^7.3.0"
  }
}
```

**What this file does:**
- Declares project name and dependencies (`express` for REST, `@grpc/*` for gRPC, `protobufjs` for size measurement)
- Defines three npm scripts: `rest`, `grpc`, `size`. These are short aliases for the long `node <file>.js` commands

### 2.2 `proto/user.proto`

```proto
syntax = "proto3";

package user;

service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
}

// ---- Request ----------------------------------------------------------
message UserRequest {
  string user_id = 1;
}

// ---- Response (mirror shape of the REST JSON exactly) -----------------
message UserResponse {
  string   user_id    = 1;
  string   name       = 2;
  string   email      = 3;
  string   created_at = 4;          // ISO-8601 string
  Preferences preferences = 5;      // nested message -> nested object
  repeated string tags = 6;         // repeated -> JSON array
}

message Preferences {
  string theme        = 1;
  string language     = 2;
  bool   email_opt_in = 3;
  int32  page_size    = 4;
}
```

**What this file does:**
- Defines the **gRPC service contract**: one RPC method `GetUser` taking a `UserRequest` and returning a `UserResponse`
- `UserResponse` mirrors the JSON shape from REST exactly: same fields, same nesting, so payloads compare 1:1
- `proto3` syntax means default values are omitted on the wire

### 2.3 `rest-server.js`

```javascript
// REST/JSON server. Same payload shape as the gRPC UserResponse.
// Port: 3000

const express = require("express");

const app = express();
const PORT = 3000;

// Canonical user record - keep field values IDENTICAL to grpc-server.js
// so payloads compare 1:1 across protocols.
const USERS = {
  "u-42": {
    user_id: "u-42",
    name: "Anika Rahman",
    email: "anika@example.com",
    created_at: "2024-03-15T08:42:11Z",
    preferences: {
      theme: "dark",
      language: "en-US",
      email_opt_in: true,
      page_size: 25,
    },
    tags: ["beta-tester", "newsletter", "power-user", "founding-member"],
  },
};

app.get("/users/:id", (req, res) => {
  const user = USERS[req.params.id];
  if (!user) return res.status(404).json({ error: "not_found" });
  res.json(user);
});

app.listen(PORT, () => {
  console.log(`REST listening on http://127.0.0.1:${PORT}`);
  console.log(`Try: curl http://127.0.0.1:${PORT}/users/u-42`);
});
```

**What this file does:**
- Spins up an Express HTTP server on port **3000** exposing `GET /users/:id`
- Returns the same hardcoded `u-42` record that the gRPC server returns: fields and values kept identical for a fair comparison
- Returns HTTP 404 with `{ error: "not_found" }` if the user ID is missing

### 2.4 `grpc-server.js`

```javascript
// gRPC server. Same canonical user record as rest-server.js.
// Port: 50051

const path = require("path");
const grpc = require("@grpc/grpc-js");
const protoLoader = require("@grpc/proto-loader");

const PROTO_PATH = path.join(__dirname, "proto", "user.proto");

const packageDef = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
});
const proto = grpc.loadPackageDefinition(packageDef).user;

// Canonical record - same field values as rest-server.js
const USERS = {
  "u-42": {
    user_id: "u-42",
    name: "Anika Rahman",
    email: "anika@example.com",
    created_at: "2024-03-15T08:42:11Z",
    preferences: {
      theme: "dark",
      language: "en-US",
      email_opt_in: true,
      page_size: 25,
    },
    tags: ["beta-tester", "newsletter", "power-user", "founding-member"],
  },
};

function getUser(call, callback) {
  const u = USERS[call.request.user_id];
  if (!u) {
    return callback({
      code: grpc.status.NOT_FOUND,
      message: "user not found",
    });
  }
  callback(null, u);
}

const server = new grpc.Server();
server.addService(proto.UserService.service, { GetUser: getUser });

server.bindAsync(
  "127.0.0.1:50051",
  grpc.ServerCredentials.createInsecure(),
  (err, port) => {
    if (err) return console.error("bindAsync failed:", err);
    console.log(`gRPC listening on 127.0.0.1:${port}`);
    console.log(
      `Try: grpcurl -plaintext -import-path ./proto -proto user.proto ` +
        `-d '{"user_id":"u-42"}' 127.0.0.1:${port} user.UserService/GetUser`,
    );
  },
);
```

**What this file does:**
- Loads `user.proto`, registers the `UserService.GetUser` method, and binds it to port **50051** (HTTP/2, insecure credentials)
- Uses `protoLoader` with `keepCase: true` so snake_case field names are preserved end-to-end
- Returns `grpc.status.NOT_FOUND` for unknown IDs, mirroring REST's 404 behavior

### 2.5 `measure-size.js`

```javascript
// Measure the BYTE LENGTH of one request and one response on each side.
// Uses the SAME field values that the two real servers send.
// gRPC bytes reported = protobuf body bytes + 5 bytes gRPC framing.

const path = require("path");
const protobuf = require("protobufjs");

function row(label, value, unit) {
  console.log(`  ${label.padEnd(28)} ${String(value).padStart(10)} ${unit}`);
}

const PROTO_PATH = path.join(__dirname, "proto", "user.proto");
const root = protobuf.loadSync(PROTO_PATH);

function make(typeName) {
  const T = root.lookupType(typeName);
  return (obj) => T.encode(T.fromObject(obj)).finish();
}

const encodeUserRequest = make("user.UserRequest");
const encodeUserResponse = make("user.UserResponse");

const req = { user_id: "u-42" };
const resp = {
  user_id: "u-42",
  name: "Anika Rahman",
  email: "anika@example.com",
  created_at: "2024-03-15T08:42:11Z",
  preferences: {
    theme: "dark",
    language: "en-US",
    email_opt_in: true,
    page_size: 25,
  },
  tags: ["beta-tester", "newsletter", "power-user", "founding-member"],
};

const reqJson = JSON.stringify(req);
const respJson = JSON.stringify(resp);
const restReqBytes = Buffer.byteLength(reqJson, "utf8");
const restRespBytes = Buffer.byteLength(respJson, "utf8");

// gRPC framing: 1 byte compressed flag + varint length prefix.
// Simplified to 5 bytes here only because the payload is small and uncompressed.
const GRPC_FRAMING_BYTES = 5;
const grpcReqBuf = encodeUserRequest(req);
const grpcRespBuf = encodeUserResponse(resp);
const grpcReqBytes = grpcReqBuf.length + GRPC_FRAMING_BYTES;
const grpcRespBytes = grpcRespBuf.length + GRPC_FRAMING_BYTES;

console.log("\n== Wire payload size (one request + one response) ==\n");
console.log("-- REST  (HTTP/JSON body bytes only) --");
row("request  JSON", Buffer.byteLength(reqJson, "utf8"), "bytes");
row("response JSON", Buffer.byteLength(respJson, "utf8"), "bytes");
row("TOTAL   bytes sent+recv", restReqBytes + restRespBytes, "bytes");

console.log("\n-- gRPC  (protobuf body + 5-byte gRPC framing) --");
row("request  protobuf body", grpcReqBuf.length, "bytes");
row("response protobuf body", grpcRespBuf.length, "bytes");
row("request  + framing", grpcReqBytes, "bytes");
row("response + framing", grpcRespBytes, "bytes");
row("TOTAL   bytes sent+recv", grpcReqBytes + grpcRespBytes, "bytes");

console.log("\n-- Saved by gRPC over REST --");
const saved =
  1 - (grpcReqBytes + grpcRespBytes) / (restReqBytes + restRespBytes);
row("saved percent", (saved * 100).toFixed(2) + " %", "");
console.log();
```

**What this file does:**
- Encodes the same payload using both formats: `JSON.stringify` for REST and `protobufjs` for gRPC
- Measures raw body bytes on each side (REST) and protobuf body + a simplified 5-byte gRPC framing constant (gRPC)
- Prints a side-by-side table plus the percent of bytes saved by gRPC, used to populate the comparison table in Section 5

## 3. Run the following command into the terminal

### 3.1 Install dependencies

```bash
sudo apt update
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs wrk
curl -fsSL -o /tmp/ghz.tar.gz https://github.com/bojand/ghz/releases/download/v0.121.0/ghz-linux-x86_64.tar.gz
tar -xzf /tmp/ghz.tar.gz -C /tmp ghz
sudo mv /tmp/ghz /usr/local/bin/ghz
ghz --version
mkdir -p ~/code/proto && cd ~/code
npm install
```

Verify each tool is installed:

```bash
node -v
npm -v
wrk --version
ghz --version
```

### 3.2 Start the two servers

#### Terminal A - REST

```bash
node rest-server.js
```

Expected output:

![REST server console output showing it listening on port 3000](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/rest-server.png)

#### Terminal B - gRPC

```bash
node grpc-server.js
```

Expected output:

![gRPC server console output showing it listening on port 50051](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/grpc-server.png)

### 3.3 Load tests (run in Terminal C)

**Quick tool primer:**
- `wrk -t4 -c64 -d10s <url>` runs `wrk` with 4 threads, 64 concurrent connections, for 10 seconds
- `ghz -c 64 -n 20000 -z 10s <addr>` runs `ghz` with 64 concurrent calls, up to 20,000 requests, capped at 10 seconds
- `p50` = median latency (50% of requests are faster than this)
- `p99` = 99th-percentile latency (only 1% of requests are slower; the most useful tail-latency number for SLOs)

Run the REST server load test using `wrk`:

```bash
wrk -t4 -c64 -d10s http://127.0.0.1:3000/users/u-42
```

You will see:

![wrk output for REST: latency avg, requests/sec, transfer rate](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/first-cmnd.png)

**What to expect:**

- **Thread latency**: avg, stdev, and max per thread (e.g. `41.51ms / 106.46ms / 1.28s`)
- **Total requests served**: ~26k in our 10s run
- **Throughput**: `Requests/sec: 2637`
- **Watch the `Latency Max` spike**: `1.28s` is the outlier that drags REST's p99 so far above its average

Now run the gRPC test under `ghz` (run as a single line, do not split it with backslashes):

```bash
ghz --insecure --proto ./proto/user.proto --call user.UserService/GetUser -d '{"user_id":"u-42"}' -c 64 -n 20000 -z 10s --format summary 127.0.0.1:50051
```

You will see:

![ghz summary: count, total, slowest, fastest, average, requests/sec](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/ghz.png)

**What to expect:**

- **Count**: total requests served (~34k in our run)
- **Total**: how long the test ran (10s)
- **Slowest**: the single slowest response (~135 ms in our run)
- **Fastest**: the single fastest response (~0.66 ms)
- **Average**: mean latency (~15 ms)
- **Requests/sec**: throughput → 3458 

![ghz response time histogram](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/histogram.png)

**What to expect:**

- **X-axis**: response time buckets in milliseconds (from ~0.6 ms up to ~135 ms)
- **Y-axis**: count of requests that fell into each bucket
- **Tallest bar (~14 ms)**: where the bulk of requests landed (~19.5k in our run)
- **Long thin tail past 100 ms**: latency outliers (only dozens of requests each)
- **Why it matters**: that small tail is exactly what makes the **p99** latency so much higher than the average

![ghz latency distribution by percentile](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/latency-distri.png)

**What to expect:**

- **Percentile lines**: `10%, 25%, 50%, 75%, 90%, 95%, 99%`. Each line says "X% of requests finished in N ms or less".
- **Example from our run**: `50 % in 13.18 ms` for gRPC means half of all requests finished in 13.18 ms or less
- **Status codes below**: 34569 `[OK]` responses + 28 `[Unavailable]` at the tail → the `[Unavailable]` ones are normal test-end disconnects, not real failures

Wire-size comparison:

```bash
node measure-size.js
```

You will see this result:

![measure-size.js output comparing REST vs gRPC byte counts](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module2/Lab%2001/images/mesure.png)

**What to expect:** A side-by-side byte-count report for REST (JSON body) and gRPC (protobuf body + 5-byte framing). Look for:
- REST **Total = 274 bytes** (18 req + 256 resp)
- gRPC **Total = 112 bytes** (5 req + 107 resp)
- A final `saved percent` line: in our run, gRPC saved **59.12 %** of the wire bytes

These numbers feed directly into the "Request payload", "Response payload", and "Total on the wire" rows of the comparison table below. Note that gRPC's *request body* is **0 bytes**: proto3 omits fields set to their default value, so an empty `UserRequest` is valid on the wire.


## 4. Comparison table: from test result

| Metric                       | REST (HTTP/JSON) | gRPC (protobuf) |
| ---------------------------- | ---------------- | --------------- |
| Request payload (bytes)      | 18               | 0 + 5 framing = 5 |
| Response payload (bytes)     | 256              | 102 + 5 framing = 107 |
| Total on the wire (req+resp) | 274              | 112             |
| Latency p50                  | 19.97 ms         | 13.18 ms        |
| Latency p99                  | 469.03 ms        | 54.88 ms        |
| Throughput (req/s, achieved) | 2955.20          | 3458.81         |


## 5. Recommendation note

For this workload, **gRPC is preferred** because:

- **59.12% smaller** wire payload (112 bytes vs 274 bytes)
- **~17% higher** throughput (3458 vs 2955 req/s)
- **8.5x tighter** p99 latency (55 ms vs 469 ms)

Switch back to REST when the client is a browser, a third-party tool that cannot speak gRPC, or a human who needs to read the JSON directly.

## 6. Conclusion

For service-to-service traffic on this workload, gRPC is the best choice: smaller payloads, higher throughput, and far tighter tail latency than REST. Choose REST only when a browser, third-party tool, or human reader needs direct access to the JSON payload.
