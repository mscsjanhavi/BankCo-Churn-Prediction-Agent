# Build Notes & Lessons Learned

This project was built hands-on in Zapier's UI over several sessions. This doc is an honest account of what actually went wrong and how it was resolved — left in deliberately, because the debugging is as informative as the final design.

## Issue 1: "Looping by Zapier" silently passed typed text instead of live data

**Symptom:** A downstream `Lookup Spreadsheet Row` step kept failing with "nothing could be found for the search," even though the same lookup worked fine when tested with hardcoded values.

**Root cause:** When configuring the Loop step's "Values to Loop" fields, clicking into a value box while a stray search term was still active in the insert panel caused Zapier to insert the *literal search text* (e.g. the string `"Formatted Rows COL C"`) into the field, rather than a live reference to that column. Visually, this is easy to miss — plain text and a live-reference "pill" look similar at a glance in some of Zapier's field states.

**How it was diagnosed:** Checked the raw "Data in" payload of the failing step and found the `lookup_value` field literally contained the words `"Formatted Rows COL C"` instead of an actual tier value like `"Gold"`. Confirmed by testing whether a text cursor could be placed *inside* the value (possible = plain text; not possible = live reference).

**Fix in the moment:** Cleared the field completely, re-inserted the reference using "Add value set" fresh rather than editing an existing broken one.

**Design fix (better):** Rather than keep fighting the Loop step's Beta-quality UI, the pipeline was redesigned to use a **Google Sheets "New or Updated Spreadsheet Row"** trigger instead of `Schedule → Get Many Rows → Loop`. This trigger fires natively per-row, eliminating the need for a Loop step entirely. This turned out to be a materially better architecture, not just a workaround — simpler, fewer failure points, and every subsequent lookup step worked correctly on the first try.

**Lesson:** When a no-code tool's "advanced" or "Beta" feature causes repeated, hard-to-diagnose failures, the fix is often to question the architecture choice itself rather than to keep debugging the feature.

## Issue 2: Uploaded Excel file wasn't visible to Zapier's Google Sheets connector

**Symptom:** The BankCo spreadsheet, visible and editable in Google Drive, did not appear in Zapier's "Spreadsheet" dropdown at all — not even via search.

**Root cause:** The file had been uploaded to Drive as a raw `.xlsx` file (shown with the Excel icon), not converted into a native Google Sheet. Zapier's Google Sheets connector only sees genuine Google Sheets, not Office files merely stored in Drive.

**Fix:** Opened the file in Drive's preview, used **File → Save as Google Sheets** to create a true converted copy, then pointed Zapier at that new file.

**Lesson:** "I can see and open it in my browser" does not mean a third-party tool can access it the same way — file *format*, not just visibility, matters for API-based integrations.

## Issue 3: Gemini's "thinking" tokens silently consumed the entire output budget

**Symptom:** The AI step succeeded (no error) but returned incomplete, garbled text — a fragment reading "34 + 2 + 2 = 144 words. Perfect!" followed by cut-off reasoning, instead of an actual email.

**Root cause:** `gemini-3.6-flash` is a reasoning ("thinking") model. It spends a portion of its token budget on internal reasoning before producing final output. With `Max Output Tokens` set to the default `1024`, the model used 984 tokens on internal thinking and had none left to write the actual answer — confirmed by the response metadata showing `"Finish Reason": "MAX_TOKENS"` and a `Thoughts Token Count` of 984.

**Fix:** Increased `Max Output Tokens` to `3000`, giving the model room to reason *and* still produce full output. Retesting then returned a complete, correctly-formatted, policy-compliant email.

**Lesson:** With reasoning-capable models, a low token limit doesn't just truncate the answer — it can eat the entire budget on invisible reasoning and leave nothing for the actual response, which looks like a formatting bug rather than a budget problem.

## Issue 4: Deprecated model name

**Symptom:** AI step failed outright with an error naming a specific retired model and suggesting its replacement.

**Fix:** Google's own error message specified the correct current model name; swapped it in directly. The most reliable fix here was simply reading the full error text rather than assuming a config problem.

## Issue 5: Zapier free-tier step limit

**Symptom:** After building and successfully testing all 7 steps, publishing was blocked with "Upgrade to Pro to publish this Zap."

**Root cause:** Zapier's free plan caps Zaps at 2 steps total; this build has 7 (trigger + filter + 2 lookups + AI step + Gmail draft + write-back). This is a real pricing limitation, not a bug — Professional-tier pricing (~$20-30/month depending on task volume) is required to run this specific design live and unattended.

**Resolution:** Documented as a known limitation rather than worked around. Every individual step is fully built and independently verified working via Zapier's per-step test runs (see main README for evidence) — the build is complete and correct; only continuous/scheduled live execution requires a paid plan.

**Lesson:** It's worth checking a tool's free-tier constraints *before* designing a multi-step architecture, not after building it — though in this case, discovering it late didn't cost anything beyond a final "aha," since every step remains independently testable and demonstrable regardless of publish status.
