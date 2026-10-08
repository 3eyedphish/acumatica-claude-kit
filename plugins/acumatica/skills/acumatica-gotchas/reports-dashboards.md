# Reports, Dashboards & GI Design

Building reports in Report Designer, designing Generic Inquiries as a reporting source, and dashboards
and pivot tables built on them. For *consuming* a GI over OData (paging, parameters, property names),
see [giql-odata.md](./giql-odata.md); this file is the authoring side. Tag legend in [SKILL.md](./SKILL.md).

> Most entries here come from Acumatica documentation and community reports, not live tests in this
> kit. Their `Verified:` lines say so; update them when you confirm one.

---

### Pick the tool by the question: GI, Report Designer, or ARM `[UNIVERSAL]`
- **Generic Inquiry**: a live, filterable grid over DACs, editable in the browser, exportable to Excel,
  and the data source for dashboards, pivot tables and OData. Default for operational lists and
  anything an integration or AI agent will read.
- **Report Designer**: a pixel-laid-out, printable document (invoices, packing slips, statements,
  certificates of analysis). A Windows desktop tool that saves `.rpx` definitions to the server.
- **Analytical Report Manager (ARM)**: financial statements from GL balances, built from row sets,
  column sets and unit sets. Use it for P&L and balance sheets instead of forcing a GI or a Designer
  report to roll up accounts.
- *Unverified as a single documented rule; this is the standard split in Acumatica's own training.*

### Use the Report Designer build that **matches the instance version** `[UNIVERSAL]`
- Report Designer connects to the instance and saves into a tenant's database. A Designer build from a
  different release can misbehave. One reported case: a 24R2 Designer opened a report from one tenant,
  was told to save to another, and **wrote it to the original tenant**. The sign-in handling changed
  between 24R1 and 24R2. Installing the matching build fixed it.
- **Do:** keep one Designer install per instance version you maintain, and confirm the target tenant
  after saving (open the report there and check its version list).
- *Source: Acumatica Community thread "Report Designer changes not saved in correct tenant" (2024-era
  builds). Not yet verified live.*

### Save edits to a stock report **as a new version**, never over the stock one `[UNIVERSAL]`
- A stock report can carry custom **versions** stored in the tenant database; the version marked active
  is what users get. Clearing the active version falls back to the stock report.
- Upgrades update the **stock** report, not your versions. A custom version keeps working but silently
  misses Acumatica's fixes to the stock layout. If Acumatica removes the stock report (or a subreport)
  in a release, a version based on it can disappear after the upgrade.
- **Do:** save customizations with *Save as new version*, keep an exported `.rpx` of each in git, and
  after every upgrade diff your version against the new stock one in a test copy.
- To tell whether a report is customized: check its Report Versions list. A report that refuses to have
  its active version cleared has no stock fallback, so it is fully custom.
- *Source: Acumatica Community threads on report versions and upgrades (21R2 to 23R1 era). Not yet
  verified live.*

### Reports and GIs are **tenant data**: they don't move by publishing code, and they outlive unpublish `[UNIVERSAL]`
- A report saved from Report Designer, and a GI saved on the Generic Inquiry form, live only in that
  tenant's database. To move one to another tenant or instance, put it in a customization project (or
  export and import it) and publish to that tenant. See
  [customization-projects.md](./customization-projects.md) for the site-wide vs per-tenant split.
- Unpublishing a project **leaves** its reports and GIs in place, so retiring one is a manual cleanup.
- A **new** report or GI also needs a **site map** entry before users can find it in the menu and
  before access rights can be granted on it. Ship that entry in the same project.
- Report and GI **IDs and names** are `[TENANT]`; read them from config, never hardcode them.
- *Documented: "Unpublishing Customization Projects", "Multitenant Customization: General Information"
  (checked 2026-10-08). Not yet verified live.*

### GI design for reporting: fewer tables, real keys, group only when you aggregate `[UNIVERSAL]`
- A GI's run time is dominated by its joins and grouping, and it grows with data volume, so a GI that is
  instant in test can take minutes in production. Common causes: tables joined but not used in results
  or conditions, grouping nobody needs, joins that don't use the full key, and reading detailed cost
  tables where a **summary** table holds the answer (item *stats* rather than item *cost* rows).
- **Do:** join on the complete key (including the branch or site where the key has one), drop unused
  tables, filter on stored key and date columns, and time the GI against production-sized data before
  building dashboards on it.
- When you group, give every non-grouped result column an explicit aggregate rather than relying on
  what the platform picks for it.
- Remember that a **calculated** (`=…`) column cannot be sorted or filtered on server-side (see
  [giql-odata.md](./giql-odata.md)). Put the filter on a stored column the formula reads from.
- *Source: Acumatica Community "Performance issue on the generic inquiry" (a 1.26 M-row GI); the
  aggregate advice is unverified general practice.*

### Dashboard widgets and pivot tables **inherit the GI** and the viewer's rights `[UNIVERSAL]`
- Table, chart and KPI widgets (scorecard, meter, trend card) take their data from an inquiry, usually a
  GI. A pivot table is also built on a GI. They are only as fast and as correct as that GI, and a slow
  GI becomes a slow dashboard for everyone who opens it.
- A widget runs as the person viewing it. A user without access to the source GI sees an empty or
  locked widget, while the dashboard's author sees data. Grant rights on the GI, not just the dashboard.
- Widgets have **caching** settings. A widget that "doesn't show today's numbers" may be serving its
  cache, so check that before you debug the GI.
- Pivot tables can be much slower than the GI they're built on, because they aggregate the GI's full result
  set rather than one page of it. Pre-aggregate in the GI (grouping plus aggregates) when the pivot is slow.
- *Documented: "Configuring Widgets" (widget types and caching settings, checked 2026-10-08). Rights
  inheritance and pivot slowness are inferred from community reports; not yet verified live.*

### A GI built for people is often the wrong GI for an integration `[UNIVERSAL]`
- Reporting GIs tend to gain parameters, calculated columns, captions and grouping, and each of those
  changes what an OData consumer receives (silent-empty parameterized reads, unfilterable calculated
  columns, property names derived from display names; see [giql-odata.md](./giql-odata.md)).
- **Do:** keep a separate, plain GI for each integration (no parameters, stored columns, stable display
  names) and let the reporting GI evolve freely. Editing a column's display name on a shared GI renames
  the OData property and breaks the consumer without any error on the GI side.
- *Follows from the documented naming rule in giql-odata.md; the separation advice is practice.*
