# ADR-006: Angular Signals como Estándar de Gestión de Estado Reactivo

- **Estado:** Aceptado
- **Fecha:** 2026-07-25
- **Autores:** Equipo 4Guard WMS Frontend
- **Módulos Afectados:** State Management (`apps/admin-console`, `libs/shared-core`)

---

## Contexto y Problema

Angular 17+ introduce la Signal API como mecanismo primitivo de reactividad granular. Tradicionalmente se usaba RxJS `BehaviorSubject` o NgRx para manejar estado local y global.

## Opciones Evaluadas

1. **NgRx Store / Redux:** Arquitectura Redux completa (actions, reducers, effects).  
   *Desventaja:* Boilerplate masivo y sobreingeniería para la mayoría de pantallas CRUD.
2. **RxJS BehaviorSubjects:** Manejo manual de suscripciones e inmunidad a memory leaks mediante `pipe(takeUntil)`.  
   *Desventaja:* Gestión de desuscripciones verbosa y ChangeDetection `OnPush` menos eficiente.
3. **Angular Signals (`signal`, `computed`, `toSignal`):** Primitivos de reactividad fina integrados en Angular 17.

## Decisión Tomada

Se adopta **Angular Signals** como el estándar obligatorio para la gestión de estado en 4GUARD WMS.
- Estado mutable: `signal<T>(initialValue)`
- Estado derivado / calculados: `computed(() => ...)`
- Inyección limpia con `inject(Service)`

```typescript
export class UserActivityService {
  private readonly _allEvents = signal<UserActivityEvent[]>([]);
  readonly filteredEvents = computed(() => {
    // Cálculo reactivo automático sin suscripciones manuales
    return this._allEvents().filter(...);
  });
}
```

## Consecuencias

### Positivas
- Detección de cambios hiper-eficiente sin recorrer todo el árbol de componentes.
- Cero memory leaks causados por suscripciones RxJS no cerradas en plantillas.
- Código hasta 40% más corto y legible.

### Negativas / Compromisos
- Interoperabilidad requerida: se usa `toSignal()` o `toObservable()` cuando se interactúa con `HttpClient` de RxJS.
