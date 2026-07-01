# openfn-prompt-library

A library of natural-language **prompts** for testing how well OpenFn's AI
assistant generates workflows **in one shot** — i.e. how good a runnable,
multi-step workflow it produces from a single prompt, with no follow-up coaching.

Each prompt describes a realistic integration story across a set of mock services.
A human tester pastes the prompt into the OpenFn AI assistant, runs the generated
workflow against the mocks, and scores it using the prompt's `evaluate` criteria.

- **The mock services** are described in [`system-catalog.md`](./system-catalog.md).
- **The prompts** live in [`prompts/`](./prompts) as one YAML file each.

---

## Why

OpenFn workflows connect systems together: a **trigger** (webhook or cron) kicks
off a chain of **steps** (jobs), each written for one system's adaptor (e.g.
`dhis2`, `commcare`, `http`, `mailgun`), and **edges** between steps can carry
conditions — that's where branching lives. The AI assistant is supposed to turn a
plain-English description into exactly that structure.

This repo is a controlled test set for that capability. By fixing the target
systems (a mock catalog) and writing precise pass/fail criteria up front, we can
run the same prompts repeatedly — across assistant versions, models, or prompt
tweaks — and compare results apples-to-apples.

---

## What the assistant sees (and what it doesn't)

The whole test rests on the assistant getting **only OpenFn's docs plus the one
`prompt` field**. It does not see `system-catalog.md`, the mocks, or any id list. To
keep that honest while still producing a workflow that actually runs, context is
split three ways, exactly as in a real OpenFn project:

1. **Connection facts** (base URLs, tokens, the Airtable base id, the Twilio
   from-number, the Mailgun domain) live in **OpenFn credentials**, set up by the
   tester. A workflow reads them via `state.configuration`; they never appear in a
   prompt, and their absence is realistic rather than a gap.
2. **Named targets** (a DHIS2 program, an org unit, a Kobo asset, an Airtable table)
   are **seeded into the mocks**. The prompt names them in plain language and a
   correct workflow resolves the name to an id at runtime.
3. **Genuine business specifics** (who to alert, a non-obvious code mapping) are
   stated **in the prompt**, because a real user would state them.

The payoff is that prompts stay short and representative instead of becoming
instance-id dumps. See `system-catalog.md` for the per-system seeded fixtures and
credential fields.

---

## The system catalog

Every prompt targets 2–10 of these ten mock services. Full details, data shapes,
and mock endpoints are in [`system-catalog.md`](./system-catalog.md).

| # | System | Role | One-liner |
|---|--------|------|-----------|
| 1 | DHIS2 | destination | Health data warehouse (tracked entities, events, aggregates) |
| 2 | CommCare | source | Mobile case management / form submissions |
| 3 | OpenMRS | source & destination | Electronic medical records (REST + FHIR R4) |
| 4 | FHIR (HAPI R4) | source & destination | Health-data interoperability standard / HIE |
| 5 | Generic HTTP | source & destination | Catch-all REST mock for any un-adapted system |
| 6 | Kobotoolbox | source | Field survey data collection |
| 7 | Primero | source & destination | Child-protection / GBV case management (UNICEF) |
| 8 | Mailgun | destination | Transactional email |
| 9 | Twilio | destination | SMS / voice |
| 10 | Airtable | source & destination | Lightweight spreadsheet-database |

---

## Repository layout

```
.
├── README.md            ← you are here (methodology + index)
├── system-catalog.md    ← the 10 mock services the prompts target
└── prompts/
    ├── 01-airtable-to-twilio-welcome-sms.yaml
    ├── 02-commcare-to-http-warehouse.yaml
    ├── …
    └── 10-kobo-primero-humanitarian-intake.yaml
```

Files are numbered roughly easy → hard. The number is just a stable id; the
`difficulty` field is authoritative.

---

## Prompt file schema

Each file in `prompts/` is a YAML document with exactly these fields:

