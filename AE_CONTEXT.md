# HMF ACTIN adverse events — openEHR template (EMC CDR PoC)

Hand-over and context file. Read this first in a new session.
Status as of 2026-11-05: **model agreed internally (HMF), not yet agreed with EMC. Template not yet built.**

## 1. What this is about

- **The PoC.** EMC (Erasmus MC) and HMF are running a joint PoC. Structured patient data is stored on the EMC side in EHRbase, an openEHR CDR.
  - **Data flow:** EMC sends raw data → ACTIN Clinical (HMF) curates and structures it → the structured output goes back to EMC → EMC stores in the CDR only what it lacks in structured form.
  - **ACTIN does not read from the CDR.**
- **What is already done.** HMF's first deliverable is the radiology lesions template: a local archetype `CLUSTER.imaging_exam-hmf_lesion.v1` plus a template and OPT. It lives in `/Users/deniskumar/hmf/repos/test-ehr`. Full PoC context is in `EMC_EHRBASE_POC_CONTEXT.md` in the actin-clinical main checkout (`/Users/deniskumar/hmf/repos/actin-clinical`).
- **Where SAE comes in.** EMC's data-point inventory has an **"SAE" row** assigned to Hartwig, with no fields defined. This repo defines what HMF will deliver for it.
- **Why HMF wants it anyway.** Extracting and structuring adverse events is something HMF wants in actin-clinical in general, not only for this PoC.
  - How actin-clinical will extract the data is **out of scope here**. This repo is only about the openEHR model (template).

## 2. Terminology (short)

- **AE (adverse event):** any unwanted medical occurrence during treatment, whether or not it is caused by the treatment.
- **SAE (serious adverse event, ICH E2A):** an AE that meets at least one of these criteria:
  - results in death;
  - is life-threatening;
  - requires or prolongs hospitalisation;
  - causes persistent disability;
  - causes a congenital anomaly;
  - is another important medical event.
- **Serious ≠ severe.** Severity is the **CTCAE grade** (1–5). A grade 3 event is not automatically an SAE, and a grade 2 event with hospitalisation is one.
- **In trials,** SAEs are reported on a separate form (CRF/sponsor) outside the hospital EHR. The EHR only contains indirect traces of them.

## 3. What the source data looks like (EMC "Naslag" feed)

The file analysed was `ACTIN_Data_Naslag.tsv`, a pseudonymised EMC feed file in the ACTIN runtime feed folder. It holds about 2.8k rows for 38 patients.

- **Columns:** `subject, authored, section_title, section_text, content_type, type_display, type_code`.
- **content_type:** always "Medisch verslag" (consult notes).
- **type_display:** mostly outpatient follow-up and phone consults, plus clinic, "Research", SEH and others.
- **section_title:** Samenvatting, Anamnese, Aanvullend onderzoek, Conclusie, Lichamelijk onderzoek, Overweging, and others.

Findings:

- **AEs exist only as free text.** There is no structured AE/SAE field anywhere.
- **The word "SAE" appears only indirectly,** for one trial patient, as a doctor's note that an SAE still has to be reported. The formal SAE report lives elsewhere.
- **AEs are often written in a near-CRF pattern, especially for trial patients:**
  - `AE: vermoeidheid graad 2, gerelateerd aan gemcitabine`
  - `slaapproblemen, graad 1, gerelateerd aan medicatie`
  - `… niet gerelateerd aan behandeling` / `… not related`
  - `neutropenie graad 3`, `misselijkheid graad 2`, `HVS graad 1`, `Perifere sensore neuropathie gr. 2`
- **Less often:**
  - action taken (`dosisreductie`, `uitstel`, `gestaakt`);
  - outcome (`diarree hersteld`);
  - hospitalisation (`Opname i.v.m. …`).
  These are usually not written in the same sentence as the AE.
- **The same AE is repeated across many notes,** because Samenvatting text is copied from visit to visit.
- **Data-quality warning.** Pseudonymisation is incomplete. One row contained an unmasked date of birth inside an embedded pathology header, and one note contained a treating doctor's surname. The user was advised to report this via Hartwig Incident Management.
  - **Do not copy raw note text into this repo** beyond short non-identifying phrases.

