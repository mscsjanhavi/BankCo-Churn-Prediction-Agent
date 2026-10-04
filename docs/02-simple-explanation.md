# BankCo Churn-Prediction Agent — Simple Explanation

**Goal:** Build a helper tool in Zapier that watches customer data in a Google Sheet, spots customers who might leave the bank, and helps the Relationship Manager (RM) send them a personal email — before it's too late.

**Important rule:** The tool never sends anything by itself. It only prepares a draft. A human (the RM) always checks it and hits send.

---

## 1. What Problem Are We Solving?

RMs manage many premium customers. It's hard for them to notice, on their own, when a customer is quietly losing interest — spending less, skipping lounge visits, giving low satisfaction scores.

This tool does that noticing for them, and also drafts a ready-to-send email, so the RM just has to review and click send.

**We'll know it's working if:**
- Customers are flagged early (at least a month before their renewal date)
- RMs act on the flag quickly
- RMs don't need to heavily rewrite the drafted emails
- Flagged customers actually stay with the bank more often than ones who weren't reached out to

---

## 2. Where the Data Lives: One Google Sheet, Six Tabs

Think of this like six simple tables, each holding one type of information. They're linked together using a `Customer_ID` — like a name tag that appears in every table.

### Tab 1: Customer_Master
The basic facts about each customer.
- Customer ID, Name, Tier (Platinum/Gold/Silver), Which RM handles them, Renewal date, How many years they've been a customer

### Tab 2: Transaction_Behavior
How much they're spending.
- Average monthly spend now vs. 3 months ago, and the % change

### Tab 3: Engagement_Signals
How much they're using bank perks.
- Lounge visits now vs. before, app logins, support tickets raised

### Tab 4: NPS_Survey
How happy they say they are.
- Latest satisfaction score (0–10) and any comments they left

### Tab 5: Offer_Policy — the rulebook
What the bank is actually allowed to offer each tier of customer.
- For example: "Platinum customers can get up to a 50% fee waiver or bonus lounge passes." This tab is the guardrail — the tool is never allowed to offer something that isn't listed here.

### Tab 6: RM_Directory
Contact details for each RM, so the tool knows who to notify.

### A 7th "logbook" tab: Agent_Log
Every time the tool flags someone and drafts an email, it writes a note here — who, when, why, and what happened. This stops it from flagging the same customer over and over.

---

## 3. How Do We Decide Someone Is "At Risk"?

We don't rely on just one number — that would be too noisy (one bad month doesn't mean someone's leaving). Instead, we add up a few warning signs into one **Risk Score**, calculated right inside the sheet using a formula:

- Spending dropped by 30% or more → +2 points
- Lounge visits dropped by half or more → +2 points
- Satisfaction score is 6 or below → +2 points
- Barely using the app → +1 point
- Renewal date is coming up soon → +1 point

**A customer is "at risk" if their score is 4 or higher AND their renewal is within 60 days.**

Because this is a simple formula in the sheet, anyone — not just a tech person — can open it and understand exactly why someone got flagged.

---

## 4. How the Zapier Tool Works, Step by Step

Think of this as an assembly line. Every Monday morning, it runs through these steps:

**Step 1 — Wake up on a schedule**
Every Monday at 8am, the tool starts running (instead of reacting instantly to every change, which would be noisy).

**Step 2 — Pull the list**
It grabs all customers from the sheet.

**Step 3 — Filter**
It keeps only the customers whose Risk Score is 4+ and renewal is coming soon. Everyone else is ignored this week.

**Step 4 — Go one customer at a time**
It loops through the flagged customers one by one, so nothing gets mixed up.

**Step 5 — Skip anyone already handled**
It checks the logbook (Agent_Log). If this customer was already emailed recently, skip them — no point flagging the same person twice.

**Step 6 — Gather the full picture**
For each customer, it looks up their spending, lounge usage, satisfaction score, and — most importantly — checks the Offer_Policy tab to see **exactly what this customer is allowed to be offered**.

**Step 7 — Find the RM's contact info**
It looks up which RM owns this customer and grabs their email.

**Step 8 — Write the draft**
It sends all this information to an AI, which writes a warm, personal-sounding email — using only the offer(s) allowed by policy. It never invents a discount that isn't on the approved list.

**Step 9 — Hand it to a human**
Instead of sending the email, it creates a **draft** in the RM's own Gmail (or sends the draft text to the RM on Slack). The RM reads it, edits if needed, and sends it themselves.

**Step 10 — Write it down**
The tool logs what happened in the Agent_Log tab, so next Monday it knows not to flag this customer again unnecessarily.

---

## 5. Teaching the AI to Write Good Emails

The instructions given to the AI are strict, so it can't go off-script:

- Only mention offers that are listed in the Offer_Policy tab for that customer's tier — nothing invented
- If there's no offer available, write a simple "let's catch up" email instead of a sales pitch
- Never mention the actual numbers (like "you visited the lounge less") — that feels like spying, not care
- Keep it warm and short — around 120–160 words
- Always end with a clear next step, like "let's schedule a call"
- Sign off with the RM's name, not the AI's

---

## 6. Situations We Plan For

| What could happen | What the tool should do |
|---|---|
| No offer is allowed for this customer | Send a simple check-in email instead of a discount offer |
| The offer is large and needs extra approval | Flag it separately for the Credit Team, don't include it in the draft yet |
| RM doesn't act for 2 weeks | Send a reminder to the RM's manager |
| Renewal date passes with no action taken | Move that customer to a "Missed" list for later review |
| A row in the sheet has missing information | Skip it rather than crash or send a broken email |

---

## 7. Why We Built It This Way

- **The risk score is calculated in the sheet, not hidden inside Zapier** — so anyone can open the sheet and understand exactly why a customer was flagged.
- **Offers come from a locked-down policy table** — this keeps the AI from ever promising something the bank hasn't approved.
- **The tool only drafts, never sends** — a human always makes the final call before any offer reaches a real customer. This matters a lot in banking.
- **It keeps a written log** — so it has "memory" and doesn't repeat itself or annoy customers with duplicate outreach.

---

## 8. Suggested Order to Build It

1. Set up the 6-tab sheet and add some sample customers (a few at-risk, a few healthy)
2. Add the Risk Score formula and check it flags the right people
3. Build the basic Zapier flow: trigger → filter → loop through customers
4. Add the AI writing step and test it until the emails sound right
5. Add the final step that creates the Gmail draft or Slack message
6. Add the logbook step, and run it twice to make sure it doesn't flag the same person twice
