# ADR-022: Consolidación Operativa por Turno y Historial Multi-Entidad de Remisiones

- **Estado:** Aceptado / Implementación completa (Fases 2–5)
- **Fecha:** 2026-10-23
- **Autores:** Equipo de Arquitectura e Ingeniería 4GUARD WMS (Frontend, Backend y Operaciones)
- **Módulos Afectados:**
  - `apps/admin-console/src/app/features/warehouse-movements` (Recepción, Traspasos, Salidas)
  - `apps/admin-console/src/app/features/admin` (Turnos y Auditoría/Historial)
  - `libs/shared-core`
  - `4guard_be/src/main/java/com/fourguard/wms` (dominio, casos de uso, controladores y persistencia)
  - Tablas: `wms.wms_shifts`, `wms.shift_sessions` (nueva), `wms.warehouse_receptions`, `wms.warehouse_outbounds`, `wms.warehouse_transfers`, `wms.audit_logs`
- **ADRs Relacionados:**
  - [ADR-006 (Angular Signals)](./ADR-006-angular-signals-state.md)
  - [ADR-007 (Cero Mocks y Persistencia Real)](./ADR-007-zero-mock-database-rule.md)
  - [ADR-008 (Contratos API, UUID, ISO-8601 UTC)](./ADR-008-api-contracts-uuid-utc.md)
  - [ADR-011 (Consolidación de Movimientos de Almacén)](./ADR-011-consolidacion-movimientos-almacen.md)
  - [ADR-014 (Hexagonal, Multi-tenant y Auditoría)](./ADR-014-arquitectura-backend-hexagonal-multitenant-audit.md)
  - [ADR-015 (Motor Documental PDF/ZPL)](./ADR-015-motor-documental-pdf-zpl-export.md)
  - [ADR-019 (Flujo Integral de Salidas)](./ADR-019-flujo-integral-salidas-almacen-caseta-y-despacho.md)
  - [ADR-020 (Flujo Integral de Recepción)](./ADR-020-flujo-integral-recepcion-almacen-caseta-y-descarga.md)
  - [ADR-021 (Reingresos y Trazabilidad Multiciclo)](../../4Guard_FE_UI/docs/adr/ADR-021-gestion-reingresos-logistica-inversa-trazabilidad.md)

> **Fuente de verdad única:** esta decisión vive en `docs/` del workspace. No se duplica en los subrepositorios para evitar la divergencia documental ya detectada entre `docs/` y `4Guard_FE_UI/docs/`.

---

## 1. Contexto y Problema

La operación diaria de almacén (Recepción, Cambio de Almacén y Salidas) hoy muestra un tablero acumulativo sin contexto de jornada:

1. **El filtro de Recepción nace en “Todos”** (`receiving-submodule.component.ts:99,791`), cuando el punto de entrada operativo real es “En Caseta” (`REGISTERED`).
2. **No existe contexto de turno en los movimientos.** Las tablas `warehouse_receptions`, `warehouse_outbounds` y `warehouse_transfers` no tienen `shift_id` ni equivalente; tampoco `audit_logs`. El catálogo de turnos existe (`wms.wms_shifts`), pero no hay “turno vigente” ni sesión operativa.
3. **El tablero no se limpia al cambiar de jornada.** Remisiones canceladas y verificadas siguen ocupando espacio visual y compiten con el trabajo pendiente.
4. **No hay historial consultable de remisiones.** La “Consola de Auditoría” actual es un servicio local en memoria (`audit.service.ts:5-16,22-71`), no una fuente real. La fuente real (`GET /audit/user-activity`) no filtra por entidad, estado de remisión ni turno.
5. **Riesgo de pérdida de trazabilidad.** Cualquier solución que borre o mute registros choca con el modelo WORM de `audit_logs` y con las reglas de estados del backend.

El objetivo es **encajar una pieza aditiva**: tableros operativos por turno, limpieza visual (no física) de canceladas/verificadas, e historial multi-entidad con búsqueda ejecutiva, sin romper los contratos ni la operación actual.

---

## 2. Opciones Evaluadas

### Opción A — Derivar el turno por ventana horaria, sin persistir
- **Pros:** no requiere migración; se puede calcular en consulta.
- **Contras:** no hay trazabilidad histórica fiable; ambigüedad con turnos solapados, nocturnos y tolerancias; un cambio de configuración reescribe el pasado; imposible auditar “qué turno tenía esta remisión”.

### Opción B — Agregar `shift_id` directo a los movimientos
- **Pros:** simple; asocia cada registro a un turno del catálogo.
- **Contras:** no modela el ciclo real de la jornada (apertura/cierre, cambio de operador, corte por hora). Si el turno se reconfigura, los registros históricos quedan apuntando a una definición mutable. No resuelve “cuándo se limpia el tablero”.

