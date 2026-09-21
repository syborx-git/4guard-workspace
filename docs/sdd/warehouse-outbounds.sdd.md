# SDD: Salidas de Almacén (Outbound Despachos F03) — Backend

> **Módulo:** Operación y Logística → Movimientos de Almacén → Salidas de Almacén  
> **HU:** HU-152 — Salidas de Almacén y Despacho Outbound (Flujo Simplificado y Optimizado)  
> **Versión:** 2.0.0 (Despacho Multi-Producto, Escaneo Masivo y Trazabilidad Integral ADR-016)  
> **Estado:** 🏁 IMPLEMENTADO / LISTO PARA PR  
> **Fecha:** 2026-09-14  
> **Patrón:** Hexagonal Architecture + Spec-Driven Development (SDD)  

---

## 1. Descripción del Módulo

El submódulo **Salidas de Almacén (Outbound F03)** gestiona el egreso formal y despacho físico de mercancía desde las bahías de almacenamiento hacia clientes o centros de distribución:

1. **Despacho Consolidado Multi-Producto (ADR-016):**
   - Una sola orden de salida (`SAL-YYYY-XXXXXX`) puede consolidar múltiples SKUs distintos provenientes de diferentes bahías y remisiones de entrada independientes.
2. **Escaneo Masivo y Búsqueda Predictiva de UAs:**
   - Lectura rápida por código de barras de tarima (SSCC / UA física) o SKU con respuesta sub-10ms para terminales RF / pistolas láser.
   - Validación masiva de lotes de tarimas en una sola petición (`/validate-pallets`).
3. **Motor FEFO y Regla de Pablo:**
   - Sugerencia automática de tarimas por menor vida útil (`expiration_date ASC`).
   - Identificación visual de **Alerta Pablo** ($\le$ 30 días) y soporte explícito para salidas por merma/destrucción de mercancía caducada sin bloqueos rígidos.
4. **Impacto Atómico en Inventario (Kardex):**
   - Transición de estado en `wms.inventory_items` hacia `DISPATCHED` (50).
   - Generación de movimientos de inventario en `wms.inventory_movements` con tipo `EXIT` y referencia cruzada al documento de entrada original (`sap_folio`).
5. **Cancelación Extraordinaria:**
   - Revocación autorizada por Administrador con contraseña, restauración de tarimas a `AVAILABLE` (30) y generación de movimientos de compensación `ENTRY`.
6. **Auditoría Homologada:**
   - Trazabilidad cronológica de eventos (`SALIDA_REGISTRADA`, `SALIDA_DESPACHADA`, `SALIDA_CANCELADA`) con metadatos de usuario, chofer, placas, montacarguista y sello.

---

## 2. Modelo de Base de Datos

### 2.1 Tabla: `wms.warehouse_outbounds` (Cabecera de Despacho)

