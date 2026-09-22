# GAP Analysis Report — Módulo `warehouse-map`
**Módulo:** P5 — Mapa Interactivo 2D de Nave y Topología Física (HU-048 / HU-127)  
**Generado:** 2026-09-21  
**Metodología:** SDOP — Demo-Gap Analysis (READ-ONLY)  
**Fuente FE:** `4Guard_FE_UI/apps/admin-console/src/app/features/inventory/warehouse-map`  
**Fuente BE:** `4guard_be` (Spring Boot / PostgreSQL)  
**Analista:** Antigravity IDE  

---

## 1. Resumen Ejecutivo

El módulo `warehouse-map` en el frontend es actualmente una **maqueta 100% aislada que opera de forma exclusiva sobre LocalStorage**. A diferencia de otros módulos (como `security` o `admin/sections`), `warehouse-map` **no realiza ninguna llamada HTTP hacia Spring Boot**; todos los datos de secciones, posiciones, capacidades, códigos SKU, lotes simulados y bloqueos QM son generados y manipulados localmente por `WarehouseLayoutService`.

Por su parte, el backend de Spring Boot y la base de datos PostgreSQL poseen un modelo relacional y transaccional robusto para `warehouse_sections`, `locations`, `products_sku` e `inventory_items`, con endpoints REST operativos en `WarehouseSectionController` y `LocationController`. Sin embargo, existe una **desconexión estructural completa** entre ambos mundos:
1. Las **885 posiciones físicas individuales** (ej. `POS-A-001` a `POS-A-170`) que la UI visualiza no existen en la tabla `wms.locations` (la base de datos actualmente solo tiene 30 registros macroscópicos).
2. Los **metadatos de geometría SVG** (`polygonPoints`, coordenadas de etiquetas `labelPosition`) no están modelados en la base de datos ni en las entidades JPA.
3. La tabla `wms.inventory_items` cuenta con **0 registros**, y no existe un controlador REST para consultar el inventario asignado a una posición física.
4. La semántica de estados difiere: en el frontend `AVAILABLE` y `OCCUPIED` son estados de la posición, mientras que en el backend la ubicación permanece `ACTIVE` y la ocupación es un valor derivado de la presencia física de un `InventoryItem` (`current_occupancy > 0`).

### Cobertura Global

| Categoría | Estado | Observación |
|:---|:---:|:---|
| Tabla en PostgreSQL (`warehouse_sections`) | ✅ Sí | 12 registros de secciones en schema `wms` |
| Tabla en PostgreSQL (`locations`) | ⚠️ Parcial | Solo 30 filas (10 macro-ubicaciones, 12 muelles, 8 bahías). Faltan las 885 posiciones |
| Tabla en PostgreSQL (`products_sku`) | ✅ Sí | 26 SKUs activos (coincidencia idéntica con el catálogo Nestlé del frontend) |
| Tabla en PostgreSQL (`inventory_items`) | ⚠️ Vacía | 0 registros en base de datos; esquema completo con soporte SSCC, SKU, Lote |
| Tabla en PostgreSQL (`inventory_movements`) | ✅ Sí | Esquema listo para registrar trazabilidad y transferencias |
| Entidades JPA (`WarehouseSection`, `Location`, `ProductSku`, `InventoryItem`) | ✅ Sí | Completamente implementadas con auditoría y versiones |
| Controller REST Secciones | ✅ Sí | `WarehouseSectionController` (`/warehouse-sections`) |
| Controller REST Ubicaciones | ✅ Sí | `LocationController` (`/locations`) con soporte FSM PATCH `/status` |
| Controller REST Mapa Topológico Compuesto | ❌ No | No existe endpoint `GET /api/v1/warehouse-map/topology` |
| Controller REST Inventario por Ubicación | ❌ No | No existe `InventoryItemController` ni endpoint para consultar inventario por posición |
| Conectividad FE → BE en `warehouse-map` | ❌ 0% | El módulo consume exclusivamente `WarehouseLayoutService` (LocalStorage) |

---

## 2. Arquitectura del Módulo FE

### Componentes y Archivos Identificados

