# Book-keeper Agent

A bookkeeping agent you run on your own computer. Drop in bank statements, credit-card
statements, and receipts. Get back a visual dashboard, an accountant-ready Excel
workbook, and a CSV for any month or the whole year.

Built for [Claude Code](https://claude.com/code). Your data never leaves your machine.

**Never touched a terminal? Read [SETUP.md](SETUP.md) instead of this file.** It's
written for that, step by step.

---

## What it actually does

- Reads bank and credit-card statements, PDF or CSV
- Files every transaction into a real **chart of accounts**, QuickBooks-style, with
  account numbers, types, and Schedule C lines
- Attaches receipts to the statement line they document, never as a separate entry
- Refuses to double-count: card payments, ATM cash, and internal transfers are
  excluded from your totals by construction, not by remembering
- Asks you about vendors it doesn't recognize, once each, then never again
- Produces a self-contained HTML dashboard with a drill-down register, a multi-sheet
  Excel workbook, and CSV exports by month or year

Two ledgers, kept separate: **business** and **personal**. Your tax return only cares
about one of them.

---

## Quick start

```bash
pip install openpyxl pypdfium2
python books.py sample      # invented example data
python books.py report      # writes to out/
```

Open `out/business-dashboard.html`.

Then install the agent so it can read your real statements:

```bash
# macOS / Linux
mkdir -p ~/.claude/skills && cp -R skill/books ~/.claude/skills/

# Windows
xcopy /E /I skill\books "%USERPROFILE%\.claude\skills\books"
```

Wipe the examples with `python books.py reset --yes`, then say `/books` in Claude Code.

---

## The design

**The engine does arithmetic. The agent does judgment.** `books.py` owns storage,
deduplication, math, and rendering. The `/books` skill owns reading a statement,
deciding what an unfamiliar vendor is, and matching a receipt to its line. Code that
guesses categories is wrong silently. An agent doing arithmetic is wrong silently.
Neither one crosses over.

### Chart of accounts

`chart_of_accounts.json` is a real one. Each account has a number, a type, a detail
type, and (for business expenses) a Schedule C line.

```
1000  Asset       bank accounts, cash on hand
2000  Liability   credit cards
3000  Equity      owner draw/contribution, family transfers
4000  Income
5000  COGS        direct cost of delivering the work
6000  Expense     business overhead
7000  Expense     personal
x990              parking lot per block, kept visible on purpose
```

**The account's type decides whether it counts.** Income counts as income,
Expense and COGS count as expense, and Asset/Liability/Equity are transfers that never
touch a total. That's why paying a credit card is neutral without anyone having to
remember to mark it: it lands on a Liability account.

`rules.json` only says which account a vendor hits. The kind, the account name, and
the Schedule C line all come from the chart, so a vendor can't disagree with its own
account, and changing a Schedule C line changes it everywhere at once. A rule pointing
at an account that doesn't exist is a hard error, not a silent drop.

### Why it doesn't double-count

Four independent layers, because this is the failure mode that quietly ruins
homemade books:

1. **Identity hash** on entity + account + date + amount + description, plus an
   occurrence index. Re-importing a statement, or loading two overlapping periods,
   changes nothing. Two identical $5 charges on one day both survive.
2. **Account type.** Card payments and internal transfers land on balance-sheet
   accounts, which are excluded from every total by definition.
3. **Summary-line guard.** `load` rejects "Previous balance", "Total purchases",
   "New balance" and friends. One of those getting in inflates a whole month.
4. **`books.py audit`.** Flags equal-and-opposite pairs across two accounts within six
   days where a side isn't already a transfer.

The audit **flags and never auto-fixes**. Two unrelated $500 movements in one week are
possible, and silently rewriting your books to resolve one would be worse than asking.

### Reading statements

**Render and read, never text-extract.** PDF pages get rendered to images via
`pypdfium2` and read visually. Text extraction on multi-column statement layouts
misassigns amounts to the wrong rows, and does it differently depending on the flags
you pass, so the output looks plausible and is wrong. CSV exports are parsed directly.

---

## Commands

| Command | What it does |
|---|---|
| `python books.py init` | create the database and folders |
| `python books.py load rows.json` | insert transactions, then auto-categorize |
| `python books.py receipts recs.json` | file receipts, match them to statement lines |
| `python books.py recat [--all]` | re-apply `rules.json` after editing it |
| `python books.py review` | JSON list of what needs a human decision |
| `python books.py audit` | look for double-counted movements |
| `python books.py report` | dashboard + xlsx + csv, per entity, into `out/` |
| `python books.py sample [--income N]` | load example data |
| `python books.py reset --yes` | wipe the ledger, keep rules and accounts |
| `python books.py selftest` | assert the money logic still holds |

You rarely type these. The agent does. They're here so you can check its work.

---

## Files

```
books.py                  the engine, one file
chart_of_accounts.json    your accounts: number, type, Schedule C line
rules.json                vendor -> account. The agent appends as it learns.
skill/books/SKILL.md      the agent
books.db                  your ledger (gitignored)
inbox/                    drop statements and receipts here (gitignored)
out/                      dashboards, workbooks, CSVs (gitignored)
```

Your ledger, statements, and reports are gitignored. Nothing financial gets committed
if you fork this.

---

## Requirements

Python 3.10+, `openpyxl`, `pypdfium2`. Claude Code for the agent half. No server, no
account, no API key, no bank login.

---

## Limits, stated plainly

- **You are the bank feed.** No Plaid, no automatic sync. You download statements and
  drop them in. If you skip months, this produces confident, incomplete books, which
  is worse than none because you'll trust them.
- **Not tax advice and it files nothing.** It organizes records and maps them to form
  lines so a human professional works faster. Have an accountant review before filing.
- **Deductions are not automatically free.** If you claim refundable credits (EITC,
  the refundable Child Tax Credit), those are curves, not lines. Past the plateau,
  more deductions lower your AGI and lower the credit with it. Model the credit before
  chasing write-offs.
- **Cash spending is invisible** unless you keep the receipt. Unmatched receipts get
  flagged rather than dropped, which is the best a system can do here.
- **Single-entry, cash-basis.** Not double-entry accrual accounting. It's built for a
  sole proprietor or single-member LLC who files a Schedule C.

---

## License

MIT. Do what you like with it.

The example data in `books.py` is invented. Every account, amount, vendor
relationship, and counterparty in it is fictional.
