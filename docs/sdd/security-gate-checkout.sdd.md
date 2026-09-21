# SDD: Centro de Control de Caseta, Ciclo de Salida (Check-Out) y Formato F01-PO-CP-7.1.3-03

- **Sistema:** 4GUARD WMS
- **Módulo:** Vigilancia & Caseta de Seguridad (`apps/admin-console/src/app/features/security/security-gate`)
- **Fecha:** 2026-09-17
- **Estado:** Implementado

---

## 1. Arquitectura de Navegación y Pestañas

```
                                  [ CENTRO DE CONTROL DE VIGILANCIA ]
                                                 │
          ┌──────────────────────┬───────────────┴───────────────┬──────────────────────┐
          ▼                      ▼                               ▼                      ▼
  [ 📋 1. NUEVO REGISTRO ] [ 🚛 2. EN PLANTA ]         [ 🏁 3. CHECK-OUT ]     [ 🗄️ 4. HISTORIAL F01 ]
  - Generador QR Pase      - Monitoreo en Rampas       - Filtro "Listo Salida" - Buscador en vivo
  - Choferes en Espera     - Status Almacén en Vivo    - Captura Hora Salida   - Reimpresión directa
  - Formulario F01 Limpio  - Botón Acción Directa      - Sellos de Salida      - Auditoría Inmutable
```

---

## 2. Especificación del Formato Oficial F01-PO-CP-7.1.3-03

| Sección Formato F01 | Componente / Campos Mapeados | Comportamiento |
| :--- | :--- | :--- |
| **Cabecera Institucional** | Logo 4Guard, No. Control `F01-PO-CP-7.1.3-03`, Rev. `01` (19/08/2025), Dueño `Seguridad Patrimonial`, Aprobó `DG` | Estático institucional |
| **Datos Generales** | `fecha`, `noCartaPorte`, `remision`, `cliente`, `operacion` (Carga/Descarga), `horaEntrada`, `horaSalida` | Dinámico del registro |
| **Datos del Transporte** | `lineaTransporte`, `nombreOperador`, `noRampa`, `placasTracto`, `noEcoTractor`, `placasCaja`, `medidasCaja`, `noSello`, `tipoTransporte` | Captura / Auto-registro |
| **Checklist EPP** | `eppZapatos`, `eppCofia`, `eppCubrebocas`, `eppChaleco` con Sí/No y observaciones manuales | Checkbox interactivo |
| **Checklist Documentos** | `docCartaPorte`, `docRemision` con Sí/No y observaciones | Checkbox interactivo |
| **Revisión Unidad** | `revInteriorCaja`, `revDanosCaja`, `revDanosPuertas`, `revOloresExtranos`, `revIndiciosPlagas` | Checkbox interactivo |
| **Firmas y Sellos** | Sello Digital Vigilancia + Firma Digital Canvas Chofer | Renderizado Base64 |

---

## 3. Contratos de API Backend

- `GET /api/v1/security-gate/passes/in-yard` $\rightarrow$ Lista unidades en planta con `warehouseStatus` e `isReadyForExit`.
- `GET /api/v1/security-gate/passes/history?search=...` $\rightarrow$ Lista histórico con filtros.
- `POST /api/v1/security-gate/passes/{token}/check-out` $\rightarrow$ Registra `departureTime`, `exitObservations` y actualiza a `COMPLETED_EXIT`.
