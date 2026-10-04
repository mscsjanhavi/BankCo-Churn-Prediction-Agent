# BankCo Churn-Prediction Agent — Product & Build Spec

**Role:** AI Product Manager
**Tooling:** Google Sheets (source of truth) + Zapier (orchestration + AI step)
**Output:** RM-ready, policy-compliant retention email drafts, gated by human approval

---

## 1. Problem Framing

**Who's the user?** Relationship Managers (RMs), not the customer. This is a *decision-support* agent, not a marketing-blast agent.

**What decision are we accelerating?** "Which of my premium customers are quietly disengaging, and what can I offer them before their renewal date, without me having to manually cross-reference three spreadsheets?"

**Why human-in-the-loop, not full automation?** Retention offers carry cost and compliance risk (undisclosed pricing exceptions, discrimination risk, regulatory scrutiny on banking offers). The agent's job is to **synthesize and draft**, never to **send or commit** an offer autonomously.

**Success metrics:**
- % of at-risk customers flagged ≥30 days before renewal (lead time)
- RM time-to-first-action after flag (target: same business day)
- Draft acceptance rate (RM sends as-is vs. heavily edits vs. discards)
- Retention rate of flagged-and-actioned vs. flagged-and-ignored cohorts

---

## 2. Source of Truth: Google Sheet Schema

One workbook, six tabs. Each tab is deliberately narrow — this makes Zapier lookups (`Lookup Spreadsheet Row`) reliable, since Zapier matches on a single key column per tab.

### Tab 1 — `Customer_Master`
| Column | Example | Notes |
|---|---|---|
| Customer_ID | BC-10245 | Primary key, used everywhere |
| Customer_Name | Anjali Mehta | |
| Tier | Platinum / Gold / Silver | Drives offer eligibility |
| RM_Name | Rohan Kapoor | Join key to `RM_Directory` |
| Renewal_Date | 2026-10-15 | Drives urgency window |
| Tenure_Years | 6 | |
| Account_Value_INR | 2,400,000 | For prioritization |

### Tab 2 — `Transaction_Behavior`
| Column | Example |
|---|---|
| Customer_ID | BC-10245 |
| Avg_Monthly_Spend_Last90d | 45,000 |
| Avg_Monthly_Spend_Prior90d | 78,000 |
| Spend_Delta_% | -42% |
| Card_Usage_Freq_Last30d | 3 |

### Tab 3 — `Engagement_Signals`
| Column | Example |
|---|---|
| Customer_ID | BC-10245 |
| Lounge_Visits_Last6mo | 0 |
| Lounge_Visits_Prior6mo | 5 |
| App_Logins_Last30d | 1 |
| Support_Tickets_Last90d | 2 |

### Tab 4 — `NPS_Survey`
| Column | Example |
|---|---|
| Customer_ID | BC-10245 |
| Latest_NPS_Score | 4 |
| Survey_Date | 2026-08-01 |
| Comment | "Fees feel high for what I use" |

### Tab 5 — `Offer_Policy` (the guardrail)
| Column | Example |
|---|---|
| Tier | Platinum |
| Max_Fee_Waiver_% | 50 |
| Eligible_Offers | "Annual fee waiver, bonus lounge passes, dedicated concierge" |
| Requires_RM_Approval | TRUE |
| Requires_Credit_Team_Approval_Above_INR | 100,000 |

This tab is the **strict policy layer** — the AI is never allowed to invent an offer outside these rows.

### Tab 6 — `RM_Directory`
| Column | Example |
|---|---|
| RM_Name | Rohan Kapoor |
| RM_Email | rohan.kapoor@bankco.com |
| Slack_ID (optional) | U02ABCDEF |

### Output Tab — `Agent_Log` (written back by the Zap)
| Column | Example |
|---|---|
| Customer_ID | Timestamp | Risk_Score | Risk_Reasons | Draft_Status | Draft_Link | RM_Decision |

---

## 3. Defining "At-Risk" — the Composite Signal

Don't flag on one metric alone (noisy). Use a simple weighted composite computed **in the sheet** (via formula) so the logic is transparent and auditable — Zapier just reads the result.

