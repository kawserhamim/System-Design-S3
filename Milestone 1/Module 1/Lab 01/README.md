# Requirements Teardown

Welcome to the **first lab** of System Design and Backend Engineering with AWS! In this lab, you'll learn the **first step of building any software**: turning a vague idea into a clear one-page plan.

## Objective

By the end, you will turn a vague prompt into a one-page `requirements.md` by:

- Pulling **functional requirements** out of the prompt.
- Cutting **scope** (In / Out / Stretch).
- Turning vague goals into **measurable NFRs**.
- Writing a doc that a new engineer can build from.

## Let's Understand the Problem

Imagine your manager says: _"Build me a URL shortener like bit.ly."_

That's it. No more info. What do you do first?

If you start coding right away, you will get stuck. Why?

- You don't know what features matter most.
- You don't know what speed the system needs.
- You don't know what to skip for now.

You need a **plan before code**. That plan is called a **Requirements Document**.

## What is Requirements Engineering?

![Requirements engineering flow: vague prompt branches into functional requirements, non-functional requirements, and scope cuts, all of which feed a requirements doc that becomes a design brief](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2001/images/requirements-engineering-flow.gif)

**Requirements engineering** is the process of making that plan. We break a vague idea into 4 parts:

1. **Functional Requirements (FR):** **What** the system does. Actions users can take. Example: "User can paste a long URL."
2. **Non-Functional Requirements (NFR):** **How well** it does it. Speed, uptime, safety. Example: "Redirect happens in under 100 ms."
3. **Scope:** What we WILL build vs what we WILL NOT build.
4. **Open Questions:** Things we still need to ask.

Spend 1 hour planning now, and you'll save 10 hours fixing mistakes later. Good requirements are the cheapest bug you'll ever fix.

### What does a good requirements doc look like?

![Good Requirement Doc](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2001/images/good%20requirement%20doc.png)

A good requirements doc has **3 rules**:

1. **Testable:** Use real numbers you can measure (like "100 ms") instead of vague words like "fast."
2. **Scoped:** Clearly write what you will build AND what you won't build.
3. **Driver-aware:** Let your top 3 needs decide the design (cache for speed, multi-region for uptime, etc.).


## Step 1: Read the Vague Prompt

Make a folder and an empty file:

```bash
mkdir -p url-shortener && cd url-shortener
touch requirements.md
```

Open `requirements.md` and paste this at the top under `# Problem`:

```markdown
# Problem

Design a URL shortener like bit.ly. Users paste a long URL and get back a short code; visiting the short code redirects to the original URL.
```

That is the **only input** you get.

---

## Step 2: Extract Functional Requirements

A **functional requirement** is **one action** the system does. Like "shorten a URL" or "redirect a user."

Each FR must:

- Be **one short sentence**.
- Start with _"the system shall..."_ or _"users can..."_.
- Be **testable on its own**.

### Example: URL Shortener

Paste this into `requirements.md` under `## Functional Requirements`:

| ID   | Requirement                                                             | How to Test It                                                  |
| ---- | ----------------------------------------------------------------------- | --------------------------------------------------------------- |
| FR-1 | The system shall accept a long URL and return a 7-character short code. | Send `POST /shorten` with a URL. You get back a 7-char code.    |
| FR-2 | The system shall redirect a short code to the original URL.             | Visit `GET /:code`. You get a `302` redirect.                   |
| FR-3 | The system shall return `404 Not Found` for an unknown short code.      | Visit a code that doesn't exist. You get `404` in 10 ms.        |
| FR-4 | The system shall accept an optional custom alias and reject duplicates. | Try a taken alias. You get `409 Conflict`.                      |
| FR-5 | The system shall record a click (timestamp + user-agent) per redirect.  | After a redirect, the click shows in analytics within 1 second. |

**Why this format?** One sentence per row = no confusion. The "How to Test" column makes every FR checkable.

---

## Step 3: Cut Scope Clearly

