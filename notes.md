# StyleNest — AI Customer Support Assistant
## Project Documentation

**Project type:** Local portfolio prototype  
**Tools:** n8n Community Edition (self-hosted in Docker), Google Sheets, Google Gemini free tier, Gmail  
**Business:** StyleNest (fictional online shop)  
**Status:** Prototype; not a production system

---

## 1. Project Overview

StyleNest is a fictional online shop used to demonstrate an AI-assisted customer support workflow. The project receives customer messages, validates contact details, summarizes and classifies each message, checks the shop FAQ for an answer, prepares a reply draft, and escalates uncertain or urgent cases by email.

### Technology Stack

- **Workflow automation:** n8n Community Edition, self-hosted locally in Docker
- **Customer intake and records:** Google Forms and Google Sheets
- **AI model and embeddings:** Google Gemini (free tier)
- **Knowledge base:** n8n Simple Vector Store
- **Notifications:** Gmail
- **Alternative intake:** n8n Webhook (POST)
- **Testing:** Separate n8n Evaluation workflow with 25 labeled test messages

---

## 2. Main Workflow Architecture

**Current design:**

Google Sheets Trigger (Row Added, every minute) → Clean Data → Valid Email? → Basic LLM Chain (Gemini + Structured Output Parser, prompt v3).

The workflow then follows two branches:

### Branch 1 — Summary, confidence and reply

1. Save Summary (Google Sheets Update Row).
2. Wait 30 seconds.
3. Confident? (IF condition: confidence ≥ 0.7).
4. **True:** AI Agent using FAQ RAG → Save Reply Draft (Status = `Draft Ready`).
5. **False:** Save Needs Human (Status = `Needs Human Review`) → Low Confidence Alert (Gmail).

### Branch 2 — Urgency

1. Is Urgent? (IF condition).
2. If urgent, send an urgent Gmail alert.

### Invalid email route

If the Valid Email? condition is false, send a Bad Email Alert through Gmail.

### Structured classification fields

The Structured Output Parser returns:

- `summary`
- `category`
- `sentiment`
- `urgent`
- `confidence`

---

## 3. Development Log

### Day 1 — Form to Sheet to AI Summary

**Goal:** Google Form → Google Sheet → AI summary → Gmail alert.

- Built the first version using a Google Sheets Trigger (Row Added), Basic LLM Chain with Gemini, and a Gmail node.
- Replaced free-text output with a Structured Output Parser so the AI returns consistent JSON fields: summary, category, sentiment, urgent, and confidence.
- **Fixed:** Reconnected an expired Google Sheets credential.
- **Fixed:** The prompt used `$json.Name` and `$json.Message`, but the actual fields were lowercase. Updated the expressions so the AI received the message.
- **Fixed:** Set Gemini temperature to `0` to reduce inconsistent classifications across repeated runs.
- **Known issue:** Some test rows had invalid email addresses. The Valid Email? condition still needs to be rechecked with those rows.

### Day 2 — FAQ Retrieval-Augmented Generation (RAG)

**Goal:** Answer customer questions only from the StyleNest FAQ, rather than from the model's general knowledge.

Created a separate workflow named **FAQ Loader**, which is run manually:

1. Manual Trigger.
2. Code node splits the FAQ text using the `## Q` heading pattern.
3. Simple Vector Store (Add Documents; Memory Key: `stylenest_faq`; Clear Store: ON).
4. Default Data Loader and Text Splitter.
5. Google Gemini embeddings.

A separate **FAQ Answer Test** workflow uses a Chat Trigger, AI Agent, and vector store to check that answers are grounded in the FAQ.

**Important:** The Simple Vector Store is held in memory. After any n8n or Docker restart, run FAQ Loader again. Otherwise, the agent may return the fallback answer: “A human agent will contact you soon”.

### Day 3 — AI Agent and Reply Draft

- Configured the AI Agent with StyleNest support rules:
  - Answer only from the FAQ tool.
  - If the FAQ does not contain the answer, use: “A human agent will contact you soon”.
  - Start with “Hi [name]”.
  - End with “StyleNest Support”.
- Connected the Simple Vector Store as a **Retrieve Documents as Tool** resource (Memory Key: `stylenest_faq`; Limit: `3`; Google Gemini embeddings).
- Saved generated replies in the Google Sheet under the **Reply Draft** column.
- **Important lesson:** In Google Sheets Update Row nodes, map only the match column (Email) and the columns that should change. Mapping extra columns can overwrite existing data with empty values.
- **Fixed:** Save Reply Draft had cleared Summary, Category, Sentiment, and Confidence. Restricted its mapping to Email, Reply Draft, and Status, then re-ran Save Summary.

---

## 4. Current AI Support and Escalation Rules

