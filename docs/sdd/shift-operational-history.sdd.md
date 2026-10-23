# SDD — Consolidación Operativa por Turno e Historial Multi-Entidad de Remisiones (FE + BE)

> **Módulo:** `shift-operational-history`
> **Repositorios:** `4Guard_FE_UI` (`admin-console`, `shared-core`) y `4guard_be`
> **Arquitectura:** Hexagonal (Ports & Adapters) · Angular 17 Standalone + Signals
> **Estado:** 🟢 Fases 2, 3, 4 y 5 implementadas (turno vigente + sesión + scoping + historial + tableros por turno)
> **ADR:** [ADR-022](../adr/ADR-022-consolidacion-operativa-por-turno-e-historial.md)

---

## 1. Objetivo

Entregar una capa operativa de jornada que:

1. Abra la pantalla de **Recepción** con el filtro **“En Caseta”** por defecto.
2. Encauce **Recepción, Cambio de Almacén y Salidas** al **turno vigente de la sucursal**.
3. Saque del tablero las remisiones **canceladas y verificadas** sin borrarlas, conservándolas en un **historial multi-entidad**.
4. Mantenga visibles los **pendientes** que cruzan el corte, marcados como arrastre.
5. Exponga un **buscador ejecutivo** por fecha (día/mes/año), cliente, estado, tipo y folio.

---

## 2. Alcance

### Incluye
- Resolución del turno vigente por sucursal (automática por configuración y manual).
- Persistencia de la sesión de turno y asociación aditiva a movimientos y auditoría.
- Re-encuadre de tableros y reglas de limpieza visual.
- Historial de recepciones, traspasos y salidas.
- Criterios de aceptación y matriz de pruebas.

### No incluye (fases posteriores)
- Exportación a Excel/PDF del historial.
- Rediseño del catálogo de Turnos.
- Reescritura de la máquina de estados de recepción/salida/traspaso.
- Cambios en la consola de auditoría actual más allá de agregar la nueva página real.

---

## 3. Actores y Permisos

| Actor | Capacidad |
|---|---|
| Operador de almacén | Ver tablero del turno vigente de su sucursal; operar pendientes. |
| Supervisor / Gerente de operaciones | Apertura/cierre manual de sesión; ver historial. |
| Administrador | Configurar turnos; ver historial completo. |
| Auditor | Consultar historial; sin edición. |

Permisos backend: `WAREHOUSE_MOVEMENTS_READ`, `RECEIVING_READ`, `SHIPPING_READ`, `SHIFTS_READ` y los `WAREHOUSE_MOVEMENTS_UPDATE/CANCEL` según operación. Toda consulta se acota por `organizationId` del JWT y `branchId` de la sesión/encabezado.

---

## 4. Glosario

- **Turno:** definición horaria del catálogo (`wms.wms_shifts`).
- **Sesión de turno:** instancia operativa abierta para una sucursal y fecha de negocio (`wms.shift_sessions`).
- **Turno vigente:** sesión `OPEN` de la sucursal, o la resuelta por horario si no existe.
- **Verificado:** recepción `COMPLETED` tras la aprobación de piezas por el usuario.
- **Arrastre:** registro pendiente de una sesión anterior que sigue visible en la actual.
- **Historial:** consulta inmutable de movimientos cerrados/cancelados.

---

## 5. Modelo de Datos (Migración V50+)

> La última migración aplicada es `V49__allow_safe_user_deletion.sql`. **No se edita ninguna migración existente.**

### 5.1 Nueva tabla `wms.shift_sessions`

```sql
CREATE TABLE wms.shift_sessions (
    id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id   UUID NOT NULL REFERENCES wms.organizations(id),
    branch_id         UUID NOT NULL REFERENCES wms.branches(id),
    shift_id          UUID NULL REFERENCES wms.wms_shifts(id),
    business_date     DATE NOT NULL,
    started_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at          TIMESTAMPTZ NULL,
    status            VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    source            VARCHAR(20) NOT NULL DEFAULT 'AUTO',
    opened_by         UUID NULL REFERENCES wms.users(id),
    closed_by         UUID NULL REFERENCES wms.users(id),
    CONSTRAINT chk_shift_session_status CHECK (status IN ('OPEN','CLOSED')),
    CONSTRAINT chk_shift_session_source CHECK (source IN ('AUTO','MANUAL','FALLBACK'))
);

CREATE UNIQUE INDEX uk_shift_sessions_open_branch
    ON wms.shift_sessions(branch_id) WHERE status = 'OPEN';

CREATE INDEX idx_shift_sessions_org_branch_date
    ON wms.shift_sessions(organization_id, branch_id, business_date);
```

