---
name: sdop-framework
description: Guía de gobernanza técnica para la Metodología SDOP (Spec-Driven Oracle-Bridge Framework) de SyborX. Úsalo para planificar, validar y ejecutar cualquier desarrollo, refactorización o migración de software asegurando cero discrepancias y cero alucinaciones.
---

# Metodología SDOP (Spec-Driven Oracle-Bridge Framework) — SyborX

El framework **SDOP** es el estándar de ingeniería y gobernanza para el desarrollo acelerado y asistido por IA en SyborX. Se basa en tres pilares inquebrantables:

## 1. Los Tres Pilares de SDOP

### Pilar 1: Spec-Driven (Especificación Viva e Inmutable)
- **Ninguna línea de código se escribe o refactoriza sin una especificación previa.**
- Todo módulo o feature debe contar con un **SDD (Software Design Document)** en `/docs/sdd/[modulo].sdd.md` y, si hay decisiones estructurales, con su respectivo **ADR (Architectural Decision Record)** en `/docs/adr/ADR-XXX-[nombre].md`.
- El SDD define:
  1. Contratos de datos (Request/Response DTOs, campos obligatorios, formatos UTC/UUID).
  2. Reglas de negocio y validaciones de borde.
  3. Matriz de estados y transiciones de ciclo de vida.
  4. Criterios de aceptación verificables.

### Pilar 2: Oracle (El Árbitro de Verdad Automatizado)
- El desarrollador o agente de IA **NO** decide si una feature está completa basándose en su propia interpretación.
- La verdad es dictada por el **Oráculo de Pruebas**:
  - Pruebas E2E (Playwright) que ejecutan los escenarios descritos en la especificación.
  - Pruebas de integración de backend que validan los contratos REST y la persistencia inmutable.
- Si el oráculo falla (rojo), el cambio es rechazado. El estado completado solo se alcanza cuando el oráculo pasa en verde (100% verde).

### Pilar 3: Bridge (Patrón Puente / Hexagonal)
- La UI (Angular) y la Lógica de Negocio (Spring Boot Use Cases) **NUNCA** se acoplan directamente a la infraestructura concreta (APIs directas, JPA, LocalStorage).
- Siempre debe mediar un puerto abstracto (`Repository` o `RepositoryPort`) y uno o más adaptadores intercambiables:
  - `HttpAdapter` (Backend real).
  - `MockAdapter` / `LocalStorageAdapter` (Modo aislado / offline / testing).
- Los orígenes de datos se alternan de forma dinámica mediante **Feature Flags** o inyección de dependencias (`InjectionToken`), garantizando que la UI y el dominio puedan ser probados de forma determinista y aislada.

---

## 2. Flujo Operativo de Trabajo para Agentes de IA

Cuando se reciba una solicitud de desarrollo o refactorización:

1. **Fase de Especificación (Spec):**
   - Localizar o crear el SDD en `/docs/sdd/`.
   - Verificar si el cambio altera un contrato existente o introduce una regla de negocio nueva.
2. **Fase de Oráculo (Oracle):**
   - Escribir o actualizar la suite de pruebas en Playwright (`/e2e/`) o JUnit (`/src/test/`) que modele los criterios de aceptación del SDD.
   - Ejecutar la prueba antes de tocar código fuente para verificar el fallo esperado (TDD / Oracle-Driven).
3. **Fase de Implementación (Bridge):**
   - Implementar los cambios asegurando la separación mediante interfaces de repositorio y DTOs.
   - Cero dependencias directas de entidades JPA en controladores o use cases.
   - Cero llamadas directas a `HttpClient` en componentes de Angular.
4. **Fase de Certificación (Verification):**
   - Ejecutar el oráculo. Si pasa en verde, la tarea se considera finalizada y certificada.
