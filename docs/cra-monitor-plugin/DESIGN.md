# CRA Monitor: Obsidian Plugin Design

*Working name: **CRA Monitor**. Status: design draft, 2026-11-22.*

A plugin for a Clinical Research Associate (CRA) who monitors several studies, each with several sites. It turns an Obsidian vault into a personal monitoring cockpit. It gives you structured notes for studies, sites and visits, visit checklists for each visit type, action items that carry forward until they close, deadline tracking for reports and letters, and a dashboard across your whole portfolio.

> **Positioning.** The plugin is a *personal working tool*. It does not replace the sponsor or CRO systems of record (CTMS, eTMF, EDC, IRT, safety database). It helps you prepare, capture, track and draft. The official record still goes into the official systems.

---

## 1. Research: the CRA as site monitor

### 1.1 Regulatory frame
- **ICH E6(R3) GCP** took effect in the EU on 23 July 2025, replacing R2. Its **Annex 2**, which covers decentralised and pragmatic designs, takes effect on 15 January 2027. R3 makes risk-based, proportionate monitoring the default. It combines on-site, remote and centralised monitoring, and it focuses on *critical-to-quality* factors instead of 100% SDV.
- The monitoring obligations sit in the sponsor's section on monitoring (§3.11.4 in R3, §5.18 in R2). The **Monitoring Plan** for each study sets visit frequency, SDV/SDR scope, report timelines and escalation rules. *Every timeline in the plugin must be configurable for each study*, because sponsors and CROs differ.

### 1.2 What a site monitor does
| Area | Typical CRA activities |
|---|---|
| **Site selection & start-up** | Pre-study / site qualification visit (PSV/SQV). Feasibility. Collect essential documents (CVs, GCP certs, 1572/IC forms, lab certs and normal ranges, contract and budget, EC/IRB approval). |
| **Site initiation (SIV)** | Protocol, IP, EDC and safety-reporting training. Training log and delegation log (DoA) sign-off. Check pharmacy and storage readiness. Site activation (green light). |
| **Interim monitoring (IMV / RMV)** | Informed consent review, including versions and re-consent after amendments. Eligibility verification. SDV/SDR of critical data. AE/SAE review and reporting timeliness. Protocol deviations. IP accountability, temperature excursions and storage. ISF review. Query follow-up. Enrollment and retention. Staff changes and retraining. |
| **Centralised / remote** | Review KRIs and risk signals, EDC data and query ageing. Remote SDV where allowed. Phone or video contacts. |
| **Close-out (COV)** | Final IP reconciliation and return or destruction. Final query resolution. ISF completeness and archiving. Notify the EC/IRB. Close remaining action items. |
| **After every visit** | **Confirmation letter** before the visit. **Monitoring visit report (MVR)** to the sponsor, usually due in 5–15 working days. **Follow-up letter (FUL)** to the site, usually due in 10–20 working days. Update CTMS and file documents to the eTMF. |
| **Between visits** | Site contacts and communication log. Track action items. Escalations and CAPA. Audit and inspection readiness support. |

### 1.3 Pain points the plugin should address
1. **Context switching** across many studies × sites, each with its own protocol version, monitoring plan, timelines and contacts.
2. **Action items fall through the cracks.** Open items must carry forward from visit to visit and appear again in each follow-up letter until resolved.
3. **Deadline pressure.** Report and letter due dates, the next-visit window, SAE reporting timelines and document expiry dates (CVs, GCP certificates, lab certifications) all compete for attention.
4. **Duplicate writing.** Notes taken on site end up re-typed into the MVR, the FUL and CTMS.
5. **Preparing for a visit** means pulling together the open items, the last visit's findings, enrollment status, the subjects due for SDV and protocol amendments.
6. **No portfolio view.** It is hard to answer "Which of my sites needs attention this week?"

---

## 2. Design principles
1. **Plain Markdown first.** Every entity is a note with YAML frontmatter, and every action item is a normal Markdown task. If you uninstall the plugin, your data is still readable. The format is compatible with the **Tasks** and **Dataview** plugins, but neither is required.
2. **Convention over configuration**, with study-level overrides for timelines, checklists and templates.
3. **Capture once, reuse many times.** Notes taken on site flow into the MVR draft, the FUL draft and the next visit's prep pack.
4. **Privacy by design.** No PHI or PII about subjects, only de-identified subject IDs. Local-only, with no network calls. Guardrails are described in §7.
5. **Fast in the field.** One command for quick capture, mobile-friendly views, and it works offline.

