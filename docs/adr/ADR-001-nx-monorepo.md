# ADR-001: Arquitectura Nx Monorepo (`apps/` + `libs/shared-core`)

- **Estado:** Aceptado
- **Fecha:** 2026-07-15
- **Autores:** Equipo 4Guard WMS Frontend
- **Módulos Afectados:** Monorepo Workspace (`admin-console`, `rf-terminal`, `shared-core`)

---

## Contexto y Problema

El sistema 4GUARD WMS requiere dos aplicaciones frontend especializadas:
1. **Admin Console:** Aplicación web de escritorio de alto rendimiento para gestión y supervisión.
2. **RF Terminal:** PWA optimizada para terminales de radiofrecuencia y tablets en piso de almacén.

Ambas aplicaciones comparten modelos de dominio (roles, organizaciones, estado de inventario, transportistas), reglas de validación y utilidades HTTP. Mantener dos repositorios separados crearía duplicación masiva y desincronización de tipos TypeScript.

## Opciones Evaluadas

1. **Repositorios Múltiples (Polyrepo):** Cada app en un repo separado con paquete npm compartido.  
   *Desventaja:* Necesidad de publicar versiones en un registro npm privado por cada pequeño cambio en los tipos DTO.
2. **Nx Monorepo (Workspace Unificado):** Un único repositorio administrado por Nx Dev Tools.  
   *Ventaja:* Compilación incremental, actualización atómica de contratos compartidos y una sola versión de dependencias.

## Decisión Tomada

Se adopta la arquitectura **Nx Monorepo**. Las aplicaciones residen bajo `apps/` (`admin-console`, `rf-terminal`) y todo el código reutilizable se abstrae a la librería `libs/shared-core` accesible mediante el alias `@4guard/shared-core`.

## Consecuencias

### Positivas
- Cambios en modelos DTO se propagan instantáneamente en todo el monorepo.
- Unificación de scripts de compilación, linter y herramientas de CI/CD.
- Reuso de clientes HTTP, interceptores y enums de negocio.

### Negativas / Compromisos
- Tamaño inicial del repositorio mayor.
- Curva de aprendizaje para desarrolladores nuevos en la CLI de Nx.
