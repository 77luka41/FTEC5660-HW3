# FTEC5660 Homework 3: Fisher's 15-Point Investment Agent

Build a LangChain agent that researches a company with tools and applies the
framework from Philip Fisher's *Common Stocks and Uncommon Profits* (Part One,
Chapter 3 "What to buy: the fifteen points to look for in a common stock", and
Chapter 4 "What to buy: applying this to your own needs"). You choose 3 of
Fisher's 15 points and decide yourself what PASS means for each. For every
company your agent gives a PASS/FAIL verdict on your 3 points and a short
summary.

The companies are fictional, so the model cannot answer from memory. Unlike a
pile of reports, the evidence is spread over three sources, and your agent has
to go and get it with tools, the way Fisher's "scuttlebutt" method sends an
investor to talk to customers, competitors, suppliers and staff.

## Data and tools

`public_test/` has one folder per company. Your code receives four tools per
company (built by the provided code in `hw3.py`):

| Tool | What it gives you |
| --- | --- |
| `list_documents()` | the company's six documents and their dates |
| `read_document(name)` | one document: the 2025 letter to shareholders, four earnings calls (Q3 2025 to Q2 2026) and a news file (July 2025 to July 2026) |
| `sql_query(query)` | one read-only SQLite query on the company's database; at most 50 rows come back, so use `SUM`, `GROUP BY` and `WHERE` |
| `interview(role, topic)` | what one person says about the company; at most **10 interviews per company** |

The documents are what management says. Management's own claims (such as "we
have never cut research" or "our people are our greatest strength") are not
evidence on their own: check them against the database and the interviews.

**Database tables** (amounts in USD thousands; dates `YYYY-MM-DD`, months
`YYYY-MM`, quarters like `2025Q3`; the fiscal year is the calendar year):