## 4. Decisions (HMF internal, 2026-11-05)

1. **PoC model = AE, not SAE.** Seriousness is almost never stated explicitly. Inferring it (linking an "opname" or SEH visit to a specific AE across notes) is reasoning, not extraction, and would be error-prone. We tell EMC honestly that we deliver AEs with grade and suspected drug, and that seriousness is a later step if a reliable source is found.
2. **Only fields that are stated explicitly in one sentence,** so an LLM can extract them reliably. The note date comes from metadata, not from the text.
3. **Only published CKM archetypes, with no local archetype.** A local `CLUSTER.hmf_adverse_event.v1` was considered, with these fields:
   - relatedness;
   - action taken;
   - outcome;
   - serious flag and seriousness criteria.

   It was **dropped for the PoC** because those fields are hard to extract. It can be added later in the same `Specific details` slot without breaking existing compositions.
4. **No explicit "no AE" record** (for example from "AE - geen"). An absent record already means "no AE recorded". No consumer needs to tell "not asked" apart from "asked, none". If that is needed later, CKM has `EVALUATION.exclusion_specific`.
5. **Root composition is `COMPOSITION.report.v1`,** the generic report. The radiology template uses `report-result.v1`, which is meant for result reports (lab/imaging) and does not fit here.
6. **Grade comes from `CLUSTER.ctcae.v1`,** not from problem_diagnosis "Severity". CTCAE is the oncology standard, and the cluster is designed to sit in problem_diagnosis "Specific details".
   - Severity only has Mild / Moderate / Severe, so it cannot hold grades 4–5.
   - The cluster is 1:1 with its AE entry (occurrences 0..1). It is a group of fields (Term + Grade), not a list.
7. **Add the ctcae cluster only when the event is a CTCAE term and a grade is stated.** ctcae.v1 misuse says: "Not to be used … outside of the CTCAE classification". A non-CTCAE event (for example "gastroenteritis", which a doctor noted they could not find in CTCAE) is recorded with the name only.
8. **Alternatives were compared and rejected** (see §4a).

Fields considered and dropped for the PoC, with the reason:
- seriousness and seriousness criteria: rarely explicit, needs cross-note inference;
- outcome: rare;
- action taken with drug: rare, often in another sentence;
- onset and resolution dates: rarely stated;
- graded relatedness scale (WHO-UMC 5-step): the data only has related / not related;
- MedDRA coding: needs a licence.

## 4a. Alternatives considered (2026-11-05)

**How to read "misuse" in CKM.** The misuse text is modelling guidance, not a technical restriction; EHRbase accepts any valid composition. Its purpose is that the same concept always lands in the same archetype. Three openEHR terms are easy to confuse:

- **"adverse reaction"** (`EVALUATION.adverse_reaction_risk`) means allergy or intolerance: the propensity to react when re-exposed to a substance.
  - problem_diagnosis misuse says "not for adverse reactions, allergies or intolerances". That line refers to this concept, not to oncology toxicity.
- **"adverse event"** in the misuse of `CLUSTER.adverse_reaction_event` means a patient-safety incident: maladministration, wrong dose, mislabelling, harm from a procedure.
- **Oncology AE / toxicity** is neither of these. The archetype made for it is `CLUSTER.ctcae.v1`, whose "use" says to nest it in `EVALUATION.problem_diagnosis` or `CLUSTER.symptom_sign`.

Other problem_diagnosis misuse lines we checked:
- "not for symptoms as described by the individual (use CLUSTER.symptom_sign)". Our AEs are clinician assessments with a grade, not patient-reported history, so problem_diagnosis fits.
- "not for explicit absence". This matches decision 4: no "AE - geen" records.

**The options compared:**

