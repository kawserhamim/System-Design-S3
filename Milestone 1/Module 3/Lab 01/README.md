# Availability Math: Serial vs Parallel

## Objective

By the end of this lab, you can:

- Explain what **availability** means and how the "nines" describe it.
- Calculate the availability of a **serial** chain of components.
- Calculate the availability of a **parallel** group of components.
- Use a small Python script to verify your math on real numbers.
- Decide when a system needs redundancy and how much redundancy helps.

## Before You Start

You should be comfortable with:

- Opening a terminal and running two or three commands.
- Editing a text file and saving it.
- Basic Python. You only need to read and run a short script.

You will need:

- Python 3.8 or newer on your machine.
- About 45 to 60 minutes of focused time.

You do not need:

- A cloud account.
- A database.
- Any prior knowledge of distributed systems.

## Why Are We Doing This Lab?

Real systems break. Hard drives fail. Servers reboot. Networks drop. A single broken piece can take down a whole product.

The question is: how do we **measure** how reliable a system is, and how do we **plan** for failures?

This lab teaches two simple formulas that answer both questions. Once you know these formulas, you can look at any architecture diagram and quickly guess whether it will keep users happy or not.

Real world anchors:

- **Gmail** promises "99.9% of an active session". That is the famous "three nines".
- **AWS S3** storage is designed for "99.999999999%" durability. That is "eleven nines" and is why your files do not disappear.
- **Your home Wi-Fi** is closer to "two nines" (about 99%). You already feel those outages every week.

These numbers are not magic. They come from the two formulas you will learn today.

## What Is Availability?

**Availability** is the share of time a service is working.

A service that is up 99% of the time has availability `0.99`. A service that is up 99.9% of the time has availability `0.999`.

The "nines" are a quick way to talk about availability.

| Nines  | Availability | Downtime per day | Downtime per month | Downtime per year |
|--------|--------------|------------------|--------------------|-------------------|
| 1 nine | 90%          | 2.4 hours        | 73 hours           | 36.5 days         |
| 2 nines| 99%          | 14.4 minutes     | 7.3 hours          | 3.65 days         |
| 3 nines| 99.9%        | 86.4 seconds     | 43.8 minutes       | 8.77 hours        |
| 4 nines| 99.99%       | 8.64 seconds     | 4.38 minutes       | 52.6 minutes      |
| 5 nines| 99.999%      | 0.86 seconds     | 25.9 seconds       | 5.26 minutes      |

Notice how much downtime drops with each new nine. That is why engineers pay a lot for "five nines" (the phone network goal).

![availability table](image/availability.png)

## Serial Availability

### The Setup

A serial system is a chain. **Every** component must work for the whole system to work.

Think of a water pipe with three filters in a row. If any one filter gets clogged, water stops at the end.

![serial chain diagram](image/serial.png)

### The Formula

When components sit in a chain, you **multiply** their availabilities:

```
A_system = A1 × A2 × A3 × ...
```

### Worked Example

Imagine three components, each up 99.9% of the time (`A1 = A2 = A3 = 0.999`).

```
A_system = 0.999 × 0.999 × 0.999 = 0.997002
```

The chain is only about **99.7%** available, even though each piece is "three nines". The chain is weaker than its weakest link. It is weaker than the **product** of all links.

### Mental Model

In a serial chain, **every** part matters. Adding more pieces always makes the system **worse**, not better. The math punishes you for adding steps.

## Parallel Availability

### The Setup

A parallel system is a group of components that **all do the same job**. If one fails, the others still cover for it.

Think of three light bulbs in a room, wired so the room is lit if any bulb works. One bulb can burn out and you still have light.

![parallel group diagram](image/parallel.png)

### The Formula

For a parallel group, you start with the failure rate of each piece, multiply those, and subtract from 1:

```
A_parallel = 1 - (1 - A1) × (1 - A2) × (1 - A3)
```

