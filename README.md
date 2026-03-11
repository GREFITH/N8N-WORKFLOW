# 🤖 AI Hiring System — n8n Automation

An end-to-end resume processing pipeline built on **n8n** that automatically monitors Gmail, Outlook, and Google Drive for incoming resumes, parses them using Azure AI, and stores structured candidate data in Google Sheets.

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     INTAKE LAYER                            │
│                                                             │
│  📧 Gmail Monitor    📧 Outlook Monitor    📁 Drive Monitor │
│  (real-time)         (real-time)           (real-time)      │
│                                                             │
│  📧 Hist. Gmail      📧 Hist. Outlook      📁 Hist. Drive   │
│  (one-time scan)     (one-time scan)       (one-time scan)  │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTP Webhook (multipart)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   PARSER LAYER                              │
│                                                             │
│   📥 Webhook Receiver                                       │
│        ↓                                                    │
│   🔀 File Type Router (PDF / DOCX / Image / Scanned PDF)    │
│        ↓                                                    │
│   🤖 Azure AI Extraction (GPT-4o + Doc Intelligence)        │
│        ↓                                                    │
│   🔁 Duplicate Check → NEW or UPDATED_RESUME                │
│        ↓                                                    │
│   💾 Google Sheets Upsert + Google Drive Archive            │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   STORAGE LAYER                             │
│                                                             │
│   📊 Google Sheets (RESUME / NO RESUME / PROCESSED_FILES)   │
│   ☁️  Google Drive (AI HIRE PROCESSED folder)               │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 Workflow Files

| File | Purpose | Trigger |
|---|---|---|
| `AI_Hiring_Resume_Parser_v1_6.json` | Core AI parser — receives all files | Webhook (always active) |
| `AI_Hiring_Gmail_Monitor_v3.0.json` | Watch Gmail inbox for resumes | Every 5 min |
| `AI_Hiring_Outlook_Monitor_v4.0.json` | Watch Outlook inbox for resumes | Every 5 min |
| `AI_Hiring_Drive_Monitor_v2.0.json` | Watch Google Drive intake folder | On new file |
| `AI_Hiring_Historical_Gmail_Scanner_v3.1.json` | Scan all past Gmail for resumes | Manual (run once) |
| `AI_Hiring_Historical_Outlook_Scanner_v2.0.json` | Scan all past Outlook for resumes | Manual (run once) |
| `AI_Hiring_Historical_Drive_Scanner_v1.1.json` | Scan existing Drive folder | Manual (run once) |

---

## ✨ Features

- **Multi-source intake** — Gmail, Outlook, Google Drive (real-time + historical)
- **AI-powered extraction** — Azure GPT-4o for text resumes, Azure Doc Intelligence OCR for scanned PDFs, GPT-4o Vision for image resumes
- **Smart deduplication** — Two-level: file hash (PROCESSED_FILES) + candidate email (RESUME sheet)
- **File type support** — PDF (selectable + scanned), DOCX, DOC, JPG, PNG, TIFF
- **20+ search keywords** — Catches direct applications, referrals, job portal forwards
- **Quota protection** — Wait nodes + retryOnFail (5x) on all Google Sheets operations
- **Production-grade binary handling** — Dynamic binary key resolution works across all sources

---

## 🛠️ Tech Stack

| Component | Service |
|---|---|
| Automation engine | n8n (self-hosted) |
| Email (Gmail) | Gmail OAuth2 |
| Email (Outlook) | Microsoft OAuth2 |
| File storage | Google Drive |
| Database | Google Sheets |
| AI (text PDF / DOCX) | Azure OpenAI GPT-4o |
| AI (scanned PDF) | Azure Document Intelligence |
| AI (image resume) | Azure GPT-4o Vision |

---

## 🚀 Quick Start

### Prerequisites
- n8n self-hosted (v1.0+)
- Google account with Gmail + Drive + Sheets access
- Microsoft account (for Outlook monitor)
- Azure subscription with:
  - Azure OpenAI resource (GPT-4o deployment)
  - Azure Document Intelligence resource

### 1. Clone the repo
```bash
git clone https://github.com/YOUR_USERNAME/ai-hiring-system.git
cd ai-hiring-system
```

### 2. Replace placeholders
See [SETUP.md](SETUP.md) for the full list of placeholders and how to replace them.

```bash
# Quick example
sed -i 's/YOUR_GOOGLE_SHEETS_ID/your-actual-sheets-id/g' workflows/*.json
```

### 3. Set up Google Sheets
Create a sheet with 4 tabs: **RESUME**, **NO RESUME**, **PROCESSED_FILES**, **SCAN_LOGS**
See [SETUP.md](SETUP.md) for exact column names.

### 4. Set up Google Drive
- Create folder `AI HIRE RESUME` (intake) → note the folder ID
- Create folder `AI HIRE PROCESSED` (archive) → note the folder ID

### 5. Import workflows into n8n
Import in this order — **Parser first, always keep it Active**:
1. Resume Parser
2. Gmail Monitor
3. Outlook Monitor
4. Drive Monitor
5. Historical scanners (run once manually)

### 6. Connect credentials in n8n
- Gmail OAuth2
- Microsoft Outlook OAuth2
- Google Sheets OAuth2
- Google Drive OAuth2
- Azure OpenAI API

---

## 🔄 How It Works

### Real-time flow (Gmail / Outlook / Drive)
```
New email/file arrives
  → Extract attachments
  → Filter: PDF / DOCX / DOC / Image only
  → Filter: Under 20MB
  → Generate djb2 hash (gm_ / ol_ / dr_ prefix)
  → Check PROCESSED_FILES sheet — already seen?
    YES → skip silently
    NO  → POST to Parser webhook → mark processed
```

### Parser flow
```
Webhook receives file
  → Route by MIME type
  → AI extraction (GPT-4o or Doc Intelligence)
  → Normalize candidate data
  → Check RESUME sheet by email
    Found   → UPDATED_RESUME (increment count)
    Missing → NEW_RESUME
  → Upsert row in RESUME sheet
  → Save file to AI HIRE PROCESSED Drive folder
```

---

## 🔒 Security Notes

- All API keys and credential IDs have been removed from this repo
- Never commit your `.env` or raw workflow exports with real keys
- Rotate your Azure API keys if you accidentally push them
- Use `.gitignore` to exclude any local files with real credentials

---

## 📋 Remaining / Future Work

- [ ] Reporting dashboard (daily/weekly summary)
- [ ] Edge case handling (password-protected PDFs, corrupted files)
- [ ] Advanced deduplication (phone matching, fuzzy name matching)

---

## 📄 License

MIT — free to use, modify, and distribute.

