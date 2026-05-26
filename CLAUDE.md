# Snowy Skips — Waste Disposal Management App

## Project overview

This is a waste disposal docket management system for **McKimm Civil Pty Ltd / Snowy Skips**, based in the Snowy Mountains region of NSW, Australia.

The system has three parts:
1. **Driver mobile app** — `snowy_skips_driver_app.html` — used by drivers on their phones to create and progress waste disposal dockets through a 4-step workflow
2. **Office desktop app** — `snowy_skips_office_app.html` — used by office staff to manage clients, view the kanban board, track nonconformances, and run environmental reports
3. **PDF docket system** — two versions of each docket: a client-facing copy (Sections 1–6) and an internal copy (all 7 sections including invoicing)

---

## Tech stack

- **Frontend:** Vanilla HTML, CSS, JavaScript — no frameworks, no build tools
- **Database:** Supabase (PostgreSQL) — `@supabase/supabase-js` v2
- **Auth:** Supabase Auth (email + password)
- **Storage:** Supabase Storage bucket `docket-photos` for stamped photo uploads
- **Hosting:** Netlify (static site, no server-side rendering)
- **PDF generation:** Python + ReportLab (run locally or via a serverless function)
- **Icons:** Tabler Icons webfont (loaded from CDN)

---

## Database schema

All tables live in Supabase. Schema is defined in `snowy_skips_deployment_guide.md`.

### Tables
| Table | Purpose |
|-------|---------|
| `clients` | Client folders — drives the driver dropdown and office board |
| `dockets` | One row per waste disposal docket — core data |
| `nonconformances` | ISO 14001:2026 NC register — linked to dockets |
| `audit_log` | Every action timestamped — stage changes, submissions, amendments |
| `photos` | Photo metadata — actual files in Supabase Storage |

### Key columns in `dockets`
- `stage` INTEGER 1–5 (1=Order, 2=Delivery, 3=Pickup, 4=Disposal, 5=Complete)
- `bins_delivered` INTEGER[] — array of bin numbers e.g. `{15, 19}`
- `bins_picked_up` INTEGER[]
- `waste_pcts` JSONB — e.g. `{"construction": 100}`
- `sigs` JSONB — e.g. `{"delivery": true, "pickup": true}`
- `has_nc` BOOLEAN — true if any nonconformance flagged

---

## Supabase credentials

> ⚠️ Never commit real credentials to Git. Store them in environment variables or a `.env` file that is listed in `.gitignore`.

Add a `.env` file to this folder (never commit it):
```
SUPABASE_URL=https://YOUR_PROJECT_ID.supabase.co
SUPABASE_ANON_KEY=YOUR_ANON_KEY_HERE
```

In the HTML files, initialise Supabase like this at the top of each `<script>` block:
```javascript
const SUPABASE_URL = 'https://YOUR_PROJECT_ID.supabase.co';
const SUPABASE_ANON_KEY = 'YOUR_ANON_KEY_HERE';
const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

Load the Supabase client in each HTML `<head>`:
```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

---

## User roles

| Role | Access | How set |
|------|--------|---------|
| Driver | Submit dockets, view own jobs, upload photos | Default (no role metadata) |
| Office | Full access — manage clients, view all dockets, NC register, reports | `raw_app_meta_data.role = "office"` set in Supabase Auth |

Check role in code:
```javascript
const { data: { session } } = await supabase.auth.getSession();
const isOffice = session?.user?.app_metadata?.role === 'office';
```

---

## Key functions to implement

These functions exist in the HTML as local/demo versions. Replace them with live Supabase calls:

### Driver app

```javascript
// Load client list for dropdown
async function loadClients() {
  const { data } = await supabase
    .from('clients')
    .select('*')
    .eq('active', true)
    .order('name');
  return data;
}

// Submit completed docket
async function submitDocket(docketData) {
  const { error } = await supabase.from('dockets').insert([docketData]);
  if (!error) {
    await supabase.from('audit_log').insert([{
      docket_id: docketData.id,
      action: 'Docket submitted — ' + docketData.client_name,
      action_type: 'create',
      performed_by: docketData.submitted_by
    }]);
  }
}

// Upload stamped photo to Supabase Storage
async function uploadPhoto(docketId, zone, dataURL) {
  const blob = await fetch(dataURL).then(r => r.blob());
  const path = `${docketId}/${zone}_${Date.now()}.jpg`;
  await supabase.storage.from('docket-photos').upload(path, blob, { contentType: 'image/jpeg' });
  await supabase.from('photos').insert([{ docket_id: docketId, zone, storage_path: path }]);
}

// Load jobs for a driver's folder view
async function loadJobs() {
  const { data } = await supabase
    .from('dockets')
    .select('*')
    .lt('stage', 5)
    .order('submitted_at', { ascending: false });
  return data;
}
```

### Office app