**Scope** means: what will the system do?

A 20-feature system takes longer to build than a 4-feature one. And ships with more bugs.

![3 Scope Bucket](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2001/images/3%20scope%20bucket.png)

Every feature goes into **one of three buckets**:

- **In:** This is part of the first version. The main thing users do.
- **Out (v1):** Skip it for now. We can add it later in version 2.
- **Stretch:** Build this only AFTER all the "In" items are finished.



### Example Scope Table

Add this to `requirements.md` under `## Scope`:

| Feature                       | Status       | Why This Decision                                             |
| ----------------------------- | ------------ | ------------------------------------------------------------- |
| User accounts / login         | **Out (v1)** | Login is not needed to shorten a URL. Revisit in v2.          |
| Custom domains for short URLs | **Out (v1)** | Needs DNS setup. Too much work for v1.                        |
| Link expiry / TTL             | **Out (v1)** | Needs a background cleanup job. Defer until asked.            |
| QR-code generation            | **Out (v1)** | Only presentation. Can be done client-side later.             |
| Spam / abuse detection        | **Out (v1)** | A full classifier is its own product. Add a denylist in v1.1. |
| Bulk import via CSV           | **Stretch**  | Useful, but the API already handles bulk via repeat calls.    |

### Simple Rules

- **One reason per row.** You'll re-read it when planning v2.
- **Aim for 5 to 8 features.** Fewer than 5? You missed some. More than 8? You're collecting instead of cutting.
- **When in doubt, cut.** Adding later is easier than removing.

---

## Step 4: Pick 3 Non-Functional Drivers

NFRs shape the architecture of your app. For first version of your app(v1), pick **exactly 3**. More than 3 and you can't choose when they clash.

![A good NFR](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2001/images/A%20good%20NFR.png)

A good NFR has **4 parts**:

1. **Metric** - what you measure
2. **Target number** - what value you want
3. **Load condition** - how much traffic
4. **Measurement method** - how you check it

### Example NFRs for the URL Shortener

Add this to `requirements.md` under `## Non-Functional Requirements`:

### NFR-1: Redirect Latency

| Field              | Value                                                            |
| ------------------ | ---------------------------------------------------------------- |
| Metric             | How long does a redirect take?                                   |
| Target number      | p99 less than **100 ms**                                          |
| Load condition     | Sustained **10,000 redirects/sec**                                |
| Measurement method | Histogram on `GET /:code`; alert if p99 exceeds 100 ms for 5 min |

### NFR-2: Availability

| Field              | Value                                                  |
| ------------------ | ------------------------------------------------------ |
| Metric             | Is the redirect endpoint up?                           |
| Target number      | **99.9%** (about 43 min downtime per month)            |
| Load condition     | Same 10,000 redirects/sec                              |
| Measurement method | Synthetic check from 3 regions every 30 seconds        |

### NFR-3: Storage Durability

| Field              | Value                                                               |
| ------------------ | ------------------------------------------------------------------- |
| Metric             | Will we lose any stored short-URL mapping?                           |
| Target number      | **99.999999999%** (eleven nines, S3 standard)                        |
| Load condition     | 100 writes/sec sustained                                            |
| Measurement method | Quarterly disaster recovery drill; restore backup and confirm checksum |

**Why these 3?** Redirects are the main action, so latency and availability matter most. Storage durability stops data loss over time. Auth, throughput, cost come later.

**Important:** Always include the **load condition**. "p99 < 100 ms" alone is meaningless. At 10 QPS or 10k QPS, those are different systems.

---

## Step 5: Write the Requirements Doc

Pull it all together on **one page**. The whole file should fit on **one printed page**. If it doesn't fit, you haven't cut enough. Here is the example of the structure.

![Skeleton](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2001/images/skeleton.png)

### Full Example: URL Shortener `requirements.md`

