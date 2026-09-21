# ADR-004: Guard RBAC Centralizado basado en Rutas

- **Estado:** Aceptado
- **Fecha:** 2026-07-20
- **Autores:** Equipo 4Guard WMS Frontend
- **Módulos Afectados:** Router & Security (`app.routes.ts`, `auth.state.ts`)

---

## Contexto y Problema

4GUARD WMS cuenta con una matriz estricta de Roles y Permisos (RBAC). El acceso no autorizado a módulos de administración o auditoría debe bloquearse antes de renderizar la pantalla.

## Opciones Evaluadas

1. **Validación dentro del Componente:** Cada componente en `ngOnInit()` verifica los permisos del usuario.  
   *Desventaja:* Dispersión de lógica de seguridad y riesgo de descuidos al crear nuevas pantallas.
2. **`rbacGuard` a nivel de Router Angular:** Un guard funcional global que lee el metadato `data: { module: '...' }` de la ruta.

## Decisión Tomada

Se implementa `rbacGuard` centralizado en el router de Angular. Las rutas protegidas declaran explícitamente el módulo requerido en la propiedad `data`.

```typescript
{
  path: 'user-activity',
  canActivate: [rbacGuard],
  data: { module: 'user-activity' },
  loadChildren: () => import('./user-activity.routes')
}
```

## Consecuencias

### Positivas
- Seguridad declarativa y centralizada.
- Componentes 100% limpios de lógica de autorización.
- Redirección automática a `/dashboard` o `/login` ante intentos no autorizados.

### Negativas / Compromisos
- Requiere mantener sincronizada la lista de módulos en `WmsModule` / `RoleEnum`.
