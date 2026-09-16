---
name: partner-one-pager
description: >-
  Build a seller-facing Microsoft partner one-pager, grounded in authenticated internal
  sources. Resolves the partner to a PartnerOneID, retrieves Partner Center specializations
  and designations, Partner-Influenced ACR, co-sell opportunity signals, marketplace offers
  and the PMX partner team — each with a source and an explicit confidence — then renders
  using the canonical partner one-pager template. WHEN the user asks to "create a partner one-pager", "build a
  partner brief", "make a battlecard", or runs /partner-one-pager for a named partner. DO
  NOT use for rendering an arbitrary HTML artifact (use /web-artifacts-builder directly) or
  for public marketplace lookup alone (that needs no auth).
---

# Partner one-pager

Create a concise, seller-facing Microsoft partner one-pager for the named partner. Ground internal metrics in authenticated tools, use public sources only for public positioning, and clearly label anything that cannot be verified.

## When to use

Use this skill when the user asks to create a partner one-pager, partner brief, partner battlecard, or runs `/partner-one-pager` for a named partner.

Do not use it for arbitrary HTML artifacts or simple marketplace lookup.

## Inputs

Accept either form:

```text
/partner-one-pager for Contoso
/partner-one-pager for [PARTNER NAME] = Contoso
```

Treat the supplied name as the partner. Ask for clarification only when the name is missing or resolves to multiple plausible PartnerOneIDs.

If the seller uploads or links an existing partner one-pager with the request, treat it as an additional reference input and follow the uploaded-one-pager workflow below.

## Core rules

- Do not invent facts. If a field cannot be grounded, render it as `Validate`.
- Do not turn a grounded zero or empty result into `Validate`; render it as `grounded (none)`.
- Keep internal-only data clearly labelled and do not use confidential internal links as customer-facing CTAs.
- Every metric must include source, period or as-of date, scope, and confidence.
- Never merge multiple PartnerOneIDs unless the user explicitly confirms they represent the same partner and should be combined.
- Compute financial totals from fact tables before joining dimensions; use dimensions only for labels and state match rate when displaying dimension-derived names.
- Authenticated backend sources are authoritative for numbers and key partner facts. Never preserve an uploaded one-pager's metric merely because it already appears in a seller asset.
- Treat uploaded one-pager content as contextual evidence until each claim is confirmed, contradicted, or marked `Validate`.
- Lead with customer and seller business impact, not product features.
- Do not present pipeline value, partner revenue, Marketplace billed sales, or PAEC as quota retired. Label each amount by what it actually measures.
- In Opportunity Signals, omit unavailable fields instead of rendering a visible `Validate` row. Preserve the missing-field status in generation evidence and the final response.

## Business impact contract

Use these formulas to keep the one-pager seller-focused:

| Section | Required formulation |
| --- | --- |
| Headline | `Help <customer type> achieve <business outcome> with <joint solution>` |
| Use case | `<Action> to <measurable customer outcome>` |
| Seller reason | `<Buying signal> → <customer impact> → <Microsoft motion>` |
| Proof | `<Metric> · <period> · <scope> · <confidence>` |
| Incentive | `<Program> · <eligibility status> · <maximum benefit when verified>` |
| CTA | `Target <account type> → position <offer> → use <incentive> → contact <owner>` |

Render a compact `Seller opportunity` strip with five fields:

1. `Customer signal`
2. `Business outcome`
3. `Microsoft pull-through`
4. `Incentive`
5. `Next action`

Rules:

- Outcomes must precede products and features.
- Every seller reason must explain why the signal matters commercially and what Microsoft motion it creates.
- Phrase use cases as outcomes, such as `Reduce outages with grid-edge visibility`, rather than product-only labels.
- Prefer grounded customer reach, Marketplace sales, pipeline, wins, close rates, and customer consumption as proof.
- Remove claims that do not change seller targeting, positioning, funding, or next action.
- Never headline PAEC or partner self-consumption as customer impact.

## Classification labels

| Label | Meaning |
| --- | --- |
| `Public` | Confirmed from public web or marketplace sources. |
| `Internal confirmed` | Grounded through Lakehouse, PMX, or another authenticated internal source. |
| `grounded (none)` | The source was queried and returned zero or no applicable records. |
| `Validate` | Not retrievable, ambiguous, conflicting, or below the publishable evidence bar. |

## Source priority

