# ADR-002: Componentes Standalone (Eliminación de NgModules)

- **Estado:** Aceptado
- **Fecha:** 2026-07-16
- **Autores:** Equipo 4Guard WMS Frontend
- **Módulos Afectados:** `apps/admin-console`, `apps/rf-terminal`

---

## Contexto y Problema

Tradicionalmente Angular requería declarar componentes dentro de `NgModule`. Esto generaba boilerplate excesivo, acoplamiento innecesario y ralentizaba la carga diferida (lazy loading).

## Opciones Evaluadas

1. **NgModules Tradicionales:** Declarar componentes en módulos de características (`AdminModule`, `CarriersModule`).
2. **Standalone Components (Angular 17+):** Eliminar `NgModule` y declarar dependencias directamente en la propiedad `imports` de la anotación `@Component`.

## Decisión Tomada

Todo nuevo componente en 4GUARD WMS se crea como **Standalone Component** (`standalone: true`). Queda prohibida la creación de nuevos `NgModule`.

## Consecuencias

### Positivas
- Árbol de código más limpio y modular.
- Carga diferida granular por componente a nivel de router (`loadComponent` y `loadChildren` con funciones).
- Facilita el uso de la Signal API de Angular 17+.

### Negativas / Compromisos
- Cada componente debe importar explícitamente `CommonModule`, `ReactiveFormsModule`, etc., en su arreglo de `imports`.