```markdown
# URL Shortener: Requirements

## Problem

Design a URL shortener like bit.ly. Users paste a long URL and get back a short
code; visiting the short code redirects to the original URL.

## Functional Requirements

| ID   | Requirement                                                             | How to Test It                                               |
| ---- | ----------------------------------------------------------------------- | ------------------------------------------------------------ |
| FR-1 | The system shall accept a long URL and return a 7-character short code. | Send `POST /shorten` and verify that the response has 7 chars. |
| FR-2 | The system shall redirect a short code to the original URL.             | Visit `GET /:code` and verify that it returns a `302` redirect. |
| FR-3 | The system shall return `404 Not Found` for an unknown short code.      | Request a nonexistent code and verify the `404` response.    |
| FR-4 | Users can provide an optional custom alias.                             | Create a link with an available alias and verify it resolves. |
| FR-5 | The system shall reject duplicate custom aliases.                      | Submit an existing alias and verify a `409 Conflict` response. |
| FR-6 | The system shall record a click for every successful redirect.         | Follow a short link and verify analytics updates within 1 second. |

## Scope

| Feature                       | Status       | Why This Decision                                      |
| ----------------------------- | ------------ | ------------------------------------------------------ |
| Shorten a URL                 | **In**       | This is the system's primary function.                 |
| Redirect short links          | **In**       | Users must be able to reach the original URL.          |
| Click analytics               | **In**       | Basic usage data is useful for link owners.            |
| Custom aliases                | **In**       | Users can create memorable short links.                |
| User accounts / login         | **Out (v1)** | Login is not required for the core shortening flow.    |
| Custom domains                | **Out (v1)** | This requires extra DNS and certificate management.    |
| Link expiry                   | **Out (v1)** | This requires expiration and cleanup logic.            |
| QR-code generation            | **Out (v1)** | It can be added later without changing the core system. |
| Spam detection               | **Out (v1)** | A complete abuse-detection system is beyond v1 scope.  |
| Bulk CSV import               | **Stretch**  | Build it only after all core features are complete.    |

## Non-Functional Requirements

### NFR-1: Redirect Latency

| Field    | Value                                                       |
| -------- | ----------------------------------------------------------- |
| Metric   | How long does a redirect take?                              |
| Target   | p99 < 100 ms                                                |
| Load     | Sustained 10,000 redirects/sec                              |
| How to measure | Histogram on `GET /:code`; alert if p99 > 100 ms for 5 min |

### NFR-2: Availability

| Field    | Value                                                       |
| -------- | ----------------------------------------------------------- |
| Metric   | Is the redirect endpoint up?                                |
| Target   | 99.9% (about 43 min downtime per month)                     |
| Load     | Same 10,000 redirects/sec                                   |
| How to measure | Synthetic check from 3 regions every 30 seconds        |

### NFR-3: Storage Durability

| Field    | Value                                                       |
| -------- | ----------------------------------------------------------- |
| Metric   | Will we lose any stored short-URL mapping?                  |
| Target   | 99.999999999% (eleven nines, S3 standard)                   |
| Load     | 100 writes/sec sustained                                    |
| How to measure | Quarterly disaster recovery drill: restore backup and confirm checksum |

## Open Questions

- Are short codes case-sensitive, or do we force one case on read?
- Max length of input URL, and how is it validated?
- Should the redirect be 301 (cached) or 302 (not cached)?
- How long do we keep click events?
- Rate limit: per-IP, per-API-key, or global?
- Policy on deleting a short code: 410 Gone or keep working?
```

**Why this shape?** One screen = about 60 to 80 lines of Markdown. Always include Open Questions. Even a good plan misses things.

---

## Step 6: Verification Checklist

Run this checklist on your `requirements.md`. If any box is "no", go back and fix it before moving on.

![Checkbox](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Lab%2001/images/Checkbox.png)

**Don't move on until every box is checked.**

---

## Conclusion

In this lab you split **FRs** from **NFRs**, cut **scope** into In / Out / Stretch, and made vague goals **measurable**. The end result: a one-page `requirements.md` any engineer can build from.
