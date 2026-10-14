# Remote Instance Integration for ServiceNow · v1.0

Bidirectional **Incident integration between two ServiceNow instances** (often called eBonding), built as a scoped application with IntegrationHub, Workflow Studio (Flow Designer) and the Import Set API.

A customer instance sends Incidents to a vendor (provider) instance; the vendor's progress, comments, resolution and attachments flow back automatically, with **no duplicate records, no update loops, and a full audit log**. Verified end to end between two Personal Developer Instances on the Australia release; all 13 test scenarios pass.

> The outbound Actions are named and shaped like the **ServiceNow Remote Instance Spoke** actions, so the app can switch to the spoke on instances with an Integration Hub subscription, without redesign.

---

## Table of contents

1. [Features](#features)
2. [Architecture](#architecture)
3. [What syncs, and what does not](#what-syncs-and-what-does-not)
4. [Design decisions](#design-decisions)
5. [Prerequisites](#prerequisites)
6. [What travels how](#what-travels-how)
7. [Installation](#installation)
8. [Verify](#verify)
9. [Test results](#test-results)
10. [Lessons learned during the build](#lessons-learned-during-the-build)
11. [Updating an installed instance](#updating-an-installed-instance)
12. [Troubleshooting](#troubleshooting)
13. [Security](#security)
14. [Application contents](#application-contents)
15. [Uninstall / rollback](#uninstall--rollback)
16. [Version history](#version-history)

---

## Features

- **Create and update with one call** – Import Set API with transform maps that **coalesce on Correlation ID**, so a retry or a repeated message never creates a duplicate.
- **Bidirectional** – details go Customer → Provider; state and resolution go Provider → Customer; comments and attachments go both ways.
- **Loop prevention** – every trigger listens only to the fields it owns and ignores changes written by the integration user.
- **Reusable, spoke-compatible Actions** – one Action for every record send, one for attachments; JSON built with `JSON.stringify`, calls through a Connection & Credential Alias with a retry policy, responses validated.
- **Centralized logging** – one Subflow writes every success and failure to **Remote Sync Log**; on failure it adds a work note with a clickable link to the log row, and a notification emails the admins.
- **Secure by default** – OAuth 2.0 **Client Credentials**, API access limited by auth scope, web-service-only integration user with least-privilege roles, no secrets in Git.

## Architecture

```text
 CUSTOMER instance                                     PROVIDER instance
 ─────────────────                                     ─────────────────
 Incident (assigned to Vendor Support)                 Incident (Customer Escalations)
   │ Flow:       Send Incident Details                   ▲
   │ BR+Subflow: Send Comment                            │ transform map: create or update
   │ Flow:       Send Attachment                         │ (coalesce on Correlation ID)
   ▼                                                     │
 Action: Create or Update Remote Record ── POST ──►  staging table Incident From Customer
         Using Import Set                    /api/now/import/…

 staging table Incident From Provider  ◄── POST ──  Action (same app)
   │ transform map: update only                          ▲ Flow:       Send Progress
   │ (coalesce on Correlation ID, inserts ignored)       │ BR+Subflow: Send Comment
   ▼                                                     │ Flow:       Send Attachment
 Incident                                              Incident

 Both instances: Subflow "Remote Sync: Write Log" → Remote Sync Log + failure email
```

The **same app** is installed on both instances. Four system properties tell each instance whether it is the Customer or the Provider, whom it calls, and where it posts.

## What syncs, and what does not

| Incident field | Synced | Direction | How |
| --- | --- | --- | --- |
| Short description, Description | Yes | Customer → Provider | Text (quotes, line breaks, non-English text preserved) |
| Caller | Yes | Customer → Provider | Caller's **email**, matched by email on the receiver |
| Impact, Urgency | Yes | Customer → Provider | Priority is recalculated on the receiver |
| Category, Subcategory | Yes | Customer → Provider | Unknown values ignored |
| Service, Service offering, Configuration item | Yes, by name | Customer → Provider | Matched by name; unmatched names are written to a work note, never created |
| State, Resolution code, Resolution notes | Yes | Provider → Customer | Sent together |
| Additional comments | Yes | Both | Each new comment once, never echoed back |
| Attachments | Yes | Both | Attachment API, never echoed back |
| Work notes, Watch list, Work notes list | **Never** | – | Internal to each company |
| Assignment group, Assigned to | **Never** | – | Each side routes its own work |
| Priority, Channel, Resolved by, Resolved, SLAs | No | – | Calculated or owned by each instance |
| Parent incident, Problem, Change request | No | – | Records exist only in one instance |

## Design decisions

| Decision | Why |
| --- | --- |
| **Import Set API**, not Table API | Coalesce gives create-or-update in one call; the receiver keeps control of validation and mapping. |
| **Correlation ID** as the link on both sides | Sys IDs differ per instance; each side stores the partner's sys_id and coalesces on it. |
| **Choice action = ignore** on reference and choice field maps | Partner data can never create users, CIs, services or choice values. |
| **OOTB first** | The ServiceNow Remote Instance Spoke was evaluated; it needs an Integration Hub subscription. The custom Actions use the same names and pattern, so swapping later is a one-step change. |
| **Business Rule + Subflow** for comments | Flow triggers cannot use journal fields; a Business Rule detects *Additional comments changes* declaratively and starts a reusable Subflow asynchronously via `sn_fd.FlowAPI`. |
| **One logging Subflow** | Success and failure logging, the failure work note and its link live in one place. |
| **OAuth 2.0 Client Credentials** | Machine-to-machine; no user password stored or sent; tokens are short-lived and renewed automatically. |
| **Alias in the app, connections in Global** | The Actions only know the alias name; each instance plugs in its own URL and credential, which never reach Git. |

---

## Prerequisites

Check every item on **both** instances.

| Requirement | How to check | Why |
| --- | --- | --- |
| Two ServiceNow instances on the **same family release** (built and tested on **Australia**) | *System Diagnostics → Stats* | Source-control import between releases can fail |
| Admin access | – | Setup only; the integration runs as a non-admin user |
| **Flow Designer - Designer** (`sn_flow_designer`) **29.2.x or later** | *All → Plugins* | Needed for the permanent fix of PRB2005619 |
| IntegrationHub **REST step** | Workflow Studio → new Action → a REST step can be added | Used by both Actions (available on PDIs) |
| **OAuth 2.0** plugin active | *System OAuth → Application Registry* exists | Instance-to-instance authentication |
| GitHub account and a Personal Access Token (`repo` scope) | – | To install the app from source control |

### Global prerequisite update set (Australia release)

On the Australia release, the *changes*, *changes to* and *changes from* operators are missing from Workflow Studio trigger conditions (**PRB2005619**). The Flows need them.

1. Download `Flow Designer – Enable "changes" Operators on Incident Fields (PRB2005619)` from this repository's **Releases** (v1.0 assets).
2. *System Update Sets → Retrieved Update Sets → Import Update Set from XML* → **Preview** → resolve collisions → **Commit**.

It adds `extended_operators=VALCHANGES;CHANGESFROM;CHANGESTO` to the dictionary entries of `short_description`, `description`, `impact`, `urgency`, `state`, `close_notes`, `cmdb_ci`, `business_service`, `service_offering` (Task) and `category`, `subcategory`, `close_code` (Incident). Existing attributes are kept; only condition-builder operators are added. These records are **Global**, so they cannot be part of the scoped app. Skip this once PRB2005619 is fixed on your instance.

## What travels how

| Part | Contents | Moved by |
| --- | --- | --- |
| **App** | Tables, transform maps, properties, alias, Actions, Flows, Subflows, Business Rule, notification, cross-scope privileges | GitHub (source control) |
| **Global prerequisites** | PRB2005619 dictionary attributes | Update set XML (release asset) |
| **Per-instance configuration** | Integration user, groups, OAuth clients, outbound OAuth profiles, credentials, connections, property values, role grants, form layout, index check | **By hand**, following this README |

OAuth clients, profiles, credentials and connections are never moved with update sets or Git: their secrets are encrypted with each instance's own key, and their URLs differ per instance.

---

## Installation

> **Application picker:** use **Global** for users, groups, OAuth, credentials and connections; use **Remote Instance Integration** only when editing the app's own records (for example the property values).

**Naming rule** for per-instance records: name each from the point of view of the instance it lives on. *Inbound* = who may call me; *Outbound* = whom I call.

| Record | On the Customer | On the Provider |
| --- | --- | --- |
| Inbound OAuth client | `Inbound from Provider (<provider>)` | `Inbound from Customer (<customer>)` |
| Outbound OAuth profile | `Outbound to Provider (<provider>)` | `Outbound to Customer (<customer>)` |
| OAuth 2.0 credential | `Outbound to Provider – Credential` | `Outbound to Customer – Credential` |
| HTTP(s) connection | `Outbound to Provider – Connection` | `Outbound to Customer – Connection` |

The outbound profile **Outbound to X** always gets the Client ID and secret of the **Inbound from …** client that lives on **X**.

### Step 1 – Prepare both instances (Global)

**1.1 Integration user** – *User Administration → Users → New*: User ID `integration.inbound`, **Web service access only** ✓, no password needed, roles `import_set_loader`, `import_transformer`, `itil`.

**1.2 Groups**

| Group | Instance | Members / roles |
| --- | --- | --- |
| Vendor Support | Customer | No members (assigning a ticket here sends it to the vendor) |
| Customer Escalations | Provider | Vendor agents with `itil` (queue for incoming tickets) |
| Integration Admins | Both | People who support the integration (log role added in Step 3/4) |

**1.3 Enable inbound Client Credentials** – *sys_properties.list*: `glide.oauth.inbound.client.credential.grant_type.enabled` = **true** (create as *true \| false* if missing).

**1.4 Inbound OAuth client** – *Machine Identity Console → Inbound integrations → New integration → **OAuth - Client credentials grant***:

| Field | Value |
| --- | --- |
| Name | per the naming table |
| Provider name | the partner, for example `ServiceNow Provider (<provider>)` |
| OAuth application user | `integration.inbound` |
| Auth scope | create `remote_instance_integration`, limited to **Import Set API** and **Attachment API** |
| Allow access only to APIs in selected scope | ✓ |

Copy the **Client ID** and **Client secret** for the partner's Step 3/4.

**1.5 Global prerequisite update set** – apply it (see Prerequisites).

### Step 2 – Install the app (both)

1. *Credentials → New → Basic Auth*: name `GitHub`, user = GitHub username, password = Personal Access Token.
2. **ServiceNow Studio → Import from source control**: repo URL, branch `main`, credential `GitHub`.
3. Confirm **Remote Instance Integration** (scope `x_1664823_remote_0`) contains the [application contents](#application-contents).

### Step 3 – Configure the Customer (Global)

**3.1 Outbound OAuth profile** – *System OAuth → Application Registry → New → Connect to a third party OAuth Provider - Outbound*:

| Field | Value |
| --- | --- |
| Name | `Outbound to Provider (<provider>)` |
| Client ID / Client Secret | from **Inbound from Customer (<customer>)** on the **Provider** |
| Default Grant type | **Client Credentials** |
| Token URL | `https://<provider>.service-now.com/oauth_token.do` |
| Accessible from | All application scopes |
| Other fields | empty / default |

**3.2 OAuth 2.0 credential** – *Credentials → New → OAuth 2.0 Credentials*: name `Outbound to Provider – Credential`, OAuth Entity Profile = the profile above, Credential alias **empty** (do not unlock), MID unchecked → Submit → **Get OAuth Token** (expect *completed successfully*).

**3.3 Connection** – open alias `x_1664823_remote_0.ServiceNowRemoteInstance` → **Connections → New → HTTP(s) Connection**: name `Outbound to Provider – Connection`, credential from 3.2, URL `https://<provider>.service-now.com`, Active ✓, MID unchecked. Create it in **Global**; only one connection may be active per alias.

**3.4 Properties** (picker: Remote Instance Integration)

| Property | Customer value |
| --- | --- |
| `x_1664823_remote_0.local_system_name` | `CUSTOMER-<customer>` |
| `x_1664823_remote_0.partner_system_name` | `PROVIDER-<provider>` |
| `x_1664823_remote_0.partner_staging_table` | `x_1664823_remote_0_incident_from_customer` |
| `x_1664823_remote_0.default_assignment_group` | *(empty)* |

**3.5 Form, roles, notification**
- Incident form → **Configure → Related Lists** → add **Remote Sync Log → Incident**.
- *Integration Admins* → role **Remote Sync Log** (`x_1664823_remote_0.remote_sync_log_user` or similar).
- `integration.inbound` → role for staging table **Incident From Provider** (without it the Provider's calls get **403**).
- Notification **Remote sync failed** → re-select **Integration Admins**.

### Step 4 – Configure the Provider (Global unless noted)

**4.1–4.3** – as 3.1–3.3, with: profile `Outbound to Customer (<customer>)` using the Client ID / secret of **Inbound from Provider (<provider>)** on the **Customer**, token URL `https://<customer>.service-now.com/oauth_token.do`, credential `Outbound to Customer – Credential`, connection `Outbound to Customer – Connection` with URL `https://<customer>.service-now.com`.

**4.4 Properties** (picker: **Remote Instance Integration**, not Global)

| Property | Provider value |
| --- | --- |
| `x_1664823_remote_0.local_system_name` | `PROVIDER-<provider>` |
| `x_1664823_remote_0.partner_system_name` | `CUSTOMER-<customer>` |
| `x_1664823_remote_0.partner_staging_table` | `x_1664823_remote_0_incident_from_provider` |
| `x_1664823_remote_0.default_assignment_group` | sys_id of *Customer Escalations* |

**4.5 Form, roles, notification, index**
- Related list, *Integration Admins* log role and notification group, as in 3.5.
- `integration.inbound` → role for staging table **Incident From Customer**.
- *System Definition → Tables → Incident → Database Indexes*: an index on `correlation_id` must exist; otherwise use **Index Coalesce Fields** on transform map *Incident From Customer*.

---

## Verify

**Smoke test (one Action call, on the Customer)** – Workflow Studio → Action **Create or Update Remote Record Using Import Set** → **Test** with a test Incident's sys_id and number, `short_description` `Smoke test`, `caller_email` that exists on both instances, impact/urgency `3`.

| Run | Expected |
| --- | --- |
| First | `status_code 201`, `import_status inserted`, a Provider number |
| Same inputs again | `import_status unchanged`, same number |
| Changed short description | `import_status updated`, same number |

**End to end** – assign a Customer Incident to **Vendor Support**: a Provider Incident appears in *Customer Escalations*, the Customer Incident gets a Correlation ID and the work note *Vendor ticket created: INC…*, and Remote Sync Log shows **Success**.

## Test results

All scenarios passed on v1.0 (Customer dev311022 ↔ Provider dev353640, Australia release).

| # | Scenario | Result |
| --- | --- | --- |
| 1 | Create: assign to Vendor Support → Provider Incident in Customer Escalations, linked both ways | ✅ |
| 2 | Caller matched by email | ✅ |
| 3 | Update: no new Incident, log *updated* | ✅ |
| 4 | Customer comment → Provider, once | ✅ |
| 5 | Vendor comment → Customer, once | ✅ |
| 6 | No echo of comments | ✅ |
| 7 | Work notes stay private | ✅ |
| 8 | Vendor state and resolution (code + notes) → Customer | ✅ |
| 9 | Incident not assigned to Vendor Support → nothing sent | ✅ |
| 10 | Quotes, line breaks and Urdu text preserved | ✅ |
| 11 | Attachment sent once in each direction, no echo | ✅ |
| 12 | Wrong client secret → Failed log, work note with link, email | ✅ |
| 13 | Secret restored → Success | ✅ |

## Lessons learned during the build

| Issue found | Resolution |
| --- | --- |
| *changes* operators missing in Flow trigger conditions (Australia, PRB2005619) | Permanent fix from the ServiceNow KB: Flow Designer 29.2.x + `extended_operators` dictionary attribute, shipped as a Global prerequisite update set |
| Loop guard placed inside an OR group of the trigger condition caused repeated sends | Rebuilt conditions as *scope* **and** *not changed by integration.inbound* **and** (*field changes* **or** …) |
| Same data sent twice produced *ignored: No field values changed* and false failure alerts | Treated as `import_status = unchanged` (success); only real rejections fail |
| Journal fields are not available in Flow trigger conditions | Business Rule with a declarative *Additional comments changes* condition starts a Subflow via `sn_fd.FlowAPI` |
| A connection accidentally created in the app scope reached Git and the Provider | Removed from the app; connections are created only in Global, per instance |
| Pull from repository reset the Provider's property values | Properties edited in the app scope, so a pull can stash and re-apply them |
| OAuth `access_denied` | Client ID copied from the wrong instance; fixed by the inbound/outbound naming rule |
| Action Status → Message pill greyed out in Error Handlers | Used *Top Level Catch → Error Status → Message*, which carries the error of whichever step failed |

## Updating an installed instance

1. Change the app **only on the development instance** (Customer), then **Commit Changes**. Before committing, check *All update sets*: runtime cross-scope privileges are often recorded in *Default*.
2. On the Provider: **ServiceNow Studio → Source control → Pull from repository** (classic Studio: *Apply Remote Changes*).
3. When the pull lists uncommitted changes (the Crypto Module Key Policy and the four property values), choose **Stash local changes** – never *Discard* – then **apply the stash** after the pull.
4. Re-check the Provider property values and run **Get OAuth Token** once.
5. **Never commit from the Provider.**

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| *Unable to load connection with alias ID* | No active connection on the alias on this instance | Step 3.3 / 4.3 |
| Get OAuth Token: `access_denied` | Client ID/secret from the wrong client or instance; client-credentials property not true on the partner; no OAuth application user | Re-copy from the partner's *Inbound from …* client; check Step 1.3 / 1.4 |
| HTTP 401 | Wrong or rotated client secret | Update the outbound profile, Get OAuth Token |
| HTTP 403 | Auth scope missing an API, or `integration.inbound` lacks the staging-table role | Step 1.4, 3.5 / 4.5 |
| HTTP 400 *Invalid table* | Wrong `partner_staging_table` | Step 3.4 / 4.4 |
| Payload contains only `u_sender_system` | Build-payload Script step input variables not mapped | Map each Script step input to its Action input pill; Save and Publish |
| Provider Incidents arrive with an empty Assignment group | Provider properties reset by a pull | Re-apply Step 4.4 in the app scope; stash on future pulls |
| Repeated sends / duplicate attachments | Loop guard not ANDed in a trigger condition | See Lessons learned |
| *changes* operator missing | PRB2005619 | Apply the Global prerequisite update set |

Every message, success or failure, is in **Remote Sync Log** (related list on the Incident).

## Security

| Control | Setting |
| --- | --- |
| Authentication | OAuth 2.0 Client Credentials; short-lived tokens; no user password stored or sent |
| API restriction | Auth scope limited to Import Set API and Attachment API |
| Integration user | Web service access only; `import_set_loader`, `import_transformer`, `itil`, one staging-table role |
| Secrets | Global-scope credentials behind a Connection & Credential Alias; never in the app, update sets or Git |
| Data shared | No work notes, watch lists or internal fields |
| Flows | Run as System user |

## Application contents

Scope `x_1664823_remote_0` – **Remote Instance Integration**

| Type | Name | Runs on |
| --- | --- | --- |
| Table (staging) | Incident From Customer | Provider |
| Table (staging) | Incident From Provider | Customer |
| Transform map | Incident From Customer → Incident (create or update) | Provider |
| Transform map | Incident From Provider → Incident (update only, inserts ignored) | Customer |
| Table | Remote Sync Log | Both |
| System properties | `local_system_name`, `partner_system_name`, `partner_staging_table`, `default_assignment_group` | Both (values per instance) |
| Connection & Credential Alias | `ServiceNowRemoteInstance` | Both (connection per instance) |
| Action | Create or Update Remote Record Using Import Set | Both |
| Action | Copy Attachment To Remote Instance | Both |
| Flow | Remote Sync: Send Incident Details | Customer |
| Flow | Remote Sync: Send Progress | Provider |
| Flow | Remote Sync: Send Attachment | Both |
| Business Rule + Subflow | Remote Sync: Send Comment | Both |
| Subflow | Remote Sync: Write Log | Both |
| Notification | Remote sync failed | Both |
| Cross-scope privileges | incident (write), properties, ScopedGlideElement, ImportLog.warn, ScriptableFlowRunner | Both |

## Uninstall / rollback

1. Deactivate the Flows and the Business Rule on both instances.
2. Uninstall the app (*System Applications → My Company Applications*).
3. Delete the Global records: OAuth clients and profiles, credentials, connections, `integration.inbound`, groups.
4. To remove the prerequisite: back out the update set, or delete the `extended_operators` entry from the listed dictionary entries.

## Version history

| Version | Notes |
| --- | --- |
| 1.0 | Bidirectional Incident sync (details, progress, resolution, comments, attachments); Import Set API with coalesce; OAuth Client Credentials; centralized logging and failure alerts; all 13 tests passed |

---

**Author:** <Raja Aqib> · ServiceNow Developer
