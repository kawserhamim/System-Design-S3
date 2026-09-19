---
name: poridhi-lab
description: Create a Poridhi-style hands-on lab README for System Design Season 3 (Milestone 1, Modules 1 and 2). Use when the user asks for a "Poridhi lab", a "System Design S3 lab", a lab for "Module X Lab YY", or wants to copy the style of the existing Poridhi labs (Requirements Teardown, Hello Distributed System, RPC vs REST, etc.). Generates one lab folder with README.md and an images/ folder. Enforces no em-dash, beginner English, visuals first, copy-pasteable code blocks, and tight length. Do NOT use for general tutorials, blog posts, or non-Poridhi formats.
---

# Poridhi Lab Skill

Use this skill to write a hands-on lab in the style of Poridhi Labs' System Design Season 3 (Milestone 1).

## When to use this skill

Invoke when the user asks for:

- A new Poridhi lab, or a lab "like the Poridhi ones".
- A lab for a specific Module and Lab number, such as "Module 3 Lab 1".
- A review or rewrite of an existing Poridhi lab to fix style issues.
- A lab on a system design topic: transport (REST, gRPC), failures, consistency, caching, sharding, load balancing.

Do not invoke for general tutorials or blog posts.

## Lab folder layout

Place the lab here:

```
Poridhi Labs/System Design S3/Milestone 1/Module X/Lab YY/
├── README.md        # the lab itself, all instructions live here
└── images/          # all diagrams and screenshots
    ├── thumb.png
    ├── concept-1.png
    ├── concept-2.png
    └── ...
```

Rules:

- Folder names match the existing pattern: `Module 1`, `Module 2`, `Module3` (use whatever already exists in the repo for that module).
- `Lab 01`, `Lab 02` style with a space and zero-padded number.
- `images/` is lowercase, always plural.
- Image filenames are lowercase with hyphens: `rpc-flow.png`, `step-2-output.png`.
- Source files like `server.js`, `client.js`, `docker-compose.yml`, `proto/user.proto` sit at the same level as `README.md`.

## README skeleton

Copy this template and fill it in. The thumbnail image goes right under the title, before any text, so the reader sees the picture first.

```
# <Lab Title>

![Lab thumbnail](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/ModuleX/Lab%200Y/images/thumb.png)

<One to three sentences: what the learner will build and why it matters.>

## Objective

By the end of this lab, you can:
- <Outcome one>
- <Outcome two>
- <Outcome three>

## Deliverable

<A single concrete artifact, e.g. "A working RPC client and server that exchange a `GetUser` request and response over gRPC.">

## <Concept One>

![Concept diagram](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/ModuleX/Lab%200Y/images/concept-1.png)

<Two to four sentences in plain English.>

## <Concept Two>

![Concept diagram](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/ModuleX/Lab%200Y/images/concept-2.png)

<Two to four sentences.>

## Step 1: <verb> <thing>

<One sentence of why this step exists.>

![Step diagram or screenshot](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/ModuleX/Lab%200Y/images/step-1.png)

```bash
<command or file contents>
```

**Explanation:** <One short sentence in plain English.>

---

## Step 2: <verb> <thing>

<Repeat the same pattern: short reason, image, code block, one-line explanation, `---`.>

## 5. Run the following commands in the terminal

```bash
<commands>
```

![Run output](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/ModuleX/Lab%200Y/images/run-output.png)

**What to expect:** <A few lines describing the expected terminal output in plain English.>

## Verification checklist

- [ ] <Thing the learner can now do>
- [ ] <Second thing>
- [ ] <Third thing>

## Conclusion

<One to three sentences that close the lab and point to the next lab or topic.>
```

## Style rules

Apply all of these without exception:

1. **No em-dash characters.** Use commas, colons, periods, or hyphens only. Search the file for `—` and `–` before saving. Replace any found.
2. **Beginner English.** Short sentences. Plain words. If a term like "RPC" or "consensus" must appear, define it on first use in one short clause.
3. **Visuals first.** Every section starts with an image, then text. The thumbnail sits right under the title, before the intro paragraph.
4. **Every step gets a picture.** Each `## Step` block includes a diagram, screenshot, or ASCII figure before its code block. If you cannot draw one, add a small text-based diagram in a fenced block.
5. **Code is copy-pasteable.** No `<placeholder>` text inside runnable code. The learner should be able to copy a block and run it as-is.
6. **One explanation line per block.** After every code block, write either `**Explanation:**` (for terminal commands) or `**What this file does:**` (for full files). One sentence, plain English.
7. **Use tables for comparisons.** Trade-offs, decision rules, and summary numbers go in tables. Short rows. First column names the option. Three to six rows is ideal.
8. **Number steps clearly.** Each hands-on step is `## Step N: <verb> <thing>`. Use strong verbs: create, write, run, edit, add, build, test, send, measure. End each step with `---`.
9. **One run section per lab.** Use `## 5. Run the following commands in the terminal` (or the next number) to gather everything that needs to run together, with a screenshot of expected output.
10. **Keep paragraphs short.** Two to five sentences per paragraph. Three to six rows per table.
11. **Verification before Conclusion.** Place the yes/no checklist just before the Conclusion. Three to six items.
12. **Stay tight.** Aim for 700 to 1200 words total for the whole README. If the draft grows past 1500 words, cut.

