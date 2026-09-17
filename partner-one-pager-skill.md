---
name: "partner-one-pager"
description: "Build a seller-facing Microsoft partner one-pager grounded in authenticated internal sources; resolves PartnerOneID, retrieves credentials, PI ACR, co-sell opportunity signals, RQA hero products, marketplace offers, and PMX team context, then renders the canonical one-pager template."
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
- The GitHub-hosted `partner-one-pager-template.html` structure must be followed exactly. Do not add, remove, rename, reorder, merge, or collapse template sections, cards, rows, labels, or placeholders unless the user explicitly asks for that template change. Fill every template placeholder with grounded content, `grounded (none)`, or `Validate` according to the classification rules.
- Populate `Hero Products` only from the partner-scoped RQA Hero Product dropdown with Fiscal Year cleared to `All`; do not substitute Marketplace offers, uploaded product lists, or the FY27 primary-product measure.
- Lead with customer and seller business impact, not product features.
- Do not present pipeline value, partner revenue, Marketplace billed sales, or PAEC as quota retired. Label each amount by what it actually measures.
- In Opportunity Signals, render only `Co-sell contract value`, `Registered co-sell deals`, `Partner Close Rate`, and `MACC Eligibility`; do not add Marketplace billed sales, PAEC, partner revenue, customer Azure consumption, or any other metric to that section. Omit unavailable allowed fields instead of rendering a visible `Validate` row. Preserve the missing-field status in generation evidence and the final response.

## Business impact contract

Use these formulas to keep the one-pager seller-focused:

| Section | Required formulation |
| --- | --- |
| Headline | `Help <customer type> achieve <business outcome> with <joint solution>` |
| Use case | `<Action> to <measurable customer outcome>` |
| Seller reason | `<Buying signal> -> <customer impact> -> <Microsoft motion>` |
| Proof | `<Metric> - <period> - <scope> - <confidence>` |
| Incentive | `<Program> - <eligibility status> - <maximum benefit when verified>` |
| Next action | `<Owner> - <specific follow-up> - <target account or opportunity>` |
| CTA | `Target <account type> -> position <offer> -> use <incentive> -> contact <owner>` |

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

1. Lakehouse (`lakehouse_status`, `lakehouse_query`) - PartnerOneID, Partner Center credentials, PI ACR, co-sell deals and opportunity status, marketplace billed sales, MACC/customer commitments where available.
2. PMX tools - partner management accounts, account team, contacts, projects, deliverables.
3. RQA Qualify report - partner-scoped Hero Product dropdown values with Fiscal Year set to `All`.
4. Marketplace tools - public offers, transactable status, offer-level MACC eligibility.
5. Public web - positioning, website, logo, public proof points, customer-safe CTAs.

If Lakehouse is unavailable, say so and continue with any connected PMX, RQA, Marketplace, and public sources. Fields that specifically require Lakehouse grounding must be labelled `Validate`.

## Uploaded one-pager workflow

When an existing partner one-pager is supplied:

1. Read it before gathering new content and extract its partner identity, value proposition, solution plays, industries, customer segments, use cases, proof points, metrics, incentives, contacts, CTAs, links, logos, and source dates.
2. Reuse pertinent seller-facing context when it remains relevant, especially positioning, solution narratives, customer signals, use cases, approved branding, and useful CTAs.
3. Independently retrieve all numbers and key facts from Lakehouse, PMX, RQA, Marketplace, and current public sources. Backend-grounded values replace uploaded values.
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
4. `MACC Eligibility`

These are the only fields allowed in the rendered `Opportunity Signals` section. Do not include `Marketplace billed sales`, `PAEC`, partner revenue, customer Azure consumption, marketplace ACR, customer MACC commitment values, or any other commercial metric in this section. Those signals may be used in the proof strip, seller reasons, or generation evidence when relevant and properly labeled.

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

Compute `Partner Close Rate` as the co-sell/referral win rate: the percentage of distinct partner referral/co-sell records with either a Partner Referral ID or linked MSX Opportunity ID that are marked `Won` in Partner Center/co-sell status or have a linked MSX opportunity with MSX, billed revenue, or consumption status marked `Won`.

