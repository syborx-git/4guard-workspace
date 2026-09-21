# SDD — Recepción de Mercancía (F01 WMS)

**Proyecto:** 4GUARD WMS  
**Módulo:** Operación y Logística → Movimientos de Almacén → Recepción de Mercancía  
**Tipo de Operación:** Transaccional de Ingreso / Alta de Mercancía y UAs (Inbound F01)  
**Estado:** Aprobado e Implementado  
**Versión:** 2.0 (Arquitectura Master-Detail Unificada con Control & Auditoría)  
**Patrón:** Spec-Driven Development (SDD)  
**Autor:** Equipo de Ingeniería 4GUARD / Synexia Framework  

---

## 1. Objetivo

Implementar y estandarizar el módulo **Recepción de Mercancía (F01)** en **4GUARD WMS** para controlar el ingreso físico y lógico de mercancías en andén de descarga, asegurando:

1. **Captura Previa en Caseta de Seguridad:** Registro de arribo de transporte, placas (tracto/caja), transportista, chofer, rampa asignada y sellos de seguridad.
2. **Alta de Recepción en Andén:** Captura de lote de fabricación, fechas de elaboración/caducidad, producto (SKU), piezas por tarima y tipo de tarima (CHEP, Madera Estándar, Plástica, etc.).
3. **Escáner Reactivo de UAs:** Registro unitario o por lote de códigos de tarima (SSCC/UA) con validación de unicidad.
4. **Compuerta de Seguridad con Doble Factor:** Autorización obligatoria por Líder/Supervisor de Almacén para el cierre final o cancelación extraordinaria.
5. **Control y Auditoría Integral:** Trazabilidad completa con registro cronológico de eventos (`RECEPCION_CREADA`, `RECEPCION_COMPLETADA`, `TARIMA_EDITADA`, `RECEPCION_CANCELADA`), metadatos rápidos y visualización de deltas de cambios.
6. **Comprobante Operativo Imprimible F01:** Emisión de pauta formal con desglose de tarimas, piezas y bloques de firma reglamentarios.

---

## 2. Arquitectura de la Interfaz (Master-Detail Workbench)

La interfaz se encuentra homologada con **Gestión de Transportistas** y **Gestión de Montacarguistas**, operando bajo el patrón **Master-Detail Workbench**:

### 2.1 Tarjetas KPI Superiores (4 Métricas en Tiempo Real)
* **Total Recepciones (`move_to_inbox`):** Conteo general de folios de recepción.
* **En Espera / Descarga (`hourglass_top`):** Recepciones registradas en caseta pendientes de cierre en andén.
* **Completadas (`check_circle`):** Recepciones autorizadas con ingreso exitoso al inventario.
* **Canceladas (`cancel`):** Recepciones revocadas por incidencias o rechazo de calidad.

### 2.2 Panel Izquierdo: Directorio Master de Recepciones
* Buscador reactivo por Folio (`#26506`), No. Remisión, Cliente, SKU o Producto.
* Filtros por estatus: `Todos`, `En Espera`, `Completados`, `Cancelados`.
* Tarjetas interactivas con folio dorado, cliente, remisión, rampa y badge de estado.
* Botón institucional dorado **`[ + Nueva Pre-Recepción ]`**.

### 2.3 Panel Derecho: Espacio de Trabajo Unificado
Opera en tres modos reactivos controlados por `formMode: 'idle' | 'create' | 'detail'`:

1. **Modo Inicial / Sin Selección (`formMode: 'idle'`):**
   * Tarjeta centrada con ícono, guía operativa y acceso rápido a nueva pre-recepción.
2. **Modo Captura / Alta de Pre-Recepción (`formMode: 'create'`):**
   * Formulario directo de captura de caseta (Transportista, Placas, Rampa, Chofer, Sellos).
3. **Modo Detalle / Proceso de Descarga (`formMode: 'detail'`):**
   * **Información General de Caseta:** Metadatos del transporte y chofer.
   * **Parámetros de Recepción:** Formulario interactivo (si está en espera) o ficha de solo lectura (si está completado/cancelado).
   * **Captura de Tarimas (UA):** Barra de escáner y tabla interactiva de tarimas descargadas.
   * **Información de Control & Auditoría:** Metadatos rápidos (Folio, Organización, Registrador, Fecha) y línea de tiempo interactiva (`carriers-timeline-preview`) con nodos codificados por color (Esmeralda, Azul, Ámbar, Rojo).
   * **Botonera de Acción:** Guardar avance, autorizar cierre e imprimir formato oficial.

