# SDD — Frontend: Módulo Gestión de Proveedores (`4Guard_FE_UI`)

> **Módulo:** `suppliers`  
> **Repositorio:** `4Guard_FE_UI` · **Ruta:** `apps/admin-console/src/app/features/admin/suppliers/`  
> **Framework:** Angular 17+ (Standalone Components, Signals Reactivos, Reactive Forms)  
> **Rol / Permiso:** `WMS_ADMIN`, `PURCHASING_MANAGER`, `SUPPLIERS_MANAGE`  
> **Estado:** 🟢 Integrado con Backend Real (`SupplierController.java`)  

---

## 1. Objetivo y Alcance

Permitir el registro, control y monitoreo de los **Proveedores Comerciales y Fabricantes** que abastecen de producto y materia prima a los almacenes de la organización:

1. **Datos Fiscales y Comerciales:** Razón Social, Nombre Comercial, RFC fiscal, Domicilio Fiscal y Moneda de Facturación preferente.
2. **Clasificación y Categorización:** Categorías de mercancía suministrada (Perecederos, Electrónica, Secos, Químicos, Empaque).
3. **Condiciones Comerciales y Puntos de Contacto:** Plazos de crédito, tiempo de entrega promedio (Lead Time en días) y directorio de ejecutivos de cuenta.
4. **Historial de Desempeño y Recepción:** Vinculación con las órdenes de descarga en el andén de recepción (`warehouse-receiving`).

---

## 2. Estructura de Archivos del Módulo

```
apps/admin-console/src/app/features/admin/suppliers/
├── supplier-management/
│   ├── supplier-management.component.ts    ← Componente Standalone reactivo
│   ├── supplier-management.component.html  ← Template Split-View con KPIs
│   └── supplier-management.component.css   ← Estilos con tokens Synexia
├── models/
│   └── supplier.models.ts                  ← Modelos TypeScript homologados con SupplierResponse.java
├── services/
│   └── supplier.service.ts                 ← Servicio HTTP REST hacia /api/v1/suppliers
└── supplier.routes.ts                      ← Rutas del módulo
```

---

## 3. Normativa de Homologación de Componentes (ADR-013)

| Componente | Implementación en este Módulo | Requisitos de Cumplimiento |
|---|---|---|
| **Hero Header** | `.hero-header` | Icono navy 52x52px (`store`), botón badge `.btn-back-admin` hacia `/admin`, eyebrow `CADENA DE SUMINISTRO` en monospace dorado, H1 `Gestión de Proveedores`. |
| **KPI Cards Grid** | `.carriers-kpi-grid` | Total Proveedores, Activos (verde), Con Órdenes Pendientes (dorado), Inactivos (gris). |
| **Directorio Split-View** | `.carriers-directory` (35%) | Buscador reactivo por RFC, Razón Social o Código; chips de filtro por categoría; lista con badges. |
| **Formulario por Secciones** | `.carriers-form` (65%) | Secciones: `DATOS FISCALES`, `CONDICIONES COMERCIALES`, `CONTACTOS Y ATENCIÓN`, `AUDITORÍA`. |
| **Selectores & Dropdowns** | `.form-select` | Selector de moneda de facturación (`MXN`, `USD`, `EUR`) y selector de términos de pago (`CONTADO`, `CREDITO_30`, etc.). |
| **Data Table de Contactos**| `.table-container`, `.data-table` | Tabla de ejecutivos de venta con columnas: Nombre, Cargo, Teléfono, Correo y Acciones. |

---

## 4. Estado Reactivo del Componente (`Signals`)

| Signal | Tipo | Descripción |
|---|---|---|
| `suppliers` | `WritableSignal<Supplier[]>` | Catálogo completo de proveedores |
| `selectedSupplier` | `WritableSignal<Supplier \| null>` | Proveedor activo en edición |
| `searchTerm` | `WritableSignal<string>` | Búsqueda por RFC o Razón Social |
| `categoryFilter` | `WritableSignal<string>` | Filtro por categoría de mercancía |
| `filteredSuppliers`| `ComputedSignal<Supplier[]>` | Lista reactiva de proveedores filtrados |
| `totalSuppliers` | `ComputedSignal<number>` | KPI: Conteo total de proveedores |
| `activeSuppliers`| `ComputedSignal<number>` | KPI: Proveedores activos |
| `isLoading` | `WritableSignal<boolean>` | Indicador de carga de datos |
| `isSaving` | `WritableSignal<boolean>` | Indicador de persistencia |

---

## 5. Modelos de Datos TypeScript (`supplier.models.ts`)

```typescript
export type SupplierStatus = 'ACTIVE' | 'SUSPENDED' | 'INACTIVE';

export interface SupplierContact {
  id?: string;
  name: string;
  department: string;              // 'Ventas', 'Logística', 'Crédito y Cobranza'
  phone: string;
  email: string;
  isPrimary: boolean;
}

export interface Supplier {
  id: string;                      // UUID v4
  organizationId: string;          // UUID v4
  code: string;                    // PRV-001
  legalName: string;               // Razón Social
  commercialName: string;          // Nombre Comercial
  rfc: string;                     // RFC Fiscal
  status: SupplierStatus;
  categories: string[];            // ['PERECEDEROS', 'REFRIGERADOS']
  paymentTerms: string;            // 'CREDITO_30_DIAS'
  defaultCurrency: string;         // 'MXN'
  leadTimeDays: number;            // 3 días promedio de surtido
  address: string;
  city: string;
  state: string;
  postalCode: string;
  contacts: SupplierContact[];
  createdAt: string;
  updatedAt: string;
}

export interface CreateSupplierRequest {
  legalName: string;
  commercialName: string;
  rfc: string;
  categories: string[];
  paymentTerms: string;
  defaultCurrency: string;
  leadTimeDays: number;
  address: string;
  city: string;
  state: string;
  postalCode: string;
  contacts: SupplierContact[];
}
```

---

## 6. Contrato HTTP REST (`SupplierService`)

| Método | Verbo | Endpoint | Descripción |
|---|---|---|---|
| `getSuppliers()` | `GET` | `/api/v1/suppliers` | Obtiene los proveedores de la organización |
| `getSupplierById(id)` | `GET` | `/api/v1/suppliers/{id}` | Recupera el detalle completo y contactos |
| `createSupplier(dto)` | `POST` | `/api/v1/suppliers` | Registra un nuevo proveedor |
| `updateSupplier(id, dto)` | `PUT` | `/api/v1/suppliers/{id}` | Actualiza información comercial y fiscal |
| `updateStatus(id, req)` | `PATCH` | `/api/v1/suppliers/{id}/status` | Alterna estatus activo/suspendido |
| `getSupplierAudit(id)` | `GET` | `/api/v1/suppliers/{id}/audit` | Historial de auditoría diferencial |

---

## 7. Validaciones y Notificaciones

1. **RFC:** Validación obligatoria con formato oficial SAT mexicano.
2. **Validación de Contactos:** Cada proveedor debe contar con al menos un contacto principal (`isPrimary === true`).
3. **Notificaciones:** Mensajes de retroalimentación exclusivos con `ToastService`.
