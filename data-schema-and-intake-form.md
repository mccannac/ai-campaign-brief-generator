# AI Campaign Brief Generator — Data Schema, Intake Form & QA Rules

## 1. Airtable Base: "Campaign Brief Generator"

Create one Airtable base with these five tables. Field types are Airtable field types.

### Table: `Campaigns`
| Field | Type | Notes |
|---|---|---|
| Campaign Name | Single line text | |
| Campaign Type | Single select | Lead Gen / Brand Awareness / Product Launch / Retention-Upsell / Event Promotion |
| Business Objective | Long text | |
| Desired Outcome | Single line text | e.g. "150 SQLs" |
| Offer | Long text | |
| Budget | Currency | |
| Timeline Start | Date | |
| Timeline End | Date | |
| Geography | Single line text | Default "All" if blank |
| Channels | Multiple select | LinkedIn / Google Search / Meta / Email / Display / Events |
| Audience/Persona | Link to `Personas` | |
| Constraints | Long text | |
| Requester Name | Single line text | |
| Requester Email | Email | |
| Status | Single select | Submitted / Validating / Processing / Awaiting Review / Approved / Rejected / Revision Requested |
| Draft Brief (JSON) | Long text | Raw output of Call 4 |
| Final Approved Brief | Long text | Copied from Draft Brief on approval, plus reviewer edits |
| QA Flags | Long text | Output of Automated QA node |
| Reviewer | Single line text | |
| Review Notes | Long text | |
| Approved Date | Date | |
| Created At | Created time | Auto |

### Table: `Personas`
| Field | Type | Notes |
|---|---|---|
| Persona Name | Single line text | Primary field — this is what the intake form dropdown pulls from |
| ICP Description | Long text | |
| Pain Points | Long text | |
| Jobs To Be Done | Long text | |
| Buying Stage | Single select | Awareness / Consideration / Decision |
| Notes | Long text | |

### Table: `Products`
| Field | Type | Notes |
|---|---|---|
| Product Name | Single line text | |
| Positioning Statement | Long text | |
| Value Props | Long text | |
| Brand Guidelines Notes | Long text | e.g. "no discount language" |
| Related Campaigns | Link to `Campaigns` | |

### Table: `HistoricalFacts`
Reusable, normalized performance snippets — this is what makes later campaigns start "richer" than the first.
| Field | Type | Notes |
|---|---|---|
| Related Campaign | Link to `Campaigns` | |
| Source | Single select | Google Ads / Meta / GA4 / CRM / Email |
| Metric | Single line text | e.g. "CPL" |
| Value | Single line text | |
| Segment/Channel | Single line text | |
| Date Range | Single line text | |
| Data Freshness Flag | Single select | Fresh / Stale / Missing |

### Table: `Learnings`
The knowledge-base / feedback-loop table (Section 13 of the blueprint).
| Field | Type | Notes |
|---|---|---|
| Related Campaign | Link to `Campaigns` | |
| Type | Single select | Insight / Hypothesis Validated / Hypothesis Invalidated / Human Edit / Post-Campaign Result |
| Description | Long text | |
| Source Brief | Link to `Campaigns` | |
| Date Logged | Date | Auto |

---

## 2. Intake Form (Tally / Google Forms)

Fields map 1:1 to `Campaigns` table columns so the form submission can write directly into Airtable.

| Field | Type | Required | Validation |
|---|---|---|---|
| Campaign Name | Short text | Yes | Non-empty |
| Campaign Type | Dropdown | Yes | Must match one of the 5 fixed types (not free text — this is what keeps taxonomy clean, per blueprint gap #11) |
| Business Objective | Paragraph | Yes | Non-empty |
| Desired Outcome | Short text | Yes | Non-empty |
| Offer | Paragraph | Yes | Non-empty |
| Budget | Number | Yes | > 0 |
| Timeline Start | Date | Yes | — |
| Timeline End | Date | Yes | Must be after Timeline Start |
| Geography | Short text | No | Defaults to "All" |
| Channels | Multi-select checkboxes | Yes | At least 1 selected |
| Target Audience / Persona | Dropdown (populated from `Personas` table) | Yes | Must select an existing persona; if none fits, requester should request a new persona be added first rather than free-typing one |
| Constraints | Paragraph | No | — |
| Requester Name | Short text | Yes | — |
| Requester Email | Email | Yes | Valid email format |

**Why persona is a dropdown, not free text:** this is the single highest-leverage validation rule in the whole intake form — it's what prevents the Normalize step from ever hitting an audience description the system has no data for, and it's what makes historical facts reusable across campaigns for the same persona.

---

## 3. Automated QA Rule Set (pseudocode for the `Automated QA` Code node)

Runs against the assembled brief JSON (output of Call 4) before it reaches the human reviewer.

```
function runQA(brief, facts, insights, hypotheses, recommendations, campaignInputs) {
  const flags = [];

  // 1. Required sections present and non-empty
  const requiredSections = [
    'executive_summary','business_objective','marketing_objective','target_audience',
    'customer_problem','offer_and_value_proposition','historical_insights',
    'strategic_hypotheses','campaign_strategy_and_messaging_pillars','creative_angles',
    'channel_strategy','funnel_customer_journey','recommended_kpis_and_measurement_plan',
    'testing_plan','budget_considerations','risks_and_constraints','open_questions',
    'required_human_decisions'
  ];
  for (const section of requiredSections) {
    if (!brief[section] || brief[section].length === 0) {
      flags.push(`Missing or empty section: ${section}`);
    }
  }

  // 2. Every tagged claim resolves to a real source id
  const validIds = new Set([
    ...facts.map(f => f.id),
    ...insights.map(i => i.id),
    ...hypotheses.map(h => h.id),
    ...recommendations.map(r => r.id)
  ]);
  const allBriefText = JSON.stringify(brief);
  const citedIds = [...allBriefText.matchAll(/\[([a-z]\d+)\]/g)].map(m => m[1]);
  for (const id of citedIds) {
    if (!validIds.has(id)) flags.push(`Brief cites unknown source id: ${id}`);
  }

  // 3. Hypotheses aren't stated with fact-level certainty
  const certaintyWords = ['will generate', 'guaranteed', 'proven', 'definitely'];
  for (const h of brief.strategic_hypotheses || []) {
    if (certaintyWords.some(w => h.toLowerCase().includes(w))) {
      flags.push(`Hypothesis stated with unwarranted certainty: "${h.slice(0,60)}..."`);
    }
  }

  // 4. Channel allocation sanity check, if percentages are present
  const pctMatches = [...allBriefText.matchAll(/(\d{1,3})%/g)].map(m => Number(m[1]));
  // (only enforced if the brief proposes a full channel split — otherwise skipped)

  // 5. Timeline sanity
  if (new Date(campaignInputs.Timeline_End) <= new Date(campaignInputs.Timeline_Start)) {
    flags.push('Timeline_End is not after Timeline_Start');
  }

  // 6. Light PII scan (aggregated data only should reach this stage)
  const piiPattern = /[\w.+-]+@[\w-]+\.[a-z]{2,}|\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/i;
  if (piiPattern.test(allBriefText)) {
    flags.push('Possible PII detected in brief text — review before proceeding');
  }

  return { passed: flags.length === 0, flags };
}
```

**On QA failure:** route to a Slack alert and set `Campaigns.Status = "QA Flagged"` rather than silently retrying — this is the one loop in the workflow deliberately *not* fully automated, since repeated auto-retries against the same bad input tend to just produce a differently-flawed brief. A human decides whether to fix the input and resubmit or manually patch the brief.