### 5.2 Columnas aditivas

```sql
ALTER TABLE wms.warehouse_receptions
    ADD COLUMN shift_session_id UUID NULL REFERENCES wms.shift_sessions(id);
ALTER TABLE wms.warehouse_outbounds
    ADD COLUMN shift_session_id UUID NULL REFERENCES wms.shift_sessions(id);
ALTER TABLE wms.warehouse_transfers
    ADD COLUMN shift_session_id UUID NULL REFERENCES wms.shift_sessions(id);
ALTER TABLE wms.audit_logs
    ADD COLUMN shift_session_id UUID NULL REFERENCES wms.shift_sessions(id);
```

Índices:

```sql
CREATE INDEX idx_wr_shift_session  ON wms.warehouse_receptions(shift_session_id);
CREATE INDEX idx_wo_shift_session  ON wms.warehouse_outbounds(shift_session_id);
CREATE INDEX idx_wt_shift_session  ON wms.warehouse_transfers(shift_session_id);
CREATE INDEX idx_audit_shift_session ON wms.audit_logs(shift_session_id);
```

### 5.3 Reglas de datos

- `shift_session_id` es nullable para no romper registros históricos existentes.
- Los registros nuevos se estampan al crear (recepción/salida/traspaso) y en cada evento de auditoría.
- No se ejecuta UPDATE/DELETE sobre `audit_logs` (WORM).
- `business_date` se calcula en la zona horaria de la sucursal.

---

## 6. Contratos REST

### 6.1 `GET /api/v1/shifts/active?branchId={uuid}`

**Response** `ApiResponse<ActiveShiftSessionResponse>`:

```json
{
  "success": true,
  "message": "Turno vigente resuelto con éxito",
  "data": {
    "sessionId": null,
    "shiftId": "f13f0907-9fa5-4bdf-87db-2eb5e7683952",
    "shiftCode": "TRN-VES-01",
    "shiftName": "Turno Vespertino CDMX",
    "startTime": "08:00:00",
    "endTime": "16:00:00",
    "isOvernight": false,
    "toleranceMinutes": 15,
    "businessDate": "2026-10-23",
    "branchId": "b73f0907-9fa5-4bdf-87db-2eb5e7683936",
    "branchName": "CENTRO DE DISTRIBUCION TOLUCA",
    "timezone": "America/Mexico_City",
    "configured": true,
    "source": "AUTO",
    "resolvedAt": "2026-10-23T17:11:52Z"
  }
}
```

Notas de implementación (Fases 2 y 3):

- `sessionId` se puebla en Fase 3. Si no existía sesión abierta, el endpoint la crea de forma idempotente.
- `configured = false` cuando la sucursal no tiene turnos activos; `shiftId = null` y `source = "FALLBACK"`.
- `configured = true` y `shiftId = null` cuando hay turnos configurados pero ninguno está vigente en este instante.
- El endpoint es accesible para cualquier usuario autenticado y valida que la sucursal pertenezca a su organización.
- Verificado contra backend real: resolvió el turno vigente respetando zona horaria, día operativo y tolerancia; dos llamadas consecutivas devolvieron el mismo `sessionId`.
- Migración `V50__shift_sessions_and_operational_scoping.sql` aplicada correctamente en PostgreSQL.
- `shift_session_id` se estampa al crear recepción, salida y traspaso, y en cada `audit_logs` cuando hay sesión abierta.
- Listados de movimientos soportan `shiftSessionId` y `scope=CURRENT|HISTORY|ALL`. Validado: `CURRENT` devuelve solo estados activos; `HISTORY` devuelve cerrados/cancelados; `ALL` conserva el comportamiento previo.

### 6.2 `POST /api/v1/shifts/sessions/open`

Request:

```json
{ "branchId": "uuid", "shiftId": "uuid|null", "reason": "string|null" }
```

Response `ApiResponse<ActiveShiftSessionResponse>`. Idempotente: si ya existe sesión `OPEN` para la sucursal, la devuelve.

### 6.3 `POST /api/v1/shifts/sessions/{id}/close`

Request:

```json
{ "reason": "string|null" }
```

Response `ApiResponse<ActiveShiftSessionResponse>` con `status = "CLOSED"` y `endedAt`.

