# System Brief: URL Shortener

This brief pulls together the requirements doc, Observed Failure Mode, and the system spec into one document so that any new engineer can read and start building from it. The system turns a long URL into a short code, and redirects anyone who clicks that code to the original URL.

![System Brief Overview](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2004/images/thumbnail.png)

Create markdown file named `system-brief.md` and add all contents: 

## 1. Requirements (from Lab 1)

**Problem:** Build a URL shortener like bit.ly. A user pastes a long URL, gets back a short code. Visiting the short code returns a redirect to the long URL.

**Functional Requirements:**
- FR-1: Accept a long URL and return a 7-character short code.
- FR-2: Redirect a short code to the original URL (302).
- FR-3: Return 404 for an unknown short code.
- FR-4: Accept an optional custom alias.
- FR-5: Reject duplicate custom aliases (409).
- FR-6: Record a click (timestamp + user-agent) per redirect.

**Non-Functional Requirements (the 3 drivers):**
| NFR | Metric | Target | Load |
| --- | --- | --- | --- |
| Latency | Redirect time | p99 < 100 ms | 10,000 redirects/sec |
| Availability | Endpoint uptime | 99.9% | Same load |
| Durability | Link data loss | 99.999999999% | 100 writes/sec |

**Scope:**
- **In:** shorten, redirect, custom aliases, basic clicks.
- **Out (v1):** login, custom domains, expiry, QR codes, spam detection.
- **Stretch:** bulk CSV import.

## 2. Observed Failure Mode (from Lab 2)

In the network test, two Node.js processes talked over TCP (one client, one server). The failure we observed was **partial failure**: the server was killed with SIGKILL while the client was still alive. The client saw the socket close, logged `line closed`, and exited cleanly. We also measured latency: RTT was ~1-3 ms on, and ~200 ms when we added a 200 ms delay on the server side.

**What this means for our design:**
- The network tells us if the other side is alive. We cannot assume the database or app server is reachable at all times.
- Latency is real and must be budgeted (NFR-1: p99 < 100 ms).
- The app server must stay stateless. If one crashes mid-request, the next request should just go to another app server and read the same database.

## 3. System Spec (from Lab 3)

**Components (4 boxes):**

Client -> Load Balancer -> App Server -> Database. The database holds all state; everything else is stateless.

![System Components](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2004/images/components.png)

**Data Flow:**

- *Shorten (write):* Client -> Load Balancer -> App Server -> Database (write `{code, long_url}`) -> 201 with short URL.
- *Redirect (read):* Client -> Load Balancer -> App Server -> Database (lookup by code) -> 302 with `Location: long_url`.

![System Data Flow](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2004/images/flows.png)

**APIs:**
- `POST /api/shorten` -> 201 returns `{short_url, code}`; 400 on bad input; 409 if alias taken.
- `GET /:code` -> 302 to long URL, or 404.
- `GET /api/links/:code` -> metadata (stretch).
- `GET /api/links/:code/analytics` -> click data (stretch).

**Data Model (one table):**

![Links table schema](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2004/images/table.png)


**Key Design Decisions:**
- Short code: 7 random base62 chars (~3.5 trillion options).
- App server: stateless, any server can handle any request.
- Database: single table, all state lives here.
- Custom aliases: optional, can collide (409 on duplicate).
- `created_at`: set by the database clock, not the client.
- Redirect: 302 (not 301), so we can add analytics later.
- v1 has no delete and no auth.

## Conclusion
This brief combines all the requirements: it lists all the what (FRs, NFRs, a network failure we observed) and the how (4 components, 2 main APIs, one table) answers of the system. It is small enough to read in five minutes and start coding. Anything not listed here is out of scope for v1.
