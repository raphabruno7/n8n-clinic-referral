# n8n Clinic Referral Automation

Automate patient referrals from clinic to partner centers — end-to-end, no manual work.

- Reads new rows from Google Sheets (populated via Google Forms)
- Generates a signed referral PDF from a Word template (via Microsoft Graph API)
- Sends a formatted email in Spanish to the partner center with the PDF attached
- Updates the Google Sheet row with status, timestamp, and document links
- Sends automatic follow-up reminders after 7 days if no confirmation received

---

## Prerequisites

| Requirement | Notes |
|---|---|
| n8n (self-hosted) | v1.30+ recommended |
| Google Cloud project | Sheets API + OAuth2 credentials |
| Microsoft Azure app | Graph API with `Mail.Send`, `Files.ReadWrite`, `Sites.ReadWrite.All` |
| SharePoint site | Word templates uploaded (see `/templates`) |
| Microsoft Outlook account | For sending emails |

---

## Setup (5 steps)

**1. Import workflows**
- In n8n → Workflows → Import from file
- Import `workflows/clinic-referral-main.json`
- Import `workflows/clinic-referral-reminder.json`

**2. Configure credentials**
- Add `Google Sheets OAuth2` credential
- Add `Microsoft OAuth2` credential (Azure app client ID + secret)

**3. Upload Word templates to SharePoint**
- Upload the 4 `.docx` files from `/templates` to your SharePoint drive
- Copy each file's item ID from the Graph API URL

**4. Set environment variables**
Copy `.env.example` → `.env` and fill in all values (see below)

**5. Activate workflows**
- Open each workflow → toggle Active → confirm

---

## Environment Variables

```env
GOOGLE_SHEET_ID=           # Google Sheets document ID (from URL)
SHAREPOINT_SITE_ID=        # SharePoint site ID (from Graph API)
SHAREPOINT_DRIVE_ID=       # Drive ID within the site
TEMPLATE_ID_5S=            # SharePoint item ID for referral_5sessions.docx
TEMPLATE_ID_10S=           # SharePoint item ID for referral_10sessions.docx
TEMPLATE_ID_15S=           # SharePoint item ID for referral_15sessions.docx
TEMPLATE_ID_20S=           # SharePoint item ID for referral_20sessions.docx
```

---

## Google Sheet Structure

**Tab: Encaminhamentos** (main data, filled by Google Forms)

| Column | Description |
|---|---|
| nombre_paciente | Full patient name |
| numero_historia | Clinical history number |
| expediente | File/dossier number |
| numero_ss | Social security number |
| centro_id | Partner center ID (matches Centros tab) |
| numero_sesiones | 5 / 10 / 15 / 20 |
| fecha | Referral date |
| diagnostico | Diagnosis / reason for referral |
| medico_remitente | Referring physician name |
| especialidad | Specialty requested |
| telefono_paciente | Patient phone number |
| fecha_nacimiento | Patient date of birth |
| observaciones | Additional notes |
| status | *(auto-filled)* enviado / reminder_sent / error |
| timestamp | *(auto-filled)* ISO timestamp |
| pdf_link | *(auto-filled)* SharePoint PDF link |
| email_link | *(auto-filled)* Outlook message ID |

**Tab: Centros** (partner center lookup)

| Column | Description |
|---|---|
| centro_id | Unique ID (e.g. CTR-001) |
| nombre | Center display name |
| email | Email address for referral delivery |

---

## Customization

- **Add more sessions options** — duplicate a `Fetch Template Xs` node and add a branch in Switch
- **Change email language** — edit the HTML body in the `Send Email (Outlook)` node
- **Add more placeholders** — add to the `placeholders` object in the `Fill Placeholders (JS)` Code node and update the `.docx` templates
- **Change reminder interval** — edit the JS filter in `Filter > 7 days ago` (line: `setDate(getDate() - 7)`)

---

## Workflow Diagram

```
Google Forms
     ↓
Google Sheets (new row trigger, every 5 min)
     ↓
IF: status = "" ?
     ↓ yes
Lookup partner center email (Centros tab)
     ↓
Switch: 5 / 10 / 15 / 20 sessions
     ↓
Fetch Word template from SharePoint (Graph API)
     ↓
Fill 13 placeholders in .docx XML (Code JS)
     ↓
Upload filled .docx to SharePoint (Graph API)
     ↓
Convert Word → PDF (Graph API ?format=pdf)
     ↓
Send email via Outlook (HTML + PDF attachment)
     ↓
Update Sheet row: status=enviado, timestamp, links

--- Follow-up (separate workflow) ---
Schedule: Mon–Fri 08:00
     ↓
Read rows with status=enviado > 7 days
     ↓
Send reminder email via Outlook
     ↓
Update Sheet row: status=reminder_sent
```

---

Built by [Raphael Bruno](https://github.com/raphabruno7) · Ressonance Labs
