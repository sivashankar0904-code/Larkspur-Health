# Larkspur Health × Saguaro Family Medicine — Approach

**Thesis:** one channel-agnostic agent that **drafts** the *administrative and logistical* work for staff approval and never makes an *unbounded clinical judgment*. It detects clinical risk and routes it to a human fast; safety actions are immediate, never gated. Autonomy is earned intent by intent — the agent starts in draft-and-approve and graduates specific intents to auto-execution only after measured accuracy, with Saguaro's medical director accountable for any clinical protocol.

---

## Section A — Questions

| # | Question | What the answer changes |
|---|---|---|
| 1 | Saguaro runs licensed nurse triage protocols — do we integrate with that triage platform/line (does it expose an API), is AI execution of the content permitted, and will the medical director sign the agent's **intake script + red-flag routing**? | Decides whether the agent feeds the existing nurse line (preferred) or runs any disposition itself. |
| 2 | *(Privacy officer)* Can every subprocessor sign a BAA and accept no-audio-retention + no-PHI-training terms — and does the answer differ for real-time streaming vs. storage? | Decides the ASR/TTS/telephony/LLM stack and our one-line answer to "where does the audio go." |
| 3 | What Epic access will we actually be granted (on-FHIR APIs, Cadence scheduling, med list/refill, referral/prior-auth, results), via what path (App Orchard/Interconnect vs. middleware like Redox), and with what onboarding lead time? | Epic is the 8-week long pole; decides which intents are automated vs. handed to a human. |
| 4 | What is Saguaro's actual clinical escalation coverage — who answers, by hour and by state — and are those nurses and on-call physicians licensed in the patient's state? (AZ and NM are Nurse Licensure Compact members; **Nevada is not**.) | Decides whether we can escalate at all for NV patients or must contract a state-matched triage vendor, and whether we fix the "page the on-call doctor for everything" behavior. |
| 5 | Which payers/plans are in scope, and which clearinghouse gives us real-time eligibility (X12 270/271)? | Decides whether contact A's "do you take my Humana plan" is answered in-flow or handed to billing. |
| 6 | Where does prior-auth status actually live — the payer's or RBM's portal (for imaging, often eviCore/Carelon), a clearinghouse 278 feed, or Epic's own copy — and who owns follow-up when it stalls? | Decides whether the agent can report status at all, or only read Epic's (often stale) copy and open a coordinator task. |
| 7 | Does Saguaro already contract a qualified medical interpreter service (Section 1557 requires language access), which clinics actually have bilingual staff, and is AI Spanish acceptable for administrative content vs. clinical? | Decides the ASR/TTS choice, when we bridge a live interpreter, and the Spanish containment target. |
| 8 | What identity factors may we use (DOB, caller-ID match, address, last-4 SSN), and what may we disclose to a verified caller, an unverified number, and a proxy/representative? | Decides contacts C and D and the PHI guardrails for the whole agent. |
| 9 | Are we permitted to send PHI by SMS at all, and what TCPA/HIPAA consent do we hold? | Decides whether the text channel can do anything clinical (contact D) or stays administrative. |
| 10 | Does Saguaro have a written refill protocol — which drugs, who signs it, and can a nurse/MA process it under standing order in AZ, NV, and NM? | Decides the refill drug list, who authorizes, and the containment ceiling; without it, refills only take a message. |
| 11 | Can Saguaro rank its 8 Arizona clinics on call volume, peak hold time, after-hours leakage, Spanish share, MA share, Epic config, and — above all — front-desk willingness to co-design? | Decides which clinic is the pilot; it must be locked in week 1 because Epic provisioning, phone routing, and training depend on it. |
| 12 | Who carries medical-legal liability for agent actions, and what state-specific rules apply across AZ/NV/NM (recording consent, MA rules)? | Decides consent flows, recording disclosure, and guardrail strictness. |

---

## Section B — Assumptions