```javascript
// Real-time board — updates when any driver submits
supabase
  .channel('dockets')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'dockets' }, () => {
    renderBoard();
  })
  .subscribe();

// Add client (syncs to driver dropdown immediately)
async function addClient(data) {
  await supabase.from('clients').insert([data]);
}

// Remove client (soft delete — preserves docket history)
async function removeClient(id) {
  await supabase.from('clients').update({ active: false }).eq('id', id);
}

// Load NC register
async function loadNonconformances() {
  const { data } = await supabase
    .from('nonconformances')
    .select('*, dockets(client_name, location)')
    .order('created_at', { ascending: false });
  return data;
}
```

---

## Authentication flow

Both apps need a login screen shown before any content. Pattern:

```javascript
async function init() {
  const { data: { session } } = await supabase.auth.getSession();
  if (!session) {
    showLoginScreen();
  } else {
    showMainApp(session.user);
  }
}

// Listen for auth state changes
supabase.auth.onAuthStateChange((event, session) => {
  if (event === 'SIGNED_OUT') showLoginScreen();
  if (event === 'SIGNED_IN') showMainApp(session.user);
});
```

---

## Photo stamping

Photos are stamped client-side using HTML Canvas before upload. The stamp contains:
- Date and time in yellow (`#f5c842`) — format: `25 May 2026  2:30 PM UTC+10:00`
- `McKimm Civil Pty Ltd` in yellow
- `Snowy Skips` in white

The stamping function `stampImage(src, callback)` is already implemented in both HTML files. Do not change it.

---

## Docket ID format

IDs are generated as `MCK-001`, `MCK-002` etc. In production, generate sequentially from the database:

```javascript
async function generateDocketId() {
  const { count } = await supabase
    .from('dockets')
    .select('*', { count: 'exact', head: true });
  return 'MCK-' + String((count || 0) + 1).padStart(3, '0');
}
```

---

## Compliance requirements

This app is designed to meet **ISO 14001:2026** and **NSW EPA** waste tracking requirements:

- Every docket has an NSW EPA waste classification field
- VENM waste triggers a certificate capture flow (cert number, NATA lab, lab reference, test date)
- Nonconformances are logged per stage with corrective action and responsible person
- All docket amendments are recorded with reason and authorising person (ISO 14001:2026 Clause 6.3)
- Disposal facility EPA licence number is captured
- Disposal outcome (Landfill/Recycled/Recovered/Reused) and diversion % are tracked
- All records include a 7-year retention date
- A full audit trail timestamps every action

Do not remove any of these fields when modifying the forms.

---

## PDF generation

PDFs are generated using Python + ReportLab (`make_pdf_v3.py`).

- **Client copy** — `snowy_skips_docket_CLIENT.pdf` — Sections 1–6 only (no invoice/completion)
- **Internal copy** — `snowy_skips_docket_INTERNAL.pdf` — All 7 sections

The Snowy Skips logo is embedded as a base64-encoded PNG. The logo source file is at `logo_hires.png`.

To regenerate PDFs with live docket data, the Python script needs to accept docket data as input. This is a future enhancement — currently PDFs are generated with sample data.

---

## Folder / file structure (target)

```
snowy-skips/
├── CLAUDE.md                              ← this file
├── .env                                   ← Supabase credentials (never commit)
├── .gitignore
├── driver/
│   └── index.html                         ← snowy_skips_driver_app.html
├── office/
│   └── index.html                         ← office desktop app
├── pdfs/
│   ├── snowy_skips_docket_CLIENT.pdf
│   └── snowy_skips_docket_INTERNAL.pdf
├── scripts/
│   └── make_pdf_v3.py                     ← PDF generation script
├── open_docket.html                       ← Edge launcher
├── snowy_skips_deployment_guide.md
└── _redirects                             ← Netlify routing
```

---

## Netlify deployment

The app deploys as a static site on Netlify. No server required.

`_redirects` file content:
```
/driver/*   /driver/index.html   200
/office/*   /office/index.html   200
```

Target URLs:
- `https://app.snowyskips.com.au/driver/` — driver mobile app
- `https://app.snowyskips.com.au/office/` — office desktop app

---

## Business context

- **Company:** McKimm Civil Pty Ltd trading as Snowy Skips
- **Location:** Snowy Mountains region, NSW, Australia (UTC+10:00)
- **Operation:** Skip bin hire and waste disposal service
- **Named clients:** Bordeaux, Depot, Kitt Bryce Building, Lumina Constructions, Nex Gen Projects, OS Solutions, Prefabulous Homes, R D Miller, Rydges Hotel Jindabyne
- **General folder:** One-off residential and commercial jobs
- **Disposal facilities:** McKimm Depot, Cooma Landfill, Bombala Landfill, Jindabyne Landfill, CWT - Bega Valley, Albury Landfill, Canberra Mugga Lane
- **Bin range:** Bins 1–100
- **Currency:** AUD
- **Date format:** DD Mon YYYY (e.g. 27 Mar 2026)
- **Time zone:** UTC+10:00 (AEST)

---

## Common tasks for Claude Code

Here are prompts you can use directly in Claude Code:

**Preview the app locally:**
> "Start a local server for this project so I can open the driver app and office app in my browser"

**Connect to Supabase:**
> "Add Supabase to the driver app using my credentials in .env — replace the demo data with live database calls for clients and dockets"

