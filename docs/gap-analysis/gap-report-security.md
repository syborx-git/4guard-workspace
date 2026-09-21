# GAP Analysis Report — Módulo `security`
**Módulo:** Caseta de Seguridad & Pases QR Digitales (F01-PO-CP-7.1.3-03)
**Generado:** 2026-09-21
**Metodología:** SDOP — Demo-Gap Analysis (READ-ONLY)
**Fuente FE:** `4Guard_FE_UI/apps/admin-console/src/app/features/security`
**Fuente BE:** `4guard_be` (Spring Boot / PostgreSQL)
**Analista:** Antigravity IDE

---

## 1. Resumen Ejecutivo

El módulo `security` en el frontend tiene **arquitectura FUNCIONAL y correctamente conectada al backend real**. No existen adaptadores LocalStorage puros como fuente única de datos; en cambio, el módulo usa un patrón híbrido (HttpAdapter + fallback a LocalStorage como cache resiliente). El backend posee tablas, entidades JPA, casos de uso, DTOs y controladores REST completamente implementados. Las brechas identificadas son mayoritariamente de tipo **PARTIAL** (campos sin mapear correctamente) e **INCORRECT** (inconsistencias de naming entre FE y BE).

### Cobertura Global

| Categoría | Estado |
|:---|:---|
| Tabla en PostgreSQL | ✅ `wms.security_pre_checkins` (V21 + V22) |
| Entidad JPA | ✅ `SecurityPreCheckinEntity` |
| DTOs Request/Response | ✅ 4 Request + 2 Response |
| Controller REST | ✅ `SecurityGateController` (`/api/v1/security-gate`) |
| Service (UseCase) | ✅ `SecurityGateService` |
| Dominio / Port | ✅ `SecurityGateUseCase` |
| Conectividad FE → BE | ✅ `WarehouseMovementsApiService` con endpoints reales |

---

## 2. Arquitectura del Módulo FE

### Componentes identificados

| Componente | Tipo | Archivo |
|:---|:---|:---|
| `SecurityGateComponent` | Componente Principal (Multi-Tab) | `security-gate/security-gate.component.ts` |
| `CarrierCheckinComponent` | Portal Móvil del Chofer (QR) | `carrier-checkin/carrier-checkin.component.ts` |
| `QrSimulatorCardComponent` | Sub-componente (UI helper) | `qr-simulator-card/qr-simulator-card.component.ts` |

### Pestañas del módulo `security-gate`

| Tab | `SecurityGateTab` Enum | Función |
|:---|:---|:---|
| Nuevo Registro / Pases QR | `REGISTRATION` | Generar pase QR, cargar datos de chofer, submit check-in |
| Unidades en Planta | `IN_YARD` | Listar vehículos dentro, filtrar, marcar salida |
| Historial y Auditoría | `HISTORY` | Registros completados con `status: COMPLETED_EXIT` |

> **Nota:** La pestaña `CHECKOUT` no existe como enum separado; el check-out se realiza dentro de `IN_YARD` mediante un modal superpuesto.

---

## 3. Mapa de Datos — LocalStorage Adapter (Resilience Cache)

### Clave LocalStorage: `4g_local_passes`

El módulo NO usa un `LocalStorageAdapter` formal de clase inyectable. En su lugar, tanto `SecurityGateComponent` como `CarrierCheckinComponent` y `WarehouseMovementsApiService` comparten acceso directo a `localStorage.getItem('4g_local_passes')`.

#### Estructura del objeto local persistido por `SecurityGateComponent.savePassToLocalStorage()`:

```typescript
{
  id: string,                  // 'pass-loc-' + Date.now()
  token: string,               // Token QR
  status: string,              // 'PENDING_DRIVER'
  operationType: string,       // 'CARGA' | 'DESCARGA'
  docNumber: string,
  noCartaPorte: string,
  remision: string,
  clientCode: string,
  clientName: string,
  carrierLineCode: string,
  carrierLine: string,
  driverName: string,
  tractorPlates: string,
  createdAt: ISO string,
  updatedAt: ISO string
}
```

