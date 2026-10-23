# web3-app

A take-home exercise from January 2024 (an infrastructure engineer
assignment). The code answers part of Task 1 below: a Python service that
watches USDT `Transfer` events on Ethereum mainnet and sends an email when a
transfer goes to the zero address, plus a small Flask page that lists recent
transfer events.

## What is in the repo

| Path | What it does |
| --- | --- |
| `backend/main.py` | Connects to Ethereum mainnet through Infura, polls the token contract's `Transfer` events every 10 seconds, keeps the latest 100 events in a Redis list named `events`, and emails `RECIPIENT_EMAIL` through Gmail SMTP when the `to` address is `0x0000000000000000000000000000000000000000`. |
| `frontend/app.py`, `frontend/templates/status.html` | Flask app. `/` renders the page and `/stream` is a server-sent events endpoint that sends the events stored in Redis. |
| `sweep.yaml`, `.github/ISSUE_TEMPLATE/sweep-template.yml` | Config added by the Sweep AI bot in 2025. The code does not use it. |

## Running it

You need Python 3, a Redis server on `localhost:6379`, an Infura project ID,
and a Gmail account with an app password for sending alerts.

There is no requirements file. The code uses the web3.py 6 API
(`create_filter(fromBlock=...)`, renamed in web3.py 7), so pin it:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install "web3>=6,<7" redis python-dotenv flask
```

Create a `.env` file in the repo root (it is gitignored):

```
INFURA_PROJECT_ID=...
GMAIL_USER=...
GMAIL_PASSWORD=...          # a Gmail app password
RECIPIENT_EMAIL=...
TOKEN_CONTRACT_ADDRESS=0xdAC17F958D2ee523a2206206994597C13D831ec7
TOKEN_CONTRACT_ABI=[...]    # the token ABI as a JSON string; must include the Transfer event
```

`0xdAC17F958D2ee523a2206206994597C13D831ec7` is the USDT contract on
Ethereum mainnet. Any ERC-20 address and ABI work the same way.

Start the monitor and the page in two terminals:

```bash
python backend/main.py
python frontend/app.py      # Flask dev server with debug on, http://127.0.0.1:5000
```

## Status and known limitations

The repo has not changed since the exercise was submitted, apart from the
Sweep config.

- Only Task 1 code is here. The written answers to Q1 and Q2 and the Task 2
  reward function are not in the repo.
- The page does not update live. The backend writes to a Redis list but
  never publishes to the `events` channel that `/stream` subscribes to, so
  the page shows only the events stored when it loads.
- The backend stores every `Transfer` event of the configured token, not
  only zero-address transfers. Only zero-address transfers send an email.
- If the polling loop raises an error (a network failure or an expired
  filter, for example), the process logs it and exits. There is no retry,
  restart or second provider, so the code does not meet the brief's "no
  downtime" goal on its own.
- There are no tests.

## The assignment brief

### Task 1: monitoring system design

Implement a system that monitors USDT transfers to the zero address
(`0x0000000000000000000000000000000000000000`) on Ethereum mainnet and
notifies the team when one happens, so that:

- the team is notified within 2 minutes of the transfer event,
- it is resilient (no downtime),
- it is cost-effective,
- it can easily be extended to monitor other addresses or other tokens.

Questions:

- Q1: Give the technical details (pseudo-code or an overview of the
  algorithm) and explain why the design is better than other designs.
- Q2: Describe the loopholes and trade-offs of the design that could stop
  the monitoring from working.

### Task 2: reward distribution

- Q3: Write a Python 3 function, with tests for normal and edge cases, that
  calculates each user's reward:

```python
def calculate_reward(events: list[tuple[str, int]]) -> dict[str, float]:
    raise NotImplemented()
```

10,000 USDT is distributed to users of a lending protocol between timestamp
0 and 3600 (one hour), at about 2.778 USDT per second. At each point in time
the tokens are split in proportion to each user's shares. A user action
either increases shares or decreases them (shares never go negative). Events
are tuples of `(user, timestamp, share_adjust)`.

Example: `[("A", 0, 2), ("B", 2, 1), ("A", 10, -1)]`

- From t=0 to t=1 only A holds shares, so A gets 2.778 USDT.
- From t=1 to t=10 A has 2 shares and B has 1, so A gets 16.667 USDT and B
  gets 8.333 USDT.
- From t=10 to t=3600 A and B have 1 share each, so each gets 4986.111 USDT.

Expected output: `{"A": 5005.556, "B": 4994.444}`

The brief is quoted as given, with two inconsistencies: the signature types
events as 2-tuples while the example uses 3-tuples, and the example lists
B's event at t=2 while the walkthrough and totals assume t=1.

## License

GPL-3.0. See [LICENSE](LICENSE).
