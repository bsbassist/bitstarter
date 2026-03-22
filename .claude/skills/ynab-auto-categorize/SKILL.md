---
name: ynab-auto-categorize
description: Automatically categorize and approve YNAB transactions using AI reasoning and payee pattern learning. Use when the user wants to review and categorize their uncategorized or unapproved YNAB transactions.
---

# YNAB Auto-Categorize Skill

Review, learn from, and auto-categorize uncategorized and unapproved YNAB transactions using payee pattern matching and AI reasoning, then apply updates in bulk via the YNAB API.

## Prerequisites

- `YNAB_API_TOKEN` environment variable must be set (Personal Access Token from app.ynab.com/settings/developer)
- `curl` and `jq` must be available
- Internet access to reach `https://api.ynab.com/v1`

## Workflow

Make a todo list for all the tasks in this workflow and work on them one after another.

### 1. Verify Auth and Token

Check that the token is present and the API responds successfully:

```bash
if [ -z "${YNAB_API_TOKEN:-}" ]; then
  echo "ERROR: YNAB_API_TOKEN is not set."
  echo "Get a Personal Access Token from: https://app.ynab.com/settings/developer"
  echo "Then run: export YNAB_API_TOKEN=your_token_here"
  exit 1
fi

curl -sf -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $YNAB_API_TOKEN" \
  "https://api.ynab.com/v1/user"
```

If the HTTP status is not `200`, tell the user their token is invalid or expired and stop. Provide the link to regenerate: https://app.ynab.com/settings/developer

### 2. Select Budget

Fetch all budgets:

```bash
curl -sf \
  -H "Authorization: Bearer $YNAB_API_TOKEN" \
  "https://api.ynab.com/v1/budgets" \
  | jq '.data.budgets[] | {id, name, last_modified_on}'
```

Selection logic:
- If there is only one budget, use it automatically and tell the user which budget was selected.
- If there are multiple budgets, present a numbered list and ask the user to pick one.
- Store the selected budget ID as `BUDGET_ID` for all subsequent API calls.

### 3. Fetch Categories

Pull all category groups and their child categories to build a lookup map:

```bash
curl -sf \
  -H "Authorization: Bearer $YNAB_API_TOKEN" \
  "https://api.ynab.com/v1/budgets/${BUDGET_ID}/categories" \
  | jq '[.data.category_groups[]
      | select(.hidden == false and .name != "Internal Master Category")
      | {
          group: .name,
          categories: [.categories[]
            | select(.hidden == false and .deleted == false)
            | {id, name}]
        }]'
```

Build an in-memory map of `category_name -> category_id` and `category_id -> {name, group}` for display and for constructing PATCH payloads.

### 4. Fetch Uncategorized / Unapproved Transactions

Retrieve all unapproved transactions (includes both uncategorized and categorized-but-unapproved):

```bash
curl -sf \
  -H "Authorization: Bearer $YNAB_API_TOKEN" \
  "https://api.ynab.com/v1/budgets/${BUDGET_ID}/transactions?type=unapproved" \
  | jq '[.data.transactions[] | {
      id,
      date,
      amount,
      memo,
      payee_name,
      category_id,
      category_name,
      account_name,
      approved
    }]'
```

Amount notes:
- Amounts are in milliunits; divide by 1000 to display in dollars.
- Negative amounts are outflows (expenses). Positive amounts are inflows (income).

If the result is an empty array, inform the user there are no uncategorized or unapproved transactions and stop gracefully.

### 5. Fetch Recent Approved Transactions for Pattern Learning

Pull the last 200 approved, categorized transactions to learn payee-to-category patterns:

```bash
curl -sf \
  -H "Authorization: Bearer $YNAB_API_TOKEN" \
  "https://api.ynab.com/v1/budgets/${BUDGET_ID}/transactions?type=approved" \
  | jq '[.data.transactions
      | sort_by(.date) | reverse | .[:200][]
      | select(.category_id != null and .payee_name != null and .deleted == false)
      | {payee_name, category_id, category_name}]'
```

Build a payee-to-category frequency map in memory:
- For each payee name (normalized: trimmed and lowercased), count how many times each `category_id` was used.
- The most frequently used category for a payee becomes the "learned" suggestion for that payee.
- Example structure: `{"shell": {"cat-id-abc": {count: 7, name: "Fuel & Gas"}}, "whole foods": {...}}`

### 6. Suggest Categories for Each Transaction

For every unapproved/uncategorized transaction, determine the best category using this priority cascade:

1. **Already categorized** — if `category_id` is already set and non-null, keep it; only flag for approval.
2. **Exact payee match** — if the normalized payee name exactly matches a key in the learned map, use the most common category.
3. **Partial payee match** — if the normalized payee name contains a learned payee as a substring (or vice versa), use that mapping.
4. **AI reasoning** — use your knowledge of common merchants and transaction patterns:
   - "Shell", "BP", "Chevron", "Exxon", "Arco", "Mobil", "Valero" → Fuel / Gas
   - "Whole Foods", "Trader Joe's", "Kroger", "Safeway", "Publix", "Costco", "Aldi", "Sprouts" → Groceries
   - "Netflix", "Spotify", "Hulu", "Disney+", "Apple.com/bill", "Google Play", "Amazon Prime" → Subscriptions / Entertainment
   - "Amazon", "Amazon.com" → Shopping (unless memo suggests otherwise)
   - "Starbucks", "Dunkin", "Peet's Coffee", "Dutch Bros" → Coffee Shops / Dining Out
   - "CVS", "Walgreens", "Rite Aid", "Duane Reade" → Pharmacy / Medical
   - "PG&E", "Con Edison", "Xcel Energy", "National Grid", "Duke Energy" → Utilities
   - "Uber", "Lyft" → Transportation / Rideshare
   - "Uber Eats", "DoorDash", "Grubhub", "Postmates" → Dining Out / Restaurants
   - Large positive inflow (amount > $500) → likely Income or Transfer
   - Large round outflow (e.g. $1,200, $1,500, $2,000) → possibly Rent/Mortgage or Loan Payment
