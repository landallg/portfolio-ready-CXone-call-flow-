# Portfolio‑Ready CXone Call Flow architecture

## 1. Overview

This project demonstrates a production‑style CXone inbound voice flow with:

- Menu routing
- Data dips (Studio + external API)
- Callback logic
- Error handling and failover
- Logging and monitoring
- Transfer logic to skills and agents

---

## 2. High‑level architecture

**Channel entry:**

- **Inbound PSTN call** → carrier → CXone voice entry point (DNIS / ACD skill).

**Core CXone components:**

- **Studio script:** main IVR logic (menus, data dips, callback, transfers).
- **ACD skills:** queues for Sales, Support, Billing, etc.
- **Agents:** MAX/Agent app logged into skills.

**External integrations:**

- **HTTP data dip:** REST API (CRM / account service) for caller lookup and routing attributes.
- **Optional AWS components:**  
  - **Lambda:** lightweight transformation or orchestration.  
  - **S3:** store logs/exports if needed.  
  - **CloudWatch:** monitor Lambda/API health.

**Observability:**

- **Studio logging:** custom log lines at key decision points.
- **CXone reporting:** ACD/IVR reports, callback performance.
- **External monitoring:** API health, latency, error rates.

---

## 3. Call flow summary

1. **Call entry & normalization**
2. **Greeting & language selection (optional)**
3. **Main menu routing**
4. **Data dip to external system**
5. **Business logic decisions (VIP, delinquent, standard, etc.)**
6. **Offer callback (if wait time high)**
7. **Error handling & failover**
8. **Logging & monitoring**
9. **Transfer to skills/agents**

---

## 4. Menu routing design

**Goals:**

- Keep menus shallow and intuitive.
- Make it easy to extend without breaking the flow.

**Example main menu:**

- **1 – Sales**
- **2 – Support**
- **3 – Billing**
- **0 – Operator / General**

**Key design points:**

- **Input handling:**  
  - **Valid input:** route to the correct branch.  
  - **Invalid input:** play error prompt, allow limited retries, then failover (operator or disconnect with message).
- **Timeouts:**  
  - If no input, replay menu once, then route to default (e.g., operator or general queue).
- **Extensibility:**  
  - Menu options map to **named labels** or **sub‑scripts**, not hard‑coded skill IDs.

---

## 5. Data dips (Studio + external API)

**Purpose:**

- Personalize routing and screen pops.
- Apply business rules (VIP, delinquent, high‑value product, etc.).

**Typical data dip sequence:**

1. **Collect identifier:**
   - Use **ANI** or prompt for **account number** / **order ID**.
2. **Prepare request:**
   - Build JSON payload with caller ID, account number, and call context.
3. **HTTP request block (Studio):**
   - Method: `GET` or `POST` to external API.
   - Include auth (API key / token) via headers.
4. **Parse response:**
   - Extract fields like:
     - **customerType** (VIP, standard)
     - **status** (active, delinquent)
     - **preferredSkill** (Sales, Support, Billing)
5. **Routing decision:**
   - Map response to:
     - Target skill
     - Priority
     - Screen pop data

**Defensive design:**

- **Timeouts:** short, reasonable HTTP timeout.
- **Fallback:** if API fails, route to a safe default skill and log the failure.
- **No hard dependency:** the call must still complete even if the data dip is down.

---

## 6. Callback logic

**When to offer callback:**

- Estimated wait time (EWT) above threshold.
- Queue depth above threshold.
- Specific skills only (e.g., Support, not Operator).

**Callback flow:**

1. **Check EWT / queue conditions.**
2. **Offer callback option:**
   - “Press 1 to receive a callback and keep your place in line.”
3. **Collect callback number:**
   - Use caller’s ANI by default.
   - Optionally allow entering a different number.
4. **Confirm number:**
   - Read back the number and confirm.
5. **Create callback request:**
   - Use CXone callback object / skill.
6. **Exit IVR:**
   - Play confirmation message and end call.

**Design considerations:**

- **Retry logic:** if invalid number, allow limited retries.
- **Logging:** log callback creation with correlation ID.
- **Reporting:** ensure callback skill is visible in CXone reports.

---

## 7. Error handling and failover paths

**Types of errors:**

- **Menu errors:** invalid input, no input.
- **Data dip errors:** HTTP timeout, non‑200 response, parse failure.
- **Platform issues:** skill unavailable, transfer failure.

**Strategies:**

- **Menu errors:**
  - Limit retries (e.g., 2–3).
  - After max retries, route to operator or play polite disconnect message.
- **Data dip errors:**
  - Catch non‑success codes.
  - Set a flag like `dataDipStatus = FAILED`.
  - Route to default skill with generic prompts.
- **Transfer errors:**
  - If transfer to primary skill fails, attempt:
    - Backup skill (e.g., general queue).
    - Operator / reception.
    - Fallback message and disconnect as last resort.

**Principle:**

- The caller should **never be stuck** in a dead end.
- Every branch has a **safe exit**.

---

## 8. Logging and monitoring strategy

**What to log in Studio:**

- **Call entry:** timestamp, DNIS, ANI, entry script version.
- **Menu choices:** which option selected, retries, timeouts.
- **Data dip:** request ID, success/failure, key routing attributes (not sensitive data).
- **Callback:** created/not created, callback number (masked if needed), callback skill.
- **Errors:** type, location in script, fallback path taken.
- **Transfers:** target skill, priority, any overrides.

**Where logs go:**

- **Studio log output:** for troubleshooting and development.
- **CXone reporting:** ACD/IVR reports for volumes, abandon, service level.
- **External monitoring (optional):**
  - Send key events to:
    - **CloudWatch** (if using AWS)
    - **Log aggregation** (Splunk, ELK, etc.)

**Monitoring focus:**

- Data dip failure rate.
- Callback success rate and completion.
- Menu option distribution (are callers confused?).
- Transfer failures or unusual routing patterns.

---

## 9. Transfer logic to skills and agents

**Skill mapping:**

- **Sales:** `SKILL_SALES`
- **Support:** `SKILL_SUPPORT`
- **Billing:** `SKILL_BILLING`
- **Operator:** `SKILL_OPERATOR` or direct extension.

**Routing inputs:**

- **From menu:** direct mapping (1 → Sales, 2 → Support, etc.).
- **From data dip:** override menu choice if business rules require (e.g., delinquent → Billing).
- **From business rules:** VIP → higher priority or dedicated skill.

**Transfer behavior:**

- **Set call variables:**
  - `customerType`, `accountId`, `reasonCode`, etc.
- **Screen pop:**
  - Pass variables to agent desktop for context.
- **Priority:**
  - Increase priority for VIP or high‑value segments.
- **Overflow:**
  - If primary skill overloaded, overflow to backup skill with clear tagging.

---

## 10. Security and configuration

**Security:**

- Do not log sensitive data (full card numbers, SSN, etc.).
- Mask or truncate identifiers where appropriate.
- Use secure transport (HTTPS) for all data dips.

**Configuration management:**

- Externalize:
  - API URLs
  - Timeouts
  - Thresholds (EWT, queue depth)
  - Skill IDs
- Keep them in **Studio parameters** or **config files**, not hard‑coded.

---