### 6.4 Listados extendidos (aditivos)

`GET /warehouse-receptions`, `/warehouse-outbounds`, `/warehouse-transfers` aceptan:

| Parámetro | Tipo | Descripción |
|---|---|---|
| `shiftSessionId` | UUID (opcional) | Filtra por sesión. |
| `scope` | `CURRENT` \| `HISTORY` \| `ALL` | `CURRENT` = sesión vigente + activos + arrastre; `HISTORY` = cerrados/cancelados; `ALL` = sin filtro. Default `ALL` para compatibilidad. |
| `status` | string | Se mantiene el contrato actual. |
| `search` | string | Se mantiene. |

### 6.5 `GET /api/v1/audit/remissions`

Query params:

| Parámetro | Tipo | Descripción |
|---|---|---|
| `from`, `to` | ISO-8601 UTC | Rango de fechas. La UI calcula los atajos día/mes/año. |
| `clientId` | UUID | Cliente/depositante. Los traspasos se excluyen al filtrar por cliente porque no tienen cliente asociado. |
| `status` | string | `COMPLETED`, `CANCELLED`, etc. |
| `entityType` | `RECEPTION` \| `OUTBOUND` \| `TRANSFER` | Tipo. |
| `shiftSessionId` | UUID | Turno. |
| `search` | string | Folio, remisión, cliente, operador. |
| `page`, `size` | int | Paginación. |

**Response** `ApiResponse<RemissionHistoryPageResponse>`:

```json
{
  "success": true,
  "message": "Historial de remisiones",
  "data": {
    "items": [
      {
        "id": "uuid",
        "entityType": "RECEPTION",
        "folio": "REC-2026-000123",
        "remisionNumber": "REM-99881",
        "clientId": "uuid",
        "clientName": "NESTLÉ MÉXICO",
        "branchId": "uuid",
        "branchName": "BR-TOL-01",
        "status": "COMPLETED",
        "shiftSessionId": "uuid",
        "shiftName": "Turno 1",
        "operator": "enrique",
        "pallets": 24,
        "pieces": 48,
        "createdAt": "2026-10-22T08:15:00Z",
        "completedAt": "2026-10-22T09:30:00Z",
        "cancelledAt": null,
        "cancellationReason": null
      }
    ],
    "summary": {
      "total": 8,
      "completed": 5,
      "cancelled": 1,
      "pending": 2,
      "byClient": [{ "clientId": "uuid", "clientName": "NESTLÉ MÉXICO", "count": 3 }],
      "byShift": [{ "shiftSessionId": "uuid", "shiftName": "Turno 1", "count": 5 }]
    },
    "page": 1,
    "size": 20,
    "totalPages": 1
  }
}
```

### 6.6 `GET /api/v1/audit/user-activity` (extensión)

Se agregan query params opcionales `entityType` y `shiftSessionId`. Sin ellos, el comportamiento actual no cambia.

---

## 7. DTOs y Mapeo

### Backend
- `ActiveShiftSessionResponse`
- `OpenShiftSessionRequest`, `CloseShiftSessionRequest`
- `RemissionHistoryItemResponse`, `RemissionHistoryPageResponse`, `RemissionHistorySummaryResponse`
- `ShiftSession` (dominio), `ShiftSessionRepositoryPort`, `ShiftSessionService`.

### Frontend
- `ActiveShiftSession`, `RemissionHistoryItem`, `RemissionHistorySummary`, `RemissionHistoryFilters`.
- `ShiftContextService` con signals: `activeSession`, `configured`, `isLoading`, `carryOverCount`, `refresh()`.
- Extensión opcional de `ReceptionHeader`, `OutboundHeader`, `TransferHeader` con `shiftSessionId` y `carryOver`.

---

## 8. Reglas de Negocio y Transiciones

1. **Apertura automática:** al consultar el turno vigente, si no hay sesión `OPEN` y hay turno aplicable por horario, se abre (`source = AUTO`).
2. **Fallback:** sin turnos configurados, se abre sesión (`source = FALLBACK`, `shiftId = null`) para no bloquear la operación.
3. **Corte:** al abrir una nueva sesión, la anterior pasa a `CLOSED` con `ended_at`. No se borra.
4. **Estampado:** toda recepción/salida/traspaso creada dentro de una sesión `OPEN` hereda `shift_session_id`.
5. **Tablero `CURRENT`:**
   - Incluye estados `REGISTERED`, `ASSIGNED`, `IN_PROGRESS`, `DISCHARGED` de la sesión vigente.
   - Incluye pendientes de sesiones anteriores con `carryOver = true`.
   - Excluye `COMPLETED` y `CANCELLED`.
