# Rally (Broadcom Agile Central) Agent in Microsoft Copilot Studio — Starter Kit

This starter kit gives you a clear path to build a Copilot that answers questions about **stories, planning, iterations, and team capacity** from Rally (Broadcom Agile Central). It includes:

- Reference architecture
- Step‑by‑step build guide in Copilot Studio
- Topic/intent design with sample utterances
- Power Automate actions to call Rally APIs
- A ready‑to‑paste Custom Connector (OpenAPI) skeleton
- Prompt templates (grounded, tool‑aware)
- Example queries for common asks (current sprint stories, capacity this week, etc.)
- Response formatting patterns (tabular + summaries)
- Testing plan & guardrails

> ⚠️ Notes
> - Rally exposes multiple APIs (REST WS v2.x and Lookback). The examples below target **REST WS v2.x**. Your org may require a different auth header (API Key header name, Basic auth, or ZSESSIONID). Swap the header style in the Custom Connector to match your Rally tenant policy.
> - Replace `{{RALLY_BASE}}` with your base (often `https://rally1.rallydev.com/slm/webservice/v2.0`).

---

## 1) Reference Architecture

```
User → Copilot Studio (Topics + Orchestrator)
     → Actions (Power Automate) → Custom Connector → Rally REST API
     → (optional) Dataverse cache / SharePoint for transient logs
     → (optional) Graph/Entra ID to resolve user ↔ Rally username
```

**Why this works**
- Copilot handles NLU (utterance → intent + entities).
- Power Automate encapsulates API calls and response shaping.
- The Custom Connector centralizes auth + base URL so you don’t repeat it in every flow.

---

## 2) Create the Copilot

1. **New Copilot** in Copilot Studio → Give it a meaningful name (e.g., *Rally Delivery Copilot*).
2. **Security** → enable **Generative Answers** (optional) only for your own docs; **do not** let it hallucinate Rally data. We’ll **ground** through Actions.
3. **Data** → **Add an action** → *Create a flow* (opens Power Automate). We’ll build a flow per capability (Get Current Iteration, List Stories, Capacity, etc.).
4. **Publish** early in a dev environment (Teams or Web) for quick iteration.

---

## 3) Topics (Intents) & Entities

### Core Topics
- **Get current iteration**
  - Utterances: *what’s the current sprint?*, *which iteration are we in?*, *current sprint dates?*
- **List stories in current iteration**
  - Utterances: *show user stories for this sprint*, *stories in current iteration*, *sprint backlog*
- **Query stories by filter**
  - Utterances: *stories assigned to me*, *open stories for project Alpha*, *blocked stories*
- **Capacity for this week / iteration**
  - Utterances: *how much capacity do we have this week?*, *team capacity this sprint*, *dev capacity remaining*
- **Iteration planning summary**
  - Utterances: *sprint summary*, *planned vs accepted points*, *burndown snapshot*

### Common Entities
- `project` (Rally Project name)
- `iteration` (name like *Sprint 24*)
- `user` (display name/email; map to Rally `User`)
- `state` / `scheduleState` (Defined, In‑Progress, Completed, Accepted)
- `dateRange` (e.g., *this week*, *last 7 days*)

> Tip: Add **synonyms** (e.g., *sprint* ↔ *iteration*, *story* ↔ *user story* / *hierarchical requirement*).

---

## 4) Custom Connector (OpenAPI) Skeleton

Create a **Custom Connector** in Power Platform (from OpenAPI). Paste and adapt the skeleton below.

```yaml
openapi: 3.0.1
info:
  title: Rally REST v2
  version: 1.0.0
servers:
  - url: https://rally1.rallydev.com/slm/webservice/v2.0
components:
  securitySchemes:
    apiKeyAuth:
      type: apiKey
      in: header
      name: X-RallyAPIKey   # ← change to your org’s required header (or use Basic)
security:
  - apiKeyAuth: []
paths:
  /iteration:
    get:
      summary: List iterations (filterable)
      parameters:
        - in: query
          name: fetch
          schema: { type: string }
          example: Name,StartDate,EndDate,Project
        - in: query
          name: query
          schema: { type: string }
          example: (Project.Name = "{{project}}")
        - in: query
          name: pagesize
          schema: { type: integer }
          example: 200
      responses:
        '200': { description: OK }
  /hierarchicalrequirement:
    get:
      summary: List user stories
      parameters:
        - in: query
          name: fetch
          schema: { type: string }
          example: FormattedID,Name,Owner,PlanEstimate,ScheduleState,Blocked,Iteration,Project,Tags
        - in: query
          name: query
          schema: { type: string }
          example: (Iteration.Name = "{{iteration}}")
        - in: query
          name: pagesize
          schema: { type: integer }
          example: 200
      responses:
        '200': { description: OK }
  /useriterationcapacity:
    get:
      summary: User capacity records for an iteration
      parameters:
        - in: query
          name: fetch
          schema: { type: string }
          example: Iteration,User,CapacityPerDay,Vacation
        - in: query
          name: query
          schema: { type: string }
          example: (Iteration.Name = "{{iteration}}")
      responses:
        '200': { description: OK }
```