#### Estructura adicional persistida por `CarrierCheckinComponent.savePassToLocalStorage()` (más completa):

```typescript
{
  ...campos_anteriores,
  driverLicense: string,       // licencia de conducir
  noEcoTractor: string,        // número económico tracto
  boxPlates: string,           // placas caja
  boxDimensions: string,       // medidas caja
  transportType: string,       // tipo transporte
  sealNumbers: string[],       // sellos de seguridad
  observations: string,
  driverSignature: string,     // base64 o token
  checklistData: string,       // JSON string con EPP + Revisión Caja
  submittedAt: ISO string
}
```

---

## 4. Mapa de Campos del Formulario FE vs BE

### 4.1 Formulario `checkInForm` (SecurityGateComponent) → `GuardCheckinCompletionRequest`

| Campo FE (`checkInForm`) | Campo BE | Estado | Observación |
|:---|:---|:---:|:---|
| `operacion` | `operationType` | ⚠️ INCORRECT | FE envía `operacion`; BE espera `operationType`. El `completePayload` mapea correctamente |
| `fecha` | `docDate` (LocalDate) | ⚠️ PARTIAL | FE envía string `YYYY-MM-DD`; BE mapea a `LocalDate` ✓ |
| `horaEntrada` | `receptionTime` (LocalTime) | ⚠️ PARTIAL | FE envía `HH:mm`; BE espera `LocalTime` |
| `horaSalida` | `departureTime` (LocalTime) | ⚠️ PARTIAL | FE envía string; BE mapea a `LocalTime` |
| `noCartaPorte` | `noCartaPorte` / `docNumber` | ✅ OK | Presente en DTO y entidad |
| `remision` | `remision` / `docNumber` | ✅ OK | Presente en DTO y entidad |
| `clientCode` | `clientCode` | ✅ OK | Alineado |
| `client` | `clientName` | ⚠️ INCORRECT | FE usa `client`; BE espera `clientName`. `completePayload` lo corrige |
| `carrierLineCode` | `carrierLineCode` | ✅ OK | Alineado |
| `carrierLine` | `carrierLine` | ✅ OK | Alineado |
| `nombreOperador` | `driverName` | ⚠️ INCORRECT | FE usa `nombreOperador`; BE espera `driverName`. Mapeado en `completePayload` |
| `placasTracto` | `tractorPlates` | ⚠️ INCORRECT | Mapeado en `completePayload` |
| `noEcoTractor` | `noEcoTractor` | ✅ OK | Presente desde V22 |
| `placasCaja` | `boxPlates` | ⚠️ INCORRECT | Mapeado en `completePayload` |
| `medidasCaja` | `boxDimensions` | ⚠️ INCORRECT | Mapeado en `completePayload` |
| `tipoTransporte` | `transportType` | ⚠️ INCORRECT | Mapeado en `completePayload` |
| `rampCode` | `rampCode` | ✅ OK | Alineado |
| `rampNumber` | `rampNumber` | ✅ OK | Alineado |
| `noSello` + `sealList[]` | `sealNumbers` (List<String>) | ⚠️ PARTIAL | FE fusiona campo libre + signal array. BE espera array. Conversión correcta |
| `eppZapatos` | `checklistData.epp.zapatos` (JSONB) | ⚠️ PARTIAL | FE serializa manualmente como JSON string |
| `eppCofia` | `checklistData.epp.cofia` | ⚠️ PARTIAL | Ídem |
| `eppCubrebocas` | `checklistData.epp.cubrebocas` | ⚠️ PARTIAL | Ídem |
| `eppChaleco` | `checklistData.epp.chaleco` | ⚠️ PARTIAL | Ídem |
| `revInteriorCaja` | `checklistData.caja.interior` | ⚠️ PARTIAL | FE serializa manualmente como JSON string |
| `revDanosCaja` | `checklistData.caja.danos` | ⚠️ PARTIAL | Ídem |
| `revDanosPuertas` | `checklistData.caja.puertas` | ⚠️ PARTIAL | Ídem |
| `revOloresExtranos` | `checklistData.caja.olores` | ⚠️ PARTIAL | Ídem |
| `revIndiciosPlagas` | `checklistData.caja.plagas` | ⚠️ PARTIAL | Ídem |
| `responsableVigilanciaNombre` | `processedBy` | ⚠️ PARTIAL | Nombre del guardia; BE mapea a `processedBy`. Funcional |
| `docCartaPorte` (SI/NO) | ❌ MISSING | ❌ MISSING | No existe campo separado en BE; solo persiste dentro del JSONB concatenado |
| `docRemision` (SI/NO) | ❌ MISSING | ❌ MISSING | Ídem anterior |
| Todos los `*Obs` (11 campos) | ❌ MISSING | ❌ MISSING | Solo se incluyen en el string de observaciones concatenado |

