# SDD — Frontend: Módulo Ubicaciones Físicas y Topología Cromática (`4Guard_FE_UI`)

> **Módulo:** `layout`  
> **HU:** HU-127 — Topología y Gestión de Ubicaciones Físicas de Almacén  
> **Repositorio:** `4Guard_FE_UI` · **Ruta:** `apps/admin-console/src/app/features/layout/`  
> **Framework:** Angular 17+ (Standalone Components, Signals Reactivos, Reactive Forms)  
> **Rol / Permiso:** `WMS_ADMIN`, `WAREHOUSE_SUPERVISOR`, `LOCATIONS_MANAGE`  
> **Estado:** 🟢 Golden Standard de Referencia — Topología Cromática FSM Conectada  

---

## 1. Objetivo y Alcance

Permitir la modelación visual, volumétrica y operativa de la infraestructura física del almacén:

1. **Árbol Jerárquico de Ubicaciones:** Navegación anidada en 5 niveles: Zona de Almacenamiento ➔ Pasillo (Aisle) ➔ Bahía (Bay) ➔ Nivel (Shelf/Tier) ➔ Posición (Bin/Slot).
2. **Topología Cromática FSM en Cuadrícula 2D:** Visualización gráfica en tiempo real de las celdas del almacén con código de color según su estado operativo (Disponible, Activo/Ocupado, En Proceso, Bloqueado, Mantenimiento, Inactivo).
3. **Control de Volumetría y Capacidad:** Dimensiones (Largo, Ancho, Alto en cm), Peso Máximo Soportado (kg), tipo de almacenamiento (Rack estándar, Piso, Drive-in, Cantilever) y porcentaje de ocupación con barra reactiva.
4. **FSM y Bloqueos de Seguridad:** Mecanismo para bloquear posiciones con motivos auditados (por derrame, cuarentena o inventario cíclico).

---

## 2. Estructura de Archivos del Módulo

```
apps/admin-console/src/app/features/layout/
├── layout-management/
│   ├── layout-management.component.ts    ← Árbol jerárquico y grid cromático
│   ├── layout-management.component.html  ← Template Tree Explorer + Editor + Grid
│   └── layout-management.component.css   ← Estilos con tokens Synexia
├── layout.routes.ts                      ← Rutas del módulo
├── models/
│   └── location.models.ts                ← Modelos TypeScript homologados con LocationResponse.java
└── services/
    └── location.service.ts               ← Servicio HTTP hacia /api/v1/locations y /sections
```

---

## 3. Normativa de Homologación de Componentes (ADR-013)

| Componente | Implementación en este Módulo | Requisitos de Cumplimiento |
|---|---|---|
| **Hero Header** | `.hero-header` | Icono navy 52x52px (`warehouse`), botón badge `.btn-back-admin` hacia `/admin`, eyebrow `ESTRUCTURA DE ALMACÉN` en monospace dorado, H1 `Ubicaciones Físicas y Topología`. |
| **KPI Cards Grid** | `.carriers-kpi-grid` | Total Ubicaciones, Activas/Disponibles (verde), Bloqueadas (rojo), En Mantenimiento (ámbar). |
| **Explorador en Árbol** | `.lm-tree-explorer` (35% / 340px) | Árbol con nodos colapsables/expandibles, iconos dorados para zonas y dots de estatus FSM en hojas. |
| **Cuadrícula Topológica** | `.topology-grid` | Grid interactivo con celdas cuadradas, zoom, tooltip con código y mercancía contenida, y color FSM. |
| **Panel de Ocupación** | `.lm-occupancy-panel` | Barra con umbrales semánticos (<80% verde, 80-95% ámbar, >95% rojo) con cálculo de m³ y kg. |
| **Selectores & Dropdowns** | `.form-select` | Selectores para zona, tipo de rack y estado FSM con borde dorado. |
| **Diálogo de Bloqueo** | `<fg-confirm-dialog>` | Modal para registrar bloqueo de posición con selección obligatoria de causa de cuarentena. |