6. **Verificación:** pasa a `COMPLETED` solo cuando el usuario aprueba las piezas. Habilita la descarga del formato (ADR-015).
7. **Cancelación:** pasa a `CANCELLED`; sale del tablero; permanece en historial.
8. **Cambio de usuario:** no altera el alcance del tablero (sucursal + sesión).
9. **Inmutabilidad:** el historial se alimenta de registros existentes y de `audit_logs`; no se reescribe.

---

## 9. Frontend

### 9.1 Recepción — filtro por defecto
- Archivo: `apps/admin-console/src/app/features/warehouse-movements/pages/receiving-submodule/receiving-submodule.component.ts`.
- Cambio: `statusFilter = signal('REGISTERED')` y `activeStatusFilters = signal(['REGISTERED'])` (líneas 99 y 791).
- La lógica de filtrado (`:854-891`) no cambia.

### 9.2 `ShiftContextService`
- Nuevo servicio en `features/warehouse-movements/services/` o `core/services/`.
- `loadActiveSession(branchId)` → `GET /shifts/active`.
- Signals: `activeSession`, `configured`, `isLoading`, `carryOverCount`.
- Reintento/refresh al cambiar de turno (`endedAt` alcanzado) y al cambiar de sucursal.

### 9.3 Tableros
- Recepción: filtrar `receptions()` por sesión y `scope=CURRENT` en un `computed` local; **no mutar** la señal global de `WarehouseMovementsService`.
- Traspasos y Salidas: mismo patrón con sus servicios.
- Chip “Turno actual” y contador de arrastre en el header.
- Si `configured = false`: chip “Turno no configurado (ventana de respaldo)”.

### 9.4 Historial
- Ruta: `/admin/remissions-history` (lazy, `rbacGuard`).
- Página con:
  - Tarjetas ejecutivas: total, verificadas, canceladas, pendientes, por cliente, por turno.
  - Búsqueda avanzada: rango por día/mes/año, cliente (select searchable), estado, tipo, folio.
  - Tabla paginada + drawer de detalle reutilizando el patrón `audit-log-drawer`.
- No reutilizar `AuditLogService`.

### 9.5 Gating de descarga
- El botón “Descargar formato de recepción” se habilita solo si `status === 'COMPLETED'` y la aprobación quedó registrada.

---

## 10. Criterios de Aceptación

| ID | Criterio | Verificación |
|---|---|---|
| CA-01 | Al abrir Recepción, la píldora activa es “En Caseta”. | Playwright |
| CA-02 | El tablero operativo muestra solo la sesión vigente y estados activos. | Playwright |
| CA-03 | Una recepción `COMPLETED` desaparece del tablero y aparece en historial. | Playwright |
| CA-04 | Una recepción `CANCELLED` desaparece del tablero y aparece en historial. | Playwright |
| CA-05 | Los pendientes de la sesión anterior siguen visibles con etiqueta “Arrastre turno anterior”. | Playwright |
| CA-06 | Al cambiar de usuario en la misma sucursal, el tablero no cambia de alcance. | Playwright |
| CA-07 | Al cerrar/cambiar turno, el tablero se re-encuadra sin recargar manualmente. | Playwright |
| CA-08 | Sin turnos configurados, la operación continúa con `source = FALLBACK` y aviso visible. | Playwright + JUnit |
| CA-09 | La descarga del formato solo se habilita con recepción aprobada (`COMPLETED`). | Playwright |
| CA-10 | El historial filtra por día, mes, año, cliente, estado, tipo y folio. | Playwright |
| CA-11 | El historial incluye recepciones, traspasos y salidas. | JUnit + Playwright |
| CA-12 | No existe más de una sesión `OPEN` por sucursal. | JUnit |
| CA-13 | Los endpoints existentes mantienen su contrato sin los nuevos parámetros. | JUnit |

---

## 11. Matriz de Pruebas (Oráculo)

### Playwright (`4Guard_FE_UI/e2e/`)
- `shift-default-filter.spec.ts`: CA-01.
- `shift-board-scoping.spec.ts`: CA-02, CA-03, CA-04, CA-05, CA-06.
- `shift-rollover.spec.ts`: CA-07, CA-08.
- `reception-download-gating.spec.ts`: CA-09.
- `remission-history.spec.ts`: CA-10, CA-11.