### 4.2 Formulario `checkOutForm` → `GuardCheckOutRequest`

| Campo FE | Campo BE | Estado |
|:---|:---|:---:|
| `departureTime` | `departureTime` (LocalTime) | ✅ OK |
| `exitSealNumbers` (string csv) | `exitSealNumbers` (List<String>) | ⚠️ PARTIAL — FE parsea con split(','); correcto |
| `exitObservations` | `exitObservations` | ✅ OK — presente en V22 |
| `guardNotes` | `guardNotes` | ✅ OK |

### 4.3 Formulario `checkInForm` (CarrierCheckinComponent) → `DriverCheckinSubmissionRequest`

| Campo FE | Campo BE | Estado | Nota |
|:---|:---|:---:|:---|
| `operacion` | `operationType` | ⚠️ INCORRECT | Mismo naming mismatch |
| `clientName` / `clientCode` | `clientName` / `clientCode` | ✅ OK | Correcto en este componente |
| `carrierLine` / `carrierLineCode` | `carrierLine` / `carrierLineCode` | ✅ OK | |
| `nombreOperador` | `driverName` | ⚠️ INCORRECT | Payload mapea explícitamente |
| `driverLicense` | `driverLicense` | ✅ OK | |
| `placasTracto` | `tractorPlates` | ⚠️ INCORRECT | Payload mapea |
| `placasCaja` | `boxPlates` | ⚠️ INCORRECT | Payload mapea |
| `medidasCaja` | `boxDimensions` | ⚠️ INCORRECT | Payload mapea |
| `tipoTransporte` | `transportType` | ⚠️ INCORRECT | Payload mapea |
| `eppZapatos/Cofia/Cubrebocas/Chaleco` | `checklistData` (JSONB) | ⚠️ PARTIAL | Serialización manual JSON |
| `rev*` (5 campos) | `checklistData` (JSONB) | ⚠️ PARTIAL | Ídem |
| `driverSignature` | `driverSignature` | ✅ OK | Base64 canvas → TEXT en BD |
| `declaracionVerdad` | ❌ MISSING | ❌ MISSING | Solo validación FE, no persiste en BD |
| `noEcoTractor` | `noEcoTractor` | ✅ OK | Desde V22 |

---

## 5. Catálogo Completo de Brechas

### ❌ MISSING — Existe en FE, no en BE

| ID | Campo FE | Descripción |
|:---|:---|:---|
| M-01 | `docCartaPorte` (SI/NO) | Verificación documental como campo booleano separado. No tiene columna en BD |
| M-02 | `docRemision` (SI/NO) | Ídem anterior |
| M-03 | `declaracionVerdad` | Aceptación bajo protesta del chofer. Solo validación en FE |
| M-04 | 11 campos `*Obs` (observaciones individuales) | No existen columnas separadas; se concatenan en `observations` TEXT |
| M-05 | Metadatos F01 (`controlNumber`, `revisionNumber`, etc.) | Solo se calculan en FE para impresión. No se persisten en BD |

