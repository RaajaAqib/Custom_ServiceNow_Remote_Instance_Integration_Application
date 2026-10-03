# Remote Instance Integration for ServiceNow

Bidirectional **Incident integration between two ServiceNow instances** (often called eBonding), built as a scoped application with IntegrationHub, Workflow Studio (Flow Designer) and the Import Set API.

A customer instance sends Incidents to a vendor (provider) instance; the vendor's progress, comments, resolution and attachments flow back automatically, with no duplicate records, no update loops, and a full audit log.

> The outbound Actions are named and shaped like the **ServiceNow Remote Instance Spoke** actions, so the app can switch to the spoke on instances that have an Integration Hub subscription, without redesign.

---

## Table of contents

1. [Features](#features)  
2. [Architecture](#architecture)  
3. [What syncs, and what does not](#what-syncs-and-what-does-not)  
4. [Prerequisites](#prerequisites)  
5. [Installation overview](#installation-overview)  
6. [Step 1 – Prepare both instances](#step-1--prepare-both-instances-global-scope)  
7. [Step 2 – Install the app on both instances](#step-2--install-the-app-on-both-instances)  
8. [Step 3 – Configure the Customer instance](#step-3--configure-the-customer-instance)  
9. [Step 4 – Configure the Provider instance](#step-4--configure-the-provider-instance)  
10. [Step 5 – Verify](#step-5--verify)  
11. [Test scenarios](#test-scenarios)  
12. [Troubleshooting](#troubleshooting)  
13. [Security](#security)  
14. [Application contents](#application-contents)  
15. [Uninstall / rollback](#uninstall--rollback)  
16. [Version history](#version-history)

---

## Features

- **Create and update with one call** – the Import Set API with a transform map that **coalesces on Correlation ID**, so a retry never creates a duplicate.  
- **Bidirectional** – details go Customer → Provider; state, resolution, comments and attachments go both ways as designed.  
- **Loop prevention** – every Flow triggers only on the fields it owns, and changes written by the integration user are never sent back.  
- **Reusable outbound Actions** – one Action for every record send, one for attachments; JSON is built with `JSON.stringify`, calls go through a Connection & Credential Alias with a retry policy, and responses are validated.  
- **Centralized logging** – one Subflow writes every success and failure to the **Remote Sync Log** table, adds a work note with a clickable link on failure, and a notification emails the admins.  
- **Secure by default** – OAuth 2.0 **Client Credentials**, API access limited by auth scope, web-service-only integration user with least-privilege roles, and no secrets in Git.  
- **Spoke-compatible** – same Action and alias names as the Remote Instance Spoke.

## Architecture

 CUSTOMER instance                                   PROVIDER instance

 ─────────────────                                   ─────────────────

 Incident (Vendor Support)                           Incident (Customer Escalations)

   │ Flow A: Send Incident Details                     ▲

   │ BR \+ Subflow: Send Comment         POST /api/now/import/\<incident\_from\_customer\>

   │ Flow D: Send Attachment  ───────────────────────► Staging table ─► Transform map

   ▼                                                    (coalesce on Correlation ID,

 Action: Create or Update Remote                         create or update)

 Record Using Import Set

                                                     Incident

 Staging table ◄─ Transform map  ◄─────────────────    │ Flow C: Send Progress

 (coalesce on Correlation ID,      POST /api/now/import│ BR \+ Subflow: Send Comment

  update only, inserts ignored)    /\<incident\_from\_provider\>  Flow D: Send Attachment

 ▼                                                     ▼

 Remote Sync Log  (both instances, via Subflow "Remote Sync: Write Log")

The **same app** is installed on both instances. Four system properties tell each instance whether it is the Customer or the Provider, whom it calls, and where it posts.

## What syncs, and what does not

| Incident field | Synced | Direction | How |
| :---- | :---- | :---- | :---- |
| Short description, Description | Yes | Customer → Provider | Text |
| Caller | Yes | Customer → Provider | Caller's **email**, matched by email |
| Impact, Urgency | Yes | Customer → Provider | Priority is recalculated on the receiver |
| Category, Subcategory | Yes | Customer → Provider | Unknown values are ignored |
| Service, Service offering, Configuration item | Yes, by name | Customer → Provider | Matched by name; names written to a work note on creation |
| State, Resolution code, Resolution notes | Yes | Provider → Customer | Sent together |
| Additional comments | Yes | Both | Newest entry only |
| Attachments | Yes | Both | Attachment API |
| Work notes, Watch list, Work notes list | **Never** | – | Internal to each company |
| Assignment group, Assigned to | **Never** | – | Each side routes its own work |
| Priority, Channel, Resolved by, Resolved, SLAs | No | – | Calculated or owned by each instance |
| Parent incident, Problem, Change request | No | – | Records exist only in one instance |

---

## Prerequisites

Check every item on **both** instances before installing.

### Platform

| Requirement | How to check | Why |
| :---- | :---- | :---- |
| Two ServiceNow instances (for example two PDIs) | – | One acts as Customer, one as Provider |
| Admin access on both | – | Setup only; the integration itself runs as a non-admin user |
| Same family release on both (developed and tested on **Australia**) | *System Diagnostics → Stats* or property `glide.buildtag` | Source-control import between different releases can fail |
| **Flow Designer \- Designer** plugin (`sn_flow_designer`) **29.2.x or later** | *All → Plugins → Flow Designer \- Designer* | Required for the permanent fix of the *changes* operator defect |
| IntegrationHub **REST step** available | Workflow Studio → new Action → a **REST** step can be added | Used by both outbound Actions (available on PDIs) |
| **OAuth 2.0** plugin active | *System OAuth → Application Registry* exists | Instance-to-instance authentication |
| GitHub account, repository access and a Personal Access Token (`repo` scope) | – | To import the app from source control |

### Global prerequisite update set (Australia release only)

On the Australia release, the *changes*, *changes to* and *changes from* operators are missing from Workflow Studio trigger conditions (**PRB2005619**). The app's Flows need them.

1. Download `Flow Designer – Enable "changes" Operators on Incident Fields (PRB2005619)` from this repository's **Releases** (v1.0 assets).  
2. *System Update Sets → Retrieved Update Sets → Import Update Set from XML* → upload → **Preview** → resolve any collisions → **Commit**.

What it does: adds `extended_operators=VALCHANGES;CHANGESFROM;CHANGESTO` to the dictionary entries of `short_description`, `description`, `impact`, `urgency`, `state`, `close_notes`, `cmdb_ci`, `business_service`, `service_offering` (Task) and `category`, `subcategory`, `close_code` (Incident). Existing attributes are kept. It only adds condition-builder operators; no data or behaviour changes.

> These records belong to the **Global** scope, so they cannot be part of the scoped app. Skip this step once PRB2005619 is fixed on your instance.

### Values you will need

| Value | Customer instance | Provider instance |
| :---- | :---- | :---- |
| Instance URL | `https://<customer>.service-now.com` | `https://<provider>.service-now.com` |
| System name (property) | `CUSTOMER-<customer>` | `PROVIDER-<provider>` |
| OAuth client for the partner (created in Step 1\) | *Provider Instance Client* | *Customer Instance Client* |

---

## Installation overview

| Step | Customer | Provider | Scope |
| :---- | :---- | :---- | :---- |
| 1\. Prepare: users, groups, OAuth endpoints, properties | ✔ | ✔ | Global |
| 2\. Install the app from GitHub | ✔ | ✔ | App |
| 3\. Configure Customer: connection, properties, form, roles | ✔ |  | Global |
| 4\. Configure Provider: connection, properties, form, roles, index |  | ✔ | Global |
| 5\. Verify | ✔ | ✔ | – |

> Always do Step 1, 3 and 4 with the **Application picker set to Global**, so users, credentials and connections are never captured in the app or committed to Git.

---

## Step 1 – Prepare both instances (Global scope)

### 1.1 Integration user (both)

*User Administration → Users → New*

| Field | Value |
| :---- | :---- |
| User ID | `integration.inbound` |
| First / Last name | Integration / Inbound |
| Web service access only | ✔ |
| Password | Not needed (OAuth Client Credentials) |
| Roles | `import_set_loader`, `import_transformer`, `itil` |

The partner instance always acts as this user. The Flows ignore changes made by it, which prevents update loops.

### 1.2 Groups

| Group | Instance | Members | Roles |
| :---- | :---- | :---- | :---- |
| Vendor Support | Customer | None needed | None |
| Customer Escalations | Provider | Vendor agents (`itil`) | `itil` |
| Integration Admins | Both | People who support the integration | Remote Sync Log role (after Step 2\) |

On the Provider, copy the **sys\_id** of *Customer Escalations*; it is needed in Step 4\.

### 1.3 Enable inbound OAuth Client Credentials (both)

*sys\_properties.list* → `glide.oauth.inbound.client.credential.grant_type.enabled` \= **true** (create it as type *true | false* if missing).

### 1.4 Inbound OAuth client for the partner (both)

*Machine Identity Console → Inbound integrations → New integration → **OAuth \- Client credentials grant***

| Field | On the Customer | On the Provider |
| :---- | :---- | :---- |
| Name | `Provider Instance Client` | `Customer Instance Client` |
| Provider name | `ServiceNow Provider (<provider>)` | `ServiceNow Customer (<customer>)` |
| OAuth application user | `integration.inbound` | `integration.inbound` |
| Auth scope | Create `remote_instance_integration`, limited to **Import Set API** and **Attachment API** | Same |
| Allow access only to APIs in selected scope | ✔ | ✔ |

Save and copy the **Client ID** and **Client secret**. The Customer's pair is used on the Provider (Step 4), and the Provider's pair on the Customer (Step 3).

### 1.5 Global prerequisite update set

Apply it now if your instance is on the Australia release (see [Prerequisites](#global-prerequisite-update-set-australia-release-only)).

---

## Step 2 – Install the app on both instances

1. Application picker **Global** → *Connections & Credentials → Credentials → New → Basic Auth Credentials*: name `GitHub`, user \= GitHub username, password \= Personal Access Token.  
2. **ServiceNow Studio → Import from source control**: repository URL, branch `main`, credential `GitHub` → **Import**.  
3. Confirm the app **Remote Instance Integration** (scope `x_1664823_remote_0`) contains the components listed in [Application contents](#application-contents).

>   
> Never edit the app on an instance that only consumes it. Make changes on the development instance, commit, and use **Apply Remote Changes** elsewhere.

---

## Step 3 – Configure the Customer instance

All in **Global** scope.

### 3.1 OAuth connection to the Provider

1. *System OAuth → Application Registry → New → **Connect to a third party OAuth Provider \- Outbound***

| Field | Value |
| :---- | :---- |
| Name | `Provider Instance OAuth` |
| Client ID / Client Secret | The **Provider's** pair (*Customer Instance Client*) |
| Default Grant type | **Client Credentials** |
| Token URL | `https://<provider>.service-now.com/oauth_token.do` |

2. *Connections & Credentials → Credentials → New → OAuth 2.0 Credentials*: name `Provider OAuth`, OAuth Entity Profile \= the profile created above → Submit → **Get OAuth Token** (expect *OAuth token flow completed successfully*).  
3. *Connection & Credential Aliases* → `x_1664823_remote_0.ServiceNowRemoteInstance` → **Connections** → **New → HTTP(s) Connection**:

| Field | Value |
| :---- | :---- |
| Name | `Provider Instance` |
| Credential | `Provider OAuth` |
| Connection URL | `https://<provider>.service-now.com` |

### 3.2 System properties

*sys\_properties.list*, Name starts with `x_1664823_remote_0`:

| Property | Customer value |
| :---- | :---- |
| `x_1664823_remote_0.local_system_name` | `CUSTOMER-<customer>` |
| `x_1664823_remote_0.partner_system_name` | `PROVIDER-<provider>` |
| `x_1664823_remote_0.partner_staging_table` | `x_1664823_remote_0_incident_from_customer` |
| `x_1664823_remote_0.default_assignment_group` | *(empty)* |

### 3.3 Form, roles, notification

- Incident form → **Configure → Related Lists** → add **Remote Sync Log → Incident**.  
- *Integration Admins* group → **Roles** → add the Remote Sync Log role (`x_1664823_remote_0.*`).  
- Notification **Remote sync failed** → *Who will receive* → re-select **Integration Admins** (group sys\_ids differ per instance).

---

## Step 4 – Configure the Provider instance

All in **Global** scope.

### 4.1 OAuth connection to the Customer

Same as 3.1, with:

| Item | Provider value |
| :---- | :---- |
| OAuth profile | `Customer Instance OAuth`, Client ID / secret \= the **Customer's** pair (*Provider Instance Client*), Client Credentials, Token URL `https://<customer>.service-now.com/oauth_token.do` |
| OAuth 2.0 Credential | `Customer OAuth` → **Get OAuth Token** |
| Connection on `x_1664823_remote_0.ServiceNowRemoteInstance` | `Customer Instance`, URL `https://<customer>.service-now.com` |

### 4.2 System properties

| Property | Provider value |
| :---- | :---- |
| `x_1664823_remote_0.local_system_name` | `PROVIDER-<provider>` |
| `x_1664823_remote_0.partner_system_name` | `CUSTOMER-<customer>` |
| `x_1664823_remote_0.partner_staging_table` | `x_1664823_remote_0_incident_from_provider` |
| `x_1664823_remote_0.default_assignment_group` | sys\_id of *Customer Escalations* |

### 4.3 Form, roles, notification, index

- Add the **Remote Sync Log → Incident** related list, give the log role to *Integration Admins*, and re-select the group in the notification (as in 3.3).  
- **Index:** *System Definition → Tables → Incident → Database Indexes*. If there is no index on `correlation_id`, open transform map *Incident From Customer* → related link **Index Coalesce Fields**. Do the same check on the Customer for *Incident From Provider*.

---

## Step 5 – Verify

### Smoke test (one Action call)

On the **Customer**: Workflow Studio → Action **Create or Update Remote Record Using Import Set** → **Test**:

| Input | Value |
| :---- | :---- |
| sender\_sys\_id | sys\_id of a test Incident |
| sender\_number | its number |
| short\_description | `Smoke test` |
| caller\_email | an email that exists on both instances (for example `abel.tuter@example.com`) |
| impact / urgency | `3` / `3` |

Expected: `status_code = 201`, `import_status = inserted`, a Provider number in `remote_number`. Run again with the same sys\_id → `import_status = updated` with the **same** number. Close the smoke-test Incident on the Provider afterwards.

### End to end

Assign a Customer Incident to **Vendor Support**. Expect within seconds: a Provider Incident in *Customer Escalations*, the Customer Incident's **Correlation ID** filled with a work note *Vendor ticket created: INC…*, and a **Success** row in Remote Sync Log.

---

## Test scenarios

- [ ] **Create** – assign to Vendor Support → one Provider Incident, Correlation ID on both sides, log \= inserted.  
- [ ] **Caller lookup** – caller with the same email on both → Caller filled.  
- [ ] **Update** – change urgency on the Customer → Provider updated, no new Incident, log \= updated.  
- [ ] **Customer comment** → appears once on the Provider.  
- [ ] **Vendor comment** → appears once on the Customer.  
- [ ] **No echo** – comments never bounce back.  
- [ ] **Work notes stay private** – never sent.  
- [ ] **Resolve** – Provider resolves with code and notes → Customer Resolved with the same values.  
- [ ] **Not assigned to vendor** → nothing sent.  
- [ ] **Special characters** – quotes, line breaks, non-English text arrive exactly as typed.  
- [ ] **Attachment** – sent once in each direction, never bounced back.  
- [ ] **Partner down** – Provider hibernating → retries, then log \= Failed, work note with link, email sent.  
- [ ] **Bad credential** – wrong client secret → 401 logged as Failed.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| :---- | :---- | :---- |
| *Unable to load connection with alias ID* | No connection on the alias on this instance | Step 3.1 / 4.1 |
| **Get OAuth Token** fails | Client-credentials property not true on the partner, wrong ID/secret, or no OAuth application user | Step 1.3 / 1.4 |
| HTTP **401** | Expired or wrong client secret | Rotate the secret on the partner, update the outbound profile, Get OAuth Token |
| HTTP **403** | Auth scope missing an API, or missing role on `integration.inbound` | Auth scope must include Import Set API and Attachment API; roles per Step 1.1 |
| HTTP **400** *Invalid table* | Wrong `partner_staging_table` value | Step 3.2 / 4.2 |
| `import_status = ignored` on the Customer | Provider sent an update for an Incident that is not linked | Expected protection: the vendor cannot create customer Incidents |
| Payload contains only `u_sender_system` | Build payload Script step input variables not mapped | Map each Script step input to its Action input pill; Save and Publish the Action |
| Caller empty on the Provider | No user with that email on the Provider, or field map missing *Referenced value field name \= email* | Check the user and the transform map |
| *changes* operator missing in Flow conditions | PRB2005619 on Australia | Apply the Global prerequisite update set |
| Duplicates / loops | An old test integration still active, or old correlation data on test Incidents | Deactivate old flows; clear Correlation ID / display on old test records |

Every message, success or failure, is in **Remote Sync Log** (related list on the Incident). Failures also add a work note with a link to the log row and email *Integration Admins*.

---

## Security

| Control | Setting |
| :---- | :---- |
| Authentication | OAuth 2.0 **Client Credentials**; short-lived tokens, no user password stored or sent |
| API restriction | Auth scope limited to Import Set API and Attachment API |
| Integration user | `integration.inbound`, web service access only, roles `import_set_loader`, `import_transformer`, `itil` |
| Secrets | Held in Global-scope credentials behind a Connection & Credential Alias; never in the app or Git |
| Data shared | No work notes, watch lists or internal fields cross the boundary |
| Flows | Run as System user so behaviour does not depend on who changed the Incident |

---

## Application contents

Scope: `x_1664823_remote_0` – **Remote Instance Integration**

| Type | Name | Runs on |
| :---- | :---- | :---- |
| Table (staging) | Incident From Customer | Provider (receives) |
| Table (staging) | Incident From Provider | Customer (receives) |
| Transform map | Incident From Customer → Incident (create or update, coalesce on Correlation ID) | Provider |
| Transform map | Incident From Provider → Incident (update only, coalesce on Correlation ID) | Customer |
| Table | Remote Sync Log | Both |
| System properties | `local_system_name`, `partner_system_name`, `partner_staging_table`, `default_assignment_group` | Both, values per instance |
| Connection & Credential Alias | `ServiceNowRemoteInstance` | Both, connection per instance |
| Action | Create or Update Remote Record Using Import Set | Both |
| Action | Copy Attachment To Remote Instance | Both |
| Flow | Remote Sync: Send Incident Details | Customer |
| Business Rule \+ Subflow | Remote Sync: Send Comment | Both |
| Flow | Remote Sync: Send Progress | Provider |
| Flow | Remote Sync: Send Attachment | Both |
| Subflow | Remote Sync: Write Log | Both |
| Notification | Remote sync failed | Both |

Not in the app (per instance, Global): integration user, groups, OAuth clients, credentials, connections, form layouts, role grants, indexes, and the PRB2005619 prerequisite update set.

---

## Uninstall / rollback

1. Deactivate the Flows and the Business Rule *Remote Sync: Send Comment* on both instances.  
2. Remove the app (*System Applications → My Company Applications* → app → **Uninstall**) or delete it from the instance.  
3. Delete the Global records: OAuth clients and outbound profiles, credentials, connections, `integration.inbound`, groups.  
4. To remove the prerequisite: back out the update set, or delete the `extended_operators` entry from the listed dictionary entries.

---

## Version history

| Version | Notes |
| :---- | :---- |
| 1.0 | Bidirectional Incident sync: details, progress, comments, attachments; Import Set API with coalesce; OAuth Client Credentials; centralized logging and failure alerts |

---

**Author:** · Raja Aqib   
**Role:** ServiceNow Developer