1. Lakehouse (`lakehouse_status`, `lakehouse_query`) — PartnerOneID, Partner Center credentials, PI ACR, co-sell deals and opportunity status, marketplace billed sales, MACC/customer commitments where available.
2. PMX tools — partner management accounts, account team, contacts, projects, deliverables.
3. Marketplace tools — public offers, transactable status, offer-level MACC eligibility.
4. Public web — positioning, website, logo, public proof points, customer-safe CTAs.

If Lakehouse is unavailable, say so and continue in public-only mode. All fields requiring internal grounding must be labelled `Validate`.

## Uploaded one-pager workflow

When an existing partner one-pager is supplied:

1. Read it before gathering new content and extract its partner identity, value proposition, solution plays, industries, customer segments, use cases, proof points, metrics, incentives, contacts, CTAs, links, logos, and source dates.
2. Reuse pertinent seller-facing context when it remains relevant, especially positioning, solution narratives, customer signals, use cases, approved branding, and useful CTAs.
3. Independently retrieve all numbers and key facts from Lakehouse, PMX, Marketplace, and current public sources. Backend-grounded values replace uploaded values.
4. Reconcile material differences. Do not silently carry forward or silently overwrite conflicting claims:
   - backend confirms the claim: use the backend value and current source date
   - backend provides a newer or differently scoped value: use it and retain the scope/date difference in the generation evidence
   - backend contradicts the uploaded claim: use the backend result and flag the uploaded claim as not reproducible or conflicting
   - backend cannot verify the claim: retain it only when useful and label it `Validate`
5. Never infer that a seller-supplied statement is `Public` or `Internal confirmed` without grounding it in the corresponding source.
6. Record the uploaded file name and its date, when available, in the generation evidence as `Seller-supplied reference`; do not use it as the sole source for internal metrics.
7. Preserve the canonical template and current skill structure rather than copying the uploaded artifact's layout, unless the user explicitly asks to match its design.

## Workflow

### 1. Preflight

Run:

```text
lakehouse_status
pmx crm_whoami or equivalent PMX identity/read check
```

Proceed with the sources that are connected. Do not present public-only research as internally confirmed.

### 2. Resolve the partner

Resolve the partner to exactly one PartnerOneID before retrieving metrics.

```sql
SELECT DISTINCT PartnerOneID, PartnerOneName
FROM common.dimreportingpartnerone_reporting
WHERE LOWER(PartnerOneName) LIKE '%<partner>%'
```

If Lakehouse search is ambiguous or misses the trade name, query PMX:

```text
list_partners partnerOneName: "<partner>"
```

When PMX account name and Partner One legal name differ, show both names. If multiple PartnerOneIDs remain plausible, stop and ask the user which to use.

### 3. Retrieve Partner Center credentials

Use latest snapshots and enrolled flags only.

```sql
SELECT DISTINCT SpecializationName, SolutionArea
FROM pc.factpartnerspecialization
WHERE PartnerOneID = @PartnerOneID
  AND IsSpecializationEnrolled = 1
  AND snapshot_month = (SELECT MAX(snapshot_month) FROM pc.factpartnerspecialization)

SELECT DISTINCT SolutionArea
FROM pc.factpartnerdesignation
WHERE PartnerOneID = @PartnerOneID
  AND IsDesignationEnrolled = 1
  AND snapshot_month = (SELECT MAX(snapshot_month) FROM pc.factpartnerdesignation)
```

For ISVs or software companies, also check software designations when available:

```sql
SELECT MarketPlaceSolutionName, Designation, ProgramName, Status, EnrollmentStartDate
FROM pcs_tti.factpartnersoftwaredesignation
WHERE PartnerOneKey = @PartnerOneID
```

Do not use `EnrollmentStatus` as the credential filter. Zero enrolled rows means `grounded (none)`. State the snapshot date and credential source scope.

### 4. Retrieve Opportunity Signals

Render one compact `Opportunity Signals` block. Its fields are:

1. `Co-sell contract value`
2. `Registered co-sell deals`
3. `Partner Close Rate`

Use one row per `PSXDealID` before aggregating contract value:

```sql
WITH deals AS (
  SELECT PSXDealID,
         MAX(TotalContractValueCD) AS total_contract_value
  FROM crm.factcoselldeal
  WHERE PartnerOneID = @PartnerOneID
  GROUP BY PSXDealID
)
SELECT COUNT(*) AS registered_cosell_deals,
       SUM(total_contract_value) AS cosell_contract_value
FROM deals
```