Use the denominator as all distinct referral/co-sell records where either `PSXDealID` (Partner Referral ID) or `OpportunityID` (MSX Opportunity ID) is present. Count a win when either the referral/co-sell status is won or the linked MSX opportunity status is won. Include records with and without linked MSX Opportunity IDs.

```sql
WITH records AS (
  SELECT COALESCE(f.PSXDealID, f.OpportunityID) AS record_id,
         MAX(f.OpportunityID) AS opportunity_id,
         MAX(CASE
           WHEN d.Close_Status = 'Won'
             OR d.Status = 'Won'
             OR d.PartnerAcceptanceStatus = 'Won'
             OR o.MSX_Status = 'Won'
             OR o.Billed_Revenue_Status = 'Won'
             OR o.Consumption_Status = 'Won'
           THEN 1 ELSE 0
         END) AS is_won
  FROM crm.factcoselldeal f
  LEFT JOIN crm.dimcoselldeal d ON d.PSXDealID = f.PSXDealID
  LEFT JOIN crm.dimopportunity o ON o.Opportunity_Number = f.OpportunityID
  WHERE f.PartnerOneID = @PartnerOneID
    AND (f.PSXDealID IS NOT NULL OR f.OpportunityID IS NOT NULL)
  GROUP BY COALESCE(f.PSXDealID, f.OpportunityID)
)
SELECT COUNT(*) AS total_referral_cosell_records,
       SUM(is_won) AS won_referral_cosell_records,
       CAST(100.0 * SUM(is_won) / NULLIF(COUNT(*), 0) AS decimal(6,2))
         AS partner_close_rate_pct,
       SUM(CASE WHEN opportunity_id IS NOT NULL THEN 1 ELSE 0 END)
         AS records_with_msx_opportunity_id,
       SUM(CASE WHEN opportunity_id IS NULL THEN 1 ELSE 0 END)
         AS records_without_msx_opportunity_id,
       SUM(CASE WHEN is_won = 1 AND opportunity_id IS NOT NULL THEN 1 ELSE 0 END)
         AS wins_with_msx_opportunity_id,
       SUM(CASE WHEN is_won = 1 AND opportunity_id IS NULL THEN 1 ELSE 0 END)
         AS wins_without_msx_opportunity_id
FROM records
```

Render `Partner Close Rate` as the percentage value only, e.g. `<win rate>%`. Use this scope text format: `<wins> won / <total referral/co-sell records> opportunities`. Preserve the with-MSX-ID vs without-MSX-ID numerator and denominator breakdown in generation evidence and final response when relevant.

`Billed_Revenue_Status` is the MSX `Billed Status` field. Do not treat `Closed`, `Open`, `In-Progress`, `N/A`, or any non-won status as `Won`; `Closed` alone is not a win unless another referral/co-sell or MSX status is explicitly `Won`. If no referral/co-sell records with either Partner Referral ID or MSX Opportunity ID exist, omit Partner Close Rate from the rendered page and report `grounded (none)` in the final response.

Render `MACC Eligibility` from Azure Marketplace offer-catalog evidence only. Use each referenced offer's `isMacc` flag; do not infer partner-level eligibility from customer MACC commitments, marketplace billed sales, Partner Center designations, or an uploaded seller asset. Render the metric value as:

- `Yes` when every referenced Marketplace offer in the one-pager is MACC eligible.
- `Partial` when at least one referenced offer is MACC eligible and at least one referenced offer is not.
- `No` when referenced Marketplace offers are found and none are MACC eligible.

Use concise scope text such as `All referenced Marketplace offers`, `Eligible: <offer names>; not eligible: <offer names>`, or `Referenced Marketplace offers not MACC eligible`. If Marketplace offer lookup is unavailable or no referenced Marketplace offers can be grounded, omit the row from the rendered page and report `Validate` or `grounded (none)` in the final response and generation evidence as appropriate.

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

Publisher matching is exact and case-sensitive. MACC eligibility is per offer; render Opportunity Signals `MACC Eligibility` as `Yes` only when every referenced Marketplace offer supports that claim, otherwise use `Partial` or `No` with offer-level scope.