The term `(1 - A)` is the **failure rate** of one component. Multiplying failure rates gives the chance that **every** piece fails at once. Subtracting from 1 gives the chance that **at least one** piece still works.

### Worked Example

Same three components, each at 0.999:

```
Failure of one = 1 - 0.999 = 0.001
A_parallel = 1 - (0.001 × 0.001 × 0.001)
           = 1 - 0.000000001
           = 0.999999999
```

That is **nine nines**. Three cheap components, properly wired, give you better availability than a single perfect one.

### Mental Model

In a parallel group, **the group survives if any member survives**. More redundancy almost always helps, but the gains shrink quickly. Two replicas give most of the benefit. Five replicas do not help much more than three.

## Practice Exercise

Try these on paper first. The answers are below the code section.

**Question 1.** Four components sit in a serial chain. Each is up 99.9% of the time. What is the system availability?

**Question 2.** A load balancer (`0.999`) feeds three identical app servers in parallel (`0.999` each), then writes to a database (`0.999`). What is the overall availability?

**Question 3.** Two replicas of a service run in parallel. Each replica is up 95% of the time. Is this better or worse than one replica at 99%? By how much?

## Step 1: Set Up a Workspace

Open a terminal and run these commands. They create a folder, move into it, and confirm Python is available.

```bash
mkdir availability-lab
cd availability-lab
python3 --version
```

