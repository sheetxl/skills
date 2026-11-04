# Financial models: how the standard ones are built

When you build, complete or audit a financial model, its type has conventions a reviewer expects —
the lines, their order, their signs and units, where a balance starts and ends. Identify the model
type from its sheets and labels, read its section below, then follow the workbook's own layout: the
page says what each line computes, the workbook says where it goes and how it is written. Where the
two disagree, the workbook's existing rows win.

When auditing, a line that breaks its type's convention is a candidate, not a verdict: if the
model breaks it the same way everywhere, that is the model's own convention; a line that breaks it
where its siblings keep it is the finding.

| model | read | recognize it by |
| --- | --- | --- |
| Bond / debt accounting | [debt accounting](#debt-accounting-bonds-under-the-effective-interest-method) | amortization schedule, effective interest, discount/premium/zero-coupon/OID, carrying value, deferred tax on a bond |
| Debt schedule, cash sweep, LBO | [debt schedules](#debt-schedules-cash-sweeps-and-lbos) | tranches with beginning and ending balances, revolver, term loans, mandatory and voluntary repayment, cash available for debt repayment |
| Acquisitions | [acquisitions](#acquisitions-consolidation-equity-method-purchase-price-and-accretion) | consolidation worksheet, NCI, goodwill, equity method, investment in affiliate, purchase price allocation, EPS accretion or dilution, synergies vs premium |
| Real estate | [real estate](#real-estate-rent-roll-revenue-build-noi-and-valuation) | rent roll, units and SF, vacancy and loss-to-lease, EGI, NOI, cap rate, direct cap, reversion |
| Corporate: three statements, ratios, valuation | [corporate models](#corporate-models-three-statements-ratios-and-valuation) | income statement, balance sheet, cash flow, ratio analysis, DCF, WACC, cost of equity, comps, EV/EBITDA, sensitivity table, FY columns |

Conventions that hold across model types:

- A balance rolls forward: beginning = prior ending; ending = beginning + flows. Check the last
  period against what it must be (a repaid loan ends at 0; a balance sheet balances).
- Deductions are negative lines that subtotals add, or positive lines that subtotals subtract —
  whichever the model already does, never both.
- A new cell takes the sign of the same kind of line nearby: fees shown as negatives elsewhere on the
  sheet mean a new fee line is negative too. With nothing to match: operating expenses and taxes are
  positive; vacancy, loss-to-lease and concessions are negative; repayments are negative. Reading a
  line from another schedule can need a minus, so compare how the two places store it.
- In a template, a 0 typed across a block that should be calculated is a placeholder for the cell to
  fill, not a value to keep: a total or roll-forward of zeros above real inputs is still to be built.
- A note on the sheet that deductions are negative binds vacancy and other loss lines, dividends and
  interest, even where a formula elsewhere would be simpler with a positive. Tax and operating
  expense rows often stay positive under the same note; a line whose neighbors show its sign follows
  them.
- Write a 0 where the convention gives 0 (a deferred tax under a book-equals-tax treatment); leave a
  cell empty only where the line does not apply.

## Debt accounting: bonds under the effective-interest method

How a bond (or any debt carried at amortized cost) schedule is conventionally built: discount,
premium, zero-coupon and original-issue-discount (OID) bonds, with the book–tax split and early
extinguishment. Read it before filling one; the model's own labels and signs still win where they
say otherwise.

### The schedule

| line | formula | note |
| --- | --- | --- |
| Effective (implied) rate | `=IRR(cash flows)` of the issuer's flows: proceeds in, coupons and principal out | when the model lists the flows; otherwise the market rate it states |
| Beginning balance | first period: the issue price (proceeds); then the prior period's ending | |
| Interest expense | rate × beginning balance | |
| Cash coupon | face × coupon rate (0 for a zero-coupon bond) | |
| Amortization | interest expense − cash coupon | positive for a discount (the balance accretes up to face), negative for a premium |
| Principal repaid | face (or the extinguishment price) in the period it is paid, else 0 | |
| Change | amortization | in the last period, see below |
| Ending balance | beginning + change | |

The last period has two layouts, and a model's notes or existing rows decide between them. With
nothing to go on, a discount or zero-coupon bond usually ends at face: its last change is the
amortization and the repayment sits on its own line. A premium bond usually ends at 0: its last
change takes out the whole balance (`=-beginning`).

Worked example (invented): proceeds 9,600, face 10,000, 3-year, coupon 2%, so the rate is the IRR
of +9,600, −200, −200, −10,200: 3.426%. Year 1: interest 328.9, coupon 200, amortization 128.9,
ending 9,728.9. Year 3: beginning 9,862.2, interest 337.8, amortization 137.8, ending 10,000, and
the 10,000 repayment on its own line.

### Early extinguishment

In the period the bond is retired early: gain or loss = carrying value at retirement − price paid
(a loss when the price is higher). The principal-repaid line carries the price paid (or the face,
with the premium in the gain/loss line, as the model lays it out), and the ending balance is 0.

### Income statement and taxes

- Interest expense on the income statement is the effective interest, not the coupon. A label like
  "Interest expense (amortization)" is the amortization part alone; "(coupon)" is the cash part.
- The tax side follows the treatment the task states. Typical statements:
  - Tax carries the bond at face (or at issue price) and amortization is not deductible each year:
    the yearly book–tax difference is the amortization; deferred tax = difference × tax rate, and
    it reverses when the bond is repaid.
  - Tax follows book: no difference, so deferred tax and the deferred-tax balance are 0 in every
    period — write the 0s, do not invent a difference.
- Deferred tax = −amortization × tax rate, and its direction picks the balance it builds:
  - A discount (issued below face, OID, zero-coupon) has positive amortization, so deferred tax is
    negative and builds a deferred tax **asset**: change in DTA = −deferred tax.
  - A premium (issued above face) has negative amortization, so deferred tax is positive and builds
    a deferred tax **liability**: change in DTL = deferred tax.
  - Where the model carries both balances, the other one stays 0 in every period.
- Cash taxes = book (GAAP) tax expense − deferred tax expense. The deferred-tax balance rolls
  forward: beginning = prior ending; ending = beginning + the period's change.

### Cash flow and balance sheet links

- Financing cash flows: proceeds in the issue period (positive), principal repaid in the repayment
  period (negative). Coupons are operating or financing as the model classifies interest.
- A "change in cash" line inside a tax balance-sheet block is the principal only: it holds the
  repayment in the repayment period and stays empty in the others. Coupons do not belong on it.
- Retained earnings roll forward by net income; the liability on the balance sheet is the schedule's
  ending balance.

## Debt schedules, cash sweeps and LBOs

How corporate debt is conventionally scheduled: a block per tranche, paid down from the cash the
model has available. Read it before filling one; the model's own labels and signs still win where
they say otherwise.

### A tranche

| line | formula | note |
| --- | --- | --- |
| Beginning balance | the prior period's ending | first period: the opening amount |
| Mandatory repayment | −original amount × scheduled % | stops at the balance |
| Voluntary repayment (sweep) | −MIN(cash available to this tranche, balance after mandatory) | never below 0 cash: `-MAX(0, MIN(...))` |
| Ending balance | beginning + repayments | |
| Interest | rate × beginning balance, unless the task or a label says average | average: rate × (beginning + ending) / 2 |

Repayments are negative and the ending balance adds them. A schedule whose ending row already
subtracts a positive "repayment amount" is the other convention: follow it.

### The waterfall

- Cash available = opening cash + cash flow − minimum cash, as the model's lines give it.
- The senior tranche takes first. Each junior tranche gets what is left: cash available + the
  (negative) repayments above it.
- Totals add the tranches. A coverage ratio with no interest to divide by is `IFERROR(..., "n/a")`
  when the model shows it that way.

### The sign of interest

Match interest lines the model already shows. With none to match, use the convention for the model:

| model | interest |
| --- | --- |
| Cash sweep or debt waterfall | positive: rate × balance, and cash available subtracts it |
| LBO or a debt schedule that feeds the statements or free cash flow | negative: `=-balance*rate`; interest income stays positive |
| Consolidation or normalization income statement | positive, as an expense the subtotal subtracts |
| An adjustment in an accretion or dilution bridge | negative: it reduces earnings |

Where capex or working capital are typed negative on the sheet, the schedule's interest is negative
too and the cash flow line links it as is: `=interest`, not `=-interest`. Only where a tranche block
already shows interest as a positive amount is the line that reads it `=-interest`.

### A revolver with interest on its average balance

Interest on the average balance makes the revolver depend on itself. Solve it instead of adding a
circular reference. With no sweep of other cash, and interest stored negative:

ending = (beginning × (1 + rate / 2) − cash flow before revolver interest) / (1 − rate / 2)

Worked example (invented): beginning 1,000, rate 6%, cash flow before revolver interest 200. Ending =
(1,000 × 1.03 − 200) / 0.97 = 855.67. Interest = −(1,000 + 855.67) / 2 × 6% = −55.67. Check: 1,000 −
200 + 55.67 = 855.67.

## Acquisitions: consolidation, equity method, purchase price and accretion

How a deal is conventionally carried through the numbers. Read it before filling one; the model's
own labels, signs and notes still win where they say otherwise.

### Consolidation worksheet

Columns run acquirer, target, adjustments, consolidated, and the consolidated column adds across.

| line | adjustment |
| --- | --- |
| Cash | −the cash consideration |
| Target's equity | eliminated: −target equity |
| NCI at fair value | consideration ÷ ownership % − consideration |
| Goodwill | implied total value (consideration ÷ ownership %) − fair value of net assets |
| Step-ups | fair value − book value of the asset |

The income statement adds the two companies line by line. Tax is rate × pretax income and is stored
positive; the NCI share is target net income × (1 − ownership %), also positive, and net income to
common subtracts it, even under a label starting "Less:". The NCI share belongs in the adjustments
column and the consolidated column; net income to common in the consolidated column only. Leave both
empty in the acquirer's and target's columns. Later balance sheets roll from the
acquisition date: cash + the change in cash; NCI + NCI's income − its dividends; equity + net income
to common − common dividends; other lines hold. Dividends are negative in cash flow from financing.

### Equity method

- Book value acquired = ownership % × target book value. Excess = price − book value acquired,
  split into identified intangibles and goodwill.
- Equity income = ownership % × affiliate net income − amortization of the intangibles (intangibles ÷
  life). It is one line on the investor's income statement.
- Investment in affiliate = cost + equity income − dividends received. The investor pays the purchase
  price in cash and adds it to the investment on the acquisition date. Where the balance sheet has
  its own intangibles and goodwill lines, the price splits across them on that date: investment =
  book value acquired, intangibles and goodwill = their allocated amounts.
- After the acquisition date, cash = prior cash + the period's net income and non-cash items as the
  supporting schedules give them, and intangibles and goodwill hold.
- Cash taxes on operations = operating profit × rate. Deferred tax on equity income = equity income ×
  (1 − dividends-received deduction) × rate. Total tax expense adds them.
- An adjustments column on the balance sheet is after − before, line by line.

### Purchase price allocation

- Deferred tax liability on the step-ups = −(asset step-ups) × tax rate: a **negative** line in the
  bridge, because it reduces net assets.
- Fair value of net assets = book value + step-ups + that negative liability.
- Goodwill = purchase price − fair value of net assets.
- Depreciation of a step-up = −step-up ÷ useful life, negative, and the asset's ending = beginning +
  that line.
- The liability's own schedule opens at the positive amount (`=-bridge line`) and falls each period by
  tax rate × the period's depreciation and amortization.

### Accretion and dilution

| line | formula | sign |
| --- | --- | --- |
| Synergies, after tax | synergies × (1 − tax rate) | + |
| Interest on acquisition debt, after tax | −debt × rate × (1 − tax rate) | − |
| Interest lost on cash used, after tax | −cash × rate × (1 − tax rate) | − |
| Combined net income | sum of both companies' net income and the lines above | |
| New shares | stock-funded value ÷ acquirer's share price; stock-funded value is the equity price less debt and cash used | |
| Pro forma EPS | combined net income ÷ (acquirer's shares + new shares) | |
| Accretion / (dilution) | pro forma EPS − standalone EPS; % is that ÷ standalone | |

A note naming whose tax rate applies decides it; otherwise use the acquirer's. Later years use the same
new share count.

### Synergies against the premium

Value the **after-tax** synergies. Terminal value = final year's after-tax synergy × (1 + growth) ÷
(discount rate − growth). Net value created = present value of synergies − premium paid, where the
premium is purchase price − unaffected market cap.

## Real estate: rent roll, revenue build, NOI and valuation

How an income-property model is conventionally built — multifamily or commercial — from the rent
roll to net operating income (NOI) to a direct-capitalization or DCF value. Read it before filling
one; the model's own labels, units and signs still win where they say otherwise.

### Units: monthly, annual, per unit, per square foot

- Rents are quoted **monthly per unit** (or per SF per year for commercial, as the label says).
  Annual revenue = monthly rent × units × 12. A line that drops the 12 is a month, not a year.
- A per-unit or per-SF column beside annual dollars is a ratio of that line: annual amount ÷ units,
  or ÷ (units × average SF), and ÷ 12 again when the column is monthly ("$/SF/mo", "per unit per
  month"). Read the column header for which; the same divisor applies to every line in the column,
  totals included. In a multifamily or rent-roll model, a column headed only `$/SF` or `/SF` is per
  SF per month: rents there are monthly, so it divides by 12 too.
- Percent inputs stay percents: a 4.5% vacancy is `0.045` or a reference to the input cell.

### The revenue build

| line | formula | sign |
| --- | --- | --- |
| Gross potential rent (market rent) | market rent × units × 12 | + |
| Loss-to-lease | −(market rent − in-place rent) × occupied units × 12, or −GPR × the stated rate | − |
| Vacancy | −(potential rent) × vacancy rate, or −vacant units × rent × 12 | − |
| Concessions, credit loss, non-revenue units | −amount or −rate × base | − |
| Net rental revenue | sum of the lines above | |
| Other income (parking, fees, reimbursements) | per unit or per space × count × 12 | + |
| Effective gross income (EGI) | net rental revenue + other income | |

Deductions are **negative lines** and the subtotals **add** them (`=SUM(...)` over the block). A
model that shows a deduction positive and subtracts it is the other convention; follow whichever
the existing rows use, never mix.

### Operating expenses and NOI

- Each expense line: per-unit or per-SF rate × count, a percent of EGI (management fee), or a base
  amount grown each year.
- Growth: year 1 of a projection is the base-year amount × (1 + growth); each year after is the
  prior year × (1 + growth). The base year itself is the historical or stated figure.
- NOI = EGI − total operating expenses. Capital items (reserves, tenant improvements, leasing
  commissions, debt service) sit **below** NOI unless the model puts them above.

### Valuation

- Direct capitalization: value = stabilized (year-1 or forward) NOI ÷ cap rate. Value per unit =
  value ÷ units; per SF = value ÷ SF.
- Purchase price from a going-in cap rate: price = NOI ÷ cap rate; total cost adds closing costs
  (price × (1 + cost %)).
- DCF: cash flow each year = NOI − capital items; terminal (reversion) value = the year after the
  hold's NOI ÷ exit cap rate (or the final year's NOI, as the model states), less selling costs, a
  negative line that net proceeds add;
  value = NPV of the cash flows plus the discounted reversion at the discount rate.
- Debt metrics: debt yield = NOI ÷ loan; DSCR = NOI ÷ annual debt service; LTV = loan ÷ value.

Worked example (invented): 50 units at $1,800/month: GPR = 1,800 × 50 × 12 = 1,080,000. Vacancy 5%:
−54,000. Other income $75/unit/month: 45,000. EGI = 1,071,000. Expenses 38% of EGI: −406,980. NOI =
664,020. At a 5.5% cap rate, value = 12,073,091, or 241,462 per unit; NOI per unit per month =
664,020 ÷ 50 ÷ 12 = 1,106.7.

## Corporate models: three statements, ratios and valuation

How a company model is conventionally completed — the three statements, the ratio sheet, the cost of
capital and the valuation bridge. Most of its lines have more than one textbook definition; the
reviewer expects the one below unless the label or the model's own rows say otherwise.

### Read the model's own definitions first

- Before writing a ratio, find where the model already computes its parts: an earlier column of the
  same row, a sibling ratio, the same label on another sheet. Reuse those rows — the debt in Debt /
  Equity is the debt the model uses in Debt / EBITDA and in its EV bridge; the share count in book
  value per share is the one its market cap uses.
- Complete a row across every period column its block carries, including a date header that extends
  a pattern (`=EOMONTH(prior,12)`) and a stub or pre-operation column the row's siblings fill.
- A line the task names but the row leaves blank is still yours to fill; a row the task does not
  name is filled only when its block's other rows are.

### Ratios: label → formula

| label | formula | note |
| --- | --- | --- |
| Debt / Equity, Debt Equity Ratio | (short-term + long-term borrowings) / total equity | "long-term debt to equity" only when the label says long-term |
| Debt / EBITDA | the same total borrowings / EBITDA | |
| Payout ratio | dividend per share / diluted EPS | diluted where the model shows both; or total dividends / net profit |
| Return on average assets | PAT / AVERAGE(opening, closing total assets) | PAT before minority interest: assets carry the minority's share. "Return on total assets" without "average": PAT / closing total assets |
| Return on (average) equity | PAT / (average) total equity | "attributable" or "after minority" in the label: net profit after minority / (equity − minority interest) |
| PAT margin | PAT / revenue | |
| Net (profit) margin | net profit after minority interest / revenue | the model's own line names decide between PAT and net profit |
| Book value per share | (total equity − minority interest − proposed dividend or preference capital, where the balance sheet lists them) / shares | the shares the model's market cap uses |
| EBITDA | EBIT + depreciation and amortization | or revenue − operating costs before D&A, as the income statement is laid out |
| Interest coverage | EBITDA (or EBIT, as labeled) / finance cost | |
| Receivables, payables, inventory as % of sales or COGS | balance / flow, on the **average** balance where the model's days rows average | a forecast % held at the average of the historical years shown |
| EV / EBITDA, P / E | enterprise value / EBITDA; price or market cap / earnings | |
| **Implied** EV / EBITDA, implied P / E | the valuation's own value: (target or average price × shares + debt + minority interest − cash − investments) / EBITDA | not each period's market price; "implied" means from the valuation. The plain multiple beside it uses each period's price; the implied one uses the valuation's single price in every period |

### Cost of equity and WACC

- Cost of equity (CAPM) = risk-free rate + beta × equity risk premium.
- Adjust it as the label says: a **discount** applied to the cost of equity compounds,
  `= CoE × (1 + discount)`; a **premium** (company-specific, size, liquidity) adds, `= CoE + premium`.
- Post-tax cost of debt = pre-tax cost × (1 − tax rate). WACC = E/(D+E) × Ke + D/(D+E) × Kd × (1 − t).

### Valuation bridge

- Equity value = enterprise value − total borrowings + cash and cash equivalents − minority interest.
  Cash includes every cash-like line the balance sheet lists: other bank balances, deposits and
  liquid current investments, not the cash line alone.
- Value per share = equity value / shares. A sensitivity table of "implied value" against WACC and
  growth tabulates what the valuation summary reports — per share when it reports a price.
- Terminal value (Gordon growth) = final-year FCF × (1 + g) / (WACC − g), discounted with the final
  year's factor.

### Schedules

- Depreciation at a stated rate = rate × (opening balance + capex) unless the model or the task gives
  another base (a half-year convention, a rate on opening only).
- Closing = opening + additions − depreciation; the next opening is this closing.

### Not meaningful values

Where a ratio is undefined or not meaningful — a zero or negative denominator, a near-zero base (a
loan repaid to a sliver, a growth base of a few units), a negative valuation multiple — write the
placeholder the model already uses (`NM`, `NA`, `N/A`) as text. Never leave the cell empty or let an
error show. For multiples: `=IF(IFERROR(x,"NA")<0,"NM",IFERROR(x,"NA"))`.

### Checks

A balance check = total assets − (total liabilities + equity), 0 when it balances, unless the model
already has a check written the other way; then match it. A cash check = closing cash on the cash
flow − cash on the balance sheet.

Worked example (invented): borrowings 120 short + 380 long, equity 1,250, minority 50, PAT 210,
assets 2,000 opening and 2,200 closing. Debt / Equity = 500 / 1,250 = 0.40. Return on average assets
= 210 / 2,100 = 10.0%. CoE: 7% + 1.1 × 6% = 13.6%; with a 10% liquidity discount, 13.6% × 1.10 =
14.96%, plus a 2% specific premium, 16.96%. EV 3,000, cash 140, other bank balances 60: equity value
= 3,000 − 500 + 200 − 50 = 2,650.