### ⚠️ INCORRECT — Naming mismatch FE ↔ BE

| ID | Campo FE | Campo BE | Impacto | Workaround |
|:---|:---|:---|:---:|:---|
| I-01 | `operacion` | `operationType` | 🟡 ALTO | `completePayload` mapea. Funcional |
| I-02 | `client` (en `checkInForm`) | `clientName` | 🟡 ALTO | Riesgo en flujo directo sin QR |
| I-03 | `nombreOperador` | `driverName` | 🟡 ALTO | Mapeado en `completePayload` |
| I-04 | `placasTracto` | `tractorPlates` | 🟡 ALTO | Mapeado en payloads |
| I-05 | `placasCaja` | `boxPlates` | 🟡 ALTO | Mapeado en payloads |
| I-06 | `medidasCaja` | `boxDimensions` | 🟡 ALTO | Mapeado en payloads |
| I-07 | `tipoTransporte` | `transportType` | 🟡 ALTO | Mapeado en payloads |
| I-08 | `checkInData.client` (flujo directo) | `clientName` | 🟡 ALTO | Posible fallo en validación BE |
| I-09 | `pass.noEcoTractor` (lectura de PassResponse) | Ausente en `PassResponse.java` | 🟠 MEDIO | FE lee del cache local |
| **I-10** | **`status === 'COMPLETED_EXIT'`** | **No existe en BD** | **🔴 CRÍTICO** | **El historial siempre vacío si BE responde** |

### ⚠️ PARTIAL — Funcionalidad parcial

| ID | Área | Descripción |
|:---|:---|:---|
| P-01 | Checklist JSONB | EPP y revisión de caja serializados manualmente. No expuestos como campos estructurados en `PassResponse` |
| P-02 | Catálogos dinámicos | `CarrierCheckinComponent` tiene fallback hardcodeado con UUIDs de demo |
| P-03 | Auditoría de pase | No existe `GET /security-gate/passes/{id}/audit` |
| P-04 | Operador Montacargas | Existe en BD y DTO pero no se envía desde FE en el formulario de caseta |
| P-05 | `isReadyForExit` | Calculado en BE; si BE falla, el botón Check-Out queda bloqueado en FE |
| P-06 | Expiración de pase | `expires_at` existe en BD pero FE no muestra indicador visual de expiración |

### 👁️ VISUAL — Discrepancias de presentación

| ID | Área | Descripción |
|:---|:---|:---|
| V-01 | Pestaña "Salidas" | Check-out está embebido en modal de IN_YARD, no como pestaña propia |
| V-02 | Firma del guardia | Solo checkbox; no captura biométrica a diferencia del chofer |
| V-03 | Selector montacargas | No disponible en formulario de caseta aunque BD y BE lo soportan |

---

## 6. Endpoints REST — Cobertura

| Endpoint BE | Método FE | Estado |
|:---|:---|:---:|
| `GET /security-gate/public/catalogs` | `getPublicCatalogs()` | ✅ |
| `GET /security-gate/public/passes/{token}` | `getPublicPass(token)` | ✅ |
| `POST /security-gate/public/passes/{token}/submit` | `submitPublicDriverCheckin(token, body)` | ✅ |
| `POST /security-gate/passes/generate` | `generatePass(body)` | ✅ |
| `GET /security-gate/passes/active` | `getActivePasses()` | ✅ |
| `GET /security-gate/passes/in-yard` | `getInYardPasses()` | ✅ |
| `GET /security-gate/passes/history` | `getPassHistory()` | ✅ |
| `POST /security-gate/passes/{token}/complete` | `completePassCheckin(token, body)` | ✅ |
| `POST /security-gate/passes/{token}/check-out` | `checkOutPass(token, body)` | ✅ |
| `DELETE /security-gate/passes/{id}` | `deletePass(id, token)` | ✅ |
| `GET /security-gate/passes/{id}/audit` | ❌ NO IMPLEMENTADO | ❌ MISSING |