### JUnit (`4guard_be/src/test/`)
- `ShiftSessionServiceTest`: resolución por horario, overnight, fallback, unicidad OPEN (CA-08, CA-12).
- `WarehouseMovementScopingTest`: estampado y scoping por sesión (CA-02, CA-03, CA-04).
- `RemissionHistoryControllerTest`: contrato, filtros y paginación (CA-10, CA-11).
- `ContractCompatibilityTest`: endpoints sin parámetros nuevos (CA-13).

---

## 12. Riesgos y Mitigaciones

| Riesgo | Mitigación |
|---|---|
| Migración rompe datos existentes | Columnas nullable, sin defaults obligatorios, V50+ aditiva. |
| Doble sesión abierta por concurrencia | Índice único parcial + manejo de conflicto idempotente. |
| Turnos nocturnos o solapados | Resolución por `is_overnight`, `tolerance_minutes` y alcance; pruebas dedicadas. |
| Reconfiguración de turno afecta historial | `shift_id` snapshot en la sesión; `business_date` inmutable. |
| Multi-tenancy | `organizationId` desde JWT; validar pertenencia de `branchId`. |
| WORM de auditoría | Solo INSERT; el historial lee, no reescribe. |
| Fallbacks actuales (primer usuario habilitado) contaminan trazabilidad | No empeorar; registrar actor real y documento como deuda técnica. |
| Divergencia documental | ADR/SDD canónicos en `docs/` del workspace. |
| RBAC incompleto | Alinear permisos `WAREHOUSE_MOVEMENTS_*`/`SHIFTS_*` antes de Fase 4. |

---

## 13. Plan por Fases

1. **Fase 0:** este SDD + ADR + pruebas del oráculo.
2. **Fase 1:** filtro por defecto “En Caseta” (FE, independiente).
3. **Fase 2:** `GET /shifts/active` + `ShiftContextService` (lectura).
4. **Fase 3:** migración V50, `shift_session_id`, estampado y scoping.
5. **Fase 4:** `/audit/remissions` + página de historial.
6. **Fase 5:** re-encuadre automático, arrastre, gating de descarga y quality gates.

---

### 6.7 Implementación Fase 4 (historial real)

- Backend: `GET /api/v1/audit/remissions` implementado con unión en memoria de recepciones, salidas y traspasos, filtros de fecha, cliente, estatus, tipo, turno, búsqueda y paginación.
- Resumen ejecutivo: total, verificadas, canceladas, pendientes, por cliente y por turno.
- Frontend: página real `Administración → Historial de Remisiones` (`/admin/remissions-history`) con tarjetas ejecutivas, presets Hoy/Mes/Año/Personalizado, filtros de cliente/estatus/tipo/búsqueda, tabla paginada y drawer de detalle.
- Validado contra datos reales: 11 remisiones (5 recepciones, 3 salidas, 3 traspasos), filtros por entidad/estatus/búsqueda/cliente y paginación correctos.
- Nota: los registros históricos anteriores a la Fase 3 muestran `shiftName = null` porque no tienen `shift_session_id`. Los movimientos nuevos sí se asocian al turno.

### 6.8 Implementación Fase 5 (tableros por turno)

- Los tableros de Recepción, Cambios de Almacén y Salidas filtran localmente por `shiftSessionId` + estados activos.
- Los pendientes de sesiones anteriores permanecen visibles y se etiquetan como **“Arrastre turno anterior”**.
- Verificados y cancelados de sesiones anteriores salen del tablero operativo y quedan en el historial.
- `ShiftContextService` refresca el turno vigente cada 60 segundos; al cambiar la sesión, los tableros se re-encuadran solos.
- Los tres tableros muestran un chip con el turno vigente y su horario.
- Los DTOs de resumen y detalle (recepción, salida, traspaso) exponen `shiftSessionId` para que el frontend pueda aplicar el scoping.
- La descarga del formato F01 ya estaba correctamente restringida a recepciones `COMPLETED`; no requirió cambios.
- Compilación validada: backend `BUILD SUCCESS` y frontend build exitoso.

## 14. Trazabilidad

- ADR: [ADR-022](../adr/ADR-022-consolidacion-operativa-por-turno-e-historial.md)
- Reglas globales: ADR-006, ADR-007, ADR-008, ADR-014.
- Operación: ADR-019, ADR-020, ADR-021.
- Documento: ADR-015 (formato de recepción).
