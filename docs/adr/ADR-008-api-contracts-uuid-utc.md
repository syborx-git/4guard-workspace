# ADR-008: Estándar de Contratos API (`ApiResponse<T>`, UUID, ISO-8601 UTC)

- **Estado:** Aceptado
- **Fecha:** 2026-07-29
- **Autores:** Equipo 4Guard WMS (Frontend & Backend)
- **Módulos Afectados:** Intercambio de datos HTTP (`docs/api/contracts.md`, `@4guard/shared-core`)

---

## Contexto y Problema

Inconsistencias en los formatos de respuesta HTTP, tipos de identificadores (enteros autoincrementales vs. UUIDs) y zonas horarias de fechas causaban errores de parseo y colisiones en bases de datos multi-tenant.

## Opciones Evaluadas

1. **Respuestas HTTP heterogéneas / IDs numéricos:** Cada controller devuelve estructuras distintas.  
   *Desventaja:* Difícil estandarización de interceptores y riesgo de adivinación de IDs (seguridad).
2. **Estándar Unificado Enterprise:**
   - Wrapper genérico: `ApiResponse<T>` con `success`, `message` y `data`.
   - Identificadores únicos: `UUID` v4 (representados como `string` en TypeScript).
   - Fechas y horas: Formato ISO-8601 UTC completo (`yyyy-MM-ddTHH:mm:ssZ`).

## Decisión Tomada

Se adopta el **Estándar Unificado Enterprise** para todas las comunicaciones API entre `admin-console` y `4Guard_BEAPI`.

```typescript
export interface ApiResponse<T> {
  success: boolean;
  message: string;
  data: T;
  timestamp?: string; // ISO-8601 UTC
}
```

## Consecuencias

### Positivas
- Trazabilidad y parseo uniforme de respuestas en todos los servicios de Angular.
- Seguridad mejorada: los UUIDs no permiten enumeración directa de recursos.
- Cero desfases por zonas horarias entre cliente web, servidor Java y base de datos PostgreSQL.

### Negativas / Compromisos
- Los query params de fecha deben formatearse siempre explícitamente en UTC antes de enviarse.
