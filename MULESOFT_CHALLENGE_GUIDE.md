# MuleSoft Challenge - Complete Study & Build Guide

> Written for: Ahmed | Company: QiCard (Integration team) | Email received: Oct 7, 2026
> Target: be ready for the live coding session in **5-7 days (by ~Oct 14)**
> Rule: **you** build it step by step; Claude helps when you're blocked. Nothing is committed or pushed for you.

---

## 0. Read this first (the big picture)

They are **not** testing whether you are a MuleSoft expert. The email says it plainly:

> "assess how quickly you can understand MuleSoft concepts, build a small integration, and explain your approach."

What they evaluate (from the email, in their words):

| They evaluate | What it means for you |
|---|---|
| Backend fundamentals | HTTP, status codes, REST, JSON, validation, error handling - you already know this |
| Learning ability | Show you picked up Mule fast: use the right Mule building blocks, not hacks |
| Problem-solving approach | When something breaks, debug calmly: logger, Evaluate Expression, Studio debugger |
| Code clarity | Clean flows, good names, config in properties, small DataWeave scripts |
| API & integration understanding | Experience/Process/System layers, HTTP Request connector, downstream errors |
| Communication | Explain WHY you designed it that way, out loud, simply |

**Your advantage:** you're a backend dev. Mule is "just" a visual/XML way of doing what you already do in code (routing, validation, calling another service, mapping JSON, handling errors). Every Mule concept below is mapped to something you already know.

**The live session** may ask you to: modify the API, add a DataWeave transformation, consume an API, handle an error, debug a flow, or explain design decisions. So the goal is not just "finished project" but **"I understand every line of my project well enough to change it in 15 minutes while someone watches."**

---

## 1. The requirements, simplified

### 1.1 Deliverables checklist

- [ ] Mule 4 project in Anypoint Studio
- [ ] 3 endpoints:
  - [ ] `GET /customers/{customerId}` -> customer details
  - [ ] `POST /customers` -> create customer
  - [ ] `GET /customers` -> customer list
- [ ] JSON in, JSON out
- [ ] Validate required request fields (POST)
- [ ] Correct HTTP status codes: **200, 201, 400, 404, 500** (and 502/504 per the sample files)
- [ ] Data store: in-memory list / static JSON / mock data (**no database needed**)
- [ ] `GET /customers/{customerId}` calls an **external service** (public mock API or your own local mock backend)
  - [ ] Uses the **HTTP Request connector**
  - [ ] Backend URL stored in a **configuration property** (not hardcoded)
  - [ ] **Timeout** set
  - [ ] Handle downstream **404, other 4xx, 5xx, timeout**
- [ ] **DataWeave** for request and response mapping
- [ ] **Controlled error handling** + **standard error response** format
- [ ] **Logging with a correlation ID**
- [ ] **No sensitive data in logs** (email, mobile, DOB, address...)
- [ ] **Environment-based config** through property files (dev / test / prod style)
- [ ] **Git repository** (on GitHub) with source code
- [ ] **README** with how to run and test
- [ ] If you create your own JSON spec/sample files, include them in the repo (the provided ones too)
- [ ] Tell them when you're ready for the live session

### 1.2 What the sample JSON files tell you

The zip (`ahmed-mulesoft-test/`) is basically your **API contract**. Treat it as the spec.

**Requests**

| File | Purpose |
|---|---|
| `requests/create-customer-request.json` | Valid POST body. Fields: `firstName`, `lastName`, `email`, `mobileNumber`, `dateOfBirth`, `address{city,country,postalCode}` |
| `requests/create-customer-invalid-request.json` | Invalid POST body: **no `lastName`**, **bad email** (`not-a-valid-email`). Use this to test the 400 path |

**Responses**

