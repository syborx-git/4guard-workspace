# SDD — Frontend: Módulo Gestión de Usuarios (`4Guard_FE_UI`)

> **Módulo:** `users`  
> **Repositorio:** `4Guard_FE_UI` · **Ruta:** `apps/admin-console/src/app/features/admin/users/`  
> **Framework:** Angular 17+ (Standalone Components, Signals Reactivos, Reactive Forms)  
> **Rol / Permiso:** `SUPER_ADMIN`, `WMS_ADMIN`, `USERS_MANAGE`  
> **Estado:** 🟢 Golden Standard — Integrado con Spring Security y Base de Datos Real  

---

## 1. Objetivo y Alcance

Centralizar la administración del personal con acceso a la plataforma 4GUARD WMS (Administradores, Supervisores de Almacén, Operadores de Mesa de Control, Inspectores de Calidad y Jefes de Seguridad), controlando:

1. **Ciclo de Vida de Cuentas de Usuario:** Creación, edición de perfil, asignación de roles y revocación/suspensión de accesos.
2. **Seguridad y Contraseñas:** Emisión de contraseñas temporales criptográficas con caducidad forzada al primer inicio de sesión y restablecimiento manual de credenciales mediante modal desacoplado.
3. **Control Multi-sucursal:** Asignación de almacenes y sucursales operativas autorizadas por usuario.
4. **Historial y Trazabilidad de Sesiones:** Bitácora de accesos, intentos fallidos y auditoría forense con deltas de cambios en privilegios.

---

## 2. Estructura de Archivos del Módulo

```
apps/admin-console/src/app/features/admin/users/
├── users-list.component.ts           ← Componente principal con Signals y orquestación
├── users-list.component.html         ← Template Split-View con modales integrados
├── users-list.component.css          ← Estilos BEM con tokens Synexia
├── confirm-dialog/                   ← Modal desacoplado de confirmación de revocación
├── set-password-modal/               ← Diálogo seguro de establecimiento de contraseña
└── temp-password-modal/              ← Diálogo generador de contraseña temporal copiable
```

---

## 3. Normativa de Homologación de Componentes (ADR-013)

| Componente | Implementación en este Módulo | Requisitos de Cumplimiento |
|---|---|---|
| **Hero Header** | `.hero-header` | Icono navy 52x52px (`group`), breadcrumb `.btn-back-admin` hacia `/admin`, eyebrow `SEGURIDAD & ACCESOS` en monospace dorado, H1 `Gestión de Usuarios`. |
| **KPI Cards Grid** | `.users-kpi-grid` (4 cards) | Total Usuarios, Activos (verde), Inactivos/Suspendidos (ámbar/rojo), Administradores y Supervisores (morado corporativo). |
| **Directorio Split-View** | `.users-directory` (35%) | Buscador con debounce para nombre, username o correo; selector de estado (`ACTIVE`, `INACTIVE`, `SUSPENDED`); selector de rol del sistema. |
| **Data Table / Directorio**| `.user-item`, `.data-table` | Elementos de lista con avatar circular con iniciales, badges de rol y badge de estatus; tabla de sesiones activas con IP, navegador y botón de revocación. |
| **Modales Desacoplados** | `set-password-modal`, `temp-password-modal` | Contenedores `div` sin etiquetas `<form>` anidadas, botón de copiar al portapapeles con feedback visual y cierre seguro con `escape`. |
| **Formulario de Detalle**| `.users-form` (65%) | Agrupación por leyendas doradas: `DATOS DE IDENTIDAD`, `ROLES Y PRIVILEGIOS`, `ALMACENES ASIGNADOS`, `SEGURIDAD Y AUDITORÍA`. |

---

## 4. Estado Reactivo del Componente (`Signals`)

