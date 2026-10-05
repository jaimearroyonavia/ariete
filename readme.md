# Ariete: Agente Autónomo para Diagnóstico y Remediación de Infraestructura

**Versión:** 1.0 (MVP)  
**Estado:** Especificación + Arquitectura Completadas  
**Último Actualizado:** 2026-10-05

---

## ¿Qué es Ariete?

Ariete es un agente autónomo que resuelve problemas de infraestructura sin esperar a un operador humano. Cuando un servicio cae:

1. **Detecta** la caída en <60 segundos (monitoreo continuo)
2. **Diagnostica** el root cause (análisis automático de logs)
3. **Propone** remediación (usando LLM + knowledge base)
4. **Ejecuta** con aprobación operador (o autónomo en horario permitido)

**Problema que resuelve:**  
Equipos sin soporte 24/7 pierden horas en resolución de problemas triviales. Ariete reduce MTTR de 60+ minutos a <5 minutos.

---

## Arquitectura en 30 segundos

```
┌──────────────────────┐
│   Web UI (React)     │
│  - Dashboard         │
│  - Botón "Aprobar"   │
└──────────┬───────────┘
           │ HTTPS
┌──────────▼───────────┐
│  Rails API Server    │
└──────────┬───────────┘
    ┌──────┴────┬────┐
    ▼           ▼    ▼
PostgreSQL  Sidekiq Redis
```

**Stack:** Rails 8 + PostgreSQL + Redis + Sidekiq + OpenAI API

---

## Requisitos Previos

```bash
Ruby 3.3+
PostgreSQL 16+
Redis 7+
Node.js 18+
```

---

## Épics & Roadmap

**v1 - MVP (6 semanas):**
- E0: Admin & Infrastructure Setup
- E1: URL Monitoring & Detection
- E2: Diagnostics & Analysis
- E3: Approval & Execution (+ autonomous variant)

**v2 - Enhancements:**
- E4: RAG Knowledge Base
- E5: Audit Dashboard
- E6: Advanced Security

Ver [_bmad-output/planning/epics.md](./_bmad-output/planning/epics.md) para historias detalladas.

---

## Documentación

- **[SPEC](./docs/SPEC.md)** — Capabilities + constraints
- **[ARCHITECTURE](./docs/ARCHITECTURE-SPINE.md)** — Decisiones arquitectónicas
- **[DATA-MODEL](./docs/DATA-MODEL.md)** — Schema
- **[EPICS](./docs/EPICS.md)** — User stories

---

## Seguridad (CRÍTICA)

✅ Credenciales encriptadas  
✅ Toda acción auditable  
✅ Aprobación humana requerida  
✅ Rate limiting  
✅ No plaintext secrets en logs  

---

## Contacto

- **Issues:** GitHub Issues
- **Docs:** `/docs`
- **Slack:** #ariete-dev

---

Última actualización: 2026-10-05 | Ariete v1.0
