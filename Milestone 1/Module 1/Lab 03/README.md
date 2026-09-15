# Whiteboard to Spec

In this lab we will translate a hand-drawn architecture into a structured, reviewable system spec (into components, data flow, APIs).

![Whiteboard idea becomes a clean system spec, then code](
https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/whiteboard-to-spec-flow.png)

**Objective:**

- Translate a whiteboard architecture into a clean, structured, reviewable spec.
- Document the system's components, data flow, and APIs in plain text.
- Provide enough detail so a new engineer can review it and draw the architecture from memory.


## 1. What is a System Spec?

A **system spec** is a written description of how a system works. Think of it as the **blueprint** for the system. Engineers use it to agree on what to build before writing code.

![Whiteboard idea becomes a clean system spec, then code](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/system-specs.png)

It answers three questions:

1. **What are the parts?** (components)
2. **How do they talk?** (data flow)
3. **What can users call?** (APIs)

A good spec is **short, clear, and testable**. If a new engineer can read it and know what to build, the spec works.


## 2. The System: URL Shortener

We will spec a **URL Shortener** (same family of system as `bit.ly`, `tinyurl.com`, and `t.co`)

**How it works:** A URL shortener takes a long URL (e.g. `https://example.com/some/very/long/path?with=query`) and gives back a short code (e.g. `aB3xd9z`). Visiting `https://shortener.com/aB3xd9z` redirects the browser to the original long URL.

### Why a URL Shortener?

- Has **one write path** (`POST /shorten`) and **one read path** (`GET /:code`).
- Easy to draw on a whiteboard with 4 boxes.
- Lets us cover **load balancer, app server, database, redirect** in a tiny system.
- Familiar - every web user has clicked a bit.ly link.


## 3. Big Picture (Whiteboard View)

Imagine drawing this on a whiteboard with 4 boxes and arrows:

![URL shortener architecture: Client → Load Balancer → App Server → Database, with reverse arrows for responses](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/url-shortener-architecture.png)

Four **components**, three **arrows** between them. That is the whole system. Below we describe each box in plain words.


## 4. Components

A **component** is one piece of the system that does one job. A URL shortener has 4 components.

### Component 1: Client (Browser / Phone)

![Whiteboard idea becomes a clean system spec, then code](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/component-1.png)

*Actions:* paste a long URL, click "Shorten", open a short link, follow the 302 to the original site.

### Component 2: Load Balancer

![Whiteboard idea becomes a clean system spec, then code](
https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/component-2.png)

The front door. Every request hits this first. It terminates TLS, picks an app server, and forwards the request. For our v1 the load balancer mostly **forwards requests** to one app server. When we scale to many app servers, the load balancer picks which one should handle each request.

### Component 3: App Server (Backend)

The brain. **Stateless**. Holds the business logic.

![Whiteboard idea becomes a clean system spec, then code](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/component-3.png)

- On `POST /api/shorten`: validate the URL. Use the `custom_alias` if provided, otherwise generate a random 7-char code. Save `{code, long_url}` to the database, return the short URL.
- On `GET /:code`: look up the code in the database. If found, return a `302` with the `Location` header pointing at `long_url`. If not found, return `404`.

The app server is **stateless** - it holds no memory between requests. Every link lives in the database. We can run 1 or 100 app servers and they all behave the same.

> **Where is the state?** The **Database** holds all state. Every `POST /api/shorten` writes `{code, long_url}` to the DB. Every `GET /:code` reads from the DB. The app server itself keeps nothing in memory between requests - it just reads and writes the database. If app server #1 crashes mid-request, app server #2 takes the next request, reads the same DB, and returns the same answer.

### Component 4: Database

![Whiteboard idea becomes a clean system spec, then code](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/component-4.png)

Where the `short_code → long_url` mappings live on disk. Survives restarts. We look up by code, so `code` is the **primary key** and `long_url` is the value we redirect to.


## 5. Data Flow (How a Request Travels)