---

## 3. Modelo de Datos y Eventos de Auditoría

```typescript
export interface ReceptionHeader {
  folio: string;
  status: 'REGISTERED' | 'COMPLETED' | 'CANCELLED';
  checkIn: CheckInCasetaData;
  lotNumber?: string;
  elaborationDate?: string;
  expirationDate?: string;
  productId: string;
  productName: string;
  supplierName?: string;
  piecesPerPallet: number;
  selectedPalletType: PalletType;
  observations?: string;
  storageLocation?: string;
  pallets: ReceptionPalletItem[];
  createdAt: string;
  completedAt?: string;
  capturedBy: string;
  leaderAuthorizedBy?: string;
  cancellationReason?: string;
  cancelledAt?: string;
}

export interface MovementAuditEntry {
  id?: string;
  action: string;
  actionLabel?: string;
  username: string;
  timestamp: string;
  details?: MovementAuditDetail[];
  reason?: string;
  authorizedBy?: string;
  observations?: string;
}
```

---

## 4. Matriz de Estados y Acciones de Auditoría

| Acción | Disparador | Color Nodo | Datos Registrados |
|---|---|---|---|
| `RECEPCION_CREADA` | Registro de pre-recepción en caseta | Esmeralda (`.carriers-tl-node--emerald`) | Línea, Rampa, Placas, Usuario |
| `RECEPCION_ACTUALIZADA` | Modificación de ubicación u observaciones | Ámbar (`.carriers-tl-node--amber`) | Campos modificados (deltas) |
| `TARIMA_EDITADA` | Corrección manual de piezas o tipo de tarima | Ámbar (`.carriers-tl-node--amber`) | Código UA, piezas previas y nuevas |
| `RECEPCION_COMPLETADA` | Autorización del líder y cierre de descarga | Azul (`.carriers-tl-node--blue`) | Lote, SKU, Total Tarimas, Piezas, Líder |
| `RECEPCION_CANCELADA` | Revocación extraordinaria por administrador | Rojo (`.carriers-tl-node--red`) | Motivo de cancelación, Autorizador |

---

## 5. Diseño Visual y Tipografía (Midnight Navy & Prestige Gold)

* **Tipografías:**
  * Display / Títulos: `'Outfit'`, sans-serif.
  * Cuerpo / Formularios: `'Inter'`, sans-serif.
  * Códigos / Folios / Placas / Fechas: `'JetBrains Mono'`, monospace.
* **Colores Institucionales:**
  * Midnight Navy: `--navy: #172033`, `--navy-mid: #25324a`.
  * Prestige Gold: `--gold: #c5a86b`, `--gold-light: #e0c87a`, `--gold-bg: rgba(197, 168, 107, 0.10)`.
  * Soporte Dark Mode nativo con `:host-context(.theme-dark)` y `.dark`.

---

## 6. Contratos Backend REST (Spring Boot — `/api/v1/warehouse-receptions`)

| Método | Endpoint | DTO / Payload | Descripción |
|---|---|---|---|
| `POST` | `/check-in` | `CreateCheckInRequest` | Registro inicial de arribo en caseta |
| `PUT` | `/{id}/parameters` | `UpdateReceptionParametersRequest` | Guardar avance de parámetros en andén |
| `GET` | `/{id}` | N/A | Consulta de detalle con tarimas y sellos |
| `GET` | `/` | Query params: `organizationId`, `branchId`, `status`, `search` | Consulta de listado master con KPIs |
| `POST` | `/{id}/pallets` | `AddReceptionPalletsRequest` | Escáner y alta de UAs |
| `PUT` | `/{id}/pallets/{palletId}` | `UpdatePalletRequest` | Edición de tarima individual |
| `DELETE` | `/{id}/pallets/{palletId}` | N/A | Eliminación de tarima |
| `POST` | `/{id}/complete` | `CompleteReceptionRequest` | Autorización Líder y cierre F01 |
| `POST` | `/{id}/cancel` | `CancelReceptionRequest` | Cancelación extraordinaria con Admin |
| `PUT` | `/{id}/change-remision` | `ChangeRemisionRequest` | Modificación de remisión con auditoría |
| `GET` | `/{id}/audit` | N/A | Historial de auditoría cronológica |