---

## 4. Estado Reactivo del Componente (`Signals`)

| Signal | Tipo | Descripción |
|---|---|---|
| `zones` | `WritableSignal<WarehouseZone[]>` | Árbol estructurado de zonas y pasillos |
| `selectedLocation` | `WritableSignal<Location \| null>` | Ubicación actualmente en inspección |
| `viewMode` | `WritableSignal<'TREE' \| 'GRID'>` | Conmutador entre vista de árbol y mapa cromático |
| `occupancyRate` | `ComputedSignal<number>` | Porcentaje de ocupación volumétrica del nodo seleccionado |
| `totalLocations` | `ComputedSignal<number>` | KPI: Conteo total de posiciones |
| `availableLocations`| `ComputedSignal<number>` | KPI: Posiciones listas para recibir carga |
| `blockedLocations` | `ComputedSignal<number>` | KPI: Posiciones bloqueadas o en cuarentena |

---

## 5. Modelos de Datos TypeScript (`location.models.ts`)

```typescript
export type LocationStatus = 'AVAILABLE' | 'OCCUPIED' | 'IN_PROCESS' | 'BLOCKED' | 'MAINTENANCE' | 'INACTIVE';
export type StorageType = 'STANDARD_RACK' | 'FLOOR_LANE' | 'DRIVE_IN' | 'CANTILEVER' | 'COLD_ROOM';

export interface Location {
  id: string;                      // UUID v4
  organizationId: string;
  branchId: string;
  code: string;                    // 'A-01-02-C-1' (Zona-Pasillo-Bahía-Nivel-Posición)
  zone: string;                    // 'ZONA A'
  aisle: string;                   // '01'
  bay: string;                     // '02'
  level: string;                   // 'C'
  position: string;                // '1'
  type: StorageType;
  status: LocationStatus;
  maxWeightKg: number;
  currentWeightKg: number;
  maxVolumeM3: number;
  currentVolumeM3: number;
  isBlocked: boolean;
  blockReason?: string;
  createdAt: string;
  updatedAt: string;
}

export interface CreateLocationRequest {
  branchId: string;
  zone: string;
  aisle: string;
  bay: string;
  level: string;
  position: string;
  type: StorageType;
  maxWeightKg: number;
  maxVolumeM3: number;
}
```

---

## 6. Contrato HTTP REST (`LocationService`)

| Método | Verbo | Endpoint | Descripción |
|---|---|---|---|
| `getLocationsTree()` | `GET` | `/api/v1/locations/tree` | Recupera el árbol jerárquico de zonas y bahías |
| `getGridByZone(zId)` | `GET` | `/api/v1/locations/grid?zone={id}` | Obtiene la matriz 2D para la topología cromática |
| `createLocation(dto)`| `POST` | `/api/v1/locations` | Da de alta una nueva posición en el rack |
| `updateLocation(id)` | `PUT` | `/api/v1/locations/{id}` | Modifica dimensiones y capacidades |
| `blockLocation(id, r)`| `POST` | `/api/v1/locations/{id}/block` | Bloquea la posición por motivo de seguridad |
| `unblockLocation(id)`| `POST` | `/api/v1/locations/{id}/unblock` | Desbloquea la posición y la regresa a AVAILABLE |
| `getLocationAudit(id)`| `GET` | `/api/v1/locations/{id}/audit` | Trazabilidad de movimientos y bloqueos |

---

## 7. Validaciones y Notificaciones

1. **Unicidad de Código de Ubicación:** Validación reactiva que impide registrar dos posiciones con la misma nomenclatura física dentro de la misma sucursal.
2. **Pesos y Dimensiones:** Validación de que `maxWeightKg > 0` y `maxVolumeM3 > 0`.
3. **Notificaciones:** Mensajes emitidos mediante `ToastService`.
