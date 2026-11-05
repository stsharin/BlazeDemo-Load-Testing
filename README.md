# BlazeDemo JMeter Load Testing

Performance and functional-flow tests for [BlazeDemo](https://blazedemo.com), a public practice site for load testing. The project covers the full ticket-buying journey, from the home page to the purchase confirmation, and is built with Apache JMeter.

> **Note:** BlazeDemo is a shared public site. The tests here are kept deliberately small (max 20 virtual users, max 5 minutes per run) and must never be run as a real stress, spike or soak test against it. See [Load limits](#load-limits) below.

## User flow under test

| # | Step | Request | Check |
|---|------|---------|-------|
| 01 | Open the home page | `GET /` | Page contains "Welcome to the Simple Travel Agency" |
| 02 | Search flights | `POST /reserve.php` | Page lists flights for the chosen route |
| 03 | Choose a flight | `POST /purchase.php` | Page says "has been reserved" |
| 04 | Buy the ticket | `POST /confirmation.php` | Page contains "Thank you for your purchase" |

Each step has a **Response Assertion** that checks the page text, not just the HTTP status code. A **Duration Assertion** fails any request slower than 3000 ms.

## Project structure

```
blazedemo-jmeter/
├── test-plans/          JMeter test plans (.jmx)
│   └── smoke-test.jmx
├── test-data/           CSV data used by the tests
│   ├── flights.csv      Departure and destination city pairs
│   └── passengers.csv   Fake passenger names and test card numbers
├── results/             Raw result files (.jtl), not committed
├── reports/             Generated HTML reports, not committed
├── .gitignore
└── README.md
```

## Requirements

- macOS, Linux or Windows
- Java 17
- Apache JMeter 5.6 or later (tested with 5.6.3)

On macOS with Homebrew:

```bash
brew install openjdk@17 jmeter
```

## How to run the tests

Run all commands from the **project root** so the relative CSV paths resolve. Use **non-GUI mode** only; the GUI is for editing test plans, not running them.

### Smoke test (1 user, 1 loop)

Use this to confirm the flow still works before running anything bigger.

```bash
rm -rf results/smoke.jtl reports/smoke   # JMeter will not reuse an old results file or report folder
jmeter -n -t test-plans/smoke-test.jmx -l results/smoke.jtl -e -o reports/smoke
```

Before each run, delete the old results file and report folder, otherwise JMeter stops with "Results file is not empty". You can also add `-f` to the `jmeter` command, which deletes both for you.

The HTML report opens from `reports/smoke/index.html`.

### Changing the target

The host and protocol are JMeter properties, so you can point the plan at another environment without editing it:

```bash
jmeter -n -t test-plans/smoke-test.jmx -Jhost=example.com -Jprotocol=https -l results/smoke.jtl
```

## Load limits

This is a shared public site, so every test in this project stays inside these limits:

| Limit | Value | Why |
|-------|-------|-----|
| Maximum virtual users | 20 | Keeps load light enough for a public practice site that other people use |
| Maximum run length | 5 minutes | Short runs are enough to measure response times without sustained pressure |
| Think time | 1–3 seconds between steps (Uniform Random Timer) | Real users pause between pages, and without think time the load is much higher than a real user would generate |
| Stress, spike and soak tests | Not run against the live site | These push a system past normal use and are not appropriate for a shared public site |

## Results

### Smoke test (1 user, 1 loop)

| Metric | Result |
|--------|--------|
| Status | PASS (all 4 steps, all assertions) |
| Average response time | 917 ms |
| 95th percentile | 1540 ms (4 samples only, so this is the slowest request) |
| Error rate | 0% |

| Step | Response time |
|------|---------------|
| 01 - Home Page | 945 ms |
| 02 - Search Flights | 776 ms |
| 03 - Choose Flight | 407 ms |
| 04 - Buy Ticket | 1540 ms |

Assertions were also checked against a deliberately wrong expected string, and they correctly failed. That confirms the checks run and are not passing by default.

### Load test (10 users, 60 s ramp-up, 3 min)

| Metric | Result |
|--------|--------|
| Status | FAIL on the 3000 ms Duration Assertion (5 requests); all pages returned 200 and passed their text checks |
| Requests | 593 |
| Average response time | 547 ms |
| 95th percentile | 826 ms |
| Error rate (failed assertions) | 0.84% (5 of 593) |
| Slowest response | 12,698 ms |

Five requests (on steps 01 and 03) were slower than 3 seconds. The slow requests happened at different times during the run, not all at once. The 95th percentile (826 ms) is well under the limit, so most users saw normal speed.

### Step-up test (5 → 10 → 20 users)

| Metric | Result |
|--------|--------|
| Status | PASS (0 failed assertions, 0 requests over 3000 ms) |
| Requests | 649 |
| Average response time | 415 ms |
| 95th percentile | 528 ms |
| Error rate | 0% |
| Slowest response | 1,990 ms |

<!-- Add screenshots of the HTML report here, for example:
![Smoke test report](docs/smoke-report.png)
-->