| | **A. problem_diagnosis + ctcae** (chosen) | **B. OBSERVATION.adverse_reaction_monitoring → CLUSTER.adverse_reaction_event → ctcae** | **C. OBSERVATION.story → CLUSTER.symptom_sign → ctcae** |
|---|---|---|---|
| Intended for | a clinical problem; ctcae.v1 names it as its parent | monitoring for reactions during or after giving a substance (its "use" mentions chemotherapy) | symptoms as reported by the patient |
| AE term | Problem/Diagnosis name | Manifestation | Symptom/Sign name |
| Grade | ctcae in Specific details | ctcae (explicitly allowed in the Reaction details slot) | ctcae in Specific details |
| Suspected drug | Cause, 0..1 | Substance, **mandatory 1..1** | — |
| AE "not related to treatment" | ✅ (no Cause) | ❌ it is by definition a reaction *caused by* a substance | ✅ |
| Lab-based AE (e.g. neutropenia) | ✅ | ✅ | ❌ not a symptom |
| Nesting depth | 2 | 3 | 3 |

**Why not B**, although its name sounds closest:
- The drug is mandatory, but many notes give none ("misselijkheid graad 2"). Filling it would mean inferring the drug from the treatment history, which is exactly the hard extraction we ruled out.
- B asserts causality: a reaction caused by the substance. The ICH definitions of AE and SAE imply no causality, and the notes contain explicit "niet gerelateerd" statements.
- B would be the better fit only for a model of *drug-related* reactions with a known drug.

**Why not C:**
- Lab-based AEs do not fit.
- A graded clinician assessment is not patient-reported history.

**One trade-off of A:** "Cause" is generic ("a cause, set of causes, or manner of causation"), while B's "Specific substance" is semantically exact. EMC must accept the use of Cause for the drug (§8, question 2).

**Talking points for EMC:**
1. We deliver AEs, not SAEs. Seriousness is not written in HiX notes; SAEs are reported outside the EHR.
2. We use only published CKM archetypes: report.v1 → problem_diagnosis.v1 → ctcae.v1. This is the combination ctcae.v1 itself recommends.
3. We do not use `adverse_reaction_*`. In openEHR those are for allergies, or for reactions where the substance is mandatory, so unrelated AEs would not fit.
4. Confirm two things with EMC:
   - whether Cause may hold the suspected drug;
   - whether there is one record per event or per mention.
5. The model is extensible. Seriousness, outcome and action can be added later as a cluster in the same slot, and existing data stays valid.

## 5. Template model

Template name (proposed): **`HMF ACTIN adverse events`**

```
COMPOSITION.report.v1  (category = event)
│   context/start_time = note date; Report ID, Status, Comment = 0..0
│
└─ content: EVALUATION.problem_diagnosis.v1   [0..*]   ← one entry = one AE
     ├─ Problem/Diagnosis name ........... 1..1  ← AE term (text)
     ├─ Clinical description ............. 0..1  ← source sentence
     ├─ Cause ............................ 0..1  ← suspected drug, ONLY if text says "gerelateerd aan X"
     ├─ Date/time clinically recognised .. 0..1  ← note date (authored)
     │   all other elements and slots ..... 0..0 (where AD allows; see §7)
     │
     └─ Specific details (slot):
          └─ CLUSTER.ctcae.v1  [0..1]  ← only when a grade is stated AND the event is a CTCAE term
               ├─ Term ............ 1..1  ← AE term (same as name)
               ├─ Grade category .. 1..1  ← DV_ORDINAL 1–5 (Grade 0 not used)
               ├─ CTCAE version ... 0..1  ← usually unknown, may be left empty
               └─ Category, Grade description = 0..0
```

How the archetypes link:

- **report.v1 → problem_diagnosis:** a normal ENTRY in the composition `content`. report.v1 does not constrain content, so any ENTRY is allowed.
- **problem_diagnosis → ctcae:** via the open slot `Specific details` (at0043, `include /.*/`). ctcae.v1's own "use" text names this slot as its intended place.
- **ctcae** has Term and Grade category mandatory (1..1). That is why the whole cluster is 0..1: it is added only when a grade is known.

Field → source mapping (source = ACTIN output; extraction design not decided):