| Campo | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `id` | UUID | PK, NOT NULL | Identificador único RFC 4122 v4 |
| `organization_id` | UUID | FK `wms.organizations.id`, NOT NULL | Organización multi-tenant |
| `branch_id` | UUID | FK `wms.branches.id`, NOT NULL | Sucursal / Planta |
| `folio` | VARCHAR(30) | NOT NULL, UNIQUE | Consecutivo formato `SAL-YYYY-XXXXXX` |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT `COMPLETED` | `DRAFT`, `CONFIRMED`, `COMPLETED`, `CANCELLED` |
| `client_id` | UUID | FK `wms.clients.id`, NOT NULL | Cliente propietario |
| `destination_id` | UUID | FK `wms.client_destinations.id`, NULLABLE | Destino físico / Planta |
| `destination_name` | VARCHAR(200) | NULLABLE | Snapshot del nombre de destino |
| `destination_address` | TEXT | NULLABLE | Snapshot de la dirección de entrega |
| `carrier_id` | UUID | FK `wms.carriers.id`, NULLABLE | Línea transportista |
| `forklift_operator_id`| UUID | FK `wms.forklift_operators.id`, NULLABLE| Montacarguista asignado |
| `transport_type` | VARCHAR(30) | NOT NULL, DEFAULT `TRAILER` | `CAMION`, `TORTON`, `TRAILER` |
| `driver_name` | VARCHAR(150) | NOT NULL | Nombre del chofer |
| `economicNumber` | VARCHAR(30) | NULLABLE | No. Económico del tracto |
| `box_economic_number`| VARCHAR(30) | NULLABLE | No. Económico de la caja |
| `tractor_plates` | VARCHAR(20) | NOT NULL | Placas del tractocamión |
| `box_plates` | VARCHAR(20) | NOT NULL | Placas de la caja seca/refrigerada |
| `seal_number` | VARCHAR(50) | NOT NULL | No. de sello / marchamo / cincho de seguridad |
| `remision_no` | VARCHAR(60) | NOT NULL | No. de remisión de salida |
| `total_pallets` | INT | NOT NULL, DEFAULT 0 | Total tarimas despachadas |
| `total_pieces` | NUMERIC(12,2) | NOT NULL, DEFAULT 0 | Total piezas |
| `distinct_skus` | INT | NOT NULL, DEFAULT 0 | Cantidad de SKUs distintos |
| `cancelled_at` | TIMESTAMPTZ | NULLABLE | Fecha de cancelación |
| `cancellation_reason` | TEXT | NULLABLE | Motivo de revocación |
| `cancelled_by` | VARCHAR(100) | NULLABLE | Administrador autorizador |
| `version` | BIGINT | NOT NULL, DEFAULT 1 | Optimistic locking |
| `created_at` / `updated_at` | TIMESTAMPTZ | AUTO | Timestamps de auditoría |
| `created_by` / `updated_by` | VARCHAR(36) | NOT NULL | Usuario emisor |

### 2.2 Tabla: `wms.warehouse_outbound_items` (Detalle de Tarimas / UAs Despachadas)

| Campo | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `id` | UUID | PK, NOT NULL | Identificador único del ítem de salida |
| `outbound_id` | UUID | FK `wms.warehouse_outbounds.id`, NOT NULL | Relación con cabecera |
| `item_id` | UUID | FK `wms.inventory_items.id`, NOT NULL | Tarima física en inventario |
| `pieces` | NUMERIC(10,2) | NOT NULL, DEFAULT 0 | Cantidad de piezas en la tarima |
| `pallet_code` | VARCHAR(50) | NOT NULL | Código UA / SSCC (`037613041909243094`) |
| `lot_number` | VARCHAR(50) | NULLABLE | Lote de fabricación |
| `expiration_date` | DATE | NULLABLE | Fecha de caducidad |
| `location_code` | VARCHAR(50) | NULLABLE | Bahía de origen (`A-14`, `M-98`) |
| `created_at` | TIMESTAMPTZ | AUTO | Timestamp de registro |

---

## 3. Endpoints REST

**Base path:** `/api/v1/warehouse-outbounds`

| Método | Ruta | Permiso | Descripción |
|---|---|---|---|
| `POST` | `/` | `WAREHOUSE_MOVEMENTS_CREATE` | Registrar salida / despacho outbound (Atómico) |
| `GET` | `/{id}` | `WAREHOUSE_MOVEMENTS_READ` | Detalle completo de salida y tarimas |
| `GET` | `/` | `WAREHOUSE_MOVEMENTS_READ` | Listar salidas con filtros y búsqueda |
| `POST` | `/{id}/cancel` | `WAREHOUSE_MOVEMENTS_CANCEL` | Cancelar salida con reautenticación Admin |
| `GET` | `/inventory-batches` | `WAREHOUSE_MOVEMENTS_READ` | Consultar lotes disponibles con sugerencia FEFO y búsqueda |
| `GET` | `/scan-pallet` | `WAREHOUSE_MOVEMENTS_READ` | Escaneo rápido sub-10ms de tarima por SSCC/código de barras |
| `POST` | `/validate-pallets` | `WAREHOUSE_MOVEMENTS_READ` | Validación masiva de lote de códigos de barras |
| `GET` | `/{id}/audit` | `WAREHOUSE_MOVEMENTS_READ` | Consultar auditoría y línea de tiempo |

---