| Signal | Tipo | Descripción |
|---|---|---|
| `users` | `WritableSignal<UserResponse[]>` | Lista completa de usuarios obtenida de la base de datos |
| `selectedUser` | `WritableSignal<UserResponse \| null>`| Usuario seleccionado en la columna de edición/detalle |
| `filterText` | `WritableSignal<string>` | Búsqueda por texto en nombre, email o username |
| `filterStatus` | `WritableSignal<string>` | Filtro por estatus de cuenta (`ACTIVE`, `SUSPENDED`, etc.) |
| `filterRole` | `WritableSignal<string>` | Filtro por nombre de rol asignado |
| `filteredUsers` | `ComputedSignal<UserResponse[]>`| Lista computada reactivamente combinando filtros |
| `totalUsers` | `ComputedSignal<number>` | KPI: Conteo total de usuarios |
| `kpiActive` | `ComputedSignal<number>` | KPI: Conteo de usuarios activos |
| `kpiInactive` | `ComputedSignal<number>` | KPI: Conteo de usuarios suspendidos/inactivos |
| `kpiAdminSupervisors` | `ComputedSignal<number>` | KPI: Conteo de usuarios con rol administrativo |
| `isLoadingUsers` | `WritableSignal<boolean>` | Indicador de carga de usuarios |

---

## 5. Modelos de Datos TypeScript (`user.models.ts`)

```typescript
export type UserStatus = 'ACTIVE' | 'INACTIVE' | 'SUSPENDED' | 'LOCKED';

export interface UserRoleDto {
  id: string;
  name: string;                    // 'ROLE_ADMIN', 'ROLE_SUPERVISOR', etc.
  label: string;                   // 'Administrador del WMS'
}

export interface UserResponse {
  id: string;                      // UUID v4
  organizationId: string;          // UUID v4
  username: string;
  email: string;
  firstName: string;
  lastName: string;
  fullName: string;
  phone?: string;
  status: UserStatus;
  roles: UserRoleDto[];
  assignedBranchIds: string[];     // UUIDs de sucursales autorizadas
  mustChangePassword: boolean;
  lastLoginAt?: string;            // ISO-8601 UTC
  createdAt: string;
  updatedAt: string;
}

export interface CreateUserRequest {
  username: string;
  email: string;
  firstName: string;
  lastName: string;
  phone?: string;
  roleIds: string[];
  assignedBranchIds: string[];
  temporaryPassword?: string;
  requirePasswordChange: boolean;
}
```

---

## 6. Contrato HTTP REST (`UsersService`)

| Método | Verbo | Endpoint | Descripción |
|---|---|---|---|
| `getUsers()` | `GET` | `/api/v1/users` | Lista todos los usuarios de la organización |
| `getUserById(id)` | `GET` | `/api/v1/users/{id}` | Obtiene el perfil completo y roles de un usuario |
| `createUser(dto)` | `POST` | `/api/v1/users` | Da de alta una nueva cuenta con envío de credenciales |
| `updateUser(id, dto)` | `PUT` | `/api/v1/users/{id}` | Actualiza datos de perfil y almacenes asignados |
| `updateStatus(id, req)` | `PATCH` | `/api/v1/users/{id}/status` | Suspende, activa o bloquea una cuenta |
| `generateTempPassword(id)`| `POST` | `/api/v1/users/{id}/temp-password` | Genera contraseña temporal de un solo uso |
| `setPassword(id, req)` | `POST` | `/api/v1/users/{id}/set-password` | Establece manualmente la contraseña del usuario |
| `revokeSession(id, sessId)`| `DELETE`| `/api/v1/users/{id}/sessions/{sessId}`| Cierra la sesión activa de un dispositivo remoto |
| `getUserAudit(id)` | `GET` | `/api/v1/users/{id}/audit` | Obtiene la bitácora de cambios con deltas |

---

## 7. Manejo de Errores y Seguridad

1. **Protección de Auto-suspensión:** La interfaz deshabilita los botones de suspensión y revocación de privilegios sobre la cuenta del propio usuario en sesión.
2. **Generación Segura de Contraseña Temporal:** Las contraseñas temporales se muestran una única vez en un modal con botón de copiado rápido y aviso de caducidad (24 horas).
3. **Notificaciones:** Retroalimentación mediante `ToastService` con prevención de mensajes técnicos crudos.