Label co-sell contract value as pipeline or commercial potential, not quota retired.

Compute `Partner Close Rate` at the distinct MSX opportunity grain for the current and prior Microsoft fiscal years:

```sql
WITH partner_opportunities AS (
  SELECT DISTINCT f.OpportunityID
  FROM crm.factcoselldeal f
  JOIN crm.dimcoselldeal d ON d.PSXDealID = f.PSXDealID
  WHERE f.PartnerOneID = @PartnerOneID
    AND f.IPPartnerOneID = @PartnerOneID
    AND f.IPCoSellPartnerOneKey = @PartnerOneID
    AND d.CreatedFiscalYear IN (@prior_fy, @current_fy)
    AND d.Status = 'Active'
    AND d.PartnerAcceptanceStatus IN ('Accepted', 'Won')
    AND f.OpportunityID IS NOT NULL
),
scored AS (
  SELECT p.OpportunityID,
         MAX(CASE
           WHEN o.Billed_Revenue_Status = 'Won'
             OR o.Consumption_Status = 'Won'
           THEN 1 ELSE 0
         END) AS IsWon
  FROM partner_opportunities p
  LEFT JOIN crm.dimopportunity o
    ON o.Opportunity_Number = p.OpportunityID
  GROUP BY p.OpportunityID
)
SELECT COUNT(*) AS qualifying_opportunities,
       SUM(IsWon) AS won_opportunities,
       CAST(100.0 * SUM(IsWon) / NULLIF(COUNT(*), 0) AS decimal(5,2))
         AS partner_close_rate_pct
FROM scored
```

`Billed_Revenue_Status` is the MSX `Billed Status` field. Do not treat `Closed`, `Open`, `In-Progress`, `N/A`, or a deal-level acceptance status as `Won`. Show the numerator, denominator, fiscal-year scope, and formula beside the percentage. If no qualifying opportunities exist, omit Partner Close Rate from the rendered page and report `grounded (none)` in the final response.

### 5. Retrieve PI ACR and association mix

Compute totals from `pov.factaggregatedacr_pat` only; do not inner join customer dimensions when calculating totals.

```sql
SELECT AssociationType,
       COUNT(DISTINCT CustomerID) AS customers,
       SUM(ACR) AS acr
FROM pov.factaggregatedacr_pat
WHERE PartnerOneID = @PartnerOneID
  AND Monthkey BETWEEN @from_month AND @to_month
GROUP BY AssociationType
```

Rules:

- Sum all PI ACR association types present: `CSP Tier1`, `CSP Tier2`, `Partner Admin Link`, `Partner As End Customer`, `Marketplace`, and `Partner IP Embed`.
- Do not exclude `Partner As End Customer`; disclose it. If PAEC exceeds about 30% of total, break it out as partner self-consumption.
- State fiscal year, months, geography if scoped, and whether the period is closed.
- Include a caveat when `Partner Admin Link` is material because PAL Reader cannot be excluded from this table.
- If one customer exceeds about 30% of ACR, show concentration and customer dimension match rate when names are displayed.

### 6. Retrieve marketplace and billed sales signals

Use marketplace search for public offers:

```text
marketplace_search_offers publisher: "<exact publisher display name>"
```

Publisher matching is exact and case-sensitive. MACC eligibility is per offer; never render a partner-level `MACC: Yes` unless every referenced offer supports that claim or the wording names the eligible offers.

For ISVs, retrieve marketplace billed sales when available:

```sql
SELECT COUNT(DISTINCT TPAccountID) AS customers,
       SUM(TotalPriceCDAmount) AS billed_sales
FROM crm.factmarketplacebilledsales
WHERE PartnerOneID = @PartnerOneID
  AND MonthKey BETWEEN @from_month AND @to_month
```

Report marketplace ACR and marketplace billed sales as separate metrics.

### Quota-relevant commercial amounts

Retrieve and display available monetary fields in this priority order:

1. **Customer Azure consumption** — `TrueACRConsumption` or another grounded customer ACR measure attributable to the partner offering. This is the strongest Azure quota-retirement signal.
2. **Marketplace billed sales** — useful for Marketplace commercial motions, but do not imply it necessarily retires Azure consumption quota.
3. **Co-sell contract value** — `TotalContractValueCD`, deduplicated to one row per `PSXDealID`; label as pipeline or commercial potential, not quota retired.
4. **Partner revenue** — `PartnerRevenueinUSD` or `Potential_Partner_Revenue_in_USD`, deduplicated to one row per `PSXDealID`; label as partner economics, not Microsoft quota.
5. **PAEC** — ACR under `Partner As End Customer`; label as partner self-consumption, not customer-offering pull-through.