| Archivo | Tipo / Rol | Responsabilidad |
|:---|:---|:---|
| `warehouse-map.component.ts` | Componente Standalone Principal | Control del visor SVG 2D, pan/zoom interactivo, drill-down por sección, inspector modal de posición, formulario de bloqueo QM y filtros reactivos |
| `warehouse-map.component.html` | Template Visual | Canvas SVG con 11 polígonos de naves/almacenes, KPI bar superior, panel lateral deslizante y modal inspector de posición |
| `warehouse-map.component.css` | Hoja de Estilos | Tokens Synexia Dark, animaciones de pulso en vivo, gradientes de estado y layout responsivo |
| `warehouse-layout.service.ts` | LocalStorage Adapter / Mock Store | Mantiene en memoria y en `localStorage` las secciones y genera las posiciones deterministas |
| `warehouse-layout.models.ts` | Modelos de Datos Locales | Define `WarehouseSection`, `PositionDetail`, `PositionStatus`, `SectionStatus`, `WarehouseLayoutStats` |
| `warehouse-location.models.ts` | Modelos Compartidos | Define `DockItem` y `WarehouseBayItem` (HU-030 / HU-048) |
| `inventory.routes.ts` | Enrutamiento | Registra la ruta `/inventory/map` |

### Flujo Funcional de la Interfaz

```
[Plano General 2D (SVG 1000x870)]
       │
       ├── Hover en Polígono ────> Tooltip con Posiciones, Capacidad y Materiales
       │
       └── Clic en Polígono ─────> Drill-Down (Abre Panel Lateral)
                                         │
                                         ├── Filtro por Estado (ALL, OCCUPIED, AVAILABLE, BLOCKED)
                                         ├── Búsqueda por SKU, Posición o Lote
                                         │
                                         └── Clic en Tarjeta de Posición ──> Modal Inspector
                                                                                   │
                                                                                   ├── Acción: Liberar (AVAILABLE)
                                                                                   ├── Acción: Ocupar (OCCUPIED)
                                                                                   └── Acción: Bloquear QM (BLOCKED)
                                                                                           └── Formulario Motivo + Comentario
```

---

## 3. Mapa de Datos — LocalStorage Adapter (`WarehouseLayoutService`)

### Claves de LocalStorage Utilizadas

1. `4guard_warehouse_layout_v1`: Almacena el array serializado de `WarehouseSection[]`.
2. `4guard_warehouse_layout_v1_pos_{sectionId}`: Almacena el array de `PositionDetail[]` generado o modificado para cada sección (ej. `4guard_warehouse_layout_v1_pos_A`).

### 3.1 Estructura del Objeto `WarehouseSection` en LocalStorage

```typescript
{
  id: string,                  // 'A', 'E', 'F (D)', 'G', 'I', 'J(C)', 'L', 'K', 'H', 'B', 'C'
  code: string,                // 'A', 'E', etc.
  name: string,                // 'Almacén A - Embarques y Café Verde'
  category: string,            // 'Secos & Producto Terminado'
  posFijas: number,            // 170
  capacidadTarimas: number,    // 3740
  factorEstiba: string,        // '22 tarimas/pos'
  materials: string[],         // ['43211385 Envase Vidrio NESCAFE DOLCA 180g', ...]
  notes: string,               // 'Área QUALAMEX en posiciones 149 a 170. Incluye Rampa 2.'
  status: 'LOADED' | 'PENDING',
  polygonPoints: string,       // '800,188 976,188 976,330 878,330 878,790 800,790'
  labelPosition: { x: 865, y: 530 },
  sublabelPosition: { x: 865, y: 548 }
}
```

### 3.2 Estructura del Objeto `PositionDetail` en LocalStorage

```typescript
{
  id: string,                  // 'POS-A-1'
  positionNumber: number,      // 1..170
  code: string,                // 'POS-001'
  sectionId: string,           // 'A'
  sectionName: string,         // 'Almacén A - Embarques y Café Verde'
  skuCode: string,             // '43211385'
  skuDescription: string,      // '43211385 Envase Vidrio NESCAFE DOLCA 180g'
  status: PositionStatus,      // 'AVAILABLE' | 'OCCUPIED' | 'BLOCKED' | 'MAINTENANCE'
  capacityTarimas: number,     // 22
  currentTarimas: number,      // 0..22
  batchNumber: string,         // 'LOTE-202604-001'
  lastMovement: string,        // '21 sep 16:30' (string formateado)
  blockReason?: string         // 'Inspección de Calidad QM (Cuarentena)' o 'Motivo — Nota: comentario'
}
```

