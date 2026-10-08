# Customization Projects

Customization projects (the Customization Project Editor and Customization Projects form), DAC and
graph extensions, publishing, and how customizations surface to integrations. See the tag legend in
[SKILL.md](./SKILL.md).

> Most entries here are drawn from Acumatica's own documentation and community reports, not from live
> testing in this kit. Their `Verified:` lines say so. Confirm against your version before relying on
> one, and update the line when you do.

---

### Publishing is **all-or-nothing for the set**: an unselected project gets **unpublished** `[UNIVERSAL]`
- Publishing does not add one project on top of what is live. The platform removes every published
  project, merges the projects currently selected, and applies that merged result. A project that was
  live but is not selected in that publish is **silently unpublished**, whether you publish from the
  Customization Projects form (SM204505) or from inside one project's editor.
- That is also the documented way to unpublish one project: clear it and republish the rest.
- **Do:** before every publish, check the full list of projects that should be live and select all of
  them. Treat "publish just my project" as a change to the whole instance.
- **Don't** publish from a tenant other than the one you normally use: the form lists only the projects
  uploaded to the current tenant, so projects published elsewhere are invisible there and are
  unpublished by the merge.
- *Documented: "To Publish a Single Project", "Project Unpublishing: To Unpublish a Single Project",
  "Multitenant Customization: General Information" (current doc set, checked 2026-10-08). Not yet
  verified live in this kit.*

### Code and layout are **site-wide**; data items are **per tenant** `[UNIVERSAL]`
- On a multitenant site, compiled code, DAC/graph extensions, screen layout changes, custom tables and
  files land in the shared website, so **every tenant runs them**, including tenants you never
  published to.
- Database content from a project (generic inquiries, reports, site map entries, imported data) is
  written only into the tenant(s) you published to. A GI or report that "exists in test but not in
  live" on the same site is usually this, not a failed publish.
- **Do:** use *Publish to Multiple Tenants* when the data items need to reach several tenants. Once the
  website is already updated, its *Apply Changes Only to Database* option pushes just the data.
- **Do:** guard tenant-specific behavior in code with `IsActive()` (see below), because you can't scope
  code to a tenant by publishing.
- *Documented: "Multitenant Customization: General Information" (checked 2026-10-08). Not yet verified
  live.*

### **Unpublishing does not roll back the database** `[UNIVERSAL]`
- Unpublishing removes code, custom files and layout changes from the website. It leaves behind custom
  tables and columns and the data in them, along with site map entries, generic inquiries and reports
  the project created.
- Consequence: "unpublish to test the stock behavior" is not a clean revert, and a later re-publish
  meets the leftover columns and data.
- **Do:** clean up leftover site map entries, GIs and reports by hand after retiring a project, and
  remove custom columns deliberately (a script you own), never by assuming unpublish did it.
- *Documented: "Unpublishing Customization Projects" (checked 2026-10-08). Not yet verified live.*

### Custom columns on **stock tables** need the `Usr` prefix, or an upgrade can drop them `[UNIVERSAL]`
- A field added to an existing Acumatica table through a DAC extension must be named with the `Usr`
  prefix (`UsrRepairType`). The upgrade process rebuilds stock tables and can drop non-standard columns
  that lack the prefix. Columns on your **own** tables are not affected and can be named freely.
- For more than a few fields, prefer a **DAC extension table**: a separate table carrying the base
  table's key columns plus `CompanyID` and `DeletedDatabaseRecord`, mapped with `[PXTable]`. Its
  `IsOptional` setting chooses an inner or left join. Retiring it later is a table drop instead of
  columns stranded in a core table.
- Field *names* and *which* extensions exist are `[TENANT]`; never carry them into another repo.
- *Source: Acumatica blog "Creating Acumatica DAC Extension Tables" (reviewed 2023-03); community
  report of an upgrade dropping un-prefixed columns. Not yet verified live.*

