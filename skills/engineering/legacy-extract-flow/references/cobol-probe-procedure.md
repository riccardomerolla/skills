# B-Mode COBOL Probe Procedure

## Preamble

COBOL probes are surgical, recorded, and exceptional. The default extraction path for `legacy-extract-flow` reads only J2EE source: JSPs, servlets, and ESB configuration. A probe is justified only when both of the following are true:

1. A business rule clearly lives mainframe-side (the J2EE source invokes an ESB call to compute or enforce it, rather than implementing it locally).
2. The ESB contract for the relevant call is not expressive enough to specify the rule for a Spring Boot adapter to wrap correctly: field names and types are present, but the behavioral condition is opaque.

If either condition is not met, the rule can be specified from J2EE + ESB contract alone. Do not probe.

---

## Procedure

### Step 1: Justify the probe

Write one sentence in the flow file's business-rules section stating: (a) why the ESB contract is insufficient, and (b) what behavioral detail is needed to specify the rule. This sentence is mandatory. Refuse to open COBOL without it.

Example: *"ESB contract MC-LOAN-001 lists `DECLINED` as a possible value for `approvalStatus` but does not document the condition that produces it; the delinquency check cannot be specified from the J2EE side alone."*

### Step 2: Locate the paragraph(s)

Use the mainframe contract surface in `legacy/inventory.md` as the starting point. Map the ESB call's transaction code or CICS program name to the COBOL program. Read the PROCEDURE DIVISION. Locate the paragraph(s) that implement the rule, not the entire program.

Do not read the program top-to-bottom for "context." Navigate to the rule boundary.

### Step 3: Lift the rule, not the code

Restate the rule in plain language in the flow file's business-rules section. One sentence. Do not transcribe COBOL syntax, variable names, or copybook field names verbatim. Cite the COBOL location (program name, paragraph name, and line range) as provenance.

Example rule statement: *"An applicant with a delinquency flag on their account receives DECLINED regardless of loan amount or term."*
Example provenance: `LNAPPRVL` paragraph `CHKDELQ` lines 320–340.

### Step 4: Stop at the rule boundary

The probe ends when the rule is captured. Do not extract surrounding glue: file I/O, DB2 calls, VSAM reads, COMMAREA formatting, error-handling scaffolding. If understanding the rule requires following a PERFORM to another paragraph, follow it, then stop. One rule, one probe.

### Step 5: Record the probe

Add a row to the COBOL probe audit log in the flow file (section 10 of `references/flow-template.md`). Required fields: probe number, ESB call ID, COBOL program, COBOL paragraph, line range, rule ID, justification sentence from step 1. This is the regulator-facing audit trail. Every probe is listed; none are omitted.

---

## Forbidden

Do not open COBOL for any of the following reasons:

- "Context": wanting to understand how the program works broadly.
- "Completeness": not wanting the flow file to have gaps.
- The J2EE-side rule was easy enough to specify without it; if it was easy, the ESB contract was sufficient; no probe needed.
- Curiosity about neighboring paragraphs or data structures.

Defaulting to probes destroys the strangler-fig extraction posture. It also pollutes the regulator artifact with mainframe reads that were not necessary to specify the replacement behavior. The regulator will ask: "why did you read this?" The justification sentence (step 1) is your answer. If you cannot write it, do not probe.
