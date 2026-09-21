# SDD: Gestión de Montacarguistas — Backend

> **Módulo:** Catálogo de Montacarguistas (Forklift Operators)  
> **HU:** HU-142 — Gestión de Montacarguistas  
> **Versión:** 1.0.0  
> **Estado:** 🏁 CERRADO / LISTO PARA PR  
> **Fecha de Cierre:** 2026-08-18  
> **Rama:** `catalogo-montacarga-be`  

---

## 1. Descripción del Módulo

El módulo de **Gestión de Montacarguistas** administra el catálogo de operadores certificados responsables de las maniobras físicas en andenes: descarga en recepción, reubicaciones, traspasos entre bahías y carga en despacho.

> **⚠️ Importante:** Los montacarguistas **NO son usuarios del sistema**. No tienen credenciales de login ni roles en el WMS. Son un catálogo operativo de referencia para asignar responsables a los movimientos de almacén.

---

## 2. Modelo de Datos (BD: `wms.forklift_operators`)

| Campo | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `id` | UUID | PK, NOT NULL | Identificador único del registro |
| `organization_id` | UUID | FK `wms.organizations.id`, NOT NULL | Organización propietaria |
| `branch_id` | UUID | FK `wms.branches.id`, NULLABLE | Sucursal asignada (opcional) |
| `code` | VARCHAR(30) | NOT NULL, UNIQUE(org) | Código operativo auto-generado `MC-001` |
| `first_name` | VARCHAR(100) | NOT NULL | Nombre(s) del operador |
| `last_name_paternal` | VARCHAR(100) | NOT NULL | Apellido paterno |
| `last_name_maternal` | VARCHAR(100) | NOT NULL | Apellido materno |
| `full_name` | VARCHAR(310) | NOT NULL | Concatenación desnormalizada (service) |
| `license_number_dc3` | VARCHAR(50) | NOT NULL, UNIQUE(org) | Número DC-3 ante STPS |
| `license_expiration_date` | DATE | NOT NULL | Fecha de vencimiento de certificación |
| `license_status` | VARCHAR(20) | NOT NULL, DEFAULT `VIGENTE` | Calculado: `VIGENTE`, `POR_VENCER`, `VENCIDA` |
| `shift_id` | UUID | FK `wms.wms_shifts.id`, NULLABLE | Turno asignado del catálogo maestro |
| `shift_name` | VARCHAR(150) | NULLABLE | Nombre desnormalizado del turno |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `ACTIVO` | `ACTIVO` \| `INACTIVO` |
| `is_deleted` | BOOLEAN | NOT NULL, DEFAULT FALSE | Soft-delete flag |
| `version` | BIGINT | NOT NULL, DEFAULT 1 | Optimistic locking |
| `created_at` | TIMESTAMPTZ | AUTO | Timestamp de creación |
| `updated_at` | TIMESTAMPTZ | AUTO | Timestamp de última modificación |
| `created_by` | VARCHAR(36) | FK usuario | Username del creador |
| `updated_by` | VARCHAR(36) | FK usuario | Username del último editor |

### Reglas de Vigencia de Licencia

```
daysRemaining = licenseExpirationDate - TODAY
  if daysRemaining > 30  → VIGENTE    (verde)
  if daysRemaining 0-30  → POR_VENCER (amarillo/alerta)
  if daysRemaining < 0   → VENCIDA    (rojo)
```

---

## 3. Endpoints REST

**Base path:** `/api/v1/forklift-operators`

| Método | Ruta | Permiso | Descripción |
|---|---|---|---|
| `POST` | `/` | `FORKLIFT_OPERATORS_CREATE` | Crear nuevo montacarguista. Código auto-generado `MC-XXX`. |
| `PUT` | `/{id}` | `FORKLIFT_OPERATORS_UPDATE` | Actualizar datos. Código inmutable. |
| `GET` | `/{id}` | `FORKLIFT_OPERATORS_READ` | Obtener detalle por UUID. |
| `GET` | `/` | `FORKLIFT_OPERATORS_READ` | Listar con filtros: `organizationId`, `branchId`, `status`, `licenseStatus`, `search`. |
| `DELETE` | `/{id}` | `FORKLIFT_OPERATORS_DELETE` | Baja lógica (soft delete). |
| `PATCH` | `/{id}/status` | `FORKLIFT_OPERATORS_STATUS_CHANGE` | Alternar `ACTIVO` / `INACTIVO`. |
| `GET` | `/{id}/audit` | `FORKLIFT_OPERATORS_READ` | Historial de auditoría con deltas por campo. |

### Ejemplo: Crear Montacarguista

**Request** `POST /api/v1/forklift-operators`
```json
{
  "organizationId": "a53f0907-9fa5-4bdf-87db-2eb5e7683935",
  "firstName": "Juan Manuel",
  "lastNamePaternal": "Pérez",
  "lastNameMaternal": "García",
  "licenseNumberDc3": "LIC-MC-001",
  "licenseExpirationDate": "2027-06-30",
  "shiftId": "c12f0907-9fa5-4bdf-87db-2eb5e7683940"
}
```

**Response** `200 OK`
```json
{
  "success": true,
  "message": "Montacarguista registrado con éxito",
  "data": {
    "id": "d45f0907-9fa5-4bdf-87db-2eb5e7683950",
    "code": "MC-001",
    "fullName": "Juan Manuel Pérez García",
    "licenseStatus": "VIGENTE",
    "status": "ACTIVO",
    "version": 1
  }
}
```