### 3.3 Mecanismo de Generación Sintética de Datos
En ausencia de conexión con el backend, `WarehouseLayoutService` genera deterministamente el estado inicial mediante una función de dispersión aritmética:
`hash = (section.code.charCodeAt(0) * 17 + i * 13) % 100`
- `hash < 10` → `status = 'BLOCKED'`, `currentTarimas = 0`, motivo cuarentena por defecto.
- `10 <= hash < 25` → `status = 'AVAILABLE'`, `currentTarimas = 0`.
- `25 <= hash < 40` → `status = 'OCCUPIED'`, `currentTarimas = floor(capacity * 0.5)`.
- `hash >= 40` → `status = 'OCCUPIED'`, `currentTarimas = capacity`.

---

## 4. Mapa de Campos: Frontend vs Backend / PostgreSQL

### 4.1 Secciones: `WarehouseSection` (FE) vs `wms.warehouse_sections` (BE)

| Campo FE (`WarehouseSection`) | Campo BE / Columna BD | Tipo BE | Estado | Observación |
|:---|:---|:---|:---:|:---|
| `id` | `id` | `UUID` | ⚠️ INCORRECT | FE usa letras `'A'`, `'E'`; BE usa UUIDs v4 |
| `code` | `code` | `VARCHAR(10)` | ⚠️ INCORRECT | FE usa `'A'`, `'F (D)'`; BE usa `'SEC-ALM-A'`, `'SEC-ALM-D'` |
| `name` | `name` | `VARCHAR(100)` | ⚠️ PARTIAL | BE concatena capacidad en el nombre: `Almacén A (Cap. 1496 Pallets / 68 Pos)` |
| `category` | ❌ Ausente | — | ❌ MISSING | No existe columna de categoría en `warehouse_sections` |
| `posFijas` | ❌ Ausente | — | ❌ MISSING | No existe columna; en BE es derivable con `count(locations)` |
| `capacidadTarimas` | ❌ Ausente | — | ❌ MISSING | No existe columna; en BE es derivable con `sum(locations.capacity_units)` |
| `factorEstiba` | ❌ Ausente | — | ❌ MISSING | Factor de estiba (`22 tarimas/pos`) solo existe en FE |
| `materials` (`string[]`) | ❌ Ausente | — | ❌ MISSING | No existe relación muchos a muchos de SKUs permitidos por sección |
| `notes` | ❌ Ausente | — | ❌ MISSING | No existe columna de notas en `warehouse_sections` |
| `status` | `status` | `VARCHAR(20)` | ⚠️ INCORRECT | FE: `'LOADED'` / `'PENDING'`. BE: `'ACTIVE'` / `'INACTIVE'` |
| `polygonPoints` | ❌ Ausente | — | ❌ MISSING | Coordenadas del polígono SVG ausentes en BD |
| `labelPosition` | ❌ Ausente | — | ❌ MISSING | Coordenadas de etiquetas visuales ausentes en BD |
| `sublabelPosition`| ❌ Ausente | — | ❌ MISSING | Coordenadas de sub-etiquetas ausentes en BD |
| — | `branch_id` | `UUID` | ❌ MISSING (en FE) | Requerido por BE para aislamiento multi-sucursal |
| — | `version`, `created_at`, `updated_at` | Audit fields | ❌ MISSING (en FE) | Soportado por BE en `BaseVersionedEntity` |

### 4.2 Posiciones / Ubicaciones: `PositionDetail` (FE) vs `wms.locations` (BE)

