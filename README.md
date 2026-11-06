I built a small group-finance model because adding companies' accounts together gave me totals that counted internal bills twice. I wanted to see what the group earned from outside customers, even when each company kept its books in a different currency.

I load a Canadian parent and three subsidiaries, convert their balances into Canadian dollars, and remove bills and amounts owed between them. The result is an annual statement pack with a balance sheet, income statement, cash flows, changes in equity, prior-year comparisons and explanatory notes, plus a month-end checklist. I also roll up three years of Base and Downside plans into CSV files that Power BI can import. All names, transactions, plans and exchange rates are synthetic.

To try it, clone this repository and run the example with **Python 3.11 or newer**:

```sh
git clone https://github.com/jackaI1/ifrs-consolidation-model.git
cd ifrs-consolidation-model
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e '.[dev]'
ifrs-close --statutory --input examples/statutory_group --output output/first-close
```

Open `output/first-close/annual_financial_statements.md` to read the complete example pack, or open its `annual_*.csv` tables in a spreadsheet. My [generated example report](examples/statutory_group/report/annual_financial_statements.md) is also readable on GitHub without installing anything. I use a new output folder for each run so an older close cannot look like a successful new one. A blocked close exits with code 1 and writes diagnostics instead of statements.

I use a fictional Canadian federal service company that has chosen IFRS within Canadian accounting rules. The annual pack covers the years ended 31 October 2026 and 2025:

| Result in Canadian dollars | 2026 | 2025 |
| --- | ---: | ---: |
| Revenue from outside the group | 41,950 | 17,200 |
| Profit after tax | 11,381.50 | 4,400 |
| Assets and liabilities plus equity | 69,456.50 | 54,400 |
| Operating cash flow | 11,381.50 | 4,400 |
| Currency translation movement in OCI | 3,675 | 0 |

Cash moves from 54,400 to 69,456.50 through operating cash of 11,381.50 and a separate currency effect of 3,675. Equity moves by the same profit and OCI amounts. Those links are checked against the source ledgers, cash movements and opening reserves before anything is published.

OCI means other comprehensive income: changes shown separately from ordinary profit. The translation reserve is the accumulated currency effect in equity; the current period's movement goes to OCI. I keep those amounts separate so a change in exchange rates does not look like extra sales.

### What I put in and get out

My [annual example folder](examples/statutory_group) contains current, comparative and opening ledgers, their exchange rates, cash and equity movement schedules, and disclosure inputs. A ledger is a list of account balances whose debits and credits add to zero. The parent reports in CAD; the three subsidiaries use USD, EUR and GBP. `plan.csv` adds annual income and expense forecasts for 2027 to 2029.

I keep a [workpaper example](examples/three_subs) for a single October close with different average and closing rates. It is useful for seeing the currency translation mechanics on their own:

```sh
ifrs-close --input examples/three_subs --output output/monthly-close
```

That smaller example still produces CAD 15,040 profit, CAD 73,125 assets and CAD 2,885 translation OCI. It does not include the additional annual statement inputs. My [annual reporting guide](docs/statutory.md) explains the Canadian reporting basis, supported facts and independent hand calculations.
The output also includes a translated ledger showing each rate used, elimination journals showing every removal, and a translation-reserve roll-forward showing opening balance, movement and closing balance. `status.json` records input-file hashes and whether the automated checks passed. `fact_financials.csv` and four dimension tables are the Power BI import set. My [Power BI guide](docs/power-bi.md) explains the relationships, signs and filters that prevent counting the same amounts twice.

To change the example, I copy the input folder and edit the CSVs and rates together. My [input and accounting guide](docs/accounting.md) explains the columns, assumptions, and hand calculations. Missing rates, unbalanced ledgers and unmatched intercompany items stop the whole pack, including the plans. I fix the source instead of adding a balancing amount.

### Accounting choices and boundaries

I translate assets and liabilities at the period-end rate, income and expenses at an average rate, and opening equity at supplied historical rates. Historical opening retained earnings must reproduce the prior translated balance. The translation reserve makes the translated statement balance only after the local ledger has already passed its own balance check. I supply the opening reserve explicitly and report its movement in OCI.

For intercompany eliminations, I require two reciprocal entries with the same reference, the right account pair, and exactly matching amounts after translation. I remove both sides through a journal. I also offset the parent's investment against each subsidiary's original capital while keeping post-formation retained earnings in group equity.

I follow the relevant mechanics in IAS 21, the foreign-currency translation standard, and IFRS 10, the consolidation standard. **Workpaper mode produces IFRS-style workpapers, not a complete statutory filing.** Annual mode adds comparative statements, current/non-current presentation, cash flows, equity movements and notes for the supported Canadian service group. My model supports wholly owned subsidiaries held from formation, non-hyperinflationary currencies and internal services consumed in the period. Average rates are appropriate only when they reasonably approximate transaction-date rates. I do not handle acquisitions, goodwill, minority ownership, unrealised inventory profit or tax adjustments automatically. The annual profile has no external working capital, fixed assets, leases, borrowing or owner distributions; unsupported material accounts and movements stop that pack. A real company still needs professional review, board authorisation and any required audit or filing. The synthetic report is not filed or audited. The checklist keeps five human review items open even when every automated check passes.

### Code and checks

I keep the calculation in `src/ifrs_consolidation/model.py`, the annual presentation and cross-period checks in `annual.py`, and the CSV interface in `cli.py`. The engine uses decimal arithmetic without rounding amounts before reconciliation. Plans use the same translation and reciprocal matching logic, scoped independently to each scenario and year. They are income-only forecasts, with fixed budget exchange rates in this example, rather than projected balance sheets.

```sh
python -m pytest -q
ruff check .
```

The suite has **75 passing tests**, including hand-calculated statements, historical equity, opening reserves, elimination journals, missing rates, one-cent mismatches, malformed files and the full command-line export. The annual pack has **99 passing automated checks**, including reconciled opening and comparative ledgers, direct cash flows, tax notes and equity carry-forwards. The original workpaper pack keeps its **26 passing automated checks** across one actual close and six plan groups. GitHub Actions runs the suite and example on Python 3.11 and 3.13, and saves the generated pack as a downloadable artifact. There are no runtime dependencies outside Python's standard library.
