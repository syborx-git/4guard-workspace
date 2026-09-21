# ADR-018: Homologación del Ciclo de Salida (Check-Out) en Caseta, Centro de Control Multi-Pestaña y Formato Oficial F01-PO-CP-7.1.3-03

- **Estado:** Aceptado
- **Fecha:** 2026-09-17
- **Autores:** Equipo 4GUARD WMS (Seguridad Patrimonial, Almacén, Frontend & Backend)
- **Módulos Afectados:** `apps/admin-console`, `4guard_be`, `libs/shared-core`
- **ADRs Relacionados:** [ADR-015 (Motor Documental PDF & ZPL)](./ADR-015-motor-documental-pdf-zpl-export.md), [ADR-017 (Auto-Registro QR Bimodal Chofer)](./ADR-017-bimodal-security-gate-qr-driver-self-registration.md)

---

## 1. Contexto y Problema

El control patrimonial y la operación de patio de 4GUARD WMS requerían resolver tres aspectos críticos:
1. **Fidelidad Absoluta al Formato Físico de la Empresa (`F01-PO-CP-7.1.3-03` / `CHECK LIST DE TRANSPORTE`):**
   La empresa cuenta con un formato físico oficial emitido por Seguridad Patrimonial. Al completarse la operación, se debe emitir un documento idéntico con sus metadatos (Revisión 01, Fecha 19/08/2025, Aprobó DG, etc.), tablas de EPP, documentación, revisión de unidad y firmas digitales.
2. **Ciclo de Retorno y Check-Out de Caseta:**
   Cuando el área operativa de Recepción o Despacho completa y autoriza la maniobra en andén, la unidad debe retornar a la supervisión de Caseta de Seguridad para registrar la **Hora de Salida**, verificar sellos finales y cerrar el ciclo con auditoría inmutable (`SALIDA_VEHICULO_REGISTRADA`).
3. **Ergonomía UI/UX en Caseta (Vigilancia):**
   Un formulario único sobrecargado genera fricción operativa al gestionar múltiples unidades simultáneas. Se requería una arquitectura por pestañas de trabajo (*Tabs*) y que los campos de observaciones inicien **vacíos por defecto** para llenado manual veraz.

---

## 2. Decisión Tomada

### 2.1 Replicación del Formato Oficial F01 (`PrintTransportChecklistLayoutComponent`)
* **Homologación con ADR-015:** Se utiliza `PrintService` para renderizado en cliente en alta resolución (2x DPI) mediante `html2canvas` y `jsPDF`.
* **Componente Oficial:** `PrintTransportChecklistLayoutComponent` replica de forma idéntica:
  - Encabezado con logo 4Guard, No. de Control `F01-PO-CP-7.1.3-03`, Rev. `01` y autorizaciones (`SP`, `CG`, `DG`).
  - Tabla 1: Datos Generales (Fecha, Carta Porte, Remisión, Cliente, Procedimiento Carga/Descarga, Horas de Entrada y Salida).
  - Tabla 2: Datos del Transporte (Línea, Operador, Rampa, Placas tracto/caja, No. Económico, Medidas y Sellos).
  - Tabla 3: Revisión del Transporte (Checklist EPP, Documentos y Unidad con marcas Sí/No y observaciones manuales).
  - Tabla 4: Responsables y Firmas (Sello de Vigilancia y Firma Digital capturada del Chofer).

### 2.2 Ciclo Completo de Salida (Check-Out)
* **Retorno Automático a Caseta:** Al autorizar el Líder de Almacén una recepción (`COMPLETED`) o salida (`DISPATCHED`), el pase pasa a estado `isReadyForExit = true`.
* **Endpoint de Check-Out:** `POST /api/v1/security-gate/passes/{token}/check-out` registra la `departureTime`, sellos de salida y observaciones finales, transicionando el pase a `COMPLETED_EXIT`.
* **Trazabilidad y Auditoría:** Se registran eventos relacionales inmutables en `wms.audit_logs`.

### 2.3 Dashboard UI/UX en 4 Pestañas de Trabajo
1. **📋 1. Nuevo Registro & Pases QR:** Generador de QR dinámico para chofer, cola de auto-registros móviles recibidos y captura manual rápida con campos limpios.
2. **🚛 2. Unidades en Planta / Rampa:** Monitoreo en vivo de unidades operando en andenes con estatus de almacén (`REGISTRADO`, `EN DESCARGA`, `DESCARGA FINALIZADA`).
3. **🏁 3. Salidas Pendientes (Check-Out):** Unidades autorizadas por Almacén listas para registrar hora de salida y boleta F01.
4. **🗄️ 4. Historial & Auditoría F01:** Buscador en vivo por folio, chofer o placas con botón de reimpresión directa.
