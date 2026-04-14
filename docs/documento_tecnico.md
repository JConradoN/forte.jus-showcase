# Forte.jus — Documento Técnico

**Para profissionais de TI e desenvolvedores**

| Campo | Valor |
|---|---|
| Versão | 0.3.0 |
| Data | Março 2026 |
| Status | MVP em produção — escritório piloto |
| Repositório | privado (acesso sob solicitação) |

---

## Visão Geral

O Forte.jus é um sistema RAG (Retrieval-Augmented Generation) on-premise para análise de processos jurídicos. Roda 100% localmente, sem dependência de APIs externas. Foi projetado para escritórios de advocacia que precisam de soberania de dados (LGPD) e funcionamento offline.

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│                     Cliente (LAN)                       │
│              Browser → Open WebUI :3000                 │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTP (OpenAI-compatible API)
┌───────────────────────▼─────────────────────────────────┐
│              FastAPI — fortejus-api :8000                │
│                                                         │
│  POST /v1/chat/completions  →  RAG pipeline             │
│  POST /ingest               →  PDF ingestion            │
│  POST /chat/trechos         →  debug retrieval          │
└──────────┬────────────────────────┬─────────────────────┘
           │                        │
┌──────────▼──────────┐  ┌──────────▼──────────┐
│   Qdrant :6333      │  │   Ollama :11434      │
│   Vector DB         │  │                     │
│   ┌──────────────┐  │  │  nomic-embed-text   │
│   │  processos   │  │  │  deepseek-r1:8b     │
│   │  PDFs ingest │  │  │  (qwen3.5:9b opção) │
│   └──────────────┘  │  └─────────────────────┘
│   ┌──────────────┐  │
│   │  legislacao  │  │
│   │  CP/CPP/CC   │  │
│   │  CPC/LEP/CF  │  │
│   │  Súmulas STJ │  │
│   └──────────────┘  │
└─────────────────────┘
```

---

## Stack

| Camada | Tecnologia | Versão |
|---|---|---|
| LLM runtime | Ollama | latest |
| Modelo de chat | deepseek-r1:8b / qwen3.5:9b | — |
| Modelo de embedding | nomic-embed-text | latest |
| Vector DB | Qdrant | latest |
| Backend | FastAPI + Uvicorn | 0.115 / 0.30 |
| Interface | Open WebUI | 0.8.12 |
| Orquestração | n8n | latest |
| Extração PDF | PyMuPDF (fitz) | 1.24.10 |
| Cliente SOAP | Zeep | 4.2.1 |
| Containers | Docker + Compose | 28.2 / v5.1 |
| GPU | NVIDIA RTX 3060 12GB | CUDA 12.8 |
| OS | Ubuntu 24.04 LTS | kernel 6.17 |

---

## Pipeline de Ingestão (RAG)

### Parent-Child Chunking

```
PDF (N páginas)
    │
    ▼ PyMuPDF → texto por página
    │
    ▼ Parent chunks (~1500 tokens, sem overlap)
    │       └── Child chunks (~300 tokens, 20% overlap)
    │
    ▼ nomic-embed-text → embeddings 768d (batches de 32)
    │
    ▼ Qdrant upsert
         payload: {
           processo, arquivo,
           child_text, child_start_page, child_end_page,
           parent_text, parent_start_page, parent_end_page,
           parent_index
         }
```

**Justificativa:** child chunks garantem precisão na busca vetorial; parent chunks fornecem contexto expandido ao LLM — evita respostas truncadas sem aumentar o número de vetores indexados.

### Busca (query time) — Dual-Collection

```
pergunta (string)
    │
    ▼ nomic-embed-text → embedding 768d
    │
    ├─► Qdrant search em `processos` (cosine, top_k=5)
    │        filter: { processo: "XXXXXXX-XX.XXXX.X.XX.XXXX" }
    │        Deduplica por parent_index → list[Trecho]
    │
    └─► Qdrant search em `legislacao` (cosine, top_k=8)
             ├── Busca semântica (filtro por sigla se detectada na pergunta)
             └── Lookup exato por artigo (scroll, sigla+numero, score=1.0)
                  → artigos mencionados nos trechos do processo
                  → list[Artigo]
    │
    ▼ Monta prompt com duas seções:
         === TRECHOS DO PROCESSO ===   (parent_texts com páginas)
         === LEGISLAÇÃO RELEVANTE ===  (artigos com sigla e número)
    │
    ▼ Ollama /api/chat (stream, num_ctx=8192)
    │
    ▼ SSE → Open WebUI (formato OpenAI Chat Completions)
