# UNICEF <> UNHCR Interagency Interoperability

## Project Status

> **Note on implementation history:** The original OpenFn implementation connecting Primero and proGres used UNHCR's DTP middleware. That **DTP-based Gambella pilot** has been inactive since July 2025. Its configuration is preserved on the `adaptors-version-upgrade` branch at [github.com/OpenFn/primero-progres](https://github.com/OpenFn/primero-progres).
>
> **The current implementation** uses UNHCR's **PING middleware** and is documented in this README. This document supersedes the DTP-era documentation while retaining the project context, programmatic requirements, and compliance information that remain applicable.

Contact [support@openfn.org](mailto:support@openfn.org) for assistance with either implementation.

---

## 1. Introduction

The United Nations High Commissioner for Refugees (UNHCR) and the United Nations Children's Fund (UNICEF) operate protection and case management systems used to support refugees, displaced populations, and vulnerable children.

**proGres** is UNHCR's core case management and registration platform used to manage refugee records, interventions, and protection activities.

**Primero** is UNICEF's case management platform used by protection actors to manage cases, referrals, and services for children and families.

To support coordinated protection programming, referrals and referral decisions must move reliably between systems. This integration uses two intermediary layers:

- **PING middleware** – exposes controlled APIs for interacting with proGres data
- **OpenFn** – orchestrates integrations, performs transformations, and executes system-to-system communication

Through this architecture, referral information and referral decisions are automatically synchronized between **proGres** and **Primero**, reducing manual data entry and improving coordination between agencies.