| Campo FE (`PositionDetail`) | Campo BE / Columna BD | Tipo BE | Estado | Observación |
|:---|:---|:---|:---:|:---|
| `id` | `id` | `UUID` | ⚠️ INCORRECT | FE usa `'POS-A-1'`; BE usa UUIDs v4 |
| `positionNumber` | `position` | `VARCHAR(10)` | ⚠️ PARTIAL | FE usa entero `1..170`; BE almacena string (ej. `'P01'`, `'A'`) |
| `code` | `code` | `VARCHAR(30)` | ⚠️ PARTIAL | FE usa `'POS-001'`; BE usa formato jerárquico `'LOC-A-01-N1'` |
| `sectionId` | `section_id` | `UUID` (FK) | ⚠️ INCORRECT | FE asocia código de una letra; BE asocia FK UUID a `warehouse_sections` |
| `sectionName` | `section.name` | `VARCHAR(100)` | ✅ OK | Disponible en join/mapeo JPA |
| `status` | `status` | `LocationStatus` | ⚠️ INCORRECT | FE: `AVAILABLE`, `OCCUPIED`, `BLOCKED`, `MAINTENANCE`. BE: `ACTIVE`, `BLOCKED`, `MAINTENANCE`, `INACTIVE` |
| `capacityTarimas` | `capacity_units` | `INTEGER` | ✅ OK | Capacidad unitaria en tarimas |
| `currentTarimas` | `current_occupancy` | `INTEGER` | ✅ OK | Conteo de tarimas ocupadas actualmente |
| `blockReason` | `status_reason` / `block_reason`| `VARCHAR(300)` / `TEXT` | ✅ OK | Mapeo directo de motivo de bloqueo |
| `skuCode` | ❌ No en locations | — | ⚠️ DATA | Pertenece a `wms.inventory_items.sku_id` |
| `skuDescription` | ❌ No en locations | — | ⚠️ DATA | Pertenece a `wms.products_sku.name` vía `inventory_items` |
| `batchNumber` | ❌ No en locations | — | ⚠️ DATA | Pertenece a `wms.inventory_items.batch_number` |
| `lastMovement` | ❌ No en locations | — | ⚠️ DATA | Pertenece a `wms.inventory_movements.created_at` |
| — | `zone`, `aisle`, `rack`, `level` | `VARCHAR` / `INT` | ❌ MISSING (en FE) | Atributos topológicos normalizados presentes en BE |
| — | `coord_x`, `coord_y`, `coord_z` | `INTEGER` | ⚠️ PARTIAL | Coordenadas 3D en BE, no usadas por el SVG 2D actual |

### 4.3 Formulario de Bloqueo QM (Modal Inspector) vs `PATCH /locations/{id}/status`

| Campo Formulario FE | Campo Payload BE (`UpdateLocationStatusRequest`) | Estado | Observación |
|:---|:---|:---:|:---|
| `newStatus = 'BLOCKED'` | `status = LocationStatus.BLOCKED` | ✅ OK | Mapea directamente al enum de BE |
| `blockReason` (Select 10 motivos) | `reason` (String) | ⚠️ PARTIAL | FE envía motivo seleccionado; BE lo valida obligatorio pero acepta cualquier String |
| `blockComment` (Textarea opcional) | `reason` (concatenado) | ⚠️ PARTIAL | FE concatena `reason — Nota: comment` dentro del string |
| — | Regla FSM de validación | ✅ OK | BE rechaza transiciones inválidas con HTTP 422 |
| Acción "Liberar" | `status = LocationStatus.ACTIVE` + `reason = null` | ✅ OK | Válido en FSM (BLOCKED → ACTIVE) |
| Acción "Ocupar" | ❌ No existe endpoint directo en `LocationController` | ❌ MISSING | La ocupación física requiere registrar un `InventoryItem` o mutar `current_occupancy` |

### 4.4 Catálogo de Productos y SKUs: Coincidencia de Datos

Al evaluar los materiales cargados en el frontend contra la tabla `wms.products_sku`, se comprobó que **los SKUs utilizados en el demo coinciden con el catálogo maestro en PostgreSQL**:

