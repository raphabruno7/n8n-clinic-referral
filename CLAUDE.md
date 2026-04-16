# n8n Clinic Referral Automation — CLAUDE.md

## Project Overview

Portfolio project built to demonstrate n8n automation skills on Upwork.
Automates the full patient referral flow for clinics: a Google Form submission
triggers a workflow that generates a signed Word PDF via Microsoft Graph API
and sends it via Outlook to a partner center, then tracks status back in Google Sheets.

Built for jobs like: "n8n clinic referral automation", "Word template PDF Outlook n8n",
"Google Sheets to email automation". Stack matches exactly what Upwork clients request.

GitHub: https://github.com/raphabruno7/n8n-clinic-referral

---

## Architecture

### Stack
- **n8n** (self-hosted, VPS) — workflow orchestration
- **Google Forms → Google Sheets** — patient data entry and source of truth
- **Microsoft Graph API** — fetch Word template from SharePoint, fill, convert to PDF
- **Microsoft Outlook** — send email with PDF attachment
- **JavaScript (n8n Code node)** — XML manipulation of .docx to fill 13 placeholders
- **python-docx** — used offline to generate the 4 Word template files

### Folder Structure
```
n8n-clinic-referral/
├── workflows/
│   ├── clinic-referral-main.json       # 14 nodes — main referral flow
│   └── clinic-referral-reminder.json   # 7 nodes — 7-day follow-up cron
├── templates/
│   ├── referral_5sessions.docx         # Word template with {{PLACEHOLDERS}}
│   ├── referral_10sessions.docx
│   ├── referral_15sessions.docx
│   └── referral_20sessions.docx
├── sheets/
│   ├── sample-data.csv                 # Fake patient data for demo
│   └── centros.csv                     # Partner centers lookup table
├── docs/
│   └── README.md                       # 1-page setup guide
├── .env.example                        # All required env vars documented
└── .gitignore
```

### WF-1 Node Flow (clinic-referral-main)
```
Google Sheets Trigger (polling 5min)
  → IF: status = "" ?
  → Google Sheets Read (lookup partner center email)
  → Switch: numero_sesiones (5/10/15/20)
  → HTTP Request (fetch Word template from SharePoint)
  → Code JS (fill 13 placeholders in .docx XML)
  → HTTP Request (upload filled .docx to SharePoint)
  → HTTP Request (Graph API: Word → PDF ?format=pdf)
  → Microsoft Outlook (send HTML email + PDF attachment)
  → Google Sheets Update (status=enviado, timestamp, pdf_link, email_link)
  → Error Trigger (parallel) → Mark Row as Error
```

### WF-2 Node Flow (clinic-referral-reminder)
```
Schedule Trigger (cron: 0 8 * * 1-5)
  → Google Sheets Read (filter status=enviado)
  → Code JS (filter rows older than 7 days)
  → IF: has pending follow-ups?
  → Google Sheets Read (lookup centro email)
  → Microsoft Outlook (send follow-up in Spanish)
  → Google Sheets Update (status=reminder_sent)
```

---

## Current State

### Done ✅
- 2 workflow JSONs created and imported into local n8n (ids: fvsZfjEIXkVIC21N, TSjCgjulJ8qyZvVs)
- 4 Word templates with 13 Spanish placeholders generated via python-docx
- sample-data.csv and centros.csv with fake data
- README.md with 5-step setup guide
- .env.example with all required variables documented
- GitHub repo published: raphabruno7/n8n-clinic-referral
- Portfolio GIF created: ~/Desktop/n8n_clinic_portfolio.gif (1.56MB)
- Upwork portfolio entry text written (600 char description + skills)

### Not Done / Pending ⏳
- Workflows not tested end-to-end (no real Graph API + Outlook credentials)
- Word templates not yet uploaded to SharePoint (client would do this)
- No demo video recorded yet (user will do manually)
- Tags in workflow JSONs were objects `{name: "x"}` — fixed to strings for REST API import

---

## Decisions & Rationale

| Decision | Why |
|---|---|
| Graph API for Word→PDF | Exact same stack as the Upwork job that inspired this project |
| Switch node for session count | Cleaner than IF chains; easy to extend with more session options |
| XML manipulation in Code JS (not a library) | n8n Code node has no file system access; JSZip + XML string replace is the correct approach |
| python-docx for template generation | Offline script, no dependencies in the workflow itself |
| Credentials removed from JSON | Portable import — user connects their own Google/Microsoft credentials after import |
| Tags as strings in REST import | n8n REST API v2.6.4 rejects `{name: "x"}` objects in tags array — must be plain strings |
| Error handler as parallel branch | n8n errorTrigger runs as a separate subworkflow; marks Sheet row as "error" without blocking main flow |

---

## Next Steps (priority order)

1. **Record demo video** (2–3 min) — show workflows list → open WF-Main → explain nodes → open WF-Reminder → explain cron
2. **Upload GIF + video to Upwork portfolio** entry (form already filled: title, role, description, skills)
3. **Test end-to-end** with real Microsoft Graph credentials (user has them configured) — fill 1 row in Sheet → verify email arrives with PDF
4. **Upload Word templates to SharePoint** and update TEMPLATE_ID_* env vars in n8n
5. **Add README screenshots** to GitHub repo for better visual appeal

---

## Conventions

- **Placeholders** in Word templates: `{{UPPER_SNAKE_CASE}}` — e.g. `{{NOMBRE_PACIENTE}}`
- **13 placeholders**: NOMBRE_PACIENTE, NUMERO_HISTORIA, EXPEDIENTE, NUMERO_SS, CENTRO_DERIVADO, FECHA, NUMERO_SESIONES, DIAGNOSTICO, MEDICO_REMITENTE, ESPECIALIDAD, TELEFONO_PACIENTE, FECHA_NACIMIENTO, OBSERVACIONES
- **Sheet column names**: lowercase snake_case, Portuguese (matches existing content-engine pattern)
- **Workflow names**: `Clinic Referral — [Purpose]`
- **Node names**: descriptive English, action-first — e.g. "Fetch Template 5s", "Fill Placeholders (JS)", "Update Sheet Status"
- **Status values in Sheet**: `""` (pending) → `"enviado"` → `"reminder_sent"` / `"error"`
- **n8n env vars**: referenced as `{{ $env.VAR_NAME }}` inside node parameters
- **Email language**: Spanish throughout (subject, body, all placeholders)