```

**Lookup exato de artigos:** quando o texto do processo cita `art. 155 do CP`, o retriever extrai `(CP, 155)` via regex e busca o artigo exato no Qdrant com `scroll()` + filtro de payload. O resultado recebe score=1.0 e é sempre incluído no contexto do LLM, evitando que o modelo invente a redação da lei.

---

## Base de Legislação

### Conteúdo indexado (coleção `legislacao`)

| Sigla | Lei | Artigos/Itens |
|---|---|---|
| CP | Código Penal (Dec.-Lei 2.848/1940) | 346 |
| CPP | Código de Processo Penal (Dec.-Lei 3.689/1941) | ~810 |
| CC | Código Civil (Lei 10.406/2002) | ~2046 |
| CPC | Código de Processo Civil (Lei 13.105/2015) | ~1072 |
| LEP | Lei de Execução Penal (Lei 7.210/1984) | ~204 |
| CF | Constituição Federal de 1988 | ~250 |
| STJ | Súmulas do Superior Tribunal de Justiça | 471 |

**Total: 4073 pontos**. Arquivos-fonte: HTML do Planalto (leis) + VerbetesSTJ_asc.txt (súmulas).

### Pipeline de indexação de legislação

```
HTML Planalto / TXT súmulas
    │
    ▼ services/legislacao/parser.py (BeautifulSoup4)
         ART_RE: r"(?:^|\s)Art\." com .search() — lida com &nbsp; do Planalto
    │
    ▼ Truncamento a 2000 chars (artigos longos não excedem limite do embed)
    │
    ▼ nomic-embed-text → embedding 768d
    │
    ▼ Qdrant upsert (offset = points_count atual para não sobrescrever)
         payload: { lei, sigla, artigo, numero, texto }
```

### Comandos de manutenção

```bash
# Indexar todas as leis (arquivos em /mnt/vault/legislacao/)
docker exec fortejus-api python -m services.legislacao.batch --vault /mnt/legislacao

# Apenas súmulas
docker exec fortejus-api python -m services.legislacao.batch --vault /mnt/legislacao --apenas-sumulas
```

---

## API Endpoints

### `POST /ingest`
```
multipart/form-data
  processo: string (CNJ format)
  file: PDF

Response 200:
  { processo, arquivo, paginas, parents, children }
```

### `POST /v1/chat/completions`
OpenAI-compatible. Detecta número CNJ na mensagem via regex.
```
Body: { model: "forte.jus", messages: [...], stream: true }
Response: SSE stream (text/event-stream)
```

### `POST /chat/trechos`
```
{ processo, pergunta, top_k }
Response: [ { score, paginas, trecho } ]
```

---

## Integração MNI/PJe — O diferencial do sistema

Esta é a funcionalidade central que distingue o Forte.jus de qualquer outro assistente jurídico com IA: **conexão direta com o PJe do TJAP**, sem intermediários em nuvem.

### O que a integração entrega

```
┌─────────────────────────────────────────────────────────┐
│                   PJe — TJAP                            │
│   pje.tjap.jus.br/1g/intercomunicacao?wsdl              │
│   pje.tjap.jus.br/2g/intercomunicacao?wsdl              │
└───────────────────────┬─────────────────────────────────┘
                        │ SOAP MNI 2.2.2 (HTTPS)
                        │ autenticação: CPF + senha no body
┌───────────────────────▼─────────────────────────────────┐
│            services/mni/client.py (Zeep)                │
│                                                         │
│  consultarAvisosPendentes  → lista intimações novas     │
│  consultarTeorComunicacao  → baixa PDF da intimação     │
│  consultarProcesso         → metadados + movimentações  │
│  confirmarRecebimento      → marca intimação como lida  │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                   n8n (workflow)                        │
│                                                         │
│  Cron: verifica intimações a cada N minutos             │
│  Novidade encontrada →                                  │
│    1. Baixa PDF do processo                             │
│    2. Dispara ingestão RAG (POST /ingest)               │
│    3. Envia alerta (Telegram / WhatsApp / e-mail)       │
└─────────────────────────────────────────────────────────┘
```

### Fluxo completo de uma intimação

```
1. n8n (cron, ex: a cada 30 min)
      ↓
2. MNI: consultarAvisosPendentes(CPF, senha)
      ↓ lista de avisos novos
3. Para cada aviso:
   a. consultarTeorComunicacao → PDF do documento
   b. POST /ingest (número processo + PDF)
      → PyMuPDF → chunks → embeddings → Qdrant
   c. consultarProcesso → metadados (partes, tipo, vara)
   d. confirmarRecebimento → aviso marcado como lido
   e. Notificação → Telegram/WhatsApp com resumo gerado por RAG
      ↓
4. Advogado recebe no celular:
   "⚖️ Nova intimação — Processo XXXXXXX
    Prazo: 5 dias úteis (vence DD/MM/AAAA)
    Resumo: [gerado pelo LLM com citação de página]"