| SKU en Frontend (`warehouse-layout.service.ts`) | Registro en `wms.products_sku` (PostgreSQL) | Coincidencia |
|:---|:---|:---:|
| `43211385 Envase Vidrio NESCAFE DOLCA 180g` | `code: 43211385`, `name: ENVASE VIDRIO NESCAFE DOLCA 180G` | ✅ 100% |
| `43519988 Envase Vidrio NESCAFE DOLCA 175+25g` | `code: 43519988`, `name: ENVASE VIDRIO NESCAFE DOLCA 175+25G` | ✅ 100% |
| `41165316 Envase Vidrio NESCAFE DOLCA 50g` | `code: 41165316`, `name: ENVASE VIDRIO NESCAFE DOLCA 50G` | ✅ 100% |
| `41165277 Botella Vidrio Salsa Inglesa C&B 1090g` | `code: 41165277`, `name: BOTELLA VIDRIO SALSA INGLESA C&B 1090 G` | ✅ 100% |
| `44271527 Envase Vidrio Dawn NESCAFE 350g Ligero MX` | `code: 44271527`, `name: ENVASE VIDRIO DAWN NESCAFE 350 G LIGERO MX` | ✅ 100% |
| `41165274 BOTELLA VIDRIO JUGOS Y SALSAS MAG 800ML` | `code: 41165274`, `name: BOTELLA VIDRIO JUGOS Y SALSAS MAG 800ML` | ✅ 100% |
| `44318043 Jar Glass Dawn NESCAFE 100g 2` | `code: 44318043`, `name: ENVASE VIDRIO DAWN NESCAFE 100G` | ✅ 100% |
| `41165272 BOTELLA VIDRIO JUGOS Y SALSAS MAG 100ML` | `code: 41165272`, `name: BOTELLA VIDRIO JUGOS Y SALSAS MAG 100ML` | ✅ 100% |
| `41165273 BOTELLA VIDRIO JUGOS Y SALSAS MAG 200ML` | `code: 41165273`, `name: BOTELLA VIDRIO JUGOS Y SALSAS MAG 200ML` | ✅ 100% |
| `41165275 BOTELLA VIDRIO SALSA INGLESA C&B 160 G` | `code: 41165275`, `name: BOTELLA VIDRIO SALSA INGLESA C&B 160 G` | ✅ 100% |
| `41165276 BOTELLA VIDRIO SALSA INGLESA C&B 320 G` | `code: 41165276`, `name: BOTELLA VIDRIO SALSA INGLESA C&B 320 G` | ✅ 100% |
| `43457162 BOTELLA VIDRIO MAGGI 50ML` | `code: 43457162`, `name: BOTELLA VIDRIO MAGGI 50ML` | ✅ 100% |
| `44271537 Envase Vidrio Dawn NESCAFE Ligero 120g` | `code: 44271537`, `name: ENVASE VIDRIO DAWN NESCAFE LIGERO 120G` | ✅ 100% |
| `43543406 Envase Vidrio NESCAFE Dawn Jar 230g MX` | `code: 43543406`, `name: ENVASE VIDRIO NESCAFE DAWN JAR 230G MX` | ✅ 100% |
| `41165793 NESCAFE CLASICO 300g` | `code: 41165793`, `name: ENVASE VIDRIO NESCAFE CLASICO 300G` | ✅ 100% |
| `43759734 24K 50g` | `code: 43759734`, `name: ENVASE VIDRIO NESCAFE 24K 50G` | ✅ 100% |

---

## 5. Catálogo Completo de Brechas

### ❌ MISSING — Funcionalidades o datos que no existen en Backend / BD

| ID | Área | Descripción | Impacto |
|:---|:---|:---|:---:|
| **M-01** | Geometría SVG | No existe persistencia para `polygonPoints`, `labelPosition` ni `sublabelPosition` en BD ni en `WarehouseSectionEntity`. El backend desconoce la representación espacial 2D | 🔴 CRÍTICO |
| **M-02** | Registro de 885 Posiciones | En `wms.locations` solo existen 30 registros macroscópicos. Faltan por sembrar o sincronizar las 885 posiciones operativas individuales requeridas por la UI | 🔴 CRÍTICO |
| **M-03** | Endpoint Compuesto de Topología | No existe un endpoint tipo `GET /api/v1/warehouse-map/topology` que consolide naves, polígonos, métricas y posiciones en un solo viaje HTTP (evitando N+1 requests) | 🟡 ALTO |
| **M-04** | API de Inventario por Ubicación | No existe `InventoryItemController` ni endpoint para obtener el detalle de SKU, lote, SSCC y tarimas contenidas en una posición (`GET /inventory-items?locationId=...`) | 🟡 ALTO |
| **M-05** | Metadatos de Sección | Campos `category`, `factorEstiba`, `materials` y `notes` no existen en `wms.warehouse_sections` ni en `WarehouseSectionResponse` | 🟠 MEDIO |
| **M-06** | Consumo de Auditoría de Posición | El backend provee `GET /locations/{id}/audit`, pero el modal inspector del frontend no lo consume ni muestra el historial forense de quién bloqueó o liberó | 🟢 BAJA |