5. **Fallback** — if confidence is low, mark as `(needs review)` with no `category_id` assigned.

For each transaction, record:
- `suggested_category_id` (null if needs review)
- `suggested_category_name` (display string)
- `confidence`: `high` / `medium` / `low` / `needs-review`
- `reason`: brief explanation, e.g. "Learned: 8 past transactions", "AI: gas station merchant", "Already categorized"

### 7. Present Interactive Review Table

Display a formatted review table showing all transactions. Example:

```
YNAB Auto-Categorize Review
Budget: My Budget  |  Transactions to review: 8
──────────────────────────────────────────────────────────────────────────────────────────
 #   Date        Account          Payee                     Amount      Suggestion          Conf
──────────────────────────────────────────────────────────────────────────────────────────
 1   2026-03-20  Checking         Shell #4821              -$45.00     Fuel & Gas           high
 2   2026-03-19  Checking         Whole Foods Market      -$127.43     Groceries            high
 3   2026-03-18  Checking         Netflix.com              -$15.49     Subscriptions        high  [already set]
 4   2026-03-17  Visa             ACME WIDGET CO           -$89.00     (needs review)       —
 5   2026-03-16  Checking         Starbucks #0042           -$6.75     Coffee Shops         med
 6   2026-03-15  Checking         Amazon.com               -$34.99     Shopping             med
 7   2026-03-14  Checking         Payroll Direct Dep     +$2,450.00    Income: Salary       med
 8   2026-03-12  Checking         LANDLORD LLC           -$1,800.00    Rent/Mortgage        low
──────────────────────────────────────────────────────────────────────────────────────────
Learned from 47 payee patterns across 200 recent transactions.
```

After the table, present options:

```
Options:
  [A]  Approve all with suggested categories (skips any marked "needs review")
  [N]  Cancel — make no changes
  [1-8] Edit the category for a specific transaction (e.g. type "4")
  [list] Show all available categories

What would you like to do?
```

Handle user input in a loop:

- **A**: Confirm count of transactions that will be updated, then proceed to Step 8.
- **N**: Exit without making any changes. Tell the user nothing was modified.
- **A number (e.g. "4")**: Ask: "Enter a category name for [Payee] (or 'skip' to leave as needs-review):". Accept a partial name and fuzzy-match against the category list. Show top 3 matches if ambiguous. Update the suggestion in the table, then re-display and prompt again.
- **"list"**: Print all categories grouped by their category group, then re-display the table and prompt.

Continue looping until the user types A or N.

### 8. Apply Changes via PATCH

Construct the bulk update payload and submit it:

```bash
# Build the JSON payload from confirmed selections, then:
curl -sf -X PATCH \
  -H "Authorization: Bearer $YNAB_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "transactions": [
      {"id": "<txn-id>", "category_id": "<cat-id>", "approved": true},
      {"id": "<txn-id>", "approved": true}
    ]
  }' \
  "https://api.ynab.com/v1/budgets/${BUDGET_ID}/transactions"
```

Payload construction rules:
- **Truly uncategorized** (`category_id` was null): include both `category_id` and `approved: true`.
- **Already categorized but unapproved**: include only `approved: true` (do not override the existing category).
- **Needs review**: omit from the payload entirely — leave these transactions untouched in YNAB.

Check the response. If not 2xx, extract and display the error detail from `.error.detail` in the JSON response. List which transaction IDs were affected.

### 9. Commit Skill File and Push

After completing the workflow:

```bash
git -C /home/user/bitstarter checkout claude/ynab-auto-categorize-skill-ljcI6 2>/dev/null || \
  git -C /home/user/bitstarter checkout -b claude/ynab-auto-categorize-skill-ljcI6
git -C /home/user/bitstarter add /home/claude/.claude/skills/ynab-auto-categorize/SKILL.md
git -C /home/user/bitstarter commit -m "Add YNAB auto-categorize skill"
git -C /home/user/bitstarter push -u origin claude/ynab-auto-categorize-skill-ljcI6
```

## Error Handling Reference

| Situation | Action |
|---|---|
| `YNAB_API_TOKEN` not set | Print setup instructions and stop |
| 401 from API | Token invalid/expired — link to https://app.ynab.com/settings/developer |
| 404 on budget | Budget ID not found — re-run budget selection |
| Empty unapproved transactions | Inform user all transactions are already categorized/approved |
| No matching category for AI suggestion | Mark as `(needs review)`, skip in PATCH |
| PATCH returns error | Show which transaction IDs failed + API error detail |
| `jq` not found | `sudo apt install jq` or `brew install jq` |

## Wrap up

In your final message to the user, provide a summary in this format:

**YNAB Auto-Categorize Complete**

- Budget: [Budget Name]
- Transactions reviewed: [total]
- Categorized and approved: [count]
- Already categorized, now approved: [count]
- Skipped (needs review): [count]
- Pattern learning: [N] payee patterns from [M] recent approved transactions

Validation:
1. [✅/‼️] API authentication
2. [✅/‼️] Categories loaded ([N] categories across [M] groups)
3. [✅/‼️] Transactions updated via PATCH

Remind the user they can re-run `/ynab-auto-categorize` at any time to catch new transactions. Payee patterns are re-learned fresh each session from their transaction history — no stale rules.
