# Savings Sweep: build spec

> **To Claude Code on my laptop:** read this whole file, then build it in a new folder
> (not inside the coffee shop project). Start with manual mode (section 5) so I can use it
> right away, then add SimpleFIN. Ask me before anything that needs my bank login or a token.

## 1. Problem

I'm paid monthly. At the start of each month I want to move whatever is left of last month's
pay into savings. Rent and my Bank of America card are paid by the 1st. My two Amex cards and
my Amazon credit account are paid later (around the 9th and the 12th), so on the 1st my
checking account still holds money that is already spoken for. Working out each card's
statement balance by hand is tedious and error-prone.

## 2. The rule

```
safe_to_move = checking_available
             − sum(current balance of every card)
             − sum(other payments made from checking before the next payday)
             − cushion
```

- Use each card's **current balance** (posted + pending charges), **not** the statement
  balance. The current balance includes every unpaid charge, whether it landed on the last
  statement or came after it. This treats every card like a debit card: a charge counts against
  checking the moment it's made, so statement closing dates never matter.
- Cards already paid (Bank of America, usually Bilt) simply show $0.
- The money left in checking is what the Amex/Amazon autopays (statement balance) draw from on
  the 9th/12th. Charges made after those statements closed stay covered until next month.
- Three outcomes:
  - `ok`: result > 0. Move that amount.
  - `cushion`: checking covers cards and other payments, but not the cushion. Move nothing.
  - `short`: checking can't cover cards and other payments. Show the shortfall prominently.

## 3. My accounts

| Account               | Role     | Paid                     | Notes                                                  |
|-----------------------|----------|--------------------------|--------------------------------------------------------|
| Checking              | checking | n/a                      | Where my salary lands. Use the *available* balance.    |
| Bank of America card  | card     | by the 1st               | Usually $0 at sweep time.                              |
| Amex Platinum         | card     | ~9th or ~12th (confirm)  |                                                        |
| Amex Gold             | card     | ~9th or ~12th (confirm)  | I'm cancelling one of the two Amex cards in March 2027. |
| Amazon credit account | card     | after the 1st            | Issuer may be Synchrony. Might not connect through an aggregator, so it may need manual entry. |
| Bilt                  | card     | rent only                | Include it anyway; normally $0.                        |

Ask me each card's exact due day before writing my real config.

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

sweep list                                           # show checking, cards and payments
sweep add card "Chase Sapphire"                      # add a card (prompts to link it, see below)
sweep add payment "Car loan" --amount 320 --due-day 15
sweep rename "Amex Gold" "Amex Gold (old)"
sweep remove "Amex Gold"                             # asks to confirm first
```

**Payment destinations** are everything checking still has to pay. There are two kinds:

- `card`: the balance changes, so it's fetched (or typed in) on every run.
- `payment`: a fixed amount on a due day, paid straight from checking (loan, utility, rent
  that hasn't cleared).

`sweep add card` lists the SimpleFIN accounts not yet in config and lets me pick one, or choose
"enter by hand" (`source = "manual"`). `add`, `rename` and `remove` edit `config.toml` with
`tomlkit` so my comments and ordering survive, and write a `config.toml.bak` first. Name
matching is case-insensitive. An unknown name lists the names that do exist.

### Example output

```
Savings sweep · Sun Nov 1, 2026          balances as of Nov 1, 8:12 AM

Checking                                   6,850.00
Cards
  Bank of America                              0.00
  Amex Platinum                            1,240.18
  Amex Gold                                  386.40
  Amazon credit (entered by hand)            212.75
  Bilt                                         0.00
                                          −1,839.33
Other payments before next payday
  Utilities (Nov 15)                         −95.00
Cushion                                     −500.00
───────────────────────────────────────────────────
Move to savings                            4,415.67
```

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
payday  = 1          # day of month the salary lands; drives the payments window

[[accounts]]
name = "Checking"
role = "checking"
simplefin_id = "ACT-..."       # copied from `sweep accounts`

[[accounts]]
name = "Amex Platinum"
role = "card"
simplefin_id = "ACT-..."

[[accounts]]
name = "Amazon credit"
role = "card"
source = "manual"              # not connected; prompt for it each run

[[accounts]]
name = "Old savings"
role = "ignore"                # shown by `sweep accounts`, never counted

[[payments]]
name = "Utilities"
amount = 95.00
due_day = 15
```