### ⚠️ INCORRECT — Inconsistencias de naming, identificadores y semántica

| ID | Campo / Área | Estado Frontend | Estado Backend | Impacto |
|:---|:---|:---|:---|:---:|
| **I-01** | Identificadores de Sección | IDs alfanuméricos cortos (`'A'`, `'E'`, `'F (D)'`, `'J(C)'`) | UUIDs v4 en `id`, y códigos tipo `'SEC-ALM-A'`, `'SEC-ALM-D'`, `'SEC-ALM-J'`. Además, la sección `'K'` no existe en BD | 🔴 CRÍTICO |
| **I-02** | Semántica FSM de Ocupación | `OCCUPIED` es un estado del enum `PositionStatus` en FE | En BE, el estado FSM es `ACTIVE` y la ocupación es un atributo numérico (`currentOccupancy > 0`) condicionado por stock | 🟡 ALTO |
| **I-03** | Estatus de Sección | `SectionStatus = 'LOADED' \| 'PENDING'` | `WarehouseSectionStatus = 'ACTIVE' \| 'INACTIVE'` | 🟠 MEDIO |
| **I-04** | Desalineación de Capacidades | Almacén A: FE reporta 170 pos / 3740 tarimas. En BD el nombre reporta 68 pos / 1496 pallets. Almacenes B, C y H: En FE figuran `PENDING` con 0 pos; en BD ya tienen 36, 86 y 25 posiciones activas | 🟡 ALTO |

### ⚠️ PARTIAL — Funcionalidad soportada a medias o con workarounds

| ID | Área | Descripción | Impacto |
|:---|:---|:---|:---:|
| **P-01** | Catálogo de Motivos de Bloqueo | FE define 10 motivos en `WMS_BLOCK_REASONS`; BE acepta texto libre en `statusReason` sin validación contra tabla de catálogo normalizada (`cat_block_reasons`) | 🟠 MEDIO |
| **P-02** | Cálculo de Métricas (KPIs) | Las métricas (`WarehouseLayoutStats`, % de ocupación, total tarimas) se calculan en TypeScript mediante `computed()`; no existe endpoint analítico en Spring Boot | 🟠 MEDIO |
| **P-03** | Reglas de Almacenamiento por Sección | Las secciones tienen materiales fijos en FE; en BE no existe la regla de qué familias o SKUs pueden ubicarse en cada sección | 🟠 MEDIO |

### 👁️ VISUAL — Discrepancias de presentación y experiencia de usuario

| ID | Área | Descripción | Impacto |
|:---|:---|:---|:---:|
| **V-01** | Indicador "EN VIVO" | El badge muestra pulso verde "EN VIVO", pero no está conectado a WebSockets (STOMP) ni a Server-Sent Events (SSE) para reflejar cambios en tiempo real de montacargas | 🟢 BAJA |
| **V-02** | Filtro `MAINTENANCE` Incompleto | El filtro de pestañas del panel lateral contiene `ALL`, `OCCUPIED`, `AVAILABLE`, `BLOCKED`, pero omite `MAINTENANCE` a pesar de estar definido en el enum | 🟢 BAJA |
| **V-03** | Nivel de Detalle en el Plano SVG | El plano general muestra el contorno del edificio pero no dibuja pasillos ni racks en el canvas principal; la granularidad física se relega a la lista del drawer | 🟢 BAJA |

### 💾 DATA — Discrepancias de persistencia, integridad y sincronización

| ID | Área | Descripción | Impacto |
|:---|:---|:---|:---:|
| **D-01** | Persistencia Aislada en LocalStorage | Todas las mutaciones ("Liberar", "Ocupar", "Bloquear QM") se escriben en `localStorage`; ninguna otra sesión de usuario ve los cambios | 🔴 CRÍTICO |
| **D-02** | Lotes y Fechas Sintéticas | Los números de lote (`LOTE-2026XX-YYY`) y fechas de movimiento se generan mediante una fórmula matemática con `Date.now() - hash * 3600000`, sin consultar movimientos reales | 🟡 ALTO |
| **D-03** | Tabla `inventory_items` Vacía | La tabla `wms.inventory_items` cuenta con 0 filas en la base de datos, impidiendo desplegar inventario real en las ubicaciones | 🟡 ALTO |

