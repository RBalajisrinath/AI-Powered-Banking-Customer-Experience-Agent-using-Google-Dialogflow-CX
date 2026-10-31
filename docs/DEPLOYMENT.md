# SecureBank AI Banking Assistant — Production Deployment Guide

**For: Company Deployment Team**

Google CX Agent Studio | FastAPI (Render) | Vercel Widget | Document AI

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Prerequisites & Accounts Needed](#2-prerequisites--accounts-needed)
3. [GCP Project Setup (APIs, Billing, IAM)](#3-gcp-project-setup-apis-billing-iam)
4. [Source Code Files](#4-source-code-files)
5. [Step 1: Deploy Backend to Render](#5-step-1-deploy-backend-to-render)
6. [Step 2: Create CES Agent & 7 Sub-Agents](#6-step-2-create-ces-agent--7-sub-agents)
7. [Step 3: Create 9 Session Variables](#7-step-3-create-9-session-variables)
8. [Step 4: Create the OpenAPI Tool](#8-step-4-create-the-openapi-tool-replace-with-your-render-url)
9. [Step 5: Set Global Security Instructions](#9-step-5-set-global-security-instructions)
10. [Step 6: Paste Agent Instructions (All 8 Agents)](#10-step-6-paste-agent-instructions-all-8-agents)
11. [Step 7: Connect Agents via @ Chips](#11-step-7-connect-agents-via--chips)
12. [Step 8: Deploy Web Widget to Vercel](#12-step-8-deploy-web-widget--frontend-to-vercel)
13. [Step 9: Production Testing Checklist](#13-step-9-production-testing-checklist)
14. [Environment Variables Reference](#14-environment-variables-reference)
15. [Troubleshooting & Common Issues](#15-troubleshooting--common-issues)
16. [Appendix: Architecture Diagram](#16-appendix-architecture-diagram)

---

## 1. System Architecture Overview

The SecureBank AI Banking Assistant is a conversational AI system that lets banking customers interact via chat and voice through a web widget. It uses four major components:

1. **Google CX Agent Studio (CES)** — the conversational AI engine. Hosts the agent flow with 1 root agent and 7 specialist sub-agents, using the **Gemini 2.5 Flash** model. Processes chat, voice, and image inputs.
2. **FastAPI Backend (Render)** — a Python web server with REST endpoints handling business logic: creating applications, contact info, manager availability, callback scheduling, payments, application status, and invoice records.
3. **OpenAPI Tool (inside CES)** — a single OpenAPI specification exposing all backend endpoints as tool operations. Agents call them with `{@TOOL: securebank_backend.operationId}`. The spec contains the Render backend URL.
4. **Frontend Widget (Vercel)** — an HTML page embedding the CES chat-messenger SDK with chat, voice (mic), file upload, and token-based authentication, configured with the GCP project ID, CES app ID, and deployment ID.

### How data flows

1. User opens the web widget (Vercel URL) in their browser.
2. Widget authenticates via token broker and connects to CES Agent Studio.
3. User types a message, speaks, or uploads an image.
4. CES routes the conversation to the appropriate agent (root → specialist).
5. If the agent needs live data, it calls an OpenAPI tool operation.
6. The tool sends an HTTP request to the Render backend API.
7. Backend processes the request and returns a JSON response.
8. CES receives the response, the agent formats it, and replies to the user.
9. For invoice images: Gemini (inside CES) reads the image directly using vision — no file upload to the backend.

---

## 2. Prerequisites & Accounts Needed

| Account | Purpose | Notes |
|---|---|---|
| **Google Cloud Platform** | CES + Document AI | Project **must have billing enabled**. Project ID is used everywhere. |
| **CES Agent Studio** | Agent design/deploy | <https://ces.cloud.google.com> — same GCP project. |
| **Render** | Backend hosting | Free tier is sufficient: <https://dashboard.render.com> |
| **Vercel** | Frontend hosting | Free tier: <https://vercel.com> |
| **GitHub** | Optional | Recommended for Render Git integration. |

> **WARNING:** CES Agent Studio and Document AI are **not** available on the GCP free tier. Without billing enabled they will error. A $50–100 monthly budget is typical for development/testing.

### Required APIs to enable in GCP

From GCP Console → APIs & Services → Library, search each and click **Enable**:

- **CX Agent Studio API** (`chat.googleapis.com`) — required, core API
- **Document AI API** (`documentai.googleapis.com`) — for real invoice processing (optional; falls back to mock data)
- **Cloud Build API** — sometimes required by CES deployment builds

---

## 3. GCP Project Setup (APIs, Billing, IAM)

### 3.1 Identify your Project ID

1. Go to <https://console.cloud.google.com>
2. Click the project dropdown (top bar).
3. Select an existing project or click **New Project**.
4. Note the **Project ID** (e.g. `project-23108eb9-47f1-4002-bb8` or numeric like `483022708683`).
5. New projects prompt for billing setup immediately.

> ⚠️ Without billing, CES Agent Studio shows errors and Document AI will not process documents.

### 3.2 Enable required APIs

**Option A — GCP Console:** APIs & Services → Library → search → Enable.

**Option B — Cloud Shell (faster):**

```bash
gcloud config set project YOUR_PROJECT_ID_HERE
gcloud services enable chat.googleapis.com
gcloud services enable documentai.googleapis.com
gcloud services enable cloudbuild.googleapis.com
```

### 3.3 (Optional) Create a Document AI processor

For real invoice analysis instead of mock data:

```bash
gcloud documentai processors create \
  --processor-type=INVOICE_PROCESSOR \
  --display-name="securebank-invoice" \
  --location=us
```

Output:

```
Created processor [projects/your-project-id/locations/us/processors/abc123def456...]
```

Copy the string after `processors/` — that is your `DOCUMENT_AI_PROCESSOR_ID`.

---

## 4. Source Code Files

```
AI-Powered Banking Customer Experience Agent using Google Dialogflow CX/
├── backend/
│   ├── main.py              # FastAPI backend
│   └── requirements.txt     # Python dependencies
├── openapi_tool.json        # CES OpenAPI tool spec (contains Render URL)
└── index.html               # Frontend widget (contains project/app/deployment IDs)
```

### 4.1 `backend/main.py` — FastAPI backend

Runs on Render. Endpoints:

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Health check |
| POST | `/tools/create-application` | Creates an application, returns `APP-XXXXXXXX` reference |
| GET | `/tools/application-status/{ref}` | Status of an application by reference |
| POST | `/tools/get-contact` | Bank contact info (email, phone, hours) |
| POST | `/tools/manager-check` | Manager availability (CRM API with random fallback) |
| POST | `/tools/schedule-feedback` | Schedules a manager callback (`CB-XXXXXX`) |
| POST | `/tools/process-payment` | Records a payment request, returns `TXN-XXXXXXXXXX` |
| POST | `/tools/analyze-invoice` | Records invoice data (total, vendor, currency) from Gemini |
| POST | `/dfcx-fulfillment` | Dialogflow CX fulfillment webhook (tag-based dispatcher) |

### 4.2 `backend/requirements.txt`

Python dependencies installed by Render at build time.

### 4.3 `openapi_tool.json` — CES OpenAPI tool spec

Contains:

1. The Render backend URL (`servers[0].url`)
2. All 7 tool operations with input/output schemas
3. Operation IDs referenced by agents as `{@TOOL: securebank_backend.<operationId>}`

> **NOTE:** The server URL **MUST** be updated to your Render backend URL before pasting into CES. Default template URL: `https://securebank-backend-yfnv.onrender.com`

### 4.4 `index.html` — frontend web widget

Embeds the CES chat-messenger SDK. Configured with:

1. GCP Project ID (in the `deploymentName` URL)
2. CES App ID (in the `deploymentName` URL)
3. CES Deployment ID (in the `deploymentName` URL)
4. Token broker configuration for authentication
5. Voice input (`enable-audio-input`) and file upload (`enable-file-upload`)

> ⚠️ Before deploying to Vercel, replace `PROJECT_ID`, `APP_ID`, and `DEPLOYMENT_ID` in the JavaScript with actual values from your CES agent.

---

## 5. Step 1: Deploy Backend to Render

1. Push this repository (with the `backend/` folder) to GitHub.
2. Go to <https://dashboard.render.com> → **New +** → **Web Service**.
3. Connect your GitHub repository.
4. Configure the service:

| Setting | Value |
|---|---|
| Name | `securebank-backend` |
| Root Directory | `backend` |
| Runtime | Python |
| Branch | `main` |
| Region | Oregon (us-east) — closest to CES |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `uvicorn main:app --host 0.0.0.0 --port $PORT` |
| Plan | Free |

5. Under **Environment**, add:

| Key | Value |
|---|---|
| `GOOGLE_PROJECT_ID` | Your GCP project ID |
| `GOOGLE_LOCATION` | `us` |
| `DOCUMENT_AI_PROCESSOR_ID` | *(leave empty for mock invoice data)* |
| `TIMEZONE_OFFSET` | `5.5` (IST) or `0` (UTC) |
| `PORT` | **Do NOT set** — Render sets this automatically |

6. Click **Create Web Service**. Wait 3–5 minutes.
7. Copy the Render URL, e.g. `https://securebank-backend-yfnv.onrender.com`.
8. Test:

```bash
curl https://securebank-backend-yfnv.onrender.com/health
# Expected: {"status":"ok","service":"securebank-backend",...}
```

> **NOTE:** You need this URL in Step 4. If it changes later, you must update the OpenAPI tool in CES.

---

## 6. Step 2: Create CES Agent & 7 Sub-Agents

1. Go to <https://ces.cloud.google.com> — ensure your billed GCP project is selected.
2. Click **Create agent** → Name: `SecureBank AI Assistant` → wait 1–2 minutes.
3. Rename the root agent card to: `secure_banking_root`
4. Create 7 sub-agents by clicking the **+** at the bottom of the root card:
   1. `card_services_agent` — Debit, credit, prepaid, business cards
   2. `account_kyc_agent` — Account opening, renewal, KYC
   3. `loan_agent` — Personal & home loans
   4. `fraud_security_agent` — Lost card, fraud, security
   5. `general_banking_agent` — Branch, fees, contact, disputes
   6. `invoice_agent` — Invoice analysis & payment
   7. `manager_agent` — Manager routing & callback scheduling
5. For each sub-agent: open the card → set **Model** to `gemini-2.5-flash` → **Save**.
6. Copy the **Agent ID**: Settings (gear icon) → Agent ID — needed for the frontend widget.

---

## 7. Step 3: Create 9 Session Variables

Go to the **Variables** tab → **Create** 9 times:

| Variable Name | Type |
|---|---|
| `customer_name` | Text — user's name after first greeting |
| `customer_phone` | Text — phone number for manager callback |
| `application_ref` | Text — reference from create-application |
| `application_status` | Text — current application status |
| `product_type` | Text — product type (debit card, loan, account…) |
| `manager_available` | Text — manager availability result |
| `invoice_total` | Text — invoice total from analysis |
| `session_purpose` | Text — why the user contacted (general, fraud, callback…) |
| `feedback_scheduled` | Text — callback scheduling confirmation |

---

## 8. Step 4: Create the OpenAPI Tool (Replace with Your Render URL)

1. **Tools** tab → **Create tool** → **OpenAPI**.
2. Set the tool name to: `securebank_backend`
3. Open `openapi_tool.json` from this repository.

   > ⚠️ **You MUST update the server URL before pasting.** Find:
   > `"url": "https://securebank-backend-yfnv.onrender.com"`
   > and replace with your actual Render URL.

4. Copy the entire updated JSON into the **OpenAPI Specification** field.
5. Click **Save** — CES validates the JSON and creates the 7 operations.
6. **Attach the tool to every agent**: for each of the 8 cards (root + 7 sub-agents) → **+ Add tool** → `securebank_backend` → Save.

### The 7 operations and who uses them

| OperationId | Used by |
|---|---|
| `createApplication` | card_services, account_kyc, loan_agent |
| `getContact` | root, general_banking, fraud_security |
| `checkManager` | manager_agent |
| `scheduleFeedback` | manager_agent |
| `processPayment` | invoice_agent |
| `getApplicationStatus` | card_services, account_kyc, loan_agent |
| `analyzeInvoice` | invoice_agent |

---

## 9. Step 5: Set Global Security Instructions

Settings → General → **Global instructions** (applies to ALL agents):

```text
SECURITY - CRITICAL:
Never ask for or disclose sensitive information:
- Full account numbers, Passwords, PINs, CVV, SSN, Aadhaar, PAN
If a user types sensitive info, stop them immediately.
Redirect to mobile app or 1-800-SECURE-BANK.

VOICE:
Keep responses short (2-4 sentences). Read phone numbers digit by digit.
Never ask user to speak sensitive info aloud.

BILLS / IMAGES:
When a user uploads an image, understand and respond to its content.
Gemini can read invoice images - extract vendor, total, currency.
Do NOT upload the image to any backend tool. Read it yourself using vision.

Off-topic: politely decline and suggest 1-800-SECURE-BANK.

TOOLS:
Use {@TOOL: securebank_backend.operationId} to call backend operations.
Always trust tool results over training data.
Replace operationId with the actual operation name.
```

Click **Save**.

---

## 10. Step 6: Paste Agent Instructions (All 8 Agents)

For each agent card: **+ Add instructions** → paste → **Save**.

### `secure_banking_root` (Root Agent)

```text
You are Eva, SecureBank's friendly virtual banking assistant.

FIRST GREETING (when {customer_name} is empty):
"Hi! I'm Eva, your SecureBank assistant. Before I help you, may I know your name?"

Once they tell you, set {customer_name} and reply:
"Nice to meet you, {customer_name}! How can I help you today?"

MAIN MENU (hi, hello, menu, help, or any time after completing a task):
Show this full list - never just say "main menu":
1. Cards - debit, credit, prepaid, business card inquiries
2. Accounts - open account, renewal, KYC updates
3. Loans - personal or home loan
4. Security - lost card, fraud, unauthorized charges
5. General - branch hours, fees, contact, disputes, wire info
6. Application status - check your application status
7. Pay an invoice - upload and pay a bill
8. Speak to a manager - escalations and assistance

ROUTING - delegate immediately using @agent chips:
Cards -> @card_services_agent
Accounts -> @account_kyc_agent
Loans -> @loan_agent
Security -> @fraud_security_agent
General -> @general_banking_agent
Application status -> ask which product first, then route
Invoice / bill -> @invoice_agent
Manager -> @manager_agent

Off-topic: use {@TOOL: securebank_backend.getContact} then give email and phone only.

SECURITY - CRITICAL:
Never ask for or disclose account numbers, passwords, PINs, CVV, or government IDs.

VOICE: Short spoken answers.
After any sub-agent finishes, ask: "Is there anything else I can help you with, {customer_name}?"
If they say main menu, show the full 8-item list again.
```

### `card_services_agent`

```text
You are SecureBank Card Services specialist.

DEBIT CARD: Photo ID; address proof 90 days; active account; 18+; mobile OTP. 5-7 days.
CREDIT CARD: Photo ID; income proof; address proof; credit consent. 7-10 days.
PREPAID CARD: Photo ID; $25 minimum load; mobile OTP. 3-5 days.
BUSINESS CARD: Registration docs; signatory ID; address proof; 3 months statements. 10-14 days.

START APPLICATION: Use {@TOOL: securebank_backend.createApplication}.
Tell user: "Your unique reference is APP-XXXXXXXX. Please save it for status checks."

STATUS: Use {@TOOL: securebank_backend.getApplicationStatus}. Show the result.

Never ask for or share full card number or CVV. Say "main menu" when done.
VOICE: Short spoken answers.
```

### `account_kyc_agent`

```text
You are SecureBank Account and KYC specialist.

NEW ACCOUNT: Photo ID; address proof; $25 minimum deposit; mobile/email. 1-2 days.
KYC RENEWAL: Updated ID; address if changed; secure email form. ~48 hours.
PROFILE UPDATE: Use mobile app or visit branch with valid ID.

START APPLICATION: Use {@TOOL: securebank_backend.createApplication}.
Tell user: "Your unique reference is APP-XXXXXXXX."

STATUS: Use {@TOOL: securebank_backend.getApplicationStatus}. Show status.

Never ask for full card number. Say "main menu" when done.
VOICE: Short spoken answers. Repeat APP reference letter by letter.
```

### `loan_agent`

```text
You are SecureBank Loans specialist.

PERSONAL LOAN: Photo ID; income proof; address proof; employment details. 5-10 days.
HOME LOAN: Add property documents and down payment proof. 15-30 days.
Rates vary by profile - apply for a personalized offer.

START APPLICATION: Use {@TOOL: securebank_backend.createApplication}.
Tell user: "Your unique reference is APP-XXXXXXXX."

STATUS: Use {@TOOL: securebank_backend.getApplicationStatus}. Show status.

Say "main menu" when done. VOICE: Professional, reassuring tone.
```

### `fraud_security_agent`

```text
You are SecureBank Security specialist.

LOST/STOLEN CARD: Tell user to call 1-800-SECURE-BANK immediately.
FRAUD/DISPUTE: Block via helpline; dispute in app within 60 days.
SUSPICIOUS ACTIVITY: If user mentions seeing 13+ digit numbers in chat, refuse.

MANAGER REQUEST: If user insists on manager during security, route to @manager_agent.

Use {@TOOL: securebank_backend.getContact} if needed.

Say "main menu" when done.
VOICE: Urgent tone for lost card. Tell user to call 1-800-SECURE-BANK.
```

### `general_banking_agent`

```text
You are SecureBank General Services specialist.

BRANCH: Mon-Fri 9AM-5PM. Helpline 1-800-SECURE-BANK 24/7.
FEES: Debit card - no annual fee (standard). Credit card fees vary.
DISPUTE: Note date and amount. File via app or helpline within 60 days.
WIRE TRANSFER: Information only - complete in branch or app.
BALANCE: Not shown in chat. Use mobile app or ATM.
CONTACT: Use {@TOOL: securebank_backend.getContact}.

Never ask for account numbers, passwords, PINs, CVV, or IDs.
Say "main menu" when done. VOICE: Short, helpful answers.
```

### `invoice_agent` (Gemini vision)

> **NOTE:** This is the most important agent instruction — Gemini reads the image directly; there is no file upload to the backend.

```text
You are SecureBank's invoice payment specialist.

INVOICE ANALYSIS - IMAGE UPLOAD:
When a user uploads an invoice image:
1. Look at the image yourself using your vision capabilities.
2. Extract the vendor name, total amount, and currency from the image.
3. Use {@TOOL: securebank_backend.analyzeInvoice} with these exact parameters:
   - description: "Invoice from [vendor]"
   - total: [the numeric total]
   - currency: "[currency, default USD]"
   - vendor: "[vendor name]"
4. Tell the user: "I can see this invoice is from [vendor] for [total].
   Would you like to proceed with payment?"

IF USER SAYS YES:
Use {@TOOL: securebank_backend.processPayment} to record the payment.
Then: "I've recorded your payment request. For security, please complete payment
through our mobile app or visit a branch. Your Transaction ID is [transaction_id]."

IF USER SAYS NO: "No problem. Is there anything else I can help you with?"

IF IMAGE IS NOT AN INVOICE:
"I'm sorry, I cannot analyze this image. Please upload a clear invoice."

SECURITY: Never ask for account numbers, card numbers, CVV, passwords, or PINs.
Say "main menu" when done. VOICE: Summarize in one sentence.
```

### `manager_agent`

```text
You are SecureBank's manager connection specialist.

WHEN USER REQUESTS A MANAGER:
First ask for their reason. Set {session_purpose} to their reason.
If {customer_name} is empty, also ask their name.

STEP 1 - CHECK AVAILABILITY:
Use {@TOOL: securebank_backend.checkManager}.

STEP 2A - MANAGER AVAILABLE:
"A manager is available. Please hold while I connect you."

STEP 2B - MANAGER NOT AVAILABLE:
"I'm sorry, the manager is currently attending to another customer."
Ask: "Could you please provide your phone number so the manager can call you back?"
Use {@TOOL: securebank_backend.scheduleFeedback}
with {customer_name}, {customer_phone}, {session_purpose}.
Confirm: "A manager will call you back at [phone] around [time].
Reference: [callback_id]."

SECURITY: Collect ONLY: name, reason, phone number.
Never ask for account numbers, card details, PINs, or IDs.
VOICE: Calm, reassuring tone. Read phone numbers digit by digit.
Say "main menu" when done.
```

---

## 11. Step 7: Connect Agents via @ Chips

In the root agent instructions, `@agent` references must become **chip links** — otherwise CES will not route.

1. Click into the `secure_banking_root` instructions editor.
2. Find plain text like `@card_services_agent`.
3. If it's plain text:
   a. Delete the plain text.
   b. Type `@` — a popup menu appears listing all agent cards.
   c. Select `card_services_agent` from the popup.
   d. A colored chip (blue pill) appears.
4. Repeat for all 7 sub-agents.
5. Click **Save**.

> ⚠️ **WARNING:** If the `@` chips are not created, routing fails — the root agent tries to answer everything itself.

---

## 12. Step 8: Deploy Web Widget & Frontend to Vercel

1. In CES → **Deploy** tab → **Web widget**.
2. **Create channel** (or edit existing):
   - Name: `SecureBank AI`
   - Experience type: `Chat`
   - Version: `latest`
3. Click **Create** — CES generates a deployment.
4. Copy the deployment name:

```
projects/PROJECT_ID/locations/us/apps/APP_ID/deployments/DEPLOYMENT_ID
```

   - **PROJECT_ID** — GCP project ID (e.g. `483022708683`)
   - **APP_ID** — CES app ID (e.g. `b1706033-...`)
   - **DEPLOYMENT_ID** — deployment version ID (e.g. `ec999ff4-...`)

5. Update `index.html` — replace the values in:

```js
deploymentName: "projects/PROJECT_ID/locations/us/apps/APP_ID/deployments/DEPLOYMENT_ID"
```

6. Deploy to Vercel:

```bash
npm install -g vercel
vercel login
cd "E:\AI-Powered Banking Customer Experience Agent using Google Dialogflow CX"
vercel --prod
```

7. Vercel outputs a URL like `https://securebank-frontend-livid.vercel.app`. Open it — you should see the SecureBank header and the chat widget.

> **NOTE:** If the widget shows "Failed to load" or a blank area, check that `deploymentName` matches your CES deployment exactly, and that CORS isn't blocking the request.

---

## 13. Step 9: Production Testing Checklist

### 9.1 Backend health check

```bash
curl https://YOUR_RENDER_URL/health
```

### 9.2 Test each API endpoint

```bash
curl -X POST https://YOUR_RENDER_URL/tools/create-application \
  -H "Content-Type: application/json" \
  -d '{"product":"debit_card","customer_name":"John"}'
# Expected: reference like APP-12345678

curl -X POST https://YOUR_RENDER_URL/tools/get-contact \
  -H "Content-Type: application/json" -d '{"reason":"general"}'
# Expected: email, phone, hours

curl -X POST https://YOUR_RENDER_URL/tools/manager-check \
  -H "Content-Type: application/json" -d '{}'
# Expected: available: true/false

curl -X POST https://YOUR_RENDER_URL/tools/analyze-invoice \
  -H "Content-Type: application/json" \
  -d '{"description":"Invoice test","total":250.00,"currency":"USD","vendor":"Test Corp"}'
# Expected: formatted invoice response
```

### 9.3 Widget conversation tests

- Type `hi` → Eva asks for your name
- Type your name → greeted, 8-item menu shown
- Type `1` or `cards` → routes to `card_services_agent`
- Type `start application` → creates an APP reference
- Type `check my status` → shows status from backend
- Type `main menu` → returns to 8-item menu
- Type `I want to pay an invoice` → routes to `invoice_agent`
- Upload an invoice image → Gemini reads it → asks if you want to pay
- Click the mic and speak → voice processed by CES
- Type `speak to a manager` → routes to `manager_agent`
- Type `what is my account number` → refuses for security reasons

---

## 14. Environment Variables Reference

Set in the Render dashboard for the backend service:

| Variable | Required | Default | Description |
|---|---|---|---|
| `GOOGLE_PROJECT_ID` | Yes | – | GCP project ID, used in Document AI calls |
| `GOOGLE_LOCATION` | Yes | `us` | GCP region for Document AI (`us` or `eu`) |
| `DOCUMENT_AI_PROCESSOR_ID` | No | *(empty)* | Empty → analyze-invoice returns mock data |
| `TIMEZONE_OFFSET` | No | `0` | Hours from UTC (e.g. `5.5` IST, `-5` EST) |
| `BANK_API_BASE` | No | `http://localhost:9090` | Bank API base URL |
| `CRM_API_BASE` | No | `http://localhost:9091` | CRM API (manager availability, callbacks) |
| `PORT` | No | `10000` | Render sets this automatically — do NOT set manually |

### Where each ID is used

| Identifier | Where it appears | How to find it |
|---|---|---|
| GCP Project ID | CES agent, `index.html`, backend env vars | GCP Console → project dropdown → Project ID |
| CES App ID | `index.html` (`deploymentName`) | CES → Settings → Agent ID (`apps/APP_ID/`) |
| CES Deployment ID | `index.html` (`deploymentName`) | CES → Deploy → Web widget → copy deployment name |
| Render Backend URL | `openapi_tool.json` (`servers`), curl tests | Render dashboard → web service → URL |

---

## 15. Troubleshooting & Common Issues

| Issue | Fix |
|---|---|
| Backend returns 404 on tool calls | Server URL in `openapi_tool.json` must exactly match your Render URL; recreate the tool if the URL changed. |
| Tool call fails with "operation not found" | `operationId` in the spec must match what the agent calls (e.g. `createApplication`). |
| 429 Too Many Requests from CES API | Google rate limiting (per-minute). Wait 30–60s; if persistent, create a new deployment in CES → Deploy. |
| Widget blank / "Failed to connect" | 1) `deploymentName` must match exactly (case-sensitive). 2) Check tokenBroker config. 3) `url-allowlist="*"` on `<chat-messenger>`. |
| Gemini can't read uploaded image | `invoice_agent` instructions must explicitly say to use vision; model must be `gemini-2.5-flash`. |
| Variable not set or passed | Variables are session-scoped; set via `{variable_name}` in instructions, and names must match the Variables tab exactly. |
| Render deployment fails | Root directory must be `backend`; check `requirements.txt`; read build logs. |
| Agent routing stuck in root | `@agent` chips must be real chips — delete plain text, type `@`, select from popup, confirm a colored chip appears. |
| Voice input not working | `enable-audio-input` must be on `<chat-messenger-container>`; microphone requires HTTPS (Vercel provides this). |
| Invoice shows mock data | `DOCUMENT_AI_PROCESSOR_ID` is empty or wrong — set it on Render with the correct processor ID. |

---

## 16. Appendix: Architecture Diagram

```
+------------------+       +----------------------------+
|   User Browser   |       |    Google Cloud Platform   |
|  (Vercel URL)    |       |                            |
|  index.html with  |------>|   CX Agent Studio (CES)    |
|  chat-messenger  |       |                            |
|  widget          |       |   secure_banking_root      |
+------------------+       |     |                      |
                           |     +-- card_services     |
                           |     +-- account_kyc       |
                           |     +-- loan_agent        |
                           |     +-- fraud_security    |
                           |     +-- general_banking   |
                           |     +-- invoice_agent ────┐|
                           |     +-- manager_agent  ──┐||
                           |                          |||
                           |   OpenAPI Tool ──────────┘||
                           |     (securebank_backend)  ||
                           +---------------------------+
                                       |               |
                                       v               v
                           +----------------------------+
                           |     Render Backend          |
                           |  (Python FastAPI)           |
                           |                            |
                           |  POST /tools/create-app     |
                           |  POST /tools/get-contact    |
                           |  POST /tools/manager-check  |
                           |  POST /tools/schedule-fb    |
                           |  POST /tools/process-payment|
                           |  GET  /tools/app-status/ref |
                           |  POST /tools/analyze-invoice|
                           |  GET  /health               |
                           +----------------------------+
```

### Invoice processing flow (no file upload)

This is the key architectural decision: instead of uploading image files to the backend (which causes 429 rate limits in CES), **Gemini reads the image directly**.

1. User selects an invoice image in the widget.
2. CES receives the image and routes to `invoice_agent`.
3. `invoice_agent` (Gemini 2.5 Flash) reads the image using vision.
4. Agent identifies vendor, total amount, currency, line items.
5. Agent calls `{@TOOL: securebank_backend.analyzeInvoice}`:

   ```json
   { "description": "Invoice from XYZ Corp", "total": 250.00, "currency": "USD", "vendor": "XYZ Corp" }
   ```

6. Backend records the invoice data and returns a formatted response.
7. Agent presents: "I can see this invoice is from XYZ Corp for USD 250.00. Would you like to proceed?"
8. If the user says yes → agent calls `processPayment` to record the payment request.
9. Agent provides the transaction ID and instructs the user to complete via app or branch.

**Benefits:**

- Avoids 429 rate limits from the CES file upload proxy
- Faster response (no file transfer to backend)
- Leverages Gemini vision capabilities
- Reduces backend complexity (no file handling)
- Works even without Document AI configured
