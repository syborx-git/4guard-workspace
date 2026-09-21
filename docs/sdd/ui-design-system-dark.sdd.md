# SDD — Estándar de Diseño Visual & UI/UX (4GUARD WMS Dark Mode)

**Proyecto:** 4GUARD WMS  
**Documento:** Especificación Técnica de Diseño Visual Homologado (Dark Mode Enterprise)  
**Tipo:** Estándar de Arquitectura UI/UX & Design Tokens  
**Estado:** Activo & Obligatorio para Todos los Módulos  
**Versión:** 2.2 (Homologación Unificada Dashboard + Inventario + Recepciones)  
**Ámbito:** Modo Oscuro (*Dark Mode Enterprise Luxury / Obsidian Slate & Prestige Gold*).  
**Regla de Portabilidad:** Prohibidas rutas absolutas del SO (`c:/Users/...`). Todos los imports y referencias deben ser relativos (`../../`) o usar alias (`@4guard/shared-core`).

---

## 1. Filosofía Visual y Armonía con Light Mode

El **Modo Oscuro** de 4GUARD WMS está concebido como una extensión simétrica, fluida y de alta legibilidad del Modo Claro:

1. **Cero Fondos Planos:** El canvas principal utiliza un fondo Obsidian/Midnight Navy `#0b1119` con un suave gradiente radial dorado `rgba(208, 175, 103, 0.05)` en la esquina superior derecha.
2. **Superficies y Tarjetas Liquid Glass:**
   - Tarjetas principales: `background: rgba(17, 26, 38, 0.94)` con borde translúcido `rgba(255, 255, 255, 0.08)`.
   - Hover: `background: rgba(23, 35, 50, 0.98)` con borde dorado sutil `rgba(208, 175, 103, 0.25)`.
3. **Tipografía Jerárquica de Alto Contraste:**
   - **Títulos y Cifras Principales:** Blanco platino `#edf1f5` (`'Outfit'`, `'Inter'`).
   - **Subtítulos y Textos Secundarios:** Gris pizarra frío `#9aa6b4` (alta legibilidad sobre fondos oscuros, cero fatiga visual).
   - **Metadatos y Desactivados:** `#667585`.
   - **Identificadores Técnicos (Mono):** `#d0af67` o `#edf1f5` (`'JetBrains Mono'`).
4. **Semáforos y Acentos Operativos:**
   - **Gold Prestige:** `#d0af67` (Acento institucional primario, bordes e iconos clave).
   - **Éxito / Disponible:** Verde esmeralda vivo `#67c78b` sobre fondo `rgba(76, 175, 112, 0.12)`.
   - **Advertencia / Cuarentena:** Ámbar cálido `#e0aa4f` sobre fondo `rgba(224, 170, 79, 0.12)`.
   - **Crítico / Bloqueado:** Rojo coral `#ef7773` sobre fondo `rgba(239, 83, 80, 0.12)`.
   - **Información / Picking:** Azul zafiro `#77a8df` sobre fondo `rgba(96, 165, 250, 0.12)`.

---

## 2. Paleta de Colores & Design Tokens (Dark Mode)

Todo componente debe implementar el selector `:host-context(.theme-dark), :host-context(.dark)` con los siguientes tokens:

```css
:host-context(.theme-dark),
:host-context(.dark) {
  /* ── Canvas & Superficies ── */
  --bg-page:        #0b1119;
  --bg-card:        rgba(17, 26, 38, 0.94);
  --bg-card-hover:  rgba(23, 35, 50, 0.98);
  --border-card:    rgba(255, 255, 255, 0.08);
  --shadow-card:    0 16px 36px rgba(0, 0, 0, 0.40), inset 0 1px 0 rgba(255, 255, 255, 0.05);

  /* ── Textos ── */
  --text-primary:   #edf1f5;
  --text-secondary: #9aa6b4;
  --text-muted:     #667585;
  --text-gold:      #d0af67;

  /* ── Marca Institucional ── */
  --navy:           #dfe7ef;
  --navy-mid:       #16212f;
  --navy-light:     #1f2d3e;
  --gold:           #d0af67;
  --gold-light:     #e1c47c;
  --gold-bg:        rgba(208, 175, 103, 0.12);
  --gold-border:    rgba(208, 175, 103, 0.25);

  /* ── Estados & Semáforos ── */
  --c-success:      #67c78b;
  --c-success-bg:   rgba(76, 175, 112, 0.12);
  --c-success-bdr:  rgba(76, 175, 112, 0.24);
  --c-warning:      #e0aa4f;
  --c-warning-bg:   rgba(224, 170, 79, 0.12);
  --c-warning-bdr:  rgba(224, 170, 79, 0.24);
  --c-danger:       #ef7773;
  --c-danger-bg:    rgba(239, 83, 80, 0.12);
  --c-danger-bdr:   rgba(239, 83, 80, 0.24);
  --c-info:         #77a8df;
  --c-info-bg:      rgba(96, 165, 250, 0.12);
  --c-info-bdr:     rgba(96, 165, 250, 0.24);

  color: var(--text-primary);
  background:
    radial-gradient(circle at 90% 5%, rgba(208, 175, 103, 0.05), transparent 30rem),
    var(--bg-page);
}
```

---

## 3. Módulos Adaptados

1. **Dashboard Torre de Control (`/dashboard`):**
   - Canvas Midnight Navy `#0b1119` integrado con el Sidebar y Header.
   - Tarjetas Bento Grid oscuras con bordes nítidos y sombras envolventes.
   - Tipografía en blanco platino `#edf1f5` y gris pizarra `#9aa6b4`.
   - Anillos de saturación y medidor de riesgo con trazos oscuros `rgba(255, 255, 255, 0.08)` y arcos en dorado `#d0af67`.
   - Modal de desglose por bodega homologado con superficies oscuras y botones dorados.

2. **Inventario & Mapa 2D de Topología Cromática (`/inventory`):**
   - Grid de 8 columnas interactivo con bahías que respetan los semáforos de saturación sin deslumbrar en modo oscuro.
   - Bahías de baja saturación adaptadas a carbón cálido `#201c18` con texto en oro `#e1c47c`.
   - Barra de almacenes reales y leyenda superior integradas en glassmorphism con chips informativos dorados.
   - Panel lateral flotante de detalle de bahía (`.bay-detail`) con métricas, barras de ocupación y alertas QM.

3. **Consulta de Inventarios & Exportación Excel (`/inventory-query`):**
   - KPI Summary superior de 4 columnas en Liquid Glass con tipografía en mono y chips semánticos.
   - Toolbar de acciones con buscador reactivo, filtros multicriterio, analytics y exportación a Excel oficial (21 columnas).
   - Tabla ejecutiva de existencias con encabezado contrastado `#0b1119`, hover sutil dorado y badges de semáforo de caducidad (En tiempo, Próximo <30d, Caduco).
   - Modal de Filtros Multicriterio (`fg-inventory-query-filter-modal`) con radio buttons de Regla de Pablo y dropdowns oficiales.
   - Modal de Analítica Histórica (`fg-inventory-analytics-modal`) con trazabilidad por secuencias de folios y gráfica de barras animada por año.

4. **Centro de Recepciones & Wizard Bóveda (`/receiving`):**
   - **Centro de Recepciones (`ReceivingCenterComponent`):** Hero con mini resumen ejecutivo táctico, tarjeta de recepción en curso con pulso dorado, toolbar de búsqueda y tabla maestra enriquecida con badges de estado y transportista.
   - **Preparación de Expediente (`ReceptionCreateComponent`):** Header de expediente con migas de pan y selector de fase, tarjetas de resumen, checklists de validación y tabla de líneas de orden de compra en liquid glass.
   - **La Bóveda: Wizard de Recepción (`ReceivingWizardComponent`):** Stepper secuencial de recepción con círculos contrastados en oro, tabla de líneas de inspección y conteo ciego, panel cuádruple y escáner de códigos de barras.
   - **Drawers Laterales (`DockAssignmentDrawerComponent` & `PurchaseOrderDetailDrawerComponent`):** Paneles desplegables con backdrop blur, grid de tarjetas de rampa, visor documental y soporte de adjuntos PDF.
