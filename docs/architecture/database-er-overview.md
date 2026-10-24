---
title: Adatbázis ER-áttekintés
summary: A Paperclip adatmodelljének lényegi entitásai, tulajdonságai és kapcsolatai mermaid ER-diagramokon
---

# Adatbázis ER-áttekintés

Állapot: tanulási / feltérképező anyag (nem normatív)
Forrás: `packages/db/src/schema/*.ts` (Drizzle ORM), `71c04c0` commit, 2026-10-24
Méret: 173 sémafájl, 301 tábla, 931 idegen kulcs, 377 migráció (`packages/db/src/migrations/`)

## 1. Cél és olvasási útmutató

A dokumentum célja, hogy a Paperclip **működését az adatmodellen keresztül** lehessen
megérteni. Nem a teljes séma másolata: a ~300 táblából a működést hordozó
83 entitás szerepel diagramon, mindegyik csak a **kulcsmezőivel**. A többi tábla a
[8. fejezet](#8-teljes-táblaindex-domainenként) indexében domainenként be van sorolva, így
semmi nem marad térkép nélkül.

Felépítés (egy diagram = egy felelősségi kör):

1. Globális konvenciók – a mindenhol visszatérő minták
2. Mag-áttekintés – a 16 központi entitás egy képen
3. Domain-szeletek – 8 domain, saját diagrammal
4. Rétegek és adatfolyam – hogyan jut el a sor a UI-ig
5. Megfigyelések – tanulási pontok
6. Teljes táblaindex

### Jelmagyarázat

| Jel | Jelentés |
|---|---|
| `PK` / `FK` / `UK` | elsődleges kulcs / idegen kulcs / egyedi kulcs |
| `\|\|--o{` | kötelező szülő (NOT NULL FK), 0..N gyerek |
| `\|o--o{` | opcionális szülő (nullable FK), 0..N gyerek |
| `\|\|--o\|` | 1 : 0..1 (pl. a gyerek PK-ja egyben FK) |
| `}o--o{` | N : M kapcsolótáblán keresztül (a címke a kapcsolótábla) |
| szaggatott vonal (`..`) | **logikai** hivatkozás, adatbázis-szintű FK nélkül (pl. `text` user ID, polimorf `*_type` + `*_id`) |
| típusok | `uuid`, `text`, `int`, `bigint`, `numeric`, `bool`, `jsonb`, `ts` (= `timestamptz`) |

A `created_at` / `updated_at` mezőket a diagramok nem mutatják; szinte minden táblában jelen vannak.

## 2. Globális konvenciók

- **Azonosítók.** A domain-táblák PK-ja `uuid` (`defaultRandom()`). Kivétel a Better Auth
  táblái (`user`, `account`, `session`, `verification` – `auth.ts`), ahol a PK `text`, és
  a `heartbeat_run_events`, ahol `bigserial` (append-only napló).
- **Multi-tenancy (company scope).** 268 táblában van `company_id`, ebből 252-ben valódi FK a
  `companies.id`-ra. Ez az `AGENTS.md` 5.1 invariánsa: minden domain-entitás egy céghez
  tartozik. Instance-szintű (cég nélküli) táblák pl.: `plugins`, `environments`,
  `instance_settings`, `instance_user_roles`, a Better Auth táblák, `mcp_oauth_*`.
- **Tenant-biztos összetett FK-k.** 47 tábla 95 olyan FK-t tartalmaz, ami `(company_id, x_id)`
  párra mutat (a szülőn `UNIQUE (company_id, id)`). Így az adatbázis maga akadályozza meg,
  hogy egy sor más cég entitására hivatkozzon. Példák: `chat_endpoints → agents`,
  `connection_grants → tool_connections`, `work_assessments → heartbeat_runs`.
- **Polimorf aktor (agent vagy ember).** Ahol egy műveletet ágens és ember is végezhet, két
  oszlop van: `*_agent_id uuid FK → agents` és `*_user_id text` (Better Auth `user.id`,
  többnyire **FK nélkül**). Pl. `issues.assignee_agent_id` / `assignee_user_id`,
  `issue_comments.author_agent_id` / `author_user_id`.
- **Polimorf principal.** Jogosultsági táblák `principal_type` (`"user" | "agent"`,
  `PRINCIPAL_TYPES` a `packages/shared/src/constants.ts`-ben) + `principal_id text`
  párossal dolgoznak (`company_memberships`, `principal_permission_grants`).
  Hasonlóan: `activity_log.actor_type/actor_id`, `budget_policies.scope_type/scope_id`.
- **Hierarchiák önhivatkozással.** `goals.parent_id`, `issues.parent_id`,
  `agents.reports_to` (szervezeti fa), `folders.parent_id`, `cases.parent_case_id`.
- **Állapot és konfiguráció.** `status` mezők `text` típusú enumok (az értékkészlet a
  `packages/shared` konstansaiban és validátoraiban él), a rugalmas konfiguráció `jsonb`.
- **Törlési szabályok.** 931 FK-ból 454 `cascade`, 255 `set null`, 18 `restrict`,
  204 alapértelmezett (`no action`). A gyerek-/kapcsolótáblák jellemzően `cascade`-elnek,
  a futási hivatkozások (`*_run_id`) jellemzően `set null`-t használnak.

## 3. Mag-áttekintés

A 16 központi entitás. Az FK-gráf alapján a legtöbbet hivatkozott táblák: `companies` (252),
`agents` (134), `issues` (73), `heartbeat_runs` (58). A `companies` kapcsolatát csak a
fő entitásoknál rajzoltuk be – valójában szinte minden tábla a céghez tartozik.

```mermaid
erDiagram
    companies ||--o{ company_memberships : "tagság"
    user |o..o{ company_memberships : "principal (user)"
    companies ||--o{ agents : "alkalmaz"
    agents |o--o{ agents : "reports_to"
    companies ||--o{ goals : ""
    goals |o--o{ goals : "parent_id"
    companies ||--o{ projects : ""
    goals |o--o{ projects : "goal_id"
    projects ||--o{ project_workspaces : "repo / cwd"
    companies ||--o{ issues : ""
    projects |o--o{ issues : "project_id"
    goals |o--o{ issues : "goal_id"
    issues |o--o{ issues : "parent_id"
    agents |o--o{ issues : "assignee_agent_id"
    issues ||--o{ issue_comments : "szál"
    agents ||--o{ heartbeat_runs : "futtat"
    issues |o--o{ heartbeat_runs : "issue_id"
    heartbeat_runs |o--o{ issues : "checkout_run_id"
    execution_workspaces |o--o{ issues : "execution_workspace_id"
    project_workspaces |o--o{ execution_workspaces : ""
    issues }o--o{ documents : "issue_documents"
    agents |o--o{ approvals : "requested_by"
    issues }o--o{ approvals : "issue_approvals"
    heartbeat_runs |o--o{ cost_events : "költség"
    agents |o--o{ cost_events : ""
    agents |o--o{ routines : "assignee"
    routines }o--o{ issues : "routine_runs"
    heartbeat_runs |o--o{ activity_log : "run_id"
```

**Egy mondatban:** a *company* (AI-ágens cég) *agent*-eket alkalmaz, akik *goal*-okhoz és
*project*-ekhez kötött *issue*-kat (feladatokat) dolgoznak fel *heartbeat run*-okban, egy
*execution workspace*-ben; minden futás *cost event*-eket és *activity log* bejegyzéseket
termel, a kormányzott lépésekhez *approval* kell, az ismétlődő munkát *routine*-ok indítják.

## 4. Domain-szeletek

### 4.1 Tenant, identitás, hozzáférés

A `companies` a tenant gyökere. Az emberek a Better Auth `user` táblában élnek; a cégtagság
polimorf (`company_memberships`), így ember és ágens is lehet tag. Az ágensek bearer API
kulccsal (`agent_api_keys`, hash-elve), az operátorok board kulccsal vagy sessionnel
azonosítanak. Új tagok meghívón (`invites`) és csatlakozási kérelmen (`join_requests`) át
érkeznek; a kérelem jóváhagyása ágenst is létrehozhat.

```mermaid
erDiagram
    companies {
        uuid id PK
        text name
        text status "active | paused | archived"
        text issue_prefix UK "pl. PAP"
        int issue_counter "issue azonosító sorszám"
        int budget_monthly_cents
        bool require_board_approval_for_new_agents
    }
    user {
        text id PK "Better Auth"
        text name
        text email
        bool email_verified
    }
    account {
        text id PK
        text user_id FK
        text provider_id
        text account_id
    }
    session {
        text id PK
        text user_id FK
        text token
        ts expires_at
    }
    company_memberships {
        uuid id PK
        uuid company_id FK
        text principal_type "user | agent"
        text principal_id
        text status
        text membership_role
    }
    principal_permission_grants {
        uuid id PK
        uuid company_id FK
        text principal_type
        text principal_id
        text permission_key
        jsonb scope
    }
    instance_user_roles {
        uuid id PK
        text user_id
        text role "pl. instance admin"
    }
    invites {
        uuid id PK
        uuid company_id FK "nullable"
        text invite_type
        text token_hash UK
        ts expires_at
    }
    join_requests {
        uuid id PK
        uuid invite_id FK, UK
        uuid company_id FK
        text request_type
        text status
        uuid created_agent_id FK
    }
    board_api_keys {
        uuid id PK
        text user_id FK
        text key_hash UK
        ts revoked_at
    }
    agent_api_keys {
        uuid id PK
        uuid agent_id FK
        uuid company_id FK
        text key_hash
        jsonb scope_config
    }
    agents {
        uuid id PK
        uuid company_id FK
        text name
    }

    user ||--o{ account : ""
    user ||--o{ session : ""
    user ||--o{ board_api_keys : ""
    user |o..o{ instance_user_roles : "user_id"
    companies ||--o{ company_memberships : ""
    user |o..o{ company_memberships : "principal_type=user"
    agents |o..o{ company_memberships : "principal_type=agent"
    companies ||--o{ principal_permission_grants : ""
    companies |o--o{ invites : ""
    invites ||--o| join_requests : ""
    companies ||--o{ join_requests : ""
    agents |o--o{ join_requests : "created_agent_id"
    companies ||--o{ agents : ""
    agents ||--o{ agent_api_keys : ""
```

További táblák: `cli_auth_challenges`, `issue_access_grants`, `project_memberships`,
`project_access_members`, `document_memberships`, `run_identity_contexts`,
`agent_identity_keys`, felhasználói preferenciák (`*_preferences`, `inbox_dismissals`).

### 4.2 Ágensek

Az `agents` a második legfontosabb tábla. Egy ágensnek van szerepe, adaptere
(`adapter_type` + `adapter_config`, pl. Claude/Codex), saját havi költségkerete és
életciklus-állapota. A szervezeti fát a `reports_to` önhivatkozás adja. A konfiguráció- és
instrukcióváltozások revíziós táblákba kerülnek (auditálható, visszaállítható). A futás
közötti állapot (session, tokenösszesítők) az `agent_runtime_state`-ben, a feladatonkénti
adapter-session az `agent_task_sessions`-ben él. A felébresztési kérések
(`agent_wakeup_requests`) idempotens kulccsal sorba állnak, és `heartbeat_runs`-t indítanak.

```mermaid
erDiagram
    agents {
        uuid id PK
        uuid company_id FK
        text name
        text role "general | ceo | cto ..."
        text status "idle | running | paused | error ..."
        text lifecycle_state
        uuid reports_to FK "szervezeti fa"
        text adapter_type
        jsonb adapter_config
        uuid default_environment_id FK
        int budget_monthly_cents
        jsonb permissions
    }
    environments {
        uuid id PK "instance-szintű"
        text name UK
        text driver
        jsonb config
    }
    agent_config_revisions {
        uuid id PK
        uuid agent_id FK
        text source
        jsonb changed_keys
        jsonb before_config
        jsonb after_config
    }
    agent_instruction_revisions {
        uuid id PK
        uuid agent_id FK "company_id+agent_id"
        text entry_file
        text content_hash
        uuid parent_revision_id
    }
    agent_runtime_state {
        uuid agent_id PK, FK
        text session_id
        jsonb state_json
        bigint total_input_tokens
        numeric total_cost_cents
    }
    agent_task_sessions {
        uuid id PK
        uuid agent_id FK
        text task_key
        jsonb session_params_json
        uuid last_run_id FK
    }
    agent_wakeup_requests {
        uuid id PK
        uuid agent_id FK
        text source
        text status
        text idempotency_key
        int coalesced_count
    }
    agent_memberships {
        uuid id PK
        uuid agent_id FK
        text user_id
        text state
    }
    heartbeat_runs {
        uuid id PK
        uuid agent_id FK
        uuid wakeup_request_id FK
    }

    agents |o--o{ agents : "reports_to"
    environments |o--o{ agents : "default_environment_id"
    agents ||--o{ agent_config_revisions : ""
    agents ||--o{ agent_instruction_revisions : ""
    agents ||--o| agent_runtime_state : "1:1"
    agents ||--o{ agent_task_sessions : ""
    heartbeat_runs |o--o{ agent_task_sessions : "last_run_id"
    agents ||--o{ agent_wakeup_requests : ""
    agent_wakeup_requests |o--o{ heartbeat_runs : "elindítja"
    agents ||--o{ heartbeat_runs : ""
    agents ||--o{ agent_memberships : "user kedvencek"
```

További táblák: `agent_instruction_heads`, `agent_instruction_working_copies`,
`agent_session_goal_actions`, `agent_commentary`, `managed_agent_profiles`,
`remote_agent_profiles`, `adapter_auth_sessions`, külső runnerek
(`muse_runner.ts`, `dot_runner.ts`, `grok_bot.ts`).

### 4.3 Munkaszervezés (célok, projektek, feladatok)

A munka hierarchiája: **goal → project → issue**. A célok fát alkotnak (`level`:
company / team / agent / task). A projekt opcionálisan egy célhoz tartozik, és több célhoz is
köthető (`project_goals`). A projekt munkaterületei (`project_workspaces`) a repo/cwd
forrást írják le. Az `issues` a rendszer munkaegysége: **egyetlen felelőse** van (ágens
*vagy* ember), atomikus checkout-tal (`checkout_run_id`, `execution_locked_at`) zárolható,
és szülő-gyerek fát alkot. Körülötte kommentek, címkék, relációk (pl. blokkolás),
mellékletek és munkatermékek (PR, branch, preview URL, artefakt) csoportosulnak.

```mermaid
erDiagram
    goals {
        uuid id PK
        uuid company_id FK
        text title
        text level "company | team | agent | task"
        text status
        uuid parent_id FK
        uuid owner_agent_id FK
    }
    projects {
        uuid id PK
        uuid company_id FK
        uuid goal_id FK
        text name
        text status
        text visibility
        uuid lead_agent_id FK
        jsonb execution_workspace_policy
    }
    project_goals {
        uuid project_id PK, FK
        uuid goal_id PK, FK
    }
    project_workspaces {
        uuid id PK
        uuid project_id FK
        text source_type
        text repo_url
        text cwd
        bool is_primary
    }
    issues {
        uuid id PK
        uuid company_id FK
        text identifier UK "pl. PAP-42"
        text title
        text status "backlog | todo | in_progress | done ..."
        text priority
        uuid parent_id FK
        uuid project_id FK
        uuid goal_id FK
        uuid assignee_agent_id FK
        text assignee_user_id
        uuid checkout_run_id FK
        uuid execution_workspace_id FK
    }
    issue_comments {
        uuid id PK
        uuid issue_id FK
        uuid author_agent_id FK
        text author_user_id
        uuid created_by_run_id FK
        text body
    }
    issue_relations {
        uuid id PK
        uuid issue_id FK
        uuid related_issue_id FK
        text type "pl. blocks"
    }
    labels {
        uuid id PK
        uuid company_id FK
        text name UK
        text color
    }
    issue_labels {
        uuid issue_id PK, FK
        uuid label_id PK, FK
    }
    issue_attachments {
        uuid id PK
        uuid issue_id FK
        uuid asset_id FK, UK
        uuid issue_comment_id FK
    }
    issue_work_products {
        uuid id PK
        uuid issue_id FK
        text type "pull_request | branch | artifact ..."
        text provider
        text url
        text review_state
    }
    agents {
        uuid id PK
    }

    goals |o--o{ goals : "parent_id"
    agents |o--o{ goals : "owner"
    goals |o--o{ projects : "goal_id"
    projects ||--o{ project_goals : ""
    goals ||--o{ project_goals : ""
    agents |o--o{ projects : "lead"
    projects ||--o{ project_workspaces : ""
    projects |o--o{ issues : ""
    goals |o--o{ issues : ""
    issues |o--o{ issues : "parent_id"
    agents |o--o{ issues : "assignee"
    issues ||--o{ issue_comments : ""
    agents |o--o{ issue_comments : "author"
    issues ||--o{ issue_relations : "issue_id"
    issues ||--o{ issue_relations : "related_issue_id"
    issues ||--o{ issue_labels : ""
    labels ||--o{ issue_labels : ""
    issues ||--o{ issue_attachments : ""
    issue_comments |o--o{ issue_attachments : ""
    issues ||--o{ issue_work_products : ""
```

További táblák: `issue_thread_interactions` (strukturált kérdés-válasz a szálban), az
`issue_*` család többi tagja (watchdogok, tree hold-ok, read state, recovery), `cases`
(ügykezelés), `pipelines` / `pipeline_*`, `workflows`, `folders`, `feedback_*`.

### 4.4 Végrehajtás (heartbeat runok és munkaterületek)

Az ágens nem folyamatosan fut: **heartbeat**-ekben ébred. Egy `agent_wakeup_requests` sor
egy `heartbeat_runs` sort indít (a futás állapota, kimenet, költség-elszámolás állapota,
újrapróbálás `retry_of_run_id`-val). A futás részletes eseménynaplója a
`heartbeat_run_events` (a „run log”, csak a helyi DB-ben). A kód egy
`execution_workspaces`-ben fut (projekt-munkaterületből származtatott git ág / cwd), amit
`environment_leases` köt egy futtatókörnyezethez. A munkaterületen futó parancsok
(`workspace_operations`) és szolgáltatások (`workspace_runtime_services`) külön táblában
naplózódnak.

```mermaid
erDiagram
    agent_wakeup_requests {
        uuid id PK
        uuid agent_id FK
        text source
        text status
    }
    heartbeat_runs {
        uuid id PK
        uuid company_id FK
        uuid agent_id FK
        uuid issue_id FK
        uuid wakeup_request_id FK
        text invocation_source "timer | assignment | on_demand ..."
        text status "queued | running | succeeded | failed | timed_out ..."
        ts started_at
        ts finished_at
        int exit_code
        bool cost_accounting_pending
        uuid retry_of_run_id FK
    }
    heartbeat_run_events {
        bigint id PK
        uuid run_id FK
        bigint seq UK "run_id+seq"
        text event_type
        text stream
        jsonb payload
    }
    issues {
        uuid id PK
        uuid checkout_run_id FK
        uuid execution_workspace_id FK
    }
    project_workspaces {
        uuid id PK
    }
    execution_workspaces {
        uuid id PK
        uuid project_id FK
        uuid project_workspace_id FK
        uuid source_issue_id FK
        text mode
        text status
        text branch_name
        text provider_type
        uuid derived_from_execution_workspace_id FK
    }
    environments {
        uuid id PK
        text driver
    }
    environment_leases {
        uuid id PK
        uuid environment_id FK
        uuid execution_workspace_id FK
        uuid heartbeat_run_id FK
        text status
        ts expires_at
    }
    workspace_operations {
        uuid id PK
        uuid execution_workspace_id FK
        uuid heartbeat_run_id FK
        text phase
        text command
        int exit_code
    }
    workspace_runtime_services {
        uuid id PK
        uuid execution_workspace_id FK
        text service_name
        int port
        text url
        text health_status
    }

    agent_wakeup_requests |o--o{ heartbeat_runs : "elindítja"
    heartbeat_runs |o--o{ heartbeat_runs : "retry_of_run_id"
    heartbeat_runs ||--o{ heartbeat_run_events : "run log"
    issues |o--o{ heartbeat_runs : "issue_id"
    heartbeat_runs |o--o{ issues : "checkout / execution run"
    project_workspaces |o--o{ execution_workspaces : ""
    issues |o--o{ execution_workspaces : "source_issue_id"
    execution_workspaces |o--o{ issues : "execution_workspace_id"
    execution_workspaces |o--o{ execution_workspaces : "derived_from"
    environments |o--o{ environment_leases : ""
    execution_workspaces |o--o{ environment_leases : ""
    heartbeat_runs |o--o{ environment_leases : ""
    execution_workspaces |o--o{ workspace_operations : ""
    heartbeat_runs |o--o{ workspace_operations : ""
    execution_workspaces |o--o{ workspace_runtime_services : ""
```

További táblák: `heartbeat_run_watchdog_decisions`, `execution_workspace_*` (repók, hozzáférés,
restore pointok, runtime leasek), `environment_custom_image_*`, `native_run_results`,
`provider_trace_records`, `runner_api_response_reservations`, `browser_use_*`, `smoke_*`,
`resource_lifecycle_events`.

### 4.5 Governance és költség

Három kontrollsík invariáns él itt (`AGENTS.md` 5.3): **jóváhagyási kapu**, **költségkeret
hard-stop** és **activity log**. Az `approvals` egy általános jóváhagyási kérelem
(`type` + `payload`), ami issue-khoz köthető. A `budget_policies` polimorf hatókörre
(`scope_type` + `scope_id`: cég / ágens / projekt) határoz meg ablakos limitet; túllépéskor
`budget_incidents` keletkezik, ami jóváhagyást kérhet. Minden LLM-hívás `cost_events` sort
termel (token, modell, cent), a pénzügyi könyvelés a `finance_events`-ben részletesebb.
Minden mutáló művelet az `activity_log`-ba kerül.

```mermaid
erDiagram
    approvals {
        uuid id PK
        uuid company_id FK
        text type "pl. hire_agent"
        text status "pending | approved | revision_requested ..."
        jsonb payload
        uuid requested_by_agent_id FK
        text decided_by_user_id
    }
    approval_comments {
        uuid id PK
        uuid approval_id FK
        uuid author_agent_id FK
        text body
    }
    issue_approvals {
        uuid issue_id PK, FK
        uuid approval_id PK, FK
    }
    budget_policies {
        uuid id PK
        uuid company_id FK
        text scope_type "company | agent | project"
        uuid scope_id
        text metric
        text window_kind
        int amount
        bool hard_stop_enabled
    }
    budget_incidents {
        uuid id PK
        uuid policy_id FK
        ts window_start
        text threshold_type "soft | hard"
        numeric amount_observed
        text status
        uuid approval_id FK
    }
    cost_events {
        uuid id PK
        uuid company_id FK
        uuid agent_id FK
        uuid issue_id FK
        uuid project_id FK
        uuid heartbeat_run_id FK
        uuid subscription_id FK
        text provider
        text model
        int input_tokens
        int output_tokens
        numeric cost_cents
        text idempotency_key UK
    }
    finance_events {
        uuid id PK
        uuid cost_event_id FK
        text event_kind
        text direction
        numeric amount_cents
        text currency
    }
    ai_subscriptions {
        uuid id PK
        uuid company_id FK
        text provider
        text account_key
        uuid merged_into_id FK
    }
    activity_log {
        uuid id PK
        uuid company_id FK
        text actor_type "user | agent | system | plugin"
        text actor_id
        text action "pl. goal.created"
        text entity_type
        text entity_id
        uuid run_id FK
        jsonb details
    }
    issues {
        uuid id PK
    }
    agents {
        uuid id PK
    }
    heartbeat_runs {
        uuid id PK
    }

    approvals ||--o{ approval_comments : ""
    issues ||--o{ issue_approvals : ""
    approvals ||--o{ issue_approvals : ""
    agents |o--o{ approvals : "requested_by"
    budget_policies ||--o{ budget_incidents : ""
    approvals |o--o{ budget_incidents : "approval_id"
    agents |o..o{ budget_policies : "scope_type=agent"
    agents |o--o{ cost_events : ""
    issues |o--o{ cost_events : ""
    heartbeat_runs |o--o{ cost_events : ""
    ai_subscriptions |o--o{ cost_events : ""
    cost_events |o--o{ finance_events : ""
    heartbeat_runs |o--o{ activity_log : "run_id"
    agents |o--o{ activity_log : "agent_id"
```

További táblák: `accounting.ts` (számlák, budget reservation, run usage receiptek),
`ai_subscription_prices`, `ai_subscription_connections`.

### 4.6 Dokumentumok és tudás

A `documents` cégszintű, verziózott markdown dokumentum (a legfrissebb tartalom
denormalizálva a `latest_body`-ban, a története a `document_revisions`-ben). A dokumentum
kulcs alapján csatolódik issue-hoz (`issue_documents`, pl. `plan`) vagy routine-hoz
(`routine_documents`); egy dokumentum pontosan egy tulajdonoshoz tartozik (`document_id` UK
a kapcsolótáblán). Szövegrészletekhez annotációs szálak és kommentek kapcsolódnak. A
bináris fájlok az `assets` táblában (objektumtár kulcs + sha256) vannak.

```mermaid
erDiagram
    documents {
        uuid id PK
        uuid company_id FK
        text title
        text format "markdown"
        text latest_body
        uuid latest_revision_id "logikai, FK nélkül"
        int latest_revision_number
        uuid locked_by_agent_id FK
    }
    document_revisions {
        uuid id PK
        uuid document_id FK
        int revision_number UK "document_id+revision_number"
        text body
        text change_summary
        uuid created_by_run_id FK
    }
    issue_documents {
        uuid id PK
        uuid issue_id FK
        uuid document_id FK, UK
        text key "pl. plan"
    }
    routine_documents {
        uuid id PK
        uuid routine_id FK
        uuid document_id FK, UK
        text key
    }
    document_annotation_threads {
        uuid id PK
        uuid document_id FK
        uuid issue_id FK
        uuid current_revision_id FK
        text selected_text
        text status
    }
    document_annotation_comments {
        uuid id PK
        uuid thread_id FK
        uuid author_agent_id FK
        text author_user_id
        text body
    }
    assets {
        uuid id PK
        uuid company_id FK
        text provider
        text object_key UK
        text content_type
        bigint byte_size
        text sha256
    }
    issue_attachments {
        uuid id PK
        uuid asset_id FK, UK
    }
    issues {
        uuid id PK
    }
    routines {
        uuid id PK
    }

    documents ||--o{ document_revisions : ""
    document_revisions |o..o| documents : "latest_revision_id"
    issues ||--o{ issue_documents : ""
    documents ||--o| issue_documents : ""
    routines ||--o{ routine_documents : ""
    documents ||--o| routine_documents : ""
    documents ||--o{ document_annotation_threads : ""
    document_revisions |o--o{ document_annotation_threads : "current_revision"
    issues |o--o{ document_annotation_threads : ""
    document_annotation_threads ||--o{ document_annotation_comments : ""
    assets ||--o| issue_attachments : ""
    issues ||--o{ issue_attachments : ""
```

További táblák: `document_annotation_anchor_snapshots`, `document_memberships`,
`company_logos`, `summary_slots`, `status_cards` / `status_card_updates`.

### 4.7 Automatizálás és döntések

**Routine-ok.** Egy `routines` sor ismétlődő munkát ír le (kinek, melyik projektben,
milyen konkurencia- és pótlási szabállyal). Indítói a `routine_triggers` (schedule/cron, webhook, api –
a webhook titokkal aláírva), minden indítás egy `routine_runs` sor, ami issue-t hoz létre
(`linked_issue_id`). A routine definíciója verziózott (`routine_revisions`).

```mermaid
erDiagram
    routines {
        uuid id PK
        uuid company_id FK
        uuid project_id FK
        uuid folder_id FK
        uuid goal_id FK
        uuid parent_issue_id FK
        uuid assignee_agent_id FK
        text title
        text status
        text concurrency_policy "coalesce_if_active ..."
        text catch_up_policy "skip_missed ..."
    }
    routine_triggers {
        uuid id PK
        uuid routine_id FK
        text kind "schedule | webhook | api"
        text cron_expression
        text timezone
        ts next_run_at
        text public_id UK
        uuid secret_id FK
    }
    routine_revisions {
        uuid id PK
        uuid routine_id FK
        int revision_number UK
        jsonb snapshot
    }
    routine_runs {
        uuid id PK
        uuid routine_id FK
        uuid trigger_id FK
        uuid routine_revision_id FK
        text status
        ts triggered_at
        uuid linked_issue_id FK
    }
    folders {
        uuid id PK
        text kind
        uuid parent_id FK
        text slug
    }
    company_secrets {
        uuid id PK
    }
    issues {
        uuid id PK
    }
    agents {
        uuid id PK
    }

    folders |o--o{ folders : "parent_id"
    folders |o--o{ routines : ""
    agents |o--o{ routines : "assignee"
    issues |o--o{ routines : "parent_issue_id"
    routines ||--o{ routine_triggers : ""
    company_secrets |o--o{ routine_triggers : "webhook aláírás"
    routines ||--o{ routine_revisions : ""
    routines ||--o{ routine_runs : ""
    routine_triggers |o--o{ routine_runs : ""
    routine_revisions |o--o{ routine_runs : ""
    issues |o--o{ routine_runs : "linked_issue_id"
```

**Döntések és státusz-arbitráció.** Ágens által kért emberi döntés: `decisions` (opciók,
lejárat, aláírt specifikáció), csomagolva `decision_bundles`-be, több cél-issue-val. Külön
lánc dönti el, hogy egy natív futás után az issue státusza változhat-e:
`completion_contracts` (mit jelent a „kész”) → `native_run_results` (a futás eredménye) →
`work_assessments` (értékelés) → `status_decisions` (státuszátmenet) →
`status_decision_effects` (kiszállítandó mellékhatások). Ezek a táblák tenant-biztos
összetett FK-kkal (`company_id + issue_id + …`) kapcsolódnak.

```mermaid
erDiagram
    decision_bundles {
        uuid id PK
        uuid origin_agent_id FK
        uuid origin_issue_id FK
        uuid origin_run_id FK
        text title
    }
    decisions {
        uuid id PK
        uuid bundle_id FK
        uuid origin_agent_id FK
        uuid origin_issue_id FK
        uuid origin_run_id FK
        text title
        jsonb options
        text status
        text chosen_option_id
        ts expires_at
    }
    decision_target_issues {
        uuid decision_id PK, FK
        uuid issue_id PK, FK
    }
    decision_effect_executions {
        uuid id PK
        uuid decision_id FK
        uuid target_issue_id FK
        text effect_type
        uuid activity_log_id FK
    }
    completion_contracts {
        uuid id PK
        uuid issue_id FK
        int revision
        text risk
        jsonb contract_json
    }
    native_run_results {
        uuid id PK
        uuid issue_id FK
        uuid run_id FK
        uuid completion_contract_id FK
        jsonb result_json
    }
    work_assessments {
        uuid id PK
        uuid issue_id FK
        uuid run_id FK
        uuid contract_id FK
        uuid result_id FK
        text prior_issue_status
    }
    status_decisions {
        uuid id PK
        uuid issue_id FK
        uuid assessment_id FK, UK
        text from_status
        text to_status
        text reason_code
        text application_state
    }
    status_decision_effects {
        uuid id PK
        uuid decision_id FK
        text effect_kind
        text delivery_state
    }
    issues {
        uuid id PK
    }
    heartbeat_runs {
        uuid id PK
    }

    decision_bundles |o--o{ decisions : ""
    issues ||--o{ decisions : "origin_issue_id"
    heartbeat_runs ||--o{ decisions : "origin_run_id"
    decisions ||--o{ decision_target_issues : ""
    issues ||--o{ decision_target_issues : ""
    decisions ||--o{ decision_effect_executions : ""
    issues ||--o{ completion_contracts : ""
    completion_contracts ||--o{ native_run_results : ""
    heartbeat_runs ||--o{ native_run_results : ""
    native_run_results ||--o{ work_assessments : ""
    completion_contracts ||--o{ work_assessments : ""
    work_assessments ||--o| status_decisions : ""
    status_decisions ||--o{ status_decision_effects : ""
```

További táblák: `decision_queues` / `decision_queue_items` / `decision_triage*`,
`company_decision_models`, `decision_invocations`, `decision_training_examples`,
`native_run_finalizations`, `fast_responses.ts`, `prompt_injection_prevention.ts`.

### 4.8 Integrációk és bővíthetőség

**Pluginek és eszköz-kapcsolatok.** A `plugins` **instance-szintű** telepítés; cégenként a
`plugin_config` konfigurálja. Az ágensek külső eszközökhöz (MCP, OAuth app, API) a
`tool_applications` → `tool_connections` → `connection_grants` láncon jutnak (alkalmazás →
konkrét kapcsolat hitelesítéssel → kinek szabad használni). Az MCP gatewayek egy
`tool_profiles` szerint szűrik az elérhető eszközöket; minden hívás `tool_invocations` sor.

```mermaid
erDiagram
    plugins {
        uuid id PK "instance-szintű"
        text plugin_key UK
        text package_name
        text version
        text status
        jsonb manifest_json
    }
    plugin_config {
        uuid id PK
        uuid plugin_id FK
        uuid company_id FK
        jsonb config_json
    }
    plugin_state {
        uuid id PK
        uuid plugin_id FK
        text scope_kind
        text namespace
        text state_key
        jsonb value_json
    }
    plugin_jobs {
        uuid id PK
        uuid plugin_id FK
        text job_key
        text schedule
    }
    tool_applications {
        uuid id PK
        uuid company_id FK
        uuid plugin_id FK
        text name UK
        text type
    }
    tool_connections {
        uuid id PK
        uuid company_id FK
        uuid application_id FK
        text transport
        text auth_kind
        text status
        text health_status
    }
    connection_grants {
        uuid id PK
        uuid connection_id FK "company_id+connection_id"
        text kind
        uuid subject_agent_id FK
        text subject_user_id
        text status
    }
    tool_profiles {
        uuid id PK
        uuid company_id FK
        text profile_key UK
        text default_action
    }
    tool_mcp_gateways {
        uuid id PK
        uuid profile_id FK
        text gateway_public_id UK
        text slug
        uuid agent_id FK
        uuid project_id FK
    }
    tool_invocations {
        uuid id PK
        uuid gateway_id FK
        uuid connection_id FK
        uuid agent_id FK
        uuid run_id FK
        text tool_name
        text policy_decision
    }
    agents {
        uuid id PK
    }

    plugins ||--o{ plugin_config : "cégenként"
    plugins ||--o{ plugin_state : ""
    plugins ||--o{ plugin_jobs : ""
    plugins |o--o{ tool_applications : ""
    tool_applications ||--o{ tool_connections : ""
    tool_connections ||--o{ connection_grants : ""
    agents |o--o{ connection_grants : "subject_agent_id"
    tool_profiles ||--o{ tool_mcp_gateways : ""
    tool_mcp_gateways |o--o{ tool_invocations : ""
    tool_connections |o--o{ tool_invocations : ""
    agents |o--o{ tool_invocations : ""
```

**Titkok, skillek, chat csatornák.** A `company_secrets` verziózott titkokat tárol
(`company_secret_versions`), amelyeket a `company_secret_bindings` köt polimorf célpontokhoz
(`target_type` + `target_id` + `config_path`, pl. egy ágens adapter-konfigjának egy mezője).
A `company_skills` a cég saját skill-katalógusa, verziókkal. A chat csatornák
(`chat_endpoints`: Slack, Teams, Telegram, e-mail, hang) egy-egy ágenshez rendelt botok;
a beszélgetések (`chat_conversations`) mindig egy issue-hoz kötődnek.

```mermaid
erDiagram
    company_secrets {
        uuid id PK
        uuid company_id FK
        text scope
        text key UK
        text provider
        int latest_version
        uuid owner_plugin_id FK
    }
    company_secret_versions {
        uuid id PK
        uuid secret_id FK
        int version UK "secret_id+version"
        text value_sha256
        text status
    }
    company_secret_bindings {
        uuid id PK
        uuid secret_id FK
        text target_type
        text target_id
        text config_path
    }
    company_skills {
        uuid id PK
        uuid company_id FK
        text key UK
        text name
        text trust_level
        uuid current_version_id FK
        uuid forked_from_skill_id FK
    }
    company_skill_versions {
        uuid id PK
        uuid company_skill_id FK
        int revision_number UK
        jsonb file_inventory
    }
    chat_endpoints {
        uuid id PK
        uuid company_id FK
        text provider "slack | teams | telegram ..."
        uuid connection_id FK
        uuid assigned_agent_id FK
        text status
    }
    chat_conversations {
        uuid id PK
        uuid endpoint_id FK
        uuid issue_id FK
        text external_conversation_id
        text state
    }
    plugins {
        uuid id PK
    }
    agents {
        uuid id PK
    }
    issues {
        uuid id PK
    }
    tool_connections {
        uuid id PK
    }

    plugins |o--o{ company_secrets : "owner_plugin_id"
    company_secrets ||--o{ company_secret_versions : ""
    company_secrets ||--o{ company_secret_bindings : ""
    agents |o..o{ company_secret_bindings : "target_type=agent"
    company_skills ||--o{ company_skill_versions : ""
    company_skill_versions |o--o| company_skills : "current_version_id"
    company_skills |o--o{ company_skills : "forked_from"
    tool_connections ||--o{ chat_endpoints : ""
    agents ||--o{ chat_endpoints : "assigned_agent_id"
    chat_endpoints ||--o{ chat_conversations : ""
    issues ||--o{ chat_conversations : ""
```

További táblák: a `plugin_*` és `tool_*` család többi tagja, `connection_*`,
`ai_connection_*`, `public_mcp.ts` (`mcp_oauth_*`, `mcp_event_*`), `external_objects`,
`company_secret_proposals`, `company_secret_provider_configs`, `user_secret_*`,
`secret_access_events`, `company_skill_*`, `chat_*`, `email.ts`, `company_observability`.

## 5. Rétegek és adatfolyam (Clean Architecture nézet)

A függőségek iránya kifelé mutat a sémától: a `packages/db` nem ismeri a szervert, a
`packages/shared` nem ismeri az adatbázist, a route nem ír közvetlenül SQL-t.

```mermaid
flowchart LR
    subgraph infra["Infrastruktúra"]
        DB[("PostgreSQL / PGlite")]
        SCHEMA["packages/db<br/>Drizzle séma, migrációk,<br/>createDb() → Db"]
    end
    subgraph contract["Kontraktus"]
        SHARED["packages/shared<br/>típusok, konstansok,<br/>zod validátorok, API path-ok"]
    end
    subgraph app["Alkalmazás / domain"]
        SVC["server/src/services<br/>pl. goalService(db),<br/>logActivity(db, …)"]
    end
    subgraph adapters["Interface adapterek"]
        ROUTE["server/src/routes<br/>Express router, validate(),<br/>assertCompanyAccess()"]
        UI["ui/<br/>React + API kliens"]
    end
    SCHEMA --> DB
    SVC --> SCHEMA
    SVC --> SHARED
    ROUTE --> SVC
    ROUTE --> SHARED
    UI -->|HTTP /api| ROUTE
    UI --> SHARED
```

Példa egy útvonalra (`server/src/routes/goals.ts` → `server/src/services/goals.ts`):

1. `POST /api/companies/:companyId/goals` – a route `validate(createGoalSchema)`-val
   (`@paperclipai/shared`) ellenőrzi a bemenetet.
2. `assertCompanyAccess(req, companyId)` – a tenant-határ a route rétegben érvényesül.
3. `goalService(db).create(...)` – a domain-logika a service-ben, Drizzle lekérdezéssel
   (`db.select().from(goals).where(eq(goals.companyId, …))`).
4. `logActivity(db, { action: "goal.created", entityType: "goal", … })` – a mutáció
   `activity_log` sort kap.

SOLID-szempontból: a service-ek konstruktor-injekcióval kapják a `Db`-t (DIP, tesztelhető),
egy service egy aggregátumért felel (SRP), a kontraktus (`shared`) a rétegek közötti stabil
interfész (ISP).

### Egy heartbeat útja az adatbázisban

```mermaid
flowchart TD
    W["agent_wakeup_requests<br/>(timer, komment, assignment)"] --> R["heartbeat_runs<br/>status = queued → running"]
    R --> C["issues.checkout_run_id<br/>atomikus checkout"]
    R --> E["heartbeat_run_events<br/>run log"]
    R --> X["execution_workspaces /<br/>environment_leases"]
    R --> K["cost_events → finance_events"]
    K --> B{"budget_policies<br/>túllépés?"}
    B -->|igen| I["budget_incidents → approvals<br/>agents.status = paused"]
    R --> M["issue_comments / issue_work_products"]
    M --> A["activity_log"]
    R --> S["native_run_results → work_assessments<br/>→ status_decisions → issues.status"]
```

## 6. Megfigyelések (tanulási pontok)

1. **Két gravitációs központ.** A `companies` (tenant) és az `agents` (aktor) szinte minden
   táblához kapcsolódik; az `issues` és a `heartbeat_runs` a munka- és végrehajtási tengely.
   Ha egy új táblát értelmezel, először azt nézd meg, ezek közül melyikhez köt.
2. **Az emberi felhasználó „puha” hivatkozás.** A `*_user_id` oszlopok többnyire `text`
   FK nélkül – a felhasználói azonosság a Better Auth-ban él, a domain csak hivatkozik rá.
3. **Polimorfizmus az adatbázisban.** `principal_type/principal_id`, `scope_type/scope_id`,
   `target_type/target_id`, `actor_type/actor_id`: rugalmas, de az integritást a service
   rétegnek kell biztosítania.
4. **Tenant-biztonság a sémában is.** Az összetett `(company_id, id)` FK-k a
   kódszintű `assertCompanyAccess` mellett második védelmi vonalat adnak.
5. **Denormalizált „legfrissebb” mezők.** `documents.latest_body`,
   `routines.latest_revision_id`, `company_skills.current_version_id`: olvasási
   teljesítményért a fejrekord tárolja az aktuális állapotot, a történet revíziós táblában van.
6. **Sok állapot `jsonb`-ben.** `adapter_config`, `execution_policy`, `execution_state`,
   `payload`: a struktúrát a `packages/shared` típusai írják le, nem az adatbázis.

## 7. Hogyan tartsd naprakészen

- Új tábla: `packages/db/src/schema/*.ts` + export a `packages/db/src/schema/index.ts`-ben,
  majd `pnpm db:generate` (lásd `doc/DATABASE.md`, `AGENTS.md` 6.).
- A diagramokba csak akkor vegyél fel új entitást, ha a működést hordozza; egyébként elég a
  lenti indexben a megfelelő domainhez sorolni.
- Üzemeltetési részletek (PGlite, Postgres, migrációk, backup): `doc/DATABASE.md`.

## 8. Teljes táblaindex domainenként

Mind a 173 sémafájl (és bennük a 301 tábla) besorolva. A `chat_telegram_draft_ids.ts`
tábla helyett egy Postgres sequence-et definiál.

### 1. Tenant, identitás, hozzáférés

| Sémafájl | Táblák |
|---|---|
| `agent_api_keys.ts` | `agent_api_keys` |
| `agent_identity_keys.ts` | `agent_identity_keys` |
| `announcement_dismissals.ts` | `announcement_publications`, `announcement_dismissals` |
| `auth.ts` | `user`, `session`, `account`, `verification` |
| `board_api_keys.ts` | `board_api_keys` |
| `cli_auth_challenges.ts` | `cli_auth_challenges` |
| `companies.ts` | `companies` |
| `company_logos.ts` | `company_logos` |
| `company_memberships.ts` | `company_memberships` |
| `company_onboarding_seeds.ts` | `company_onboarding_seeds` |
| `company_transfer_runs.ts` | `company_transfer_runs` |
| `company_user_sidebar_preferences.ts` | `company_user_sidebar_preferences` |
| `document_memberships.ts` | `document_memberships` |
| `inbox_dismissals.ts` | `inbox_dismissals` |
| `instance_settings.ts` | `instance_settings` |
| `instance_user_roles.ts` | `instance_user_roles` |
| `invites.ts` | `invites` |
| `issue_access_grants.ts` | `issue_access_grants` |
| `join_requests.ts` | `join_requests` |
| `principal_permission_grants.ts` | `principal_permission_grants` |
| `project_access_members.ts` | `project_access_members` |
| `project_memberships.ts` | `project_memberships` |
| `run_identity_contexts.ts` | `run_identity_contexts` |
| `user_company_preferences.ts` | `user_company_preferences` |
| `user_inbox_agent_policies.ts` | `user_inbox_agent_policies` |
| `user_sidebar_preferences.ts` | `user_sidebar_preferences` |

### 2. Ágensek

| Sémafájl | Táblák |
|---|---|
| `adapter_auth_sessions.ts` | `adapter_auth_sessions` |
| `agent_commentary.ts` | `agent_commentary` |
| `agent_config_revisions.ts` | `agent_config_revisions` |
| `agent_instruction_revisions.ts` | `agent_instruction_revisions`, `agent_instruction_heads`, `agent_instruction_working_copies` |
| `agent_memberships.ts` | `agent_memberships` |
| `agent_runtime_state.ts` | `agent_runtime_state` |
| `agent_task_sessions.ts` | `agent_task_sessions`, `agent_session_goal_actions` |
| `agent_wakeup_requests.ts` | `agent_wakeup_requests` |
| `agents.ts` | `agents` |
| `built_in_managed_resources.ts` | `built_in_managed_resources` |
| `dot_runner.ts` | `dot_agent_bindings`, `dot_runner_assignments`, `dot_runner_operations`, `dot_mailbox_items` |
| `grok_bot.ts` | `grok_bot_routines`, `grok_bot_dispatches` |
| `managed_agent_profiles.ts` | `managed_agent_profiles` |
| `muse_runner.ts` | `muse_agent_bindings`, `muse_credentials`, `muse_runner_assignments`, `muse_runner_operations`, `muse_mailbox_items`, `muse_input_deliveries`, `external_agent_holds`, `muse_idle_receipts`, `muse_receiver_contact_buckets` |
| `remote_agent_profiles.ts` | `remote_agent_profiles` |

### 3. Munkaszervezés

| Sémafájl | Táblák |
|---|---|
| `cases.ts` | `cases`, `case_issue_links`, `case_events`, `case_documents`, `case_labels`, `case_attachments`, `case_access_sources`, `case_handoffs` |
| `feedback_exports.ts` | `feedback_exports` |
| `feedback_votes.ts` | `feedback_votes` |
| `folders.ts` | `folders` |
| `goals.ts` | `goals` |
| `issue_approvals.ts` | `issue_approvals` |
| `issue_attachments.ts` | `issue_attachments` |
| `issue_comments.ts` | `issue_comments` |
| `issue_create_idempotency_keys.ts` | `issue_create_idempotency_keys` |
| `issue_documents.ts` | `issue_documents` |
| `issue_execution_decisions.ts` | `issue_execution_decisions` |
| `issue_inbox_archives.ts` | `issue_inbox_archives` |
| `issue_labels.ts` | `issue_labels` |
| `issue_plan_decompositions.ts` | `issue_plan_decompositions` |
| `issue_question_response_deliveries.ts` | `issue_question_response_deliveries` |
| `issue_read_states.ts` | `issue_read_states` |
| `issue_recovery_actions.ts` | `issue_recovery_actions` |
| `issue_reference_mentions.ts` | `issue_reference_mentions` |
| `issue_relations.ts` | `issue_relations` |
| `issue_thread_interactions.ts` | `issue_thread_interactions` |
| `issue_tree_hold_members.ts` | `issue_tree_hold_members` |
| `issue_tree_holds.ts` | `issue_tree_holds` |
| `issue_watchdogs.ts` | `issue_watchdogs` |
| `issue_work_products.ts` | `issue_work_products` |
| `issues.ts` | `issues` |
| `labels.ts` | `labels` |
| `pipeline_case_events.ts` | `pipeline_case_events` |
| `pipeline_cases.ts` | `pipeline_cases`, `pipeline_case_issue_links`, `pipeline_case_blockers`, `pipeline_documents`, `pipeline_case_documents`, `pipeline_automation_executions` |
| `pipelines.ts` | `pipelines`, `pipeline_stages`, `pipeline_transitions` |
| `project_goals.ts` | `project_goals` |
| `project_workspaces.ts` | `project_workspaces` |
| `projects.ts` | `projects` |
| `workflows.ts` | `workflows` |

### 4. Végrehajtás

| Sémafájl | Táblák |
|---|---|
| `browser_use.ts` | `browser_use_settings`, `browser_use_sessions`, `browser_use_runs`, `browser_use_browsers` |
| `environment_custom_image_setup_sessions.ts` | `environment_custom_image_setup_sessions` |
| `environment_custom_image_templates.ts` | `environment_custom_image_templates` |
| `environment_leases.ts` | `environment_leases` |
| `environments.ts` | `environments` |
| `execution_workspace_repositories.ts` | `execution_workspace_repositories` |
| `execution_workspace_repository_access.ts` | `execution_workspace_repository_access` |
| `execution_workspace_repository_roots.ts` | `execution_workspace_repository_roots` |
| `execution_workspace_restore_points.ts` | `execution_workspace_restore_points` |
| `execution_workspace_runtime_leases.ts` | `execution_workspace_runtime_leases` |
| `execution_workspaces.ts` | `execution_workspaces` |
| `heartbeat_run_events.ts` | `heartbeat_run_events` |
| `heartbeat_run_watchdog_decisions.ts` | `heartbeat_run_watchdog_decisions` |
| `heartbeat_runs.ts` | `heartbeat_runs` |
| `native_run_results.ts` | `native_run_results` |
| `provider_trace_records.ts` | `provider_trace_records` |
| `resource_lifecycle_events.ts` | `resource_lifecycle_events` |
| `runner_api_response_reservations.ts` | `runner_api_response_reservations` |
| `smoke_lab.ts` | `smoke_runs`, `smoke_run_steps` |
| `workspace_operations.ts` | `workspace_operations` |
| `workspace_runtime_services.ts` | `workspace_runtime_services` |

### 5. Governance és költség

| Sémafájl | Táblák |
|---|---|
| `accounting.ts` | `provider_billing_snapshots`, `run_usage_receipts`, `accounting_runtime_baselines`, `budget_reservations`, `billing_invoices`, `billing_invoice_lines`, `cost_adjustments` |
| `activity_log.ts` | `activity_log` |
| `ai_subscriptions.ts` | `ai_subscriptions`, `ai_subscription_prices`, `ai_subscription_connections` |
| `approval_comments.ts` | `approval_comments` |
| `approvals.ts` | `approvals` |
| `budget_incidents.ts` | `budget_incidents` |
| `budget_policies.ts` | `budget_policies` |
| `cost_events.ts` | `cost_events` |
| `finance_events.ts` | `finance_events` |

### 6. Dokumentumok és tudás

| Sémafájl | Táblák |
|---|---|
| `assets.ts` | `assets` |
| `document_annotation_anchor_snapshots.ts` | `document_annotation_anchor_snapshots` |
| `document_annotation_comments.ts` | `document_annotation_comments` |
| `document_annotation_threads.ts` | `document_annotation_threads` |
| `document_revisions.ts` | `document_revisions` |
| `documents.ts` | `documents` |
| `status_cards.ts` | `status_cards`, `status_card_updates` |
| `summary_slots.ts` | `summary_slots` |

### 7. Automatizálás és döntések

| Sémafájl | Táblák |
|---|---|
| `completion_contracts.ts` | `completion_contracts` |
| `decision_models.ts` | `company_decision_models`, `decision_invocations` |
| `decision_queues.ts` | `decision_queues`, `decision_queue_items`, `decision_triage`, `decision_triage_events`, `decision_retention`, `decision_archive_notification_outbox` |
| `decision_training_examples.ts` | `decision_training_examples` |
| `decisions.ts` | `decision_bundles`, `decisions`, `decision_target_issues`, `decision_effect_executions` |
| `fast_responses.ts` | `company_fast_responses`, `fast_response_requests` |
| `native_run_finalizations.ts` | `native_run_finalizations` |
| `prompt_injection_prevention.ts` | `company_prompt_injection_prevention`, `prompt_injection_invocations` |
| `routine_documents.ts` | `routine_documents` |
| `routines.ts` | `routines`, `routine_revisions`, `routine_triggers`, `routine_runs`, `routine_webhook_test_receipts` |
| `status_decision_effects.ts` | `status_decision_effects` |
| `status_decisions.ts` | `status_decisions` |
| `work_assessments.ts` | `work_assessments` |

### 8. Integrációk és bővíthetőség

| Sémafájl | Táblák |
|---|---|
| `ai_connection_defaults.ts` | `ai_connection_defaults` |
| `ai_connection_routing.ts` | `ai_connection_pools`, `ai_connection_router_cursors`, `ai_connection_task_pins` |
| `ai_provider_defaults.ts` | `ai_provider_defaults` |
| `chat_channels.ts` | `chat_endpoints`, `chat_endpoint_resources`, `chat_external_principals`, `chat_identity_links`, `chat_conversations`, `chat_deliveries`, `chat_publications`, `chat_message_links`, `chat_actions`, `chat_agent_routes`, `chat_endpoint_leases`, `chat_sdk_state` |
| `chat_completion_deliveries.ts` | `chat_task_handoffs`, `chat_completion_deliveries` |
| `chat_discord_command_owners.ts` | `chat_discord_command_owners` |
| `chat_github.ts` | `chat_github_configurations`, `chat_github_registrations`, `chat_github_reviews` |
| `chat_slack_manager_grants.ts` | `chat_slack_manager_grants` |
| `chat_slack_registrations.ts` | `chat_slack_registrations` |
| `chat_teams_file_transfers.ts` | `chat_teams_file_transfers` |
| `chat_telegram_draft_ids.ts` | _(sequence: chat_telegram_draft_ids)_ |
| `chat_voice_callbacks.ts` | `chat_voice_callbacks` |
| `chat_voice_inbound.ts` | `chat_voice_phone_lines`, `chat_voice_inbound_calls` |
| `chat_voice_reports.ts` | `chat_voice_reports` |
| `chat_voice_sessions.ts` | `chat_voice_sessions`, `chat_voice_tool_calls`, `chat_voice_replies` |
| `company_observability.ts` | `company_observability` |
| `company_secret_bindings.ts` | `company_secret_bindings` |
| `company_secret_proposals.ts` | `company_secret_proposals` |
| `company_secret_provider_configs.ts` | `company_secret_provider_configs` |
| `company_secret_versions.ts` | `company_secret_versions` |
| `company_secrets.ts` | `company_secrets` |
| `company_skill_policies.ts` | `company_skill_policies` |
| `company_skill_sources.ts` | `company_skill_sources`, `project_skill_sources`, `company_skill_source_entries`, `company_skill_import_budgets` |
| `company_skills.ts` | `company_skills`, `company_skill_repository_snapshots`, `company_skill_versions`, `company_skill_stars`, `company_skill_comments`, `company_skill_test_inputs`, `company_skill_test_run_templates`, `company_skill_test_runs` |
| `connection_event_deliveries.ts` | `connection_event_deliveries` |
| `connection_intent_deliveries.ts` | `connection_intent_deliveries` |
| `email.ts` | `email_endpoints`, `email_messages`, `email_sends` |
| `external_object_mentions.ts` | `external_object_mentions` |
| `external_objects.ts` | `external_objects` |
| `plugin_company_settings.ts` | `plugin_company_settings` |
| `plugin_config.ts` | `plugin_config` |
| `plugin_database.ts` | `plugin_database_namespaces`, `plugin_migrations` |
| `plugin_entities.ts` | `plugin_entities` |
| `plugin_jobs.ts` | `plugin_jobs`, `plugin_job_runs` |
| `plugin_lifecycle_acknowledgments.ts` | `plugin_lifecycle_acknowledgments` |
| `plugin_logs.ts` | `plugin_logs` |
| `plugin_managed_resources.ts` | `plugin_managed_resources` |
| `plugin_state.ts` | `plugin_state` |
| `plugin_webhooks.ts` | `plugin_webhook_deliveries` |
| `plugins.ts` | `plugins` |
| `public_mcp.ts` | `mcp_oauth_clients`, `mcp_oauth_metadata_admissions`, `mcp_oauth_grants`, `mcp_oauth_requests`, `mcp_oauth_tokens`, `mcp_oauth_device_requests`, `mcp_mutation_receipts`, `mcp_event_subscriptions`, `mcp_event_admissions`, `mcp_event_deliveries`, `mcp_attachment_uploads`, `mcp_file_tickets` |
| `secret_access_events.ts` | `secret_access_events` |
| `tool_access.ts` | `tool_applications`, `tool_connections`, `connection_grants`, `connection_grant_members`, `connection_grant_delegations`, `tool_connection_installs`, `tool_oauth_states`, `tool_catalog_entries`, `tool_profiles`, `tool_profile_entries`, `tool_profile_bindings`, `tool_mcp_gateways`, `tool_mcp_gateway_tokens`, `tool_policies`, `tool_runtime_slots`, `tool_stdio_command_templates`, `tool_gateway_sessions`, `tool_gateway_rate_limit_counters`, `tool_invocations`, `tool_action_requests`, `tool_call_events`, `connection_token_issuances`, `tool_rate_limit_counters`, `tool_runtime_metric_counters`, `tool_access_audit_events` |
| `tool_action_deliveries.ts` | `tool_action_deliveries` |
| `tool_connection_app_snapshots.ts` | `tool_connection_app_snapshots` |
| `tool_connection_app_syncs.ts` | `tool_connection_app_syncs` |
| `user_secret_declarations.ts` | `user_secret_declarations` |
| `user_secret_definitions.ts` | `user_secret_definitions` |