### FAQ-based reply behaviour

- The AI Agent must use the FAQ retrieval tool as its only source for customer-specific answers.
- If the FAQ does not contain the answer, use: “A human agent will contact you soon”.
- Start the reply with “Hi [name]”.
- End the reply with “StyleNest Support”.
- Save the generated reply in the **Reply Draft** column.

### Low-confidence escalation

If confidence is below `0.7`, the workflow does not create an AI reply. It sets Status to **Needs Human Review** and sends a Gmail alert.

If confidence is at least `0.7`, the message proceeds to the FAQ-based AI Agent.

**Tests performed:**

- “Ok so about the thing” → confidence `0.3`, category `other`; sent to the false/low-confidence branch.
- “Hmm not sure what to say” → confidence `0.3`; sent to the false/low-confidence branch.
- For both tests, Reply Draft stayed empty and the alert emails arrived.
- “This is the 3rd time my order is late. Terrible service.” → urgent `true`, category `complaint`, sentiment `angry`; urgent Gmail alert arrived. The FAQ had no late-order policy, so the reply used the fallback.

**Honest description:** This is a low-confidence escalation alert, not a full human approval system. There is no approve button.

---

## 5. Classifier Evaluation — Prompt v1, v2 and v3 (November 3–4)

### Evaluation workflow

Created a separate **Evaluation** workflow to measure classifier performance without writing to the customer Sheet or sending emails.

Workflow:

1. Manual Trigger.
2. Code node creates 25 labeled test messages.
3. Loop Over Items (batch size `1`).
4. Basic LLM Chain.
5. Wait 20 seconds.
6. Continue through the loop.
7. After processing, Code nodes compare predictions with expected categories and urgency values, then calculate accuracy.

### Test set

The 25 messages were written by the project builder. Messages 1–20 were the original set. Messages 21–25 were added later and covered:

- Wrong item received
- Size question
- Feedback
- Repeated refund request
- Mild packaging feedback

### Results

| Prompt version | Test set | Category result | Urgent result | Notes |
|---|---:|---:|---:|---|
| v1 | 20 messages | 20/20 | 19/20 | Message 19 (“third time my order is late”) was wrong. |
| v2 | 25 messages | 25/25 | 22/25 | Fixed message 19 but introduced three urgency errors: messages 1 and 4 were predicted true instead of false, and message 12 was predicted false instead of true. |
| v3 | 25 messages | 25/25 | 25/25 | Used five clear urgency rules, explicit non-urgent examples, and a stricter confidence rule. |

### Evaluation lesson

Broadening an urgency rule fixed one case but caused three new errors. Clear rules with explicit “not urgent” examples worked better on this test set.

### Evaluation caveat

Prompt v3 was written after reviewing errors from this same test set. The 25 messages were written by the project builder, so the set is small and relatively easy. A result of 25/25 is not proof of 100% accuracy. Real customer messages have not yet been evaluated.

---

## 6. Errors Encountered and Fixes

### Gemini 429 rate limit

- The Gemini free tier rate limit was encountered (noted during the project as 15 requests per minute).
- **Fix:** Added Loop Over Items with a 20-second Wait to slow down requests.

### 400 — `max_output_tokens must be positive`

- **Cause:** A Max Output Tokens option was set to `0` in the Gemini node.
- **Fix:** Removed that option.

### 503 — `model high demand`

- **Cause:** Temporary Google server-side issue.
- **Mitigation:** Enabled Retry On Fail on the Basic LLM Chain (maximum 3 tries, 30-second wait). This is a mitigation, not a permanent fix.

### Gmail credential needs reconnecting

- **Fix:** Reconnected using Sign in with Google.
- **Possible cause if it happens repeatedly:** The Google Cloud OAuth app may be in Testing mode if credentials expire again after around seven days.

### Google Sheets Trigger timeout and blank credential window

- The trigger timed out after approximately 6 minutes and 49 seconds, and the Google credential window stayed empty.
- **Fix attempted:** Restarted the n8n container, ran `wsl --shutdown`, then restarted Docker and n8n.
- **Cause:** Not confirmed. A stuck Docker/WSL network after several days of uptime was suspected.

### Vector store emptied after restart

- **Cause:** The Simple Vector Store is in memory.
- **Fix:** Ran FAQ Loader again to repopulate it.

### Update Row cleared existing columns

- **Cause:** Extra columns were included in the Update Row mapping and were overwritten with empty values.
- **Fix:** Restricted each Update Row mapping to the match column and only the fields that should actually be updated.

---

## 7. Error Handling (Phase 6)

- Created a separate workflow named **Error Handler**.
- Workflow: Error Trigger → Gmail.
- The alert email includes:
  - Workflow name
  - Failed node
  - Error message
  - Execution link