A **data flow** is the path a request takes from the client back to the client. Let's trace two flows.

### Flow A: Shorten a URL (Write)

![Shorten URL flow: Client → Load Balancer → App Server → Database, returning 201 Created with the short URL](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/shorten-flow-sequence.png.png)

### Flow B: Visit a Short Link (Redirect / Read)

![Redirect flow: Client → Load Balancer → App Server → Database, returning 302 Found with Location header that the browser follows to the long URL](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/redirect-flow-sequence.png.png)

Two requests. The first is short and fast. The browser does the second on its own - our server never touches it.


## 6. APIs (The Doors Into the System)

An **API** is one named call the client can make. Our URL shortener has **2 core APIs** (shorten and redirect) plus **2 stretch APIs** (metadata and analytics) for later labs.

### API 1: Shorten a URL

```
  POST /api/shorten
  Body:  { "long_url": "https://example.com/very/long/path", "custom_alias": "my-link" }
  Note:  custom_alias is optional. If provided, it becomes the `code` in the database.
  Auth:  none (public)
```

The `custom_alias` field is optional. If omitted, the server generates a random 7-char code. **Either way, the `code` stored in the database is the value the user visits** - so a custom alias like `"my-link"` is just a code chosen by the user instead of by the server.

**Response on success (201 Created):**

```json
{
  "short_url": "https://shortener.com/my-link",
  "code": "my-link",
  "long_url": "https://example.com/very/long/path",
  "created_at": "2026-08-16T10:30:00Z"
}
```

*If the request omits `custom_alias`, the server generates a random code (e.g. `aB3xd9z`) and the response shows that instead:*

```json
{
  "short_url": "https://shortener.com/aB3xd9z",
  "code": "aB3xd9z",
  "long_url": "https://example.com/very/long/path",
  "created_at": "2026-08-16T10:30:00Z"
}
```

**Response on bad input (400 Bad Request):**

```json
{ "error": "long_url is required" }
```

**Response if custom alias already exists (409 Conflict):**

```json
{ "error": "alias 'my-link' is already taken" }
```

### API 2: Visit a Short Link (the redirect)

```
  GET /:code
  Path:  :code is the 7-char short code
  Auth:  none
```

This is the **hot path**. Every click on any short link in the world hits this endpoint.

**Response on success (302 Found):**

```
  HTTP/1.1 302 Found
  Location: https://example.com/very/long/path
```

The browser receives the 302, sees the `Location` header, and follows it. Our server is done.

**Response when code does not exist (404 Not Found):**

```json
{ "error": "link not found" }
```

### API 3: Get Link Metadata (Stretch)

```
  GET /api/links/:code
```

Returns the link record without following it. Useful for dashboards.

**Response on success (200 OK):**

```json
{
  "code": "aB3xd9z",
  "short_url": "https://shortener.com/aB3xd9z",
  "long_url": "https://example.com/very/long/path",
  "created_at": "2026-08-16T10:30:00Z",
  "clicks": 142
}
```

**Response when code does not exist (404 Not Found):**

```json
{ "error": "link not found" }
```

### API 4: Click Analytics (Stretch)

```
  GET /api/links/:code/analytics
```

Returns click counts and timestamps. Out of scope for v1, listed here so later labs know where it fits.


## 7. Data Model (What the Database Stores)

One table. Three columns. That is the whole data model.

The columns are:
- `code` (VARCHAR(32) PRIMARY KEY) - the short code. Auto-generated is 7 chars; custom aliases can be up to 32 chars.
- `long_url` (TEXT NOT NULL) - the original long URL.
- `created_at` (TIMESTAMP NOT NULL) - when the link was created, set by the database.

![Links table schema: code (VARCHAR(32) PRIMARY KEY), long_url (TEXT NOT NULL), created_at (TIMESTAMP NOT NULL)](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/links-schema-table.png)

**Why `code` is primary key:** We always look up links **by code**. So it must be unique and indexed.