| OPT field | Source | Notes |
|---|---|---|
| `context/start_time` | note `authored` | |
| Problem/Diagnosis name | extracted AE term | Dutch text as written; a curated/CTCAE term is possible later |
| Clinical description | sentence the AE was extracted from | short, no identifiers |
| Cause | drug named in "gerelateerd aan …" | empty if not related or unknown; EMC must accept this use of "Cause" (CKM: "a cause, set of causes, or manner of causation") |
| Date/time clinically recognised | note `authored` | |
| ctcae / Term | same AE term | |
| ctcae / Grade category | "graad N" / "gr. N" / "grade N" | 1–5 |

RM attributes to set when writing (same as radiology):
- composer;
- context/setting (238 "other care");
- feeder_audit (system id and original note id);
- subject PARTY_SELF;
- ehr_id via EMC's mapping table.

## 6. Archetype files in this repo

All are copied unchanged from the openEHR CKM mirror (`openEHR/CKM-mirror`, `local/archetypes/…`). All are ADL 1.4, published, and start with a UTF-8 BOM.

| File | Revision | CKM |
|---|---|---|
| `openEHR-EHR-COMPOSITION.report.v1.adl` | 1.3.1 | same file as in `test-ehr` |
| `openEHR-EHR-EVALUATION.problem_diagnosis.v1.adl` | 1.7.4 | https://ckm.openehr.org/ckm/archetypes/1013.1.169 |
| `openEHR-EHR-CLUSTER.ctcae.v1.adl` | 1.0.3 | https://ckm.openehr.org/ckm/archetypes/1013.1.5151 |

There is no local archetype, so there is nothing for EMC to own or namespace.

## 7. Next steps

1. **Agree the model with EMC** (§5 and §8).
2. **Build the template in Archetype Designer** (tools.openehr.org), connected to this repo:
   - New template from `openEHR-EHR-COMPOSITION.report.v1`, named `HMF ACTIN adverse events`.
   - Composition: set Report ID, Status and Comment to 0..0. Leave the Extension slot empty.
   - Add `EVALUATION.problem_diagnosis.v1` to `content` with occurrences 0..*.
   - problem_diagnosis: keep name 1..1. Set Clinical description, Cause and Date/time clinically recognised to 0..1. Set every other element to 0..0.
   - Fill the `Specific details` slot with `CLUSTER.ctcae.v1`, occurrences 0..1. Set Category and Grade description to 0..0.
   - Lessons from the radiology template:
     - empty slots cannot be set to 0..0 in AD, but that is harmless;
     - mandatory elements cannot be hidden;
     - check the saved `.t.json` per overlay, because node ids repeat across archetypes.
   - Export **OPT 1.4 (XML)**, the only format EHRbase accepts, and commit it.
3. Optional: run a local EHRbase in Docker, upload the OPT and post an example composition.
4. Later, in actin-clinical (not designed yet): extraction from Naslag consult notes. The existing model has `FeedToxicity(name, start date parts, endDate, grade, source, …)` with `ToxicityConfig` curation, which is the natural base.

## 8. Open questions for EMC

1. Does EMC mean trial **SAE** (ICH sense) or AEs/toxicity in general? We propose AE, with seriousness out of scope for the PoC.
2. Is using problem_diagnosis **"Cause"** for the suspected drug acceptable? If not, the drug stays only in the source sentence.
3. **One record per event or per mention?** The same AE recurs across notes.
   - Per mention is simple and traceable, but creates duplicates.
   - Per event needs deduplication on the HMF side.
4. **Coding of the AE term:** free text for the PoC. Do they want CTCAE term or MedDRA later? MedDRA needs a licence.
5. Is there an existing HiX/ZIB model for AEs that we should align with?

## 9. Possible later extensions (not for the PoC)

A local `CLUSTER.hmf_adverse_event.v1`, placed in the same `Specific details` slot, could hold:
- relatedness (related / not related / unknown);
- action taken (none / dose reduced / delayed-interrupted / discontinued);
- outcome (recovered / recovering / not recovered / recovered with sequelae / fatal / unknown);
- serious (boolean);
- seriousness criteria (death / life-threatening / hospitalisation / disability / congenital anomaly / other important).
