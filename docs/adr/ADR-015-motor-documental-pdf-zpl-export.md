# ADR-015: Motor Documental para Generación de Boletas de Recepción, Traspasos, Despachos Outbound y Códigos Industriales (PDF & ZPL)

- **Estado:** Aceptado
- **Fecha:** 2026-09-12
- **Autores:** Equipo 4GUARD WMS (Frontend, Backend & Hardware)
- **Módulos Afectados:** `apps/admin-console`, `apps/rf-terminal`, `4guard_be`, `libs/shared-core`

---

## 1. Contexto y Problema

Las operaciones logísticas de 4GUARD WMS requieren generar dos categorías completamente distintas de artefactos físicos y digitales:

1. **Documentos Ejecutivos y de Soporte Legal (Formato Humano):**
   - Boletas de descarga / recepción de mercancía en andén (`ReceptionReceipt`).
   - Órdenes de traspaso interno entre bahías o almacenes (`TransferManifest`).
   - Hojas de embarque, cartas porte y notas de remisión de salida (`OutboundBillOfLading`).
   - Estos documentos deben tener diseño corporativo impecable (tipografía institucional, logos vectoriales, tablas con desglose de SKUs, lotes, fechas de caducidad, códigos QR de validación y firmas de conformidad).

2. **Etiquetas Térmicas Industriales (Formato Máquina):**
   - Etiquetas para tarimas (pallets) con código de barras GS1-128 / Code 128.
   - Identificadores de ubicación de racks y bahías.
   - Estas etiquetas deben imprimirse a alta velocidad en impresoras industriales Zebra en andenes mediante comandos nativos ZPL (*Zebra Programming Language*).

---

## 2. Decisión Tomada

Se adopta una **Arquitectura Híbrida de Generación y Emisión Documental**:

### 2.1 Generación de Documentos PDF en Frontend (`apps/admin-console`, `pdf-print-export.service.ts`)
* **Tecnología:** Generación directa en el cliente mediante **jsPDF** y **pdfmake / canvas**, eliminando la sobrecarga computacional de renderizado en el servidor para documentos administrativos.
* **Estándar Visual Corporativo:**
  - Paleta institucional: Cabeceras en Midnight Navy (`#172033`), acentos en Prestige Gold (`#c5a86b`) y líneas delimitadoras sutiles (`#e2e8f0`).
  - Metadatos estandarizados: Folio único correlativo, logotipo corporativo en base64, código QR con URL de verificación de autenticidad, tabla de partidas con cantidades solicitadas vs recibidas/despachadas, y recuadros de firma para operador de patio, chofer y supervisor.
* **Previsualización y Descarga:** El usuario puede abrir un modal con previsualización en tiempo real mediante `Blob URL` o disparar la impresión nativa del navegador con hojas dimensionadas en formato Carta (Letter) o A4.

### 2.2 Impresión Industrial ZPL por Socket TCP Backend (`4guard_be`)
* **Tecnología:** Emisión directa desde el backend a través de **Sockets TCP Raw (Puerto 9100)** hacia las direcciones IP asignadas a las impresoras Zebra de andén.
* **Justificación:** Los navegadores web y las PWAs operan bajo un modelo sandbox de seguridad que les prohíbe abrir sockets TCP directos hacia la red local del almacén. El backend actúa como broker autorizado para enviar ráfagas ZPL precompiladas.
* **Formato ZPL:** Plantillas estandarizadas con control milimétrico de densidad térmica (203 dpi / 300 dpi), campos de código de barras bidimensional (Datamatrix / QR) y unidimensional (Code 128 con dígito verificador).

---

## 3. Consecuencias

### Positivas
- **Descarga Inmediata sin Latencia:** Los PDFs se procesan en milisegundos en el cliente, evitando cuellos de botella en el servidor de Spring Boot al imprimir cientos de boletas diarias.
- **Confiabilidad en Piso de Operación:** La impresión ZPL por socket backend garantiza etiquetas nítidas, resistentes y legibles para los escáneres láser de las terminales handheld.
- **Diseño Corporativo Unificado:** Tanto las boletas PDF como las etiquetas ZPL comparten los mismos identificadores de lote, folio y códigos de barras.

### Compromisos
- Requiere mantener sincronizadas las direcciones IP de las impresoras Zebra en el catálogo de configuración de sucursales del backend.
- El bundle de frontend incluye las librerías de generación de PDF, debiendo ser cargadas perezosamente (*Lazy Loading*) solo al ingresar a módulos de exportación.