> After import, set **Security** to use your key or Basic auth. Test with a known project/iteration.

---

## 5) Power Automate Flows (Actions)

Create one flow per capability. Each flow should:
1) Accept parameters from Copilot (project, iteration, user, etc.).
2) Call the Custom Connector endpoint(s).
3) Shape output as **clean JSON** for Copilot.

### A) Get Current Iteration for Project
**Trigger:** Copilot (Action).  
**Inputs:** `project` (string), optional `date` (default = utcNow()).

**Steps:**
1. **Compose** `today = formatDateTime(utcNow(), 'yyyy-MM-dd')`.
2. **HTTP (Custom Connector → /iteration)** with query:
   - `fetch=Name,StartDate,EndDate,Project`
   - `query=(Project.Name = "@{triggerBody()?['project']}")`
3. **Filter array** where `StartDate <= today <= EndDate`.
4. **Response to Copilot** JSON:
```json
{
  "project": "<project>",
  "currentIteration": {
    "name": "Sprint 24",
    "startDate": "2025-10-28",
    "endDate": "2025-11-10"
  }
}
```

### B) List Stories in Current Iteration
**Inputs:** `project` (string).  
**Steps:**
1. **Call** flow A to get `currentIteration.name`.
2. **Custom Connector → /hierarchicalrequirement** with:
   - `fetch=FormattedID,Name,Owner,PlanEstimate,ScheduleState,Blocked,Iteration,Project`
   - `query=(Project.Name = "{project}") AND (Iteration.Name = "{iter}")`
3. **Map** items to concise objects:
```json
{
  "iteration": "Sprint 24",
  "count": 37,
  "totalPoints": 142,
  "byState": {"Defined": 10, "In-Progress": 18, "Completed": 4, "Accepted": 5},
  "stories": [
    {"id":"US12345","title":"OAuth callback fix","owner":"A. Singh","points":3,"state":"In-Progress","blocked":false},
    {"id":"US12346","title":"Capacity endpoint","owner":"S. Ramtekkar","points":5,"state":"Defined","blocked":true}
  ]
}
```

### C) Capacity This Week (or This Iteration)
**Inputs:** `project`, optional `iteration` (default = current).  
**Steps:**
1. Resolve iteration as in A.
2. **Custom Connector → /useriterationcapacity** with:
   - `fetch=Iteration,User,CapacityPerDay,Vacation`
   - `query=(Iteration.Name = "{iter}")`
3. Calculate:
   - **Team capacity (pt/day)** = Σ `CapacityPerDay` (or convert hours → points if your org tracks hours; store a **conversion factor** in an environment variable, e.g., `HOURS_PER_POINT`).
   - **Remaining capacity this week**: multiply by working days remaining (exclude weekends/holidays; keep a simple India/IN calendar or an override list in Dataverse).
4. **Response JSON**
```json
{
  "iteration": "Sprint 24",
  "weekStart": "2025-11-03",
  "weekEnd": "2025-11-09",
  "teamCapacityPerDay": 21.5,
  "remainingWorkDays": 4,
  "estimatedRemainingCapacity": 86,
  "members": [
    {"user":"A. Singh","capPerDay":6,"vacationDays":0},
    {"user":"S. Ramtekkar","capPerDay":5.5,"vacationDays":1}
  ]
}
```

> If your tenant uses **TeamCapacity** instead of user capacity, add another connector path and aggregate accordingly.

---

## 6) Copilot Action Definitions & Prompts

In each Topic, **Call an action** (your flow). Use a **grounded prompt** so the LLM doesn’t improvise data. Example for “Stories in current iteration”:

**System message (Prompt)**
> You are Rally Delivery Copilot. Answer **only** with the JSON provided by the tool calls and summarize accurately. If a value is missing, say you couldn’t find it; don’t speculate.

**User turn → Variables**
- project = captured entity or fallback to a default project (env var `DEFAULT_PROJECT`).

**Response template (to user)**
```
**Stories – {iteration} ({count} items, {totalPoints} pts)**

State split: {byState}

Top 10 by status:
{#each stories.slice(0,10)}
- {id}: {title} — {owner} — {state} ({points} pts){/each}

Need more? Say "show all stories" or add a filter, e.g., "assigned to me" or "blocked only".
```

