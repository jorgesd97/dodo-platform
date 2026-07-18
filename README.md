# DoDo — Plataforma de Agentes de Venta por WhatsApp con IA

DoDo es una plataforma SaaS que permite a negocios automatizar su proceso de venta por WhatsApp usando agentes de inteligencia artificial. El agente guía al cliente desde el interés inicial hasta la validación del pago, usando una base de conocimientos personalizada para cada negocio.

## Estado del proyecto

- **Clientes activos** en producción
- Postulación a aceleradoras
- Postulación a programa de startups
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
- **Vertex AI** como motor de inferencia para generación, clasificación y verificación
- Autenticación segura via Service Account de Google Cloud

### RAG (Retrieval-Augmented Generation)
- Parser de documentos IBM para conversión de Word/PDF a formato estructurado corriendo en VPS.
- Base de datos vectorial
- **Hybrid Search con Reciprocal Rank Fusion (RRF)** combinando semántico + BM25
- Edge Function serverless como endpoint unificado de búsqueda

### Memoria conversacional
- PostgreSQL como almacenamiento de historial de chat por sesión

## Arquitectura multi-tenant

El sistema está diseñado para servir a múltiples clientes con una sola instancia:

- **Una base de conocimientos (tabla) por cliente** en la base de datos
- **Una tabla de memoria por cliente** para historial de conversaciones
- **Un system prompt personalizado por cliente** con variables dinámicas
- **Un solo endpoint /chat**
- **Una sola Edge Function**

## Contacto

Jorge Soto — [GitHub](https://github.com/jorgesd97)
