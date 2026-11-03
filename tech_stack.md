# Cohesion tech stack




Built on Google Cloud. Use the pay-per-use services (Cloud Run, Firebase Hosting, Pub/Sub, Cloud Tasks) by default, and **delay the services with a fixed monthly cost** (Memorystore, Cloud SQL high availability, the load balancer for Cloud CDN, always-on Cloud Run instances) until they're needed.

---

## Coding environment
- **VS Code** - code editor 
- **GitHub** – code, reviews, and **GitHub Actions** for testing and deploying (generous free minutes).
- **Terraform** – the Google Cloud setup written as code, so Staging and Production match.
- **Staging environment** – a scaled-down copy of Production (no high availability, no always-on instances) to keep its cost low.

## AI coding agents
- **Antigravity, Claude, ChatGPT**

**Review and testing rules**
- The agents assist; **you make the final decision**.
- **Read every line** the AI writes before it's merged.
- Ask the AI to **list scenarios and bugs** to test (no access, emergency report, expired token, duplicate report, slow NEC).
- **Nothing ships until those tests pass** – automated tests run in GitHub Actions on every change.
- **Test tools:** pytest (API), Vitest (web), Playwright (web end-to-end), Maestro (mobile end-to-end).

## Design
- **Figma**

## Web app (PM dashboard, ops console, contractor platform)
- **React + TypeScript**, built with **Vite**.
- **TanStack Query** for fast, cached data; **Tailwind CSS** for styling.
- **Firebase Hosting** – global CDN, almost free at our scale, very fast page loads.

## Mobile apps (tenant, operative, contractor)
- **React Native + Expo**, sharing TypeScript code with the web app.
- **Expo EAS** to build and publish to the app stores, and **EAS Update** for over-the-air fixes.
- **Expo Push Service** for push notifications (free). It sends through Apple Push Notification service (APNs) on iOS and Firebase Cloud Messaging (FCM) on Android; both need setting up.

## Shared API types
The backend is Python, so types can't be shared directly. FastAPI produces an **OpenAPI spec**; generate TypeScript types and clients from it with **openapi-typescript** or **orval** in CI, so the web and mobile apps always match the API.

## Backend
| Job | Tool | Why |

| API | **Python (FastAPI) on Cloud Run** | Fast, the same language as the AI and data work; pay per use |

| Speed | **Cloud Run with 1 minimum instance** in Production | Avoids cold starts for normal traffic. It's billed around the clock at a reduced idle rate, so it's a fixed monthly cost. Extra instances added under heavy load can still start cold |

| Transfers to case systems | **Cloud Run workers + Cloud Tasks** | Per-transfer retries, backoff and rate limits per customer |
| Events (report received, case updated) 
| **Pub/Sub** | Lets one event trigger many things (AI check, transfer, notifications) |

| Timed jobs (deadlines, reminders, retention deletes) | **Cloud Scheduler** triggering **Cloud Run jobs** | Pennies a month |


| Live dashboards | **Server-sent events from Cloud Run**, fanned out with **Postgres LISTEN/NOTIFY** | No separate real-time service. Open connections keep instances running (and billed), requests close after at most 60 minutes so clients must reconnect, and LISTEN/NOTIFY makes sure every instance gets each update |


| Sign-in | **Google Identity Platform** | Microsoft and Google SSO, passwordless email links, two-step sign-in (TOTP, or SMS at a per-message charge); much cheaper than Auth0. SAML/OIDC sign-ins cost more per active user than basic sign-in |


| Secrets and keys | **Secret Manager + Cloud KMS** | For case-system tokens and API keys |


| UK addresses | **Ideal Postcodes or OS Places** | Cheaper and more accurate for UK property matching than Google Maps |


| Email and SMS | **Postmark** (email), **Twilio** (SMS) | Reliable, pay as you go |


| Errors and monitoring | **Cloud Logging, Monitoring, Trace** + **Sentry** | Sentry catches app and web crashes. The free tier allows one user and a small monthly error limit, so a team will need the paid plan |

**Duplicate messages:** Pub/Sub and Cloud Tasks can deliver the same message more than once. Every worker must use an **idempotency key** (for example, the report ID plus the action) so a repeated message never creates a duplicate report, transfer or notification.

## Data stack
| Job | Tool | Cost-effective approach |

| 1. Main app data | **Cloud SQL for PostgreSQL** (London) | Start small; turn on high availability for Production only. Use **connection pooling** – a big speed win. Cloud SQL's managed pooling may require the more expensive Enterprise Plus edition (check); otherwise run **PgBouncer** |