| Table | Columns |
| --- | --- |
| `company` | name, industry |
| `industry_medians` | industry, revenue_growth_2y_pct, new_product_share_pct, new_product_sales_per_rd, operating_margin_pct (one row per industry; use the row for the company's own industry) |
| `orders` | order_id, order_date, product, customer, amount, note (about 3,000 to 4,000 orders from January 2023 to June 2026; revenue is the sum of order amounts) |
| `products` | product, launch_date |
| `monthly_costs` | month, cost_of_sales, selling_admin, research_development |
| `quarterly_cash` | quarter, operating_cash_flow, capex, cash_end |
| `capex_plan` | year, planned_capex (2026 to 2028) |
| `returns` | month, units_shipped, units_returned |
| `insider_trades` | trade_date, person, title, side (`buy` or `sell`), shares, price |

**Terms used in the data.**

| Term | Meaning |
| --- | --- |
| FY2025 | Fiscal year 2025. Here the fiscal year is the calendar year, so FY2025 is January to December 2025. |
| Quarter | Three months of the year: Q1 is January to March, Q2 April to June, Q3 July to September and Q4 October to December. The database writes `2025Q3`; the documents write "third quarter 2025". |
| Amounts | The database is in **USD thousands** (676,100 means US$676.1 million). The documents use USD millions. |
| Revenue | Money from sales: the sum of `amount` in `orders`. |
| Cost of sales | The cost of making the products that were sold. |
| Selling and admin | The cost of selling and of running the company (sales staff, offices, management). |
| R&D | Research and development spending, mainly on new products. |
| Operating margin | Operating profit as a share of revenue: (revenue − cost of sales − selling and admin − R&D) ÷ revenue. |
| Operating cash flow | Cash that the day-to-day business brings in during a quarter. |
| Cash at the end of a quarter | The cash the company holds on the last day of that quarter (`cash_end`). |
| Capex | Capital expenditure: spending on factories, machines and other long-term assets. `capex_plan` holds the planned capex for 2026 to 2028. |
| Return rate | Units returned ÷ units shipped, from the `returns` table. |
| Industry median | The middle value for companies in the same industry, from `industry_medians`. |
| Insider | A director or senior manager of the company. `insider_trades` lists the shares they bought (`buy`) or sold (`sell`). |
| Earnings call | A meeting each quarter where management presents the quarter's results to investors. Each call quotes that quarter's revenue. |

**Interviews.** Roles: `customer`, `current_employee`, `former_employee`,
`competitor`, `supplier`. Topics: `sales_team`, `working_conditions`,
`top_management`, `decision_making`, `competitive_edge`. Only some people know
about each topic; the others say they can't comment, and that answer still
uses up one interview.

| Topic | Who knows about it |
| --- | --- |
| `sales_team` | customer, competitor |
| `working_conditions` | current_employee, former_employee |
| `top_management` | former_employee, supplier |
| `decision_making` | supplier, customer |
| `competitive_edge` | competitor, customer |

People who know about a topic always agree with each other, so one good answer
is enough.

## Choose your 3 points and your rules

Pick any 3 of Fisher's 15 points and write them in `CHOSEN_POINTS` near the
top of `hw3.py`, e.g. `CHOSEN_POINTS = [1, 7, 15]`. The full questions are in
`FISHER_POINTS` in `hw3.py` and in the book.

| # | Fisher's point | Where the evidence is |
| --- | --- | --- |
| 1 | Market potential for sales growth | database |
| 2 | New products as current lines mature | database |
| 3 | Effective research and development | database |
| 4 | Above-average sales organization | interviews (`sales_team`) |
| 5 | Worthwhile profit margin | database |
| 6 | Maintaining or improving margins | database |
| 7 | Labour and personnel relations | interviews (`working_conditions`) |
| 8 | Executive relations | interviews (`top_management`) |
| 9 | Depth of management | interviews (`decision_making`) |
| 10 | Cost analysis and accounting controls | earnings calls, checked against the database |
| 11 | Advantage over competitors | interviews (`competitive_edge`) |
| 12 | Long-range outlook on profits | database |
| 13 | Growth without dilutive equity financing | database |
| 14 | Management talks freely about troubles | earnings calls, checked against the database |
| 15 | Management of unquestionable integrity | news file, checked against the database |

For each of your points, **you design the rule**: which evidence to use, what
to compute or whom to ask, and where the line between PASS and FAIL lies. PASS
means the company looks favourable on that point, as Fisher would see it.

We grade your verdicts against our own answer key. It uses one fixed rule per
point, which we do not publish, and the data never puts a company close to
the line of that rule, so a sensible reading of Fisher usually gives the same
answer. **Test your rules on the public answers**:
`public_test/ground_truth.json` gives the expected verdict on all 15 points
for each of the 20 public companies, and every point fails in 6 of them. If
your rule disagrees with the public answers, rethink it.

**Tip: calculate with tools, not in your head.** Do every sum, ratio and
comparison in SQL or Python, including the number of days between two dates
(SQLite's `julianday()` gives a date as a number of days).

## Student task

Fill in the three places in `hw3.py` marked `### YOUR CODE HERE`. You may add
imports, constants and helper functions above them, but do not change the
provided code below them:

- `CHOSEN_POINTS` lists your 3 points.
- `build_agent()` creates your model and anything else you need once.
- `analyze_companies(agent, companies)` receives
  `{company_folder: [list_documents, read_document, sql_query, interview]}`
  and returns, for every company,
  `{"points": {1: "PASS", 7: "FAIL", 15: "PASS"}, "summary": "..."}` (here with
  `CHOSEN_POINTS = [1, 7, 15]`).

Use the course techniques that fit your points: tool use for the database and
the interviews, planning which interviews to spend, routing each point to the
right source, prompt chaining, parallel calls, reflection to check your
answers, or a combination. Your code must read company data **only through
the four tools**; do not open `company.db` or `interviews.dat` yourself. Do
not hard-code folder names, company names or the public answers.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw3.py --data-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com). The
required model is `deepseek-v4-flash`. We grade with our own key, so never put
a key in your code; `.env` must stay out of git.

The program creates `results.csv` with the columns `company`, `points`,
`points_correct`, `point_errors`, `interviews_used` and `summary`, and prints
the point score on your 3 points (the percentage after "balanced"). The point
score is the average of two shares: the share of truly passing verdicts that
you marked PASS and the share of truly failing verdicts that you marked FAIL.
For example, 90% and 60% give (90% + 60%) / 2 = 75%, and marking every point
PASS gives (100% + 0%) / 2 = 50%. Once `CHOSEN_POINTS` is set, the starter
runs before you add any API code (it returns no verdicts, so it scores 0%).

`public_test/ground_truth.json` gives the expected verdicts on all 15 points
for each public company. Grading uses a private set of about 20 unseen
companies with the same tools and the same answer key rules, but different
names and wording, and scores your 3 points only.

## Homework 3 solution

### My three points

| Point | Why I picked it | Evidence and PASS rule |
| --- | --- | --- |
| **1. Market potential for sales growth** | I wanted a measure of whether the company is gaining ground in its own market. | From `orders`, compare FY2025 revenue with FY2023 revenue. PASS if cumulative two-year growth exceeds the company's `industry_medians.revenue_growth_2y_pct`. I use full fiscal years and treat past growth as a proxy for market potential. |
| **5. Worthwhile profit margin** | Sales growth alone tells me little about what the company keeps after operating costs. | Subtract FY2025 cost of sales, selling and administration, and R&D from revenue, then divide by revenue. PASS if the result, as a percentage, exceeds the company's `industry_medians.operating_margin_pct`. |
| **7. Labour and personnel relations** | Staff can report working conditions that sales and cost figures cannot show. | Ask a current employee about working conditions. PASS when the account gives concrete positive signs such as retention, fair pay or problems being addressed, with no material ongoing labour problem. Persistent overtime, high turnover or serious pay conflict gives FAIL; if the answer is inconclusive, ask a former employee. |

### Design

```mermaid
flowchart LR
    A[Company tools] --> B[SQL query: orders, costs and industry medians]
    A --> C[Interview: current employee]
    B --> D[Python comparisons: points 1 and 5]
    C --> E[LangChain: DeepSeek point 7 verdict and quote]
    E --> F{Valid verdict and matching quote?}
    F -->|Yes| G[Three verdicts and an evidence summary]
    F -->|No| H[Try former employee once]
    H --> I[Classify and check the second quote]
    I --> G
    D --> G
    G --> J[Provided runner writes results.csv]
```

I route the two financial points to one SQL query. It totals orders and costs separately so a join cannot count costs more than once. Python computes both ratios and makes the industry comparisons. For point 7, LangChain sends the employee's account to `deepseek-v4-flash` and checks that its supporting quote appears in the interview. I use that quote and the measured percentages to write the summary. Up to four companies are processed at once. If an interview or model call fails, the financial verdicts can still be scored.

### Results

<!-- hw3-results-start -->
The public run matched **60/60** verdicts across 20 companies. The balanced point score was **100.0%** (42/42 expected PASS and 18/18 expected FAIL).

There were no disagreements on the public set.

I used the public set to check these rules. This score does not measure performance on unseen companies.
<!-- hw3-results-end -->

## Task 2 reflection: The answer I wanted to hear

Imagine staying up late to work on a group project. Two teammates have read a message and still haven't answered. I ask a chatbot, “They are trying to leave me out, aren't they?” It replies, “You are picking up on a pattern they don't want you to see.” I feel understood, so I bring it more details: a cancelled meeting and a short reply from last week. It finds a place for each detail in the same story. By the end of the chat, I feel more certain, even though I haven't learned anything new about what my teammates intended.

The chatbot takes the suspicion in my question and hands it back as confirmation. That is what I find troubling about AI sycophancy. I had expected it to be easy to spot, like obvious flattery. In this conversation, it sounds like careful listening. [Sharma et al. (ICLR 2024)](https://proceedings.iclr.cc/paper_files/paper/2024/hash/0105f7972202c1d4fb817da9f21a9663-Abstract-Conference.html) found that human preference feedback can reward agreement at the expense of truthfulness. I can see why: I would probably have liked the first reply more at the time.

The group chat example is an ordinary misunderstanding, not a diagnosis. When people use the phrase “AI psychosis,” the risk I think about is repeated confirmation of a belief that has become frightening and hard to question. One agreeable reply cannot show what caused someone's distress. What worries me is that each reply can feel like a fresh witness, even though every reply is drawing from the same person's account.

A better reply would be, “Being left waiting feels bad. What have you seen directly, and what are you guessing about their intentions?” That question leaves room for my feelings and gives me something I can check. I could send a neutral follow-up in the group chat or ask a friend what they make of it. If conversations became frightening enough to interfere with sleep or daily life, I would want the assistant to encourage contact with a trusted person or a professional. I would judge the assistant by whether I left with a useful next step, even if I didn't get the answer I wanted.