### A custom field is **invisible to contract REST** until the endpoint is extended `[UNIVERSAL]`
- Publishing a DAC extension puts the field on the screen, not in the API. The stock `Default`
  endpoint's contract does not include it, so a normal GET leaves it out and a PUT has no contract field to
  carry it.
- **Read without changing the endpoint:** the `$custom` parameter returns non-contract fields:
  `$custom=<View>.<Field>` for header fields, `$custom=<Entity>/<View>.<Field>` (and `$expand` that
  entity) for detail lines, `$custom=<View>.Attribute<ID>` for user-defined attributes. View and field
  names come from the screen's data views, not the API's field names.
- **Durable path for reads and writes:** extend the endpoint (Web Service Endpoints, SM207060) and map
  the field. Include the extended endpoint in the customization project, or it does not travel to the
  next instance.
- The extended endpoint's **name and version** are `[TENANT]` (see
  [reference-vs-tenant.md](./reference-vs-tenant.md)); read them from config.
- *Documented: "$custom Parameter", "Custom Fields" in the Integration Development Guide (checked
  2026-10-08). Not yet verified live.*

### Gate extensions with `IsActive()`, and keep it cheap `[UNIVERSAL]`
- A graph or DAC extension with no `IsActive()` is always loaded, on every screen and API call that
  builds that graph, and Acumatica calls always-on extensions a performance cost.
- When `public static bool IsActive()` returns false, the extension's views, event handlers and
  `[PXOverride]` methods are all skipped, and controls bound to its views are hidden.
- Rules: don't create a graph inside `IsActive()` (documented deadlock risk); for a data-driven switch,
  read a cached database slot rather than querying each time; a higher-level extension needs its own
  `IsActive()`, because disabling only its base extension throws. Use `IsActiveForGraph<TGraph>()` to
  switch an extension per derived graph.
- *Documented: "To Enable a Graph Extension Conditionally (IsActive)" (checked 2026-10-08). Not yet
  verified live.*

### Override through the **base delegate**, don't re-implement `[UNIVERSAL]`
- In a graph extension, override a graph method with `[PXOverride]` and a matching signature that takes
  the base method as a delegate, then call that delegate. For events, the handler form that receives
  the base handler as a parameter lets you choose whether your logic runs before or after it.
- Copying the stock method body into the extension instead freezes it at today's version: the next
  upgrade fixes the stock method and your copy keeps the old bug, with nothing failing to tell you.
- **Do:** run the Acuminator analyzer on extension code; it flags many platform-misuse patterns at
  build time.
- *Unverified as a single documented rule; consistent with "Override of a Method: General Information".*

### **Validate** customization projects **before** an upgrade, in a copy `[UNIVERSAL]`
- Code in a project compiles against the instance's platform assemblies. A version upgrade can change
  or remove the APIs it uses, and a project that won't compile can block the publish that follows the
  upgrade. The Customization Projects form has validation actions for this.
- **Do:** restore a copy of production on the target version, publish the same project set there, and
  exercise the customized screens and API calls before upgrading production. Keep each project's
  exported `.zip` (or its source-control folder) in git, so the exact published package is recoverable.
- **Do:** for CI, the Customization Web API (`publishBegin` / `publishEnd`) and the PXCommandLine tool
  can import and publish without the UI.
- *Documented: Customization Projects (SM204505) form reference; "Customization Web API: publishBegin"
  (checked 2026-10-08). Upgrade-breakage risk is general practice, not a single verified incident.*

### Lock users out while publishing a production site `[UNIVERSAL]`
- Acumatica recommends locking the system during a publish. Screens and API calls hitting a graph while
  its extensions are being swapped can fail or act on a half-applied state. An integration's retry loop
  makes this worse.
- **Do:** schedule publishes in a window, pause integrations (or have them back off), then unlock.
- *Documented: "Publishing Customization Projects" (checked 2026-10-08). Not yet verified live.*