| 2. Schema changes | **Alembic** migrations, run in GitHub Actions before each deploy | Database changes are reviewed and repeatable |

| 3. Backups and recovery | **Cloud SQL automated backups + point-in-time recovery** | Test a restore regularly; keep Cloud Storage versioning on for documents |

| 4. Reporting, analytics, ML data | **BigQuery** | **Partition tables by date** so queries scan less (you pay per data scanned) |

| 5. Photos and documents | **Cloud Storage** (private bucket), served with **signed URLs** | Resize photos on upload. Add **Cloud CDN** only when traffic justifies it – it needs an external Application Load Balancer, which costs roughly $18+ a month even when idle, and must use signed URLs or cookies |

| 6. Retention deletes | **Scheduled Cloud Run job** + **lifecycle rules** | Lifecycle rules only work on file age; retention that starts from an event (such as a case closing) needs the scheduled job |

| 7. Cache | **Start with Cloud Run memory** for short-lived caches; add **Memorystore (Redis)** when needed | Memorystore charges every hour even when idle |

| 8. Sessions and rate limits | **Postgres** until Memorystore is added | Cloud Run memory is per instance and temporary, so it can't hold sessions or shared rate-limit counts |

| 9. Postgres → BigQuery | **Start with scheduled exports or BigQuery federated queries**; switch to **Datastream** when near real-time is needed | Datastream charges per GB processed |

| 10. Search (refs, addresses, tenants) | **Postgres full-text + pg_trgm** | Free, fast for our volumes – no search service needed |

| 11. AI similarity (duplicates) | **pgvector** in Postgres | No separate vector database to pay for |

| 12. Feature flags | **A table in Postgres** (the Settings screen) | Avoids LaunchDarkly costs |

## Data protection (UK GDPR)
- Tenant details and photos from inside homes are **personal data**. Keep storage buckets private and serve files only through short-lived signed URLs.
- Store data in **London (europe-west2)**. Record any service that processes data outside the UK (for example, an AI model not offered in London) and the legal basis for it.
- Set and document **retention periods** for each type of data, enforced by the retention jobs above.
- Never log full requests or personal data.

## Cohesion AI agents
**Don't start by fine-tuning.** It's expensive, slow to change, and usually unnecessary. Do this instead:
| Need | Tool | Why |

| Agents (triage, briefings, Cohesion Assistant) | **Gemini Flash on Vertex AI** for speed and cost; **Gemini Pro** or **Claude on Vertex AI** for harder reasoning | Flash is fast and cheap for most agent work. Check which regions each model is offered in – not all are available in europe-west2 |

| Property management, damp and mould knowledge | **Retrieval (RAG)** – guidance, Awaab's Law, repair codes and past cases stored with pgvector | Update knowledge by adding documents, not retraining |

| Photo hazard and severity checks | **Start with Gemini** checking photos, with staff confirming the result; **train a small image model** (Vertex AI) once there are enough labelled photos | A trained model is fast, cheap per photo and testable, but needs labelled data first. Check that Vertex AI AutoML for images is still offered before planning on it | Build computer vision model from the ground up 

| Scheduling (appointments, routes) | **Google OR-Tools** (free) | Scheduling is maths, not language – an LLM is the wrong tool | Scheduling algorithms from the ground up 

| Data modelling help | Prompts plus the schema as context | No training needed | Fine tune on specific data modelling concepts

| Quality | A **nightly labelled test set** and **guardrails** (from the AI monitoring screens) | Catches problems before customers see them |

Fine-tune only later, once there are thousands of labelled examples and a test set proving it beats prompting with retrieval.

---

## How to make Cohesion very fast
1. **Everything in London (europe-west2) where available** – app, database, storage, and AI models that are offered there.
2. **One minimum Cloud Run instance** for the API (see Backend).
3. **Connection pooling** to Postgres, plus **indexes** on what you search (refs, postcodes, status, dates).
4. **Tenant reports return instantly.** Save first, then do the AI check, transfer and notifications in the background with Pub/Sub and Cloud Tasks.
5. **Cache dashboard numbers** for a minute instead of recalculating on every page load.
6. **CDN for the web app** (Firebase Hosting); **resize photos** so phones upload and download less.
7. **Targets to watch:** API p95 under 300 ms, report submit under 1 second, emergency transfer under 15 minutes.

## Where the money goes (watch these)
1. **Fixed monthly costs** – Cloud SQL high availability (Production only), the minimum Cloud Run instance, and the load balancer if Cloud CDN is added.
2. **Memorystore** – charges even when idle, so add it later.
3. **Datastream** – charges per GB processed; use scheduled exports until near real-time is needed.
4. **AI tokens** – use Flash by default, keep prompts short, cache repeated context.
5. **BigQuery scans** – partition tables and avoid `SELECT *`.
6. **Log volume** – don't log full requests; it's also a privacy risk.

