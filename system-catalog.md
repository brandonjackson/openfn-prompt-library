# System Catalog

This is the catalog of **mock services** used to test how well OpenFn's AI
assistant generates workflows in one shot. Every prompt in [`prompts/`](./prompts)
targets some subset (2–10) of these systems. The mocks stand in for the real
services so a generated workflow can actually run end-to-end and be scored.

These are OpenFn's bread-and-butter systems: global-health and humanitarian data
tools (DHIS2, CommCare, OpenMRS, FHIR, Kobotoolbox, Primero) plus common
notification / storage utilities (Mailgun, Twilio, Airtable) and a catch-all
Generic HTTP mock.

| # | System | Role | Typical use |
|---|--------|------|-------------|
| 1 | DHIS2 | destination | Health data warehouse: tracked entities, events, aggregate reports |
| 2 | CommCare | source | Mobile case management / form submissions from frontline workers |
| 3 | OpenMRS | source & destination | Electronic medical records (REST + FHIR R4) |
| 4 | FHIR (HAPI R4) | source & destination | Health-data interoperability standard / HIE |
| 5 | Generic HTTP | source & destination | Catch-all REST mock for any system without a dedicated adaptor |
| 6 | Kobotoolbox | source | Field survey data collection |
| 7 | Primero | source & destination | Child-protection / GBV case management (UNICEF) |
| 8 | Mailgun | destination | Transactional email notifications |
| 9 | Twilio | destination | SMS / voice notifications |
| 10 | Airtable | source & destination | Lightweight spreadsheet-database |

---

## 1. DHIS2

**What it is:** An open-source health information management platform used by
governments in 80+ countries for aggregate and individual-level health data
reporting, analysis, and surveillance.

**How it's used in workflows:** DHIS2 is typically the destination system.
Workflows pull data from mobile collection tools (CommCare, ODK, Kobo) or other
systems and push it into DHIS2 as tracked entity instances, events, or aggregate
data values. Common patterns: register a patient as a tracked entity instance,
record a clinical event against a program stage, push weekly aggregate counts as
data value sets.

**Key resources and data structures:**

- **Organisation Unit** — the geographic/administrative hierarchy (country →
  region → district → facility). Everything in DHIS2 is scoped to an org unit.

  ```json
  { "id": "DiszpKrYNg8", "name": "Ngelehun CHC", "shortName": "Ngelehun", "level": 4, "parent": { "id": "O6uvpzGd5pu" } }
  ```

- **Tracked Entity Instance (TEI)** — an individual (patient, beneficiary) being
  tracked across time. Has attributes (name, ID number, phone) and enrollments in
  programs.

  ```json
  { "trackedEntityType": "nEenWmSyUEp", "orgUnit": "DiszpKrYNg8", "attributes": [ { "attribute": "w75KJ2mc4zz", "value": "Jane" }, { "attribute": "zDhUuAYrxNC", "value": "Doe" } ] }
  ```

- **Enrollment** — links a TEI to a program (e.g. "Antenatal Care") with an
  enrollment date and org unit.

  ```json
  { "trackedEntityInstance": "dUE514NMOlo", "program": "IpHINAT79UW", "orgUnit": "DiszpKrYNg8", "enrollmentDate": "2024-01-15", "incidentDate": "2024-01-15" }
  ```