**Note on `custom_alias`:** A custom alias is just a `code` chosen by the user instead of randomly generated by the server. It is stored in the same `code` column - no separate column needed.

**Why `created_at` is set by DB, not by client:** Clients can lie. The DB clock is the source of truth.

**Example rows:**

![Example rows in the links table: aB3xd9z, zX7q1Lm, K9pQ2wX with their long URLs and created_at timestamps](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2003/images/links-table-rows-example.png)


## 8. Key Design Decisions

These are the choices that shaped the spec. Each one has a **why**.

| #   | Decision                                 | Why                                                                                                                                          |
| --- | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Short code is **7 random base62 chars**  | `62^7 ≈ 3.5 trillion` possible codes. Plenty for v1. Easy to type, hard to guess.                                                            |
| 2   | App Server is **stateless**              | Any app server can handle any request. We can run 1 or 100. Crash + retry works. No sticky sessions.                                         |
| 3   | Single table in DB                       | Reads and writes are simple key lookups. No joins needed.                                                                                    |
| 4   | Load Balancer is **separate from app**   | Lets us scale app servers horizontally without changing the public URL.                                                                      |
| 5   | Redirect is **302 Found**, not 301       | **302** = browser re-checks next time. **301** = browser caches forever (we lose analytics; you cannot change the target later). v1 = 302.   |
| 6   | Custom aliases **optional**, can collide | Most users let the server pick the code. Power users want "bit.ly/welcome". If two users ask for the same alias, we return **409 Conflict**. |
| 7   | `created_at` set by DB, not client       | The DB clock is the source of truth. Clients cannot lie about timestamps.                                                                    |
| 8   | **No delete** in v1                      | Less code. We can add `DELETE /api/links/:code` later.                                                                                       |
| 9   | No auth in v1                            | Anonymous link creation is the core flow. Auth adds 10x scope. Revisit in v2.                                                                |




## 9. Out of Scope (v1)

These are **not** part of this spec. We will cut them.

- User login / accounts
- Custom domains for short URLs (`go.mycompany.com/x`)
- Link expiry / auto-delete after N days
- QR-code generation
- Password-protected short links
- Spam / abuse detection / CAPTCHA
- A/B testing the redirect target

Each item can become a new lab later. The spec stays small now.


## 10. The Spec on One Page(system.specs.md)

If we squeeze the spec into one page, future engineers only need to read this:

```
  SYSTEM:   URL Shortener
  USERS:    Anyone with a browser
  GOAL:     Long URL → short code. Click short code → redirect to long URL.

  COMPONENTS:
    Client (browser)
       ↓
    Load Balancer (TLS, routing)
       ↓
    App Server (stateless, business logic)
       ↓
    Database (single table: links)

  APIS:
    POST /api/shorten             → 201 returns {short_url, code}; 400/409 on bad input/alias taken
    GET  /:code                   → 302 to long_url, or 404
    GET  /api/links/:code         → metadata (stretch)
    GET  /api/links/:code/analytics → click data (stretch)

  CUSTOM ALIAS:
    Optional in POST body. If provided, becomes the `code` (409 if taken).

  DATA:
    Table links (code PK VARCHAR(32), long_url TEXT, created_at TIMESTAMP)

  REDIRECT:
    302 Found (so we can add analytics later)

  OUT OF SCOPE:
    Auth, custom domains, expiry, QR codes, passwords, spam
```

This is what downstream labs will render to real code.


## 11. How to Verify This Spec

Before you call it done, run this checklist:

- [ ] Can a new engineer draw the 4 boxes from memory after reading?
- [ ] Are all 4 APIs listed with request and response?
- [ ] Is the database schema written down with column types?
- [ ] Are data flows shown for at least shorten and redirect?
- [ ] Is **out of scope** explicit (so we do not build the wrong thing)?
- [ ] Did we say **which HTTP code** the redirect uses (302)?

If any box is unchecked, fix it before moving on.


## Conclusion
A good spec is one a new engineer can read once and start building from. If this document clears the checklist above, it is ready to render into real code.