Marketplace Partner Directory is optional public evidence, separate from the Azure Marketplace offer catalog. Use it only when partner-directory profile fields are needed, such as public competencies, designations, endorsed products, locations, or contacts. If the directory search does not return the partner but the offer catalog confirms the publisher/offers, continue with offer-catalog evidence and mark the directory result as `grounded (none)` in generation evidence; do not treat that as a Marketplace presence failure.

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

1. Customer Azure consumption - `TrueACRConsumption` or another grounded customer ACR measure attributable to the partner offering. This is the strongest Azure quota-retirement signal.
2. Marketplace billed sales - useful for Marketplace commercial motions, but do not imply it necessarily retires Azure consumption quota.
3. Co-sell contract value - `TotalContractValueCD`, deduplicated to one row per `PSXDealID`; label as pipeline or commercial potential, not quota retired.
4. Partner revenue - `PartnerRevenueinUSD` or `Potential_Partner_Revenue_in_USD`, deduplicated to one row per `PSXDealID`; label as partner economics, not Microsoft quota.
5. PAEC - ACR under `Partner As End Customer`; label as partner self-consumption, not customer-offering pull-through.

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

Only render rows with a grounded, non-null monetary value. Do not show an unavailable monetary field or a `Validate` placeholder in the page. If customer Azure consumption is unavailable, start with the next available field in the priority order. Include a concise caveat beside every amount that is not direct customer Azure consumption. These quota-relevant amounts are supporting proof signals; do not add them to the `Opportunity Signals` block except for the allowed `Co-sell contract value` field.

### ISV and GISV incentive treatment

When `PartnerSubSegment` is `ISV` or `GISV`, evaluate FY27 SDC / Frontier Accelerate incentive fit:

- Certified Software Designation from `pcs_tti.factpartnersoftwaredesignation`
- Marketplace billed sales trailing 12 months
- available MACC commitment signal
- Frontier Accelerate enrollment when a source is available

Render incentive status as exactly one of:

- `Confirmed eligible` - all required gates are grounded
- `Potential fit - validate` - partner type or commercial signal aligns, but one or more gates are not confirmed
- `Not evidenced` - reachable sources do not show the required gates

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

Persist the resolved primary PDM as the `PDM Owner` value for downstream SharePoint storage. Use the PMX `_gps_primarypdm_value@OData.Community.Display.V1.FormattedValue` / `partnerAccountOwner` value as the preferred source, normalized to the display name without duplicating the role suffix. If multiple PMX partner management accounts exist, choose the primary PDM from the PMA record used for the rendered `Internal Contact`; if no single PMA is chosen, prefer the PMA with active project context and record the alternative PDMs in generation evidence.

Keep larger team lists and routing details out of the one-pager body.

### 9. Retrieve RQA Hero Products

Use the authenticated RQA Qualify report:

```text
https://msit.powerbi.com/groups/me/apps/a53cc2f2-af44-4451-b109-f48b0a7cd4e7/reports/3691bc09-1cd6-4161-98f0-75e8b8e6c4a5/7979e8d75d9d547bc19e?ctid=72f988bf-86f1-41af-91ab-2d7cd011db47&experience=power-bi
```

Retrieve the values for the `Hero Products` Partner Snapshot field as follows:

1. Open `Qualify - OKR2` in the existing authenticated Microsoft work session.
2. Set `PartnerOne Name` to the exact partner name used for the one-pager. Confirm the slicer shows that partner before reading product values.
3. Clear the `Fiscal Year` slicer and any page/report-level Fiscal Year filter so the report shows `All`. Do not default this lookup to FY27.
4. Clear any existing Hero Product selection so the `Hero Product` slicer shows `All`, then open its dropdown.
5. Capture every selectable Hero Product value available for that partner, excluding `All` and blanks. Deduplicate exact labels and preserve the report's spelling and punctuation.
6. Prefer the dropdown/model data over screenshot interpretation. If the dropdown is virtualized or truncated, capture an authenticated Power BI visual query and query `DimHeroProduct[Hero Product]` with `RPO[PartnerOne Name]` equal to the partner and `DimPCMGroup[Group Name]` equal to `OKR2`, with no Fiscal Year condition.

