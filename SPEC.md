# Savings Sweep: build spec

> **To Claude Code on my laptop:** read this whole file, then build it in a new folder
> (not inside the coffee shop project). Start with manual mode (section 5) so I can use it
> right away, then add SimpleFIN. Ask me before anything that needs my bank login or a token.

## 1. Problem

At the start of each month I want to move whatever is left of last month's pay into savings.
Rent and my Bank of America card are paid by the 1st. My two Amex cards and my Amazon credit
account are paid later (around the 9th and the 12th), so on the 1st my checking account still
holds money that is already spoken for. A paycheck that lands before those due dates covers part
of them, so I can move more than checking alone suggests, but only if it lands in time. Working
out each card's statement balance, and whether each paycheck lands in time, by hand is tedious
and error-prone.

## 2. The rule

Walk the days from today through the last due date, in date order:

```
balance = checking_available
        + each expected deposit, on the day it lands
        − each card's unpaid statement balance, on its due date
        − each other payment from checking, on its due date

safe_to_move_today = lowest balance on any of those days − cushion
```

- Use each card's **unpaid statement balance**: the last statement's balance minus any payments
  made since it closed. That's exactly what the Amex/Amazon autopays pull from checking on the
  9th/12th.
- Charges made after a statement closed are left out. They land on next month's statement and
  come out of next month's pay.
- Cards already paid (Bank of America, usually Bilt) simply show $0.
- A deposit counts only from the day it lands. One that lands before a card is due covers it;
  one that lands after doesn't help that card. Money moved today lowers the balance on every
  later day too, so the lowest point, not the end balance, sets today's amount.
- When the end balance minus the cushion is more than today's amount, the rest frees up once
  the deposit that lifts the low point lands. Report it: "Another $1,605.00 frees up on Fri
  Nov 13."
- Err on the side of moving less: a deposit dated today or earlier is already in
  `checking_available`, so don't count it again; on a day with both, payments go out before
  deposits land; deposits after the last due date belong to the next run.
- Three outcomes:
  - `ok`: `safe_to_move_today` > 0. Move that amount.
  - `cushion`: the balance never goes negative, but its lowest point doesn't clear the cushion.
    Move nothing today, and still report what frees up later.
  - `short`: the balance goes negative on some day. Show the shortfall and that day
    prominently.

## 3. My accounts

| Account               | Role     | Paid                     | Notes                                                  |
|-----------------------|----------|--------------------------|--------------------------------------------------------|
| Checking              | checking | n/a                      | Where my salary lands. Use the *available* balance.    |
| Bank of America card  | card     | by the 1st               | Usually $0 at sweep time.                              |
| Amex Platinum         | card     | ~9th or ~12th (confirm)  |                                                        |
| Amex Gold             | card     | ~9th or ~12th (confirm)  | I'm cancelling one of the two Amex cards in March 2027. |
| Amazon credit account | card     | after the 1st            | Issuer may be Synchrony. Might not connect through an aggregator, so it may need manual entry. |
| Bilt                  | card     | rent only                | Include it anyway; normally $0.                        |

Ask me each card's statement closing day and due day, and my pay schedule (how often, which day
and the usual amount), before writing my real config.

This list will change: cards get opened and cancelled, and I may add other payment
destinations like a loan. Adding or removing one must be a single command (see 4), never a
code change.

## 4. What to build

A small Python 3.11+ command-line tool called `sweep`.

```
sweep            # fetch balances, print the breakdown and the amount to move
sweep --manual   # no network: prompt for every balance
sweep accounts   # list accounts found by the data source with ids and raw balances
sweep setup      # one-time: paste a SimpleFIN setup token, store the access URL securely
sweep --json     # machine-readable output (for a notification later)

sweep list                                           # show checking, cards, payments and income
sweep add card "Chase Sapphire"                      # add a card (prompts to link it, see below)
sweep add payment "Car loan" --amount 320 --due-day 15
sweep add income "Paycheck" --amount 1700 --every week --day friday
sweep add income "Tax refund" --amount 900 --once 2026-11-20
sweep rename "Amex Gold" "Amex Gold (old)"
sweep remove "Amex Gold"                             # asks to confirm first
```

**Payment destinations** are everything checking still has to pay. There are two kinds:

- `card`: the balance changes, so it's fetched (or typed in) on every run.
- `payment`: a fixed amount on a due day, paid straight from checking (loan, utility, rent
  that hasn't cleared).

**Income** is money expected to land in checking. All I give it is how often and which day:
`--every week --day friday`, `--every month --day 15`, `--every "2 weeks" --day 2026-11-13`
(any one payday) or `--once 2026-11-20`. Without flags, `sweep add income` asks those two
questions. It works out every future date itself. The amount is what I usually get.

`sweep add card` lists the SimpleFIN accounts not yet in config and lets me pick one, or choose
"enter by hand" (`source = "manual"`), then asks for the card's due day and statement closing
day. `add`, `rename` and `remove` edit `config.toml` with
`tomlkit` so my comments and ordering survive, and write a `config.toml.bak` first. Name
matching is case-insensitive. An unknown name lists the names that do exist.

### Example output

```
Savings sweep · Sun Nov 1, 2026          balances as of Nov 1, 8:12 AM

Checking                                   6,850.00
Coming in
  Paycheck (Fri Nov 13)                   +1,700.00
Cards
  Bank of America (due Nov 1)                  0.00
  Amex Platinum (due Nov 9)                1,240.18
  Amex Gold (due Nov 9)                      386.40
  Amazon credit (due Nov 12, by hand)        212.75
  Bilt (due Nov 1)                             0.00
                                          −1,839.33
Other payments
  Utilities (due Nov 15)                     −95.00
Cushion                                     −500.00
Not free until Nov 13                     −1,605.00
───────────────────────────────────────────────────
Move to savings today                      4,510.67

Another 1,605.00 frees up on Fri Nov 13, when Paycheck lands.
```

Checking bottoms out at 5,010.67 on Nov 12, after Amazon and before the paycheck, so
5,010.67 − 500.00 can move today. Once the paycheck lands, nothing later takes checking below
6,615.67, so another 1,605.00 frees up. The `Not free until` line makes the column add up.

Use `rich` (or plain text) for the table. Warnings print above the result in a different color.

### Project layout

```
sweep/
  calc.py        # pure functions, no I/O. All the money logic lives here.
  config.py      # load and validate config.toml
  simplefin.py   # SimpleFIN client (claim token, fetch accounts)
  manual.py      # interactive prompts
  cli.py         # argument parsing and output
tests/
  fixtures/simplefin_accounts.json
  test_calc.py, test_config.py, test_simplefin.py
pyproject.toml   # entry point `sweep = sweep.cli:main`
```

### Config (`config.toml`, in the platform config dir via `platformdirs`; contains no secrets)

```toml
cushion = 500.00

[[accounts]]
name = "Checking"
role = "checking"
simplefin_id = "ACT-..."       # copied from `sweep accounts`

[[accounts]]
name = "Amex Platinum"
role = "card"
simplefin_id = "ACT-..."
statement_day = 15             # day of month the statement closes (ask me); see 6
due_day = 9                    # day of month autopay pulls the statement balance

[[accounts]]
name = "Amazon credit"
role = "card"
source = "manual"              # not connected; prompt for it each run
due_day = 12

[[accounts]]
name = "Old savings"
role = "ignore"                # shown by `sweep accounts`, never counted

[[payments]]
name = "Utilities"
amount = 95.00
due_day = 15

[[income]]
name = "Paycheck"
amount = 1700.00               # usual amount; --manual lets me correct it per run
every = "2 weeks"              # "week", "2 weeks", "month" or "once"
day = 2026-11-13               # "week": a weekday ("friday"); "month": 1-31;
                               # "2 weeks": any one payday; "once": the date
```

Every card needs `due_day`; cards fetched from SimpleFIN also need `statement_day`. Optional
per-account key: `invert_sign = true` (see 6).

### Calculation details (`calc.py`)

- Money is `Decimal` (or integer cents) everywhere. Never `float`.
- Card owed amount is never negative: a card with a credit balance counts as $0 owed.
- Working out a card's unpaid statement balance from fetched data (6, step 5) is a pure function
  here too.
- Horizon: today through the latest next due date among cards and payments. Each card and
  payment counts once, on its next due date on or after today. Clamp `due_day` to the month's
  last day (e.g. 31 → Feb 28/29). A payment due today may already have cleared; support
  `--skip NAME` to drop it for this run.
- Income: work out each landing from `every` and `day`. Weekly lands on that weekday, monthly
  on that day (clamped like `due_day`), and every 2 weeks on dates 14 days apart from `day`,
  before or after it. It counts on each date after today that falls within the horizon. A
  one-off counts once; past one-offs are ignored, and `sweep list` marks them so I can remove
  them.
- Return a result object with every line item, the totals, the lowest balance and its date,
  today's amount, the amount that frees up later and its date, and `status`
  (`ok`/`cushion`/`short`), so the CLI and `--json` share one source of truth.

## 5. Phase 1: manual mode (build first)

`sweep --manual` prompts for checking and each card's unpaid statement balance in config
order, accepts input like `$1,240.18` or `-1240.18` (a minus sign on a card is ignored), then
prints the same output. It also shows each deposit in the horizon with its configured amount;
Enter keeps it, or I type what actually came in. Accounts with `source = "manual"` use these
prompts even in normal mode.

## 6. Phase 2: automatic balances via SimpleFIN Bridge

SimpleFIN Bridge is a read-only bank aggregator for personal use with a small yearly fee. I
connect my banks on their site and get a setup token. **Read the protocol at
https://www.simplefin.org/protocol.html before writing the client** and follow it over anything
here. In outline:

1. `sweep setup`: I paste the setup token. It's base64; decode it to a claim URL, `POST` to it
   once, and get back an access URL with credentials embedded. Store the access URL with
   `keyring` (macOS Keychain / Windows Credential Manager). Never print or log it.
2. Fetch `GET {access_url}/accounts` with `start-date` set to the earliest card's last
   statement closing date, since step 5 needs the transactions posted after it. The bridge is
   rate-limited (roughly a day's worth of requests), so cache the last response for 15 minutes.
3. Each account has an id, name, org, `balance`, `available-balance` and `balance-date`
   (unix time). Use `available-balance` for checking when present. A card's `balance` is its
   current balance; SimpleFIN has no statement balance, so step 5 works it out.
4. **Sign convention:** card balances usually come back negative when I owe money. Don't assume
   it. Owed = `-balance` when negative, 0 otherwise, with `invert_sign = true` per account for
   institutions that report it the other way. `sweep accounts` must show raw balances so I can
   compare against my bank apps on the first run.
5. **Unpaid statement balance:** owed (step 4) minus the charges posted after the last
   statement closing date, floored at $0. The last closing date is the most recent
   `statement_day` on or before today, clamped to the month's last day. Charges are
   transactions that increase what I owe; payments and refunds since the closing date already
   lowered the balance, so leave them in. Ignore pending transactions: they all came after the
   statement closed.
6. Show the protocol's `errors` array to me verbatim. It usually means a connection needs to
   be re-authenticated on the SimpleFIN site.

Before I pay for it, I'll check that Bank of America, American Express, Synchrony/Amazon and
Bilt can be connected. Anything that can't is just `source = "manual"`.

## 7. Warnings

- A balance older than 36 hours (`balance-date`): print which account and how old.
- An account in config that the data source didn't return: error naming it, fall back to a
  manual prompt.
- A card payment pending in checking: the money may be subtracted twice. Print a reminder
  if a transaction description in checking matches a card name in the last 3 days.
- Result `short`: print the shortfall, the day it happens, and which payments it threatens.

## 8. Phase 3 (optional): run itself on the 1st and ping my phone

- `sweep --json | sweep notify` sends a one-line push ("Move $4,510.67 to savings") via
  ntfy.sh or Pushover (ask me which). Put the topic or key in `keyring`, not config.
- Schedule it on the 1st at 9:00 AM with launchd (macOS) or Task Scheduler (Windows), whichever
  my laptop runs. Write the plist/task file and the install command; don't install it without
  asking.

## 9. Security

- Read-only access only. The tool never moves money.
- Secrets live in `keyring`. `config.toml` has no secrets, account numbers or balances.
- No balances in logs. The cache file lives in the user cache dir with `0600` permissions.
- HTTPS with certificate verification and request timeouts. No other network calls.
- `.gitignore` covers config, cache and any `.env`.

## 10. Tests (pytest)

- `calc`: the example above gives exactly 4,510.67 today and 1,605.00 more on Nov 13;
  `cushion` and `short` states; credit balance on a card counts as $0; Decimal rounding (no
  float drift on values like 0.1 + 0.2).
- Timing: checking 4,300, statements 4,300 due Nov 9 and a 1,700 paycheck on Nov 6 gives
  1,700. The same paycheck on Nov 10, with 3,000 due Nov 9 and 1,300 due Nov 12, gives 1,300
  today and 400 more on Nov 10. With checking 1,000 instead, it's `short` by 2,000 on Nov 9.
  A deposit dated today isn't counted, and on a shared day the payment goes first.
- Unpaid statement balance: owed 1,500 with 300 of charges posted after the closing date gives
  1,200; a payment since the closing date lowers it; a statement already paid in full gives $0;
  `statement_day = 31` closes on Feb 28/29.
- Horizon: today Jan 30, `due_day = 31` falls on Jan 31 and `due_day = 2` on Feb 2;
  `due_day = 31` in February falls on Feb 28/29; weekly on Friday, run on a Friday, starts next
  Friday; an every-2-weeks `day` months in the past still lands on the right Fridays; a
  paycheck after the last due date isn't counted.
- Config: unknown role rejected; a missing `simplefin_id` without `source = "manual"` rejected;
  a SimpleFIN card without `statement_day` rejected; a card without `due_day` rejected; an
  income `every` other than week, 2 weeks, month or once rejected, and a `day` that doesn't
  fit its `every` rejected.
- `add` / `rename` / `remove`: round-trip a config with comments and keep them; duplicate
  names rejected; removing an unknown name changes nothing; a `.bak` is written first.
- SimpleFIN: parse the fixture JSON; sign normalization with and without `invert_sign`;
  stale-balance warning; `errors` array surfaced; access URL never appears in output (mock HTTP,
  no real network in tests).
- Manual input parsing: `$1,240.18`, `1240.18`, `-1240.18`, empty → 0.

## 11. Done when

- `sweep --manual` works end to end with no account connected.
- `sweep` prints a correct breakdown from my real SimpleFIN data, and the card numbers match
  each card app's statement balance, less any payment already made.
- Adding or removing a card, payment or income is one command: `sweep add` / `sweep remove`.
- All tests pass with `pytest`.

## Out of scope

Moving money, budgets or categories, transaction history, and any web or phone UI. For quick
checks on my phone I already have a manual version of this calculator as a web page.
