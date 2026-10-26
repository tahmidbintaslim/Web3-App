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
| `sweep.yaml`, `.github/ISSUE_TEMPLATE/sweep-template.yml` | Config for the Sweep AI GitHub app. The code does not use it. `sweep-ai[bot]` committed both on the `sweep/add-sweep-config` branch on 2024-01-21; they reached `main` when PR #1 was squash-merged on 2025-12-12. |

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

Copy `.env.example` to `.env` in the repo root (`.env` is gitignored) and fill it in:

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

The last code change is a "Bug Fixes" commit on 2024-01-31, which changed
how `status.html` reads the event fields. The only later change on `main`
is the Sweep config.

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

The brief from the original README. The wording is unchanged; only the
layout was converted to Markdown.

> **Task1 – Monitoring System Design**
>
> Scenario: Implement a robust system to monitor USDT transfer to the ZERO address
> (https://etherscan.io/address/0x0000000000000000000000000000000000000000) on Ethereum
> mainnet. In case the transfer happens, our team would like to be notified to take appropriate
> actions.
> Design the system, such that:
>
> - Once the token transfer event happens, our team gets notified within 2 minutes.
> - Resilient – No downtime
> - Cost-effective
> - Can easily be modified to add more token transfer monitorings (e.g. to other addresses,
>   or monitor other token addresses)
>
> **Q1: Please provide technical details (can be as in-depth as pseudo-code and/or an
> overview of the algorithm or approaches used), including the analysis of why the design
> is good, compared to other possible designs.**
>
> **Q2: Please also state the analysis of potential loopholes / trade-offs that the design you
> proposed may have (that may cause the monitoring to not function properly).**
>
> **Task 2 – Reward Distribution**
>
> **Q3: Write a Python3 code that calculates how much each user should receive the reward.**
> Please include any test cases to test the functionality, including normal and edge cases
> as well.
>
> Implement the Python function according to the format below:
>
> ```python
> def calculate_reward(events: list[tuple[str,int]]) -> dict[str, float]:
> raise NotImplemented()
> ```
>
> TODO
>
> Problem statement:
> We have 10,000 USDT to be distributed to our users who use our lending protocol during the
> timestamp 0 to 3600 (1 hour duration). The reward distribution algorithm is such that the 1000
> tokens are distributed at the rate of ~2.778 USDT / sec, where at each point in time, the tokens
> will be distributed proportionally based on the total number of user’s shares.
> Each user’s action can either be:
>
> 1. Increase shares by XX, or
> 2. Decrease shares by YY (user’s shares can never be negative)
>
> Example:
>
> - User action events – format will be a list of tuples of (user, timestamp, share_adjust):
>   - [ (“A”, 0, 2), (“B”, 2, 1), (“A”, 10, -1) ]
>
> The above example list of events means that
>
> - User A increases 2 shares at time t=0
> - User B increases 1 share at time t=2
> - User A decreases 1 share at time t=10
>
> So,
>
> - Between time t=0 & t=1, only user A is getting the rewards → A gets 2.778 USDT
> - Between time t=1 & t=10, A has 2 shares, while B has 1 share
>   → A gets 2⁄3 of what’s distributed, and B gets 1⁄3 of what’s distributed
>   → A gets 16.667 USDT and B gets 8.333 USDT
> - Between time t=10 & t=3600, A and B has 1 share each
>   → A gets 1⁄2 of what’s distributed, and B gets 1⁄2 of what’s distributed
>   → A and B each gets 4986.111 USDT
>
> So, in total, A gets 5005.556 USDT and B gets 4994.444 USDT
> Expected Output: a dictionary of {“A” : 5005.556, “B” : 4994.444 }

The brief has three inconsistencies: the signature types events as
2-tuples while the example uses 3-tuples, the example lists B's event at
t=2 while the walkthrough and totals assume t=1, and it gives both 10,000
USDT and "1000 tokens" for the same distribution.

## License

GPL-3.0. See [LICENSE](LICENSE).
