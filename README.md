# ACE Databricks — Production Notebooks

This folder contains the **production-ready** copies of the ACE pipeline
notebooks: no test cells, no diagnostic/verification stubs, no exploratory
output. They are generated automatically from the source notebooks in the
parent folder (`../`) by `../build_prod_notebooks.py`.

**Do not edit these notebooks directly** — edits are overwritten the next
time the build script runs. Edit the numbered source notebooks in `../`
instead, then re-run:

```bash
cd ACE-Databricks-Config
python3 build_prod_notebooks.py
```

> **Deployment model (read this first):** v1 production deployment is
> **manual notebook execution** in the order documented below. Packaging
> this repo as a Databricks Asset Bundle (`databricks.yaml`) is a planned
> follow-up, not part of this release — there is intentionally no
> `databricks.yaml` in the repo yet.

---

## Prerequisites — complete these BEFORE the setup session

These items have lead time and/or require elevated permissions. If they are
not in place, the notebooks will fail partway through and the session stalls.

| # | Prerequisite | Who can do it | Notes |
|---|---|---|---|
| 1 | **Unity Catalog catalog exists** (`ace_theorem`, or your chosen name) | Metastore admin / someone with `CREATE CATALOG` | `infrastructure.ipynb` creates the **schema, tables, index, and grants** — it does **not** create the catalog itself. Confirm the catalog exists and the person running the notebooks has `USE CATALOG` + `CREATE SCHEMA` on it. |
| 2 | **Service principal created in the target tenant** | Azure AD / Entra ID admin | App registration with a client secret. Record the client ID, secret, and tenant ID — they go into the secret scope in `infrastructure.ipynb` Cell 3. The SP must also be **added to the Databricks workspace** (Admin Settings → Identity → Service principals). |
| 3 | **Sonnet + Haiku external model endpoints registered** | Databricks workspace admin | Check **Model Serving → External Models** for `databricks-claude-sonnet-4-5`, `databricks-claude-sonnet-4-6`, and `databricks-claude-haiku-4-5` (or this workspace's equivalents). If the names differ, see [§4 Model-serving endpoint names](#4-model-serving-endpoint-names-sonnet--haiku). If the workspace has **no** Anthropic external models registered at all, an admin must create them first — this can take time if API keys/approvals are involved. |
| 4 | **SQL Warehouse available** | Workspace admin (to create) or anyone (to look up the ID) | Serverless or pro. Note the warehouse ID from **SQL Warehouses → your warehouse → Connection details**. |
| 5 | **UC volume for policy PDFs exists and PDFs are uploaded** | Anyone with `CREATE VOLUME` on the schema | Default path `/Volumes/ace_theorem/chat/pdf_assets`. The schema must exist first, so either pre-create it or do this step right after `infrastructure.ipynb` runs. |
| 6 | **Repo access** | ADO project admin | Whoever connects the repo to the workspace needs at least **Read** on `ACE-DatabricksAgent` in the `vbtx/ACE` ADO project. |

---

## Connecting this repo to the Databricks workspace (Azure DevOps)

1. In the Databricks workspace: **Workspace → Repos → Add → Repo** (or
   **Create → Git folder** in newer UI versions).
2. Repository URL:
   `https://dev.azure.com/vbtx/ACE/_git/ACE-DatabricksAgent`
   — Git provider: **Azure DevOps Services**.
3. **Authentication:** under **User Settings → Linked accounts → Git
   integration**, link Azure DevOps. Prefer the **Entra ID (Azure Active
   Directory)** credential flow over a personal access token — vbtx is an
   Entra tenant, and Entra-backed auth doesn't expire silently the way PATs
   do. If you must use a PAT, scope it to **Code (Read)** only.
4. The identity used here is fine as a **personal** credential for the
   initial clone and manual runs. If/when deployment is automated (asset
   bundle + Azure Pipeline), switch the integration to a service principal
   so nothing depends on an individual's linked account.
5. Pull the default branch and confirm this README and the five notebooks
   below are visible in the Git folder.

---

## What each notebook does

| File | Registers / Creates | Capability |
|---|---|---|
| [`infrastructure.ipynb`](infrastructure.ipynb) | Delta tables (`policy_documents`, `inference_logs`, `session_names`), Vector Search endpoint + index, Databricks secret scope, Unity Catalog grants | One-time setup of every persistent resource the rest of the pipeline depends on. |
| [`pdf_ingestion.ipynb`](pdf_ingestion.ipynb) | Rows in `policy_documents`, syncs the Vector Search index | Parses lending-policy PDFs from a Unity Catalog volume, chunks them by section/heading, writes chunks to Delta, triggers re-embedding. |
| [`rag_agent.ipynb`](rag_agent.ipynb) | `ace_theorem.chat.rag_agent` model + `ace-rag-agent-endpoint` | Policy Q&A agent. Searches `policy_documents_index` via Vector Search, calls Claude Sonnet to produce a cited, grounded answer. |
| [`page_context_agent.ipynb`](page_context_agent.ipynb) | `ace_theorem.chat.page_context_agent` model + `ace-lending-context-endpoint` | Answers questions about the loan/page the officer is currently viewing, using only structured data sent live by the Blazor app. No Vector Search dependency. |
| [`supervisor_deploy.ipynb`](supervisor_deploy.ipynb) | `ace_theorem.chat.supervisor_agent` model + `ace-chat-endpoint-v2` | Top-level orchestrator. Classifies intent (RAG / PAGE / BOTH / NONE), calls the two sub-agent endpoints above, synthesizes a single streamed answer with Sonnet, and logs every turn to `inference_logs` (+ `session_names` on the first turn of a session). |

`06_monitoring_agent.ipynb` (in the parent folder) has **no prod equivalent**
— it's a dev/diagnostic notebook only.

---

## Execution order for a fresh workspace

Run these **in order**, waiting for each to fully finish (and for serving
endpoints to print `READY`) before starting the next.

Approximate durations are for a warm workspace; first-time resource
provisioning can be slower. The slow steps are **Vector Search endpoint +
index creation** (step 1) and **each model-serving endpoint deployment**
(steps 3–5) — these routinely take **10–20+ minutes each**. If an endpoint
sits in `NOT_READY`/`UPDATING` for under ~25 minutes, that's normal; don't
re-run the cell.

| Step | Notebook | ~Duration | What to watch for |
|---|---|---|---|
| 1 | `infrastructure.ipynb` | 15–30 min (VS endpoint is the long pole) | Run once. Creates everything below. After Cell 10 finishes, manually confirm in the Databricks UI: the 3 Delta tables exist, the VS endpoint is `ONLINE`, the VS index is `ready=True`, and the SP has the listed grants. |
| 2 | `pdf_ingestion.ipynb` | 5–15 min depending on PDF count | Confirm `VOLUME_PATH` (Cell 2) points at a UC volume that already has your PDFs uploaded. Leave `WIPE_BEFORE_INGEST=True` for the first run. |
| 3 | `rag_agent.ipynb` | 10–20 min | Wait for `"ace-rag-agent-endpoint is READY"`. |
| 4 | `page_context_agent.ipynb` | 10–20 min | Wait for `"ace-lending-context-endpoint is READY"`. |
| 5 | `supervisor_deploy.ipynb` | 10–20 min | Depends on the endpoints from steps 3 & 4 being live (their names are hardcoded inside the Supervisor model — see below). Wait for `"ace-chat-endpoint-v2 is READY"`, then complete the [Handoff to App Dev](#handoff-to-app-dev-ace-lending) below. |

### `WIPE_BEFORE_INGEST` (notebook 2, Cell 2)

| Value | Behaviour |
|---|---|
| `True` | Deletes all rows from `policy_documents` before ingesting — guarantees a clean reload. Use for the first run and for any full reload of policy documents. |
| `False` | Append-only. Only safe when adding brand-new PDFs that have never been ingested before. |

---

## Setting this up in a new Databricks workspace — what to change

Every name below currently points at the original workspace
(catalog `ace_theorem`, schema `chat`, etc.). If you are deploying into a
**different** Unity Catalog catalog/schema, a different SQL warehouse, or a
different model-serving setup, you must update **all** of the locations
listed — the notebooks intentionally repeat some values because the
registered model classes must be fully self-contained (a Model Serving
container has no access to notebook variables).

### 1. Catalog / schema / table / index names

| Setting | Current value | Where it's defined (source of truth) | Also hardcoded in |
|---|---|---|---|
| `CATALOG` | `ace_theorem` | `infrastructure.ipynb` Cell 2 | `pdf_ingestion.ipynb` Cell 2, `rag_agent.ipynb` Cell 2, `page_context_agent.ipynb` Cell 2, `supervisor_deploy.ipynb` Cell 2 |
| `SCHEMA` | `chat` | `infrastructure.ipynb` Cell 2 | same as above |
| `VS_ENDPOINT_NAME` | `ace-chat-vs-endpoint` | `infrastructure.ipynb` Cell 2 | `pdf_ingestion.ipynb` Cell 2, `rag_agent.ipynb` Cell 2 + the `RagAgentModel` class constant inside `rag_agent.ipynb` Cell 3, `supervisor_deploy.ipynb` Cell 2 |
| `VS_INDEX_NAME` | `ace_theorem.chat.policy_documents_index` | derived from `CATALOG`/`SCHEMA` in `infrastructure.ipynb` Cell 2 | hardcoded **literal string** inside `RagAgentModel.VS_INDEX_NAME` and the `DatabricksVectorSearchIndex(index_name=...)` resource declaration in `rag_agent.ipynb` Cell 3; also `supervisor_deploy.ipynb` Cell 2 |
| `EMBEDDING_MODEL` | `databricks-gte-large-en` | `infrastructure.ipynb` Cell 2 (baked into the index at creation) | `pdf_ingestion.ipynb` Cell 2 (informational only — changing it here has no effect) |
| `TABLE_LOG` (`inference_logs`) | `ace_theorem.chat.inference_logs` | `infrastructure.ipynb` Cell 2 | hardcoded **literal SQL strings** inside `SupervisorAgentModel._log_inference()` and `_fetch_prior_context()` in `supervisor_deploy.ipynb` Cell 3 |
| `TABLE_SESSION_NAMES` (`session_names`) | `ace_theorem.chat.session_names` | `infrastructure.ipynb` Cell 2 | hardcoded **literal SQL string** inside `SupervisorAgentModel._save_session_name()` in `supervisor_deploy.ipynb` Cell 3 |

> The model classes (`RagAgentModel`, `PageContextAgentModel`,
> `SupervisorAgentModel`) cannot read notebook-level config at serving time,
> so any catalog/schema rename must be repeated **inside the class body**,
> not just in the Cell 2 config block of each notebook.

### 2. Service principal (SP) credentials & secrets

> ⚠️ **Do not commit secrets.** This is a repo-backed (Git) folder: anything
> typed into a notebook cell is one `git commit` away from living permanently
> in Azure DevOps history. After Cell 3 runs successfully and the secret
> scope exists, **revert the cell to its placeholder values before doing
> anything else** (in the Git folder UI: right-click the notebook → Git →
> discard changes, or simply re-pull). Never push a notebook containing a
> real client secret. If a secret is ever committed, treat it as compromised
> and rotate it in Azure Portal immediately.
>
> Longer-term, prefer wiring the scope to **Azure Key Vault** (which vbtx
> already uses for the app side) so secret values never touch a notebook at
> all.

| Setting | Current value | Where it's defined |
|---|---|---|
| Secret scope | `ace-secrets` | Created in `infrastructure.ipynb` Cell 3; consumed by `rag_agent.ipynb` Cell 4, `supervisor_deploy.ipynb` Cell 4 (`{{secrets/ace-secrets/...}}`) |
| `SP_CLIENT_ID` | `fb330a4b-c2bb-48b9-926c-b33a3650e1b7` (Azure AD app/client ID) | `infrastructure.ipynb` Cell 3 — replace with the new workspace's SP app ID |
| `SP_CLIENT_SECRET` | placeholder `"<paste the SP client secret here>"` | `infrastructure.ipynb` Cell 3 — paste from Azure Portal → App Registrations → Certificates & Secrets, run the cell, then **revert the cell** (see warning above) |
| `SP_TENANT_ID` | placeholder `"<paste your Azure AD tenant ID here>"` | `infrastructure.ipynb` Cell 3 — paste from Azure Portal → Azure AD → Overview |
| `DATABRICKS_SCOPE` (MSAL scope) | `2ff814a6-3304-4ab8-85cb-cd0e6f879c1d/.default` | hardcoded inside `RagAgentModel._get_token()` (`rag_agent.ipynb` Cell 3) and `SupervisorAgentModel._get_token()` (`supervisor_deploy.ipynb` Cell 3) — this is the **global Azure resource ID for the Azure Databricks service** and is the same in every tenant; it should not need to change, but verify token acquisition succeeds in the new tenant before deploying |

The infrastructure notebook will refuse to run Cell 3 until the secret and
tenant ID placeholders are filled in (it raises a `ValueError`).

The SP also needs, in the **new** workspace (granted by `infrastructure.ipynb`
where noted, otherwise manually):

- `USE CATALOG` / `USE SCHEMA` / `SELECT` / `MODIFY` on the tables (granted
  by `infrastructure.ipynb` Cell 9–10 — verify after run)
- `CAN_USE` on the SQL warehouse (granted by `infrastructure.ipynb` Cell 9)
- `CAN_QUERY` on `ace-rag-agent-endpoint` and `ace-lending-context-endpoint`
  (the Supervisor calls them with the SP token)

### 3. SQL Warehouse

| Setting | Current value | Where it's defined |
|---|---|---|
| `SQL_WAREHOUSE_ID` | `b8583158cc1cf9e3` | `infrastructure.ipynb` Cell 9 (grants the SP `CAN_USE`) and `supervisor_deploy.ipynb` Cell 2 (used by `_log_inference` and `_save_session_name` to write to `inference_logs` / `session_names` via the SQL Statement Execution API) |

Find the new warehouse's ID in the Databricks UI under
**SQL Warehouses → your warehouse → Connection details**, and update both
locations.

### 4. Model-serving endpoint names (Sonnet / Haiku)

| Setting | Current value | Where it's defined |
|---|---|---|
| Sonnet external model endpoint | `databricks-claude-sonnet-4-5` (rag/page agents) and `databricks-claude-sonnet-4-6` (supervisor synthesis) | Class constant `LLM_ENDPOINT` inside `RagAgentModel` (`rag_agent.ipynb` Cell 3), `PageContextAgentModel` (`page_context_agent.ipynb` Cell 3), and `SupervisorAgentModel` (`supervisor_deploy.ipynb` Cell 3) — also the `DatabricksServingEndpoint(...)` entries in each notebook's `resources=[...]` block |
| Haiku external model endpoint | `databricks-claude-haiku-4-5` | `FAST_LLM_ENDPOINT` in `supervisor_deploy.ipynb` Cell 2/3, and the `resources=[...]` blocks in `rag_agent.ipynb`, `page_context_agent.ipynb`, `supervisor_deploy.ipynb` |

**Verify before running anything:** Databricks Admin → Model Serving →
External Models. The names above must match an actual registered external
model in the new workspace, or model registration/deployment will fail.
If the name differs, edit `SONNET_ENDPOINT` / `HAIKU_ENDPOINT` at the top of
`../build_prod_notebooks.py` and re-run it — it substitutes `LLM_ENDPOINT`
in notebooks 03/04/05 in one pass (note: it does **not** rewrite the
hardcoded names inside `resources=[...]` blocks or `FAST_LLM_ENDPOINT` —
those need a manual check too).

### 5. Sub-agent endpoint names (Supervisor wiring)

| Setting | Current value | Where it's defined |
|---|---|---|
| `RAG_ENDPOINT_NAME` | `ace-rag-agent-endpoint` | Set when `rag_agent.ipynb` Cell 4 deploys; referenced as a class constant + `resources=[...]` entry inside `SupervisorAgentModel` (`supervisor_deploy.ipynb` Cell 3/2) |
| `PAGE_ENDPOINT_NAME` | `ace-lending-context-endpoint` | Set when `page_context_agent.ipynb` Cell 4 deploys; referenced the same way in `supervisor_deploy.ipynb` |
| `ENDPOINT_NAME` (final chat endpoint) | `ace-chat-endpoint-v2` | `supervisor_deploy.ipynb` Cell 2 |

These three names only need to change if you want different endpoint names
in the new workspace — otherwise they can be left as-is, since each notebook
creates the endpoint it references under that exact name.

---

## Assets needed and where they go

| Asset | Needed by | Location |
|---|---|---|
| Lending policy PDFs | `pdf_ingestion.ipynb` | Upload to the Unity Catalog volume at `VOLUME_PATH` (default `/Volumes/ace_theorem/chat/pdf_assets`, set in `pdf_ingestion.ipynb` Cell 2). Place files directly in this folder — subdirectories are **not** traversed. PDFs must have a real text layer (no OCR is performed); image-only/scanned PDFs produce zero chunks. |
| Azure SP credentials (client ID, secret, tenant ID) | `infrastructure.ipynb` Cell 3 (stores them), then read at serving time by `rag_agent.ipynb` and `supervisor_deploy.ipynb` | Databricks secret scope `ace-secrets` (created automatically by Cell 3 once you fill in the placeholders) |
| SQL Warehouse (existing or new) | `infrastructure.ipynb` Cell 9 (grants `CAN_USE`), `supervisor_deploy.ipynb` (writes `inference_logs` / `session_names`) | Reference by ID — see "SQL Warehouse" above |
| Sonnet + Haiku external model endpoints | `rag_agent.ipynb`, `page_context_agent.ipynb`, `supervisor_deploy.ipynb` | Must already be registered under **Model Serving → External Models** before running these notebooks |
| `ace-chat-endpoint-v2` URL | ACE-Lending Blazor app | Printed at the end of `supervisor_deploy.ipynb` Cell 4 — see [Handoff to App Dev](#handoff-to-app-dev-ace-lending) |

---

## Handoff to App Dev (ACE-Lending)

The App Dev sprint release depends on this handoff. After
`supervisor_deploy.ipynb` finishes, deliver **all** of the following — the
URL alone is not enough:

1. **Endpoint URL** — printed as `ENDPOINT_URL` at the end of
   `supervisor_deploy.ipynb` Cell 4. Goes into the ACE-Lending app's
   `appsettings.json`.
2. **Endpoint name** — `ace-chat-endpoint-v2` (App Dev references it in
   logging/monitoring config).
3. **Permission grant** — the **ACE-Lending app's service principal**
   (the identity the Blazor app uses for its M2M OAuth client-credentials
   flow) must be granted **CAN_QUERY** on `ace-chat-endpoint-v2` in this
   workspace: Serving → `ace-chat-endpoint-v2` → Permissions. Without this
   grant the app gets `403`s even with a valid token.
4. **Tenant/auth confirmation** — confirm with App Dev that the tenant ID
   and Databricks OAuth scope in the app's config match this workspace's
   tenant. If prod is a different tenant or subscription than sandbox, the
   app's token requests must change too.

**Smoke test before declaring ready:** run the test cell at the end of
`supervisor_deploy.ipynb` (or `curl` the endpoint with an SP token) and
confirm a real streamed answer comes back — including one question that
exercises the RAG path, so the full Supervisor → sub-agent → Vector Search
chain is proven. Do this **before** handing the URL to App Dev, so any
deployment issue is found here and not during their release testing.

---

## Migration checklist (summary)

0. Complete the [Prerequisites](#prerequisites--complete-these-before-the-setup-session)
   (catalog, SP, external model endpoints, SQL warehouse, PDF volume, repo
   access) and [connect the repo](#connecting-this-repo-to-the-databricks-workspace-azure-devops)
   to the workspace.
1. In the **new** workspace, edit `infrastructure.ipynb` Cell 2 (and the
   mirrored values listed in the tables above) if you want a different
   catalog/schema/index/table naming scheme. If keeping `ace_theorem.chat`,
   no changes are needed here.
2. Fill in `SP_CLIENT_ID`, `SP_CLIENT_SECRET`, `SP_TENANT_ID` in
   `infrastructure.ipynb` Cell 3 for the new tenant's service principal.
   **Revert the cell after it runs — never commit a real secret.**
3. Update `SQL_WAREHOUSE_ID` in `infrastructure.ipynb` Cell 9 and
   `supervisor_deploy.ipynb` Cell 2.
4. Verify the Sonnet/Haiku external model endpoint names exist in the new
   workspace (Model Serving → External Models). Update via
   `../build_prod_notebooks.py` if they differ, then re-run the build.
5. Upload policy PDFs to the UC volume referenced by `VOLUME_PATH` in
   `pdf_ingestion.ipynb` Cell 2 (create the volume first if it doesn't exist).
6. Run the five notebooks in the order in the table above, confirming each
   serving endpoint reaches `READY` before moving to the next.
7. Run the smoke test, then complete the
   [Handoff to App Dev](#handoff-to-app-dev-ace-lending): endpoint URL into
   `appsettings.json`, **CAN_QUERY** grant for the app's service principal,
   and tenant/auth confirmation.