---

## 3. Data model

### 3.1 Vault layout (default, configurable)
```
CRA/
  Studies/
    ABC-123/
      ABC-123.md                  ← Study note
      Sites/
        101 - City Hospital/
          101 - City Hospital.md  ← Site note
          Visits/
            2026-11-18 IMV-04.md  ← Visit note
          Contacts.md
          Deviations.md
          Comms Log.md
  Templates/CRA/                  ← user-editable templates
  Dashboards/                     ← generated or Dataview dashboards
```

### 3.2 Entity schemas (frontmatter)

**Study**
```yaml
cra-type: study
study-id: ABC-123
sponsor: Acme Pharma
cro: ExampleCRO
indication: NSCLC
phase: III
protocol-version: "4.0 (2026-06-01)"
status: active            # start-up | active | follow-up | close-out | closed
monitoring:
  imv-interval-weeks: 6
  mvr-due-days: 10        # working days after visit
  ful-due-days: 10
  confirmation-letter-days-before: 7
  sdv-strategy: "Targeted: ICF, eligibility, primary endpoint, SAEs"
  working-days: true
critical-data: [ICF, eligibility, primary-endpoint, SAE, IP-dosing]
```

**Site**
```yaml
cra-type: site
study: "[[ABC-123]]"
site-number: "101"
name: City Hospital
pi: "[[Dr. A. Cohen]]"
status: active            # selected | activated | enrolling | closed-to-enrollment | closed
activation-date: 2026-02-10
enrollment: { target: 12, screened: 9, screen-failed: 2, randomized: 7, discontinued: 1 }
sdv: { planned-subjects: 7, sdv-complete: 5 }
last-visit: "[[2026-11-18 IMV-04]]"
next-visit-due: 2026-12-30
risk-level: medium        # low | medium | high (your own judgement or from KRIs)
doc-expiry:
  - { doc: "PI GCP certificate", expires: 2027-01-15 }
  - { doc: "Lab normal ranges", expires: 2026-12-31 }
```

**Visit**
```yaml
cra-type: visit
study: "[[ABC-123]]"
site: "[[101 - City Hospital]]"
visit-type: IMV           # PSV | SIV | IMV | RMV | COV | for-cause | remote-contact
visit-number: 4
date: 2026-11-18
mode: on-site             # on-site | remote | hybrid
status: report-drafting   # planned | confirmed | done | report-drafting | report-submitted | ful-sent | closed
confirmation-letter-sent: 2026-11-10
mvr-due: 2026-12-02       # auto-calculated
mvr-submitted:
ful-due: 2026-12-02       # auto-calculated
ful-sent:
subjects-reviewed: [101-001, 101-004, 101-007]
```

**Action item** (inline task with Tasks-compatible metadata and plugin tags)
```markdown
- [ ] Re-consent 101-004 on ICF v5 #cra/action 👤 site ⚠️ major 📅 2026-12-05 ➕ 2026-11-18 🔗 IMV-04
```
- Fields: owner (`site` / `cra` / `sponsor`), severity (`minor` / `major` / `critical`), due date, date raised, the visit where it was raised, and an optional category (ICF, IP, ISF, safety, data, deviation, training, other).
- An ID such as `^ai-101-0042` gives the item a block reference, so later visits can embed it or report on it.

**Deviation**: a row or block in `Deviations.md` with an ID, subject ID, date, category, an important/not-important flag, date reported, and CAPA reference.

**Subject (optional, light)**: subject ID, status (screening / on-treatment / follow-up / discontinued), last SDV date, ICF version signed. **Never** name, DOB, initials or medical record number.

---

## 4. Features

