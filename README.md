# DoDo — Plataforma de Agentes de Venta por WhatsApp con IA

DoDo es una plataforma SaaS que permite a negocios automatizar su proceso de venta por WhatsApp usando agentes de inteligencia artificial. El agente guía al cliente desde el interés inicial hasta la validación del pago, usando una base de conocimientos personalizada para cada negocio.

## Estado del proyecto

- **Cientes activos** en producción
- Postulación a **Santander Xplorer** (aceleradora)
- Postulación a **13G Perú** (programa de startups)
- En mejora continua de stack tecnológico y propuesta de valor

## Arquitectura del sistema

```
                        ┌──────────────────────────────────────────────────────┐
                        │                     VPS                              │
                        │                                                      │
  WhatsApp ──────►  SaaS  ──────►  iPaaS  ────────────►  Sales Agent API       │
  (cliente)        (fork DoDo)   (webhooks and integration)     (LangGraph)    │
                        │                               │                      │
                        │                    ┌──────────┴──────────┐           │
                        │                    │                     │           │
                        │              IBM document parser          PaaS       │
                        │            (OCR + parsing)             (orquestador) │
                        └──────────────────────────────────────────────────────┘
                                                │
                                    ┌───────────┴───────────┐
                                    │                       │
                              Cloud database            Vertex AI
                          (PostgreSQL)              
                         ┌─────┴─────┐
                         │           │
                    Vector Store   Chat Memory
                     (hybrid)    (historial por
                                       sesión)
```

## Stack tecnológico

### Infraestructura
- **Hetzner VPS** — servidor principal donde corren todos los servicios
- **Coolify** — orquestador de contenedores Docker (alternativa self-hosted a Heroku/Vercel)
- **Docker** — cada servicio corre en su propio contenedor

### Comunicación y CRM
- **Chatwoot** (fork personalizado) — gestión de conversaciones multicanal
- **WhatsApp Business API** — canal de comunicación con clientes
- **n8n** — integrador de flujos (webhooks, triggers, transformaciones)

### Agente de ventas
- **LangGraph** — orquestación del flujo agéntico multi-paso
- **FastAPI** — API REST que expone el agente como servicio
- **Vertex AI (Gemini 2.5 Flash)** — modelos de lenguaje para generación y clasificación
- **Google Cloud Service Account** — autenticación segura para Vertex AI

### RAG (Retrieval-Augmented Generation)
- **Docling** (IBM) — conversión de documentos (Word, PDF) a Markdown estructurado
- **Supabase pgvector** — almacenamiento de embeddings vectoriales (3072 dimensiones)
- **Supabase tsvector** — índice de texto completo para búsqueda BM25
- **Gemini Embedding 001** — modelo de embeddings (3072 dimensiones)
- **Hybrid Search (RRF)** — fusión de búsqueda semántica + BM25 via Reciprocal Rank Fusion
- **Supabase Edge Functions** — endpoint serverless para búsqueda híbrida

### Memoria conversacional
- **PostgreSQL (Supabase)** — almacenamiento de historial de chat por sesión
- Cada cliente mantiene su contexto conversacional entre mensajes

## Flujo de indexación de documentos

```
Google Drive (trigger: archivo nuevo/modificado)
         │
         ▼
   n8n descarga el archivo (binario)
         │
         ▼
   Docling convierte a Markdown estructurado
         │
         ▼
   Code node: split por secciones (#### headers)
         │
         ▼
   Gemini Embedding 001: genera vector 3072d por chunk
         │
         ▼
   Supabase: inserta chunk + embedding + genera tsvector automáticamente
```

Cada vez que un documento se actualiza, el flujo borra los chunks anteriores del mismo `document_id` y reindexza, garantizando que la base de conocimientos siempre esté actualizada.

## Flujo del agente de ventas (por mensaje)