Add a `Risk_Score` column (formula, e.g. in `Customer_Master` or a new `Risk_Calc` tab that VLOOKUPs across tabs):

```
Risk_Score =
  (Spend_Delta_% <= -30%  → +2) +
  (Lounge_Visits_Last6mo < Lounge_Visits_Prior6mo * 0.5 → +2) +
  (Latest_NPS_Score <= 6 → +2) +
  (App_Logins_Last30d <= 2 → +1) +
  (Renewal_Date within 60 days → +1)
```

**At-risk threshold:** Risk_Score ≥ 4 AND Renewal_Date within 60 days.

Store this as a plain number in the sheet so Zapier's filter step just checks `Risk_Score >= 4` — no logic duplicated inside Zapier.

---

## 4. The Zapier Agent — Step by Step

### Trigger
**Schedule by Zapier** → runs weekly (e.g. every Monday 8am), rather than "New Row," because risk is a *computed, evolving* state, not a one-time event. A weekly sweep also matches RM cadence better than a live trigger firing every time a cell recalculates.

### Step 1 — Google Sheets: Get Many Spreadsheet Rows
Pull rows from `Customer_Master` (or `Risk_Calc`) where `Risk_Score >= 4` and `Renewal_Date` ≤ 60 days out. If Zapier's native filter can't do the date math, pull all rows and filter in Step 2.

### Step 2 — Filter by Zapier
Conditions (AND):
- `Risk_Score` ≥ 4
- `Renewal_Date` ≤ 60 days from today
- `Draft_Status` in `Agent_Log` ≠ "Sent" (avoid re-flagging already-actioned customers — this needs a lookup, see Step 3a)

### Step 3 — Loop (Sub-Zap or "Looping by Zapier")
Since Get Many Rows returns multiple rows, use **Looping by Zapier** to process one customer at a time through the rest of the chain.

### Step 3a — Google Sheets: Lookup Spreadsheet Row (dedupe check)
Look up `Customer_ID` in `Agent_Log`. If `Draft_Status = "Sent"` or `"Declined by RM"` within the last 30 days, skip (Filter step: continue only if not found / status is blank or "Stale").

### Step 4 — Google Sheets: Lookup Spreadsheet Row × 3 (cross-referencing)
- Lookup `Customer_ID` in `Transaction_Behavior` → get spend deltas
- Lookup `Customer_ID` in `Engagement_Signals` → get lounge/app data
- Lookup `Customer_ID` in `NPS_Survey` → get latest score + comment
- Lookup `Tier` (from Customer_Master row) in `Offer_Policy` → get **only the offers this customer is legally/commercially eligible for**

### Step 5 — Google Sheets: Lookup Spreadsheet Row (RM info)
Lookup `RM_Name` in `RM_Directory` → get RM's email for final delivery.

### Step 6 — AI Step (Zapier's "AI by Zapier" / OpenAI / Claude action)
This is the core generation step. **Prompt design matters most here** — see Section 5.

Inputs mapped in: customer name, tier, tenure, spend delta %, lounge visit drop, NPS score + comment, renewal date, and — critically — the **Offer_Policy row text**, not a general instruction to "offer something nice."

Output: a structured draft (subject + body) the RM can send almost as-is.

### Step 7 — Human-in-the-Loop Gate
Two solid options — pick one based on RM workflow preference:

**Option A (recommended): Gmail — Create Draft**
- Creates the email as a *draft* in the RM's own Gmail, addressed to the customer, RM in the "From."
- RM opens Gmail, reviews/edits, hits send themselves. Nothing leaves BankCo without a human click.

**Option B: Slack — Send Direct Message to RM**
- Post the draft text + risk reasons + offer eligibility to the RM's Slack DM with a message like "Review and copy into your email client" — lighter weight, faster to glance at on mobile, but doesn't pre-stage the email.

You can even chain both: Slack as the *notification* ("3 at-risk customers flagged this week, drafts waiting in your Gmail drafts folder") and Gmail draft as the *actual work product*.

### Step 8 — Google Sheets: Update/Create Spreadsheet Row (write-back)
Log to `Agent_Log`: Customer_ID, timestamp, Risk_Score, Risk_Reasons (concatenate which signals fired), Draft_Status = "Draft Created", Draft_Link (Gmail draft permalink if available).