### 4.1 MVP (v0.1)
| # | Feature | Details |
|---|---|---|
| F1 | **New Study / New Site wizards** | Modal that creates the folder structure and pre-filled notes. Site timelines inherit from the study. |
| F2 | **New Visit** | Pick study → site → visit type. Creates the visit note from the template for that visit type, with:<br>• a **checklist** for the visit type (see §5);<br>• an **Open action items** section, auto-filled from all open items at that site (carry-forward);<br>• auto-calculated `mvr-due`, `ful-due` and `next-visit-due`, counting working days and skipping your configured holidays. |
| F3 | **Action item engine** | Indexes `#cra/action` tasks across the vault. Commands: *Add action item*, which works from anywhere and asks for site, owner, severity and due date; and *Close action item*, which records the completion date and the visit it was resolved at. Items carry forward until closed. |
| F4 | **Portfolio dashboard** (custom view) | One screen for all studies and sites:<br>• **Due this week:** MVRs, FULs, confirmation letters, visits in their window, expiring documents;<br>• **Overdue** items in red;<br>• **Site cards:** enrollment against target, SDV progress, open items by severity, days since the last visit, risk level;<br>• filters by study, sponsor and risk. |
| F5 | **Site cockpit** (side panel) | When a study, site or visit note is open, the panel shows that site's open items, last 3 visits, upcoming deadlines, key contacts and the deviation count. |
| F6 | **Visit status workflow** | Status-bar control or command to move a visit through the lifecycle (planned → … → closed). Each step stamps its date. |
| F7 | **Quick capture** | A command or ribbon button that adds a timestamped finding to the current visit note under the chosen checklist section. You can promote the finding to an action item with one key. |

### 4.2 v0.2: drafting and prep
| # | Feature | Details |
|---|---|---|
| F8 | **Visit prep pack** | Generates a "Prep" section or note before a visit. It lists open items, the last visit's findings, enrollment changes, subjects due for SDV or SDR, new amendments and ICF versions, documents expiring soon and the agenda. It can also draft the **confirmation letter**. |
| F9 | **Follow-up letter draft** | Builds a Markdown draft for the site from visit findings and action items: new items, items still open from previous visits (as reminders), and items resolved since the last visit. Copy it out or export it to a .docx or PDF template. |
| F10 | **MVR draft scaffold** | Maps your checklist answers and findings into the MVR section structure, which you can configure for each sponsor template, so pasting into CTMS is quick. |
| F11 | **Deviation log** | Command to log a deviation, with a link to the visit, an action item and CAPA. Counts by category and importance appear on the site card. |
| F12 | **Communication log** | Fast entry for calls and emails with a site: date, contact, topic, outcome, follow-up. A follow-up can become an action item. |

### 4.3 v0.3+: insight and planning
- **Calendar / timeline view:** visits, windows and deadlines across sites, plus a **travel planner** that groups site visits by geography and week.
- **Metrics:** query ageing, action items open longer than N days, average days to close, screen-failure rate, visit compliance (visits inside the monitoring-plan window). You enter the numbers by hand or paste them from EDC or CTMS exports as CSV.
- **Document expiry tracker** with reminders.
- **Protocol amendment tracker:** for each study, the amendment version, EC approval date at each site, and whether site training and re-consent are complete.
- **Weekly review command:** creates a weekly note that summarises what changed, what is overdue and the plan for next week.
- **CSV import** of a site or visit schedule from CTMS exports.

---

## 5. Visit-type checklists (default templates, editable)

Each checklist is a Markdown section with tasks. Answers can be ✅ / ❌ / N/A plus a comment, and a ❌ can be promoted to an action item.

- **PSV/SQV:** PI interest and availability; patient population and recruitment strategy; competing studies; staff and qualifications; facilities (pharmacy, storage, temperature monitoring, −80 °C freezer, lab); EC/IRB process and timelines; source document and EHR access for monitors; archiving.
- **SIV:** protocol training; IP handling and accountability; EDC/IRT access; AE/SAE reporting and timelines; ICF process; DoA and training logs signed; ISF set up; lab kits and supplies; pharmacy ready; activation prerequisites.
- **IMV/RMV:**
  - **ICF**: every new subject, the correct version, the process documented, re-consents done.
  - **Eligibility**: verified for new subjects.
  - **SDV/SDR**: per the risk-based plan, with subjects and visits listed.
  - **Safety**: AEs and SAEs captured, reported on time, follow-ups done.
  - **Deviations**: identified and reported.
  - **IP**: accountability, storage, temperature logs and excursions.
  - **ISF**: current approvals, amendments, CVs, logs.
  - **Staff**: changes, DoA updated, training.
  - **Data**: queries open and ageing.
  - **Enrollment**: status and recruitment plan.
  - **Previous action items**: reviewed one by one.
  - **PI oversight**: discussed.
- **COV:** all data clean and queries closed; final IP reconciliation and return or destruction; ISF complete and reconciled with eTMF; EC/IRB notified; archiving arrangements; all action items closed; remaining samples or supplies handled.
- **For-cause / remote contact:** a free-form template with focus area, findings and escalation.

---

## 6. UX and commands