| Assumption | Why | If wrong |
|---|---|---|
| All subprocessors sign BAAs; audio is not retained by default; no PHI is used for training; only consented, redacted QA samples are kept with a short TTL. | Standard for HIPAA-eligible vendors; directly answers the privacy officer. | Self-host ASR/LLM or restrict the agent to non-PHI intents — a phone-channel launch blocker. |
| We get Epic *read* access plus at least one write path (scheduling and/or task creation) within 8 weeks, likely via middleware (Redox) if direct Epic onboarding is too slow. | The front desk lives in Epic. | Ship a narrower agent that takes structured messages, creates tasks outside Epic, and escalates — no autonomous booking. |
| A **state-matched** clinical escalation destination exists or can be contracted (24/7 nurse triage + on-call) that accepts a warm transfer with a structured summary. | Contact B requires it; practice occurs where the patient is, and NV is not in the Nurse Licensure Compact. | Red flags get 911/ED guidance plus a structured page to on-call, and NV coverage must be staffed with NV-licensed RNs or bought from a triage vendor. |
| Saguaro's licensed nurse triage protocols remain the clinical engine; the agent feeds them and never produces a disposition, and the medical director signs a lighter **intake script + red-flag routing**. | Saguaro runs licensed nurse triage protocols; RN judgment cannot be delegated to software. | If the triage platform can't be integrated and no one will sign an intake script, the agent ships red-flag detection + warm transfer with no structured intake. |
| Saguaro has, or will sign, a refill protocol naming eligible drugs and permitting nurse/MA processing under standing order where state law allows. | The whole refill workflow depends on it; the agent cannot authorize a refill. | Refills degrade to message-taking + a prescriber task; containment drops, but nothing unsafe ships. |
| The agent handles Spanish natively (not machine translation) for administrative intents, and a qualified interpreter is bridged in for clinical/PHI content and for any transfer to an English-only clinician. | ~30% Spanish; most clinics lack bilingual staff; Section 1557 requires language access. | All Spanish routes to bilingual staff or a live interpreter; Spanish containment drops to ~0 and after-hours Spanish is unsupported. |
| Caller-ID matching an on-file number + DOB (+ one factor) is enough for administrative PHI; clinical results and proxies need stronger checks. | Contact D. | Stricter verification and more handoffs. |
| "Humana plan" is verifiable via 270/271, and we may update registration but never guarantee coverage. | Contact A. | Route all insurance questions to billing. |
| Saguaro's results-release policy governs contact D; texting PHI requires explicit prior consent we do not have. | HIPAA/TCPA. | Add a consented, identity-verified texting flow — still verify first. |
| The 8-week pilot is one Arizona clinic, after-hours + overflow first, with a bounded intent set. | Fastest path to value, lowest blast radius. | If full 24/7 across all intents is expected: renegotiate scope and start in shadow mode. |
| The existing answering-service contract can be changed or run in parallel. | We need a fallback. | Run in shadow with human override until trust is earned. |
| Epic is the system of record; we do not create a shadow chart. | Avoid divergence and duplicate records. | We would need a sync/reconciliation layer. |

---

## Section C — Solution design

**Diagram:** [architecture.svg](./architecture.svg)

**In one sentence:** a single orchestration core with three channel adapters (phone, SMS, portal chat) that resolves identity and language, classifies intent, **drafts** administrative actions for staff approval through Epic and payer integrations, and hard-stops into a human whenever the request is clinical or the caller is unverified.

### Layers (see diagram)

