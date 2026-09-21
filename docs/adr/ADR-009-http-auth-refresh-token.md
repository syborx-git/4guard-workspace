# ADR-009: Intercepción HTTP Transparente para Refresh Token JWT

- **Estado:** Aceptado
- **Fecha:** 2026-07-30
- **Autores:** Equipo 4Guard WMS Frontend
- **Módulos Afectados:** Autenticación & HTTP (`auth.interceptor.ts`, `auth.state.ts`)

---

## Contexto y Problema

Los tokens JWT de acceso expiran periódicamente por razones de seguridad. Sin un mecanismo de renovación automática, el usuario sufría desconexiones repentinas en medio de una operación crítica en el almacén (ej. durante un conteo de inventario).

## Opciones Evaluadas

1. **Redirección Inmediata a Login en 401:** Redirigir al usuario al formulario de login ante cualquier error 401.  
   *Desventaja:* Mala experiencia de usuario; pérdida de datos en formularios no guardados.
2. **Renovación Transparente con Interceptor Functional (`authInterceptor`):**
   - Interceptar respuestas HTTP 401.
   - Solicitar un nuevo Token de Acceso usando el Refresh Token almacenado en `localStorage` / cookie segura via `POST /api/v1/auth/refresh`.
   - Reintentar automáticamente la petición original que falló.

## Decisión Tomada

Se implementa la **Renovación Transparente y Redirección por Revocación mediante Interceptor Functional de Angular 17+** (`authInterceptor` y `jwtInterceptor`).

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // Adjunta Header Authorization: Bearer <token>
  // Captura 401: Renueva token y reintenta la petición original.
  // Captura 403 / 401 de revocación: Limpia sesión y redirige a /login.
};
```

## Consecuencias

### Positivas
- Experiencia de usuario ininterrumpida mientras la sesión sea válida.
- Cero pérdida de estado o borradores de formularios.
- Centralización de credenciales `Bearer` en interceptores funcionales.
- Expulsión automática al `/login` tanto ante `401 Unauthorized` como ante `403 Forbidden` cuando una sesión es revocada administrativamente por el Backend.

### Negativas / Compromisos
- Requiere cola de peticiones si ocurren múltiples llamadas concurrentes mientras se renueva el token.