## Dashboard stack
**Decision: build our own dashboard templates using libraries – don't write charts from scratch, and don't embed a BI tool for the core dashboards.**

| Option 

| Build in-app with libraries | **Yes.** Fast, matches the Cohesion design, no per-user licence, full control over speed and data access |



**How it fits together**
- **One set of metric definitions** (open cases, overdue repairs, average repair time, SLA breaches) defined once in the backend. Dashboards, reports and automations all use the same definitions, so numbers always agree.

- **Templates are JSON** stored in Postgres: which widgets, which metrics, which filters, and the layout. Cohesion ships ready-made templates per role (PM, ops, contractor); customers can copy one and customise it.

- **A widget is a saved report** (see Reports below), so anything a customer can report on, they can put on a dashboard.

| Job | Tool | Why |
TODO: Research on charts templates 

| Charts | **Apache ECharts** (via `echarts-for-react`) | Draws on canvas, so it stays smooth with thousands of points; covers every chart type we need |


| Movable, resizable layout | **react-grid-layout** | Drag-and-drop dashboard editing without building it ourselves |

| Tables | **TanStack Table** with server-side paging, sorting and filtering | Only one page of rows is sent to the browser |

| Live operational numbers | **Postgres summary tables** refreshed every minute by a scheduled job | Dashboards read small pre-calculated tables instead of counting every case on each load |

| Live updates | **Server-sent events** (see Backend) | Counters and case lists update without a page refresh |

| Trends and history | **BigQuery reporting tables**, served through the API with caching | Big historical queries don't slow the main database. The browser never queries BigQuery directly |

| Export and sharing | **CSV, PDF** (server-side rendering with Playwright) and scheduled email via **Postmark** | Customers hand reports straight to boards and regulators |

| Internal Cohesion dashboards | **Looker Studio** (free) on BigQuery | No cost; enough for company KPIs and AI monitoring |

**Rules**
- Every dashboard query must use an index or a summary table. Test with production-sized data before release.

- Every customer only ever sees their own data. Enforce it in the API and with **Postgres row-level security** as a second safety net.

- Every widget shows **"Updated at <time>"** so users know how fresh the numbers are.

## Data migration stack
Customers bring their existing data into Cohesion: properties, tenants, open cases, case history, contractors and documents. It usually arrives as spreadsheets or exports from another case system, and it's usually messy. Smaller customers should be able to **do it themselves in the web app**; larger ones get an **assisted migration** by the Cohesion team using the same pipeline.

**Self-service import (in the web app)**
| Step | Tool | Why |

| 1. Upload | CSV or Excel upload to a **private Cloud Storage bucket per customer** through signed URLs | Secure; the original file is kept for auditing |

| 2. Map columns | Import screen built with **react-spreadsheet-import** (open source); **Gemini Flash suggests the mapping** from column names and sample rows | Customers confirm a suggested mapping instead of matching dozens of columns by hand. Avoids paid importers like Flatfile or OneSchema |

| 3. Check and fix | **Pandera** checks in the backend; errors shown row by row in the import screen so the customer can fix them there | Bad dates, missing fields and wrong types are caught before anything is loaded |

| 4. Match addresses | **Ideal Postcodes or OS Places** to find each property's **UPRN** | One reliable ID per property; stops duplicate properties |

| 5. Find duplicates | **pg_trgm** and **pgvector** similarity | Shows likely duplicates for the customer to merge or keep |

| 6. Preview | A summary before import: "1,240 properties, 3,100 tenants, 18 possible duplicates, 4 errors" | No surprises |

| 7. Load | **Python + Polars** in a **Cloud Run job**, into **staging tables** first, then into live tables in one transaction | Nothing half-loaded reaches the live system |

| 8. Undo | Every import has a **batch ID**; one click rolls the whole batch back | Customers can fix a mistake without support |

| 9. Report | A **reconciliation report**: counts and totals, source vs Cohesion | Customer signs off before go-live |

**Assisted migration and ongoing sync**
- **Mapping templates** (YAML in GitHub), one per known source system, so every new customer on that system reuses the mapping.
- **Connectors** for case systems with an API, run by the **transfer workers** (Cloud Tasks); scheduled file loads for systems without one.
- **SFTP** only for customers who insist.