## Image and diagram rules

- Three image kinds: a thumbnail under the title, a concept diagram for each concept section, a step diagram or screenshot for each step.
- Reference every image with this full URL pattern, matching the module and lab numbers in the path:

  ```
  https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/ModuleX/Lab%200Y/images/<file>.png
  ```

- URL-encode spaces as `%20`. Note: existing Poridhi labs sometimes use `Module2` and `Module 2` interchangeably. Match the spelling of the folder name that already exists in the repo for that module.
- Save the image file under the local `images/` folder, lowercase, hyphens.
- Prefer clear ASCII diagrams in fenced blocks when an image is not available. Example:

  ```
  Client  --->  Load Balancer  --->  App Server  --->  Database
     ^                                                |
     |________________________________________________|
  ```

## Code block rules

- Always set a language tag: ` ```bash `, ` ```javascript `, ` ```yaml `, ` ```python `, ` ```proto `, ` ```json `.
- Never split a long command across lines without a warning. If a line is long, add a short comment above the block: `# paste this as one line`.
- For full files like `server.js`, `Dockerfile`, `docker-compose.yml`, paste the whole file. Do not write `... rest of file ...`.
- After every file write, add `**What this file does:**` and one or two plain sentences.
- After every terminal command, add `**Explanation:**` and one short sentence.
- Reference files by name in backticks: `README.md`, `server.js`, `proto/user.proto`.

## Length budget

Target 700 to 1200 words for a finished lab README. If the draft grows past 1500 words, cut in this order:

1. Shrink intro paragraphs to one to three sentences each.
2. Drop any concept section that repeats earlier material.
3. Combine small steps into one step.
4. Move reference tables to the end and keep only the most useful rows.
5. Remove any paragraph that does not help the learner build or run something.

## Verification checklist before publishing

Run this on every finished README. If any box is unchecked, fix it before saving.

- [ ] No `—` characters anywhere in the file.
- [ ] Thumbnail image appears right under the title, before any paragraph.
- [ ] Every concept section starts with an image, then text.
- [ ] Every Step block includes a diagram, screenshot, or ASCII figure before its code block.
- [ ] Every code block has a language tag and a one-line Explanation or "What this file does" follow-up.
- [ ] Image URLs follow the GitHub raw pattern, with `%20` for spaces, and module/lab numbers matching the folder.
- [ ] Steps are numbered `## Step N: ...` and each step ends with `---`.
- [ ] Run section shows expected output as a screenshot, with a `**What to expect:**` line in plain English.
- [ ] Verification checklist has three to six yes/no items, placed just before the Conclusion.
- [ ] Conclusion is one to three sentences and points forward.
- [ ] Total word count is between 700 and 1200 words.

## Example good opening

This is the right shape for the top of a lab. Note the thumbnail comes first, then a short intro.

```
# Hello Distributed System

Build two small programs that talk to each other over a network. You will see how a request leaves one process, travels across the wire, and reaches another process.

![Hello Distributed System thumbnail](https://raw.githubusercontent.com/poridhiEng/lab-asset/main/System%20Design%20Season%203/Milestone%2001/Module%2001/Lab%2001/images/thumb.png)

## Objective

By the end of this lab, you can:
- Run two Node.js programs that exchange messages over TCP.
- Measure the round trip time, called RTT, between them.
- Show what happens when one side stops responding mid conversation.

## Deliverable

Two small Node.js programs, a server and a client, that exchange three messages over TCP and log the round trip time for each reply.
```

## Things to avoid

Common mistakes. Do not produce any of these.

- Any em-dash character anywhere in the file.
- Long paragraphs with no heading or image to break them up.
- Code blocks missing a language tag, or missing the Explanation line right after.
- Splitting one shell command across multiple lines without a warning comment.
- Images that are missing, broken, or use a path that does not match the module and lab number.
- Tables with more than six rows, or rows that are full sentences instead of short phrases.
- Steps that mix writing a file and running it into one block. Keep write steps and run steps separate.
- A Conclusion that just restates the Objective. Close the lab and point to what comes next.
- Jargon without a one-line definition the first time it appears.
- Pasting the same big code block twice. If two steps need the same file, write it once and reference it the second time.