| Field | Type | Meaning |
|-------|------|---------|
| `name` | string | Short human-readable title for the workflow. |
| `difficulty` | `easy` \| `moderate` \| `hard` | Expected difficulty for the assistant (see rubric below). |
| `n_steps` | integer | Expected number of job steps in a correct workflow (excluding the trigger). |
| `n_systems` | integer | Number of distinct external systems the workflow connects to. |
| `prompt` | string (block scalar) | The exact natural-language prompt to paste into the OpenFn AI assistant. This is the **only** thing the assistant sees. |
| `evaluate` | string (block scalar) | The tester's rubric: the story, the expected workflow shape, and concrete PASS / FAIL criteria. For prompts 04 through 10 these are scored against a single **run receipt** the workflow posts (see below); the terse prompts (01–03) are scored on their one observable outcome. |

`n_steps` and `n_systems` are *expectations*, not hard requirements — a solution
that solves the story correctly with a different step count can still pass. They're
there to flag when the assistant wildly over- or under-builds. The final run-receipt
step and the Generic HTTP receipt sink are a fixed harness convention and are **not**
counted in `n_steps` / `n_systems`, which describe the business shape of the
integration.

### Example

```yaml
name: Airtable New Beneficiary → Welcome SMS
difficulty: easy
n_steps: 2
n_systems: 2
prompt: |
  When a new beneficiary is added to our Airtable, text them a welcome message.
evaluate: |
  Story: ...
  Expected shape of a correct workflow: ...
  PASS if: ...
  FAIL if: ...
```

---

## Difficulty rubric

| Level | Rough shape | Typical signals |
|-------|-------------|-----------------|
| **easy** | 2 systems, ~1–2 steps, little/no transformation, no branching. | Straight source → destination passthrough or a single notification. |
| **moderate** | 2–4 systems, ~2–5 steps. Real field mapping/aggregation **or** one branch. | Knowing a system's data model (e.g. DHIS2 tracker), one conditional, or an aggregation. |
| **hard** | 3–7 systems, ~3–8 steps. Heavy transformation **and/or** branching, ID threading across steps, or exception handling. | Multi-entity spec-to-spec mapping, fan-out with cross-referenced ids, PII minimization, conditional escalation paths. |

---

## The 10 prompts

| # | Name | Difficulty | Steps | Systems | Exercises |
|---|------|:---:|:---:|:---:|-----------|
| 01 | Airtable New Beneficiary → Welcome SMS | easy | 2 | 2 | vague prompt |
| 02 | CommCare Form Submissions → Data Warehouse (HTTP) | easy | 2 | 2 | passthrough |
| 03 | Weekly DHIS2 Registration Summary → Email | easy | 2 | 2 | scheduled trigger |
| 04 | CommCare Pregnancy Registration → DHIS2 Tracker | moderate | 2 | 2 | transformation |
| 05 | Kobo WASH Survey → DHIS2 Aggregate Report | moderate | 3 | 2 | aggregation + guard |
| 06 | Primero High-Risk Case → Referral + Alerts | moderate | 5 | 4 | branching |
| 07 | OpenMRS Encounters → FHIR HIE (transaction bundle) | hard | 3 | 2 | heavy transformation |
| 08 | FHIR Observation Bundle → DHIS2 Events | hard | 4 | 3 | transformation + branching |
| 09 | New Beneficiary Onboarding across 6 systems | hard | 6 | 6 | fan-out + id threading |
| 10 | Humanitarian Protection Intake (GBV branch) | hard | 8 | 7 | everything at once |

Coverage across the set: system counts span 2→7; six prompts require data
transformation between specs (04, 05, 07, 08, 09, 10); four require branching logic
(06, 08, 09, 10); prompt style ranges from one terse sentence (01, 02, 03) to long,
detailed multi-paragraph specs (09, 10). Every prompt has a single clear
observable outcome to check.

---

## Running an evaluation

This is the repeatable loop. Run it per prompt (or batch across all ten).