### Opción C — Sesión de turno (`shift_sessions`) + `shift_session_id` (elegida)
- **Pros:** modela la jornada real; permite corte automático por horario y override manual; sobrevive a reconfiguraciones del catálogo (snapshot por sesión); habilita historial por sesión y por sucursal; es aditiva.
- **Contras:** requiere una migración nueva y un caso de uso de resolución de turno vigente; introduce un concepto adicional que el equipo debe operar.

---

## 3. Decisión Tomada

Se adopta la **Opción C**: una **sesión operativa de turno por sucursal**, con resolución automática por configuración de turnos y fallback cuando aún no hay turnos configurados.

### 3.1 Modelo de datos

Nueva tabla:

```sql
-- Migración nueva V50+ (no editar migraciones existentes)
CREATE TABLE wms.shift_sessions (
    id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id   UUID NOT NULL REFERENCES wms.organizations(id),
    branch_id         UUID NOT NULL REFERENCES wms.branches(id),
    shift_id          UUID NULL REFERENCES wms.wms_shifts(id),
    business_date     DATE NOT NULL,
    started_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at          TIMESTAMPTZ NULL,
    status            VARCHAR(20) NOT NULL DEFAULT 'OPEN', -- OPEN | CLOSED
    source            VARCHAR(20) NOT NULL DEFAULT 'AUTO', -- AUTO | MANUAL | FALLBACK
    opened_by         UUID NULL,
    closed_by         UUID NULL
);
```

Columnas aditivas y nullable:

```sql
ALTER TABLE wms.warehouse_receptions ADD COLUMN shift_session_id UUID NULL REFERENCES wms.shift_sessions(id);
ALTER TABLE wms.warehouse_outbounds  ADD COLUMN shift_session_id UUID NULL REFERENCES wms.shift_sessions(id);
ALTER TABLE wms.warehouse_transfers  ADD COLUMN shift_session_id UUID NULL REFERENCES wms.shift_sessions(id);
ALTER TABLE wms.audit_logs           ADD COLUMN shift_session_id UUID NULL REFERENCES wms.shift_sessions(id);
```

- Índice único parcial para impedir dos sesiones abiertas por sucursal:
  `CREATE UNIQUE INDEX uk_shift_sessions_open_branch ON wms.shift_sessions(branch_id) WHERE status = 'OPEN';`
- Índices por `shift_session_id` en las cuatro tablas.
- `shift_id` es nullable a propósito: la sesión puede existir **antes** de que el usuario configure turnos (`source = 'FALLBACK'`).

### 3.2 Resolución del turno vigente

- La sesión se llavea por **sucursal** (decisión de negocio: el tablero se comparte por sucursal, no por usuario).
- `GET /shifts/active?branchId=` resuelve:
  1. Si existe sesión `OPEN` para la sucursal, la devuelve.
  2. Si no, busca un turno `ACTIVE` aplicable por día/hora (`operating_days`, `start_time`, `end_time`, `is_overnight`, `tolerance_minutes`), considerando alcance `GLOBAL | BRANCH | WAREHOUSE_SECTION`.
  3. Si hay coincidencia, abre la sesión de forma idempotente y la devuelve.
  4. Si no hay turnos configurados, abre una sesión `FALLBACK` con `shift_id = NULL` y `business_date` local.
- La resolución es **perezosa** (en la petición) y opcionalmente reforzada por un job de cierre; así no se depende de infraestructura de cron para operar.
- Al abrir una nueva sesión, la anterior se cierra con `ended_at = fin del turno` (no se borra).

### 3.3 Reglas operativas

1. **Filtro por defecto de Recepción:** “En Caseta” (`REGISTERED`). El usuario puede navegar a En Andén, Verificado, Retornos, Canceladas y Todos.
2. **Tablero por defecto:** sesión vigente de la sucursal + estados activos (`REGISTERED`, `ASSIGNED`, `IN_PROGRESS`, `DISCHARGED`).
3. **Verificado:** una recepción pasa a `COMPLETED` cuando el usuario aprueba las piezas cargadas. Solo entonces se habilita la descarga del formato de recepción (motor PDF del ADR-015). No se inventa un estado nuevo.
4. **Canceladas y verificadas:** salen del tablero operativo y quedan en el historial. **No se eliminan físicamente.**
5. **Pendientes que cruzan el corte:** permanecen visibles en el tablero del turno entrante con la etiqueta `Arrastre turno anterior` (`carryOver = true`), porque no están canceladas ni verificadas.
6. **Cambio de usuario:** el tablero no se aísla por usuario; todos los operadores de la sucursal ven el mismo turno y el mismo arrastre.
7. **Historial:** incluye Recepción, Traspasos y Salidas. Vista de tarjetas ejecutivas + búsqueda avanzada por fecha (día/mes/año), cliente, estado, tipo de entidad y folio. Sin exportación en esta fase.

### 3.4 Contratos API (aditivos)

