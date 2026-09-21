# SDD — Frontend: Módulo Roles y Permisos RBAC (`4Guard_FE_UI`)

> **Módulo:** `roles`  
> **Repositorio:** `4Guard_FE_UI` · **Ruta:** `apps/admin-console/src/app/features/admin/roles/`  
> **Framework:** Angular 17+ (Standalone Components, Signals Reactivos, Reactive Forms)  
> **Rol / Permiso:** `SUPER_ADMIN`, `ROLES_MANAGE`  
> **Estado:** 🟢 Integrado con Spring Security y Base de Datos Real  

---

## 1. Objetivo y Alcance

Garantizar la administración del modelo de **Control de Acceso Basado en Roles (RBAC)** de 4GUARD WMS, permitiendo a los administradores definir perfiles de acceso con permisos atómicos por módulo:

1. **Gestión de Roles Corporativos:** Creación y modificación de roles estándar y personalizados (ej. `ROLE_ADMIN`, `ROLE_SUPERVISOR`, `ROLE_OPERATOR`, `ROLE_AUDITOR`).
2. **Matriz Granular de Permisos:** Agrupación jerárquica de permisos por dominio funcional (Inventario, Recepción, Salidas, Traspasos, Catálogos Maestros, Configuración del Sistema).
3. **Protección de Roles del Sistema:** Bloqueo de eliminación y modificación de roles inmutables del núcleo (`SUPER_ADMIN`).
4. **Asignación Masiva y Auditoría:** Consulta de usuarios asignados al rol e historial de mutaciones en los claims de seguridad.

---

## 2. Estructura de Archivos del Módulo

```
apps/admin-console/src/app/features/admin/roles/
├── role-management/
│   ├── role-management.component.ts    ← Componente Standalone reactivo
│   ├── role-management.component.html  ← Template Split-View con acordeón de permisos
│   └── role-management.component.css   ← Estilos con tokens Synexia
├── models/
│   └── role.models.ts                  ← Modelos TypeScript homologados con RoleResponse.java
└── roles.routes.ts                     ← Enrutamiento del módulo
```

---

## 3. Normativa de Homologación de Componentes (ADR-013)

| Componente | Implementación en este Módulo | Requisitos de Cumplimiento |
|---|---|---|
| **Hero Header** | `.hero-header` | Icono navy 52x52px (`admin_panel_settings`), botón badge `.btn-back-admin` hacia `/admin`, eyebrow `SEGURIDAD & CONTROL` en monospace dorado, H1 `Roles y Permisos del Sistema`. |
| **KPI Cards Grid** | `.carriers-kpi-grid` | Total de Roles, Roles Personalizados (dorado), Permisos Totales Disponibles (azul), Usuarios con Rol Asignado (verde). |
| **Directorio Split-View** | `.carriers-directory` (35%) | Lista de roles con chip de tipo (`SISTEMA` / `PERSONALIZADO`), conteo de usuarios y badge de estatus. |
| **Matriz de Permisos (Acordeón)** | `.permissions-matrix` (65%) | Acordeones expandibles por módulo con casillas de verificación (Checkbox) para acciones `READ`, `CREATE`, `UPDATE`, `DELETE`, `EXECUTE`. |
| **Selector de Todo / Nada** | `.toggle-all-module` | Botón rápido para marcar o desmarcar todos los permisos de un módulo con un solo clic. |
| **Diálogo de Confirmación** | `<fg-confirm-dialog>` | Modal de advertencia ante eliminación de roles asignados a usuarios activos. |

---

## 4. Estado Reactivo del Componente (`Signals`)

| Signal | Tipo | Descripción |
|---|---|---|
| `roles` | `WritableSignal<Role[]>` | Lista completa de roles de la organización |
| `selectedRole` | `WritableSignal<Role \| null>` | Rol seleccionado en edición |
| `allPermissions` | `WritableSignal<PermissionGroup[]>`| Árbol completo de permisos agrupados por módulo |
| `selectedPermissionIds`| `WritableSignal<Set<string>>` | Conjunto reactivo de IDs de permisos asignados al rol |
| `searchTerm` | `WritableSignal<string>` | Búsqueda por nombre de rol |
| `totalRoles` | `ComputedSignal<number>` | KPI: Conteo total de roles |
| `isLoading` | `WritableSignal<boolean>` | Indicador de carga |
| `isSaving` | `WritableSignal<boolean>` | Indicador de persistencia |

---

## 5. Modelos de Datos TypeScript (`role.models.ts`)

```typescript
export interface Permission {
  id: string;                      // UUID v4
  code: string;                    // 'INVENTORY_VIEW', 'RECEIVING_CONFIRM'
  name: string;                    // 'Ver Inventario'
  description: string;
  module: string;                  // 'INVENTORY', 'RECEIVING', etc.
  action: 'READ' | 'CREATE' | 'UPDATE' | 'DELETE' | 'EXECUTE';
}

export interface PermissionGroup {
  module: string;
  moduleLabel: string;
  permissions: Permission[];
}

export interface Role {
  id: string;                      // UUID v4
  organizationId: string;
  code: string;                    // 'ROLE_LOGISTICS_COORDINATOR'
  name: string;                    // 'Coordinador Logístico'
  description: string;
  isSystemRole: boolean;           // Inmutable si true
  permissions: Permission[];
  assignedUsersCount: number;
  createdAt: string;
  updatedAt: string;
}

export interface CreateRoleRequest {
  code: string;
  name: string;
  description: string;
  permissionIds: string[];
}
```

---

## 6. Contrato HTTP REST (`RoleService`)

| Método | Verbo | Endpoint | Descripción |
|---|---|---|---|
| `getRoles()` | `GET` | `/api/v1/roles` | Lista todos los roles de la organización |
| `getRoleById(id)` | `GET` | `/api/v1/roles/{id}` | Obtiene el detalle de un rol y sus permisos |
| `getPermissionsTree()` | `GET` | `/api/v1/permissions` | Obtiene el catálogo estructurado de permisos |
| `createRole(dto)` | `POST` | `/api/v1/roles` | Registra un nuevo rol personalizado |
| `updateRole(id, dto)` | `PUT` | `/api/v1/roles/{id}` | Actualiza metadatos y permisos asignados |
| `deleteRole(id)` | `DELETE` | `/api/v1/roles/{id}` | Elimina un rol no perteneciente al sistema |
| `getRoleAudit(id)` | `GET` | `/api/v1/roles/{id}/audit` | Bitácora de modificaciones en privilegios |

---

## 7. Validaciones y Notificaciones

1. **Inmutabilidad de `SUPER_ADMIN`:** El frontend bloquea los botones de edición de código y eliminación si `isSystemRole === true`.
2. **Notificaciones:** Feedback exclusivo mediante `ToastService`.
