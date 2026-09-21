# ADR-007: Directiva Estricta Cero Mocks y Consumo BD Backend

- **Estado:** Aceptado
- **Fecha:** 2026-07-28
- **Autores:** Equipo 4Guard WMS Frontend
- **Módulos Afectados:** Data & Services Layer (`admin-console/src/app/features/`)

---

## Contexto y Problema

Durante la fase de prototipado es común usar archivos de datos simulados (`MOCK_*`). Sin embargo, mantener fallbacks silenciosos a datos mock en producción oculta errores de red, disfraza fallas en los contratos de la API y genera inconsistencias en auditoría.

## Opciones Evaluadas

1. **Fallback Silencioso a Mocks:** Si el servicio HTTP falla, retornar arreglos ficticios en memoria.  
   *Desventaja:* Falsa sensación de funcionamiento; oculta fallos del Backend en producción.
2. **Cero Mocks en Producción (BD Backend Obligatoria):** Eliminar todos los arreglos estáticos en servicios. Si la API responde vacía o falla, el frontend muestra el estado vacío real (`empty state`) y emite una notificación de error con `ToastService.error()`.

## Decisión Tomada

Se establece la **Directiva Estricta de Cero Mocks**. Todos los servicios de la aplicación `admin-console` consumen estrictamente los endpoints HTTP de la base de datos real de `4Guard_BEAPI`.

```typescript
// Patrón obligatorio de manejo de datos en servicios
return this.http.get<ApiResponse<Dto[]>>(this.API_URL).pipe(
  map(res => res.data ?? []),
  catchError((err: HttpErrorResponse) => {
    this.toast.error(err.error?.message ?? 'Error al conectar con la base de datos');
    return of([]); // NUNCA retornar MOCK_DATA
  })
);
```

## Consecuencias

### Positivas
- Garantía de integridad de datos 100% auditable.
- Detección inmediata de inconsistencias en endpoints o caídas de servidor.
- Cero código muerto de datos ficticios en producción.

### Negativas / Compromisos
- La aplicación requiere que la API de Spring Boot o el entorno de staging esté disponible para pruebas de integración.