1. **Stand up the mocks.** Bring up the mock servers for the systems the prompt
   uses (see `system-catalog.md` for each system's endpoints and data shapes).
   Seed any data the story assumes (e.g. existing Airtable rows, a Kobo asset with
   submissions, OpenMRS encounters in the last 24h) **and the named targets the
   prompt references** (programs, org units, assets, tables) so they resolve by name
   at runtime. The fixtures appendix in `system-catalog.md` lists these per system.
2. **Configure OpenFn credentials.** Point each adaptor's credential at the mock base
   URLs and set the connection facts the prompt intentionally omits (tokens, the
   Airtable base id, the Twilio from-number, the Mailgun domain). These are what the
   assistant assumes exist; a generated workflow reaches them via `state.configuration`.
3. **One-shot the prompt.** Copy the `prompt` field verbatim into the OpenFn AI
   assistant and let it generate the workflow. **Do not** coach it or iterate — the
   whole point is to measure one-shot quality. Save the generated workflow.
4. **Run it.** Execute the generated workflow against the mocks with a
   representative input (for branching prompts, run once per branch — e.g. a
   high-risk *and* a low-risk case).
5. **Score with `evaluate`.** For prompts 04 through 10, read the single **run
   receipt** first: `GET /api/v1/receipts` on the Generic HTTP mock returns the ids the
   workflow created and the branch it took, which settles most PASS / FAIL criteria in
   one place. When a criterion calls for it (or you want to confirm the receipt isn't
   fabricated), spot-check the referenced side effects:
   - the workflow's **final run state** (returned ids, import summaries), and
   - the **mock side effects** — records created, messages queued, request logs
     (e.g. Twilio `Messages.json`, Mailgun events, DHIS2 import summary, the Generic
     HTTP `GET /_admin/requests` log, Airtable rows).

   The terse prompts (01–03) write no receipt; score them on their single outcome.
6. **Record the result.** Note PASS/FAIL plus useful observations: did the step
   count match `n_steps`? Did it pick the right adaptors and trigger type? Where did
   it break down? Keeping these notes over time is how you compare assistant
   versions.

A prompt **passes** only if all of its PASS criteria hold and none of its FAIL
criteria are triggered.

---

## Adding a new prompt (authoring process)

New prompts should follow the same design rules the existing ten were built from,
so the set stays balanced and every prompt is scorable.

1. **Pick a story** that connects **2–10** systems from the catalog and reads like
   something a real OpenFn user would actually build. No contrived "call every API"
   busywork.
2. **Decide what it exercises.** Across the library we want a mix, so lean toward
   whatever's under-represented:
   - *sometimes* data transformation from one system's spec to another,
   - *sometimes* branching logic (conditional edges),
   - always a coherent narrative,
   - always **one clear, observable outcome** that says whether it worked.
3. **Vary the prompt voice.** Some prompts should be short and vague (forcing the
   assistant to infer systems, trigger, and mapping); others long and explicit.
   Don't make them all the same length.
4. **Write the `evaluate` block against observable mock output.** State the story,
   the expected workflow shape, then concrete PASS / FAIL bullets that a tester can
   check by looking at created records, queued messages, and request logs — not
   vibes. For branching prompts, give the expected outcome for *each* branch. If the
   workflow touches more than one or two systems, have it post a run receipt to
   `/api/v1/receipts` carrying the ids it creates and the branch it takes, and score
   against that receipt so verification stays one-place. Keep the receipt small and
   made of runtime-produced ids, never ids you hard-code into the prompt.
5. **Set the metadata.** `difficulty` per the rubric above, `n_steps` = expected
   job steps, `n_systems` = distinct systems.
6. **Name and number the file** `NN-short-slug.yaml` in `prompts/`, keeping the
   loose easy → hard ordering.
7. **Validate** that it parses and matches the schema before committing:

   ```bash
   python3 - <<'PY'
   import glob, yaml
   REQUIRED = ["name","difficulty","n_steps","n_systems","prompt","evaluate"]
   for f in sorted(glob.glob("prompts/*.yaml")):
       d = yaml.safe_load(open(f))
       assert set(d) == set(REQUIRED), (f, set(d) ^ set(REQUIRED))
       assert d["difficulty"] in {"easy","moderate","hard"}, f
       assert isinstance(d["n_steps"], int) and isinstance(d["n_systems"], int), f
   print("ok")
   PY
   ```

8. **Update the index table** in this README.