The Hero Product dropdown is not the same as the `_Measures[# Primary Hero Products for FY27]` card. Do not use that FY27 measure to populate `Hero Products`, and do not discard valid dropdown values when the primary-product card is blank.

Render the distinct values as a concise semicolon-separated list in `{{HERO_PRODUCTS}}` and record the evidence as `RQA Qualify - OKR2; Fiscal Year: All; PartnerOne Name: <partner>`. Compact long RQA labels for seller readability while preserving evidence with the original report labels: remove leading solution-area prefixes such as `AI:`, `Apps:`, or `Dev:`, drop parenthetical taxonomy such as `(AI Apps & Agents)` when it is redundant, and simplify product names to their recognizable Microsoft product names without changing meaning. Examples: `AI: Foundry Models - OpenAI (Standard)` -> `Azure OpenAI Foundry Models`; `Apps: Azure Container Apps Serverless GPU (ACA) (AI Apps & Agents)` -> `Azure Container Apps Serverless GPU`; `Apps: Azure Kubernetes Service (AKS) (AI Apps & Agents)` -> `Azure Kubernetes Service`; `Dev: GitHub Copilot (Business, Enterprise)` -> `GitHub Copilot`. If the authenticated, partner-filtered dropdown returns no values, render `grounded (none)`. If RQA is inaccessible or the partner filter cannot be confirmed, render `Validate`.

### 10. Gather public positioning

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

Use the GitHub-hosted reusable template as the canonical output reference:

```text
https://github.com/tmathew1000/PartnerOnePager/blob/main/partner-one-pager-template.html
```

The template is named `partner-one-pager-template.html` and is the sole authority for the reusable HTML structure, visual styling, Microsoft logo SVG, iconography, proportions, spacing, and replacement placeholders. Read the GitHub template before rendering every partner one-pager. If the template cannot be accessed, report that limitation rather than silently using a stale local copy.

Render by filling the canonical GitHub template, not by recreating or hand-authoring an approximate layout. The rendered HTML must preserve the template's section order, Partner Snapshot rows, Seller opportunity fields, Opportunity Signals rows, CTA structure, labels, and placeholder-to-content mapping exactly. If a field is unavailable, keep the template field and populate it with `Validate` or omit only where the skill explicitly says omission is allowed.

### Required page structure

Use a single portrait page, approximately 980px wide and 1280px tall, with this order:

1. `Microsoft Confidential` pill at the top.
2. Top logo row with partner logo and Microsoft logo separated by a vertical divider.
3. Top-right slanted/parallelogram category ribbon.
4. Hero section with large partner + Microsoft title, blue headline, short summary, and a right-side rounded Partner Snapshot card. The left hero block and Partner Snapshot card must have matching heights. Size and vertically center the snapshot text so it uses the available card space without overflowing. Include `Hero Products` populated from the RQA Hero Product dropdown with Fiscal Year set to `All`, `Customer segments` populated from observed attributed ACR or co-sell activity, and `Internal Contact` populated from the primary PDM in PMX, formatted as `<PDM name> (PDM)`. Group commercial/public-sector variants under their segment family when needed and do not imply these are declared target segments.
5. Better Together statement.
6. Three large rounded solution cards connected by circular plus icons.
7. Compact `Seller opportunity` strip with `Customer signal`, `Business outcome`, `Microsoft pull-through`, `Incentive`, and `Next action`.
8. Split middle section: left `When to engage` checklist, right `Key use cases` row.
9. Small proof strip under use cases for the most relevant grounded commercial or quota signal.
10. `Why sellers should care` section: three stacked signal -> impact -> Microsoft-motion cards on the left and a compact `Opportunity Signals` card on the right containing only `Co-sell contract value`, `Registered co-sell deals`, `Partner Close Rate`, and `MACC Eligibility`. Format `Opportunity Signals` as a two-column list: large bold metric values in the left column and each signal title plus concise scope text in the right column, with subtle horizontal dividers between rows. Omit unavailable allowed fields. Keep this section vertically compact by minimizing margins and padding without reducing font or icon sizes.
11. Rounded Call to Action footer with partner logo, CTA copy and links, and Microsoft logo.
12. Compact footer with copyright/update date. Do not render a visible source or validation note.

