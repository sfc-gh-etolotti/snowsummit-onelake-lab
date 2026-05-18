---
name: SnowSummit_Skills
description: "SnowSummit hands-on lab: bi-directional Fabric + Snowflake Iceberg integration. Guides participants through CLD setup, star schema modeling, and managed Iceberg write-back to OneLake. CoCo executes each SQL block with inline explanations and facilitator discussion points. Triggers: snowsummit, snowsummit lab, onelake hol, onelake hands on lab, iceberg lakehouse lab, fabric snowflake lab."
---

<!-- ============================================================
     COCO EXECUTION CONTRACT — BINDING RULES
     ============================================================ -->

**YOU MUST FOLLOW THESE RULES FOR EVERY STEP IN THIS SKILL. VIOLATIONS WILL BREAK THE LIVE DEMO.**

1. **NEVER INVENT STEPS.** Only present instructions that exist in the phase files below. Do not fabricate, improvise, or infer steps — even if they seem logical. If a step is not written in the phase file, it does not exist. This is the #1 failure mode: telling participants to create objects (Lakehouses, connections, etc.) that the skill never asks for.

2. **NEVER execute SQL without first presenting the educational block above it.** Every section marked with `📖` MUST be output as conversational text to the user BEFORE any SQL in that section is shown or executed. The educational content IS the deliverable — the SQL is secondary.

3. **NEVER batch multiple phases or steps in a single response.** Each educational block + checkpoint is one conversational turn. Present it, stop, wait for "go".

4. **NEVER summarize or abbreviate the `📖` blocks.** Present the full text as written. These are facilitator talking points — they need the complete explanation to teach from.

5. **Checkpoint = HARD STOP.** When you reach a `💡 Checkpoint` or `🛑 STOP` line, you MUST stop output and wait for "go". Do not continue past it under any circumstances. Do not preview what comes next.

6. **Two-turn pattern for every step:**
   - **Turn 1**: Present the `📖` educational block + checkpoint. STOP.
   - **Turn 2** (after "go"): Execute the SQL block, present results, then present the NEXT educational block and STOP again.

7. **DISCUSSION blocks (`💬`) are mandatory pauses.** Present all talking points and wait. Do not auto-advance.

8. **STOP blocks (`⏸`) are manual gates.** These require the user to perform actions in Fabric/Azure. Present the instructions exactly as written, then wait for confirmation. Do not suggest alternative approaches.

9. **When in doubt, re-read the phase file.** If you are unsure what the next step is, re-read the relevant phase file from the `phases/` directory rather than guessing.

10. **Phase transitions**: When a phase file ends with "Phase N complete", confirm completion to the user and ask them to say "go" to continue. Then READ the next phase file and follow it.

<!-- ============================================================ -->

# SnowSummit: Fabric + Snowflake Interoperability Lab

End-to-end hands-on lab. CoCo explains every concept before running it, then gates on your acknowledgement. Nothing executes without a **"go"**.

**What gets built:**
1. **Fabric Workspace + Lakehouse** — Raw retail data loaded into OneLake
2. **Azure App Registration** — Identity for Snowflake to authenticate against Azure
3. **Catalog Integration + External Volume** — Snowflake's two credentials for OneLake
4. **Catalog Linked Database (CLD)** — Snowflake reads OneLake Iceberg tables zero-copy
5. **Star schema as Iceberg** — DIM_PRODUCT, DIM_STORE, DIM_CUSTOMER, DIM_DATE, FACT_DAILY_SALES created directly as Iceberg tables in OneLake

---

## Workshop Phases

| Phase | File | What Happens | Est. Time |
|-------|------|-------------|-----------|
| 0 | `phases/phase_0_prereqs.md` | Fabric workspace setup, Lakehouse creation, CSV upload, App Registration | 20 min |
| 1 | `phases/phase_1_collect_values.md` | Collect 5 Azure/Fabric values | 5 min |
| 2 | `phases/phase_2_infrastructure.md` | Session variables, Catalog Integration, External Volume, consent + SP grants | 15 min |
| 3 | `phases/phase_3_cld.md` | Catalog Linked Database + validation + discussion | 10 min |
| 4 | `phases/phase_4_onelake_setup.md` | OneLake write-back setup: database, Fabric connection, Snowsight wizard | 15 min |
| 5 | `phases/phase_5_star_schema_iceberg.md` | Star schema as Iceberg tables directly in OneLake | 10 min |
| 6 | `phases/phase_6_recap.md` | Recap + what's next (Power BI, Copilot Studio) | 5 min |
| 7 | `phases/phase_7_cleanup.md` | Cleanup + troubleshooting reference | 5 min |

---

## How This Skill Works

**CoCo reads one phase file at a time.** Each phase file contains:
- Educational explainers (`📖` blocks) that CoCo presents as conversation
- SQL blocks that CoCo executes only after a "go" acknowledgement
- Checkpoints and STOP gates for manual Fabric/Azure steps

**To begin**: Say **"go"** and CoCo will read `phases/phase_0_prereqs.md` and start the lab.

**To advance**: After each phase completes, say **"go"** to load the next phase.

**To skip**: Say "skip to Phase N" to jump ahead (e.g., if setup is already done).

---

**When the facilitator says "go", READ `phases/phase_0_prereqs.md` and follow its instructions exactly. Do NOT summarize — present the content as written.**
