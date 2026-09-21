# 🏛️ Architecture Decision Records (ADR) — 4GUARD WMS Backend (`4guard_be`)

> Registros inmutables de decisiones arquitectónicas y técnicas para el Backend Java 21 / Spring Boot 3 de 4GUARD WMS.

---

## 📋 Catálogo de Decisiones Arquitectónicas del Backend

| ID | Título | Estado | Fecha | Ámbito | Archivo |
|---|---|---|---|---|---|
| **ADR-007** | Directiva Estricta Cero Mocks y Persistencia Real en PostgreSQL | ✅ Aceptado | 2026-07 | Persistencia & Integridad | [`ADR-007-zero-mock-database-rule.md`](./ADR-007-zero-mock-database-rule.md) |
| **ADR-008** | Estándar de Contratos API (`ApiResponse<T>`, UUID, ISO-8601 UTC) | ✅ Aceptado | 2026-07 | Contratos REST | [`ADR-008-api-contracts-uuid-utc.md`](./ADR-008-api-contracts-uuid-utc.md) |
| **ADR-009** | Mecanismo de Autenticación JWT Stateless y Refresh Token | ✅ Aceptado | 2026-07 | Seguridad & Auth | [`ADR-009-http-auth-refresh-token.md`](./ADR-009-http-auth-refresh-token.md) |
| **ADR-010** | Adopción de la Metodología Spec-Driven Development (SDD) | ✅ Aceptado | 2026-07 | Metodología & Specs | [`ADR-010-spec-driven-development.md`](./ADR-010-spec-driven-development.md) |
| **ADR-012** | Protocolo Zone Lease y Sockets TCP Raw (Puerto 9100) para Terminal RF y Zebra ZPL | ✅ Aceptado | 2026-09 | Hardware / Sockets / RF | [`ADR-012-rf-terminal-pwa-offline-first.md`](./ADR-012-rf-terminal-pwa-offline-first.md) |
| **ADR-014** | Arquitectura Hexagonal (Ports & Adapters), Multi-tenancy Organizacional (`organizationId`) y Motor de Auditoría con Deltas | ✅ Aceptado | 2026-09 | Arquitectura Hexagonal / DDD | [`ADR-014-arquitectura-backend-hexagonal-multitenant-audit.md`](./ADR-014-arquitectura-backend-hexagonal-multitenant-audit.md) |
| **ADR-015** | Emisión de Códigos Industriales e Integración con Impresoras Térmicas ZPL | ✅ Aceptado | 2026-09 | Integración Hardware / ZPL | [`ADR-015-motor-documental-pdf-zpl-export.md`](./ADR-015-motor-documental-pdf-zpl-export.md) |