**Rules**
- Always offer a **dry run** first; keep original files until the customer signs off.
- Every load is **repeatable**: re-running updates records instead of duplicating them (match on the source ID or UPRN).
- Migration files contain personal data: delete them once the agreed retention period ends.
- **Schema changes** to our own database stay in **Alembic** (see Data stack) and are separate from customer imports.

## Data engineering, data analysis and machine learning stack
Two groups use this:
- **Customers**, through the web app: building and testing their own pipelines, reports, automations and predictions **without writing code**.

- **The Cohesion team**: building the core pipelines, analysis and models behind the product.

Both run on the same foundation, so anything the Cohesion team builds can be offered to customers later.

### Foundation (shared)
| Job | Tool | Why |
|---|---|---|
| Move data into BigQuery | **Scheduled exports or federated queries** from Postgres; **Datastream** later. Events from **Pub/Sub** go straight in with a **BigQuery subscription** | No pipeline code for the basic loads |

| Layers in BigQuery | **raw** → **clean** → **reporting** datasets | Clear ownership; reporting tables feed dashboards, reports and models |

| Keeping customers apart | **One BigQuery dataset per customer** for their reporting data and anything they build | Simple, strong isolation; easy to delete a customer's data on exit |

| Core transformations | **dbt Core** run as a **Cloud Run job** | SQL-based, version-controlled in GitHub, tested in CI |
| Scheduling | **Cloud Scheduler + Cloud Workflows** | Pennies a month. Avoid **Cloud Composer** (Airflow) until there are many dependent pipelines – it costs hundreds of pounds a month even when idle |

| Data quality | **dbt tests** (not null, unique, accepted values, freshness) | A failed test stops the pipeline before bad numbers reach a customer |

| Personal data in analytics | **Pseudonymise** tenant names and contact details in the clean layer; **BigQuery policy tags** on any column that must stay | Analysis and models rarely need to know who a tenant is |

| Cost control | Partitioned and clustered tables; a **maximum bytes billed** limit on every query; a **monthly query budget per customer** | One customer's big query can't run up a large bill |

### Customer data studio (visual, no code)
Customers work on a **visual canvas** in the web app. They drag **blocks (nodes)** onto it, connect them to build a **dataflow**, and run it. From the results they build **reports from templates**, add **ready-made machine learning predictions**, and **look up and validate data** the way they would in Excel. An **AI assistant** helps at every step. Nobody writes code.

**How a customer uses it**
1. **Build the dataflow** – drag in a source (for example, Repairs), connect a Filter, a Lookup against their survey spreadsheet, then a Group By.
2. **Check it** – click any block to see a live preview of its data; problem rows are highlighted in red.
3. **Run it** – the canvas shows each block's progress (waiting, running, done, failed) and the estimated cost before it starts.
4. **Use the results** – send the output to a report template, a prediction block, an automation or an export.
5. **Ask the assistant** at any point: "join this to the stock survey and flag properties with 2+ damp reports this year."