Use a deal-level CTE before summing co-sell values:

```sql
WITH deals AS (
  SELECT PSXDealID,
         MAX(DealEstRevenueinUSD) AS estimated_deal_revenue,
         MAX(PartnerRevenueinUSD) AS partner_revenue,
         MAX(Potential_Partner_Revenue_in_USD) AS potential_partner_revenue,
         MAX(TotalContractValueCD) AS total_contract_value,
         MAX(TrueACRConsumption) AS true_acr_consumption,
         MAX(AzurePartnerReportedACR) AS partner_reported_acr
  FROM crm.factcoselldeal
  WHERE PartnerOneID = @PartnerOneID
  GROUP BY PSXDealID
)
SELECT COUNT(*) AS deals,
       SUM(estimated_deal_revenue) AS estimated_deal_revenue,
       SUM(partner_revenue) AS partner_revenue,
       SUM(potential_partner_revenue) AS potential_partner_revenue,
       SUM(total_contract_value) AS total_contract_value,
       SUM(true_acr_consumption) AS true_acr_consumption,
       SUM(partner_reported_acr) AS partner_reported_acr
FROM deals
```

Only render rows with a grounded, non-null monetary value. Do not show an unavailable monetary field or a `Validate` placeholder in the page. If customer Azure consumption is unavailable, start with the next available field in the priority order. Include a concise caveat beside every amount that is not direct customer Azure consumption.

### ISV and GISV incentive treatment

When `PartnerSubSegment` is `ISV` or `GISV`, evaluate FY27 SDC / Frontier Accelerate incentive fit:

- Certified Software Designation from `pcs_tti.factpartnersoftwaredesignation`
- Marketplace billed sales trailing 12 months
- available MACC commitment signal
- Frontier Accelerate enrollment when a source is available

Render incentive status as exactly one of:

- `Confirmed eligible` — all required gates are grounded
- `Potential fit — validate` — partner type or commercial signal aligns, but one or more gates are not confirmed
- `Not evidenced` — reachable sources do not show the required gates

Relevant seller plays may include AI Build & Publish, Copilot Agent publishing, Azure sponsorship, pre-sales assessments, and Customer Migrate & Modernize. Show maximum benefits only when verified from a current FY27 source. Incentives facilitate assessment, build, publishing, migration, and deployment; do not describe incentive funding itself as quota retirement.

### 7. Retrieve MACC/customer commitment context

MACC offer eligibility and customer commitments are different signals. For customer commitment context, join attributed customers to MACC only after deduping customers and using the latest row per customer.

```sql
WITH cust AS (
  SELECT DISTINCT PartnerOneID, CustomerID
  FROM pov.factaggregatedacr_pat
  WHERE PartnerOneID = @PartnerOneID
    AND Monthkey BETWEEN @from_month AND @to_month
),
latest AS (
  SELECT TPID, MACCStatus, MACCExpectedConsumption, [CustomerName-TPID] AS customer_name,
         ROW_NUMBER() OVER (PARTITION BY TPID ORDER BY Date DESC, MACCExpectedConsumption DESC) AS rn
  FROM macc.factmacc
)
SELECT COUNT(DISTINCT c.CustomerID) AS customers,
       COUNT(DISTINCT l.TPID) AS with_macc,
       SUM(COALESCE(l.MACCExpectedConsumption, 0)) AS expected_consumption
FROM cust c
LEFT JOIN latest l ON l.TPID = c.CustomerID AND l.rn = 1
```

State as-of date and do not imply the partner is primary partner on the commitment unless the source proves it. If the MACC customer is the partner itself, say so.

### 8. Retrieve PMX team and project context

Use PMX for operational ownership and routing context.

```text
list_partners partnerOneNumber: "<PartnerOneID>"
load_partner_contacts_and_team partnerOneId: "<PartnerOneID>"
list_partner_team partner_id: "<partnerManagementAccountId>", role: "PTS"
list_projects accountIds / partnerKeyword / projectKeyword
```

Render only the seller-facing team summary:

- primary PDM, with email when available
- PTS count, plus one named technical contact when identifiable
- primary partner-side contact when present