**Commands (command palette, hotkeys):**
`CRA: New study` · `CRA: New site` · `CRA: New visit` · `CRA: Quick capture finding` · `CRA: Add action item` · `CRA: Close action item` · `CRA: Advance visit status` · `CRA: Generate prep pack` · `CRA: Draft follow-up letter` · `CRA: Draft MVR` · `CRA: Log deviation` · `CRA: Log communication` · `CRA: Open portfolio dashboard` · `CRA: Weekly review`

**Views:** Portfolio dashboard (main pane) · Site cockpit (right side panel) · Action items board (grouped by site, owner or severity; kanban optional) · Calendar (v0.3).

**Status bar:** "3 overdue · 5 due this week" opens the dashboard when clicked.

**Settings:**
- root folder and folder layout;
- working days and holidays (by country);
- default timelines, which each study can override;
- checklist and template folder;
- severity labels;
- MVR section mappings for each sponsor template;
- privacy guard on or off and its patterns;
- date format.

---

## 7. Privacy, confidentiality and compliance guardrails
- **No PHI or PII.** Use only sponsor-assigned subject IDs. A built-in **PHI linter** warns when a CRA note contains likely identifiers: date-of-birth patterns, national ID or MRN-like numbers, email or phone numbers in subject context, or "Mr./Mrs. + Name" near subject IDs. It warns only and never blocks.
- **Sponsor confidentiality.** Protocol details and site performance are confidential. Check your employer's policy on keeping work notes on a personal device and in cloud sync. If needed, keep the CRA vault separate and encrypted, or on a company-managed device.
- **Do not publish.** *This repository is a public Digital Garden.* The plugin will mark every note it creates with `dg-publish: false`, and the CRA vault or folder should stay out of anything that syncs to the garden. Keep CRA work in a **separate vault**.
- **No network calls**, no telemetry, and no AI or cloud features by default.
- **Not the system of record.** The plugin never claims to be GCP-compliant documentation. Official MVRs, letters, deviation reports and eTMF filings go through sponsor and CRO systems.

---

## 8. Technical architecture
- **Stack:** TypeScript, the Obsidian Plugin API, and esbuild using the official sample-plugin scaffold. The UI uses Svelte or Preact for the dashboard and cockpit views (both lightweight).
- **Modules:**
  - `model/`: entity types and frontmatter parsing and validation.
  - `index/`: an in-memory index built from `metadataCache`, covering studies, sites, visits and action items. It updates on `metadataCache.on('changed')`, so no Dataview dependency.
  - `tasks/`: action-item parser and serializer, using the Tasks emoji format plus `#cra/action`.
  - `dates/`: working-day calculator, holidays and deadline rules.
  - `templates/`: template engine with placeholders such as `{{site}}`, `{{openActions}}` and `{{checklist:IMV}}`.
  - `views/`: Dashboard, SiteCockpit, ActionBoard and Calendar.
  - `commands/` and `modals/`: wizards and quick capture.
  - `privacy/`: PHI linter and publish guard.
  - `export/`: letter and MVR drafts to Markdown, with optional .docx later.
- **Writes** go through `app.fileManager.processFrontMatter` and `vault.process` so edits are safe and atomic.
- **Performance:** the index is built incrementally, which should handle about 10 studies × 20 sites × 30 visits easily.
- **Mobile:** all views are responsive, with no Node or Electron APIs, so it works in Obsidian Mobile.
- **Testing:** unit tests for date rules, the task parser and carry-forward logic, plus a sample vault with fake data for manual QA.

---

## 9. Roadmap
| Milestone | Scope |
|---|---|
| v0.1 MVP | F1–F7, default templates and checklists, settings, privacy guard (basic) |
| v0.2 | F8–F12: prep pack, FUL and MVR drafts, deviation and comms logs |
| v0.3 | Calendar and travel planner, document expiry, amendment tracker, metrics, weekly review, CSV import |
| Later | .docx export using a sponsor template; optional Annex 2 / DCT visit types (eConsent, home-health visits) |

## 10. Open questions for you
1. Which visit types and report and letter timelines do your current sponsors or CRO use? These become the defaults.
2. Do you keep one vault for everything or a separate work vault? This affects the publish guard and folder defaults.
3. Do you already use the **Tasks**, **Dataview**, **Templater** or **Calendar** plugins? If so, the plugin stays format-compatible.
4. How do you get the numbers for enrollment, query counts and SDV status: manual entry or CTMS/EDC exports (CSV)?
5. Language: English only, or a bilingual (e.g. Hebrew) UI or templates?
6. Do you work on mobile or tablet during visits?