- Tested using Execute workflow; the email arrived.
- In the main workflow, selected **Error Handler** under Settings → Error Workflow. Initially it appeared grey in the dropdown; after reopening settings it could be selected.

- Retry On Fail is enabled for the main workflow's Basic LLM Chain and AI Agent:
  - Max tries: 3 (n8n default)
  - Wait between tries: 3 seconds (Basic LLM Chain) and 2 seconds (AI Agent)
**Not yet verified:** The Error Workflow is expected to run when the main workflow is Active, not during a manual Execute. Active-mode failure handling has not yet been tested.

---

## 8. Webhook Intake — November 4

**Goal:** Allow support messages to enter without using Google Forms, for example through curl or Postman, while keeping the same processing pipeline.

Created a separate workflow named **Webhook Intake**. It receives a POST request and appends the message to the same Google Sheet. The main workflow's Google Sheets Trigger is intended to pick up the new row. The main workflow itself was not changed for this setup.

### Webhook workflow setup

1. Webhook node:
   - Method: `POST`
   - Path: `support-message`
2. Google Sheets node:
   - Operation: Append Row
3. Map the incoming request fields from:
   - Name: `$json.body.name`
   - Email: `$json.body.email`
   - Message: `$json.body.message`
4. Timestamp expression: `{{ $now.toISO() }}`

### Problems and debugging

**Problem 1:** The Append Row node stayed grey after the webhook fired in test mode.

- **Fix:** Opened the node and clicked Execute step.

**Problem 2:** A new row showed Summary and Urgent already filled in, with Urgent = TRUE for “Where is my order?”, even though the main workflow toggle was OFF.

- Checked the Sheet version history, but it only showed the user's own Google account and could not identify which workflow wrote the cells.
- Checked Overview → Executions and found an older workflow named **My workflow 2**, which was running automatically about 10 seconds after each new row.
- The old Day 1 workflow was still published and watching the same Sheet.
- **Fix:** Unpublished the old workflow and renamed it so it could not be activated by mistake.

### Verification

- A new webhook test row (`webhooktest2`) stayed empty.
- The main workflow was then run manually.
- The row was populated with Summary, Urgent = FALSE, Category, Confidence `0.95`, Reply Draft, and Status.

**Key lesson:** When a Sheet changes unexpectedly, check Overview → Executions to identify which workflow actually ran before changing prompts or node settings. Old workflows watching the same Sheet can also write to it.

### Current webhook limitation

Only the Test URL and a manual run of the main workflow have been tested. The production Webhook URL with the main workflow Active has not yet been tested.

---

## 9. Current Status and Known Limitations

- The main workflow has been tested manually with webhook-created rows.
- The Webhook Test URL has been tested; the production Webhook URL with the main workflow Active has not been tested.
- The Simple Vector Store is in-memory and must be reloaded after n8n or Docker restarts.
- Gemini free-tier rate limits and temporary service availability can interrupt runs.
- The 25-message evaluation set is small, self-authored, and not independent.
- Real customer messages, high-volume traffic, security controls, privacy requirements, and production reliability have not been validated.
- The low-confidence route sends an alert but does not provide a full human review or approval interface.
- Active-mode Error Workflow handling has not yet been verified.

---

## 10. Recommended Next Steps

1. Test the production Webhook URL while Webhook Intake and the main workflow are Active.
2. Test end-to-end messages for each route:
   - Valid email
   - Invalid email
   - High confidence
   - Low confidence
   - Urgent message
3. Test the Error Workflow by deliberately causing a controlled failure while the main workflow is Active.
4. Run FAQ Loader after every n8n or Docker restart. Before expanding the project, consider replacing the in-memory store with persistent vector storage.
5. Expand the evaluation set with more varied, independently written customer messages, including ambiguous, misspelled, and multi-issue messages.
6. Review Google OAuth app publishing/testing settings and credential lifecycle.
7. Before any real deployment, add suitable authentication, data protection, access controls, monitoring, a rate-limit strategy, and human review procedures.

---

## 11. Portfolio Summary

Built a local AI customer support automation prototype for a fictional e-commerce shop using n8n, Google Sheets, Google Gemini, Gmail, and a FAQ-based retrieval workflow. The system validates incoming messages, produces structured summaries and classifications, drafts FAQ-grounded replies, escalates low-confidence and urgent cases, and reports workflow failures by email. Added a separate 25-message evaluation workflow and an alternative webhook intake path.

Documented limitations include in-memory vector storage, free-tier rate limits, a small self-authored evaluation set, and the fact that production webhook intake and active-mode error handling remain unverified.

---

**Project note:** This documentation describes a local learning and portfolio prototype. It should not be presented as a production-ready customer support system.