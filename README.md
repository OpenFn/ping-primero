# Integration: UNHCR proGres ↔ UNICEF Primero (via PING Middleware)

## 1. Introduction

The United Nations High Commissioner for Refugees (UNHCR) and the United Nations Children’s Fund (UNICEF) operate protection and case management systems used to support refugees, displaced populations, and vulnerable children.

**proGres** is UNHCR’s core case management and registration platform used to manage refugee records, interventions, and protection activities.

**Primero** is UNICEF’s case management platform used by protection actors to manage cases, referrals, and services for children and families.

To support coordinated protection programming, referrals and referral decisions must move reliably between systems.

This integration uses two intermediary layers:

* **PING middleware** – exposes controlled APIs for interacting with proGres data
* **OpenFn** – orchestrates integrations, performs transformations, and executes system-to-system communication

Through this architecture, referral information and referral decisions are automatically synchronized between **proGres** and **Primero**, reducing manual data entry and improving coordination between agencies.

## 2. Project Objective

The objective of this integration is to enable interoperability between UNHCR and UNICEF protection systems through automated referral data exchange.

Key goals:

* Enable automated **referral creation** between systems
* Synchronize **referral decisions** between organizations
* Reduce manual workflows and duplicate data entry
* Support coordinated case management across agencies
* Maintain traceable and auditable data exchange

The integration relies on **PING middleware APIs** for controlled data exchange and **OpenFn workflows** for orchestration and transformation.

## 3. Systems Involved

| System  | Owner      | Role in Integration                       |
| ------- | ---------- | ----------------------------------------- |
| proGres | UNHCR      | Source protection system                  |
| PING    | UNHCR      | Integration API layer exposing proGres    |
| OpenFn  | OpenFn     | Workflow orchestration and transformation |
| Primero | UNICEF     | Case management platform                  |

## 4. Credentials and Access

| System  | Credential Type                | Description                                         |
| ------- | ------------------------------ | --------------------------------------------------- |
| PING    | API credentials / Bearer Token | Used by OpenFn to retrieve and update proGres data  |
| Primero | API token                      | Used to create and update case and referral records |
| OpenFn  | Platform credentials           | Workflow execution environment                      |

Sensitive credentials are managed through OpenFn credential management and are not stored directly in workflow code.

## 5. Workflow Overview

The integration is implemented using four OpenFn workflows.

| Workflow     | Purpose                                         | Trigger   |
| ------------ | ----------------------------------------------- | --------- |
| Workflow 1.1 | Send referrals from proGres to Primero          | Scheduled |
| Workflow 1.2 | Send referral decisions from Primero to proGres | Scheduled |
| Workflow 2.1 | Send referrals from Primero to proGres          | Scheduled |
| Workflow 2.2 | Send referral decisions from proGres to Primero | Scheduled |

# 6. Workflow Details

## Workflow 1.1 – Send Referrals to Primero

[OpenFn Workflow Link](https://app.openfn.org/projects/1813e824-91aa-4a3b-a49c-afa8b33af333/w/5d2383d8-5f58-451e-aec9-5709e403eea7)

**Purpose**

Retrieves referral interventions from proGres through PING and creates or updates the corresponding cases and services in Primero.

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

<img width="1184" height="465" alt="image" src="https://github.com/user-attachments/assets/917a3966-c9fe-48fe-ae25-2c1d949e0ccb" />

**Field Mapping**

[Flow 1.1 Mapping Sheet Link](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?usp=sharing)

**Logging and Error Handling**

* API responses from PING and Primero are logged
* Validation failures stop processing for the affected records
* Failed records remain eligible for retry on the next run
* Transaction identifiers and record IDs are logged for traceability

## Workflow 1.2 – Send Decision to PING

[OpenFn Workflow Link](https://app.openfn.org/projects/1813e824-91aa-4a3b-a49c-afa8b33af333/w/32d49f20-82a0-464f-b05e-fbabd22ce202)

**Purpose**

Sends referral decisions recorded in Primero back to proGres through the PING ingestion API.

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

<img width="889" height="553" alt="image" src="https://github.com/user-attachments/assets/84ed293b-1ac6-4813-b12a-c7a0a69ef1e7" />

**Field Mapping**

[Flow 1.2 Mapping Doc Link](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1598250610#gid=1598250610)

**Logging and Error Handling**

* Invalid or incomplete referrals are skipped
* API errors returned from PING are captured in workflow logs
* Transaction identifiers are logged for debugging and reconciliation

## Workflow 2.1 – Get Referrals from Primero

[OpenFn Workflow Link](https://app.openfn.org/projects/1813e824-91aa-4a3b-a49c-afa8b33af333/w/360b3fea-569a-401b-b7d4-3a68ad6f575a)

**Purpose**

Retrieves referrals created in Primero and sends them to proGres through PING.

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

<img width="1128" height="514" alt="image" src="https://github.com/user-attachments/assets/57f36fb8-420e-4bc9-97fc-a4e39d0654c5" />

**Field Mapping**

[Flow 2.1 Mapping Sheet Link](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1904546815#gid=1904546815)

**Logging and Error Handling**

* Invalid records are skipped
* PING ingestion API responses are logged
* Ingestion status responses are recorded for monitoring

## Workflow 2.2 – Update Referral Decision in Primero

[OpenFn Workflow Link](https://app.openfn.org/projects/1813e824-91aa-4a3b-a49c-afa8b33af333/w/6a3a0cce-79bb-4a25-8fb2-40a9ff505818)

**Purpose**

Retrieves referral decisions recorded in proGres and updates the corresponding referrals in Primero.

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

<img width="1203" height="414" alt="image" src="https://github.com/user-attachments/assets/28e6543f-ed17-46dc-a96e-22954d31679e" />


**Field Mapping**

[Flow 2.2 Mapping Sheet Link](https://docs.google.com/spreadsheets/d/1YBe7QeUT1JKYicv5d88MBiY1YwIPusRG-pw112xe_Yw/edit?gid=1480651081#gid=1480651081)

**Logging and Error Handling**

* Records without a matching Primero referral are skipped
* Failed updates remain eligible for retry
* API responses and synchronization results are logged

## 7. UAT Test Cases

1. [Workflow 1.1 and 1.2 Test Plan](https://drive.google.com/file/d/1Hssrq2y3OuveXfS9NqiSt40qloZimSWu/view?usp=drive_link)
2. [Workflow 2.1 and 2.2 Test Plan](https://docs.google.com/document/d/1QCdiXYZaFmrYUGYl9enw6F2L6GVtNKwLeBkc-f5vQOU/edit?usp=drive_link)

## 8. Monitoring and Maintenance

Operational monitoring focuses on workflow execution and successful data synchronization.

Key monitoring points:

* OpenFn workflow run logs
* API responses from PING and Primero
* Transaction IDs returned by ingestion APIs
* Verification that records are created or updated in the destination system

## 9. Contacts

### UNHCR Team

| Name | Email | Role |
| ---- | ----- | ---- |
| TODO | TODO  | TODO |

### UNICEF Team

| Name | Email | Role |
| ---- | ----- | ---- |
| TODO | TODO  | TODO |
