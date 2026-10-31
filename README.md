# asexec-registry

> **Status: proposal.** Nothing is built yet. This README describes the plan.

> [!NOTE]
> Parts of this file are AI-assisted. Treat them with due skepticism.

A public, append-only registry of **disclosure promises** and whether they were kept.

A party (an AI lab, an assessor, or an outsider citing a public statement) signs each thing it will publish and by when. It then signs each item as it is published. The registry keeps every record and shows, per promise, what was promised, how it was promised, and what was delivered. It shows statuses only: no scores and no rankings. Judgement rests with the reader.

Signing and verification are done by [asexec](https://github.com/lwhitestone/asexec), which carries the record format: disclosure promises, an `origin` field recording how each promise was made (self-signed or extracted), deadline checks, and the in-toto export. This repo holds only the registry and its policy, and reads `asexec verify` output rather than recomputing it.

## How it works

1. **Commit.** Sign each promise as its own record with `asexec prereg`: a description, and a due time or no stated deadline.
2. **Upload.** Upload the prereg to the registry. It is logged with its upload time and can't be edited or removed.
3. **Fulfil.** When an item is published, sign it with `asexec postreg`, including the published content's digest and URL, and upload that too.
4. **Verify.** The registry runs `asexec verify` on each upload and shows the result.
5. **Read.** Each promise has a public page with its status (open, fulfilled, late, missing, no stated deadline, invalid, or amended), its verify result, the original files, and a one-line `asexec verify` command so anyone can rerun the check.

## Design principles

- **Mechanism vs policy.** asexec signs and verifies, and gains record-level features only. The registry decides which keys count, how statuses are computed, and how extractions work.
- **Everything is public.** Unlike AsPredicted, records can't stay private. Hiding commitments is the failure this registry exists to expose.
- **Append-only.** Nothing is edited or deleted. A re-upload is recorded as an amendment, next to the original.
- **Neutral.** The same published method applies to every party. Statuses only, no verdicts.
- **Hardened for practitioners.** Resistance to tampering, repudiation and registering after results are known comes before presentation.
- **Thin.** An upload form, a log, and pages. No accounts: the signing key identifies the party.

## Roadmap

| Version | Scope | Needs | Done when |
|---|---|---|---|
| **reg-0.1** (M1) | Upload form; append-only log with upload times; published key list; `asexec verify --json` on upload; a page per promise with status (open, fulfilled, late, missing, no stated deadline, invalid, amended), files and a rerun command; Anthropic worked example + planted-fault fixtures | asexec 0.4.0 | Every planted fault (missing, late, silent amendment, unpublished key, tampered body) shows the right status |
| **reg-0.2** (M2) | Published extraction rules; extractor prereg + observation postreg pattern with `origin: extracted`; seeded promises from several labs and assessors; in-toto download per record; [Minerval](https://minerval.ai) feed (optional) | asexec 0.5.0 | Several parties' promises are in, and existing in-toto tools verify the exports without asexec |
| **reg-0.3** (M3) | Parties register their own keys; co-signed records shown; self-signed and extracted promises side by side | asexec 0.5.x `cosign` | At least one party other than the maintainers uploads its own signed commitment |

The record format work (disclosure-promise fields, `origin`, statuses) lives in asexec 0.4.0, and the in-toto export in asexec 0.5.0, so the registry only reads `asexec verify` output. Registry hardening and long-term stewardship are out of scope through M3.

## Not in scope

- Judging whether disclosed content is true or adequate. Signatures protect the record, not its truth.
- Scores, rankings or verdicts.
- Deciding what anyone *must* commit to.
- Evidence content, vaults, replay and hardware attestation.

## Open questions

- Who hosts the registry and its log long-term?
- Does assessor co-signature fit real access workflows: when does the assessor countersign, before access is granted or before the run?
- How are disputed extractions resolved while staying neutral?

## Threat model: selective disclosure and pre-registration

*Draft v0.2. Readers are practitioners and assessors.*

### Summary

asexec proves **when** a commitment was signed and **that** the record has not changed since. It cannot prove **what the signer knew** when signing, or **whether the content is true**. So the defense system doesn't try to make selective behaviour impossible. It aims to make selective behaviour **visible** in a neutral public record.

The threat that matters most depends on what is being registered:

| Record type | Example | Main threat | Why |
|---|---|---|---|
| **Disclosure promise** | "We will publish dangerous-capability eval results for model X before deployment." | Selective **scope**: promising only what will look good | It's a promise, not a prediction. Knowing the result in advance doesn't break it. |
| **Eval pre-registration** | "Hypothesis H, grader G, analysis script S, run plan P." | Registering **after the fact** and the **file drawer** | It's a prediction. Its value depends on the signer not knowing the outcome. |

### System and parties

| Party | Role | Example |
|---|---|---|
| **Committer** | Signs a commitment (prereg) and its fulfilment (postregs) | A lab; an assessor signing its own plan |
| **Extractor** | Turns a public statement into an outsider-signed commitment | A researcher or watchdog group |
| **Verifier** | Runs `asexec verify` on the files | Anyone |
| **Reader** | Interprets the neutral record | Assessors, practitioners, outsiders comparing disclosures |
| **Registry (asexec-registry)** | Accepts uploads, keeps an append-only log, runs `asexec verify` on upload, and shows per-promise statuses | Host to be decided (open question) |

**What asexec checks today:** signature and key consistency; a drand floor (signed after round *R*); a Roughtime ceiling (signed before time *T*); the postreg chain; content digests.

**What it does not prove:** what the committer knew when signing; whether published content is true or complete; whether the committer made other commitments it never revealed (unless a registry is added, see L2).

### Security goals

| ID | Goal |
|---|---|
| G1 | **Existence:** once a commitment is made, its existence can't be hidden later. |
| G2 | **Timing:** a commitment's signing time is bounded on both sides by sources the committer doesn't control. |
| G3 | **Integrity:** neither commitments nor fulfilments can change without detection. |
| G4 | **Follow-through:** every promised item has a checkable status (fulfilled, late, amended, missing, no stated deadline, invalid). |
| G5 | **Provenance:** every commitment shows how it was made (self-signed or extracted from a cited source). |
| G6 | **Neutrality:** the same published method applies to every party, and the registry shows statuses, not verdicts. |

### Adversaries

| ID | Adversary | Wants | Capabilities |
|---|---|---|---|
| A1 | **Strategic lab** | A favourable record at low cost | Controls its keys, its timing, what it commits and publishes, and assessor access |
| A2 | **Pressured or captured assessor** | To keep access to the lab | Controls its own plan and reports; may be bound by access terms |
| A3 | **Biased extractor** | A record that's too harsh or too lenient | Chooses which statements count as promises and how to read them |
| A4 | **Factional reader** | Ammunition | Quotes the record selectively |
| A5 | **Key thief or insider** | To forge or suppress records | Uses a compromised signing key |

Out of scope: hardware-level and nation-state attacks on signing infrastructure, and attacks on drand or Roughtime themselves.

### Threats

| ID | Threat | Attack | Goal hit | Adversary |
|---|---|---|---|---|
| T1 | **File drawer** | Register many commitments or plans and reveal only the favourable ones | G1 | A1, A2 |
| T2 | **Registering after the fact** | Run the eval privately, then pre-register the plan that matches the result | G2 (in spirit) | A1 |
| T3 | **Selective scope** | Promise to disclose only what is already known to look good | G4 | A1 |
| T4 | **Unannounced keys** | Sign risky commitments with keys not tied to the organization, and disown them later | G1, G5 | A1 |
| T5 | **Silent amendment** | Reword a commitment after the fact | G3 | A1 |
| T6 | **Backdating** | Claim a commitment was made earlier than it was | G2 | A1 |
| T7 | **Partial or swapped fulfilment** | Publish a summary or a different artifact and call it fulfilment | G3, G4 | A1 |
| T8 | **Vague promises** | Write promises with no deadline or deliverable, so nothing can be measured | G4 | A1 |
| T9 | **Dropped commitment** | Delete or withhold a commitment after making it, so its absence leaves no trace | G1 | A1 |
| T10 | **Fulfilment on a technicality** | Meet the literal wording but not its intent | G4 | A1 |
| T11 | **Biased extraction** | Read prose too strictly or too leniently | G5, G6 | A3 |
| T12 | **Weaponization** | Quote statuses out of context, so signing looks like handing ammunition to opponents | G6 | A4 |
| T13 | **Gagged assessor** | Access terms stop the assessor from reporting refused access | G4 | A1 via A2 |
| T14 | **Key compromise** | Forge commitments or fulfilments | G3, G5 | A5 |

### Proposed defense system

Seven layers. Each closes or narrows specific threats. None depends on a standard still being drafted.

**L1. Cryptographic integrity (exists in asexec; already covered by its unit tests).** Signatures, content digests, the postreg chain and two-sided time bounds (drand floor, Roughtime ceiling). The registry runs `asexec verify` on each upload and shows the result; anyone can rerun it.
*Addresses:* T5, T6, T7 (swapped content).

**L2. Public registry of commitments (asexec-registry).** Parties upload their records, AsPredicted-style. Every upload is kept in a public, append-only log with its upload time. Unlike AsPredicted, nothing can stay private, so the record shows counts like "10 registered, 3 fulfilled." This turns asexec from a pre-registration tool into a registry, the distinction Data Colada draws between pre-registration (against p-hacking) and study registries (against the file drawer).
*Addresses:* T1, T9.

**L3. Organization-bound keys.** A committer publishes its signing keys at a location it controls, for example a well-known path on its domain. Only commitments from published keys count in the record. Unpublished keys carry no weight. Rotations and revocations are themselves logged.
*Addresses:* T4; limits T14 by making revocation visible.

**L4. Anchoring to things the committer can't run early (asexec feature, first goal).** These mainly protect eval pre-registrations:
- **Assessor co-signature:** an assessor countersigns the prereg before access or a run begins. This works when the assessor controls access, so the committer can't run the eval first.
- **Public deadlines:** for disclosure promises, the due date is tied to a public event, such as "before external deployment of model X."

Not pursued: letting a drand round issued after the prereg choose the eval subset or seeds. A lab that already ran the full set knows roughly what any random subset will show, so it narrows cherry-picking but not registering after the fact.

*Narrows:* T2 (it can't eliminate it).

**L5. Promises about whole classes.** The record favours promises about a whole category ("results of every dangerous-capability eval run before deployment") over single results. Cherry-picking a class is much harder than cherry-picking items, and an assessor with access can check completeness against it.
*Narrows:* T3, T10.

**L6. A neutral record method.**
- Per-promise statuses only, with no scores or rankings. "No stated deadline" is its own status, so vagueness is visible, not hidden.
- Extraction rules are published. Two extractors work on their own, and disagreements are recorded.
- A dispute process: a committer can contest an extraction. The original and the response both stay in the record.
- Every entry has a permalink to the full promise, quoted source, status history and amendments.

*Addresses:* T8 (visible), T11, T12.

**L7. Amendment discipline.** An amendment is a new signed record. It is dated, gives a reason and chains to the original, which stays visible. An amendment after the due date shows as late.
*Addresses:* T5 (beyond L1's tamper detection), T10 (partly).

#### Coverage

| Threat | L1 | L2 | L3 | L4 | L5 | L6 | L7 | After defenses |
|---|---|---|---|---|---|---|---|---|
| T1 File drawer | | ● | | | | | | Visible |
| T2 After the fact | | | | ◐ | | | | Narrowed when an assessor gates access; not prevented |
| T3 Selective scope | | | | | ◐ | ◐ | | Visible as narrow scope |
| T4 Unannounced keys | | | ● | | | | | Carries no weight |
| T5 Silent amendment | ● | | | | | | ● | Detected |
| T6 Backdating | ● | | | | | | | Detected |
| T7 Swapped fulfilment | ● | | | | | ◐ | | Detected; partial content needs a reader's judgement |
| T8 Vague promises | | | | | | ● | | Visible as "no stated deadline" |
| T9 Dropped commitment | | ● | | | | | | Visible |
| T10 Technicality | | | | | ◐ | | ◐ | Needs a reader's judgement |
| T11 Biased extraction | | | | | | ● | | Disagreement and disputes recorded |
| T12 Weaponization | | | | | | ◐ | | Reduced, not prevented |
| T13 Gagged assessor | | | | | | | | **Open:** depends on access terms |
| T14 Key compromise | ◐ | | ◐ | | | | | Revocation visible |

● addresses · ◐ partly addresses

### Residual risks and non-goals

- **Knowledge can't be proven.** No layer proves what a committer didn't know. For eval pre-registrations, state the threat model plainly: asexec proves when the plan existed, not that only one run happened.
- **Truth isn't checked.** Signatures protect the record, not the claim. Whether disclosed content is accurate or adequate is for assessors and readers to judge.
- **Scope can't be forced.** A lab can still commit to little. The record makes that visible; it doesn't prevent it.
- **Collusion.** A lab and an assessor acting together can defeat L4's co-signature.
- **Judgement calls stay with people.** Technicalities and partial fulfilment need human reading. The record supplies context, not verdicts.

### Threat model open questions

1. **Registry host.** Who runs asexec-registry and its log (a neutral host, or several mirrors)?
2. **All-public records.** Will committers accept that every upload is public, or will some need to seal content until a due date (and if so, how long before it counts as missing)?
3. **Gagged assessors (T13).** What do real access terms allow assessors to publish about refused access?
4. **Key binding.** Is a well-known path on the lab's domain enough, or is a stronger identity binding needed?
5. **Dispute process.** Who adjudicates contested extractions while keeping the record neutral?

### Sources

- [AsPredicted: Private Forever](https://aspredicted.org/messages/private_forever.php): why pre-registrations may stay private, and how abuse is handled.
- [AsPredicted: Terms of Use](https://aspredicted.org/terms@): abuse-detection algorithms and blocking.
- [Data Colada [55]: The file-drawer problem is unfixable](https://datacolada.org/55): pre-registration vs study registries.
- [FDAAA TrialsTracker (DeVito, Bacon, Goldacre)](https://www.biorxiv.org/node/82363): a public compliance tracker built from public data.
- [Evaluation of compliance with FDAAA 2007 requirements (Oxford ORA)](https://ora.ox.ac.uk/objects/uuid:9800f9a2-1c7c-47e4-8482-a8c67f14615e/files/r3n203z44x): 40.9% on-time results reporting.

## Related

- [asexec](https://github.com/lwhitestone/asexec): signing and verification.