Keep copy concise: 3 seller reasons, 4 use cases, 5 engagement signals, and 3 CTA links maximum.

### Visual style

Match the GitHub-hosted `partner-one-pager-template.html` as the visual source of truth.

- Use the mandatory `/web-artifacts-builder` Clawpilot theme script and CSS variables.
- Preserve the Microsoft field-collateral feel: white surface, bold rounded orange-red outer border, Microsoft blue accents, muted gray body copy, rounded cards, and compact executive-scan typography.
- Use inline SVG logos/icons where possible. Do not use placeholder logo boxes when a partner logo can be safely embedded or represented from an existing approved local/reference asset.
- Use actual Microsoft logo blocks as inline SVG, not plain text.
- Use compact section density so the full artifact fits on one printable page.
- Keep all colors expressed through `var(--cp-*)` except inside trusted inline SVG logo artwork.
- Keep validation caveats and internal-only provenance out of the rendered page; preserve them in the generation evidence and final response.

## Partner profile JSON requirements

Alongside the seller-facing HTML, generate a machine-readable partner profile JSON file named:

```text
[PARTNER NAME]-partner-profile.json
```

The JSON is an internal data artifact for downstream matching and retrieval. It must include:

- `schemaVersion`
- `generatedAt`
- `partner` identity fields: PartnerOneID, PartnerOneName, display name, PMX partner management account used, subsegment, website, public marketplace publisher.
- `renderedOnePager` fields: headline, summary, compact Hero Products, original RQA Hero Product labels, industries, customer segments, marketplace availability, solution cards, seller opportunity, engagement signals, use cases, seller reasons, CTA fields, internal contact.
- `metrics` with numeric values and period/scope: PI ACR association mix, co-sell contract value, registered co-sell records, partner close rate breakdown, customer Azure consumption, marketplace billed sales, MACC/customer commitment context, credentials/designations.
- `marketplaceOffers` with offer names, IDs, offer type, transactable/MACC/free plan/trial signals, storefronts, and public URLs when available.
- `pmx` with selected PDM owner, PTS count/representative contact, primary partner contact, and project-context summary.
- `evidence` as an array of source records. Every metric must preserve source system, query/filter context, period/as-of date, scope, confidence label, and any caveat.
- `validation` with fields left as `Validate`, fields that returned `grounded (none)`, and any omitted Opportunity Signals.

Rules:

- Preserve original source labels in evidence even when the rendered one-pager uses compact labels.
- Do not put secrets, bearer tokens, browser headers, raw network payloads, or unredacted temporary file paths in the JSON.
- Do not copy confidential internal links into customer-facing CTA fields. Internal SharePoint, PMX, Lakehouse, and RQA links may be stored only as evidence context when useful and clearly marked internal.
- Use stable numeric types for metrics, not formatted currency strings, and include separately formatted display values where helpful.
- If a value is ambiguous, conflicting, or not retrievable, write `Validate`; if a source was queried and returned zero rows, write `grounded (none)`.

## SharePoint publishing requirements

Save the finished HTML one-pager to the Partner One Pager SharePoint document library:

```text
https://microsoft.sharepoint.com/teams/PartnerOnePager/Partner%20One%20Pagers/Forms/AllItems.aspx
```

Save the generated partner profile JSON to the Partner Profile JSON SharePoint document library:

```text
https://microsoft.sharepoint.com/teams/PartnerOnePager/Partner%20Profile%20JSON/Forms/AllItems.aspx
```

Create or update the corresponding record in the Partner Matcher Index SharePoint list during the same publish operation:

```text
https://microsoft.sharepoint.com/teams/PartnerOnePager/Lists/Partner%20Matcher%20Index/AllItems.aspx
```

Use SharePoint-aware tooling for publishing. A complete publish includes all three steps: upload the HTML one-pager to `Partner One Pagers`, upload the JSON profile to `Partner Profile JSON`, and create or update the matching `Partner Matcher Index` row. The SharePoint document libraries and matcher index are the systems of record; do not keep a separate local copy as the deliverable. If local files must be created to support upload tooling, treat them as temporary artifacts and delete them after SharePoint upload and metadata/index verification succeeds.