## 4. Contratos de API (Payloads de Entrada y Salida)

### 4.1 `POST /api/v1/warehouse-outbounds` — Registrar Salida

#### Request Body (`CreateOutboundRequest`)
```json
{
  "organizationId": "a53f0907-9fa5-4bdf-87db-2eb5e7683935",
  "branchId": "b73f0907-9fa5-4bdf-87db-2eb5e7683936",
  "clientId": "c9b4e182-7d3f-4e91-8845-a4b5c6d7e8f9",
  "destinationId": "b9c6beee-1e5b-4e3c-8e46-32f0390bf0df",
  "destinationName": "CENTRO DE NEGOCIO PAC",
  "destinationAddress": "Planta Nestlé PAC, Toluca, Estado de México",
  "carrierId": "f1a9b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "carrierName": "TRANSPORTADORA GOLA S.A. DE C.V.",
  "forkliftOperatorId": "852584b7-37c7-4a5f-b4dc-c7b6097a9c51",
  "forkliftOperatorName": "Hector Villalva Ayala",
  "transportType": "TRAILER",
  "driverName": "Juan Carlos Pérez Morales",
  "economicNumber": "ECO-104",
  "boxEconomicNumber": "BOX-880",
  "tractorPlates": "77-AA-1B",
  "boxPlates": "99-BB-2C",
  "sealNumber": "SL-994821",
  "remisionNo": "REM-265001",
  "selectedItemIds": [
    "11111111-2222-3333-4444-555555555551",
    "11111111-2222-3333-4444-555555555552"
  ]
}
```

#### Response Body (`ApiResponse<OutboundResponse>`) — `200 OK`
```json
{
  "success": true,
  "message": "Salida registrada con éxito",
  "data": {
    "id": "e81a3d90-34b2-4d7a-89bc-99011e4f210a",
    "organizationId": "a53f0907-9fa5-4bdf-87db-2eb5e7683935",
    "branchId": "b73f0907-9fa5-4bdf-87db-2eb5e7683936",
    "folio": "SAL-2026-000003",
    "status": "COMPLETED",
    "clientId": "c9b4e182-7d3f-4e91-8845-a4b5c6d7e8f9",
    "clientName": "Nestlé México S.A. de C.V.",
    "destinationId": "b9c6beee-1e5b-4e3c-8e46-32f0390bf0df",
    "destinationName": "CENTRO DE NEGOCIO PAC",
    "destinationAddress": "Planta Nestlé PAC, Toluca, Estado de México",
    "carrierId": "f1a9b2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
    "carrierName": "TRANSPORTADORA GOLA S.A. DE C.V.",
    "forkliftOperatorId": "852584b7-37c7-4a5f-b4dc-c7b6097a9c51",
    "forkliftOperatorName": "Hector Villalva Ayala",
    "transportType": "TRAILER",
    "driverName": "Juan Carlos Pérez Morales",
    "economicNumber": "ECO-104",
    "boxEconomicNumber": "BOX-880",
    "tractorPlates": "77-AA-1B",
    "boxPlates": "99-BB-2C",
    "sealNumber": "SL-994821",
    "remisionNo": "REM-265001",
    "totalPallets": 2,
    "totalPieces": 960.00,
    "distinctSkus": 1,
    "items": [
      {
        "id": "77223344-5566-7788-99aa-bbccddeeff01",
        "itemId": "11111111-2222-3333-4444-555555555551",
        "palletCode": "037613041909243094",
        "skuCode": "8500297",
        "skuDescription": "NESCAFE CLASICO 5KG MX",
        "clientName": "Nestlé México",
        "inboundRemisionNo": "REM-26506",
        "lotNumber": "LOTE-2026-A1",
        "expirationDate": "2026-12-31",
        "locationCode": "A-14",
        "pieces": 480.00,
        "createdAt": "2026-09-14T22:38:00Z"
      }
    ],
    "version": 1,
    "createdAt": "2026-09-14T22:38:00Z",
    "createdBy": "admin"
  },
  "timestamp": "2026-09-14T22:38:00Z"
}
```

---

