# SDD: Cambio de Almacén (Traspasos Internos) — Backend

> **Módulo:** Operación y Logística → Movimientos de Almacén → Cambio de Almacén  
> **HU:** HU-151 — Cambio de Almacén y Reubicaciones  
> **Versión:** 1.0.0  
> **Estado:** 🏁 IMPLEMENTADO / LISTO PARA PR  
> **Fecha:** 2026-08-19  
> **Patrón:** Hexagonal Architecture + Spec-Driven Development (SDD)  

---

## 1. Descripción del Módulo

El submódulo **Cambio de Almacén (Traspasos Internos)** gestiona el movimiento de tarimas/UAs entre ubicaciones físicas del almacén:
1. **Reglas de Negocio:**
   - La bahía origen debe contener inventario activo (`wms.inventory_items`).
   - La bahía destino no debe ser igual al origen.
   - Requiere motivo de movimiento catalogado (`OPT_ESPACIO`, `REUB_OPERATIVA`, `LIB_BAHIA`, `CONSOLIDACION`, `SOL_CLIENTE`, `INCIDENCIA`, `OTRO`).
2. **Impacto en Inventario:**
   - Actualiza `wms.inventory_items.location_id` a la bahía destino.
   - Genera registros en `wms.inventory_movements` con tipo `TRANSFER`.
3. **Cancelación:**
   - Revoca el movimiento con autorización de Administrador y reubica las tarimas a la bahía origen.

---

## 2. Modelo de Base de Datos

### Tabla: `wms.warehouse_transfers`

| Campo | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `id` | UUID | PK, NOT NULL | Identificador único |
| `organization_id` | UUID | FK `wms.organizations.id`, NOT NULL | Organización |
| `branch_id` | UUID | FK `wms.branches.id`, NOT NULL | Sucursal |
| `folio` | VARCHAR(30) | NOT NULL, UNIQUE | Consecutivo formato `CAM-YYYY-XXXXXX` |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `COMPLETED` | `DRAFT`, `CONFIRMED`, `COMPLETED`, `CANCELLED` |
| `origin_location_id` | UUID | FK `wms.locations.id`, NOT NULL | Bahía origen |
| `destination_location_id` | UUID | FK `wms.locations.id`, NOT NULL | Bahía destino |
| `forklift_operator_id` | UUID | FK `wms.forklift_operators.id`, NULLABLE | Montacarguista asignado |
| `reason_code` | VARCHAR(30) | NOT NULL | Código de motivo |
| `reason_label` | VARCHAR(100) | NULLABLE | Etiqueta descriptiva |
| `observations` | TEXT | NULLABLE | Observaciones |
| `total_pallets` | INT | NOT NULL, DEFAULT 0 | Total tarimas trasladadas |
| `total_pieces` | NUMERIC(12,2) | NOT NULL, DEFAULT 0 | Total piezas |
| `distinct_skus` | INT | NOT NULL, DEFAULT 0 | SKUs distintos |
| `cancelled_at` | TIMESTAMPTZ | NULLABLE | Fecha de cancelación |
| `cancellation_reason` | TEXT | NULLABLE | Motivo de cancelación |
| `cancelled_by` | VARCHAR(100) | NULLABLE | Administrador que canceló |
| `version` | BIGINT | NOT NULL, DEFAULT 1 | Optimistic locking |
| `created_at` / `updated_at` | TIMESTAMPTZ | AUTO | Timestamps |

---

## 3. Endpoints REST

**Base path:** `/api/v1/warehouse-transfers`

| Método | Ruta | Permiso | Descripción |
|---|---|---|---|
| `POST` | `/` | `WAREHOUSE_MOVEMENTS_CREATE` | Registrar cambio de almacén |
| `GET` | `/{id}` | `WAREHOUSE_MOVEMENTS_READ` | Detalle completo de traspaso |
| `GET` | `/` | `WAREHOUSE_MOVEMENTS_READ` | Listar traspasos con filtros |
| `POST` | `/{id}/cancel` | `WAREHOUSE_MOVEMENTS_CANCEL` | Cancelar traspaso |
| `GET` | `/{id}/audit` | `WAREHOUSE_MOVEMENTS_READ` | Consultar auditoría |
