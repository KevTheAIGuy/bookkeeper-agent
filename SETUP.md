# Setup

Written for someone who has never opened a terminal. If a step looks intimidating,
the honest answer is that you copy a line, paste it, and press Enter. That's it.

Set aside about 30 minutes for the first run. Every month after that takes ten.

**Everything stays on your computer.** No account to create, no bank login, no data
sent anywhere. The one exception is that Claude reads your statements in order to
categorize them, which is the whole job.

---

## Part 1 — Install the two things you need

### Python

Python is the language the calculator part is written in.

**Windows:** open the Microsoft Store, search "Python 3.12", click Install.
**Mac:** go to [python.org/downloads](https://www.python.org/downloads/), download,
open the file, click through.

To check it worked, open a terminal:
- Windows: press Start, type `powershell`, press Enter
- Mac: press Cmd+Space, type `terminal`, press Enter

Paste this and press Enter:

```
python --version
```

You should see something like `Python 3.12.10`. If Mac says "command not found",
try `python3 --version` instead, and use `python3` everywhere below.

### Claude Code

This is the agent that does the reading and thinking. Go to
[claude.com/code](https://claude.com/code) and follow the install steps for your
computer. You'll need a Claude account.

---

## Part 2 — Get this project onto your computer

Easiest way, no Git required:

1. Go to the project's GitHub page
2. Click the green **Code** button, then **Download ZIP**
3. Unzip it somewhere you'll remember, like your Desktop
4. Rename the folder to `Bookkeeper` if you like

Now open a terminal **in that folder**:
- **Windows:** open the folder, click the address bar at the top, type `powershell`,
  press Enter
- **Mac:** right-click the folder, choose "New Terminal at Folder"

Install the two helper libraries:

```
pip install openpyxl pypdfium2
```

`openpyxl` writes the Excel file. `pypdfium2` turns PDF statement pages into images
so Claude can read them accurately.

---

## Part 3 — See it work before you trust it with anything real

```
python books.py sample
python books.py report
```

Open the `out` folder and double-click `business-dashboard.html`. That's the
finished product, running on invented example data.

Click around. Open the **Chart of accounts** section and click an account row to see
the register underneath. Try the **Export** dropdown and download a month.

When you're satisfied it does what you want, wipe the examples:

```
python books.py reset
```

That deletes the example ledger and the generated reports. Your rules and chart of
accounts stay.

---

## Part 4 — Install the agent

The engine does arithmetic. The **agent** does the reading and judgment. Install it
by copying the `skill` folder's contents into Claude Code's skills directory:

**Windows**
```
xcopy /E /I skill\books "%USERPROFILE%\.claude\skills\books"
```

**Mac**
```
mkdir -p ~/.claude/skills && cp -R skill/books ~/.claude/skills/
```

Then open the project folder in Claude Code and type `/books`. If it responds, you're
installed.

Tell it where the project lives the first time:

> My books project is at [paste the full folder path here]

---

## Part 5 — Set up your own accounts

Talk to Claude Code in plain English. You don't edit files by hand unless you want to.

Say something like:

> I want to set up my books. I have a business checking account at Chase, a business
> credit card at Amex, and a personal checking account at Wells Fargo. My business is
> a single-member LLC doing freelance design.

Claude will update `chart_of_accounts.json` for you. Two things it will get right and
you should know about anyway:

**Name your credit card accounts with the word `card` in them.** Like `amex-card` or
`chase-card`. There's a rule that catches credit-card payments and it looks for that
word. Get this wrong and your card payments get counted as expenses on top of the
purchases they paid for, which inflates your spending by the amount of every card
payment you make.

**Business and personal stay separate.** Two ledgers, two dashboards, two
spreadsheets. Your tax return only cares about the business one, and mixing them is
the thing that makes an accountant charge you more.

---

## Part 6 — Feed it your statements

### Where to put things

```
inbox/
  business/    business bank and credit-card statements
  personal/    personal bank and credit-card statements
  receipts/    photos and PDFs of receipts, either kind
```

Just drag files in. Any name is fine.

### Getting statements out of your bank

Log into your bank's website (not the phone app, which usually can't download) and
look for **Statements** or **Documents**. Download the month you want.

**CSV is better than PDF if your bank offers it.** Look for "Export", "Download
transactions", or "Download as spreadsheet". A CSV is exact. A PDF has to be read
visually, which works well but takes longer.

Do this for every account you have, including credit cards. **Missing one account is
the single most common way these books end up wrong**, because the total looks
plausible and nothing tells you a chunk is absent.

### Receipts

Photograph them or save the PDF, drop them in `inbox/receipts/`. Legible is enough,
it doesn't need to be straight or cropped.

A receipt is **evidence attached to a line on your statement**, never its own entry.
That's how you avoid counting the same lunch twice. If a receipt matches nothing on
any statement, you paid cash, and it gets flagged so you can see it instead of losing
it.

### Getting receipts out of your email

Three ways, easiest first.

**1. Forward them.** When a receipt arrives, forward it to yourself with the subject
`RECEIPT`. Once a month, open that search in your email, download the attachments
into `inbox/receipts/`, done. Nothing to configure, nothing gets access to your inbox.

**2. Have your email file them automatically.** In Gmail, click the search box, then
the sliders icon on the right. Put `receipt OR invoice OR "order confirmation"` in
"Has the words", check "Has attachment", click Create filter, and choose "Apply the
label" with a new label called `Receipts`. Every receipt sorts itself from then on.
Once a month you open that label and save the attachments.

**3. Let Claude search your email directly.** Claude can connect to Gmail, and once
connected you can say:

> Search my email for receipts and invoices from last month, and save the attachments
> into inbox/receipts/

To turn it on, go to your Claude connector settings and enable the Gmail connector,
then approve the permission prompt. Claude Code will pick it up.

Worth knowing before you enable it: this gives Claude read access to your mail, which
is broader than the bookkeeping job strictly needs. Option 1 or 2 keeps that access
closed and costs you about five minutes a month. Either is a legitimate choice. The
agent is instructed to only pull receipts and invoices, but the permission itself is
wider than that instruction, and you should decide with that in mind rather than
assume the instruction is the boundary.

Whichever route you pick, email is a **supplement**. Your statements are the truth.
Plenty of spending never generates an email at all.

---

## Part 7 — Close your first month

In Claude Code:

> Close my books for October

It will:

1. List the files it found and tell you which accounts and periods they cover
2. Read them, rendering PDFs to images rather than extracting text, because text
   extraction silently scrambles amounts on multi-column statement layouts
3. Load every transaction, skipping duplicates automatically
4. Categorize everything it recognizes
5. **Ask you about what it doesn't recognize**, in one batch
6. Match receipts to statement lines
7. Check for double-counted movements
8. Build your dashboard, spreadsheet, and CSV

### About step 5

The first month you might answer twenty questions. The second month, three. Every
answer gets written into `rules.json` so a vendor is asked about exactly once, ever.

Answer honestly rather than quickly. A wrong answer becomes a permanent rule, and
wrong-but-categorized is the error that survives, because nothing flags it afterward.

### Then

Open `out/business-dashboard.html`. Read the **Needs your attention** section first,
before the pretty numbers. That's where anything untrustworthy is listed.

---

## Every month after

1. Drag new statements and receipts into `inbox/`
2. Say "close my books for [month]"
3. Answer any questions
4. Open the dashboard

Do it monthly. Skip three months and the review queue gets big enough that you
won't do it at all, which is how most bookkeeping systems die.

---

## Getting it to your accountant

Open the dashboard, use the **Export** dropdown, pick **Full year**, click Download
CSV. Send them that plus `out/business-books-*.xlsx`.

The spreadsheet has a **Schedule C** sheet mapped to the actual tax form lines, and a
**Chart of Accounts** sheet showing what counts as income, what counts as an expense,
and what's excluded as a transfer. That's the part that saves them time and you money.

---

## Things worth understanding

**Transfers are not income or expenses.** Moving $500 from checking to savings, or
paying your credit card, or taking money out of the ATM, is one dollar moving, not a
dollar earned or spent. The system excludes all of it from your totals automatically.
If it didn't, your books would drift thousands of dollars off.

**A credit card purchase is the expense. Paying the card is not.** You already spent
the money when you swiped. Paying the bill just moves the debt. Count both and every
card purchase lands in your books twice.

**Cash is the blind spot.** Money you withdraw and spend leaves no record anywhere.
That's why unmatched receipts get flagged rather than dropped.

**Deductions are not automatically free.** If you claim the Earned Income Credit or
the refundable Child Tax Credit, those are curves, not lines. Past a certain point,
more business deductions lower your income and lower your credit along with it, and
you can end up worse off. If your refund is large relative to what you earned, ask
your tax preparer to model the credit before you go hunting for write-offs.

**This is not tax advice, and it does not file anything.** It organizes your records
and maps them to the right form lines so a human professional can work faster. Have
an actual accountant review it before anything gets filed.

---

## When something goes wrong

**"python is not recognized"** — Python didn't install, or the terminal was open
before you installed it. Close the terminal, open a new one, try again.

**"No module named openpyxl"** — run `pip install openpyxl pypdfium2` again and read
the output for errors.

**Numbers look too high** — you probably have a credit card whose account name is
missing the word `card`, so its payments are being counted as spending. Run
`python books.py audit` and it will show you the pairs.

**Numbers look too low** — an account is missing. Check that every bank and card
statement for the period is in `inbox/`.

**A statement won't read** — it may be a scan or password-protected. Ask your bank
for a CSV export instead.

**Ask the agent.** "Something looks wrong with my October numbers, walk me through
where they came from" is a legitimate thing to type, and it can open the register and
show you every transaction behind any figure.
