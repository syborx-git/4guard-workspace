# ADR-010: Adopción del Flujo Spec-Driven Development (SDD)

- **Estado:** Aceptado
- **Fecha:** 2026-07-31
- **Autores:** Equipo 4Guard WMS (Frontend & AI Assistants)
- **Módulos Afectados:** Proceso de Desarrollo & Documentación (`docs/`, AI Prompt Engine)

---

## Contexto y Problema

El desarrollo asistido por Inteligencia Artificial (IA) en proyectos grandes suele sufrir de alucinaciones de código, invención de endpoints inexistentes, inconsistencias en el diseño visual y rediscusiones repetitivas de arquitectura.

## Opciones Evaluadas

1. **Desarrollo Tradicional Ad-hoc:** Escribir código directamente o pasarle prompts sueltos a la IA sin contexto previo.  
   *Desventaja:* Código deshomologado, bugs de integración con el Backend y reescrituras constantes.
2. **Spec-Driven Development (SDD):** Todo cambio o nuevo módulo requiere definir/verificar la especificación en `docs/` antes de escribir código.

## Decisión Tomada

Se adopta **Spec-Driven Development (SDD)** como la metodología oficial del proyecto.
El flujo obligatorio consta de 5 pasos:
1. **Especificar:** Crear/verificar el contrato en `docs/api/modules/[modulo].md`.
2. **Trackear:** Actualizar la matriz de estado en `docs/architecture/module-status.md` (de `⬜` a `🔧`).
3. **Promptear:** Usar las plantillas homologadas de `docs/ai/prompt-library.md`.
4. **Generar & Self-Correction:** Construir el módulo Standalone + Signals + CSS Variables + BD Real.
5. **Verificar & Cerrar:** Validar runtime y marcar `✅ Completo` en `module-status.md`.

## Consecuencias

### Positivas
- Generación de código production-ready al primer intento.
- Homologación visual 100% garantizada con las pantallas de referencia (*Golden Standard*).
- Reducción drástica de deuda técnica y reescrituras.

### Negativas / Compromisos
- Exige mantener la documentación en `docs/` siempre actualizada como la fuente de verdad del sistema.