---

## 6. Endpoints REST — Cobertura y Brechas

| Endpoint Backend | Método HTTP | Estado en `warehouse-map` | Comentario |
|:---|:---:|:---:|:---|
| `/api/v1/warehouse-sections` | `GET` | ❌ No invocado | Existe en BE (`WarehouseSectionController`), devuelve las 12 secciones |
| `/api/v1/locations` | `GET` | ❌ No invocado | Existe en BE (`LocationController`), devuelve ubicaciones físicas |
| `/api/v1/locations/{id}/status` | `PATCH` | ❌ No invocado | Existe en BE; soporta cambios a `BLOCKED` y `ACTIVE` con motivo |
| `/api/v1/locations/{id}/audit` | `GET` | ❌ No invocado | Existe en BE; devuelve historial cronológico de auditoría |
| `/api/v1/products-sku` | `GET` | ❌ No invocado | Existe en BE (`ProductSkuController`), contiene los SKUs reales |
| `/api/v1/warehouse-map/topology` | `GET` | ❌ Inexistente en BE | **Requerido:** DTO compuesto con naves, polígonos y resumen de ocupación |
| `/api/v1/inventory-items?locationId=...` | `GET` | ❌ Inexistente en BE | **Requerido:** Consulta de tarimas y lotes por ubicación |

---

## 7. Matriz de Estados y Ciclo de Vida de Posición

```
Frontend (PositionStatus):
   ┌───────────┐    Ocupar     ┌───────────┐
   │ AVAILABLE │ ────────────> │ OCCUPIED  │
   └───────────┘ <──────────── └───────────┘
         │          Liberar          │
         │                           │
         │ Bloquear QM               │ Bloquear QM
         ▼                           ▼
   ┌───────────────────────────────────────┐
   │                BLOCKED                │
   └───────────────────────────────────────┘
         │
         │ Liberar
         ▼
   ┌───────────┐
   │ AVAILABLE │
   └───────────┘

Backend FSM (LocationStatus):
              ┌───────────────────────────┐
              │          ACTIVE           │ <──────────┐
              │ (currentOccupancy: 0 o >0)│            │
              └───────────────────────────┘            │
                 │                     │               │
                 │ Motivo obligatorio │ Motivo oblig. │
                 ▼                     ▼               │
         ┌───────────────┐     ┌─────────────┐         │
         │    BLOCKED    │     │ MAINTENANCE │         │
         └───────────────┘     └─────────────┘         │
                 │                     │               │
                 └─────────────────────┴───────────────┘
                                  │
                                  └── Transición a ACTIVE
```

### Principales Diferencias Semánticas:
1. En el frontend, una posición pasa directamente de `AVAILABLE` a `OCCUPIED` modificando `currentTarimas`.
2. En el backend, la ubicación siempre permanece en `status = ACTIVE` mientras esté disponible u ocupada; la ocupación física depende de la existencia de registros en `inventory_items` referenciando `location_id`.
3. El backend prohíbe pasar a `INACTIVE` si `currentOccupancy > 0` (retorna HTTP 409).
4. El backend exige de forma estricta un `reason` no nulo para transicionar a `BLOCKED` o `MAINTENANCE` (retorna HTTP 400 en su defecto).

---

## 8. Resumen de Brechas por Severidad

| Severidad | Cantidad | Descripción |
|:---:|:---:|:---|
| 🔴 **CRÍTICA** | 4 | Desconexión total de backend (D-01), falta de geometría SVG en backend (M-01), ausencia de las 885 posiciones en BD (M-02) e incompatibilidad de identificadores de sección (I-01) |
| 🟡 **ALTA** | 6 | Falta de endpoint compuesto de topología (M-03), falta de endpoint de inventario (M-04), desalineación semántica de ocupación FSM (I-02), discrepancia de capacidades (I-04), lotes sintéticos (D-02), tabla de inventario vacía (D-03) |
| 🟠 **MEDIA** | 4 | Metadatos de sección faltantes en BE (M-05), estatus de sección LOADED vs ACTIVE (I-03), catálogo de motivos libre sin tabla maestra (P-01), KPIs calculados solo en cliente (P-02) |
| 🟢 **BAJA** | 4 | Auditoría de posición no visualizada (M-06), badge "EN VIVO" no reactivo (V-01), filtro MAINTENANCE omitido en UI (V-02), falta de racks en plano SVG (V-03) |

