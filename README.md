# Forte.jus

> On-premise AI legal assistant for Brazilian law firms — RAG over court case files with mandatory citation, integrated legislation database, and direct PJe court system connection. Zero data leaves the office.

![Forte.jus — case analysis with page citations and legal reasoning](docs/screenshots/forte.jus_screenshot-1.jpg)

[![Python](https://img.shields.io/badge/Python-3.12-blue?logo=python)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-RAG_API-green?logo=fastapi)](https://fastapi.tiangolo.com/)
[![Qdrant](https://img.shields.io/badge/Qdrant-Vector_DB-red)](https://qdrant.tech/)
[![Ollama](https://img.shields.io/badge/Ollama-Local_LLM-black)](https://ollama.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**Showcase repository** — full source available on request for technical evaluation.

---

## The Problem

Lawyers in Macapá, Brazil start every morning by opening PJe (the national court system) and finding dozens of new case notifications. Each one requires opening the case file, reading, understanding the deadline, and logging the action. A 200-page case file can consume hours of initial analysis.

Cloud-based AI assistants are not an option: Brazilian LGPD (data protection law) and professional secrecy rules make it impossible to send client case files to external APIs.

Forte.jus runs entirely inside the law office — no data leaves the machine, no per-token costs, no internet dependency.

---

## What It Does

- **Direct PJe integration** — connects to the court system via MNI 2.2.2 (SOAP) protocol, automatically downloads new case notifications and PDFs without any cloud intermediary
- **"Ask the case file"** — answers questions in Portuguese with mandatory page citations (`Forte.jus p. XX`)
- **Embedded legislation database** — 4,073 indexed points covering the Brazilian Penal Code, Civil Code, Criminal Procedure Code, Civil Procedure Code, Constitution, and 471 Supreme Court precedents (STJ)
- **Exact article lookup** — when a case mentions "art. 155 CP", the system extracts the article reference and retrieves the actual legal text from the database — the LLM receives the real law, not its training memory
- **Draft generation** — writes legal briefs, responses, and motions based on case facts and applicable law; always marks sections requiring lawyer review with `[COMPLETAR]`
- **Automated notifications** — new court notifications arrive as Telegram/WhatsApp messages with AI-generated summaries and deadline calculations
- **LGPD compliant by design** — on-premise, air-gap capable, no telemetry

---

## Tech Stack

| Layer | Technology |
|---|---|
| LLM inference | Ollama — qwen3.5:9b / deepseek-r1:8b |
| Embeddings | nomic-embed-text (768d) |
| Vector DB | Qdrant — collections `processos` + `legislacao` |
| Backend | FastAPI (OpenAI-compatible API) |
| Chat interface | Open WebUI |
| Workflow automation | n8n |
| PDF extraction | PyMuPDF (fitz) |
| PJe integration | Zeep (SOAP/MNI 2.2.2) |
| Infrastructure | Docker Compose |
| GPU | NVIDIA RTX 3060 12GB / CUDA |

**Minimum hardware:** NVIDIA GPU with 8GB VRAM, 16GB RAM.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client (LAN)                         │
│              Browser → Open WebUI :3000                 │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTP (OpenAI-compatible)
┌───────────────────────▼─────────────────────────────────┐
│              FastAPI — fortejus-api :8001                │
│                                                         │
│  POST /v1/chat/completions  →  RAG pipeline             │
│  POST /ingest               →  PDF ingestion            │
└──────────┬────────────────────────┬─────────────────────┘
           │                        │
┌──────────▼──────────┐  ┌──────────▼──────────┐
│   Qdrant :6333      │  │   Ollama :11434      │
│                     │  │                     │
│  ┌──────────────┐   │  │  nomic-embed-text   │
│  │  processos   │   │  │  qwen3.5:9b         │
│  └──────────────┘   │  └─────────────────────┘
│  ┌──────────────┐   │
│  │  legislacao  │   │
│  │  4073 points │   │
│  └──────────────┘   │
└─────────────────────┘
           │
┌──────────▼──────────┐
│   n8n :5678         │
│  Cron → MNI → RAG  │
│  → Telegram/WA      │
└─────────────────────┘
```

---

## RAG Pipeline — Parent-Child Chunking

```
PDF (N pages)
    │
    ▼ PyMuPDF → text per page
    │
    ▼ Parent chunks (~1500 tokens, no overlap)
    │   └── Child chunks (~300 tokens, 20% overlap)
    │
    ▼ nomic-embed-text → 768d embeddings (batches of 32)
    │
    ▼ Qdrant upsert
         payload: { case_number, file,
                    child_text, child_start_page, child_end_page,
                    parent_text, parent_start_page, parent_end_page }
```

**Child chunks** provide precision in vector search. **Parent chunks** provide expanded context to the LLM — avoiding truncated answers without multiplying the number of indexed vectors.

### Retrieval — Dual-Collection + Exact Article Lookup

```
question
    │
    ├─► Qdrant `processos`    (cosine, top_k=5, filtered by case number)
    │        → deduplicate by parent_index → excerpts with page numbers
    │
    └─► Qdrant `legislacao`   (cosine, top_k=8)
             ├── semantic search (filtered by statute code if detected)
             └── exact lookup: "art. 155 CP" → scroll() + payload filter
                  → score=1.0, actual legal text always in context
    │
    ▼ Prompt with two sections:
         === CASE EXCERPTS === (parent_texts with page numbers)
         === APPLICABLE LAW === (articles with statute code and number)
    │
    ▼ Ollama stream → SSE → Open WebUI
```

The **exact article lookup** is the primary anti-hallucination guardrail: when a case cites "art. 155 CP" (theft), the retriever extracts the `(CP, 155)` pair via regex and fetches the exact article from Qdrant — the real statutory text enters the prompt, eliminating hallucinated penalties, deadlines, and statutory language.

---

## Legislation Database

| Code | Statute | Indexed points |
|---|---|---|
| CP | Brazilian Penal Code (Decree-Law 2.848/1940) | 346 |
| CPP | Criminal Procedure Code (Decree-Law 3.689/1941) | ~810 |
| CC | Civil Code (Law 10.406/2002) | ~2,046 |
| CPC | Civil Procedure Code (Law 13.105/2015) | ~1,072 |
| LEP | Prison Enforcement Law (Law 7.210/1984) | ~204 |
| CF | Federal Constitution of 1988 | ~250 |
| STJ | Supreme Court precedents (súmulas) | 471 |
| **Total** | | **4,073 points** |

Sources: HTML from the Brazilian government's Planalto portal + official STJ rulings. Custom parser built with BeautifulSoup4.

---

## PJe / MNI Integration — The Differentiator

Direct connection to the Brazilian national court system (PJe) via MNI 2.2.2 protocol (SOAP), no intermediaries:

```
n8n (cron, every 30 min)
    ↓
MNI: consultarAvisosPendentes(CPF, password)
    ↓ list of new notifications
For each notification:
    ├── consultarTeorComunicacao  → PDF download
    ├── POST /ingest              → chunks → embeddings → Qdrant
    ├── consultarProcesso         → case metadata (parties, court, type)
    ├── confirmarRecebimento      → marks notification as read
    └── Telegram/WhatsApp alert:
        "⚖️ New notification — Case XXXXXXX
         Deadline: 5 business days (due MM/DD/YYYY)
         Summary: [LLM-generated with page citation]"
```

MNI endpoints confirmed on TJAP (Amapá State Court):
- 1st instance: `pje.tjap.jus.br/1g/intercomunicacao?wsdl`
- 2nd instance: `pje.tjap.jus.br/2g/intercomunicacao?wsdl`

**Status:** SOAP client implemented and protocol documented. Awaiting PJe-registered lawyer credentials for production validation.

---

![Forte.jus — applicable legislation, case analysis and procedural recommendations](docs/screenshots/forte.jus_screenshot-2.jpg)

---

## Anti-Hallucination Guardrails

| Guardrail | Implementation |
|---|---|
| Case grounding | System prompt prohibits claims outside the provided case excerpts |
| Law grounding | Articles retrieved via RAG; LLM explicitly forbidden from citing law from training memory |
| Exact article lookup | Real statutory text always in context (Qdrant score=1.0) |
| Mandatory citation | Every answer must cite source pages: `(Forte.jus p. XX)` |
| Explicit uncertainty | If information is not in context, model must say so explicitly |
| Human-in-the-loop | All drafted documents marked `[COMPLETAR]` wherever lawyer review is required |

---

## Quick Start

```bash
git clone https://github.com/JConradoN/forte.jus-showcase.git
cd forte.jus-showcase

# Full source and infra files are available on request
# Contact for access to the private repository
```

---

## Roadmap

| Phase | Feature | Status |
|---|---|---|
| 1 | Docker infra + RAG chat | ✅ Done |
| 2 | Legislation database (4,073 points) | ✅ Done |
| 3 | MNI/TJAP automated integration | 🔄 Awaiting credentials |
| 4 | Telegram/WhatsApp notification agent | 📋 Planned |
| 5 | Deadline tracking dashboard (n8n) | 📋 Planned |
| 6 | Legal brief generation | 📋 Planned |
| 7 | OCR for scanned PDFs | 📋 Planned |

---

## Documentation

- [User Guide](docs/guia_usuario.md) — for lawyers and office staff (Portuguese)
- [Technical Document](docs/documento_tecnico.md) — architecture, pipeline, API reference (Portuguese)

---

## About

Built by a network and infrastructure engineer with 30 years of IT experience, applying modern RAG architecture to a domain with strict privacy requirements and zero tolerance for AI hallucination.

The legal domain is one of the hardest contexts for LLMs — factual precision matters, citations must be verifiable, and hallucinated legal text has real consequences. The architecture decisions in Forte.jus (parent-child chunking, dual-collection retrieval, exact article lookup, mandatory grounding) were made specifically to address these constraints.

**Available for freelance projects** — [github.com/JConradoN](https://github.com/JConradoN)

---

*Showcase repository — full source available upon request for technical evaluation.*  
*Not affiliated with CNJ, TJAP, or any judicial authority.*
