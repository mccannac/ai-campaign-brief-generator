[README.md](https://github.com/user-attachments/files/31713327/README.md)
# AI Campaign Brief Generator — Build Package

This package translates the architecture blueprint into buildable n8n artifacts. Five files:

| File | What it is |
|---|---|
| `n8n-main-workflow.json` | Importable n8n workflow: intake → validation → data retrieval → normalize → 4 Claude calls → automated QA → store draft → notify reviewer |
| `n8n-approval-handler-workflow.json` | Separate importable workflow: watches Airtable for the reviewer's decision, handles approve/reject branching |
| `prompt-templates-and-schemas.md` | The four system prompts + JSON schemas used by the AI nodes — **source of truth**, keep in sync with the embedded copies in the workflow JSON |
| `data-schema-and-intake-form.md` | Airtable base schema (5 tables), intake form field spec, and the deterministic QA rule set |
| `README.md` | This file |

**Put all five in a GitHub repo now**, before you touch the n8n UI. That closes gap #5 from the blueprint (no version control on prompts/schemas) from day one — every future prompt tweak becomes a diffable, revertible commit instead of an untracked change buried in n8n's UI.

---

## Why two workflows, not one

n8n workflows run start-to-finish in a single execution. Human review is asynchronous — the reviewer might approve a brief five minutes later or two days later — so it can't live inside one continuous run. The main workflow ends by writing the draft brief to Airtable and notifying the reviewer; the approval handler is a second, independent workflow that polls Airtable for the status change and takes it from there. This is standard practice for human-in-the-loop n8n design, not a workaround.

---

## Setup order

1. **Build the Airtable base first.** Create the 5 tables from `data-schema-and-intake-form.md` exactly as specified — the field names in the workflow JSON (e.g. `Draft Brief (JSON)`, `Status`) must match your Airtable columns exactly, since n8n's Airtable node maps by field name.
2. **Build the intake form** (Tally or Google Forms) using the field list in the same doc. Wire its submission to write into the `Campaigns` table, or use n8n's native Form Trigger node instead of an external form tool — the main workflow JSON already assumes the latter (`Campaign Request Form` node) so you don't need a separate form tool at all unless you prefer one.
3. **Import both JSON files** into n8n (Workflows → Import from File).
4. **Add credentials** in n8n and attach them to the placeholder nodes:
   - Airtable Personal Access Token → every Airtable node
   - Anthropic API key, as a Header Auth credential (`x-api-key`) → the four `AI:` nodes
   - Slack → the notification nodes
   - Replace every `REPLACE_WITH_BASE_ID`, `REPLACE_WITH_CREDENTIAL_ID`, and `REPLACE_WITH_CHANNEL_ID` placeholder with your real values.
5. **Replace the placeholder data-source node** (`Fetch Live Ad Platform Data (Placeholder)`) with real calls to whichever of Google Ads / GA4 / CRM you actually use — this is deliberately left unbuilt since it depends entirely on which platforms and auth methods your stack uses.
6. **Test with one fabricated campaign request** end-to-end before pointing real marketers at the form. Watch specifically for: the four AI nodes returning valid JSON (check the Parse nodes don't throw), and Automated QA correctly catching an intentionally-broken test case (e.g., submit a request with `Timeline_End` before `Timeline_Start` and confirm it gets flagged).
7. **Activate both workflows.**

---

## What's intentionally left for you to configure

- Real API credentials (nothing above will run without them)
- The live ad-platform/GA4/CRM data pull — auth methods vary too much to template
- Exact Slack channel and Airtable base IDs
- n8n node type versions may auto-upgrade on import depending on your n8n version; if a node shows a version-mismatch warning, let n8n update it rather than reverting — the parameter names used here are based on current stable versions but n8n does evolve these

## What NOT to change without thinking it through

- The four-call AI structure (facts → insights/hypotheses → recommendations → assembly). Collapsing this into one call is the single change most likely to reintroduce the "speculation presented as fact" problem the whole design exists to prevent.
- The Automated QA gate before human review — don't route straight from brief assembly to Slack notification.
- The persona dropdown being constrained to existing Airtable records rather than free text on the intake form.

---

## Suggested first real test

Once wired up, run the B2B lead-gen example from the blueprint (Section 14) through the live system with a real Personas/Products record on file, and compare the generated brief's `historical_insights` and `strategic_hypotheses` sections against what was described there — that's your sanity check that the fact/insight/hypothesis separation is actually holding up in practice, not just on paper.