> In Copilot Studio, you can return **Tables** too. Keep rows ≤ 50; offer a follow‑up to export CSV via a dedicated flow.

---

## 7) Filtering Patterns (Examples)

- **Assigned to me**: add to `query` → `(Owner.EmailAddress = "{email}")` or `(Owner = "/user/{oid}")` depending on what your tenant returns. Map Teams/Entra email → Rally user via a small cache table if needed.
- **Blocked only**: add `(Blocked = true)`
- **By tag**: include `Tags.Name contains "{tag}"`
- **Across multiple projects**: use `Project.Name contains "{prefix}"` or iterate projects and merge.

---

## 8) Example Rally Queries (REST v2.x)

> Encode your query in parentheses. Combine with `AND` / `OR`.

- **Iterations in a project**
  `GET {{RALLY_BASE}}/iteration?fetch=Name,StartDate,EndDate,Project&query=(Project.Name = "My Project")&pagesize=200`

- **Current iteration by date** (filter client‑side after fetching): where `StartDate <= today <= EndDate`.

- **User stories in iteration**
  `GET {{RALLY_BASE}}/hierarchicalrequirement?fetch=FormattedID,Name,Owner,PlanEstimate,ScheduleState,Blocked,Iteration,Project&query=((Project.Name = "My Project") AND (Iteration.Name = "Sprint 24"))&pagesize=200`

- **User capacity**
  `GET {{RALLY_BASE}}/useriterationcapacity?fetch=Iteration,User,CapacityPerDay,Vacation&query=(Iteration.Name = "Sprint 24")`

---

## 9) Error Handling & Guardrails

- If Rally returns **401/403** → tell the user their permissions or API key may be missing for that project.
- If **no current iteration** found → ask for an iteration name or show the last/next iteration with dates.
- If **pagesize limit** reached → implement pagination loop in flow; tell user you’re showing the first N and offer CSV export.
- Validate entity values (e.g., iteration name exists in project) before calling data‑heavy endpoints.

---

## 10) Testing Scenarios (copy/paste)

1. *What’s the current sprint for **Platform Team**?*
2. *Show user stories for the current sprint in **Payments**.*
3. *Which stories assigned to **me** are blocked this sprint?*
4. *How much capacity do we have left this week in **Mobile**?*
5. *Give me a sprint summary for **Web**: planned vs accepted points.*

---

## 11) Nice-to‑Haves

- **CSV/Excel export** flow that takes the `stories` array and uploads to OneDrive/SharePoint; return a link.
- **Burndown snapshot** using Lookback API (if allowed) or approximate using daily counts of `ScheduleState`.
- **Caching**: Store the last current‑iteration per project in Dataverse with a 2‑hour TTL.
- **Access control**: Restrict projects per user via a mapping table.

---

## 12) Implementation Checklist

- [ ] Custom Connector created & tested against your Rally tenant
- [ ] Environment variables: `RALLY_BASE`, `DEFAULT_PROJECT`, `HOURS_PER_POINT`
- [ ] Flow A: Get Current Iteration
- [ ] Flow B: List Stories (Current Iteration)
- [ ] Flow C: Capacity This Week/Iteration
- [ ] Topics wired to actions, with entity extraction
- [ ] Clean response templates (tables + summaries)
- [ ] Error handling tested (no data, permission, pagination)
- [ ] Publish and share to pilot users

---

## 13) Quick Auth Options (pick the one your org supports)

- **API Key in header** (e.g., `X-RallyAPIKey: <key>`)
- **Basic Auth** with user:password (service account)
- **ZSESSIONID** (legacy). Avoid if possible.

> Confirm with your Rally admin which method is enabled, and update the Connector’s **Security** accordingly.

---

## 14) Snippets for Copilot Table Rendering

When your flow returns `stories`, you can render a table like:

```
| ID     | Title                    | Owner       | Pts | State        | Blocked |
|--------|--------------------------|-------------|-----|--------------|---------|
| US1234 | OAuth callback fix       | A. Singh    | 3   | In-Progress  | No      |
| US1235 | Capacity endpoint        | S. Ramtekkar| 5   | Defined      | Yes     |
```

Keep the table to top 25–50 rows; offer an *Export all as CSV* action for the full set.

---

## 15) Next Steps

1) Plug in your **project names** & **iteration naming pattern**.  
2) Decide **capacity units** (hours vs points) and set the conversion factor.  
3) Build Flows A–C and test them live from the Copilot chat.  
4) Share with a small pilot, collect gaps, then iterate on filters and summaries.

---

If you want, provide your **Rally base URL**, **auth method**, and a sample **project + iteration name**, and I’ll tailor the connector and queries exactly to your tenant.