1. **Channels** — Phone (Twilio Programmable Voice; SIP onto Saguaro's existing numbers), SMS (Twilio, 10DLC/toll-free), Portal chat (embed/webhook). Adapters normalize every channel to one internal event schema; voice streams audio to ASR, SMS/chat are text-native.
2. **Orchestration core** — Identity & consent, language detection + routing, intent classifier, **policy/guardrail engine**, conversation state, and an audit logger on every turn.
3. **Tool layer** — Epic (FHIR + Cadence + refill/referral/results), a HIPAA-eligible LLM, payer eligibility (270/271), prior-auth status, scheduling, task creation.
4. **Human layer** — warm transfer to front desk / nurse triage / on-call, structured Epic task handoffs, and 911/ED guidance.

### One core, three channels — what is shared, what differs

Shared, because it must be identical on every channel — otherwise patients learn to channel-shop (e.g., that SMS leaks results the phone won't):

| Shared (the orchestrator) | Channel-specific (the adapter only) |
|---|---|
| Identity/verification tiers; language policy (native EN/ES + interpreter bridge); intent taxonomy; clinical guardrails (red-flag detector, clinical gate); the protocol library; consent/disclosure rules; the tool layer (Epic, eligibility, prior auth, scheduling, tasks); audit + QA | Transport and modality; latency/UX (barge-in, DTMF, message segmentation); SMS consent and TCPA quiet hours; escalation mechanics (warm transfer on voice vs. callback/task on SMS and chat); portal-only content (links, attachments); the verification starting point (portal begins authenticated; SMS/phone must verify) |

**Policy is enforced twice: in the orchestrator and again at the tool boundary.** The PHI gate sits in front of the Epic/PHI tools themselves, so no channel can reach PHI or a clinical action by bypassing the orchestrator. Conversation state is shared across channels (a chat can continue on a call), but verification does not carry over silently — identity is re-established or stepped up on the new channel. If the orchestrator is unavailable, the phone channel fails safe to a human/backup answering service and never drops the caller.

### The LLM talks; the policy engine decides

The model is a language layer with **no authority**. Every state-changing action is a **typed tool call** — a schema-validated function with declared preconditions — and the runtime executes it only after the **deterministic policy engine** approves. Flow: the LLM emits an intent + tool call with arguments → the orchestrator validates it against the tool schema → the policy engine evaluates runtime state (verification tier, channel, patient, protocol version, consent) → allow / deny / step-up / escalate → the tool executes → the result, PHI-filtered to the caller's tier, returns to the LLM to phrase.

Why deterministic: safety-critical gates must be reproducible, testable, and auditable — you cannot certify a probabilistic gate. The model's confidence can *feed* the gate (low confidence → escalate), but the threshold is a rule, not a sample.

| Layer | Decides | Never decides |
|---|---|---|
| LLM | language, disambiguation, phrasing, EN/ES, empathy, handoff summaries, *proposing* actions | identity tier, clinical disposition, whether a tool may run, what PHI to release |
| Policy engine (rules-as-code, versioned) | allow / deny / step-up / escalate per action and tier | how to phrase anything |
| Typed tools | execute one action; declare required tier + side-effect class; redact results | anything the engine did not approve |

Consequences: a prompt injection or a coercive caller can at most *propose* a tool call the engine rejects — untrusted input cannot escalate privilege, because authority sits outside the model. The model holds no Epic/PHI credential; only the gated tools do, so a hallucinating or compromised model cannot act. The cost is honest: rules need maintenance, and edge cases get denied or escalated rather than "handled gracefully" — the right trade for clinical safety.

### Red-flag detection runs on every turn

The red-flag detector is not a branch of the intent classifier — it runs on **every turn of every contact, on every channel, in the caller's own language**, in parallel with intent handling. Intent classification is fallible, and a red flag routinely appears inside an administrative call (a refill request that mentions chest pain; a scheduling chat that mentions worsening breathlessness — contact B). Waiting for a "clinical" intent would miss exactly those cases.

- **Recall over precision.** Tuned to over-fire: a false positive costs a nurse's time; a false negative costs a patient.
- **Ensemble: deterministic rules + a model, either can fire.** Fast keyword/rule patterns (chest pain, can't breathe, suicidal, bleeding, worst headache) OR a model that catches paraphrase, negation, and code-switching. Rules are auditable; the model covers what rules miss.
- **It can only escalate — never reassure or downgrade.** Once it fires, the contact is in escalation state and a later reassuring turn cannot clear it.
- **It overrides both the flow and the model.** Firing mid-refill or mid-booking stops that flow; the LLM cannot talk past it.
- **It logs every evaluation** (rule/model version + outcome) for audit and tuning.

Action ladder: **severe → 911/ED immediately; moderate → warm transfer to the nurse line with the collected intake; low → flag for clinician review and continue.**

### Vendor choices that matter

- **Telephony/voice: Twilio** — BAA, streaming, and SIP into the existing PBX so we answer on Saguaro's real lines without a phone-system replacement.
- **ASR/TTS: Deepgram** — BAA, low-latency streaming, strong Spanish; real-time voice needs streaming *with* a BAA, and Spanish quality is a hard requirement.
- **LLM: a HIPAA-eligible hosted model under BAA (Azure OpenAI or AWS Bedrock)** — PHI must not leave a BAA boundary.
- **Epic integration: Epic on FHIR + Interconnect if accessible in time; otherwise Redox as middleware** — Epic is the system of record and the critical path.
- **Eligibility/prior auth: a clearinghouse (Availity/Change)** — avoids building dozens of payer integrations.
- **Interpreter fallback: a live interpreter service** for clinical/PHI Spanish and other languages — machine translation is not acceptable for clinical content.

### Prior authorization — where the status actually lives

Prior-auth status is not one record; it is a stack of layers, and only one of them is truth.

| Layer | Where it lives | Truth? | Agent can read it? | Refresh |
|---|---|---|---|---|
| 0 — Order | Epic order / referral | intent only | yes | at order |
| 1 — Request | clearinghouse / ePA vendor (X12 278R + 275) | that a request exists | sometimes | at submit |
| 2 — **Decision** | the payer's UM system or its delegate — imaging → RBM (eviCore, Carelon MBM); drugs → PBM/Surescripts | **the only truth** | rarely: 278I inquiry, FHIR PAS (2027+), payer/RBM portal | payer-driven |
| 3 — Response | letter, fax, portal message, 278 response | a record of the decision | via the coordinator | on decision |
| 4 — Clinic copy | Epic referral/authorization record + workqueue | best effort | **yes** | manual or 278 interface |
| 5 — Patient view | MyChart, letters, the front desk's answer | least reliable | n/a | lagging |

Truth is Layer 2; the agent reads Layer 4; **the gap between them is contact C.** For a study ordered two weeks ago, Epic (Layer 4) most likely says "pending" or nothing while the real status sits at the RBM (Layer 2).

**Design consequence:** the agent reports only Layer 4, framed as "what our records show," and says **approved** only when a documented authorization number exists (a Layer 3 artifact); otherwise it opens a coordinator task and sets a callback expectation. It does **not** scrape payer/RBM portals at launch and never interprets a denial. The forward path is standards-based: CMS-0057-F requires MA, Medicaid/CHIP, and QHP payers to expose a FHIR Prior Authorization API (Da Vinci PAS/CRD/DTR) by **January 1, 2027**, with 72-hour (expedited) / 7-day (standard) decision clocks. We build to X12 278 and the FHIR PAS feed where Saguaro's clearinghouse exposes them, and keep a human for the long tail.

### Where does the audio go? (the privacy officer's question)

Audio is streamed in real time to a BAA-covered ASR for transcription and then **discarded** — not stored by Larkspur or the ASR by default, and never used to train models. Only the structured outcome (intent, action, disposition) is logged; transcripts are redacted and, if the patient consents, kept for a short TTL for QA. Everything is encrypted in transit and at rest, access is role-based and audited, and ASR can run in a Saguaro-dedicated, region-pinned tenancy if required. **One-line answer:** *"Streamed to a BAA vendor for transcription, not retained, never used for training; stored only with consent and a short retention window."*

### What it does alone vs. hands to a person

| Agent answers directly (read-only) | Agent drafts → staff approve (state change) | Agent acts immediately (safety, never gated) |
|---|---|---|
| Identity verification; language; hours/location; accepted-plan list; prior-auth **status** read (Layer 4), framed as "what our records show"; general new-patient intake questions | Appointment book/move/cancel; routine refill intake; insurance/registration updates; prior-auth follow-up; new-patient registration; post-discharge follow-up scheduling; triage intake routed to the nurse (the RN decides disposition) | Red-flag escalation (911/ED, warm transfer to the nurse line); emergency guidance |

**Draft-and-approve is the starting posture.** The agent never executes a state change; it produces a proposed action in a staff queue, and a human approves it — so the **approver is the actor of record**, which is the liability posture we want at launch. The deterministic policy engine decides not only *what may be drafted* but *who must approve* (front desk vs. nurse vs. prescriber). The approval UI shows the draft, the patient's verbatim source, and the confidence, so the human reviews rather than re-works. Approvals can be **synchronous** (staff approve while the patient waits — best UX for scheduling during business hours) or **asynchronous** (draft queued; patient gets a callback/text). Throughput is then capped by approval capacity, so the win is *seconds to approve a draft* versus *minutes to handle a call*, not zero staff. **Safety is never gated:** a red flag or emergency acts immediately.

**Saguaro runs licensed nurse triage protocols, so the agent integrates with them rather than reimplementing them.** Those protocols assume a licensed RN's assessment and judgment, and licensed content (e.g., Schmitt-Thompson) often cannot be executed by software — so the agent's clinical role is a clinician-signed **intake script and red-flag routing**: it collects the structured intake, flags red flags, and hands the patient to the RN who runs the protocol and makes the disposition. The agent never says whether the patient needs to be seen. This is lower risk, faster to ship, and sidesteps the content-license problem by using the vendor's own triage platform. The remaining questions are whether that platform exposes an API and whether the medical director will sign the intake script and routing.

### Refill protocol — the shape, and the drugs

A refill consumes authorizations remaining on an existing prescription; a renewal creates a new one — and most chronic "refill" calls are renewals. **The agent never prescribes or authorizes; it screens and routes, and the authorization is a licensed clinician's standing order signed by Saguaro's medical director.** Which drugs qualify is Saguaro's protocol, not ours, and it must be confirmed against AZ/NV/NM scope-of-practice rules; the list below is the default we would propose.

| Tier | Examples | Agent behavior |
|---|---|---|
| Protocol-eligible (stable, own prescriber, unchanged) | lisinopril, losartan, amlodipine, HCTZ, metoprolol, atorvastatin, rosuvastatin, metformin, levothyroxine, omeprazole, cetirizine, albuterol | Screen against the criteria → queue to the nurse/MA standing order or prescriber; confirm to the patient |
| Provider review (not auto) | warfarin/DOACs, insulin, psychotropics, meds with overdue monitoring (K+/Cr, eGFR, TSH), inhalers for uncontrolled asthma, dose changed or newly started <90 days, outside-prescriber meds | Create a prescriber task; set expectations; never promise |
| Never auto (hard stop) | Schedule II (oxycodone, amphetamines); Schedule III–V (benzodiazepines, zolpidem, tramadol, testosterone); gabapentin where state-controlled; buprenorphine/MAT; any do-not-refill or abuse flag | Route to prescriber; controlled substances require EPCS and a licensed prescriber |

Screening criteria (all must pass): identity verified; active med with a Saguaro prescriber of record; dose/route/frequency unchanged; not controlled; refills remaining or a protocol renewal; not "too soon" per last fill; last visit within the protocol window; no overdue monitoring for that drug class; no new symptoms, side effects, or pregnancy; no allergy/interaction flag. Any failure → prescriber task; any clinical report → escalate (never refill through it).

**Escalation mechanics:** warm transfer (voice) with a structured handoff (identity, intent, what the agent said/did, urgency); Epic task/queue item for async work (refills, prior auth, registration updates); after-hours clinical → nurse triage/on-call, never a bare page; emergencies → 911/ED guidance plus a care-team notification.

**Clinical escalation coverage — who answers, by hour and by state.** The agent never *owns* a clinical escalation; a licensed human does, and which human depends on the hour and the patient's state, because nursing/medical practice occurs where the *patient* is, not where the clinician sits. The agent resolves the escalation target at runtime from the patient's state.

| Window (patient-local) | Who owns the escalation | Licensure rule |
|---|---|---|
| Business hours | Patient's own clinic front desk → clinic RN/provider | Licensed in the patient's state |
| Evenings / weekends / holidays | After-hours nurse triage resource | AZ & NM: multistate (compact) or state license. **NV: NV single-state license required** (NV is not an NLC member). |
| Overnight | Same triage resource → on-call physician, paged with a structured summary | On-call physician licensed in the patient's state (confirm IMLC participation) |
| Any hour — emergency | Agent advises 911/ED directly and notifies the care team | None |

Saguaro spans **three time zones** — AZ is MST year-round (no DST), NV is Pacific, NM is Mountain — so "after 5pm" and the overnight window differ by up to two hours and must be resolved in patient-local time. Note the hidden landmine in today's setup: the on-call physician who is paged for everything must *also* be licensed where the patient is. **Design consequence:** if Saguaro cannot staff or buy NV-licensed coverage, the NV red-flag path degrades to 911/ED guidance plus a structured page to on-call — safe, but it does not fix the Las Vegas problem. This is one more reason to pilot in Arizona first.

**Identity verification — risk-tiered, matched to what is disclosed.** Verify once per purpose, and step up mid-contact if the purpose changes. Caller-ID is a signal, never authentication on its own; DOB alone is not enough.

| Tier | In scope | Required to proceed |
|---|---|---|
| 0 — public | hours, location, accepted-plan list, new-patient status, 911 advice | none |
| 1 — low-sensitivity admin | book/move/cancel, refill intake, prior-auth status | caller-ID matches an on-file number **AND** name + DOB + one more (on-file phone or address) |
| 2 — PHI | results, clinical info, member-specific benefits | **one-time passcode to the on-file mobile** (Twilio Verify) **plus** DOB, or an authenticated portal session |
| 3 — high-risk | record release, changing phone/PCP/insurance on file, controlled substances, payments | OTP **+** knowledge-based verification against an authoritative source (LexisNexis/IDology class), or a human / portal re-auth |

Channel notes: voice has no "something you have" except an OTP to the number on file or a callback; portal chat's authenticated session *is* the verification (but an authenticated patient is not an authenticated proxy); SMS from an on-file number is possession, and an unknown number **fails closed**. Cross-cutting rules: never confirm or deny that someone is a patient (anti-enumeration — contact D's trap); proxy is authority, not identity, so contact C checks the on-file personal representative before any disclosure; the **emergency exception** means verification never blocks a safety escalation (contact B) — advise 911, route to a clinician, disclose the minimum; on failure, retry once, then offer portal / callback to the on-file number / in-person, then a human, logging and rate-limiting attempts and never revealing which factor failed. Deliberately avoided at launch: voice biometrics (biometric-privacy statutes plus false-accept risk), SSN-only, DOB-only.

### Language — who speaks Spanish

Three different things get conflated: the agent being bilingual, a bilingual staff member, and a live interpreter. The design uses all three for different jobs.

| Job | Who does it |
|---|---|
| Administrative Spanish (scheduling, refills, PA status, plan questions) — the bulk of the 30% | The agent, **natively**: Spanish ASR/NLU/generation (Deepgram + the LLM), **not** English→Spanish machine translation, which compounds errors |
| Human-handled tasks at clinics that have bilingual staff, business hours | Bilingual front-desk/MA staff — a routing option, not the language strategy, since most clinics lack them and none are on after hours |
| Clinical/PHI content, or any warm transfer to an English-only clinician | A **qualified medical interpreter**, bridged into the live call so clinician, patient, and interpreter are all on the line |

The interpreter vendor is a real choice: **LanguageLine Solutions** (or Saguaro's existing OPI contract — Section 1557 requires language access for a clinic taking Medicare/Medicaid, so they likely have one) for over-the-phone interpretation, medically qualified and under BAA; ASL/VRI is a separate ADA service. Twilio conferences the interpreter into the call. Rules: never use a family member as interpreter; log the language used and the interpreter session for audit; escalate to a human interpreter on low ASR confidence, accent/dialect trouble, or any request for a person; and handle two-party recording consent when an interpreter joins.

**Language is detected from the caller's first utterance, not an IVR menu** — which means the greeting itself must be bilingual, because you cannot detect language before the patient speaks. Detection then runs continuously (code-switching, a mid-call switch, or a proxy), the session language persists and is shared across channels, and it is never inferred from area code or surname (contact E's 612 number says nothing). Epic's preferred-language field is a prior, not a gate — often stale. On low confidence, fall back to the bilingual prompt or a human; never guess inside a clinical call. The session language is logged and handed to the interpreter and any transfer.

### How contacts A–E move through it

- **A — Las Vegas phone, Mon 7:52am (refill + insurance change).** Verify via caller-ID-on-file + DOB. Refill: lisinopril is protocol-eligible only if it is active, unchanged, non-controlled and the patient's K+/Cr and last visit are within the protocol window → **draft** the refill for the nurse/prescriber queue and tell the patient the care team will confirm; if monitoring or a visit is overdue, or the dose changed recently, it becomes a prescriber task. No clinical advice. Insurance: recognize the plan-change intent, run 270/271; if the Humana plan is accepted, **draft** the registration update and say what to bring; if ambiguous/out-of-network, hand to billing. Never guarantee coverage or cost. *Hands off:* registration → front desk; approval → prescriber. A high-volume Monday call — a strong automation win.
- **B — Portal chat, Thu 11:40pm (post-discharge heart failure, worsening).** Classify as a **clinical red flag**: worsening dyspnea + edema within a day of HF discharge can be decompensation. The agent does **not** answer "should I be worried." It asks 2–3 closed safety questions (full sentences? chest pain? fainting?), then warm-transfers to the after-hours nurse triage/on-call clinician; on any severe sign it advises calling 911 now. It opens a high-priority task and flags the patient for the 2-business-day post-discharge follow-up. *Guardrail:* the archetype of "never alone" — if no clinician answers, default to 911/ED guidance, never reassurance.
- **C — Tucson phone, Tue 2:15pm (Spanish prior-auth status).** Detect Spanish and handle the administrative request end-to-end in native Spanish (not machine translation); if it turns clinical, bridge a qualified interpreter into the call. Authorization: it concerns her husband → check the on-file authorized representative; if absent, do not disclose — offer to add authorization or have him call. Status: report Epic's copy (an approval is stated only when a documented auth number exists); a two-week-old study with no recorded approval → flag as stalled and create a referral-coordinator task; denied → route to the coordinator, do not interpret. *Hands off:* coordinator for stalled/denied; interpreter if clinical.
- **D — Text, unknown number (requests A1C).** Unknown number + clinical result = two hard stops. The agent does **not** disclose. It replies with a compliant message: results can't be sent by text or to an unverified number; verify identity; use the portal (offer help setting it up / sending a secure link). It does **not** confirm or deny that Dana is a patient (prevents an info leak). If policy later allows consented text PHI, still verify identity first and follow the release policy, which may require clinician release. *Guardrail:* no PHI over an unverified channel, ever.
- **E — Phone, 612 area code, Mesa 85206 (new patient, persistent cough).** New patient + out-of-area + acute symptom; do not infer location from the area code — use the stated Mesa address. Registration: capture demographics + insurance, check network; if out-of-network, explain and offer self-pay/telehealth. Clinical: screen red flags (duration, fever, shortness of breath); if concerning → nurse line/same-week, else **draft** the registration + earliest new-patient slot (or telehealth) for front-desk approval. *Hands off:* new-patient registration/records → front desk; clinical concern → nurse line. *Guardrail:* no diagnosis; scheduling is administrative.

### Safety & compliance (cross-cutting)

Every turn is logged (channel, intent, action, disposition, confidence) for audit and QA. Guardrails live in code, not just prompts: a PHI gate tied to the verification tier, a clinical-intent gate tied to the protocol library, and a red-flag detector that runs on every turn and can only escalate — never reassure. Low confidence always routes to a human; the agent never fabricates and never gives clinical advice outside protocol. BAAs with every subprocessor, encryption, RBAC, no-retention, no-training, and consent/disclosure logging throughout.

---

## Section D — Eight-week scope

**Pilot:** one **Phoenix-metro Arizona clinic**, chosen by scorecard in week 1 rather than named in advance. Arizona because 8 of 14 clinics are here, it is a Nurse Licensure Compact state (no NV single-state problem), and it avoids NV/NM variance; Phoenix-metro because it carries the seasonal Spanish/Medicare volume (the Mesa snowbird effect) and the scenario's contacts cluster there.

| Criterion | Why | Gate? |
|---|---|---|
| Front desk + lead provider willing to co-design | the single best predictor of pilot success | **yes** |
| Call volume and peak hold time | signal + relief | high |
| After-hours / answering-service leakage | matches the pilot scope | high |
| Spanish share meaningful but not dominant, with **thin** bilingual staffing | makes the Spanish path's value visible and measurable | medium |
| Medicare / MA share | exercises eligibility + prior auth | medium |
| Epic version/config = fleet standard, scheduling already in use | integration risk | high |
| Panel size mid-to-large and stable | signal without noise; bounded blast radius | medium |
| Not mid-transition (new EHR, new manager, relocation) | avoid confounds | **yes** |

Expected winner: a mid-to-large Phoenix-metro primary care clinic, Medicare-heavy, roughly 15–25% Spanish, cooperative front desk, standard Epic config. As a concrete target it looks like this: **~5,000–8,000 active patients, ~40%+ Medicare/MA, ~15–25% Spanish, standard fleet Epic with Cadence scheduling in use, a cooperative front-desk manager and lead provider as champions, and thin bilingual staffing.** The physical site name comes from Saguaro's list in week 1 — the brief names regions, not clinics. Two counterintuitive calls: a **bilingual front desk is a safety net, not a reason to pick a clinic**, and **highest volume is not automatically the winner** — the gate is staff willingness, because a hostile front desk sinks an 8-week pilot faster than a modest volume does. Tucson and the NV/NM clinics are explicitly out for the pilot. Starting scope remains **after-hours + overflow/lunch**, then the Monday-morning peak.

**Ship at the one clinic:**
1. Phone + SMS + portal chat on one core (portal chat may land mid-window if Epic work runs long).
2. Identity verification + language routing (EN/ES) with interpreter fallback.
3. Established-patient scheduling: **draft** book/move/cancel for front-desk approval (Epic).
4. Routine refill intake (active, unchanged, non-controlled) **drafted** to the provider queue.
5. Prior-auth status lookup (read-only) + coordinator task.
6. Plan-accepted lookup + eligibility check **drafted** to billing.
7. Post-discharge follow-up outbound calls within 2 business days, with **drafted** scheduling — the obligation Maya named.
8. Red-flag detection + warm transfer to nurse triage/on-call, with 911 guidance.
9. A clinician-signed **triage intake script + red-flag routing** into Saguaro's existing nurse triage line, with the post-discharge HF check-in as the first use case — the RN makes the disposition.
10. Full audit logging + a QA dashboard, and a parallel/shadow run with human override.

**Leave for later:** broad symptom triage / general "do you need to come in"; clinical results over any channel; the other 13 clinics (NV/NM rollout); new-patient registration automation and out-of-network/self-pay complexity; languages beyond EN/ES; insurance guarantees and cost estimates; full clinical parity on the text channel.

**Commit to these metrics:** drafts **approved without edit** (the real accuracy signal), time-to-approve, staff minutes saved per contact, after-hours capture, hold time, book/move success, refill turnaround, PA-stall detection, post-discharge follow-up completion, escalation correctness (sample-audited), safety events (target 0), Spanish handling, patient satisfaction. With draft-and-approve, "containment" means *prepared for approval*, not *handled without a human*.

**Top risks:** Epic access slips (mitigate: middleware + a non-Epic fallback); no clinical owner named (mitigate: ship no clinical logic); BAAs/ASR blocked (mitigate: self-host or narrow scope); over-trust (mitigate: shadow mode + hard gates).

---

## Section E — Next step

Run a 60-minute working session with Maya plus Saguaro's clinical owner and privacy officer to lock questions 1, 2, 4, and 10; have Saguaro rank its 8 Arizona clinics so the pilot site is locked in week 1; convert the eight-week scope into a signed, intent-by-intent autonomy matrix; and file the Epic/Redox access request the same day, since that is the critical path.