You should see something like `Python 3.10.x` or `Python 3.11.x`. If `python3` is not found, install Python 3 from [python.org](https://www.python.org/downloads/).

## Step 2: Write the Script

Open a file called `avail.py` in your editor. Paste the code below and save it.

```python
def serial(*avail):
    result = 1.0
    for a in avail:
        result *= a
    return result

def parallel(*avail):
    failure = 1.0
    for a in avail:
        failure *= (1 - a)
    return 1 - failure

# Practice Question 1
q1 = serial(0.999, 0.999, 0.999, 0.999)

# Practice Question 2
apps = parallel(0.999, 0.999, 0.999)
q2 = serial(0.999, apps, 0.999)

# Practice Question 3
q3 = parallel(0.95, 0.95)

print(f"Q1 (4x serial, each 0.999)         = {q1:.6f}")
print(f"Q2 (LB -> 3 apps -> DB)            = {q2:.6f}")
print(f"Q3 (2x parallel at 0.95 vs 0.99)   = {q3:.6f}")
```

What each piece does:

- `serial(*avail)` multiplies all inputs together. That is the chain formula.
- `parallel(*avail)` multiplies the failure rates, then takes one minus the result.
- The three `print` lines run the practice questions and show six decimal places.

## Step 3: Run the Script

```bash
python3 avail.py
```

You should see:

```
Q1 (4x serial, each 0.999)         = 0.996006
Q2 (LB -> 3 apps -> DB)            = 0.998001
Q3 (2x parallel at 0.95 vs 0.99)   = 0.997500
```

Compare these to your hand answers:

- **Q1**: `0.9960`. About 99.60% available. Each extra serial piece costs you.
- **Q2**: `0.9980`. About 99.80%. The parallel apps lift the apps block above 0.999, but the load balancer and database are still serial bottlenecks.
- **Q3**: `0.9975`. Two replicas at 95% give 99.75%, which is **better** than one at 99% (`0.99`). Even cheap, less-reliable parts help when run in parallel.

## Step 4: Run Your Own Experiments

Open `avail.py` again and try these changes. Run the script after each one.

1. Add a fifth component in series to Q1. Watch the number drop.
2. Add a fourth replica to the Q2 app pool. Watch the Q2 number climb.
3. Change the load balancer availability in Q2 to `0.9`. Notice how one weak serial piece drags the whole system down.
4. Try `parallel(0.99, 0.99, 0.99)` and compare to `parallel(0.999, 0.999, 0.999)`. See how much each extra nine is worth.

Each change is a small lesson about how availability behaves under design choices.

## Step 5: Build Your Own Scenario

Now design a tiny system from scratch.

Pick **one** task below and answer it in your own words. Keep it short, one paragraph is fine.

1. **E-commerce cart.** A load balancer (`0.999`), two app servers in parallel (`0.999` each), a single database (`0.999`). Calculate the availability and decide if you would ship it.
2. **Photo backup.** Three identical storage nodes in parallel (`0.98` each). What do you get? Is one more node worth adding?
3. **Chat app.** A login service (`0.99`) in series with three message servers in parallel (`0.95` each). Where is the bottleneck?

For each scenario, write:

- The formula you used.
- The final number.
- One sentence on what you would change to make the system better.

## Help! I Am Stuck

### Common Errors

**`python3: command not found`**

Install Python 3 from [python.org](https://www.python.org/downloads/) or with your package manager. On Debian or Ubuntu, run `sudo apt install python3`. On macOS with Homebrew, run `brew install python`.

**`SyntaxError: invalid syntax` after pasting the script**

Most often this is a missing closing parenthesis or a smart quote. Re-copy the code block as plain text. Avoid pasting from a Word doc or rich text editor.

**The numbers do not match the expected output**

Check that you saved the file before running it. Re-run `python3 avail.py` from inside the `availability-lab` folder. If you still see different values, paste your script into a fresh file and try again.

**`IndentationError: unexpected indent`**

Python cares about spaces at the start of a line. Make sure the lines inside `serial` and `parallel` start with exactly four spaces. Do not mix tabs and spaces.

**I do not know if my hand math is right**

Use a calculator: multiply `0.999` four times for Q1, or multiply three failure rates and subtract from 1 for parallel. The script is a quick check, not the source of truth.

## Glossary

- **Availability**: the share of time a service is up and serving requests.
- **Component**: a single piece of a system, like a server, a database, or a load balancer.
- **Failure rate**: the share of time a component is **down**. Equal to `1 - availability`.
- **Nine (nines)**: the number of 9s after the decimal point in the availability number. "Three nines" means 99.9%.
- **Parallel**: a group of components that can each do the same job. The group works as long as at least one member works.
- **Redundancy**: having extra components so a failure does not break the system.
- **Serial (chain)**: a sequence of components where **every** one must work for the whole chain to work.

## Self-Check Questions

Try these without looking back at the lab. The answers live in your head and your notes.

1. A serial chain with five components at `0.999` each. Roughly what availability do you get?
2. Why does adding a sixth component to a serial chain make things worse, while adding a sixth component to a parallel group makes things better?
3. If one piece of a serial chain is only `0.9`, can the rest of the chain still be "five nines"? Why or why not?
4. A system has a serial dependency on a database. Adding more app servers in front of the database does not help much. Why?
5. Name one real service you use. Guess its availability in nines. Was your guess reasonable?

## Key Takeaways

![key takeaways part 1](image/key%20takeaways.png)

![key takeaways part 2](image/key%20takeaways2.png)

In short:

- Serial chains **multiply** availability. Every added step makes the system weaker.
- Parallel groups **multiply failure rates** and take one minus the result. Every added replica makes the system stronger, with shrinking returns.
- The "nines" are a quick way to compare real systems: Gmail is "three nines", AWS S3 is "eleven nines", and your home Wi-Fi is closer to "two nines".
- A single weak serial dependency is the most common way a system loses its nines.

![availability math recap](image/image.png)

## Conclusion

You learned two simple formulas:

- `serial = A1 × A2 × ...`
- `parallel = 1 - (1 - A1) × (1 - A2) × ...`

You ran a Python script that computes both, you tried your own scenarios, and you learned to spot bottlenecks in real architectures.

In the next lab, you will take these formulas and apply them to a real distributed system: a load balancer in front of two app servers, with a database behind. You will see how a tiny configuration change moves the nines by orders of magnitude.

If you want to keep practicing, change the numbers in `avail.py` and watch the answers move. The intuition you build now will save you hours in design reviews later.