- **Event** — a single data collection point within a program stage (e.g. "First
  antenatal visit"). Contains data values keyed by data element ID.

  ```json
  { "program": "IpHINAT79UW", "programStage": "A03MvHHogjR", "orgUnit": "DiszpKrYNg8", "eventDate": "2024-02-01", "status": "COMPLETED", "dataValues": [ { "dataElement": "qrur9Dvnyt5", "value": "22" } ] }
  ```

- **Data Value Set** — aggregate data (e.g. "120 malaria cases in January at this
  facility"). Used for routine HMIS reporting.

  ```json
  { "dataSet": "pBOMPrpg1QX", "period": "202401", "orgUnit": "DiszpKrYNg8", "dataValues": [ { "dataElement": "FTRrcoaog83", "categoryOptionCombo": "HllvX50cXC0", "value": "120" } ] }
  ```

- **Data Element** — metadata defining what a data value represents (e.g. "Weight
  in kg", "Malaria cases").

  ```json
  { "id": "qrur9Dvnyt5", "name": "Age in years", "shortName": "Age", "valueType": "INTEGER_POSITIVE", "domainType": "TRACKER" }
  ```

**Mock operations:** Create, read, update, upsert, and delete on all resources
above. Filter/query support on `GET` endpoints (e.g. `?filter=name:eq:Jane`).
`GET /api/system/info` returns mock server version. `GET /api/metadata` returns
org units, programs, data elements.

**Authoritative docs:** <https://docs.dhis2.org/en/develop/using-the-api/dhis-core-version-master/introduction.html>
— OpenAPI spec available at `GET /api/openapi.json` on any DHIS2 2.40+ instance.

---

## 2. CommCare

**What it is:** An open-source mobile data collection and case management platform
built by Dimagi, used by frontline health workers and field staff in 130+
countries.

**How it's used in workflows:** CommCare is typically the source system. Data
arrives either via webhook (CommCare forwards form submissions to OpenFn) or via
API pull (OpenFn polls CommCare for new cases/forms). Common patterns: a community
health worker submits a patient registration form in CommCare, OpenFn picks it up
and creates a record in DHIS2, OpenMRS, or Salesforce.

**Key resources and data structures:**

- **Case** — a record tracked over time (a patient, a household, a facility). Has
  a case type, properties, and lifecycle (open/closed).

  ```json
  { "case_id": "abc-123-def", "case_type": "patient", "date_opened": "2024-01-15", "owner_id": "user-456", "properties": { "first_name": "Jane", "last_name": "Doe", "age": "28", "village": "Ngelehun" }, "closed": false }
  ```

- **Form Submission** — a completed survey/form submitted from a mobile device.
  Contains the form XML data and metadata.

  ```json
  { "form": { "@xmlns": "http://openrosa.org/formdesigner/ABC123", "patient_name": "Jane Doe", "age": "28", "meta": { "instanceID": "uuid:form-789", "userID": "user-456", "timeEnd": "2024-01-15T10:30:00Z" } }, "received_on": "2024-01-15T10:31:00Z", "app_id": "app-001" }
  ```

- **Report Data** — tabular report exports from CommCare.

  ```json
  { "columns": ["case_id", "name", "status"], "rows": [["abc-123", "Jane Doe", "active"], ["def-456", "John Smith", "closed"]] }
  ```

**Mock operations:** List/search cases (`GET /a/{domain}/api/v0.5/case/`), get
single case, list form submissions (`GET /a/{domain}/api/v0.5/form/`), receive form
submission via OpenRosa (`POST /a/{domain}/receiver/`), fetch report data. Supports
pagination (`?offset=0&limit=20`) and case filtering (`?type=patient&owner_id=xxx`).

**Authoritative docs:** <https://dimagi.atlassian.net/wiki/spaces/commcarepublic/pages/2143979137/CommCare+Data+APIs>
— API Explorer at `https://www.commcarehq.org/a/{domain}/api/`

---

## 3. OpenMRS

**What it is:** An open-source electronic medical record (EMR) platform designed
for low-resource healthcare settings, powering hospitals and clinics in 40+
countries.

**How it's used in workflows:** OpenMRS can be both source and destination. Common
patterns: sync patient registrations from CommCare into OpenMRS, push clinical
encounter data from OpenMRS to DHIS2 for aggregate reporting, query OpenMRS for
patient records to update in another system. Supports both a REST API and a FHIR
R4 API.

**Key resources and data structures:**

- **Patient** — a person receiving care. Has person attributes, identifiers, and
  addresses.

  ```json
  { "uuid": "patient-uuid-123", "identifiers": [{ "identifier": "MRN-001", "identifierType": { "uuid": "type-uuid" } }], "person": { "names": [{ "givenName": "Jane", "familyName": "Doe" }], "gender": "F", "birthdate": "1996-03-15", "addresses": [{ "cityVillage": "Ngelehun", "country": "Sierra Leone" }] } }
  ```

- **Encounter** — a clinical visit or interaction (e.g. "Outpatient visit on Jan
  15"). Groups observations together.

  ```json
  { "uuid": "enc-uuid-456", "encounterDatetime": "2024-01-15T09:00:00Z", "patient": { "uuid": "patient-uuid-123" }, "encounterType": { "uuid": "enctype-uuid" }, "obs": [ { "concept": { "uuid": "weight-concept-uuid" }, "value": 62.5 } ] }
  ```

- **Observation (Obs)** — a single clinical data point (weight, diagnosis, test
  result).

  ```json
  { "uuid": "obs-uuid-789", "concept": { "uuid": "concept-uuid", "display": "Weight (kg)" }, "value": 62.5, "obsDatetime": "2024-01-15T09:00:00Z", "person": { "uuid": "patient-uuid-123" } }
  ```

- **Concept** — metadata defining what an observation measures (e.g. "Weight in
  kg", "HIV test result"). OpenMRS's core vocabulary system.

  ```json
  { "uuid": "concept-uuid", "display": "Weight (kg)", "datatype": { "display": "Numeric" }, "conceptClass": { "display": "Test" } }
  ```

**Mock operations:** Full CRUD on patients, encounters, observations, concepts via
REST (`/ws/rest/v1/{resource}`). Also supports FHIR R4 endpoints
(`/ws/fhir2/R4/Patient`, `/ws/fhir2/R4/Encounter`). Search via query params (e.g.
`?q=Jane&v=full`). The `v` parameter controls response verbosity (`ref`,
`default`, `full`).

**Authoritative docs:** <https://rest.openmrs.org/> (REST API),
<https://openmrs.atlassian.net/wiki/spaces/docs/pages/25470058/FHIR+Module> (FHIR)

---

## 4. FHIR (HAPI R4)

**What it is:** HL7 FHIR (Fast Healthcare Interoperability Resources) is the
international standard for exchanging healthcare data electronically. HAPI FHIR is
the most common open-source server implementation. This mock implements R4, the
most widely adopted version.

**How it's used in workflows:** FHIR is used as an interoperability layer between
health systems. Common patterns: transform data from a proprietary format into
FHIR resources and POST them to a shared health information exchange (HIE), query a
FHIR server for patient records matching certain criteria, sync resources between
two FHIR-compliant systems.

**Key resources and data structures:**

- **Patient** — a person receiving care.

  ```json
  { "resourceType": "Patient", "id": "pat-123", "name": [{ "given": ["Jane"], "family": "Doe" }], "gender": "female", "birthDate": "1996-03-15", "identifier": [{ "system": "http://example.org/mrn", "value": "MRN-001" }] }
  ```

- **Encounter** — a clinical interaction.

  ```json
  { "resourceType": "Encounter", "id": "enc-456", "status": "finished", "class": { "code": "AMB", "display": "ambulatory" }, "subject": { "reference": "Patient/pat-123" }, "period": { "start": "2024-01-15T09:00:00Z", "end": "2024-01-15T09:30:00Z" } }
  ```

- **Observation** — a clinical measurement or finding.

  ```json
  { "resourceType": "Observation", "id": "obs-789", "status": "final", "code": { "coding": [{ "system": "http://loinc.org", "code": "29463-7", "display": "Body weight" }] }, "subject": { "reference": "Patient/pat-123" }, "valueQuantity": { "value": 62.5, "unit": "kg" } }
  ```

- **Bundle** — a container for multiple resources, used for batch operations and
  search results.

  ```json
  { "resourceType": "Bundle", "type": "searchset", "total": 2, "entry": [ { "resource": { "resourceType": "Patient", "id": "pat-123", "..." : "..." } } ] }
  ```

**Mock operations:** CRUD on Patient, Encounter, Observation, Condition, and
Bundle. Search via `GET /{resourceType}?param=value`. Supports `_search` POST
endpoint. Returns proper Bundle wrappers for search results. Batch/transaction
Bundles via `POST /`.

**Authoritative docs:** <https://hl7.org/fhir/R4/> (spec),
<https://hapifhir.io/hapi-fhir/docs/server_plain/> (HAPI implementation)

---

## 5. Generic HTTP

**What it is:** A catch-all mock that accepts any REST request, stores the payload,
and returns it. This backs the OpenFn `http` adaptor, which is used to integrate
with any system that has a REST API but no dedicated adaptor.

**How it's used in workflows:** When a workflow targets a system that doesn't have
a dedicated adaptor (or when you're prototyping), you use the `http` adaptor with
raw HTTP operations (`get`, `post`, `put`, `patch`, `delete`). The generic mock
captures whatever you send and makes it queryable.

**Key behavior:**

- **Any path works:** `POST /api/v1/whatever` stores the body under collection
  "whatever" with an auto-generated ID. `GET /api/v1/whatever` lists everything in
  that collection. `GET /api/v1/whatever/{id}` returns a single record.
- **Echo mode:** Every response includes a `_mock` field showing what was received
  (method, path, headers, body) for debugging.
- **Request log:** `GET /_admin/requests` shows the full history.

Example data structure (auto-wrapped):

```json
// POST /api/v1/referrals with body { "name": "Jane", "type": "nutrition" }
// Returns:
{ "id": "auto-uuid-123", "name": "Jane", "type": "nutrition", "_createdAt": "2024-01-15T10:00:00Z" }
```

**Mock operations:** Full CRUD on any path. Content-type aware (JSON,
form-urlencoded, XML passthrough). Useful for mocking any system not explicitly
supported.

**Authoritative docs:** <https://docs.openfn.org/adaptors/http> (OpenFn HTTP
adaptor docs)

---

## 6. Kobotoolbox

**What it is:** An open-source suite for field data collection used by humanitarian
organizations, featuring a form builder, mobile data collection app (KoboCollect),
and data management tools. Free for humanitarian use.

**How it's used in workflows:** Kobotoolbox is almost always a source system.
Workflows pull submitted survey data from Kobo and push it to a destination
(DHIS2, OpenMRS, a database). Common patterns: poll for new submissions on a form
asset, transform the response data, and load into the target system.

**Key resources and data structures:**

- **Asset** — a form/survey definition. Has a uid, title, deployment status, and
  list of questions.

  ```json
  { "uid": "aKEj3xKFrZ5pDn", "name": "Household Survey Q1", "asset_type": "survey", "deployment__active": true, "deployment__submission_count": 142, "date_created": "2024-01-01T00:00:00Z", "owner__username": "fieldteam" }
  ```

- **Submission (datum)** — a single completed survey response.

  ```json
  { "_id": 12345, "_uuid": "sub-uuid-abc", "_submission_time": "2024-01-15T10:30:00Z", "_submitted_by": "fieldworker1", "household_head_name": "Jane Doe", "household_size": 5, "water_source": "borehole", "gps_location": "8.484 -13.228 0 0" }
  ```

**Mock operations:** List assets (`GET /api/v2/assets/`), get asset detail
(`GET /api/v2/assets/{uid}/`), list submissions (`GET /api/v2/assets/{uid}/data/`),
get single submission, create submission. Supports `?format=json` and pagination
(`?start=0&limit=30`).

**Authoritative docs:** <https://support.kobotoolbox.org/api.html>,
<https://kf.kobotoolbox.org/api/v2/> (interactive API explorer)

---

## 7. Primero

**What it is:** An open-source child protection and gender-based violence (GBV)
case management platform built by UNICEF, used in 40+ countries for managing cases
of vulnerable children and survivors of violence.

**How it's used in workflows:** Primero is both source and destination. Common
patterns: receive a referral from another agency and create a case in Primero,
sync case updates from Primero to DHIS2 for aggregate reporting, facilitate
inter-agency referrals between Primero and UNHCR's Progres system.

**Key resources and data structures:**

- **Case** — an individual child protection or GBV case with rich nested data.

  ```json
  { "id": "case-uuid-123", "case_id": "CP-2024-001", "status": "open", "registration_date": "2024-01-15", "owned_by": "caseworker1", "data": { "name_first": "Jane", "name_last": "Doe", "age": 12, "sex": "female", "protection_concerns": ["neglect", "abuse"], "risk_level": "high" } }
  ```

- **Incident** — a reported protection incident.

  ```json
  { "id": "inc-uuid-456", "incident_id": "INC-2024-001", "status": "open", "incident_date": "2024-01-14", "data": { "incident_type": "physical_violence", "perpetrator_relationship": "parent", "survivor_age": 12 } }
  ```

- **Referral** — a transfer of a case between services or agencies.

  ```json
  { "id": "ref-uuid-789", "record_type": "case", "record_id": "case-uuid-123", "transitioned_to": "agency_b_worker", "service_type": "health_medical", "notes": "Urgent medical attention needed" }
  ```

**Mock operations:** List cases (`GET /api/v2/cases`), get/create/update case,
list/create incidents, create referrals. Auth via `POST /api/v2/tokens` which
returns a bearer token. Supports pagination and basic filtering.

**Authoritative docs:** <https://github.com/primero/primero/wiki/API-Documentation>,
Primero API v2 guides

---

## 8. Mailgun

**What it is:** A transactional email API service for sending, receiving, and
tracking email programmatically. Used for notification workflows.

**How it's used in workflows:** Mailgun is a destination system for notification
steps. Common patterns: a workflow processes data from a source system, then sends
a confirmation or alert email as the final step. Also used for sending reports or
summaries to stakeholders.

**Key resources and data structures:**

- **Message (send)** — an outbound email. Sent as form-urlencoded or JSON.

  ```json
  { "from": "OpenFn <noreply@example.org>", "to": "admin@health.gov", "subject": "Weekly Report: 142 cases processed", "text": "See attached summary...", "html": "<h1>Weekly Report</h1><p>142 cases processed</p>" }
  ```

  Response:

  ```json
  { "id": "<msg-uuid-123@sandbox.mailgun.org>", "message": "Queued. Thank you." }
  ```

- **Event** — a delivery tracking event (delivered, opened, bounced, etc.).

  ```json
  { "event": "delivered", "timestamp": 1705312200, "recipient": "admin@health.gov", "message": { "headers": { "message-id": "msg-uuid-123" } }, "delivery-status": { "code": 250, "description": "" } }
  ```

**Mock operations:** Send message (`POST /v3/{domain}/messages`), list events
(`GET /v3/{domain}/events`), get stats (`GET /v3/{domain}/stats/total`). Sending
always returns success. Events are generated automatically from sent messages (mock
delivers everything immediately).

**Authoritative docs:** <https://documentation.mailgun.com/docs/mailgun/api-reference/openapi-final/tag/Messages/>

---

## 9. Twilio

**What it is:** A cloud communications API for sending SMS/MMS messages and making
voice calls. Used for SMS-based notifications and two-way messaging workflows.

**How it's used in workflows:** Twilio is a destination system for SMS notification
steps. Common patterns: send an SMS alert to a health worker when a new case is
assigned, send appointment reminders to patients, send verification codes.
Occasionally used as a source via webhook (incoming SMS triggers a workflow).

**Key resources and data structures:**

- **Message** — an outbound or inbound SMS.

  ```json
  {
    "sid": "SMxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "account_sid": "ACtest123456",
    "from": "+15551234567",
    "to": "+23276543210",
    "body": "Reminder: Clinic visit tomorrow at 9am",
    "status": "queued",
    "direction": "outbound-api",
    "date_created": "2024-01-15T10:00:00Z",
    "date_sent": null,
    "price": null
  }
  ```

  Send request (form-urlencoded):

  ```
  To=+23276543210&From=+15551234567&Body=Reminder: Clinic visit tomorrow at 9am
  ```

- **Call** — a voice call record.

  ```json
  { "sid": "CAxxxxxxxxxxxxxxxxxxxxxxxxxxxxx", "from": "+15551234567", "to": "+23276543210", "status": "completed", "duration": "45", "direction": "outbound-api" }
  ```

**Mock operations:** Send message (`POST /2010-04-01/Accounts/{sid}/Messages.json`),
list messages (`GET .../Messages.json`), get single message. Sending always returns
`status: "queued"`. Mock auto-advances status to "sent" then "delivered" on
subsequent reads. List/get calls.

**Authoritative docs:** <https://www.twilio.com/docs/sms/api/message-resource>

---

## 10. Airtable

**What it is:** A cloud spreadsheet-database hybrid with a clean REST API. Used by
smaller organizations as a lightweight data store, CRM, or project tracker when a
full database is overkill.

**How it's used in workflows:** Airtable can be source or destination. Common
patterns: sync beneficiary data from a field collection tool into an Airtable base
for program staff to review, pull records from Airtable and push to a reporting
system, use Airtable as the "master list" that feeds into multiple downstream
systems.

**Key resources and data structures:**

- **Record** — a row in a table. Fields are defined per table. Airtable wraps
  everything in a `fields` object.

  ```json
  {
    "id": "recABC123",
    "createdTime": "2024-01-15T10:00:00.000Z",
    "fields": {
      "Name": "Jane Doe",
      "Age": 28,
      "Village": "Ngelehun",
      "Status": "Active",
      "Registration Date": "2024-01-15",
      "Linked Cases": ["recDEF456"]
    }
  }
  ```

- **List response** — paginated list of records.

  ```json
  { "records": [ { "id": "recABC123", "fields": { "..." : "..." } } ], "offset": "itrXXXXXX" }
  ```

- **Batch create/update** — up to 10 records per request.

  ```json
  { "records": [ { "fields": { "Name": "Jane Doe", "Age": 28 } }, { "fields": { "Name": "John Smith", "Age": 35 } } ] }
  ```

**Mock operations:** List records (`GET /v0/{baseId}/{tableName}`), get single
record, create records (single or batch), update records (single or batch via
`PATCH`), delete records. Supports `?filterByFormula=`, `?sort[0][field]=Name`,
`?pageSize=`, `?offset=` for pagination. Batch limit of 10 records per request
enforced.

**Authoritative docs:** <https://airtable.com/developers/web/api/introduction>
