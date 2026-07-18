# DoDo — Plataforma de Agentes de Venta por WhatsApp con IA

DoDo es una plataforma SaaS que permite a negocios automatizar su proceso de venta por WhatsApp usando agentes de inteligencia artificial. El agente guía al cliente desde el interés inicial hasta la validación del pago, usando una base de conocimientos personalizada para cada negocio.

## Estado del proyecto

- **Clientes activos** en producción
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

## Capas del sistema

### Infraestructura
- VPS dedicado con servicios containerizados en Docker
- Plataforma PaaS self-hosted para orquestación y despliegue automático desde GitHub
- Despliegue continuo: push a main → build automático

### Comunicación
- CRM open-source (fork personalizado) como hub de conversaciones
- WhatsApp Business API como canal principal
- Plataforma iPaaS para integración de flujos, webhooks y transformaciones de datos

### Agente de ventas
- **LangGraph** como orquestador del flujo agéntico multi-paso
- **FastAPI** como API REST que expone el agente como microservicio
- **Vertex AI** como motor de inferencia para generación, clasificación y verificación
- Autenticación segura via Service Account de Google Cloud

### RAG (Retrieval-Augmented Generation)
- Parser de documentos IBM para conversión de Word/PDF a formato estructurado corriendo en VPS.
- Base de datos vectorial con pgvector (embeddings de n dimensiones)
- Índice de texto completo con tsvector para búsqueda BM25
- **Hybrid Search con Reciprocal Rank Fusion (RRF)** combinando semántico + BM25
- Edge Function serverless como endpoint unificado de búsqueda

### Memoria conversacional
- PostgreSQL como almacenamiento de historial de chat por sesión

## Flujo de indexación de documentos

```
Cloud storage (trigger: archivo nuevo/modificado)
         │
         ▼
   iPaaS descarga el archivo (binario)
         │
         ▼
   Document parser convierte a Markdown estructurado
         │
         ▼
   Split por secciones semánticas (#### headers)
         │
         ▼
   Embedding model genera vector nd por chunk
         │
         ▼
   Database inserta chunk + embedding + genera índice FTS automáticamente
```


## Flujo del agente de ventas (por mensaje)

```
Cliente envía mensaje por WhatsApp
         │
         ▼
   CRM recibe → webhook → iPaaS
         │
         ▼
   iPaaS arma el request: pregunta + system_prompt + session_id + table_name
         │
         ▼
   POST /chat → Sales Agent API
         │
         ▼
   ┌─────────────────────────────────────────┐
   │           LangGraph Workflow            │
   │                                         │
   │  1. Cargar historial (PostgreSQL)       │
   │  2. Buscar en KB (Edge Function)        │
   │     └── Embedding + Hybrid Search       │
   │         (BM25 + Semántico + RRF)        │
   │  3. Evaluar relevancia (LLM rápido)     │
   │  4. Generar respuesta (LLM principal)   │
   │  5. Verificar respuesta (LLM rápido)    │
   │     └── Si falla → reintentar (máx 2)  │
   │  6. Guardar en memoria                  │
   └─────────────────────────────────────────┘
         │
         ▼
   Respuesta → iPaaS → CRM → WhatsApp
```

## Búsqueda híbrida (Hybrid Search)

La búsqueda de información relevante combina dos enfoques complementarios usando Reciprocal Rank Fusion (RRF):

- **Búsqueda semántica** (pgvector): encuentra chunks por significado.
- **Búsqueda BM25** (tsvector): encuentra chunks por keywords exactas.
- **RRF Fusion**: combina ambos rankings en uno solo, ponderando semántico (0.7) sobre BM25 (0.3) por defecto. Los pesos son configurables por request.

La función de búsqueda es genérica — recibe table_name como parámetro, permitiendo reutilizarla para cualquier base de conocimientos sin duplicar código.

## Arquitectura multi-tenant

El sistema está diseñado para servir a múltiples clientes con una sola instancia:

- **Una base de conocimientos (tabla) por cliente** en la base de datos
- **Una tabla de memoria por cliente** para historial de conversaciones
- **Un system prompt personalizado por cliente** con variables dinámicas (nombre del agente, empresa, moneda, tipo de producto)
- **Un solo endpoint /chat** que recibe table_name y memory_table como parámetros
- **Una sola Edge Function** de búsqueda híbrida genérica

## Decisiones técnicas relevantes

**¿Por qué un flujo agéntico multi-paso y no un agente ReAct simple?**
Un agente ReAct decide en cada turno si usar la herramienta de búsqueda — incluso para un "hola" — porque el LLM tiene autonomía total. Con un flujo multi-paso determinístico, el sistema siempre busca contexto relevante, pero un agente de clasificación rápido decide si ese contexto aplica antes de generar la respuesta. Esto reduce llamadas innecesarias al LLM y permite agregar verificación de alucinaciones como paso separado.

**¿Por qué un parser de documentos dedicado y no extracción directa de texto?**
El parser preserva la estructura del documento (headers, tablas, listas) en Markdown, lo que permite hacer chunking inteligente por secciones en vez de cortar por caracteres arbitrarios. Un chunk "Política de Precios" contiene toda la tabla de precios completa, no media tabla.

**¿Por qué Hybrid Search y no solo búsqueda semántica?**
Las consultas de clientes son mixtas: "PS5 precio" es keyword-driven (BM25 gana), pero "quiero algo para jugar con mis hijos" es semántica. RRF combina ambos enfoques sin que uno domine al otro.

**¿Por qué Vertex AI?**
Ofrece rate limits de producción, SLAs empresariales, y compatibilidad con créditos de Google Cloud — ventajas que la API pública gratuita no garantiza.

## Contacto

Jorge Soto — [GitHub](https://github.com/jorgesd97)