| Verbo | Endpoint | Cambio |
|---|---|---|
| `GET` | `/shifts/active?branchId=` | Nuevo. Devuelve sesión vigente + `configured`. |
| `POST` | `/shifts/sessions/open` | Nuevo. Apertura manual (supervisor/admin). |
| `POST` | `/shifts/sessions/{id}/close` | Nuevo. Cierre manual. |
| `GET` | `/warehouse-receptions` | Nuevo query opcional `shiftSessionId` y `scope=CURRENT|HISTORY|ALL`. Sin el parámetro, comportamiento actual. |
| `GET` | `/warehouse-outbounds` | Idem. |
| `GET` | `/warehouse-transfers` | Idem. |
| `GET` | `/audit/remissions` | Nuevo. Historial multi-entidad paginado + resumen ejecutivo. |
| `GET` | `/audit/user-activity` | Nuevos query opcionales `entityType`, `shiftSessionId`. |

Todo response se mantiene envuelto en `ApiResponse<T>`, IDs UUID y fechas ISO-8601 UTC (ADR-008).

### 3.5 Frontend

- `ReceivingSubmoduleComponent`: inicializar `statusFilter` y `activeStatusFilters` en `REGISTERED`.
- Nuevo `ShiftContextService` (signals): `activeSession`, `configured`, `carryOverCount`, `refresh()`.
- Los tableros de Recepción/Traspasos/Salidas consumen el contexto de turno **sin mutar** las señales globales de `WarehouseMovementsService` (para no romper `pendingReceptions` ni ocupación de rampas).
- Nueva página real de historial en Administración → Auditoría (`/admin/remissions-history`), con tarjetas ejecutivas, filtros avanzados y drawer de detalle. **No** se reutiliza `AuditLogService`.
- El botón de descarga del formato de recepción se habilita solo con `status = COMPLETED` y aprobación registrada.

### 3.6 Historial multi-entidad

`GET /audit/remissions` devuelve una fila por remisión/folio con:

- `entityType` (`RECEPTION | OUTBOUND | TRANSFER`), `folio`, `remisionNumber`, `clientId/clientName`, `status`.
- `shiftSessionId`, `shiftName`, `branchId`, `operator`, `createdAt`, `completedAt`, `cancelledAt`, `cancellationReason`.
- Totales (`pallets`, `pieces`) y resultado de verificación cuando aplique.
- Resumen ejecutivo: conteos por estado, por cliente y por turno.

---

## 4. Consecuencias

### Positivas
- Tablero operativo limpio y alineado a la jornada real, sin pérdida de información.
- Trazabilidad histórica inmutable por sesión de turno y sucursal.
- Historial único para recepción, traspasos y salidas.
- Cambios aditivos: los endpoints y DTOs actuales siguen funcionando.
- Funciona aun antes de que el usuario configure turnos (fallback explícito).

### Negativas / Compromisos
- Nueva migración y un caso de uso de resolución de turno que debe mantenerse.
- La sesión de turno es un concepto operativo nuevo: requiere capacitación y permisos.
- Riesgo de desfase si la configuración de turnos cambia; se mitiga con snapshot (`shift_id` en la sesión) y `business_date`.
- La vista histórica agrega carga de consulta; requiere índices y paginación.

---

## 5. Plan por Fases

| Fase | Alcance | Repo |
|---|---|---|
| 0 | Este ADR + SDD + pruebas del oráculo | docs |
| 1 | Filtro por defecto “En Caseta” | FE |
| 2 | `GET /shifts/active` + `ShiftContextService` (solo lectura) | BE + FE |
| 3 | Migración V50, `shift_session_id`, scoping y estampado | BE |
| 4 | Historial `/audit/remissions` + página real | BE + FE |
| 5 | Re-encuadre automático, arrastre, gating de descarga y quality gates | FE |

---

## 6. Notas de Auditoría y Verificación

- **Oráculo Playwright:** filtro por defecto, tablero por turno, arrastre, historial y filtros de fecha/cliente.
- **Oráculo JUnit:** resolución de sesión, corte por horario, unicidad de sesión abierta por sucursal, scoping multi-tenant, inmutabilidad de `audit_logs` y no mutación de canceladas/verificadas.
- **Quality gates:** `npm run sdd:review`, `ng build admin-console` y pruebas de integración del backend en verde.
- **Prohibiciones:** no editar migraciones V1–V49; no hacer UPDATE/DELETE en `audit_logs`; no romper firmas existentes; no usar `AuthState.currentShift` como `shiftId`.

---

## 7. Trazabilidad

- SDD: [`../sdd/shift-operational-history.sdd.md`](../sdd/shift-operational-history.sdd.md)
- Implementación FE: pendiente de aprobación.
- Implementación BE: pendiente de aprobación (rama `edgar-cambiosPablo` desde `develop` cuando inicie).
