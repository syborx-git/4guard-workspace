# ADR-016: Despacho Multi-Producto, Manifiesto Acumulativo de Tarimas (UAs) y Trazabilidad de Ciclo de Vida en Salidas de Almacén

- **Estado:** Aceptado
- **Fecha:** 2026-09-14
- **Autores:** Equipo de Arquitectura e Ingeniería 4GUARD WMS (Frontend, Backend & BD)
- **Módulos Afectados:**
  - `apps/admin-console/src/app/features/warehouse-movements`
  - `4guard_be/src/main/java/com/fourguard/wms/application/usecase/WarehouseOutboundService.java`
  - `4guard_be/src/main/java/com/fourguard/wms/infrastructure/persistence/entity/WarehouseOutboundItemEntity.java`
  - `wms.warehouse_outbounds`, `wms.warehouse_outbound_items`, `wms.inventory_items`, `wms.inventory_movements`

---

## 1. Contexto y Problema Operativo

En las operaciones logísticas y de despacho de almacén de 4GUARD WMS:

1. **Despacho Consolidado Multi-Producto / Multi-Bahía:**
   En los pedidos reales de clientes y cedis, una sola orden o remisión de salida (`SAL-YYYY-XXXXXX`) frecuentemente consolida múltiples SKUs distintos (ej. 10 tarimas de Café, 4 tarimas de Leche, 2 tarimas de Envases) provenientes de diversas bahías y de remisiones de entrada independientes.
   La interfaz anterior restringía la selección a un solo SKU activo, perdiendo la selección previa al cambiar de producto o cliente.

2. **Selección Granular por Tarima Física (UA / SSCC) y Partición de Remisiones:**
   Cuando una remisión de entrada contiene $N$ tarimas (ej. 30 tarimas), los clientes pueden solicitar una extracción parcial (ej. 10 tarimas específicas por UA, o las 10 prioritarias por FEFO). El operador debe poder seleccionar la cantidad exacta con auto-selección inteligente o mediante escaneo directo con pistola RF en andén.

3. **Reglas FEFO vs. Alertas Pablo y Salidas de Merma/Destrucción:**
   - La regla **FEFO** (First Expired, First Out) debe sugerir automáticamente las tarimas con menor vida útil que se encuentren en estado óptimo.
   - La **Alerta Pablo** (<30 días de vida útil) debe actuar como una notificación visual de advertencia sin ocultar la mercancía del catálogo.
   - En casos operativos de **salidas por merma, devolución o destrucción**, el sistema no debe bloquear rígidamente las tarimas caducadas, sino permitir su despacho explícito identificándolas claramente con badges de auditoría.

4. **Trazabilidad y Amarre de Datos de Ciclo de Vida (End-to-End Audit):**
   Para auditorías regulatorias y fiscales, cada tarima despachada en `warehouse_outbound_items` debe conservar la referencia inmutable hacia:
   - Su identificador interno de inventario (`item_id` / `inventory_items.id`).
   - Su código de tarima físico / UA (`pallet_code` / `sscc`).
   - El número de documento / remisión de entrada original (`sap_folio` / `doc_number`).
   - La bahía de origen, lote y fecha de caducidad.
   - La transición atómica de estado en el Kardex (`AVAILABLE` -> `DISPATCHED`).

---

## 2. Decisión de Arquitectura

### 2.1 Manifiesto Acumulativo de Despacho (Dispatch Basket)
Se implementa en el frontend un estado reactivo acumulativo (`selectedPalletsMap`) que mantiene todas las tarimas seleccionadas a través de múltiples búsquedas de productos. El usuario puede:
- Buscar o escanear productos y UAs libremente sin perder los ítems ya incorporados al manifiesto.
- Visualizar un panel consolidado tipo **"Manifiesto de Despacho"** con subtotales por producto, total de tarimas, piezas y SKUs distintos.
- Eliminar tarimas de forma unitaria o por lote directamente desde el manifiesto.

### 2.2 Desacoplamiento de Cliente en Búsqueda y Validación Backend
- La búsqueda de productos en el Paso 2 consulta el inventario general y catálogo consolidado del almacén.
- El backend (`WarehouseOutboundService`) valida que las tarimas seleccionadas existan y se encuentren en estado `AVAILABLE` (o salida extraordinaria autorizada), descontándolas atómicamente y registrando el movimiento de salida en `inventory_movements`.

### 2.3 Modelo de Trazabilidad Integral
Cada registro de `warehouse_outbound_items` y `OutboundItemResponse` preserva los metadatos de origen:
```
[Recepción de Entrada (F01)]
    └── Doc Entrada (sap_folio / REM-XXXX)
          └── inventory_items (id, sscc, sku_id, batch, exp_date, location)
                 │
                 ▼ [Despacho Outbound (F03) - SAL-YYYY-XXXXXX]
                 ├── warehouse_outbounds (folio, carrier, driver, seal, operator)
                 ├── warehouse_outbound_items (item_id, pallet_code, remision_no, lot, pieces)
                 └── inventory_movements (EXIT / OUTBOUND con referencia cruzada)
```

---

## 3. Consecuencias y Beneficios

### Positivas
- **Flexibilidad Operativa Total:** Despachos mono-producto y multi-producto sin fricción.
- **Trazabilidad Inquebrantable:** Ciclo de vida completo desde la recepción hasta la salida para auditorías.
- **Optimización de Andén:** Selección ágil por cantidad (FEFO) o por escaneo directo de UA (pistola RF / tablet).
- **Control de Caducidades:** Visibilidad clara de alertas preventivas (Pablo) y soporte para mermas/destrucción.

### Cumplimiento
- Compatible con el estándar Synexia UI, Signals de Angular 17+ y arquitectura hexagonal Spring Boot 3.
