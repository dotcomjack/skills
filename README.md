# DotcomJack rules for Claude Code

Two skills, 42 rules, 19 of them carrying the measured failure or run behind them. They are the verification rules dotcomjack.com and a few dozen products are built under, exported from the live rulebook, not rewritten for show.

## Install

```sh
npx skills add dotcomjack/skills
```

or, in Claude Code:

```
/plugin marketplace add dotcomjack/skills
/plugin install dotcomjack-rules@dotcomjack
```

## What is in it

- **prove-it**: no claim of done, fixed or verified without an artifact. 27 rules, 12 with the measured failure behind them.
- **tests-ran**: a green run with zero tests executed is a failure, measured per runner (vitest, jest, node --test, pytest, xcodebuild). 15 rules, 7 measured on real repos.

## Why these

Every rule here cost a real round first: a review that ran once and missed the regression its own fix added, a speed fix measured on staging while production sat slow for two weeks, a suite that exited 0 having run nothing. The rule is what made it not happen twice.

More, and the record they came from: https://dotcomjack.com/skills#kit

MIT licensed.
