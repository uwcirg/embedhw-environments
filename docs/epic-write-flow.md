# Writing CNICS PRO QuestionnaireResponses to Epic flowsheets as Observations

How a questionnaire a patient completes in DHAIR ends up as flowsheet rows in
UCSD Epic. Diagrams are [Mermaid](https://mermaid.js.org/) and render on GitHub;
edit them in place or paste into <https://mermaid.live> to preview.

The step numbers (1 to 7) mean the same thing in the first two diagrams.

## Components

Solid arrows are the runtime write path. Dashed arrows are one-time setup or
flows outside this document.

```mermaid
flowchart LR
    subgraph PREP["One-time setup"]
        direction TB
        FQ["fhir-questionnaires pipeline<br/>one Questionnaire set per Epic environment,<br/>each item mapped to a flowsheet FHIR ID"]
        REG["Epic app registration<br/>fhir.epic.com"]
    end

    DHAIR["DHAIR<br/>PRO website, cron every minute"]

    subgraph CIRG["CIRG deployment: embedhw-environments"]
        direction TB
        HAPI[("App FHIR store<br/>HAPI")]
        FM["fishmouth"]
        FBA["fhirbackendauth<br/>hosts JWK Set, gets OAuth2 tokens"]
    end

    subgraph EPIC["UCSD Epic"]
        direction TB
        BAPP["App: CNICS Backend Reader 2026<br/>backend, OAuth2"]
        WAPP["App: CNICS PRO Writer 2026<br/>patient context, EMP basic auth"]
        RAPP["App: CNICS PRO Reader 2026-03<br/>provider SMART launch"]
        FS[("Patient flowsheets")]
    end

    DHAIR -->|"1. Patient + QuestionnaireResponse<br/>basic auth"| HAPI
    DHAIR -->|"2. notify with QuestionnaireResponse ID<br/>basic auth"| FM
    FM -->|"3. $extract"| HAPI
    FM -->|"4. find Patient by MRN"| FBA
    FBA -->|"OAuth2 backend token"| BAPP
    FM -->|"5. create Observations<br/>as the patient"| WAPP
    WAPP --> FS
    FM -->|"6. copy of Observations<br/>with Epic ID"| HAPI
    FM -.->|"7. 2xx or error"| DHAIR

    FQ -.->|"Questionnaires with<br/>extraction metadata"| HAPI
    REG -.-> EPIC
    RAPP -.->|"reads QuestionnaireResponses<br/>not covered here"| HAPI
```

## Write sequence

Happy path for one completed questionnaire. fishmouth handles the whole thing
inside the one request from DHAIR, so a 2xx in step 7 means both the extract
and the write to Epic succeeded.

```mermaid
sequenceDiagram
    participant D as DHAIR (CNICS PRO)
    participant H as App FHIR store (HAPI)
    participant F as fishmouth
    participant B as "CNICS Backend Reader" Epic app
    participant W as "CNICS PRO Writer 2026" Epic app

    Note over D: Patient finishes a questionnaire.<br/>Cron picks it up within a minute.

    D->>H: 1. Write Patient and QuestionnaireResponse (basic auth)
    Note over D,H: QuestionnaireResponse.authored = time the<br/>patient last touched any answer

    D->>F: 2. Notification Bundle with QuestionnaireResponse ID (basic auth)

    F->>H: 3. QuestionnaireResponse/[id]/$extract
    H-->>F: Bundle of Observations, one per flowsheet row
    Note over F,H: HAPI does not save these.<br/>effectiveDateTime comes from authored.

    F->>H: 4. Read Patient to get MRN
    F->>B: 4. Search Patient by MRN (OAuth2 backend token)
    B-->>F: Epic Patient FHIR ID and MyChart user ID

    Note over F: Point each Observation at the Epic Patient

    loop each Observation
        F->>W: 5. Create Observation (EMP basic auth, MyChart user ID header)
        W-->>F: Created, Epic Observation ID
    end
    Note over W: Rows appear in the patient's flowsheet.<br/>No Encounter or Episode reference needed.

    F->>H: 6. Save Observations with Epic Observation ID as identifier
    Note over F,H: For playback and forensics. Can be switched off.

    F-->>D: 7. 2xx
```

## Failures and backfill

What DHAIR does around step 2, and where a write can fail.

```mermaid
flowchart TD
    CRON(["DHAIR cron, every minute"]) --> NEW["Pick up newly completed<br/>QuestionnaireResponses"]
    NEW --> FIRST{"Patient's first PRO<br/>since go-live?"}
    FIRST -->|yes| BF["Also queue that patient's earlier responses<br/>capped by FISHMOUTH_BACKFILL_MAX_PENDING_PER_PATIENT"]
    FIRST -->|no| SEND
    BF --> SEND["POST notification to fishmouth"]

    SEND --> RESP{"Response within 5 s?"}
    RESP -->|2xx| DONE(["Done: extracted and written to Epic"]):::ok
    RESP -->|"timeout or error"| TRY{"Fewer than 3 attempts?"}
    TRY -->|yes| SEND
    TRY -->|no| FAIL(["Gives up. Error is in fishmouth logs."]):::bad

    subgraph WHY["Why fishmouth returns an error"]
        direction TB
        E1["$extract failed"]
        E2["No Epic Patient found for the MRN"]
        E3["Patient has no flowsheet order yet"]
        E4["Epic already has an Observation for the same<br/>flowsheet ID, patient and effectiveDateTime"]
    end
    WHY -.-> RESP

    classDef ok stroke:#2e7d32,stroke-width:2px
    classDef bad stroke:#c62828,stroke-width:2px
```

Patients who never complete a PRO after go-live are not backfilled, at UCSD's
request.

## To confirm

Points the diagrams assume or leave open:

- **fhirbackendauth in step 4.** Drawn as the hop between fishmouth and Epic for
  the Patient search, inferred from `UPSTREAM_SEARCH_URL` in `fishmouth.env` and
  the JWK Set route in `base/docker-compose.yaml`.
- **Source of the MyChart user ID (WPR).** Drawn as coming back from the Patient
  search in step 4.
- **After the third failed attempt.** Drawn as a dead end; whether DHAIR picks
  the response up again on a later cron run is not shown.
- **Retry after a slow success.** If fishmouth takes longer than 5 s but the
  Epic write goes through, DHAIR's retry would hit Epic's duplicate rule (E4)
  and come back as an error.
- **Partial writes.** Whether some Observations from one response can land in
  Epic while others fail, and what step 7 returns then.