| File | HTTP | Meaning |
|---|---|---|
| `customer-200.json` | 200 | Single customer: all fields + `customerId`, `status`, `createdAt` |
| `customers-200.json` | 200 | List: `{ "customers": [ ...summary... ], "count": 2 }` - note: **summary fields only** (no DOB/address) |
| `create-customer-201.json` | 201 | Created customer = request fields + server-generated `customerId` (`CUST-10001`), `status: "ACTIVE"`, `createdAt` (ISO UTC) |
| `error-400.json` | 400 | `VALIDATION_ERROR` + `details[]` of `{field, message}` |
| `error-404.json` | 404 | `CUSTOMER_NOT_FOUND` |
| `error-500.json` | 500 | `INTERNAL_ERROR` |
| `error-502.json` | 502 | `DOWNSTREAM_SERVICE_ERROR` (backend returned an error) |
| `error-504.json` | 504 | `DOWNSTREAM_TIMEOUT` (backend didn't answer in time) |

**Things to notice (interviewers love when you notice these):**

1. **One standard error shape everywhere:**
   ```json
   { "error": { "code": "...", "message": "...", "correlationId": "...", "details": [ {"field":"...","message":"..."} ] } }
   ```
   `details` only exists on validation errors.
2. **Every error carries `correlationId`** -> it must be the same ID you log. This is how support traces a request.
3. **The list response differs from the single response** (summary vs full) -> you need a DataWeave mapping for each.
4. **Server-generated fields** on create: `customerId`, `status`, `createdAt`. The client must not send these.
5. **502 and 504 are not in the email's list** (200/201/400/404/500) but the files include them. That is the expected mapping for "downstream error" vs "downstream timeout". Implement them.
6. The valid request has **7 top-level fields**; the invalid one is missing `lastName` and has a bad email -> matches the two `details` entries in `error-400.json` exactly. Your validation should be able to reproduce that output.
7. Error 404 message includes the ID: `"Customer CUST-99999 was not found."` -> dynamic message.

---

## 2. Concepts you must understand (backend-dev translations)

### 2.1 MuleSoft & Anypoint Studio

- **MuleSoft** = an integration platform. You build **applications** that receive a message (HTTP, file, queue...), transform it, call other systems, and respond.
- **Mule 4** = the current runtime (don't read Mule 3 tutorials - syntax differs a lot).
- **Anypoint Studio** = Eclipse-based IDE. You drag components on a canvas, but everything is saved as **XML** in `src/main/mule/*.xml`. Learn to read both the canvas and the XML ("Configuration XML" tab). Interviewers may ask you to change XML directly.
- **Maven** is the build tool under the hood (`pom.xml`).
- Project layout:
  ```
  src/main/mule/        -> flow XML files
  src/main/resources/   -> properties files, DataWeave (.dwl), JSON samples, api spec
  src/test/munit/       -> tests (optional bonus)
  pom.xml               -> dependencies (connectors)
  mule-artifact.json    -> app metadata / min runtime version
  ```

### 2.2 API-led connectivity (Experience / Process / System)

| Layer | Job | Analogy |
|---|---|---|
| **Experience API** | Shaped for one consumer (mobile app, web). Exposes what the client needs | Controller / BFF |
| **Process API** | Business logic/orchestration across several systems | Service layer |
| **System API** | Thin wrapper over **one** system of record (DB, core banking, CRM) | Repository / adapter |

For this challenge: **your Mule app = the Customer API (Experience/Process-ish)**, and the **mock backend = the System API/system of record**. Be able to say this in the interview:

> "The Mule app exposes a clean, consumer-facing contract and hides the backend's shape. If the backend changes, I only change the mapping in one place."

### 2.3 Flows, subflows, private flows

| Thing | Has event source (listener)? | Own error handling? | Use for |
|---|---|---|---|
| **Flow** | Yes (or called via Flow Reference) | Yes | Entry points, units with own error strategy |
| **Sub-flow** | No | **No** (uses caller's) | Reusable chunk of logic (e.g., "log + set correlation") |
| **Private flow** | No (no source) | Yes | Reusable logic that needs its own error handling |

Call them with **Flow Reference**. Think: flow = function with try/catch, sub-flow = inline helper.

### 2.4 The Mule Event (MOST IMPORTANT concept)

Every message moving through a flow is an **event**:

| Part | What it is | Mutable? | Example |
|---|---|---|---|
| `payload` | The main data | Yes (each processor can replace it) | request body, response from backend |
| `attributes` | Metadata of the message **source/last connector** | Read-only-ish (replaced by next connector) | HTTP: `attributes.uriParams.customerId`, `attributes.queryParams`, `attributes.headers`, `attributes.statusCode` |
| `vars` | Variables **you** store in the flow | Yes (via Set Variable) | `vars.customerId` |
| `correlationId` | Built-in ID for tracing this event | Read-only | `correlationId` |
| `error` | Only exists in an **error handler** | - | `error.errorType.identifier`, `error.description`, `error.cause` |

**Gotcha #1:** after the HTTP Request connector runs, `payload` becomes the **backend's response** and `attributes` becomes the **backend's HTTP attributes** (the listener's attributes are gone!). Fix: save what you need in a **variable** BEFORE the call (e.g., `vars.customerId`).

**Gotcha #2:** `payload` after `Logger` is unchanged; after `Transform Message` it is the transform output; after `Set Payload` it's whatever you set.

**"Predefined variables"** the email mentions = built-ins you didn't declare yourself: `payload`, `attributes`, `vars`, `error`, `correlationId`, `message`, `server`, `app`, `p()` (property function) etc.

### 2.5 Connectors & global configuration

- A **connector** is a plug-in module (HTTP, Validation, File, DB...). Added via the Mule Palette ("Search in Exchange" if missing) -> shows up in `pom.xml`.
- A **global config element** (in `global-config.xml` or similar) holds shared connection settings. You'll need:
  - **HTTP Listener config** (host `0.0.0.0`, port from properties)
  - **HTTP Request config** (base URL/host/port/protocol from properties, timeout)
  - **Configuration properties** (file name)
  - **APIkit Router config** (if you use APIkit)
- Operations reference configs by name (`config-ref`).

### 2.6 Properties & environments

- Put values in `src/main/resources/config-dev.yaml` (or `.properties`).
- Load with a **Configuration properties** global element using `${env}`: `file="config-${env}.yaml"`.
- Reference with `${http.listener.port}` in XML or `p('backend.baseUrl')` in DataWeave.
- Choose env at runtime: `-Denv=dev` (Studio: Run Configurations -> Arguments -> VM arguments). Default it with a `mule.env`/`env` property.
- Secrets (passwords/API keys) belong in **secure properties** (encrypted) - for this test just mention it; don't commit real secrets.
- **Why interviewers care:** "no hardcoded URLs/ports/timeouts anywhere".

### 2.7 APIkit vs hand-written listener

Two valid designs:

- **A) APIkit (contract-first):** write a RAML/OAS spec -> Studio generates a main flow + one flow per endpoint, validates requests against the spec, and raises `APIKIT:BAD_REQUEST`, `APIKIT:NOT_FOUND`, `APIKIT:METHOD_NOT_ALLOWED`... automatically. This is what companies actually use and a **strong signal** in an interview. Slightly more to learn.
- **B) Plain HTTP Listener + Choice router:** one listener per path, manual routing. Simpler, fewer moving parts.

**Recommendation:** Do **A (APIkit with an OpenAPI/RAML spec)** if you have time, because the sample JSON files can become the spec examples and the spec is a deliverable "you may be asked to modify". If stuck, fall back to B. Either way you must be able to explain the choice.

### 2.8 Error handling in Mule 4

- Errors have a **type** `NAMESPACE:IDENTIFIER`, e.g. `HTTP:NOT_FOUND`, `HTTP:TIMEOUT`, `HTTP:CONNECTIVITY`, `HTTP:INTERNAL_SERVER_ERROR`, `VALIDATION:INVALID_EMAIL`, `APIKIT:BAD_REQUEST`, `MULE:EXPRESSION`, `ANY`.
- **On Error Propagate** = handle error, run the steps, then **re-throw** (flow ends with error; the HTTP listener then uses the error response). Use this when you want the listener to send the error response and the transaction/flow to be marked failed.
- **On Error Continue** = handle error, run steps, then the flow **continues as success** (the last payload becomes the success response).
- **Scopes:** a **Try** scope handles errors locally inside part of a flow; an **error-handler** at flow level covers the whole flow; **global error handler** (a named `<error-handler>` + default error handler config) covers all flows. Best practice: one global/shared handler so the response shape is consistent.
- Matching: each handler can match by `type="HTTP:NOT_FOUND"` (specific) or `when="#[...]"` (expression). Order matters: **first match wins**, so put specific types before `ANY`.
- **Setting HTTP status on the listener response:** in the HTTP Listener's **Responses** tab (success + error response), use `statusCode` expression, usually from a variable: `#[vars.httpStatus default 200]`. For errors set `vars.httpStatus` in the handler.

**Planned error mapping (this is your design - memorise it):**

| Situation | Mule error type | HTTP status | Error `code` |
|---|---|---|---|
| Missing/invalid field in POST body | `VALIDATION:*` or `APIKIT:BAD_REQUEST` | 400 | `VALIDATION_ERROR` |
| Customer id not in my data / backend returns 404 | `HTTP:NOT_FOUND` (on the Request connector) | 404 | `CUSTOMER_NOT_FOUND` |
| Backend returns other 4xx | `HTTP:BAD_REQUEST`, `HTTP:UNAUTHORIZED`, ... | 502 (your call - justify it) | `DOWNSTREAM_SERVICE_ERROR` |
| Backend returns 5xx | `HTTP:INTERNAL_SERVER_ERROR`, `HTTP:SERVICE_UNAVAILABLE` | 502 | `DOWNSTREAM_SERVICE_ERROR` |
| Backend slow | `HTTP:TIMEOUT` | 504 | `DOWNSTREAM_TIMEOUT` |
| Backend unreachable (down, refused) | `HTTP:CONNECTIVITY` | 502 (or 503) | `DOWNSTREAM_SERVICE_ERROR` |
| Unknown path | `APIKIT:NOT_FOUND` | 404 | `NOT_FOUND` |
| Anything else | `ANY` | 500 | `INTERNAL_ERROR` (never leak stack traces) |

> Design question you will get asked: *"Why 502 for downstream 4xx?"* Good answer: from the caller's point of view their request was fine; the failure is in the upstream chain, so a gateway-type error (502) is more honest than blaming the caller. Exception: downstream 404 for the customer is a legit "not found" for the caller, so we map it to 404.

### 2.9 Logging & correlation ID

- Mule 4 already creates a `correlationId` for every event. If the caller sends header `X-Correlation-ID`, the HTTP Listener can use it (configure via the **"Correlation ID" header** setting / or set it explicitly) - otherwise Mule generates a UUID.
- Plan:
  1. Accept `X-Correlation-ID` if provided; else use Mule's generated one.
  2. Log it in **every** log line: `INFO` start, `INFO` calling backend, `INFO` end/error.
  3. Return it in the error body and as a response header `X-Correlation-ID`.
  4. **Pass it to the backend** as an `X-Correlation-ID` header on the HTTP Request.
- Logger message example (no PII):
  ```
  "[#[correlationId]] GET /customers/{id} started customerId=#[vars.customerId]"
  ```
- **Never log:** email, mobile number, date of birth, address, full request/response payloads. Customer ID is generally OK (it's an internal identifier), but be ready to justify it. Log **field names that failed validation**, not their values.
- Logger levels: `INFO` normal flow, `WARN` handled client errors (4xx), `ERROR` unexpected/5xx.

### 2.10 HTTP Request connector (consuming the backend)

Config you set (all from properties):
- Protocol, Host, Port, Base path
- **Response timeout** (e.g., 5000 ms) and **connection timeout**
- Operation: Method `GET`, Path `/customers/{id}` with **URI parameters** (`#[vars.customerId]`) - don't concatenate strings by hand.
- Headers: `X-Correlation-ID`, `Accept: application/json`
- **Response validation:** by default Mule raises an error for non-2xx statuses (that's why `HTTP:NOT_FOUND` etc. exist). Don't turn this off - rely on it.
- **Reconnection/retry:** be ready to explain; for a GET that is idempotent a small retry is OK, for POST be careful (duplicates).

### 2.11 DataWeave 2.0 (the thing you'll practice the most)

DataWeave = functional transformation language. Every script has a **header** (directives) and a **body** after `---`.

```dataweave
%dw 2.0
output application/json
---
{ ... }
```

Concepts to master (in this order):
1. Selectors: `payload.firstName`, `payload.address.city`, `attributes.uriParams.customerId`
2. Output building: objects `{}`, arrays `[]`
3. `default`: `payload.status default "ACTIVE"`
4. `map`, `filter`, `pluck`, `mapObject`
5. String ops: `upper()`, `trim()`, `++` (concat)
6. `if / else`, `match`
7. Dates/time: `now()`, `now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}` (UTC), `as Date`
8. IDs: `uuid()`, or a counter via `vars` / Object Store
9. Variables in header: `var x = ...`, functions `fun f(a) = ...`
10. Omitting nulls: `payload filterObject ((v) -> v != null)`
11. Properties: `p('backend.baseUrl')`
12. Referring to the error: `error.description`, `error.errorType.identifier`

**Scripts you'll need to write (and be able to rewrite from memory):**

*(a) Build the 201 response from the request:*
```dataweave
%dw 2.0
output application/json
---
{
  customerId: vars.newCustomerId,
  firstName: payload.firstName,
  lastName: payload.lastName,
  email: payload.email,
  mobileNumber: payload.mobileNumber,
  dateOfBirth: payload.dateOfBirth,
  address: payload.address,
  status: "ACTIVE",
  createdAt: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}
```
*(b) List response (summary fields + count):*
```dataweave
%dw 2.0
output application/json
var customers = vars.customerStore
---
{
  customers: customers map (c) -> {
    customerId: c.customerId,
    firstName: c.firstName,
    lastName: c.lastName,
    email: c.email,
    mobileNumber: c.mobileNumber,
    status: c.status
  },
  count: sizeOf(customers)
}
```
*(c) Standard error response:*
```dataweave
%dw 2.0
output application/json
---
{
  error: {
    code: vars.errorCode,
    message: vars.errorMessage,
    correlationId: correlationId
  } ++ (if (vars.errorDetails != null) { details: vars.errorDetails } else {})
}
```
*(d) Backend-to-API mapping (if backend field names differ from your contract):* map them explicitly, never pass the backend payload through unchanged. This is "DataWeave mapping for responses".

**Rule:** if you can't explain a DataWeave line, you can't keep it in the project.

### 2.12 Where to keep customer data (no DB)

Pick ONE and explain it:
- **Object Store connector** (in-memory, Mule-native) - good: shows real Mule knowledge, supports `store/retrieve/retrieveAll`.
- **Static JSON file** read at startup / each call (`classpath` resource, `readUrl("classpath://data/customers.json", "application/json")`) - simplest, read-only.
- **Variable/ in-memory in a flow** - doesn't persist across requests; **don't** rely on it for POST.

**Recommendation:** Seed from `customers.json`, keep created customers in an **Object Store** so `POST` then `GET` works within a running app. Mention it resets on restart (in-memory).

Remember `GET /customers/{id}` must call the **backend**. So a reasonable split:
- `GET /customers` + `POST /customers` -> local store (Object Store / JSON)
- `GET /customers/{id}` -> HTTP Request to mock backend (this is what the task says)
Be ready to be asked "what if POST'd customers should also be returned by `GET /{id}`?" (Answer: the System API/backend should own the data; POST would call the backend too; I kept the scope as specified.)

---

## 3. Study plan (7 days, ~3-4 hours/day; compress to 5 if needed)

> Strategy: **learn -> immediately do** each day. Don't binge videos; every day ends with something that runs.

### Day 0 (today, ~1h) - Setup
- [ ] Install **Anypoint Studio** (latest 7.x) + the Mule runtime it offers (Mule 4.x). Check the Java version it requires (Java 17 for recent runtimes) and install the JDK Studio asks for.
- [ ] Install **Postman** (or Insomnia) / have `curl`.
- [ ] Install Node.js (for the mock backend, see section 5) - or choose WireMock/Mockoon.
- [ ] Create GitHub repo (empty, you will push manually).
- [ ] Create a free **Anypoint Platform** account (needed for Exchange connectors/downloads, free trial).

### Day 1 - Fundamentals + Hello World
Learn:
- What is Mule 4, API-led connectivity, Studio UI tour (Package Explorer, Mule Palette, Canvas, Console, Problems, Debug).
- Event model: payload / attributes / vars (section 2.4).
Do:
- [ ] Create project `customer-api`.
- [ ] Flow: **HTTP Listener** (`GET /hello`) -> **Set Payload** -> **Logger**. Run, call from Postman.
- [ ] Add **Set Variable** and read `attributes.queryParams` (`/hello?name=Ahmed`) and return `"Hello Ahmed"`.
- [ ] Put the port in a properties file and load it.
Checkpoint question: *What's in `payload` and `attributes` after the listener? After a Set Variable?*

### Day 2 - DataWeave basics
Learn: sections 2.11 items 1-8.
Do (use the **DataWeave Playground** online + Transform Message preview):
- [ ] Transform `requests/create-customer-request.json` -> `responses/create-customer-201.json` (server-generated fields).
- [ ] Transform a list of full customers -> `customers-200.json` shape (summary + count).
- [ ] Add `default`, `if/else`, `map`, `filter` exercises (e.g., only ACTIVE customers).
Checkpoint: write script (a) and (b) from section 2.11 without looking.

### Day 3 - REST API skeleton (3 endpoints with mock data)
Learn: Flows vs sub-flows, Flow Reference, Choice router, APIkit (optional), config files.
Do:
- [ ] Write the API spec (OpenAPI 3 or RAML) using the sample JSON as examples. Put it in `src/main/resources/api/`.
- [ ] Generate the flows (APIkit) or build listeners/choice by hand.
- [ ] `GET /customers` returns the list; `POST /customers` returns 201 + created customer; `GET /customers/{id}` returns from mock data (for now) or 404.
- [ ] Test all three in Postman.
Checkpoint: set a status code on the listener response from a variable.

### Day 4 - Validation + error handling
Learn: section 2.8; Validation module (`Is not blank string`, `Is email`, `Is not null`...); `On Error Propagate` vs `Continue`.
Do:
- [ ] Install the **Validation** module.
- [ ] Validate the POST body: required = `firstName`, `lastName`, `email`, `mobileNumber` (decide about `dateOfBirth` and `address` and write it in the README). Collect **all** failed fields into `details[]` (not just the first one) -> reproduces `error-400.json`.
- [ ] Global error handler: map error types -> standard error JSON (section 2.8 table).
- [ ] Test with `create-customer-invalid-request.json` -> exact 400 body.
Checkpoint: explain Propagate vs Continue to a rubber duck in 30 seconds.

### Day 5 - HTTP Request connector + mock backend
Learn: section 2.10, URI params, timeouts, downstream error types.
Do:
- [ ] Build the mock backend (section 5) with these routes: success, 404, 400, 500, and a slow route (sleeps longer than your timeout).
- [ ] Put backend host/port/path/timeouts in `config-*.yaml`.
- [ ] `GET /customers/{id}`: save `vars.customerId` -> Logger -> HTTP Request -> Transform to the customer-200 shape.
- [ ] Trigger and verify each: 200, downstream 404 -> 404, downstream 500 -> 502, timeout -> 504, backend stopped -> 502.
Checkpoint: why do we save `customerId` in a variable before the Request?

### Day 6 - Engineering practices
Do:
- [ ] Correlation ID: accept/generate, log everywhere, send to backend, return in header + error body.
- [ ] Review all loggers: **no PII**.
- [ ] Environments: `config-dev.yaml`, `config-test.yaml`, `config-prod.yaml` (different ports/timeouts/log level); run with `-Denv=...`.
- [ ] (Bonus) 2-3 **MUnit** tests: 200, 400, 404.
- [ ] Clean names: flows `get-customer-by-id-flow`, vars `customerId`, files grouped (`api.xml`, `global-config.xml`, `error-handling.xml`, `customer-impl.xml`).
- [ ] Debug practice: put a breakpoint, step through, inspect payload/vars/attributes.

### Day 7 - Polish, README, rehearsal
- [ ] README (section 7 template).
- [ ] Add Postman collection or `curl` list + the provided JSON files to the repo.
- [ ] `.gitignore` (target/, .mule/, .studio/ ... see 6.2).
- [ ] Clean clone test: clone your repo to another folder, follow the README, make sure it runs.
- [ ] Rehearse the interview questions (section 9) **out loud**.
- [ ] Mock live-coding drills (section 8).
- [ ] Email them: "ready for the live session".

> If you fall behind: protect these in order -> (1) 3 endpoints working, (2) error handling + status codes, (3) HTTP Request + timeout, (4) correlation id + safe logs, (5) README. APIkit, MUnit and 3 environments are "nice to have".

---

## 4. Suggested architecture & project structure

```
GET/POST  http://localhost:8081/api/customers...
      |
 [Mule: customer-api]                      [Mock backend: localhost:3000]
  listener -> validate -> flow -> HTTP Request ----------------------> /backend/customers/{id}
      ^            |                              <---- JSON / 404 / 5xx / delay
      |        global error handler -> standard error JSON (+ correlationId)
```

Suggested files:

```
customer-api/
  README.md
  pom.xml
  src/main/mule/
    customer-api.xml          # APIkit main flow + router (or listeners)
    customer-implementation.xml  # business flows (get-by-id, get-all, create)
    global-config.xml         # listener, request, properties configs
    error-handling.xml        # global error handler
    common.xml                # sub-flows: log-start, set-correlation, etc.
  src/main/resources/
    config-dev.yaml  config-test.yaml  config-prod.yaml
    api/customer-api.raml (or .yaml)
    dwl/ (optional reusable .dwl scripts)
    mock-data/customers.json
  mock-backend/               # your local mock (Node/WireMock/Mockoon)
  samples/                    # the provided requests/ and responses/ folders
  postman/customer-api.postman_collection.json   (nice to have)
```

Example properties (`config-dev.yaml`):
```yaml
http:
  listener:
    host: "0.0.0.0"
    port: "8081"
backend:
  protocol: "HTTP"
  host: "localhost"
  port: "3000"
  basePath: "/backend"
  responseTimeoutMs: "3000"
  connectionTimeoutMs: "2000"
logging:
  level: "INFO"
```

---

## 5. Mock backend options (pick the easiest for you)

| Option | Effort | Notes |
|---|---|---|
| **Small Node.js script** (plain `http` or express) | Low for you | Full control; simulate delay with `setTimeout`. **Recommended** - you know Node |
| **WireMock standalone** (jar) | Low | JSON mapping files; built-in `fixedDelay` |
| **Mockoon** (GUI) | Lowest | Click to create routes + latency |
| Public mock (`mockbin`, `jsonplaceholder`, `reqres`) | Lowest | Shapes don't match; can't force timeouts/5xx reliably -> weaker demo |

Behaviour your mock should support (use special IDs to trigger cases):

| Request | Mock returns |
|---|---|
| `GET /backend/customers/CUST-10001` | 200 + customer JSON |
| `GET /backend/customers/CUST-99999` | 404 |
| `GET /backend/customers/CUST-BAD` | 400 |
| `GET /backend/customers/CUST-ERR` | 500 |
| `GET /backend/customers/CUST-SLOW` | 200 after > timeout (e.g., 10 s) |
| mock **stopped** | connection refused -> `HTTP:CONNECTIVITY` |

Put the mock in the repo (`mock-backend/`) and document how to start it in the README. Graders must be able to reproduce your demo in two commands.

---

## 6. Build order (step by step, with what to check at each step)

1. **Skeleton:** project, global config, properties, listener on `/api/*`. Check: app starts, port from properties.
2. **Contract:** spec file with the 3 endpoints + examples from the samples.
3. **Mock data endpoints:** `GET /customers`, `POST /customers` (store), responses via DataWeave. Check vs `customers-200.json`, `create-customer-201.json`.
4. **Validation:** required fields, email format; build `details[]`. Check vs `error-400.json`.
5. **Error handler:** one place, standard JSON, correct status codes. Check vs `error-404/500.json`.
6. **Backend + HTTP Request:** `GET /customers/{id}` -> backend. Check vs `customer-200.json`.
7. **Downstream error mapping:** 404/4xx/5xx/timeout/connectivity. Check vs `error-502/504.json`.
8. **Correlation ID + safe logging:** grep your console output for email/phone - must find none.
9. **Environments:** switch `-Denv`, verify behaviour changes (e.g., tiny timeout in `test` to force 504).
10. **README + repo cleanup + clean-clone test.**

### 6.2 `.gitignore` (minimum)
```
target/
.mule/
.studio/
.settings/
.project
.classpath
*.log
node_modules/
.DS_Store
```
(Keep `pom.xml`, `mule-artifact.json`, `src/`. Do **not** commit real secrets.)

### 6.3 Manual test commands (use in the README too)
```bash
# list
curl -i http://localhost:8081/api/customers
# get one (hits the mock backend)
curl -i http://localhost:8081/api/customers/CUST-10001
# 404 from backend
curl -i http://localhost:8081/api/customers/CUST-99999
# create (valid)
curl -i -X POST http://localhost:8081/api/customers -H "Content-Type: application/json" -d @samples/requests/create-customer-request.json
# create (invalid -> 400)
curl -i -X POST http://localhost:8081/api/customers -H "Content-Type: application/json" -d @samples/requests/create-customer-invalid-request.json
# with correlation id
curl -i http://localhost:8081/api/customers/CUST-10001 -H "X-Correlation-ID: test-123"
```

---

## 7. README template (fill it in at the end)

```markdown
# Customer API (MuleSoft 4)

## What it does
Short description + the 3 endpoints table.

## Architecture
Small diagram: client -> Mule customer-api -> mock backend. Mention API-led layers.

## Prerequisites
Anypoint Studio X, Mule runtime 4.x, JDK 17, Node.js (for the mock), Postman/curl.

## Run
1. Start the mock backend: `cd mock-backend && npm install && npm start`
2. Import project into Anypoint Studio (File > Import > Anypoint Studio > Packaged mule application / Existing project)
3. Run with VM arg `-Denv=dev` (Run Configurations > Arguments)
4. API at http://localhost:8081/api

## Configuration
Table of properties (backend.host, backend.port, backend.responseTimeoutMs, ...) and environments.

## Endpoints, examples, status codes
Table: endpoint | success | errors. Link to /samples.

## Error handling
Table from section 2.8 (error type -> HTTP status -> code).

## Logging & correlation ID
How it works; what is never logged.

## Testing
curl commands; how to trigger 404/5xx/timeout with the mock IDs. MUnit if added.

## Design decisions & assumptions
Why APIkit/Object Store; which fields are required; 4xx->502 reasoning; limitations (in-memory, resets on restart).
```

---

## 8. Live coding drills (practice these until easy)

Each should take **<= 10-15 min**. Time yourself.

1. Add a field `nationalId` (optional) to POST and the 201 response.
2. Add `GET /customers?status=ACTIVE` filter (query param).
3. Add a DataWeave transform that returns `fullName` ("Ahmed Nadhim") and masks the mobile number (`+96477****567`).
4. Make `lastName` optional and `dateOfBirth` required.
5. Change the timeout from properties and show the 504.
6. Add a new error: backend returns 429 -> map to 503 `DOWNSTREAM_RATE_LIMITED`.
7. Consume a **different** public API (e.g., a `GET https://jsonplaceholder.typicode.com/users/1`) and map to a small custom JSON.
8. Given a broken flow (payload is empty after a request, or `vars.customerId` is null) -> find the bug using the debugger/Logger.
9. Add a `PUT /customers/{id}` that updates the store.
10. Add a new environment `uat` with a different backend port.
11. Convert a flow to use a sub-flow + Flow Reference for logging.
12. Explain how you would add authentication (Client ID enforcement / API Manager policy / Basic auth) - words only.

**Debugging toolkit (say this during the session - it shows method):**
1. Read the **console** stack: error type + description + which processor failed.
2. Add a **Logger** with `#[payload]`/`#[vars]` locally (remove after; never leave PII).
3. Use **Studio Debugger**: breakpoints, Mule Debugger view shows payload/attributes/vars.
4. Check the **Transform Message** preview with sample data.
5. Check the **XML** for typos (`config-ref`, flow names).
6. Reproduce with `curl` to remove client variables.
7. Common causes: payload replaced after a connector, missing `output application/json`, missing `Content-Type` header, property key typo, port already in use, wrong error type in the handler order.

---

## 9. Interview questions you should be able to answer

**MuleSoft basics**
- What is API-led connectivity? Give an example from this project.
- Difference between flow, sub-flow, private flow?
- What's in a Mule event? What does `attributes` hold after an HTTP listener vs after an HTTP Request?
- What's `vars` for? Why did you store `customerId` before calling the backend?
- What is DataWeave? `map` vs `mapObject` vs `pluck`? What does `default` do?
- On Error Propagate vs On Error Continue?
- How do you load environment-specific configuration?
- What is APIkit and what does it generate/validate for you?
- What is a connector? What is a global config element?

**Design decisions (this project)**
- Why did you pick this error mapping? Why 502 vs 504?
- Where does validation happen and why there?
- How does the correlation ID travel end to end?
- What would you change for production? (real DB/System API, secure properties, retries + circuit breaker, API Manager policies: rate limiting/client-id, MUnit coverage, CloudHub/RTF deployment, structured JSON logging, idempotency key for POST, pagination for `GET /customers`.)
- What are the limitations of your current solution? (in-memory resets, no auth, no pagination, no duplicate email check...). Saying these yourself is a **strength**.

**Backend fundamentals (they will mix these in)**
- 400 vs 404 vs 422; 401 vs 403; 502 vs 503 vs 504.
- Idempotency of GET/POST/PUT/DELETE.
- Why not return stack traces to clients?
- PII and logging; GDPR-style thinking.
- REST naming (`/customers`, plural nouns), `Location` header on 201 (optional bonus: `Location: /customers/CUST-10001`).

---

## 10. Learning resources (focus on Mule 4 only)

Free & official (best):
- **MuleSoft Developer Portal / Training** - "Getting Started with Anypoint Platform" and "Anypoint Platform Development: Fundamentals (Mule 4)" self-paced material (free on training.mulesoft.com & MuleSoft Trailhead).
- **Trailhead (Salesforce) - MuleSoft trails:** "MuleSoft Basics", "Anypoint Studio Basics" etc. (hands-on).
- **MuleSoft docs:** docs.mulesoft.com -> *Mule Runtime 4.x* -> "Mule Components", "DataWeave", "Error Handling", "Configuring Properties", "HTTP Connector".
- **DataWeave Playground:** developer.mulesoft.com/learn/dataweave/ (practice in the browser, includes tutorials) - **use daily**.
- **MuleSoft YouTube channel** (Mule 4 tutorials), and Mule Mentors / "Mule Training" community videos.
- **MuleSoft Community / StackOverflow** (`mule4`, `dataweave` tags) for specific errors.

Ask the interviewer for training material too - the email explicitly invites it. A good short question to send:
> "Which Mule runtime version and Studio version would you like me to use, should I use APIkit with RAML/OAS, and is there any preferred logging format?"
Asking this shows initiative and removes guesswork.

---

## 11. Quick glossary

| Term | Meaning |
|---|---|
| Anypoint Studio | The Mule IDE |
| Mule app | Deployable project (a `.jar`) with flows |
| Flow | A sequence of processors triggered by a source |
| Event source | What starts a flow (HTTP Listener, Scheduler, ...) |
| Processor/Component | A step in a flow (Logger, Set Variable, Transform...) |
| Scope | Wrapper component (Try, For Each, Async) |
| Router | Branching (Choice, Scatter-Gather) |
| Payload / Attributes / Vars | Event data / metadata / your variables |
| Correlation ID | Request trace ID |
| Connector | Plug-in to talk to a system/protocol |
| DataWeave (DW) | Transformation language |
| `.dwl` | DataWeave script file |
| APIkit | Tooling that builds flows from an API spec and validates requests |
| RAML / OAS | API specification languages (OpenAPI Spec) |
| Exchange | Anypoint's asset repository (connectors, specs) |
| Object Store | Mule key/value storage |
| MUnit | Mule's unit-testing framework |
| CloudHub / RTF | MuleSoft's cloud / Kubernetes runtimes (deployment - just know the names) |
| API Manager | Governance: policies, rate limits, client IDs |

---

## 12. Surprises to expect in the live session (and how to handle them)

The email lists "modify the API, add a DataWeave transformation, consume an API, handle an error, debug a flow, explain design decisions". Reading between the lines, they want to know that the project is **yours** and not memorised or AI-generated. Expect any of these:

| Surprise | Likelihood | Prep section |
|---|---|---|
| Change/extend **your** customer project live | Very high | 12.2 |
| Build a **new small Mule project from scratch** in Studio, on a screen share | High | 12.3 |
| **Code review** of a snippet in Java / C / C# (clean code, SOLID, bugs) | Medium-high (they said "backend fundamentals") | 12.1 |
| Debug a **broken flow** they prepared | High | section 8 |
| "Why did you do X instead of Y?" on any line | Certain | section 13 |
| Whiteboard-style question (REST design, HTTP codes, idempotency) | Medium | section 9 |

**Be honest about AI.** If you used AI to help learn, that is fine, but the test is whether you can do it **without** it. Practise at least the last two days with no AI open: write flows, DataWeave and error handlers yourself. If they ask whether you used AI, answer truthfully: "I used it to explain concepts and unblock myself, and I built and tested the flows myself." That's only a good answer if it's true, so make it true.

### 12.1 Code review in Java / C / C# (language you may not know well)

You don't need to know every language. You need a **review method** that works on any of them, and you say it out loud.

**The 6-pass method (say each pass as you do it):**

1. **Understand:** "Let me read it once and say what it does in one sentence."
2. **Correctness & bugs:** off-by-one, null handling, wrong condition, resource leaks, integer overflow, race conditions, swallowed exceptions.
3. **Security:** SQL concatenation (injection), hardcoded secrets, logging PII, missing input validation, unsafe deserialization, buffer overflows (C).
4. **Clean code:** naming, function length, duplication, magic numbers, dead code, comments that lie, deep nesting.
5. **Design / SOLID:** who does what, what depends on what (table below).
6. **Testing & operations:** is it testable, errors handled and logged (without PII), timeouts, retries, config externalised.

Finish with: **severity ranking** (blocker / major / minor / nit) and **how you'd fix the top 2**. Reviewers like prioritisation more than a long list.

**SOLID in one table (with the smell that reveals a violation):**

| Principle | Meaning | Smell in code | Fix |
|---|---|---|---|
| **S**ingle Responsibility | One reason to change per class/function | A `CustomerService` that validates, queries the DB, sends email and formats JSON | Split into validator, repository, notifier, mapper |
| **O**pen/Closed | Extend without editing old code | Long `if/else` or `switch` on type that grows with every new case | Polymorphism/strategy/map of handlers |
| **L**iskov Substitution | Subtypes work wherever the base type does | Subclass throws `NotSupportedException` or ignores a base contract | Re-model hierarchy, prefer composition |
| **I**nterface Segregation | Small focused interfaces | Interface with 15 methods; implementers leave many empty | Split into role interfaces |
| **D**ependency Inversion | Depend on abstractions, not concrete classes | `new SqlRepository()` inside business class; can't unit test | Inject an interface via constructor |

**Mule-flavoured analogy (impressive to mention):** SRP = one flow does one job; a flow that validates, calls 3 systems and builds the response should be split with Flow References. DIP = URLs and credentials come from properties/config, not hardcoded in the flow.

**Clean-code smells checklist:**
- Names that don't say intent (`d`, `data2`, `doStuff`)
- Functions > ~20 lines or > 3 parameters
- Boolean flag parameters
- Duplicated blocks
- Catch-all `catch (Exception e) {}` (swallowing) or `printStackTrace`
- Returning `null` instead of empty collection/Optional
- Magic numbers/strings (`if (status == 3)`)
- Mixed abstraction levels (HTTP parsing + business rules in one method)
- Commented-out code
- Global/mutable static state
- In C: unchecked `malloc`, missing `free` (leak), `strcpy` (overflow), no bounds checks, uninitialised variables, returning pointer to local variable
- In C#: not disposing (`using`), `async void`, `.Result`/`.Wait()` deadlocks, `catch (Exception)`, string concatenation in loops
- In Java: unclosed streams (use try-with-resources), `==` on Strings, mutable public fields, `Optional.get()` without check, checked-exception swallowing, `SimpleDateFormat` shared across threads

**Practice example (try it, then compare with the notes below):**

```java
public class CustomerService {
    public static Connection conn = DriverManager.getConnection("jdbc:mysql://prod/db", "root", "admin123");

    public String process(String type, String id, String email, int flag) {
        try {
            if (type == "GET") {
                Statement s = conn.createStatement();
                ResultSet r = s.executeQuery("select * from customers where id = '" + id + "'");
                System.out.println("Customer email: " + email);
                if (flag == 1) { return r.getString("name"); }
                else { return null; }
            } else if (type == "CREATE") {
                // TODO
            }
        } catch (Exception e) { }
        return "";
    }
}
```
Notes: hardcoded prod credentials (security) | `static` shared connection, never closed (leak/thread-safety) | `==` for strings (bug) | SQL injection | logs PII (email) | swallowed exception | returns `null` and `""` inconsistently | magic `flag` | string "type" switch violates **OCP** | does DB + routing + logging = violates **SRP** | concrete `Connection` creation = violates **DIP** | `r.next()` never called (bug) | unused `else if` branch, `select *`.

**Practice plan:** do 3-4 reviews like this (ask Claude to generate a snippet in Java, C and C# and mark your answer). Time-box each to 8 minutes. Practise saying findings in order of severity.

### 12.2 "Open your current project and add a feature"

This is the most likely task. Use this playbook every time (say each step aloud):

1. **Clarify** the requirement in one sentence and ask 1-2 questions (input, output, error cases). *"So the new endpoint returns X, what should it do when Y?"*
2. **Locate** where the change goes. Use your project map (section 4). Know which XML file holds which flow so you don't search for 2 minutes.
3. **Plan out loud:** "I'll add a flow, a DataWeave transform, a property, and one error mapping."
4. **Smallest working slice first:** hardcode a response, run it, see 200. Then add real logic. Never write 10 steps and run once.
5. **Config, not constants:** new URLs/timeouts go to the properties files.
6. **Reuse:** use the existing global error handler and logging sub-flow; don't copy and paste.
7. **Test** with curl/Postman, including one failure case.
8. **Recap:** "I changed these 3 files. Trade-off was X."

**Typical change requests and where they touch:**

| Request | What you change |
|---|---|
| Add `PUT /customers/{id}` | Spec (RAML/OAS) -> new APIkit flow -> validation -> store update -> DW response -> 404 case |
| Add new field to a payload | Spec example/schema, validation, DW mapping(s), store, sample JSON files, README |
| Add a query filter `?status=` | `attributes.queryParams.status` -> DW `filter` -> default when absent |
| Call a second backend | New HTTP Request config + properties; new error mappings; correlation ID header |
| Change error format | Only the error-handler DW script (that's why it's centralised) |
| Add pagination | `limit`/`offset` query params, `[offset to offset+limit-1]` DW slice, `total` in response |
| Mask sensitive data | DW function used in logs/responses |
| Add new env | New `config-<env>.yaml`, run with `-Denv` |

**If you freeze:** say what you're thinking. "I'm checking whether the variable still exists after the connector." Silence is worse than a wrong guess.

### 12.3 "Build a new Mule project from scratch" (screen share)

They may give a 30-60 minute brief you've never seen, e.g.:
- Accounts/Orders/Products API with 2-3 endpoints (same patterns, new domain)
- "Transfer money" endpoint: validate amount > 0, source != target, call a balance service, return 200/409/422
- CSV/JSON file -> transform -> HTTP POST to an API
- Scheduler every minute -> call API -> log summary
- Aggregate two API calls (Scatter-Gather) -> merged JSON
- Read from a database (H2/MySQL) -> JSON (you might not have a DB; know the **Database connector** basics)

**Your reusable skeleton (memorise this order, it works for any new API):**

1. New Mule project (File > New > Mule Project), name it clearly, pick the runtime.
2. `global-config.xml`: **Configuration properties** (`config-${env}.yaml`), **HTTP Listener config** (port from property).
3. Create `config-dev.yaml` with port/URLs/timeouts.
4. Main flow: **Listener** (`/api/...`) -> **Logger** (correlationId, no PII) -> **Set Variable** (save inputs) -> logic -> **Transform Message** -> response.
5. **Validation** early (fail fast, 400).
6. Global **error handler**: HTTP/timeout/validation/ANY -> standard JSON with `correlationId`.
7. Run, test with Postman, commit.

Keep a **personal cheat-sheet** (1 page, from this skeleton + DW snippets in section 2.11). Re-type the skeleton from memory 3 times this week with different domains (practise: Orders API, then Accounts API, then Products API). Third time should take ~20 min.

**Studio fluency (small things that save minutes in a live demo):** know how to create a flow, drag components, edit the XML tab, add a module from Exchange (Mule Palette > Add Modules), open the DataWeave preview, set VM arguments, start the debugger, view the console, and clean/restart a stuck app (stop all, delete `.mule`, restart). Practise screen sharing with Studio open once, text size big enough to read.

**If they give you something totally unfamiliar** (e.g., Salesforce connector, JMS): say so, then show your method: open docs.mulesoft.com for that connector, find the "Quick start" config, copy the pattern, test with the smallest example. They want to see **how you learn**, not that you know everything.

### 12.4 Talking while coding (communication score)

- Narrate intent, not keystrokes: "I'm validating first so bad input never reaches the backend."
- Name trade-offs: "I could do X (simpler) or Y (more robust). I'll pick X now because Z."
- When something fails: read the error out loud, form one hypothesis, test it. Do not randomly change things.
- It is fine to say "I don't remember the exact syntax, I'd check the docs." Then look it up quickly. That is normal engineering.
- Ask clarifying questions before building. It's rated positively.

---

## 13. Decision log (pros / cons / what to say)

For each decision: **what I chose, the alternatives, honest pros/cons, and the sentence I'd say in the interview.** Confidence comes from knowing the cons of your own choice. Edit the "Chosen" column if you decide differently, and make sure you believe the reason.

### D1. Contract-first with APIkit vs plain HTTP Listener + Choice

| | APIkit (spec-first) | Plain Listener + Choice router |
|---|---|---|
| Pros | Industry standard; spec is documentation and contract; auto 404/405/400 for bad routes/methods/bodies; one flow per operation; easy to add endpoints | Fewer moving parts; full control; quicker to build; easier to explain every line |
| Cons | More generated "magic" to understand; spec errors confuse beginners; extra error types (`APIKIT:*`) to map | You re-implement routing/validation; doesn't scale; no contract file; looks less professional |
| **Chosen** | **APIkit** *(fallback: plain, if APIkit blocks you by Day 3)* | |

*Say:* "I chose contract-first so the spec is the single source of truth and APIkit handles routing and basic validation. The trade-off is more generated structure, so I made sure I understand the router and its error types."

### D2. Spec language: RAML vs OpenAPI (OAS 3)

| | RAML | OAS 3 |
|---|---|---|
| Pros | MuleSoft-native, concise, great Studio/Design Center tooling | Industry-wide standard; also used outside MuleSoft; tooling everywhere |
| Cons | Less common outside MuleSoft | Verbose; some Mule tooling supports it a bit less smoothly |
| **Chosen** | **Whichever you can write confidently in Day 3.** Use OAS if you want a transferable skill, RAML if you want fewer typing errors | |

*Say:* "Either is valid; I picked X because ... and I can convert to the other."

### D3. Storage for customers: Object Store vs static JSON vs variable

| | Object Store | Static JSON file | In-memory var |
|---|---|---|---|
| Pros | Real Mule feature; POST then GET works; shows platform knowledge | Simplest; zero config; deterministic | Zero setup |
| Cons | Extra module/config to learn; resets on restart (in-memory) | Read-only; POST can't persist | Lost after each request, POST useless |
| **Chosen** | **Object Store seeded from JSON** | | |

*Say:* "The brief says a DB isn't required. I used an Object Store so POST then GET is consistent while the app runs, seeded from a JSON file. It resets on restart; in production this would be the System API backed by a real DB."

### D4. Which endpoints use the backend

| | Only `GET /{id}` calls backend (as specified) | All endpoints call backend |
|---|---|---|
| Pros | Matches the brief exactly; less to build; clear scope | Architecturally consistent; one source of truth |
| Cons | Data split between local store and backend (inconsistency risk) | Mock backend becomes larger; more work; beyond the brief |
| **Chosen** | **Only `GET /{id}`**, document the limitation | |

*Say:* "I followed the brief. I know the data split is not ideal in production and I'd move everything behind the System API."

### D5. Mock backend: Node script vs WireMock vs Mockoon vs public API

| | Node script | WireMock | Mockoon | Public mock API |
|---|---|---|---|---|
| Pros | Full control, simulate delay/5xx/outage easily, you know Node | Powerful, standard in testing | GUI, very fast | No setup |
| Cons | Extra runtime for graders (Node) | Needs Java jar + mapping files | GUI-based config is harder to version in Git | Can't force 5xx/timeouts; response shape doesn't match; may go offline |
| **Chosen** | **Node script in `mock-backend/`** | | | |

*Say:* "I wanted to force each failure mode on demand, so I wrote a tiny mock with special IDs. It's in the repo with a one-command start."

### D6. Property format: YAML vs `.properties`

| | YAML | `.properties` |
|---|---|---|
| Pros | Hierarchical, readable, groups settings (`backend.host`) | Simple, universally understood, works everywhere in Mule |
| Cons | Indentation errors; secure-properties tooling slightly less common | Flat, repetitive keys |
| **Chosen** | **YAML** *(switch to `.properties` if indentation bites you; both are fine)* | |

### D7. Validation approach: APIkit/spec only vs Validation module vs DataWeave

| | APIkit schema validation | Validation module | DataWeave custom |
|---|---|---|---|
| Pros | Free; contract-driven | Declarative, readable, many ready checks (`is-email`, `is-not-blank-string`) | Collect **all** errors into `details[]` exactly like `error-400.json`; full control |
| Cons | Error messages generic; hard to match required `details` shape | Throws on the **first** failure by default | More code; you maintain the rules |
| **Chosen** | **Combine: spec for structure, DataWeave for collecting all field errors, then raise one validation error** | | |

*Say:* "The sample 400 response lists several failed fields, so failing on the first error would not match the contract. I validate all fields in one DataWeave step and raise one controlled error with `details`."

### D8. On Error Propagate vs On Error Continue

| | Propagate | Continue |
|---|---|---|
| Pros | Error stays an error (flow marked failed); transactions roll back; honest semantics | Simple: the handler's payload is the response |
| Cons | Need to set the response/status via the listener error response | Flow looks "successful" internally (metrics/monitoring hide failures); easy to return 200 by mistake |
| **Chosen** | **On Error Propagate** with `httpStatus` variable consumed by the listener's error response | |

*Say:* "Propagate keeps failures visible to monitoring and keeps the semantics correct; Continue would make the flow look successful."

### D9. Central (global) error handler vs per-flow handlers

| | Global shared handler | Per-flow handlers |
|---|---|---|
| Pros | One error format; DRY; changing format = one file; consistent logging | Fine-grained, flow-specific behaviour |
| Cons | Needs a variable convention; flow-specific cases still need local handlers | Duplication; inconsistent responses; harder maintenance |
| **Chosen** | **Global handler + a small local Try/handler only where needed** | |

### D10. Mapping downstream errors to statuses

| Case | Choice | Alternative | Why |
|---|---|---|---|
| Backend 404 | **404** `CUSTOMER_NOT_FOUND` | 502 | The caller asked for a missing resource; legit not-found |
| Backend other 4xx | **502** `DOWNSTREAM_SERVICE_ERROR` | pass the same 4xx through | Our API built the downstream request, so a 4xx there is our integration bug, not the caller's fault. Passing it through leaks internals |
| Backend 5xx | **502** | 500 / 503 | Gateway semantics: upstream failed |
| Timeout | **504** | 502 / 408 | 504 = gateway timeout; 408 is about the *client* being slow |
| Connectivity refused | **502** (alt 503) | 503 | 503 implies "temporarily unavailable, retry later"; 502 is the safe general choice. Be ready to defend either |
| Unknown | **500** `INTERNAL_ERROR` | - | Never leak stack traces |

### D11. Validation failure status: 400 vs 422

| | 400 | 422 |
|---|---|---|
| Pros | Matches the brief and the sample files; widely used | More precise for "well-formed but semantically invalid" |
| Cons | Less precise | Not in the brief; deviates from sample |
| **Chosen** | **400** | |

### D12. Correlation ID source

| | Always generate | Accept `X-Correlation-ID`, else generate (chosen) |
|---|---|---|
| Pros | Simple; trustworthy | Traceable across systems/callers; standard practice |
| Cons | Can't trace across callers | Trusting caller input: sanitise/length-limit it (log injection) |
| **Chosen** | **Accept if present and valid, else Mule's `correlationId`**; send to backend; return in header + body | |

### D13. Logging approach

| | Log whole payloads (debug-friendly) | Log metadata only (chosen) |
|---|---|---|
| Pros | Easy debugging | No PII leak; compliant; smaller logs |
| Cons | Leaks emails/phones/DOB; breaks the brief | Harder debugging |
| **Chosen** | **Metadata only:** correlationId, method, path, customerId, status, duration, failed field **names**. Detailed payload logging is DEBUG-only and still masked | |

*Say:* "A bank/payments company will care most about this. Customer ID is logged because it's an internal identifier; email/phone/DOB never."

### D14. Timeouts and retries

| | Chosen | Alternative | Reasoning |
|---|---|---|---|
| Response timeout | **3000 ms** from properties | 30s default | A customer lookup must be fast; failing fast protects the caller. Configurable per env |
| Connection timeout | **2000 ms** | default | Detect dead hosts quickly |
| Retries | **None for now** (mention small retry for GET only) | Auto-retry | Retrying hides failures and can multiply load; POST retries risk duplicates. A circuit breaker would be the production answer |

### D15. Where to store `customerId`: variable vs re-read later

| | `vars.customerId` before the call (chosen) | Re-read `attributes.uriParams` after the call |
|---|---|---|
| Pros | Safe: `attributes` is replaced by the connector | None |
| Cons | One extra Set Variable | Breaks: attributes then belong to the backend response |

### D16. Customer ID generation

| | Sequential counter `CUST-10001...` (chosen) | UUID |
|---|---|---|
| Pros | Matches the sample (`CUST-10001`); human friendly | No collisions across instances; no shared counter |
| Cons | Needs shared state, can collide across multiple runtime nodes, guessable | Doesn't match the sample format |
| **Chosen** | **Counter in Object Store** for the demo; mention UUID/DB sequence for production | |

### D17. Project/file organisation

| | Split by responsibility (chosen) | One big XML |
|---|---|---|
| Pros | Easy to find things in a live session; reusable pieces; cleaner Git diffs | Fewer files |
| Cons | More files to navigate | Unreadable at scale; merge conflicts |

### D18. Extras: MUnit tests and Postman collection

| | Do | Skip |
|---|---|---|
| Pros | Shows engineering maturity; protects your own refactors | Saves time |
| Cons | Learning curve + time | Looks less complete |
| **Chosen** | **Add 2-3 MUnit tests and a Postman collection only after everything required works** | |

### D19. Open product decisions (write the answer in the README)

| Question | Proposal | Cons / risk |
|---|---|---|
| Required POST fields | `firstName`, `lastName`, `email`, `mobileNumber` | If they expect `dateOfBirth` required, you can switch in minutes. Document it |
| Email format check | Simple regex | Won't catch all invalid addresses; fine for the brief |
| Mobile format | Starts with `+`, digits only (E.164-style) | Too strict for local formats |
| Duplicate email | Not checked (out of scope) | Mention as a future 409 Conflict |
| Optional vs unknown fields | Ignore unknown fields | Strict mode would reject them (400) |

### Confidence rule for decisions

For every decision be ready to answer, in this order: **(1) What did you choose? (2) What else could you have done? (3) Why did you choose this? (4) What's the downside? (5) What would you change in production?** If you can answer all five for D1-D19, you'll sound senior regardless of MuleSoft experience.

---

### How we'll work together
- You build one step at a time (section 6). When blocked, paste: the **error from the console**, the **XML of the flow**, and what you expected.
- I will explain concepts, review your XML/DataWeave, and help debug - **I won't commit or push to GitHub; you'll do that manually.**
- Before the live session we'll do a mock interview using sections 8 and 9.
