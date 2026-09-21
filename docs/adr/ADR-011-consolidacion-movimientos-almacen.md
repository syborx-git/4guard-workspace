# ADR-011: Consolidación de Navegación y Vistas Transaccionales en Movimientos de Almacén

- **Estado:** Aceptado
- **Fecha:** 2026-08-14
- **Autores:** Equipo 4Guard WMS (Frontend & AI Assistants)
- **Módulos Afectados:** `apps/admin-console/src/app/features/warehouse-movements`

---

## Contexto y Problema

El sistema legado de gestión de almacenes manejaba una estructura jerárquica de menús y submenús hiper-fragmentada (por ejemplo, submenús independientes para "Capturar Datos", "Alta", "Consulta", "Cancelar", "Cambio de Remisión", "Alta Cambio", "Consulta Cambio", "Alta Salidas", "Consulta Salidas").

Esta navegación provoca:
1. Exceso de clics y pérdida de contexto operativo para los supervisores y choferes en andén.
2. Fragmentación del código Angular en decenas de componentes desacoplados con lógica duplicada.
3. Inconsistencia con el patrón unificado Master-Detail / Split-View homologado en 4GUARD WMS (`/carriers`, `/layout`, `/users`).

---

## Opciones Evaluadas

1. **Reproducción Literal del Menú Legado:** Crear 9 pantallas independientes asignadas a rutas secundarias en Angular.  
   *Desventaja:* Experiencia obsoleta, alta fricción operativa, duplicidad de código.
2. **Consolidación Transaccional Unificada basada en Entidades (Seleccionada):** Consolidar la experiencia en 3 módulos principales centrados en la Entidad de Negocio:
   - **Recepción de Mercancía (`/receiving`):** Vista transaccional donde la búsqueda, captura de caseta, alta de descarga, cambio de remisión y cancelación conviven en el contexto de la entidad `Recepción`.
   - **Cambio de Almacén (`/transfers`):** Registro de traspasos con consulta integrada y comprobación en tiempo real de bahías en ceros.
   - **Salidas de Almacén (`/outbound`):** Despacho outbound con sugerencia FIFO/FEFO y consulta histórica.

---

## Decisión Tomada

Se aprueba la **Consolidación Transaccional Unificada basada en Entidades**.
Se preservan las capacidades del sistema legado, pero se integran dentro del **Design System Synexia** con el patrón **Split-View / Master-Detail** y **Signals de Angular 17+**.

---

## Consecuencias

### Positivas
- Reducción drástica del número de clics y transiciones de pantalla.
- Trazabilidad y contexto persistente en tiempo real.
- Homologación 100% con los estándares visuales de 4GUARD WMS.

### Negativas / Compromisos
- Requiere educar a los usuarios del sistema legado sobre la nueva disposición unificada basada en la entidad de negocio.