You can find an overview, expectations, prerequisites, and timelines for implementing interoperability at [www.cpims.org/interoperability](https://www.cpims.org/interoperability).

---

## 2. Project Objective

The objective of this integration is to enable interoperability between UNHCR and UNICEF protection systems through automated referral data exchange.

Key goals:

- Enable automated **referral creation** between systems
- Synchronize **referral decisions** between organizations
- Reduce manual workflows and duplicate data entry
- Support coordinated case management across agencies
- Maintain traceable and auditable data exchange

The integration relies on **PING middleware APIs** for controlled data exchange and **OpenFn workflows** for orchestration and transformation.

---

## 3. Programmatic Requirements

| Requirement | Solution Overview |
| --- | --- |
| Primero will be a source of intervention cases and decisions and a destination for intervention cases and decisions coming from proGres v4. | OpenFn automates data extraction via the Primero v2 API to get and process outbound referrals that should be sent to proGres. Using OpenFn workflows, this automation is configured to run on a scheduled basis, extract data from Primero in JSON format, and transform it to align with PING system requirements and the agreed mapping specifications. |
| proGres v4 will be a source of intervention cases and decisions and a destination for intervention cases and decisions coming from Primero. | OpenFn polls the PING middleware APIs on a scheduled basis to retrieve new referrals and decisions from proGres. On retrieval, OpenFn workflows validate and transform the data and forward it to Primero. Decisions made in Primero are subsequently sent back to proGres via PING ingestion APIs. |
| Due to the highly sensitive nature of the data, it must be secured properly. There is a high need for authorization and authentication to control access to the data. | OpenFn out-of-box platform features are leveraged to ensure the highest level of security. Credentials are stored in OpenFn's credential management system and are never embedded in workflow code. See the _Security and Compliance_ section for additional details. |

---

## 4. Systems Involved

| System | Owner | Role in Integration |
| --- | --- | --- |
| proGres v4 | UNHCR | Source protection and case management system |
| PING | UNHCR | Integration API layer exposing proGres data |
| OpenFn | OpenFn | Workflow orchestration and transformation |
| Primero v2 | UNICEF | Case management platform |

---

## 5. Credentials and Access

| System | Credential Type | Description |
| --- | --- | --- |
| PING | API credentials / Bearer Token | Used by OpenFn to retrieve and update proGres data |
| Primero | API token | Used to create and update case and referral records |
| [OpenFn](https://docs.openfn.org) | Platform credentials | Integration platform used to automate and orchestrate data exchange workflows |

Sensitive credentials are managed through OpenFn credential management and are not stored directly in workflow code.

---

## 6. Data Flows & Workflow Overview

The integration is implemented as four OpenFn workflows covering two bidirectional flows.

| Workflow | Direction | Purpose | Trigger |
| --- | --- | --- | --- |
| Workflow 1.1 | proGres → Primero | Send referrals from proGres to Primero | Scheduled Cron |
| Workflow 1.2 | Primero → proGres | Send referral decisions from Primero to proGres | Scheduled Cron |
| Workflow 2.1 | Primero → proGres | Send referrals from Primero to proGres | Scheduled Cron |
| Workflow 2.2 | proGres → Primero | Send referral decisions from proGres to Primero | Scheduled Cron |

Full documentation details can be [found here](https://drive.google.com/drive/folders/1Y0FGxB3jctjCUo6EtMGp0h46cjBVdudO).

**Flow 1 (proGres → Primero Referral Sharing)**

This flow automates the sending of referrals originating in proGres to Primero (Workflow 1.1) and the return of Primero's acceptance or rejection decisions back to proGres (Workflow 1.2).

**Flow 2 (Primero → proGres Referral Sharing)**

This flow automates the sending of referrals originating in Primero to proGres (Workflow 2.1) and the return of proGres's acceptance or rejection decisions back to Primero (Workflow 2.2).

If consent is revoked by a Primero user, please see Annex 1: Consent and Assent in Primero's CPIMS+ and proGres v4 Child Protection Modules for the applicable SOP.

Find both data flow diagrams [here](https://docs.google.com/presentation/d/1S_BuMzJ2MzcvJCoHUPWxkfwYkFP-V-ValIWH4EP3Cj8/edit#slide=id.p).

---

## 7. Workflow Details

### Workflow 1.1 – Send Referrals to Primero

[OpenFn Workflow Link](https://app.openfn.org/projects/1813e824-91aa-4a3b-a49c-afa8b33af333/w/5d2383d8-5f58-451e-aec9-5709e403eea7)
<img width="1314" height="849" alt="image" src="https://github.com/user-attachments/assets/3add88f5-299d-4df2-b187-29e899b34d11" />

<img width="1314" height="849" alt="Workflow 1.1 canvas" src="https://github.com/user-attachments/assets/3add88f5-299d-4df2-b187-29e899b34d11" />

**Purpose**

Retrieves referral interventions from proGres through PING and creates or updates the corresponding cases and services in Primero. The full specification is described [here](https://docs.google.com/document/d/1g1otwtnoddVX0pi8MPZ-tSnFs2H-0MwE/edit?usp=drive_link&ouid=100028693760577730296&rtpof=true&sd=true).

**Trigger**

Scheduled workflow execution.

**Workflow Steps**

1. Retrieve referral interventions from PING using the retrieval API.
2. Retrieve detailed intervention and individual information.
3. Aggregate related data such as languages and specific needs.
4. Transform proGres data into the Primero case and service format.
5. Search for an existing Primero case using the UNHCR individual identifier.
6. Create a new case or update the existing case in Primero.
7. Update the intervention transfer status in proGres through PING.

**Workflow Diagram**

<img width="1184" height="465" alt="Workflow 1.1 diagram" src="https://github.com/user-attachments/assets/917a3966-c9fe-48fe-ae25-2c1d949e0ccb" />

**Field Mapping**

[Flow 1.1 Mapping Sheet](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?usp=sharing)

**Logging and Error Handling**

- API responses from PING and Primero are logged.
- Validation failures stop processing for the affected records.
- Failed records remain eligible for retry on the next run.
- Transaction identifiers and record IDs are logged for traceability.

---

### Workflow 1.2 – Send Decision to PING

[OpenFn Workflow Link](https://app.openfn.org/projects/1813e824-91aa-4a3b-a49c-afa8b33af333/w/32d49f20-82a0-464f-b05e-fbabd22ce202)
<img width="1314" height="849" alt="image" src="https://github.com/user-attachments/assets/63f8a716-7ba2-4f55-b903-733b34f9ccb7" />

<img width="1314" height="849" alt="Workflow 1.2 canvas" src="https://github.com/user-attachments/assets/63f8a716-7ba2-4f55-b903-733b34f9ccb7" />

**Purpose**

Sends referral decisions recorded in Primero back to proGres through the PING ingestion API. The full specification is described [here](https://docs.google.com/document/d/1K8PR7a-l3AhL-nSaG6dPtlp2hubQUyPN/edit?usp=drive_link&ouid=100028693760577730296&rtpof=true&sd=true).

**Trigger**

Scheduled workflow execution.

**Workflow Steps**

1. Retrieve updated cases from Primero using a timestamp cursor.
2. Filter referrals assigned to UNHCR.
3. Extract referral decision status and intervention identifiers.
4. Transform decision information into the proGres intervention decision format.
5. Send the decision payload to the PING ingestion API.
6. Confirm ingestion status using the returned transaction identifier.

**Workflow Diagram**

<img width="889" height="553" alt="Workflow 1.2 diagram" src="https://github.com/user-attachments/assets/84ed293b-1ac6-4813-b12a-c7a0a69ef1e7" />

**Field Mapping**

[Flow 1.2 Mapping Sheet](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1598250610#gid=1598250610)

**Logging and Error Handling**

- Invalid or incomplete referrals are skipped.
- API errors returned from PING are captured in workflow logs.
- Transaction identifiers are logged for debugging and reconciliation.

---

### Workflow 2.1 – Get Referrals from Primero

[OpenFn Workflow Link](https://app.openfn.org/projects/1813e824-91aa-4a3b-a49c-afa8b33af333/w/360b3fea-569a-401b-b7d4-3a68ad6f575a)
<img width="1314" height="849" alt="image" src="https://github.com/user-attachments/assets/5f3b5fde-5550-4527-9b92-173bd7dd7480" />

<img width="1314" height="849" alt="Workflow 2.1 canvas" src="https://github.com/user-attachments/assets/5f3b5fde-5550-4527-9b92-173bd7dd7480" />

**Purpose**

Retrieves referrals created in Primero and sends them to proGres through PING. The full specification is described [here](https://docs.google.com/document/d/1WVOnuc6XGsUIv-kbWq775K8jstNT_UE4/edit?usp=drive_link&ouid=100028693760577730296&rtpof=true&sd=true).

**Trigger**

Scheduled workflow execution.

**Workflow Steps**

1. Retrieve recently updated cases from Primero.
2. Filter cases where referrals are assigned to UNHCR.
3. Retrieve referral records associated with those cases.
4. Retrieve Primero user details for referral contact information.
5. Transform referral data into the proGres intervention structure.
6. Send the referral payload to the PING ingestion API.
7. Query the ingestion status endpoint to confirm processing.

**Workflow Diagram**

<img width="1128" height="514" alt="Workflow 2.1 diagram" src="https://github.com/user-attachments/assets/57f36fb8-420e-4bc9-97fc-a4e39d0654c5" />

**Field Mapping**

[Flow 2.1 Mapping Sheet](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1904546815#gid=1904546815)

**Logging and Error Handling**

- Invalid records are skipped.
- PING ingestion API responses are logged.
- Ingestion status responses are recorded for monitoring.

---

### Workflow 2.2 – Update Referral Decision in Primero

[OpenFn Workflow Link](https://app.openfn.org/projects/1813e824-91aa-4a3b-a49c-afa8b33af333/w/6a3a0cce-79bb-4a25-8fb2-40a9ff505818)
<img width="1314" height="849" alt="image" src="https://github.com/user-attachments/assets/73b8e34f-32ec-47ea-92e9-9c8eb19001d9" />

<img width="1314" height="849" alt="Workflow 2.2 canvas" src="https://github.com/user-attachments/assets/73b8e34f-32ec-47ea-92e9-9c8eb19001d9" />

**Purpose**

Retrieves referral decisions recorded in proGres and updates the corresponding referrals in Primero. The full specification is described [here](https://docs.google.com/document/d/1Nh84EPAxt1EY2fAIGbZz2fh5Mq8nslq8/edit?usp=drive_link&ouid=100028693760577730296&rtpof=true&sd=true).

**Trigger**

Scheduled workflow execution.

**Workflow Steps**

1. Retrieve referral decision records from PING.
2. Filter decision records based on status values.
3. Retrieve detailed decision information for each referral.
4. Locate the corresponding referral within Primero.
5. Update the referral decision status in Primero.
6. Mark the record as synchronized in proGres.

**Workflow Diagram**

<img width="1203" height="414" alt="Workflow 2.2 diagram" src="https://github.com/user-attachments/assets/28e6543f-ed17-46dc-a96e-22954d31679e" />

**Field Mapping**

[Flow 2.2 Mapping Sheet](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1480651081#gid=1480651081)

**Logging and Error Handling**

- Records without a matching Primero referral are skipped.
- Failed updates remain eligible for retry.
- API responses and synchronization results are logged.

---

## 8. What Information Can Be Shared

This interoperability solution leverages the standardized Global Inter-Agency Case Management Task Force Interagency Forms in Primero but may be localized for each implementation to meet unique data sharing agreements and/or Primero system configurations.

The following details are shared between Primero and proGres:

1. Consent of the child
2. Basic identification (name(s), date of birth, sex, current address, telephone, languages spoken, UNHCR individual ID, UNHCR registration group number)
3. Services (type of service, reason for referral, implementing agency, service requestor details, and referral status)
4. Users (configured for receiving proGres referrals)

> **Important:** These workflows transmit refugee PII across three systems — full name, date of birth, sex, UNHCR ID, phone, address, and protection concerns including SGBV indicators. Confirm that data processing agreements between UNHCR and UNICEF cover the specific fields in scope, and that OpenFn's role as a data processor is reflected in those agreements.

The exchange of only a defined set of service types (a.k.a. "intervention types") is supported between agencies. See the [mapping specification](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?usp=sharing) for the current list. Primero and proGres users must be trained to NOT send other service types in interagency referral requests.

This solution can be re-implemented and localized with a relatively low level of effort for other Primero systems. To reuse it, implementers can clone the OpenFn project configuration and make the adjustments noted in the _Configuration Considerations_ section. The key steps for re-implementation are outlined at [github.com/OpenFn/unicef-unhcr-io](https://github.com/OpenFn/unicef-unhcr-io).

Data element mapping specifications detail which fields will be exchanged between systems. Given each implementation's unique data sharing agreement and Primero localizations, these may change. For every Primero field in the mapping, the document notes: (1) the related Interagency Form, and (2) whether the mapping may be localized.

If changes are to be made to the mapping specification, they should (1) first be identified and documented by business owners in the mapping document, then (2) provided to technical implementers to make the corresponding OpenFn workflow changes. For change requests, duplicate the relevant tab to create a new version, and add highlighting to show changes for version control purposes.

---

## 9. Solution Assumptions

**Workflow 1.1**

The PING endpoint SHP-2358 may return multiple rows per intervention due to joins on language and specific needs. The workflow deduplicates these before sending to Primero. If SHP-2358 changes its response structure, deduplication will produce incorrect results without erroring.

If more than one Primero case matches on `unhcr_individual_no`, only the first result is used with no warning.

**Workflow 1.2**

If a service's `progres_interventionnumber` doesn't follow the expected comma-delimited format (intervention number, LP intervention ID, business unit), the record is dropped with a log message and never retried. This will quietly discard any referral created before the composite format was introduced.

**Workflow 2.1**

Cases are filtered to those where at least one service has `service_implementing_agency === 'UNHCR'` — exact case match. A Primero instance configured with lowercase or mixed-case values will result in all referrals being silently excluded.

When a caseworker's Primero profile is incomplete, the workflow silently substitutes the CPIMS admin's contact details (name, email, and phone) in the referral sent to proGres. The receiving party in proGres has no indication that this substitution occurred.

If a Primero `service_type` value isn't found in the service map, the workflow sends the legal support GUID to proGres with no warning. A misconfigured or extended Primero lookup list will produce silent mismatch in proGres.

**Workflow 2.2**

The `statusText()` function defaults anything that isn't explicitly `125080002` to `"accepted"`. If proGres introduces a new status code, it will be written to Primero as an acceptance.

**General Assumptions**

1. UNICEF Primero updates on services will not be shared with UNHCR/proGres; only the original referral request and subsequent decisions are exchanged.
2. If a UNICEF Primero user revokes consent for a case, a manual SOP will be followed for communicating that with UNHCR. The interoperability solution does not communicate this change automatically.
3. Only the configured set of service/intervention types is supported. See the mapping specification linked above.

---

## 10. UAT Test Cases

1. [Workflow 1.1 and 1.2 Test Plan](https://drive.google.com/file/d/1Hssrq2y3OuveXfS9NqiSt40qloZimSWu/view?usp=drive_link)
2. [Workflow 2.1 and 2.2 Test Plan](https://docs.google.com/document/d/1QCdiXYZaFmrYUGYl9enw6F2L6GVtNKwLeBkc-f5vQOU/edit?usp=drive_link)

---

## 11. Configuration Considerations

When implementing this interoperability solution for a new Primero/proGres instance, implementers and system administrators should consider the following:

1. **PING Environment Identifiers:** Every PING API call carries three identifiers — `ShippingProcessId`, `PartnerId`, and `InteropId` — that are specific to this UNHCR-UNICEF deployment. Before deploying elsewhere, all of these must be replaced with values issued by PING for the new environment. The partner ID also appears directly in the ingestion endpoint URL (e.g. `/api/ingestion/v2/PNR-1363/data`), so it must be updated there as well, not just in the request body.

2. **Service/Intervention Type GUIDs:** Two mappings translate service types between Primero and proGres (one per flow direction). Both are hardcoded with proGres GUIDs that belong to this specific proGres installation. A different country's proGres instance will have different GUIDs, and incorrect values may be accepted silently. These must be verified against the target environment before go-live.
   - To update mappings for the **proGres → Primero** direction, locate the intervention type map in Workflow 1.1. New services should follow the format: `'proGres intervention type description': 'Primero DB value'`.
   - To update mappings for the **Primero → proGres** direction, locate the service map in Workflow 2.1. New services should follow the format: `'Primero DB value': 'proGres intervention GUID'`.
   - **Reminder:** Make any mapping changes in the mapping specification document before updating the workflow configuration.

3. **Intervention Type Descriptions (Workflow 1.1):** The workflow only accepts a fixed set of proGres intervention type descriptions. Anything outside that list throws an error. If the new deployment's proGres uses different terminology, this map must be updated or referrals will fail on arrival.

4. **Language Lookup Keys:** The language lookup keys (e.g. `language1`, `language2`) are specific to this Primero instance's configuration. A different Primero deployment will almost certainly use different keys. Both the inbound and outbound language maps must be replaced.

5. **Required Fields in Primero:** Are there any fields on the Primero case or related forms that are (1) instance-specific localizations and (2) required when a case is registered? If so, these must be included in the mapping specifications. The source value must either come from a corresponding field in proGres, or a default value must be provided; otherwise a required field will be submitted blank.

6. **Focal Point User:** Has the "focal point user" for receiving referrals from UNHCR been identified? Initially, one user (e.g. `progresv4_primero_intake`) reviews all referrals from proGres and reassigns them to the appropriate case workers. As the programme expands, per-agency intake users (e.g. `plan_international_intake`, `save_the_children_intake`) can be configured.

7. **Implementing Agency Value:** All services are configured to be exchanged with UNHCR. The Primero workflow filters on `service_implementing_agency === 'UNHCR'` (exact case match). Ensure the Primero instance's lookup list value matches exactly, and that users select "UNHCR" and `unhcr_cw` when creating outbound referrals.

8. **Hardcoded Values Requiring Human Decisions Before Go-Live:** The following values do not cause an immediate error if left unchanged but will behave incorrectly in production:
   - The fallback CPIMS admin email in Workflow 2.1
   - The two proGres organization GUIDs in Workflow 2.1 (`progres_businessunit` and `progres_organizationfrom`)
   - The `module_id` (set to `primeromodule-cp`) in Workflow 1.1

---

## 12. Monitoring and Maintenance

Operational monitoring focuses on workflow execution and successful data synchronization.

Key monitoring points:

- OpenFn workflow run logs
- API responses from PING and Primero
- Transaction IDs returned by ingestion APIs
- Verification that records are created or updated in the destination system

---

## 13. Security and Compliance
 
The OpenFn platform complies with the [UNICEF CLASS I System Security Requirements](https://www.ungm.org/UNUser/Documents/DownloadPublicDocument?docId=784274). OpenFn is not a standalone system but rather a system carefully configured by UNICEF, OpenFn, or UNICEF's partners to comply with project-level security requirements and to connect with other systems in the UNICEF ecosystem (such as Primero). OFG implementation consultants must support partners to ensure the OpenFn implementation is in compliance with relevant security requirements and adhering to best practices.
 
Full Security and Go-Live Checklist: [view here](https://docs.google.com/document/d/1FPspJEJLW6wVQOY6Glq5Wc0xmf85gbIt/edit).
 
| Checklist Item | Comments | Timeline | Status |
| --- | --- | --- | --- |
| 1. Do not persist Primero data as Messages in OpenFn | OpenFn projects should be configured to not persist Primero information in the OpenFn project inbox. Primero case data should never sit at rest in OpenFn. OpenFn is the data processor; UNICEF is the data controller. | Before go-live and connection to production systems | ⏳ Pending — to be verified before go-live. |
| 2. Delete notification payloads after successful sync | OpenFn production projects will be configured to delete payloads received from PING after the data has been successfully synced with Primero, so that no data is stored on OpenFn servers at rest. | Before go-live and connection to production systems | ⏳ Pending — to be verified before go-live. |
| 3. Configure Github repository and connect with OpenFn project | GitHub provides version control and management of different development pipelines and change requests. | Before integration setup begins | ⏳ Pending — repository not yet configured for production. |
| 4. Seek partner sign-off on information logged in OpenFn Activity History | Do not log any personally identifiable information — only system IDs and date timestamps required for transaction auditing. | Before go-live and connection to production systems | ⏳ Pending — partner sign-off not yet obtained. |
| 5. Reset OpenFn credentials for production system and share with UNHCR | PING will authenticate to OpenFn using credentials that must be reset before go-live and shared securely with UNHCR administrators. | Before go-live and connection to production systems | ⏳ Pending — prototype credentials in use; production credentials not yet issued. |
| 6. Confirm appropriate access settings for the Github repository | GitHub repositories should never contain data, only integration workflow code. Consider making the repository private and granting read/write access only to relevant project administrators. | Before go-live and connection to production systems | ⏳ Pending — access controls not yet finalized. |
| 7. Confirm list of administrator users who need access to the OpenFn project | Only project admins who need access to OpenFn for ongoing integration monitoring should be granted access to the production project on OpenFn. | Before go-live and connection to production systems | ⏳ Pending — user list not yet confirmed. |
| 8. Confirm with UNICEF and UNHCR partners that OpenFn credential is granted API-only access | Ensure no overly broad or unnecessary permissions are granted to the OpenFn credentials used to access Primero or PING. | Before go-live and connection to production systems | ⏳ Pending — to be confirmed with partners before go-live. |
| 9. Train administrators on OpenFn platform administration, user management, and integration activity monitoring | Training is critical to total solution handover and ensuring security best practices are maintained during the project lifetime. | Before go-live and connection to production systems | ⏳ Pending — training not yet scheduled. |
 
---

## 14. Links

**Current PING-based solution documentation**
- [Workflow Documentation Folder](https://drive.google.com/drive/folders/1Y0FGxB3jctjCUo6EtMGp0h46cjBVdudO)
- [Flow 1.1 Mapping Sheet](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?usp=sharing)
- [Flow 1.2 Mapping Sheet](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1598250610#gid=1598250610)
- [Flow 2.1 Mapping Sheet](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1904546815#gid=1904546815)
- [Flow 2.2 Mapping Sheet](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1480651081#gid=1480651081)
- [Security and Go-Live Checklist](https://docs.google.com/document/d/1FPspJEJLW6wVQOY6Glq5Wc0xmf85gbIt/edit)
- [Workflow 1.1 and 1.2 Test Plan](https://drive.google.com/file/d/1Hssrq2y3OuveXfS9NqiSt40qloZimSWu/view?usp=drive_link)
- [Workflow 2.1 and 2.2 Test Plan](https://docs.google.com/document/d/1QCdiXYZaFmrYUGYl9enw6F2L6GVtNKwLeBkc-f5vQOU/edit?usp=drive_link)

**Gambella DTP-era solution documentation (archived)**
- [Github site with detailed documentation](https://openfn.github.io/primero-progres/)
- [Ethiopia, Gambella Primero – ProGres IA Referral Exchange Mapping Specification (2021 Final Version)](https://docs.google.com/spreadsheets/d/1j5bVbg1-c3Pwyx3DiALxaOD4ulGTEdEGJCrgu2DVT38/edit#gid=1470043016)
- [Archived Legacy Decision-Making IA Mapping Specification for Ethiopia, Gambella (2019–2021)](https://docs.google.com/spreadsheets/d/1ieoiGsdGuOA1E3jbw0lWkW-H-V9RzrtxYUdrRsHpOF4/edit#gid=790782914)
- [Primero IO Training for Administrators](https://docs.google.com/presentation/d/1u8A2Ke4n7i4_IXF65wA5yKeBIcfUijVvnUwnvPjEtM4/edit#slide=id.ga81cdd0b96_0_755)
- [Primero-Progres IO Flows and Testing Steps](https://docs.google.com/presentation/u/2/d/1H9ncQvGcWrT6nVn--wAYKxjNIzKoMt7IkbFiEc1_F6s/edit#slide=id.geff43a9d2b_0_128)

**Template interagency solution documentation** (reusable/replicable)
- [Github site with detailed documentation](https://github.com/OpenFn/unicef-unhcr-io)
- [IA Data Element Mapping Specification template](https://docs.google.com/spreadsheets/d/1y3bFz7AL8H4D-H-G4WVx-vdrxwzroyGKTvFCMBvjwCI/edit#gid=1470043016)
- [Programmatic prerequisites, expectations and timelines](https://www.cpims.org/interoperability)

---

## 15. Support Contacts

### UNHCR Team

| Name | Email | Role |
| ---- | ----- | ---- |
| Jessica Stuart-Clark | stuartcl@unhcr.org  | Child Protection Officer |

### UNICEF Team

| Name | Email | Role |
| ---- | ----- | ---- |
| Jan Panchalingam | jpanchalingam@unicef.org  | Primero Lead |

### OpenFn

Post on [community.openfn.org](https://community.openfn.org) or contact [support@openfn.org](mailto:support@openfn.org) for private queries.
