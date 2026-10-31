# savings-sweep

Tells me how much I can move to savings (or spend) today. It walks the coming days in date
order: checking, plus each paycheck on the day it lands, minus each card's unpaid statement
balance and each other payment on its due date.

```
move to savings today = lowest balance checking reaches on any of those days − cushion
```

A paycheck only covers bills due after it lands. When it lands too late for some of them, part
of it frees up later, and it says how much and on which day.

- `web/calculator.html`: the phone calculator. Open it in any browser and type in balances,
  due days and paychecks.
- `SPEC.md`: the build spec for a command-line version that fetches balances automatically.
  Give it to Claude Code: "Read SPEC.md and build it."