For generated HTML one-pagers and JSON profiles, use SharePoint REST `Files/add` as the preferred first-run upload path. Do not create an empty Graph DriveItem placeholder before streaming content; that can leave a 0-byte file if the follow-up upload fails. Use browser file chooser upload only as a user-approved fallback when Scout file uploads are enabled.

### Optimal upload path (choose by scenario)

Pick the upload route by whether the target file already exists, to avoid slow retries:

- Replacing a file that already exists in the target library: overwrite the existing SharePoint file in place using the authenticated SharePoint REST `Files/add(...,overwrite=true)` flow below. `workiq_upload_file` may be used only when it accepts the existing file's `sharePointUrl` or `driveId + itemId`; if it returns an invalid-argument or access-denied error, switch directly to SharePoint REST instead of creating an incremented filename.
- Creating a brand-new file (the common one-pager and JSON-profile case): use the authenticated browser-side SharePoint REST `Files/add` flow below. Scout's current `workiq_upload_file` wrapper does not expose a create-in-folder operation (`driveId + parentFolderItemId + fileName`), so browser REST is the reliable first-run path today.
- If a direct Microsoft Graph upload tool is available (`PUT /drives/{driveId}/items/{folderItemId}:/{fileName}:/content` with the token handled internally), prefer it over the browser for new files. Do not scrape bearer tokens from browser sessions or token caches.

Operational gotchas that cause slow retries — avoid them up front:

- Write or copy the generated HTML and JSON into a browser-accessible root (the Microsoft Scout working directory), not a session-only `.scout` path. Playwright helper scripts can only read from allowed roots.
- The outer Playwright runtime has no `require`, `process`, or `atob`. Pass the file as base64 into `page.evaluate` and decode with `atob` inside the page context; upload the resulting `Uint8Array` as the fetch body.
- Do not rely on the OS file chooser (`browser_file_upload`); it is disabled unless the user enables Scout file uploads.
- Build the SharePoint browser session once, then reuse it for the digest, upload, readback, and metadata update.

1. Resolve the SharePoint site with `workiq_list_sharepoint_lists` against `https://microsoft.sharepoint.com/teams/PartnerOnePager`. Locate these targets by display name or list name:
   - `Partner One Pagers` document library, server-relative folder path `/teams/PartnerOnePager/Partner One Pagers`.
   - `Partner Profile JSON` document library, server-relative folder path `/teams/PartnerOnePager/Partner Profile JSON`.
   - `Partner Matcher Index` list.
   Capture `siteId`, library/list IDs, existing filename state, and target web URLs. Treat `Forms/AllItems.aspx` and `AllItems.aspx` URLs as entry points only; after resolution, use the returned IDs and list/library names for follow-up schema, metadata, item lookup, and verification. If SharePoint reads are throttled, do not guess writes; report the throttling blocker.
2. Preserve the human-readable filename `[PARTNER NAME]-one-pager.html` for HTML and `[PARTNER NAME]-partner-profile.json` for JSON. If a file with the same name already exists, overwrite it in place so the partner has one current SharePoint one-pager and one current JSON profile. Create an incremented filename, e.g. `[PARTNER NAME]-one-pager-2.html` and `[PARTNER NAME]-partner-profile-2.json`, only when the user explicitly asks to preserve the existing version as a separate historical copy.
3. Ensure a browser session is signed into `https://microsoft.sharepoint.com/teams/PartnerOnePager`. Get a SharePoint request digest by POSTing to `https://microsoft.sharepoint.com/teams/PartnerOnePager/_api/contextinfo` with `credentials: 'include'` and `Accept: application/json;odata=nometadata`.
4. Upload the actual HTML bytes directly to the HTML library with SharePoint REST:

   ```text
   POST https://microsoft.sharepoint.com/teams/PartnerOnePager/_api/web/GetFolderByServerRelativeUrl('/teams/PartnerOnePager/Partner%20One%20Pagers')/Files/add(url='[FILENAME].html',overwrite=[true|false])
   Headers:
   - Accept: application/json;odata=nometadata
   - Content-Type: text/html
   - X-RequestDigest: [FormDigestValue]
   Body: raw UTF-8 HTML bytes
   ```

