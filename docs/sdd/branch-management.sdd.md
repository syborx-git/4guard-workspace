# SDD — Frontend: Módulo Gestión de Sucursales y Centros de Distribución (`4Guard_FE_UI`)

> **Módulo:** `branches`  
> **Repositorio:** `4Guard_FE_UI` · **Ruta:** `apps/admin-console/src/app/features/admin/branches/`  
> **Framework:** Angular 17+ (Standalone Components, Signals Reactivos, Reactive Forms)  
> **Rol / Permiso:** `SUPER_ADMIN`, `WMS_ADMIN`, `BRANCHES_MANAGE`  
> **Estado:** 🟢 Integrado con Backend Real (`BranchController.java`)  

---

## 1. Objetivo y Alcance

Administrar el catálogo de **Sucursales, Plantas y Centros de Distribución (CEDIS)** físicos que integran la red logística de la organización:

1. **Datos Físicos y Coordenadas Geográficas:** Registro de dirección fiscal/física, código postal, latitud/longitud para geocercas y cálculo de rutas de transporte.
2. **Capacidad Instalada:** Control de superficie en m², número de andenes de carga/descarga, capacidad máxima de tarimas (posiciones pallet) y zonas refrigeradas.
3. **Parámetros de Operación Local:** Configuración de zona horaria (Timezone), prefijo de folios de remisión y vinculación con almacenes lógicos.
4. **Estado FSM:** Transiciones controladas entre `ACTIVE`, `MAINTENANCE` e `INACTIVE`.

---

## 2. Estructura de Archivos del Módulo

```
apps/admin-console/src/app/features/admin/branches/
├── branch-management/
│   ├── branch-management.component.ts    ← Componente Standalone reactivo
│   ├── branch-management.component.html  ← Template Split-View con mapa/coordenadas
│   └── branch-management.component.css   ← Estilos homologados con tokens Synexia
├── branches.routes.ts                    ← Configuración de enrutamiento
└── models/
    └── branch.models.ts                  ← Modelos TypeScript homologados con BranchResponse.java
```

---

## 3. Normativa de Homologación de Componentes (ADR-013)

| Componente | Implementación en este Módulo | Requisitos de Cumplimiento |
|---|---|---|
| **Hero Header** | `.hero-header` | Icono navy 52x52px (`domain`), botón badge `.btn-back-admin` hacia `/admin`, eyebrow `RED LOGÍSTICA` en monospace dorado, H1 `Gestión de Sucursales y CEDIS`. |
| **KPI Cards Grid** | `.carriers-kpi-grid` | Total Sucursales, Operativas (verde), En Mantenimiento (ámbar), Andenes Totales Disponibles (dorado). |
| **Directorio Split-View** | `.carriers-directory` (35%) | Tarjetas con código de sucursal en chip mono, nombre de ciudad/estado y badge de estatus. |
| **Formulario por Secciones** | `.carriers-form` (65%) | Agrupado en: `DATOS GENERALES`, `UBICACIÓN GEOGRÁFICA`, `CAPACIDAD Y ANDENES`, `AUDITORÍA`. |
| **Selectores & Dropdowns** | `.form-select` | Selector de tipo de centro logístico (`CEDIS_PRINCIPAL`, `BODEGA_CROSSDOCK`, `PLANTA_PRODUCCION`) y selector de zona horaria IANA. |
| **Diálogos de Confirmación** | `<fg-confirm-dialog>` | Modal de confirmación para suspensiones o puesta en mantenimiento de sucursal. |

---

## 4. Estado Reactivo del Componente (`Signals`)

| Signal | Tipo | Descripción |
|---|---|---|
| `branches` | `WritableSignal<Branch[]>` | Catálogo completo de sucursales obtenido del servidor |
| `selectedBranch` | `WritableSignal<Branch \| null>` | Sucursal seleccionada para edición |
| `searchQuery` | `WritableSignal<string>` | Búsqueda por código, nombre o ciudad |
| `statusFilter` | `WritableSignal<string>` | Filtro por estatus operativo |
| `filteredBranches` | `ComputedSignal<Branch[]>` | Lista reactiva de sucursales filtradas |
| `totalBranches` | `ComputedSignal<number>` | KPI: Total de centros registrados |
| `activeBranches` | `ComputedSignal<number>` | KPI: Centros con estatus activo |
| `isLoading` | `WritableSignal<boolean>` | Indicador de carga de datos |
| `isSaving` | `WritableSignal<boolean>` | Indicador de guardado |

---

## 5. Modelos de Datos TypeScript (`branch.models.ts`)

```typescript
export type BranchType = 'MAIN_CEDIS' | 'REGIONAL_WAREHOUSE' | 'CROSS_DOCK' | 'PRODUCTION_PLANT';
export type BranchStatus = 'ACTIVE' | 'MAINTENANCE' | 'INACTIVE';

export interface Branch {
  id: string;                      // UUID v4
  organizationId: string;          // UUID v4
  code: string;                    // SUC-01
  name: string;                    // 'CEDIS Central Guadalajara'
  type: BranchType;
  status: BranchStatus;
  street: string;
  externalNumber: string;
  internalNumber?: string;
  neighborhood: string;
  city: string;
  state: string;
  postalCode: string;
  country: string;
  latitude?: number;
  longitude?: number;
  timezone: string;                // 'America/Mexico_City'
  totalDocks: number;
  totalPalletCapacity: number;
  coveredAreaM2: number;
  createdAt: string;
  updatedAt: string;
}

export interface CreateBranchRequest {
  code: string;
  name: string;
  type: BranchType;
  street: string;
  externalNumber: string;
  internalNumber?: string;
  neighborhood: string;
  city: string;
  state: string;
  postalCode: string;
  country: string;
  latitude?: number;
  longitude?: number;
  timezone: string;
  totalDocks: number;
  totalPalletCapacity: number;
  coveredAreaM2: number;
}
```

---

## 6. Contrato HTTP REST (`BranchService`)

| Método | Verbo | Endpoint | Descripción |
|---|---|---|---|
| `getBranches()` | `GET` | `/api/v1/branches` | Lista todas las sucursales de la organización |
| `getBranchById(id)` | `GET` | `/api/v1/branches/{id}` | Obtiene los detalles completos de la sucursal |
| `createBranch(dto)` | `POST` | `/api/v1/branches` | Registra una nueva sucursal con validación de código único |
| `updateBranch(id, dto)` | `PUT` | `/api/v1/branches/{id}` | Actualiza datos de dirección y capacidad física |
| `toggleStatus(id, req)` | `PATCH` | `/api/v1/branches/{id}/status` | Modifica el estatus operativo |
| `getBranchAudit(id)` | `GET` | `/api/v1/branches/{id}/audit` | Historial de deltas de auditoría |

---

## 7. Validaciones y Notificaciones

1. **Código Postal Mexicano:** Validador RegExp `^[0-9]{5}$`.
2. **Capacidades Positivas:** Validación de `totalDocks >= 1`, `totalPalletCapacity >= 0`.
3. **Notificaciones:** Retroalimentación mediante `ToastService` con soporte para mensajes de validación de backend.