#### Blocks available on the canvas
| Group | Blocks |
|---|---|
| **Sources** | Cohesion data (cases, properties, tenants, repairs, contractors), uploaded CSV or Excel files, connected systems (see Data connectors) |
| **Shape** | Filter, choose columns, rename, sort, remove duplicates, combine (union), join, group and summarise, pivot, split column |
| **Calculate** | New column using **Excel-style formulas** (`IF`, `DATEDIF`, `CONCAT`, `ROUND`…) or a point-and-click formula builder |
| **Lookup and validate** | **Lookup** (like Excel `XLOOKUP` / `VLOOKUP`) against any table or uploaded reference list, with exact or close matching; **Validate** against rules (required, allowed values, date ranges, format of postcodes, phone numbers and emails); **Address lookup** to find the UPRN; **Check against list** (for example, a contractor's approved trades) |
| **Predict** | Any ready-made model from the model catalogue (see below) |
| **Output** | Save as a table, send to a report, start an automation, export to CSV or Excel, send to another system |

#### Tech stack
| Part | Tool | Why |
|---|---|---|
| **Visual canvas** | **React Flow** (`@xyflow/react`) | The standard library for node-based editors: drag, connect, zoom, minimap, custom blocks. Open source (MIT) |

| Automatic tidy layout | **ELK.js** | One click arranges a messy flow neatly |

| Canvas state and undo/redo | **Zustand** + **zundo** | Small, fast state store with undo/redo built in |

| Block settings panels | **shadcn/ui** components with **React Hook Form** + **Zod** | Consistent forms; Zod checks settings before anything runs |

| **Data preview grid** (Excel-like) | **AG Grid Community** | Fast with large tables, feels like a spreadsheet: sort, filter, resize, highlight problem cells. Free (MIT) |

| **Excel-style formulas** | Our own small formula language, parsed with **Chevrotain** and translated to SQL in the backend | Customers use familiar functions; only functions we allow can run. Avoids HyperFormula, which needs a paid licence for commercial use |

| Saving flows | Flow saved as **JSON** in Postgres, checked with **Pydantic**, with **draft and published versions** | Every change is versioned and can be rolled back |

| **Turning a flow into a query** | **SQLGlot** (Python) builds BigQuery SQL from the flow – each block becomes one step of the query | SQL is built from a structure, never by pasting text together, so customers can't inject SQL or reach other customers' data |
| Running flows | **BigQuery** jobs in the customer's own dataset, run by **Cloud Workflows** | Handles millions of rows with no servers to manage |

| Live run status on the canvas | **Server-sent events** (see Backend) | Each block turns green, amber or red as it runs |
| Previews | The same query on a **sample of rows**, with a low bytes limit, cached for a few minutes | Previews appear in about a second and cost almost nothing |

| Cost estimate before running | BigQuery **dry run** | Shows "this run will scan 2 GB" before the customer clicks Run |

| Schedules and triggers | **Cloud Scheduler** (daily, weekly) or **Pub/Sub** (when new data arrives) | Flows keep results up to date without anyone pressing Run |

| Lookups | **Joins** in BigQuery for exact matches; **EDIT_DISTANCE** for close matches ("Hight St" → "High St") | Works on any number of rows, unlike spreadsheet lookups |

| Address and external lookups | **BigQuery remote functions** calling a Cloud Run service that uses **Ideal Postcodes or OS Places**, with results cached | UPRN lookups inside a flow, without paying twice for the same address |

| Validation results | Failed rows go to an **Issues table** with the reason; shown in red in the preview grid | Customers see exactly which rows failed and why, then fix them or send them back to the source |

| Lineage ("where did this number come from?") | Built from the saved flow | Click any report number to see the blocks and sources behind it |

#### Report templates
| Part | Tool | Why |
|---|---|---|
| Template library | Ready-made templates stored as **JSON**: Awaab's Law compliance, damp and mould, repairs performance, **Tenant Satisfaction Measures**, contractor performance, SLA breaches | Covers what housing providers report to boards and the Regulator of Social Housing |

| Building a report | Customer picks a template, connects it to a flow output or Cohesion data, and adjusts filters and date ranges. The **AI assistant matches columns** to the template | Reports in minutes, not days |

| Drawing reports | **Apache ECharts** and **TanStack Table**, the same as dashboards | Reports can be pinned straight onto a dashboard |

| Export and schedule | **PDF** (rendered with Playwright on Cloud Run), **Excel** (via **openpyxl**), scheduled email with **Postmark** | Ready to send to boards and regulators |

#### Ready-made machine learning models
Customers **don't build models**. They choose from a **model catalogue** built and tested by Cohesion, drop a **Predict** block on the canvas, and connect their data.

| Model | What it predicts | Built with |
|---|---|---|
| Damp and mould risk | Which properties are likely to report damp or mould | **BigQuery ML** (boosted trees) |

| Repair time | How long a repair is likely to take | **BigQuery ML** (boosted trees) |

| Repeat repair risk | Which repairs are likely to fail and come back | **BigQuery ML** (boosted trees) |

| Contractor no-show risk | Which appointments are likely to be missed | **BigQuery ML** (logistic regression) |

| Demand forecast | Repair and case volumes for the coming weeks | **BigQuery ML** (`ARIMA_PLUS`) |

| Property groups | Properties with similar problems, for planned work | **BigQuery ML** (k-means) |

| Unusual activity | Sudden spikes in cases or costs | **BigQuery ML** (anomaly detection) |

| Part | Tool | Why |
|---|---|---|
| Training on the customer's data | **BigQuery ML** inside the customer's own dataset, started with one click | Each customer's model learns from their own homes; data never mixes between customers |

| Reasons for each prediction | **`ML.EXPLAIN_PREDICT`** turned into plain English | "High risk: 3 damp reports in 12 months, no ventilation work recorded" |

| Quality shown to customers | A plain-English score ("right about 8 times in 10 on your past data") and a warning if there isn't enough data | Customers know how much to trust it |

| Model catalogue and versions | **Vertex AI Model Registry** for Cohesion's master versions; catalogue entries in Postgres | Cohesion can improve a model and roll it out to every customer |

#### AI assistant
| Part | Tool | Why |
|---|---|---|
| Model | **Gemini Flash on Vertex AI**; **Gemini Pro** for long or complicated flows | Fast and cheap for most requests |

| What it can do | **Tool calls** to: add, connect and set up blocks; build a whole flow from a sentence; suggest lookups and validation rules; match columns to report templates; explain errors and results in plain English; recommend a model | Customers describe what they want instead of learning every block |

| How changes are shown | The assistant's changes appear as **faded "proposed" blocks** that the customer accepts or rejects | Nothing is saved, run or sent without the customer's say-so |

| What it can see | Column names, data types, row counts and summary statistics – **not** raw tenant details | Keeps personal data out of prompts |

| Safety checks | Every flow the assistant proposes goes through the **same checks** (Zod, Pydantic, SQLGlot) as one built by hand | A wrong suggestion can't break anything |

| Chat panel | **assistant-ui** (React), streaming over server-sent events | Ready-made chat component that sits beside the canvas |
| Quality | Assistant requests and outcomes added to the **nightly labelled test set** | Catches drops in quality after prompt or model changes |

#### Rules for the data studio
- **No raw SQL or code for customers.** Every block is a fixed, tested step; that keeps data safe, costs predictable and flows easy to support.
- **Each customer only reaches their own dataset.** Enforced by the SQL builder and by BigQuery permissions.
- **Cost limits:** every run has a maximum bytes limit and counts toward the customer's monthly budget; the estimate is shown before running.
- **Roles:** viewers see results; builders edit drafts; only approvers **publish** a flow or schedule it.
- **Test before publishing:** preview on sample data, check the Issues table, and compare against the previous published version.
- **Audit log** of who changed, ran or published what, and when.
- **Automations** ("when an emergency case has no contractor after 2 hours, alert the ops lead") are built the same way: an Output block that starts an automation, checked by workers on **Pub/Sub** events.

### Cohesion team: data analysis
| Job | Tool | Why |
|---|---|---|
| Exploring data | **BigQuery Studio** (SQL and notebooks) | Built in; nothing extra to pay for |

| Deeper analysis | **Jupyter** notebooks (VS Code or Colab Enterprise) with **Polars**, **pandas** and **DuckDB** | Fast analysis of exported data |

| Shared charts | **Looker Studio** | One place for internal reporting |

### Cohesion team: machine learning
| Job | Tool | Why |
|---|---|---|
| Table-based predictions (risk, repair time, no-shows) | **BigQuery ML** | Train and run models with SQL where the data already is; the same models power customer Predictions |
| Custom models (photo hazard and severity) | **Vertex AI** training with **PyTorch** or **scikit-learn** | For models BigQuery ML can't build |
| Labelling | Staff confirmations in the app saved as labels, plus **Label Studio** (self-hosted) for bulk labelling | Every Gemini photo check that staff confirm or correct becomes training data |
| Experiments and model versions | **Vertex AI Experiments** + **Vertex AI Model Registry** | Every model is versioned and can be rolled back |
| Serving | **Batch predictions in BigQuery** where possible; a **Cloud Run** service for real-time | Vertex AI endpoints charge every hour even when idle – use them only for large models |
| LLM and agent quality | The **nightly labelled test set** as a Cloud Run job, results in BigQuery and Looker Studio | Catches quality drops after a prompt, model or data change |
| Retraining | **Vertex AI Pipelines** – only once models need regular retraining | Not needed until several models are in production |

**Rules**
- A new model replaces the old one only if it **beats it on the test set**.
- Check live models weekly for **drift** (inputs or results changing over time).
- Every prediction shown to a customer comes with a **plain-English reason** ("high risk: 3 damp reports in 12 months, no ventilation work recorded"). Predictions inform staff decisions; they never make them.
- Training data follows the same retention and GDPR rules as everything else.


# DASHBOARD TEMPLATES 
https://demo.cork-vue3.sbthemes.com/elements/buttons
https://www.tremor.so/
https://www.shadcn.io/template/bundui-shadcn-dashboard-free
https://github.com/TailAdmin/free-react-tailwind-admin-dashboard
https://21st.dev/community/components/s/dashboard


## Data connectors stack
Data connectors **bring a customer's data into Cohesion** for the data studio, dashboards and migration: databases, files, cloud drives and business systems. They're one-way and run on a schedule. (Two-way, record-by-record exchange with case systems is covered in the next section.)

Connected data lands in the customer's **raw BigQuery dataset** and appears as a **Source block** in the data studio.

**Decision: use dlt (open source) for the connectors, run as Cloud Run jobs.** Don't pay for Fivetran (charged per row synced, costly at scale), and don't self-host Airbyte until it's needed (it needs a permanently running server or Kubernetes).

| Job | Tool | Why |
|---|---|---|
| Connector engine | **dlt** (data load tool, Python, open source) run as **Cloud Run jobs** | Ready-made sources for databases, Salesforce, files and REST APIs; loads to BigQuery; handles incremental loads and schema changes. Pay only while it runs |
| Scheduling | **Cloud Scheduler** + **Cloud Workflows** | Hourly, daily or weekly syncs per connector |
| Only new and changed data | dlt **incremental loading** on an "updated at" field or ID | Each sync moves only what changed – faster and cheaper |
| New or changed columns in the source | dlt **schema evolution**, with a notification to the customer | Syncs don't break when the source system adds a column |
| Signing in to cloud systems | **OAuth** connections managed by **Nango** (open source, self-hosted on Cloud Run), tokens kept in **Secret Manager** | Customers click "Connect" and sign in – no passwords shared with us; tokens refresh automatically |
| Setting up a connector | A **"Connect a source" screen** in the web app: pick a system, sign in or enter details, choose tables, set a schedule, test the connection | No code, same look as the data studio |

**Supported sources**
| Source type | Examples | How |
|---|---|---|
| Uploaded files | CSV, Excel | Upload screen (see Data migration) |
| Cloud drives | **SharePoint / OneDrive**, **Google Drive** | **Microsoft Graph API** and **Google Drive API**; picks up new files in a chosen folder |
| SFTP | Nightly exports from older systems | **SFTPGo** on a small Compute Engine VM, saving to Cloud Storage. Add only when a customer needs it – it's a fixed monthly cost, and Cloud Run can't receive SFTP |
| Cloud databases | PostgreSQL, MySQL, SQL Server | dlt **SQL database** source over an encrypted connection with a read-only user |
| Databases on the customer's own network | NEC, Civica or other systems' SQL Server and Oracle databases | **Cohesion Connect agent**: a small Docker container the customer runs inside their network. It reads the database and sends data **out** to Cohesion over HTTPS, so the customer doesn't have to open their firewall. Avoids a Cloud VPN tunnel (a fixed monthly cost per customer) |
| Business systems | **Salesforce**, **Microsoft Dynamics 365**, HubSpot | dlt sources (Salesforce **Bulk API 2.0**; Dynamics **Dataverse Web API**) |
| Any other REST API | Contractor portals, survey tools, smart sensors (damp and humidity) | dlt **REST API source**, set up on a form (address, sign-in method, paging) – no code |
| Google data | Google Sheets, Google Analytics | **BigQuery Data Transfer Service** (free for most Google sources) |

**Rules**
- Read-only access to customer systems wherever possible.
- Every sync records **rows moved, time taken and errors**; failures show on the connector screen and alert the customer.
- Connected data follows the same **retention and GDPR rules** as everything else; disconnecting a source offers to delete its data.

## Sending and receiving information with other systems (Salesforce, NEC and others)
This is **live, two-way exchange of individual records**: a tenant report becomes a case in NEC; a repair status change in NEC updates Cohesion; a new complaint is sent to Salesforce. It must be reliable, fast (emergency transfers within 15 minutes) and traceable.

**How it works**
1. **Cohesion has its own standard data model** (case, repair, property, tenant, appointment, contractor).
2. Each external system has an **adapter** that translates between Cohesion's model and that system's model and API.
3. Each customer has a **mapping**: which fields go where, and how codes translate (priority codes, repair codes, Schedule of Rates (SOR) codes, case statuses). Set up on a mapping screen, with **AI suggesting the matches**.
4. **Sending:** a change in Cohesion creates an event → a **transfer task** per system → the adapter calls the system's API → the system's reference number is saved back on the Cohesion record.
5. **Receiving:** the system notifies Cohesion through a **webhook** where it can; otherwise Cohesion **polls** for changes on a schedule.

### Systems and how to connect
| System | Sending to it | Receiving from it | Notes |
|---|---|---|---|
| **NEC Housing** | NEC's integration APIs or web services, depending on the customer's version and licence | NEC's event notifications where offered; otherwise scheduled polling or the Cohesion Connect agent | **Confirm with NEC** which APIs each customer has; API access often needs a **partner agreement** and can take months |
| **Salesforce** | **REST API** (single records), **Bulk API 2.0** (large batches) via **simple-salesforce**; sign-in with an OAuth **connected app** | **Change Data Capture** or **Platform Events**; or a simple **Salesforce Flow** that calls our webhook | Each Salesforce org has a **daily API call limit** – batch where possible |
| **Microsoft Dynamics 365** | **Dataverse Web API** | Dataverse **webhooks** | Common with housing providers that use Microsoft |
| **Other housing systems** (Civica CX, MRI, Aareon QL, Orchard and others) | Their APIs or web services | Webhooks or polling | Same approach as NEC: confirm APIs and partner terms for each vendor |
| **Contractor and job systems** | Their APIs, or email/PDF job tickets for small contractors | Webhooks, polling, or the contractor app | |
| **Customers' own systems** | – | – | **Cohesion public API** and **outgoing webhooks** (see below) |

### Tech stack
| Job | Tool | Why |
|---|---|---|
| Adapters | One module per system in the backend, all built to the **same interface** (create, update, get status, list changes) | Adding a new system means writing one adapter; the rest is shared |
| Never losing an event | **Transactional outbox**: the change and its "to send" event are saved in the **same Postgres transaction**, then published to **Pub/Sub** | If anything crashes mid-way, the event is still there and gets sent |
| Sending, with retries | **Cloud Tasks**, one queue per system and customer, with **rate limits** matching each system's limits and **exponential backoff** | One slow or broken system never holds up the others |
| No duplicates | An **idempotency key** on every transfer; check the system for an existing record before creating one | Retries never create a second case in NEC or Salesforce |
| Failed transfers | After the final retry, the transfer goes to a **dead-letter queue** and appears in the **ops console** with the error and a **Retry** button | Nothing fails silently |
| Receiving webhooks | A **Cloud Run** endpoint that checks the **signature**, saves the message, replies at once, then processes it in the background | Fast replies, so the sending system doesn't time out and resend |
| Polling | **Cloud Scheduler** triggering **Cloud Run jobs**, asking only for changes since the last check | For systems without webhooks |
| Stopping loops | Every change is tagged with **where it came from**; changes that came from a system are never sent back to it | Prevents two systems updating each other forever |
| Conflicts | A **system of record per field** (for example, NEC owns the repair status, Cohesion owns the tenant's photos) | Clear rules for which value wins |
| Sign-in to systems | **Nango** for OAuth (Salesforce, Dynamics); API keys and certificates in **Secret Manager** | Tokens refresh automatically and are never stored in code |
| Fixed outgoing IP address | **Direct VPC egress** from Cloud Run + **Cloud NAT** with a reserved static IP | Many housing providers only accept connections from approved IP addresses. Cloud NAT is a small fixed monthly cost – add it when the first customer requires it |
| Status on every record | "Sent to NEC ✓ 10:42 – NEC ref 12345", or "Waiting to retry (3 of 8)" | Staff can see where every transfer is without asking support |
| Monitoring and alerts | **Cloud Monitoring** per system and customer: success rate, delay, queue length; an **alert if an emergency transfer isn't confirmed within 15 minutes** | Problems are caught before customers report them |
| Full history | Every request and response saved (with personal data removed) for a set period | Fast fault-finding and an audit trail |

### Cohesion's own API and webhooks
For customers and partners who want to connect their own systems.

| Job | Tool | Why |
|---|---|---|
| Public API | **REST API** with an **OpenAPI** spec, versioned (`/v1/`), with API keys or OAuth per customer, and rate limits | Standard, well documented, easy to build against |
| API documentation | Generated from the OpenAPI spec with **Scalar** or **Redoc** | Always matches the real API |
| Outgoing webhooks | Customers subscribe to events (case created, repair completed, deadline at risk); each message is **signed (HMAC)** and retried with backoff | Customers get updates instantly without polling |
| No-code connections (later) | Apps for **Microsoft Power Automate** and **Zapier** | Many housing providers already use Power Automate |

### Testing integrations
- A **test environment for every system**: Salesforce sandboxes, NEC test environments, and **mock servers** (**Prism** from an OpenAPI spec, or **WireMock**) when no sandbox exists.
- **Recorded responses** (**vcrpy** / **pytest-recording**) so adapter tests run in GitHub Actions without calling real systems.
- **Contract tests** for every adapter: create, update, status change, duplicate, expired token, rate limit hit, system down, slow response.
- A **mapping test** before go-live: send sample records end to end and have the customer confirm they look right in their system.

### Rules
- **Send only the data each system needs** (data minimisation under UK GDPR); record what's shared with whom in the customer's data sharing agreement.
- **Emergencies first:** emergency transfers use their own high-priority queue.
- **Never block the tenant:** a report is saved in Cohesion first; sending to other systems always happens in the background.
- **Sign vendor agreements early** – API access to housing systems is often the slowest part of onboarding a customer.