Keep larger team lists and routing details out of the one-pager body.

### 9. Gather public positioning

Use marketplace and public web sources for customer-safe positioning:

- website and logo
- short value proposition
- customer segments observed in attributed ACR or co-sell activity
- 3 solution areas or plays
- 3 to 5 when-to-engage customer signals
- 4 key use cases
- up to 3 CTA links

Do not use internal-only source names or links as customer-facing calls to action.

## Rendering requirements

Delegate rendering to `/web-artifacts-builder` and create a self-contained HTML file named:

```text
[PARTNER NAME]-one-pager.html
```

Use the SharePoint-hosted reusable template as the canonical output reference:

```text
https://microsoft.sharepoint.com/:u:/r/teams/PartnerOnePager/Page%20templates/Forms/AllItems.aspx?id=%2Fteams%2FPartnerOnePager%2FPage%20templates%2Fpartner%2Done%2Dpager%2Dtemplate%2Ehtml&parent=%2Fteams%2FPartnerOnePager%2FPage%20templates&p=true&share=cQrdN3HeIfl3Q5f225csuPH7EgUB7QI5%2DnUMJQdnwnLQsNFgRQ
```

The template is named `partner-one-pager template` and is the sole authority for the reusable HTML structure, visual styling, Microsoft logo SVG, iconography, proportions, spacing, and replacement placeholders. Read the SharePoint template before rendering every partner one-pager. If the template cannot be accessed, report that limitation rather than silently using a stale local copy.

### Required page structure

Use a single portrait page, approximately 980px wide and 1280px tall, with this order:

1. `Microsoft Confidential` pill at the top.
2. Top logo row with partner logo and Microsoft logo separated by a vertical divider.
3. Top-right slanted/parallelogram category ribbon.
4. Hero section with large partner + Microsoft title, blue headline, short summary, and a right-side rounded Partner Snapshot card. The left hero block and Partner Snapshot card must have matching heights. Size and vertically center the snapshot text so it uses the available card space without overflowing. Include `Customer segments` populated from observed attributed ACR or co-sell activity and `Internal Contact` populated from the primary PDM in PMX, formatted as `<PDM name> (PDM)`. Group commercial/public-sector variants under their segment family when needed and do not imply these are declared target segments.
5. Better Together statement.
6. Three large rounded solution cards connected by circular plus icons.
7. Compact `Seller opportunity` strip with `Customer signal`, `Business outcome`, `Microsoft pull-through`, `Incentive`, and `Next action`.
8. Split middle section: left `When to engage` checklist, right `Key use cases` row.
9. Small proof strip under use cases for the most relevant grounded commercial or quota signal.
10. `Why sellers should care` section: three stacked signal → impact → Microsoft-motion cards on the left and a compact monetary proof list on the right. Order available amounts as customer Azure consumption, Marketplace billed sales, co-sell contract value, partner revenue, then PAEC. Omit unavailable monetary fields. Keep this section vertically compact by minimizing margins and padding without reducing font or icon sizes.
11. Marketplace / MACC / transactable callout strip.
12. Rounded Call to Action footer with partner logo, CTA copy and links, and Microsoft logo.
13. Compact footer with copyright/update date. Do not render a visible source or validation note.

Keep copy concise: 3 seller reasons, 4 use cases, 5 engagement signals, and 3 CTA links maximum.

### Visual style

Match the SharePoint-hosted `partner-one-pager-template.html` as the visual source of truth.

- Use the mandatory `/web-artifacts-builder` Clawpilot theme script and CSS variables.
- Preserve the Microsoft field-collateral feel: white surface, bold rounded orange-red outer border, Microsoft blue accents, muted gray body copy, rounded cards, and compact executive-scan typography.
- Use inline SVG logos/icons where possible. Do not use placeholder logo boxes when a partner logo can be safely embedded or represented from an existing approved local/reference asset.
- Use actual Microsoft logo blocks as inline SVG, not plain text.
- Use compact section density so the full artifact fits on one printable page.
- Keep all colors expressed through `var(--cp-*)` except inside trusted inline SVG logo artwork.
- Keep validation caveats and internal-only provenance out of the rendered page; preserve them in the generation evidence and final response.

## Final response

Report back in three bullets or fewer:

- saved path and clickable file link
- confirmed internal metrics used, with source systems
- fields left as `Validate`, and fields that returned `grounded (none)`