**Add login screen:**
> "Add a login screen to both apps using Supabase Auth. Drivers and office staff log in with email and password"

**Deploy to Netlify:**
> "Set up the folder structure for Netlify deployment and deploy the site — give me a live URL"

**Connect custom domain:**
> "Walk me through connecting the domain app.snowyskips.com.au to the Netlify deployment"

**Test everything:**
> "Run through the full docket workflow — create a test docket as a driver, check it appears in the office board, move it through all stages, and verify the audit trail is correct"

**Regenerate PDF with real data:**
> "Update the PDF generation script to accept a docket ID, fetch the data from Supabase, and generate a properly populated PDF"

**Add a new feature:**
> "Add email notifications — when a driver submits a docket, send an email to alistair@snowyskips.com.au with the docket summary"

---

## Additional workflows to build — Site Safety & Incident Reporting

### Source app
**Dashpivot** (app.dashpivot.com) — McKimm Civil Pty Ltd / Snowy Skips workspace

### How to access
The reference forms are exported from Dashpivot and stored in `reference/dashpivot-forms/`.
Screenshots of the workflow board columns are in `reference/dashpivot-screenshots/`.

If direct browser access is needed:
> Tell Claude Code: "Open app.dashpivot.com, log in with my credentials, navigate to the Site Safety and Incident Reporting templates under McKimm Civil, read every field and workflow column, then rebuild them as new HTML apps in this project following the same design pattern as the driver waste disposal app"

### Design requirements for all new workflows
- Same visual style as the existing driver app — navy `#0a2240`, orange `#e8611a`, teal `#1D9E75`
- Same Snowy Skips logo in the top bar
- Same mobile-first layout for driver/field staff use
- Same Supabase database (add new tables for safety/incident data)
- Same folder/client structure — field staff see their assigned jobs/sites
- Same photo stamping system — all photos auto-stamped with date, time, McKimm Civil Pty Ltd, Snowy Skips
- Same signature system — tap to sign at each stage
- Same audit trail — every action logged with timestamp and user

### Safety & incident workflows to replicate

#### 1. Site Safety Inspection
Typical fields to capture:
- Site / location
- Inspector name and date
- Hazard identification (with photo)
- Risk rating (Low / Medium / High / Critical)
- Corrective action required
- Responsible person and due date
- Sign-off signature
- Follow-up status

Workflow columns (kanban):
- Scheduled → In Progress → Completed → Closed

#### 2. Incident / Near Miss Report
Typical fields to capture:
- Incident type (Injury / Near Miss / Property Damage / Environmental)
- Date, time and location
- Person(s) involved
- Description of incident
- Immediate actions taken
- Root cause analysis
- Corrective and preventive actions
- Photos (auto-stamped)
- Witness signatures
- Manager sign-off

Workflow columns:
- Reported → Under Investigation → Actions Required → Closed

#### 3. Toolbox Talk / Safety Meeting Record
Typical fields to capture:
- Date, site, facilitator
- Topic / agenda
- Attendees (with signatures)
- Key points discussed
- Action items

#### 4. SWMS — Safe Work Method Statement
Typical fields to capture:
- Task / activity description
- Hazards identified
- Control measures
- PPE required
- Signatures of all workers

### Database tables to add for safety workflows

```sql
CREATE TABLE safety_inspections (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  client_id UUID REFERENCES clients(id),
  site TEXT,
  inspector TEXT,
  inspection_date DATE,
  hazards JSONB,
  risk_rating TEXT,
  corrective_action TEXT,
  responsible_person TEXT,
  due_date DATE,
  stage TEXT DEFAULT 'scheduled',
  submitted_by TEXT,
  submitted_at TIMESTAMPTZ DEFAULT now(),
  sigs JSONB DEFAULT '{}'::jsonb
);

CREATE TABLE incidents (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  client_id UUID REFERENCES clients(id),
  incident_type TEXT,
  incident_date DATE,
  incident_time TIME,
  location TEXT,
  persons_involved TEXT,
  description TEXT,
  immediate_actions TEXT,
  root_cause TEXT,
  corrective_actions TEXT,
  stage TEXT DEFAULT 'reported',
  submitted_by TEXT,
  submitted_at TIMESTAMPTZ DEFAULT now(),
  sigs JSONB DEFAULT '{}'::jsonb
);
```

### Claude Code prompts to build these workflows

**Replicate from Dashpivot directly:**
> "Open app.dashpivot.com in a browser, log in, navigate to the Site Safety and Incident Reporting templates, read every field and workflow column carefully, then build them as new HTML apps following the exact same design pattern as snowy_skips_driver_app.html"

**Replicate from exported PDFs:**
> "Read all the PDF files in reference/dashpivot-forms/, understand the field structure of each safety and incident form, and rebuild them as mobile-first HTML apps matching the style of the existing driver app"

**Add to the office dashboard:**
> "Add a Safety & Incidents section to the office desktop app with a kanban board for safety inspections and a separate board for incidents, matching the style of the existing waste disposal board"

**Connect to Supabase:**
> "Add the safety_inspections and incidents tables to Supabase and wire up the new safety workflow apps to save and load from the database"