5. Upload the actual JSON bytes directly to the JSON library with SharePoint REST:

   ```text
   POST https://microsoft.sharepoint.com/teams/PartnerOnePager/_api/web/GetFolderByServerRelativeUrl('/teams/PartnerOnePager/Partner%20Profile%20JSON')/Files/add(url='[FILENAME].json',overwrite=[true|false])
   Headers:
   - Accept: application/json;odata=nometadata
   - Content-Type: application/json
   - X-RequestDigest: [FormDigestValue]
   Body: raw UTF-8 JSON bytes
   ```

6. Verify both uploaded SharePoint drive items can be read back and that each `size` / `FileSizeDisplay` is non-zero and matches the local byte length. If either uploaded file is 0 bytes, treat publishing as failed: delete or replace the placeholder before proceeding.
7. Read the `Partner One Pagers` document library schema with `workiq_get_sharepoint_list_schema` and find the API-facing column whose display name is `PDM Owner`.
8. Update the uploaded HTML document's list item metadata so `PDM Owner` stores the resolved primary PDM for the one-pager record. If `PDM Owner` is a person field, resolve the PDM to a Microsoft 365 user and update the field with SharePoint `ValidateUpdateListItem`, using a value like `[{"Key":"i:0#.f|membership|alias@microsoft.com"}]`. If the column is text, store the normalized PDM display name string. Do not create or rename SharePoint columns unless the user explicitly asks.
9. Upsert one item in the `Partner Matcher Index` list keyed by `PartnerOneID` when present; otherwise key by exact `PartnerName`. Use `workiq_get_sharepoint_list_schema` to confirm API-facing column names at runtime. The known API-facing columns are:
   - `Title`: use `<PartnerName> (<PartnerOneID>)`.
   - `PartnerOneID`
   - `PartnerName`
   - `Industries`
   - `SolutionAreas`
   - `UseCases`
   - `MarketplaceOffers`
   - `MACCEligible`
   - `CoSellDealCount`
   - `CoSellContractValueUSD`
   - `PartnerCloseRate`
   - `PDM`
   - `SearchText`
   - `ApprovalStatus`
   - `IndexStatus`
   - `ApprovedBy`
   - `ApprovedDate`
   - `LastIndexedDate`
   - `SourceHtmlETag`
   - `OnePagerUrl`
   - `JsonUrl`

   Populate the index from the same grounded profile JSON. Store list-like values as semicolon-separated text. At upload time set `ApprovalStatus` to `Pending`, set `IndexStatus` to `Pending Approval`, leave `ApprovedBy` and `ApprovedDate` blank unless they are already confirmed from SharePoint approval metadata, and set `LastIndexedDate` to `utcNow()`. Store the uploaded document ETag in `SourceHtmlETag`, the HTML URL in `OnePagerUrl`, and the JSON URL in `JsonUrl`. The approval Power Automate flow owns the post-approval update: approved one-pagers become `ApprovalStatus = Approved` and `IndexStatus = Current`; denied one-pagers become `ApprovalStatus = Needs Revision` and `IndexStatus = Needs Revision`.
10. Verify the uploaded HTML document item can be read back, `PDM Owner` metadata is populated, the JSON document item can be read back, and the matcher-index item contains the expected PartnerOneID, URLs, key metrics, `ApprovalStatus = Pending`, and `IndexStatus = Pending Approval`. If upload, metadata update, or index upsert is blocked by permissions, throttling, missing tools, missing/ambiguous column schema, or an unsupported column type, report the blocker clearly and keep any temporary local files only if they are needed for user recovery.

Do not expose internal evidence notes inside the uploaded HTML. Store full evidence in the JSON profile and operational routing values in SharePoint metadata/list fields only.

## Final response

Report back in three bullets or fewer:

- SharePoint location or upload blocker
- confirmed internal metrics used, with source systems
- fields left as `Validate`, fields that returned `grounded (none)`, the `PDM Owner` metadata value saved to SharePoint, JSON profile location, and Partner Matcher Index upsert status