```
Cliente envía mensaje por WhatsApp
         │
         ▼
   Chatwoot recibe → webhook → n8n
         │
         ▼
   n8n arma el request: pregunta + system_prompt + session_id + table_name
         │
         ▼
   POST /chat → FastAPI (Sales Agent API)
         │
         ▼
   ┌─────────────────────────────────────────┐
   │           LangGraph Workflow            │
   │                                         │
   │  1. Cargar historial (PostgreSQL)       │
   │  2. Buscar en KB (Edge Function)        │
   │     └── Embedding + Hybrid Search       │
   │         (BM25 + Semántico + RRF)        │
   │  3. Evaluar relevancia (Gemini Flash)   │
   │  4. Generar respuesta (Gemini Flash)    │
   │  5. Verificar respuesta (Gemini Flash)  │
   │     └── Si falla → reintentar (máx 2)  │
   │  6. Guardar en memoria                  │
   └─────────────────────────────────────────┘
         │
         ▼
   Respuesta → n8n → Chatwoot → WhatsApp
```

## Búsqueda híbrida (Hybrid Search)

La búsqueda de información relevante combina dos enfoques complementarios usando Reciprocal Rank Fusion (RRF):

- **Búsqueda semántica** (pgvector): encuentra chunks por significado. "quiero algo para jugar con mis hijos" encuentra consolas familiares aunque no use la palabra "consola".
- **Búsqueda BM25** (tsvector): encuentra chunks por keywords exactas. "PS5 Slim 1TB precio" encuentra el producto exacto por coincidencia de términos.
- **RRF Fusion**: combina ambos rankings en uno solo, ponderando semántico (0.7) sobre BM25 (0.3) por defecto. Los pesos son configurables por request.

La función de búsqueda es genérica — recibe `table_name` como parámetro, permitiendo reutilizarla para cualquier base de conocimientos sin duplicar código.

## Arquitectura multi-tenant

El sistema está diseñado para servir a múltiples clientes con una sola instancia:

- **Una base de conocimientos (tabla) por cliente** en Supabase
- **Una tabla de memoria por cliente** para historial de conversaciones
- **Un system prompt personalizado por cliente** con variables dinámicas (nombre del agente, empresa, moneda, tipo de producto)
- **Un solo endpoint `/chat`** que recibe `table_name` y `memory_table` como parámetros
- **Una sola Edge Function** de búsqueda híbrida genérica

## Seguridad

- API Key obligatoria para acceder al endpoint `/chat`
- Documentación de la API deshabilitada en producción (`docs_url=None`)
- Variables sensibles como variables de entorno en Coolify (nunca en código)
- Credenciales de Google Cloud (Service Account JSON) se escriben en archivo temporal en runtime, nunca persisten en disco ni en repositorio

## Decisiones técnicas relevantes

**¿Por qué LangGraph y no el agente ReAct nativo de n8n?**
El agente ReAct de n8n siempre llama la tool de búsqueda — incluso para un "hola" — porque el LLM decide en cada turno si usar la herramienta. Con LangGraph, el flujo es determinístico: siempre busca contexto, pero un agente de relevancia (barato, Gemini Flash Lite) decide si ese contexto aplica antes de generar la respuesta. Además, LangGraph permite agregar verificación de alucinaciones como paso separado.

**¿Por qué Docling y no PyMuPDF o Unstructured?**
Docling (IBM) preserva la estructura del documento (headers, tablas, listas) en Markdown, lo que permite hacer chunking inteligente por secciones en vez de cortar por caracteres arbitrarios. Un chunk "Política de Precios" contiene toda la tabla de precios completa, no media tabla.

**¿Por qué Hybrid Search y no solo semántico?**
Las consultas de clientes son mixtas: "PS5 precio" es keyword-driven (BM25 gana), pero "quiero algo para jugar con mis hijos" es semántica. RRF combina ambos sin que uno domine al otro.

**¿Por qué Vertex AI y no la API pública de Gemini?**
Google Cloud ofrece créditos gratuitos ($300 USD) que solo aplican via Vertex AI, no via AI Studio. Vertex AI también ofrece rate limits más altos y SLAs de producción.

## Contacto

Jorge Soto — [GitHub](https://github.com/jorgesd97)