---

## 4. Arquitectura Hexagonal

```
Presentation Layer
  └── ForkliftOperatorController          (/api/v1/forklift-operators)

Application Layer
  ├── ForkliftOperatorUseCase             (Puerto IN)
  ├── ForkliftOperatorService             (Implementación + Auditoría)
  ├── CreateForkliftOperatorRequest.java
  ├── UpdateForkliftOperatorRequest.java
  ├── UpdateForkliftOperatorStatusRequest.java
  ├── ForkliftOperatorResponse.java
  ├── ForkliftOperatorAuditResponse.java
  └── ForkliftOperatorMapper.java         (MapStruct)

Domain Layer
  ├── ForkliftOperatorStatus.java         (ACTIVO, INACTIVO)
  ├── LicenseStatus.java                  (VIGENTE, POR_VENCER, VENCIDA)
  ├── ForkliftOperatorRepositoryPort.java (Puerto OUT)
  └── ForkliftOperatorUseCase.java        (Puerto IN)

Infrastructure Layer
  ├── ForkliftOperatorEntity.java         (JPA + BaseVersionedEntity)
  ├── ForkliftOperatorJpaRepository.java  (Spring Data JPA)
  └── ForkliftOperatorPersistenceAdapter.java
```

---

## 5. Reglas de Negocio

1. **Código Auto-generado:** El código operativo `MC-XXX` es generado por el servicio con `count + 1` por organización. El cliente **no puede proporcionarlo ni modificarlo**.
2. **DC-3 Único:** El número de licencia DC-3 debe ser único dentro de la misma organización (`UNIQUE(organization_id, license_number_dc3)`).
3. **fullName Desnormalizado:** El campo `fullName` siempre es recalculado por el servicio como `firstName + lastNamePaternal + lastNameMaternal`.
4. **licenseStatus Computado:** `licenseStatus` es recalculado en cada operación de escritura basándose en `licenseExpirationDate` vs. `TODAY`.
5. **Soft Delete:** Las bajas son lógicas (`is_deleted = true`). El registro permanece en BD para trazabilidad de auditoría.
6. **Turno Desnormalizado:** Al asignar un turno, se copia el `shift.name` a `shift_name` para consultas sin JOIN.

---

## 6. Auditoría Transaccional

Todas las operaciones de escritura generan un registro en `wms.audit_logs` con deltas campo a campo:

| Acción | Trigger |
|---|---|
| `FORKLIFT_OPERATOR_CREATED` | POST exitoso |
| `FORKLIFT_OPERATOR_UPDATED` | PUT exitoso |
| `FORKLIFT_OPERATOR_STATUS_CHANGED` | PATCH /status exitoso |
| `FORKLIFT_OPERATOR_DELETED` | DELETE exitoso |

Cada entrada incluye: `userId`, `action`, `entityType = "FORKLIFT_OPERATOR"`, `entityId`, `beforeState` (snapshot antes del cambio), `afterState` (estado final), y los detalles de campo (`fieldName`, `oldValue`, `newValue`).

---

## 7. Seguridad y Permisos

| Permiso | Roles con acceso |
|---|---|
| `FORKLIFT_OPERATORS_READ` | SUPER_ADMIN, OPERATIONS_MANAGER, WAREHOUSE_SUPERVISOR, SHIFT_LEADER |
| `FORKLIFT_OPERATORS_CREATE` | SUPER_ADMIN, OPERATIONS_MANAGER |
| `FORKLIFT_OPERATORS_UPDATE` | SUPER_ADMIN, OPERATIONS_MANAGER |
| `FORKLIFT_OPERATORS_DELETE` | SUPER_ADMIN, OPERATIONS_MANAGER |
| `FORKLIFT_OPERATORS_STATUS_CHANGE` | SUPER_ADMIN, OPERATIONS_MANAGER, WAREHOUSE_SUPERVISOR |

---

## 8. Migración de Base de Datos

**Archivo:** `V6__forklift_operators_schema.sql`

La migración V6 ejecuta automáticamente al iniciar el BE contra la BD (Flyway):
1. Crea tabla `wms.forklift_operators` con sus FKs a `organizations`, `branches` y `wms_shifts`.
2. Crea 5 índices de rendimiento.
3. Inserta los 5 permisos específicos del módulo.
4. Asigna permisos a los 4 roles administrativos/operativos.

---

## 9. Tests

**Archivo:** `ForkliftOperatorServiceTest.java`

| Test | Escenario |
|---|---|
| `createOperator_success` | Crea MC-001 con código auto-generado |
| `createOperator_organizationNotFound` | Lanza EntityNotFoundException |
| `createOperator_dc3LicenseDuplicate` | Lanza ValidationException por DC-3 duplicado |
| `updateOperator_success` | Actualiza nombre y recalcula fullName |
| `updateOperator_notFound` | Lanza EntityNotFoundException |
| `getOperatorById_success` | Retorna respuesta correcta |
| `getOperatorById_notFound` | Lanza EntityNotFoundException |
| `updateOperatorStatus_toInactivo` | Cambia estatus a INACTIVO |
| `updateOperatorStatus_invalidStatus` | Lanza ValidationException por SUSPENDIDO |
| `deleteOperator_success` | Ejecuta soft delete |
| `deleteOperator_notFound` | Lanza EntityNotFoundException |
| `getOperators_byOrganization` | Retorna lista filtrada |
