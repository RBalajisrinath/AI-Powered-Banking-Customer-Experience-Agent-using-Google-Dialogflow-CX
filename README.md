# SecureBank AI Banking Assistant

An AI-powered banking customer experience agent built on **Google Dialogflow CX / CX Agent Studio (CES)**, with a **FastAPI** backend, a **web chat + voice widget** frontend, and **Google Document AI** for invoice processing.

Customers can chat or speak with "Eva", SecureBank's virtual assistant, to open applications, check status, analyze and pay invoices, get contact info, and escalate to a human manager.

---

## Table of Contents

- [System Architecture](#system-architecture)
- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Backend API Endpoints](#backend-api-endpoints)
- [Session Variables](#session-variables)
- [Agent Team](#agent-team)
- [Invoice Processing (No File Upload)](#invoice-processing-no-file-upload)
- [Quick Start (Local Development)](#quick-start-local-development)
- [Deployment](#deployment)
- [Environment Variables](#environment-variables)
- [Testing Checklist](#testing-checklist)
- [Troubleshooting](#troubleshooting)

---

## System Architecture

The system has four major components:

| Component | Technology | Hosting | Role |
|---|---|---|---|
| **Conversational AI** | CX Agent Studio (CES), Gemini 2.5 Flash | Google Cloud | Agent flow: 1 root agent + 7 specialist sub-agents. Handles chat, voice, and image input. |
| **Backend** | FastAPI (Python) | Render | 8 REST endpoints implementing the business logic (applications, contact info, manager checks, callbacks, payments, invoice records). |
| **Tool layer** | OpenAPI 3.0 spec (`openapi_tool.json`) | Inside CES | Exposes all backend endpoints to the agents as tool operations under the name `securebank_backend`. |
| **Frontend** | `index.html` + CES chat-messenger SDK | Vercel | Web widget with chat, voice input (mic), file upload, and token-based auth. |

```
+------------------+       +----------------------------+
|   User Browser   |       |    Google Cloud Platform   |
|  (Vercel URL)    |       |                            |
|  index.html +    |------>|   CX Agent Studio (CES)    |
|  chat-messenger  |       |   secure_banking_root      |
|  widget          |       |     |-- card_services      |
+------------------+       |     |-- account_kyc        |
                           |     |-- loan_agent         |
                           |     |-- fraud_security     |
                           |     |-- general_banking    |
                           |     |-- invoice_agent ----+|
                           |     |-- manager_agent ---+||
                           |   OpenAPI Tool ----------+||
                           |     (securebank_backend) ||
                           +---------------------------+
                                       |              |
                                       v              v
                           +----------------------------+
                           |   Render Backend (FastAPI) |
                           |   /tools/* REST endpoints  |
                           |   /dfcx-fulfillment webhook|
                           +----------------------------+
```

---

## How It Works

A single conversation flows through the system like this:

1. **User opens the widget** (Vercel URL) in their browser.
2. **Widget authenticates** via the token broker and connects to CES using the `deploymentName` (project / app / deployment IDs baked into `index.html`).
3. **User types, speaks, or uploads an image.**
4. **CES routes the conversation**: root agent (`secure_banking_root`) greets the user, collects the name, shows the 8-item main menu, then delegates to the right specialist sub-agent using `@agent` chips.
5. **If live data is needed**, the agent calls an OpenAPI tool operation, e.g. `{@TOOL: securebank_backend.createApplication}`.
6. **CES sends an HTTP request** to the Render backend defined in the `servers` URL of `openapi_tool.json`.
7. **The backend processes the request** (creates an application reference, checks manager availability, records a payment, …) and returns JSON.
8. **CES receives the response**, the agent formats it conversationally, and replies to the user. Results are stored in session variables (e.g. `application_ref`, `customer_name`).
9. **For invoice images**, Gemini (inside CES) reads the image directly with vision — no file upload to the backend.

There is also an optional **fulfillment webhook** (`POST /dfcx-fulfillment` in `backend/main.py`) that can be attached to Dialogflow CX flows as a single entry point: it dispatches on `fulfillmentInfo.tag` (`create-application`, `manager-check`, `schedule-feedback`, `analyze-invoice`, `get-contact`) and calls the same internal tool endpoints.

---

## Project Structure

```
AI-Powered Banking Customer Experience Agent using Google Dialogflow CX/
├── backend/
│   ├── main.py              # FastAPI server — all business-logic endpoints
│   ├── requirements.txt     # Python dependencies
│   └── .gitignore
├── docs/
│   └── DEPLOYMENT.md        # Full production deployment guide (GCP, Render, Vercel)
├── index.html               # Frontend web widget (CES chat-messenger SDK)
├── openapi_tool.json        # OpenAPI spec pasted into CES as the securebank_backend tool
├── .gitattributes
└── .gitignore
```

---

## Backend API Endpoints

`backend/main.py` — FastAPI app served by `uvicorn main:app`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Health check for uptime monitoring |
| POST | `/tools/create-application` | Creates an application, returns an `APP-XXXXXXXX` reference |
| GET | `/tools/application-status/{reference}` | Returns the stored status of an application |
| POST | `/tools/get-contact` | Returns bank contact info (email, phone, hours) |
| POST | `/tools/manager-check` | Manager availability — tries the CRM API, falls back to random |
| POST | `/tools/schedule-feedback` | Schedules a manager callback, returns `CB-XXXXXX` reference |
| POST | `/tools/analyze-invoice` | Records invoice data (total, vendor, currency) extracted by Gemini |
| POST | `/tools/process-payment` | Records a payment request, returns a `TXN-XXXXXXXXXX` ID |
| POST | `/dfcx-fulfillment` | Dialogflow CX fulfillment webhook (tag-based dispatcher) |

Each tool endpoint maps to an `operationId` in `openapi_tool.json` (`createApplication`, `getApplicationStatus`, `getContact`, `checkManager`, `scheduleFeedback`, `analyzeInvoice`, `processPayment`) which is what agents reference as `{@TOOL: securebank_backend.<operationId>}`.

---

## Session Variables

CES session variables persist data across the conversation (created in the **Variables** tab):

| Variable | Stores |
|---|---|
| `customer_name` | The user's name (collected on first greeting) |
| `customer_phone` | Phone number for manager callback |
| `application_ref` | Reference number returned by `create-application` |
| `application_status` | Current application status |
| `product_type` | Product the user is applying for (debit card, loan, account…) |
| `manager_available` | Manager availability result |
| `invoice_total` | Invoice total from analysis |
| `session_purpose` | Why the user contacted us (general, fraud, callback…) |
| `feedback_scheduled` | Callback scheduling confirmation |

---

## Agent Team

| Agent | Responsibility |
|---|---|
| `secure_banking_root` (Eva) | Greeting, name capture, 8-item main menu, routing to sub-agents |
| `card_services_agent` | Debit / credit / prepaid / business cards |
| `account_kyc_agent` | Account opening, renewal, KYC updates |
| `loan_agent` | Personal and home loans |
| `fraud_security_agent` | Lost card, fraud, unauthorized charges |
| `general_banking_agent` | Branch hours, fees, contact, disputes, wire info |
| `invoice_agent` | Invoice analysis (Gemini vision) and payment recording |
| `manager_agent` | Manager availability check and callback scheduling |

**Security policy (global instructions):** the agents never ask for or disclose account numbers, passwords, PINs, CVV, SSN, Aadhaar, or PAN. If a user types sensitive info, they stop the user and redirect to the mobile app or 1-800-SECURE-BANK.

Full agent instruction blocks (copy-paste ready) are in [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md).

---

## Invoice Processing (No File Upload)

A key architectural decision: instead of uploading image files through the CES proxy (which causes 429 rate limits), **Gemini reads the image directly**:

1. User uploads an invoice image in the widget.
2. CES routes to `invoice_agent` (Gemini 2.5 Flash).
3. The agent reads the image with vision and extracts vendor, total, currency.
4. It calls `{@TOOL: securebank_backend.analyzeInvoice}` with those values as JSON parameters.
5. The backend records the data and returns a formatted confirmation.
6. On "yes", the agent calls `processPayment` and gives the user a transaction ID.
7. The user completes payment in the mobile app or at a branch (for security).

Benefits: no 429 rate limits, faster responses, no file handling in the backend, and it still works with mock invoice data if Document AI isn't configured.

---

## Quick Start (Local Development)

```bash
# 1. Install dependencies
cd backend
pip install -r requirements.txt

# 2. (Optional) configure environment
#    Create backend/.env with:
#      GOOGLE_PROJECT_ID=your-gcp-project-id
#      GOOGLE_LOCATION=us
#      DOCUMENT_AI_PROCESSOR_ID=          # empty = mock invoice data
#      TIMEZONE_OFFSET=5.5                # e.g. IST
#      BANK_API_BASE=http://localhost:9090
#      CRM_API_BASE=http://localhost:9091

# 3. Run the server
uvicorn main:app --reload --port 8000

# 4. Verify
curl http://localhost:8000/health
```

Interactive API docs: <http://localhost:8000/docs>

---

## Deployment

See **[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)** for the full production guide covering:

1. GCP project setup (APIs, billing, IAM, Document AI processor)
2. Backend deployment to Render
3. Creating the CES agent, 7 sub-agents, 9 session variables, and the OpenAPI tool
4. Global security instructions and per-agent instructions
5. Connecting agents with `@` chips
6. Frontend deployment to Vercel
7. Production testing checklist and troubleshooting

**High-level flow:**

```bash
# Backend → Render (root directory: "backend")
# Build:  pip install -r requirements.txt
# Start:  uvicorn main:app --host 0.0.0.0 --port $PORT

# Frontend → Vercel
npm install -g vercel
vercel login
vercel --prod
```

⚠️ After deploying the backend, update the `servers` URL in `openapi_tool.json` to your Render URL **before** pasting it into CES.

---

## Environment Variables

Set in the Render dashboard (backend service):

| Variable | Required | Default | Description |
|---|---|---|---|
| `GOOGLE_PROJECT_ID` | Yes | – | GCP project ID (used for Document AI) |
| `GOOGLE_LOCATION` | Yes | `us` | GCP region for the Document AI processor |
| `DOCUMENT_AI_PROCESSOR_ID` | No | *(empty)* | Empty → invoice analysis returns mock data |
| `TIMEZONE_OFFSET` | No | `0` | Hours from UTC (e.g. `5.5` for IST, `-5` for EST) |
| `BANK_API_BASE` | No | `http://localhost:9090` | Bank API base URL |
| `CRM_API_BASE` | No | `http://localhost:9091` | CRM API base URL (manager availability, callbacks) |
| `PORT` | No | – | Set automatically by Render — do **not** set manually |

**Where each identifier is used:**

| Identifier | Appears in | How to find it |
|---|---|---|
| GCP Project ID | CES agent, `index.html`, backend env vars | GCP Console → project dropdown |
| CES App ID | `index.html` (`deploymentName`) | CES → Settings → Agent ID (`apps/APP_ID/`) |
| CES Deployment ID | `index.html` (`deploymentName`) | CES → Deploy → Web widget → deployment name |
| Render Backend URL | `openapi_tool.json` (`servers`) | Render dashboard → web service → URL |

---

## Testing Checklist

```bash
curl https://YOUR_RENDER_URL/health
curl -X POST https://YOUR_RENDER_URL/tools/create-application \
  -H "Content-Type: application/json" \
  -d '{"product":"debit_card","customer_name":"John"}'
curl -X POST https://YOUR_RENDER_URL/tools/manager-check \
  -H "Content-Type: application/json" -d '{}'
```

In the widget:

- `hi` → Eva asks for your name → greets you → shows the 8-item menu
- `cards` → routes to `card_services_agent`
- `start application` → creates and displays an `APP-XXXXXXXX` reference
- `check my status` → shows application status from the backend
- `main menu` → returns to the full menu
- Upload an invoice image → Gemini reads it → asks if you want to pay
- Click the mic and speak → voice is processed by CES
- `speak to a manager` → checks availability, or schedules a callback
- `what is my account number` → refused for security reasons

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Backend returns 404 on tool calls | `servers` URL in `openapi_tool.json` must exactly match your Render URL |
| "operation not found" | `operationId` in the spec must match what the agent calls (e.g. `createApplication`) |
| 429 Too Many Requests | Google rate limit — wait 30–60s, or create a new CES deployment |
| Widget blank / "Failed to connect" | Verify `deploymentName` matches exactly, token broker config, and `url-allowlist="*"` |
| Gemini won't read the invoice image | `invoice_agent` must be told to use vision; model must be `gemini-2.5-flash` |
| Session variable not set | Names in instructions must match the Variables tab exactly |
| Render deployment fails | Root directory must be `backend`; check build logs |
| Routing stuck in root | `@agent` chips must be real chips — type `@` and pick from the popup |
| Voice input not working | `enable-audio-input` must be on the container; requires HTTPS |

More issues (Document AI mock data, etc.) are covered in [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md).