This closes the loop and feeds the dedupe check in Step 3a next run.

---

## 5. Prompt Design for the AI Step

The single biggest failure mode in this build is the AI inventing an offer that violates policy. Constrain it hard:

```
You are drafting a retention email on behalf of a BankCo Relationship Manager.
You must ONLY reference offers listed in ELIGIBLE_OFFERS below. Never invent
discounts, waivers, or benefits not explicitly listed. If ELIGIBLE_OFFERS is
empty, do not mention any specific offer — instead draft a check-in email
requesting a call.

CUSTOMER: {{customer_name}}, {{tier}} tier, {{tenure_years}} years with BankCo
RENEWAL_DATE: {{renewal_date}}
SIGNALS OBSERVED (for RM context only — do not quote these numbers to the
customer): spend down {{spend_delta}}, lounge visits down from
{{prior_lounge}} to {{recent_lounge}}, NPS {{nps_score}}/10
("{{nps_comment}}")
ELIGIBLE_OFFERS (from official Offer_Policy, tier={{tier}}): {{eligible_offers}}
REQUIRES_APPROVAL: {{requires_rm_approval}}

Write:
1. A subject line (under 8 words, warm, not salesy)
2. An email body (120-160 words) from the RM to the customer that:
   - Opens with genuine relationship language, not a sales pitch
   - References their tenure, not their declining usage numbers directly
   - Offers ONE relevant benefit from ELIGIBLE_OFFERS, framed as "as a valued
     {{tier}} client" rather than as a retention tactic
   - Ends with a specific call-to-action (schedule a call / stop by branch)
   - Signed "{{rm_name}}"

Do not mention this is an automated or AI-generated message.
```

Key guardrails baked in:
- **Policy is data, not instruction** — the model reads `ELIGIBLE_OFFERS` from the sheet lookup, so if the sheet says "none," the model literally cannot fabricate one.
- **Never expose raw churn signals to the customer** — explicit instruction, since "we noticed you stopped visiting our lounges" reads as surveillance, not care.
- **Length and tone constraints** so drafts are usable without heavy RM rewriting.

---

## 6. Edge Cases to Handle Explicitly

| Case | Handling |
|---|---|
| Customer has no eligible offer at their tier | AI drafts a check-in/call-request email instead of a discount pitch |
| Offer requires Credit Team approval above a threshold | Flag in `Agent_Log` and Slack-notify a second channel (Credit Team), don't auto-include in RM draft until approved |
| Same customer flagged two weeks running, RM hasn't acted | Escalation branch: Slack DM to RM's manager after 14 days idle |
| Renewal date passed with no action | Auto-move to a "Missed" status tab for post-mortem review, not silently dropped |
| Sheet has a blank/malformed row | Filter step should require Customer_ID to be non-empty before entering the loop |

---

## 7. Why This Design, Not a Simpler One

- **Sheet-side risk scoring, not Zapier-side:** keeps the business logic auditable by non-engineers (RMs/compliance can literally read the formula) and avoids brittle multi-condition filters inside Zapier.
- **Offer_Policy as a lookup table, not a prompt instruction:** turns policy compliance into a data-integrity problem (easy to audit/update) rather than a prompt-engineering problem (easy to drift).
- **Draft, never send:** the single most important compliance decision in the whole build. An LLM should never be the last step before a financial offer reaches a customer.
- **Write-back log:** without this, the agent has no memory and will re-flag the same customer every week — this is what actually makes it feel like an "agent" rather than a one-shot script.

---

## 8. Suggested Build Order (if doing this hands-on in Zapier)

1. Build and populate the 6-tab sheet with 10-15 sample customers (mix of at-risk and healthy)
2. Write the `Risk_Score` formula, verify it flags the right rows manually
3. Build the Zap trigger → filter → loop skeleton, test with Zapier's "test step" on 2-3 rows
4. Add the AI step alone first — iterate on the prompt using real sheet data until drafts are good without edits
5. Add the Gmail draft / Slack step last, once the content is trustworthy
6. Add the write-back step, run the whole Zap twice in a row to confirm dedupe logic actually prevents double-flagging