Optional per-account keys: `invert_sign = true` (see 6), `add_pending = true` (see 6).

### Calculation details (`calc.py`)

- Money is `Decimal` (or integer cents) everywhere. Never `float`.
- Card owed amount is never negative: a card with a credit balance counts as $0 owed.
- Payments window: include a payment if its next due date falls on or after today and before the
  next payday. Clamp `due_day` / `payday` to the month's last day (e.g. 31 → Feb 28/29).
  A payment due today may already have cleared; support `--skip NAME` to drop it for this run.
- Return a result object with every line item, the totals, `status` (`ok`/`cushion`/`short`)
  and the amount, so the CLI and `--json` share one source of truth.

## 5. Phase 1: manual mode (build first)

`sweep --manual` prompts for checking and each card in config order, accepts input like
`$1,240.18` or `-1240.18` (a minus sign on a card is ignored), then prints the same output.
Accounts with `source = "manual"` use these prompts even in normal mode.

## 6. Phase 2: automatic balances via SimpleFIN Bridge

SimpleFIN Bridge is a read-only bank aggregator for personal use with a small yearly fee. I
connect my banks on their site and get a setup token. **Read the protocol at
https://www.simplefin.org/protocol.html before writing the client** and follow it over anything
here. In outline:

1. `sweep setup`: I paste the setup token. It's base64; decode it to a claim URL, `POST` to it
   once, and get back an access URL with credentials embedded. Store the access URL with
   `keyring` (macOS Keychain / Windows Credential Manager). Never print or log it.
2. Fetch `GET {access_url}/accounts`. Use the protocol's balances-only option if it exists;
   otherwise a short `start-date` window to keep the response small. The bridge is rate-limited
   (roughly a day's worth of requests), so cache the last response for 15 minutes.
3. Each account has an id, name, org, `balance`, `available-balance` and `balance-date`
   (unix time). Use `available-balance` for checking when present.
4. **Sign convention:** card balances usually come back negative when I owe money. Don't assume
   it. Owed = `-balance` when negative, 0 otherwise, with `invert_sign = true` per account for
   institutions that report it the other way. `sweep accounts` must show raw balances so I can
   compare against my bank apps on the first run.
5. **Pending charges:** some institutions leave them out of the balance. With
   `add_pending = true`, fetch with `pending=1` and add that account's pending charges.
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
- Result `short`: print the shortfall and which upcoming payments it threatens.

## 8. Phase 3 (optional): run itself on the 1st and ping my phone

- `sweep --json | sweep notify` sends a one-line push ("Move $4,415.67 to savings") via
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

- `calc`: the example above gives exactly 4,415.67; `cushion` and `short` states; credit
  balance on a card counts as $0; Decimal rounding (no float drift on values like 0.1 + 0.2).
- Payments window: today Nov 1 with payday 1 includes a payment due Nov 15 and excludes Dec 2;
  today Jan 30 with payday 1 includes a payment due Jan 31; `due_day = 31` in February.
- Config: unknown role rejected; a missing `simplefin_id` without `source = "manual"` rejected.
- `add` / `rename` / `remove`: round-trip a config with comments and keep them; duplicate
  names rejected; removing an unknown name changes nothing; a `.bak` is written first.
- SimpleFIN: parse the fixture JSON; sign normalization with and without `invert_sign`;
  stale-balance warning; `errors` array surfaced; access URL never appears in output (mock HTTP,
  no real network in tests).
- Manual input parsing: `$1,240.18`, `1240.18`, `-1240.18`, empty → 0.

## 11. Done when

- `sweep --manual` works end to end with no account connected.
- `sweep` prints a correct breakdown from my real SimpleFIN data, and the card numbers match
  each card app's current balance.
- Adding or removing a card or payment is one command: `sweep add` / `sweep remove`.
- All tests pass with `pytest`.

## Out of scope

Moving money, budgets or categories, transaction history, and any web or phone UI. For quick
checks on my phone I already have a manual version of this calculator as a web page.