---

## 9. Recomendaciones Prioritizadas de Implementación

### Fase 1 — Modelado y Persistencia en Base de Datos (P1)
- **[BE / DB]** Crear migración Flyway (`V23__seed_warehouse_map_topology.sql`) para:
  1. Insertar la sección faltante `SEC-ALM-K` (Almacén K - Racks Libres & Anexo).
  2. Homologar las capacidades de las secciones A, E, F(D), G, I, J(C), L, K con los valores acordados.
  3. Sembrar las 885 posiciones individuales en `wms.locations` con formato estructurado (`code: POS-A-001`, `section_id`, `type: PALLET`, `capacity_units`).
- **[BE / DB]** Añadir columna `metadata jsonb` o tabla `wms.warehouse_section_layouts` para almacenar los puntos poligonales SVG (`polygon_points`) y las coordenadas de etiquetas (`label_x`, `label_y`), permitiendo dinamismo visual sin hardcoding en el cliente.

### Fase 2 — Adaptador de Servicios y Bridge Hexagonal (P2)
- **[FE]** Refactorizar `WarehouseLayoutService` para seguir el patrón SDOP Bridge:
  - Crear `WarehouseLayoutHttpAdapter` que implemente un puerto abstracto `WarehouseLayoutRepository`.
  - Reemplazar la lectura directa de `localStorage` por llamadas HTTP a Spring Boot, manteniendo el adapter local únicamente como fallback resiliente o mock de pruebas E2E.
- **[BE]** Crear un endpoint optimizado `GET /api/v1/warehouse-map/topology?branchId=...` en un nuevo controlador `WarehouseMapController` que devuelva las secciones con su geometría SVG, posiciones activas, estado FSM y conteos de tarimas en una única carga inicial.

### Fase 3 — Gestión de Bloqueo QM e Integración FSM (P2)
- **[FE]** Conectar el formulario modal de bloqueo QM al endpoint existente:
  - Invocar `PATCH /api/v1/locations/{id}/status` con payload `{ status: 'BLOCKED', reason: blockReason + ' — ' + blockComment }`.
  - En la acción "Liberar", invocar `PATCH /api/v1/locations/{id}/status` con `{ status: 'ACTIVE' }`.
- **[BE]** Crear tabla de catálogo `wms.cat_block_reasons` con los 10 motivos estándar WMS y exponer `GET /api/v1/catalogs/block-reasons`.

### Fase 4 — Trazabilidad de Inventario y Tiempo Real (P3 / P4)
- **[BE]** Implementar endpoint `GET /api/v1/inventory-items?locationId=...` para alimentar el modal inspector con datos reales de SKU, lote, SSCC y fecha de ingreso desde `wms.inventory_items`.
- **[FE]** Integrar pestaña de auditoría en el inspector modal consumiendo `GET /api/v1/locations/{id}/audit`.
- **[FE / BE]** Conectar el badge "EN VIVO" a un topic WebSocket (`/topic/warehouse-map/{branchId}`) para que cuando un montacargas o guardia realice una recepción o transferencia, las celdas cambien de color sin recargar la página.

---

## 10. Estado de Especificaciones y Documentación

| Documento | Existe | Cubre `warehouse-map` | Estado |
|:---|:---:|:---:|:---|
| `docs/sdd/layout-management.sdd.md` | ✅ Sí | ⚠️ Parcial | Especifica el módulo `/layout` (árbol de 5 niveles y grid), pero no el plano SVG de naves `/inventory/map` |
| `docs/adr/ADR-013-normativa-homologacion-componentes.md` | ✅ Sí | ✅ Sí | Cubre lineamientos de componentes, tokens y colores FSM |
| SDD específico para `WarehouseMapComponent` (Plano 2D SVG) | ❌ No | ❌ No | **FALTANTE:** Debe redactarse `docs/sdd/warehouse-map-2d.sdd.md` previo a la fase de implementación |

---

*Reporte generado automáticamente por Antigravity IDE — Análisis 100% READ-ONLY.*  
*Ningún archivo fuente en Angular ni en Spring Boot fue modificado durante la ejecución de esta auditoría.*
