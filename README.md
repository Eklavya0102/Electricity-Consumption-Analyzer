# Electricity Consumption Analyzer

A small Python application demonstrating:
- Functional Programming: `map()`, `filter()`, `reduce()`, `lambda`, and list comprehension.
- TCP Client–Server Programming: Python `socket` module.
- Finite State Machine (FSM): `RECEIVED`, `VALIDATING`, `ANALYZING`, `COMPLETED`, and `ERROR`.
- Symbolic Computing: SymPy derives the energy and monthly-cost formulas and cross-checks them against the numeric result.
- Testing: 15 automated `unittest` tests, including the three required scenarios plus FSM, SymPy, and real TCP socket tests.

## Files

- `client.py` — sends the sample dataset and displays the report.
- `server.py` — accepts TCP requests and returns analysis as JSON.
- `analyzer.py` — validation, FSM, functional processing, and SymPy formulas.
- `test_analyzer.py` — 15 automated tests (analysis, validation, FSM, SymPy, TCP client–server). No manually started server is needed.
- `test_report.py` — detailed formatted test report for demonstration.
- `TEST_RESULTS.txt` — recorded output of `python -m unittest -v test_analyzer.py`.
- `DETAILED_TEST_REPORT_SAMPLE.txt` — recorded output of `python test_report.py`.
- `requirements.txt` — SymPy dependency.

## Requirements

Python 3.10+ recommended.

## Setup (Windows PowerShell or Command Prompt)

1. Open a terminal in this project folder.

2. Install the dependency:

   ```bash
   python -m pip install -r requirements.txt
   ```

## Run the application

Open **two terminal windows** in the project folder. Activate the virtual environment in both if you created one.

Terminal 1 — start the server:
```bash
python server.py
```

Terminal 2 — start the client:
```bash
python client.py
```

The client sends the sample appliance dataset to the server over TCP. The server validates and analyzes it, then sends the JSON report back to the client. Use `Ctrl+C` in the server terminal to stop the server.

### Message format (TCP protocol)

Each request and each response is **one JSON object followed by a newline** (`\n`), one request per connection. Because TCP is a byte stream, both sides keep calling `recv()` until they see the newline instead of assuming one `recv()` returns the whole message. The server answers bad input (malformed JSON, invalid UTF-8, truncated data, requests over 1 MB, silent clients after 10 seconds) with a JSON error reply, and a client that disconnects abruptly does not stop the server.

Request example:

```json
{"tariff": 8.0, "appliances": [{"name": "LED Bulb", "power_w": 10, "hours_per_day": 6}]}
```

### Example dataset and expected output

`client.py` sends this dataset (tariff Rs 8.00/kWh, 30-day month):

| Appliance | Power (W) | Hours/day | Daily kWh | Monthly kWh |
|---|---:|---:|---:|---:|
| LED Bulb | 10 | 6 | 0.060 | 1.800 |
| Ceiling Fan | 75 | 8 | 0.600 | 18.000 |
| Laptop | 65 | 5 | 0.325 | 9.750 |
| Refrigerator | 150 | 10 | 1.500 | 45.000 |
| Electric Heater | 1000 | 1 | 1.000 | 30.000 |

Expected totals: 3.485 kWh/day, 104.550 kWh/month, estimated bill **Rs 836.40**. Appliances at or above 30 kWh/month are flagged as high usage: Refrigerator and Electric Heater. The client also prints the FSM transition log (current state, input condition, next state) and the SymPy check. The server terminal prints the same FSM transitions for each request.

The server binds to `127.0.0.1`, so this example is intended for client and server on the same computer. For computers on a shared network, configure a trusted server IP and firewall deliberately; do not expose this demo server to the public internet.

## Run the tests

For a detailed, presentation-friendly report with inputs, expected results, actual results, FSM traces, and a final pass/fail summary, run:

```bash
python test_report.py
```

For the standard automated unit-test runner, also run:

```bash
python -m unittest -v test_analyzer.py
```

Expected result: `Ran 15 tests ... OK` for `test_analyzer.py` and `ALL TESTS PASSED` (3 test cases) for `test_report.py`. The tests start their own temporary TCP server on a free port, so nothing else needs to be running.

## FSM states and transitions

| Current state | Input/condition | Next state |
|---|---|---|
| `RECEIVED` | JSON payload received | `VALIDATING` |
| `RECEIVED` | Invalid payload event | `ERROR` |
| `VALIDATING` | All records valid | `ANALYZING` |
| `VALIDATING` | Invalid record | `ERROR` |
| `ANALYZING` | Calculations succeed | `COMPLETED` |
| `ANALYZING` | Calculation failure | `ERROR` |

The successful path is `RECEIVED → VALIDATING → ANALYZING → COMPLETED`. Every transition is logged as (current state, input condition, next state) and returned to the client. An event that is not allowed in the current state raises an error. `ANALYZING → ERROR` happens, for example, when a valid-looking input produces a result too large to represent.

## Functional programming concepts

All of these are in `analyze_payload()` in `analyzer.py`:

- `map()` with a `lambda` converts each appliance record into one with its daily kWh.
- `filter()` with a `lambda` keeps only appliances that are actually used (hours greater than 0).
- `reduce()` (from `functools`) combines the daily kWh of all active appliances into one total.
- A list comprehension builds the per-appliance monthly details and the list of high-usage appliances (30 kWh/month or more).

## Symbolic calculation

SymPy defines the symbols and derives these expressions:

- Daily energy (kWh): `power_w * hours_day / 1000`
- Monthly energy (kWh): `power_w * hours_day * days / 1000`
- Monthly bill (INR): `power_w * hours_day * days * tariff / 1000`

After the numeric calculation, every appliance record is substituted into these SymPy formulas and the symbolic totals are compared with the numeric totals. The client prints whether they agree.

The example assumes 30 days and a flat tariff of ₹8/kWh. These are demonstration assumptions, not a utility bill quote. Real tariffs may include slabs, fixed charges, taxes, and other adjustments.

## Test cases and expected results

| Test | Input | Expected result |
|---|---|---|
| 1. Normal usage | 10 W bulb, 5 h/day, ₹8/kWh | 0.05 kWh/day, 1.5 kWh/month, ₹12/month |
| 2. High consumption | 2000 W heater, 5 h/day, ₹8/kWh | 10 kWh/day, 300 kWh/month, ₹2400/month; heater flagged |
| 3. Invalid data | Fan usage = 30 h/day | Rejected with validation error; FSM ends in `ERROR` |

## Limitations

This is an educational prototype using sample data and a flat tariff. It does not connect to a smart meter or utility provider. The TCP service is a local classroom demo, not a production-hardened server.