### 4.2 `GET /api/v1/warehouse-outbounds/scan-pallet` — Escaneo Rápido de Tarima

#### Query Params
- `barcode`: `037613041909243094` (SSCC, UA externa o UUID de tarima)
- `organizationId`: UUID (opcional)
- `branchId`: UUID (opcional)

#### Response Body (`ApiResponse<ScanPalletResponse>`) — `200 OK`
```json
{
  "success": true,
  "message": "Tarima encontrada",
  "data": {
    "itemId": "11111111-2222-3333-4444-555555555551",
    "palletCode": "037613041909243094",
    "externalUa": "UA-8810-1",
    "skuId": "99887766-5544-3322-1100-aabbccddeeff",
    "skuCode": "8500297",
    "productName": "NESCAFE CLASICO 5KG MX",
    "category": "CAFÉ Y BEBIDAS",
    "clientId": "c9b4e182-7d3f-4e91-8845-a4b5c6d7e8f9",
    "clientName": "Nestlé México",
    "lotNumber": "LOTE-2026-A1",
    "inboundRemisionNo": "REM-26506",
    "manufacturingDate": "2026-01-15",
    "expirationDate": "2026-10-10",
    "daysRemaining": 26,
    "pabloStatus": "PABLO_ALERT",
    "pabloLabel": "Alerta Pablo (26d)",
    "isSuggestedFefo": true,
    "pieces": 480.00,
    "palletTypeId": "MADERA_ESTANDAR",
    "palletTypeLabel": "Madera Estándar",
    "locationCode": "A-14",
    "state": "AVAILABLE"
  }
}
```

---

### 4.3 `POST /api/v1/warehouse-outbounds/{id}/cancel` — Cancelar Salida

#### Request Body (`CancelOutboundRequest`)
```json
{
  "adminUsername": "admin",
  "adminPassword": "adminPassword",
  "reason": "Cancelación solicitada por mesa de control por cambio de unidad de transporte"
}
```

#### Response Body (`ApiResponse<OutboundResponse>`) — `200 OK`
```json
{
  "success": true,
  "message": "Salida cancelada con éxito",
  "data": {
    "id": "e81a3d90-34b2-4d7a-89bc-99011e4f210a",
    "folio": "SAL-2026-000003",
    "status": "CANCELLED",
    "cancelledAt": "2026-09-14T22:45:00Z",
    "cancellationReason": "Cancelación solicitada por mesa de control por cambio de unidad de transporte",
    "cancelledBy": "Hector Villalva Ayala"
  }
}
```

---

## 5. Reglas de Negocio Backend

| Regla | Descripción |
|---|---|
| **RN-OUT-01** | **Folio Único Inmutable:** Se genera mediante secuencia atómica PostgreSQL `seq_outbound_folio` con formato `SAL-YYYY-XXXXXX`. |
| **RN-OUT-02** | **Validación de Disponibilidad:** Las tarimas deben encontrarse en estado `AVAILABLE` (30) o `EXPIRED` (para salidas extraordinarias por merma autorizadas). Tarimas en estado `DISPATCHED`, `QUARANTINED` o `RESERVED` causan rechazo `422 Unprocessable Entity`. |
| **RN-OUT-03** | **Actualización Atómica del Kardex:** Al despachar, cada ítem genera un movimiento `EXIT` en `inventory_movements` conservando la trazabilidad hacia la remisión de entrada original (`sap_folio`). |
| **RN-OUT-04** | **Reversión en Cancelación:** La cancelación requiere autenticación criptográfica de usuario con rol Administrador/Supervisor. Restaura todas las tarimas asociadas al estado `AVAILABLE` y genera movimientos compensatorios `ENTRY`. |
| **RN-OUT-05** | **Batch Fetching de Alto Rendimiento:** `createOutbound` y `validatePallets` recuperan ítems en una sola consulta indexada con `JOIN FETCH` para soporte de andén con alta concurrencia. |
| **RN-OUT-06** | **Desacoplamiento de Destino:** Soporta destinos globales de clientes y plantas compartidas (`client_id = NULL` en `client_destinations`), permitiendo selección en andén con snapshot histórico inmutable. |