```

### Endpoints confirmados (TJAP)

```
1º Grau: https://pje.tjap.jus.br/1g/intercomunicacao?wsdl
2º Grau: https://pje.tjap.jus.br/2g/intercomunicacao?wsdl
```

Ambos respondem e expõem o `IntercomunicacaoService` conforme o protocolo MNI 2.2.2 do CNJ.

### Autenticação

Autenticação via CPF + senha diretamente no corpo SOAP — sem OAuth, sem token separado, sem certificado digital para consulta. O mesmo login do advogado no PJe web.

### Status

Protocolo documentado, endpoints verificados, cliente SOAP base implementado (`services/mni/client_example.py`). **Aguardando credenciais reais de advogado cadastrado no PJe/TJAP** para validar e ativar o fluxo completo.

---

## Guardrails implementados

1. **Grounding em processo:** system prompt proíbe afirmações sobre fatos fora dos trechos fornecidos
2. **Grounding em legislação:** artigos de lei são fornecidos via RAG — o LLM é proibido de citar dispositivos legais da memória de treinamento
3. **Lookup exato de artigos:** quando o processo cita `art. 155 do CP`, o retriever busca o artigo exato no Qdrant via `scroll()` com filtro de payload (sigla + número), score=1.0 — o texto real da lei entra no prompt, eliminando alucinação de penas, prazos e redação
4. **Citação obrigatória:** toda resposta cita página(s) de origem `(Forte.jus p. XX)`
5. **Admissão de ignorância:** se a informação não está no contexto, o modelo declara explicitamente
6. **Human-in-the-loop:** seção `[PONTOS DE REFLEXÃO]` em toda resposta; minutas sempre marcadas com `[COMPLETAR]` nos campos que exigem revisão do advogado

---

## Estrutura de diretórios

```
forte.jus/
├── infra/
│   ├── docker-compose.yml     # Qdrant + n8n + API
│   ├── Dockerfile.api
│   └── .env.example
├── services/
│   ├── api/
│   │   ├── main.py            # FastAPI app
│   │   └── openai_compat.py   # /v1/chat/completions
│   ├── ingest/
│   │   ├── extractor.py       # PyMuPDF → PageBlock
│   │   ├── chunker.py         # Parent-Child chunking
│   │   ├── embedder.py        # nomic-embed-text via Ollama
│   │   ├── indexer.py         # Qdrant upsert/delete
│   │   ├── pipeline.py        # orquestrador
│   │   └── batch.py           # ingestão em lote
│   ├── rag/
│   │   ├── retriever.py       # busca dual-collection + lookup exato
│   │   └── generator.py       # prompt + Ollama stream
│   ├── legislacao/
│   │   ├── parser.py          # parser HTML Planalto (BeautifulSoup4)
│   │   ├── parser_sumulas.py  # parser súmulas STJ/STF
│   │   ├── indexer.py         # Qdrant upsert coleção legislacao
│   │   └── batch.py           # indexação em lote de leis e súmulas
│   └── mni/
│       └── client_example.py  # SOAP MNI (WIP)
├── docs/
│   ├── mni/                   # documentação protocolo MNI
│   ├── pitch_parceiro.md
│   ├── guia_usuario.md
│   └── documento_tecnico.md
└── data/fortaleza/            # volumes persistentes (gitignored)
```

---

## Como rodar localmente

**Requisitos:** Docker 28+, NVIDIA Container Toolkit, GPU NVIDIA com 8GB+ VRAM

```bash
git clone <repo>
cd forte.jus/infra
cp .env.example .env
# edite .env com as chaves

docker compose up -d

# baixar modelos
docker exec ollama ollama pull deepseek-r1:8b
docker exec ollama ollama pull nomic-embed-text

# ingestão de teste
curl -X POST http://localhost:8001/ingest \
  -F "processo=0034567-89.2023.8.03.0001" \
  -F "file=@processo.pdf"

# chat
curl -X POST http://localhost:8001/chat \
  -H "Content-Type: application/json" \
  -d '{"processo":"0034567-89.2023.8.03.0001","pergunta":"Quais são os fatos?"}'

# indexar base de legislação (CP, CPP, CC, CPC, LEP, CF + súmulas STJ)
# arquivos HTML do Planalto devem estar em /mnt/vault/legislacao/
docker exec fortejus-api python -m services.legislacao.batch --vault /mnt/legislacao
```

---

## Onde há espaço para contribuição

| Área | Status | Complexidade |
|---|---|---|
| Integração MNI/SOAP com TJAP | WIP — falta credenciais reais | Média |
| Agente Telegram/WhatsApp (OpenClaw) | Planejado | Média |
| Dashboard de prazos (n8n) | Planejado | Baixa |
| Geração de minutas de petições | Planejado | Alta |
| OCR para PDFs escaneados | Não iniciado | Média |
| Detecção automática de seções jurídicas | Não iniciado | Alta |
| Testes automatizados | Não iniciado | Baixa |
| Frontend próprio (substituir Open WebUI) | Futuro | Alta |

---

## Contato

Se tiver interesse em contribuir ou integrar ao seu produto, entre em contato:

**[seu nome / e-mail / LinkedIn]**

---

*Forte.jus é um projeto independente, desenvolvido em Macapá/AP. Não é afiliado ao CNJ, TJAP ou qualquer entidade judicial.*