---

## 7. Ciclo de Vida de Estados del Pase

| Estado | Existe en BD | Manejado en FE | Descripción |
|:---|:---:|:---:|:---|
| `PENDING_DRIVER` | ✅ | ✅ | Pase generado, esperando chofer |
| `SUBMITTED` | ✅ | ✅ | Chofer completó auto-registro |
| `COMPLETED` | ✅ | ⚠️ PARTIAL | Guardia autorizó y generó folio WMS |
| `COMPLETED_EXIT` | ❌ NO EXISTE | ✅ FE Only | **BRECHA CRÍTICA I-10** |
| `CANCELLED` | ✅ | ✅ | Pase descartado |

---

## 8. Resumen de Brechas por Severidad

| Severidad | Cantidad | Descripción |
|:---|:---:|:---|
| 🔴 CRÍTICA | 1 | `COMPLETED_EXIT` status inexistente en BD — historial siempre vacío (I-10) |
| 🟡 ALTA | 9 | Naming mismatch FE ↔ BE (I-01 a I-09) — funcional por workarounds |
| 🟡 ALTA | 1 | Operador Montacargas no asignado (P-04) |
| 🟠 MEDIA | 5 | Campos MISSING sin columna en BD (M-01 a M-05) |
| 🟢 BAJA | 3 | Issues visuales de UX (V-01 a V-03) |
| 🟢 BAJA | 6 | Funcionalidad parcial con workarounds (P-01 a P-06) |

---

## 9. Recomendaciones Prioritizadas

### P1 — CRÍTICO
- **[FE]** Corregir el filtro del historial: cambiar `status === 'COMPLETED_EXIT'` por `status === 'COMPLETED' && p.exitedAt != null` en `reloadHistoryPasses()` de `SecurityGateComponent`.

### P2 — ALTO
- **[FE]** Alinear `client` → `clientName` en el objeto `checkInData` del flujo de registro directo sin QR.
- **[BE]** Añadir `noEcoTractor` al `PassResponse.java` para exponer el número económico en las respuestas del servidor.

### P3 — MEDIO
- **[FE]** Agregar selector de Operador de Montacargas en el formulario `checkInForm` del guardia.
- **[FE]** Mostrar indicador visual cuando `expiresAt < now()` en la lista de pases activos.

### P4 — BAJO
- **[BE]** Implementar `GET /security-gate/passes/{id}/audit` para trazabilidad forense completa del pase.
- **[DOCS]** Crear SDD específico para el portal del chofer (`CarrierCheckinComponent`).

---

## 10. Estado de Especificaciones

| Documento | Existe | Cubre security |
|:---|:---:|:---|
| `docs/sdd/security-gate-checkout.sdd.md` | ✅ | Checkout y F01 |
| `docs/adr/ADR-017-bimodal-security-gate-qr-driver-self-registration.md` | ✅ | Flujo bimodal QR |
| `docs/adr/ADR-018-security-gate-checkout-and-f01-transport-checklist-homologation.md` | ✅ | Homologación F01 |
| `docs/adr/ADR-019-flujo-integral-salidas-almacen-caseta-y-despacho.md` | ✅ | Ciclo de salidas |
| `docs/adr/ADR-020-flujo-integral-recepcion-almacen-caseta-y-descarga.md` | ✅ | Ciclo de recepciones |
| SDD específico para `CarrierCheckinComponent` (portal chofer) | ❌ | **FALTANTE** |

---

*Reporte generado automáticamente por Antigravity IDE — Análisis 100% READ-ONLY.*
*Ningún archivo fue modificado en Angular ni en Spring Boot.*
