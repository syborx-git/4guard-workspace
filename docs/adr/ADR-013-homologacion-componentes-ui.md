# ADR-013: Estándar Universal de Homologación de Componentes de UI (Data Tables, Selects Dinámicos, Datepickers Industriales, Modales Desacoplados y Hero Header)

- **Estado:** Aceptado
- **Fecha:** 2026-09-12
- **Autores:** Equipo 4GUARD WMS (Frontend & Architecture)
- **Módulos Afectados:** `apps/admin-console`, `apps/rf-terminal`, `libs/shared-core`, `docs/design`, `docs/sdd`

---

## 1. Contexto y Problema

A medida que el sistema **4GUARD WMS** ha crecido en número de módulos administrativos y operativos (Transportistas, Clientes, Turnos, Alertas, Divisas, Ubicaciones, Movimientos de Almacén, Licencias, Usuarios, Sucursales y Proveedores), se identificaron inconsistencias visuales y de comportamiento cuando diferentes desarrolladores o asistentes de IA implementaban componentes comunes:

1. **Tablas de Datos Heterogéneas:** Distintas formas de renderizar cabeceras, falta de ordenamiento homogéneo, ausencia de barras de paginación uniformes, o estilos dispersos en celdas de acciones y estatus.
2. **Selectores Desalineados:** Uso de selects HTML nativos sin estilizar frente a selectores personalizados, inconsistencia en el manejo de placeholders, alturas variables y falta de soporte de búsqueda para catálogos extensos.
3. **Datepickers No Estandarizados:** Captura de fechas en formatos mixtos (`YYYY-MM-DD` vs `DD/MM/YYYY`), popovers que rompían el tema oscuro/claro y falta de accesos rápidos (*Hoy*, *Últimos 7 días*, etc.).
4. **Colisión en Modales y Formularios:** Inserción de etiquetas anidadas `<form>` dentro de modales que provocaban recargas accidentales de la página o envíos no deseados del formulario padre.
5. **Cabeceras Dispersas:** Pantallas sin enlace directo de retorno al Hub de Administración (`/admin`), o con variaciones de títulos, subtítulos y badges de categoría.

---

## 2. Decisión Tomada

Se establece la **Normativa Obligatoria de Homologación de Componentes de UI** para todas las aplicaciones del monorepo (`admin-console` y `rf-terminal`), soportada por el **Synexia Theme Engine** (`.theme-dark` y `.theme-light`).

Todo componente nuevo o refactorizado DEBE cumplir con las siguientes especificaciones estructurales y de diseño:

### 2.1 Data Tables (`.table-container`, `.data-table`)
* **Contenedor:** Toda tabla debe residir en un contenedor `.table-container` con borde `$border-subtle`, esquinas redondeadas (`$radius-md` / 12px) y fondo `$surface`.
* **Cabecera (`thead`):** Fondo `$bg-primary`, tipografía `$font-ui`, padding `10px 16px`, texto mayúsculas sutil, soporte para columnas ordenables con clase `.sortable` e indicadores de dirección (`.sort-asc`, `.sort-desc`) en color `$gold-light`.
* **Filas y Hover (`tbody tr`):** Fondo dinámico `var(--bg-card)`, borde inferior `var(--border-card)`, hover suave con tinte dorado `rgba(197, 168, 107, 0.06)`, y estado seleccionado con borde izquierdo dorado de 2px (`border-left: 2px solid var(--gold)`).
* **Tipos de Celda Estándar:**
  - `.td-primary`: Texto principal en color `$text-primary` con peso medio.
  - `.td-mono`: Códigos, RFC, UUIDs, IPs con fuente `'JetBrains Mono'`.
  - `.td-status`: Centrado o alineado con el `status-badge` oficial (`.status-badge--active`, `.status-badge--inactive`, etc.).
  - `.td-actions`: Contenedor flex con espacio de 4px para botones de acción rápida de 32x32px (`.btn-icon`).
* **Paginación Integrada (`.table-pagination`):** Barra inferior con resumen de registros (*Mostrando 1 a 10 de 145*), selector de tamaño de página (10, 25, 50, 100), botones de anterior/siguiente y páginas numeradas con estado activo dorado.
* **Estados Vacío y Esqueleto:** Inclusión obligatoria de `.table-empty` con icono Material y mensaje contextual, así como filas esqueleto animadas con shimmer (`.table-skeleton-row`) durante la carga.

### 2.2 Selectores y Dropdowns (`.form-select`, `.fg-select-searchable`)
* **Altura y Bordes:** Altura fija de 40px (`$min-height-input`), border-radius de 8px (`$radius-sm`), padding `0 16px`.
* **Flecha Decorativa:** Flecha SVG estilizada en color `$text-tertiary` integrada como `background-image` en la esquina derecha.
* **Enfoque:** Resplandor dorado corporativo sin outline nativo: `border-color: var(--gold); box-shadow: 0 0 0 3px rgba(234, 195, 73, 0.12);`.
* **Catálogos Extensos:** Para listas de más de 8 opciones (ej. Clientes, Operadores, SKUs, Sucursales), debe emplearse el patrón *Searchable Dropdown* con input de búsqueda interactivo y filtrado en tiempo real.

### 2.3 Datepickers Industriales (`.fg-datepicker`)
* **Formato de Presentación:** Siempre `DD/MM/YYYY` en la interfaz de usuario con icono Material `calendar_today` en el costado derecho o izquierdo.
* **Persistencia en API:** Obligatoriamente transformado a formato **ISO-8601 UTC** (`YYYY-MM-DD` o `YYYY-MM-DDTHH:mm:ssZ`) en los DTOs de petición/respuesta según **ADR-008**.
* **Rangos de Fecha:** Validación reactiva estricta de no inversión (`fechaInicio <= fechaFin`). Soporte para chips de selección rápida: *Hoy*, *Ayer*, *Últimos 7 días*, *Este Mes*, *Personalizado*.
* **Adaptabilidad Temática:** Popovers de calendario estilizados con variables CSS del tema, evitando fondos blancos no adaptables en modo oscuro.

### 2.4 Modales Desacoplados y Diálogos de Confirmación
* **Prohibición de `window.confirm()` y `window.alert()`:** Toda acción destructiva (eliminar, revocar, suspender, cambiar estado irreversible) debe utilizar el componente homologado `<fg-confirm-dialog>`.
* **Prevención de Envíos Accidentales:** Los modales deben encapsularse en contenedores `<div>` (ej. `.dialog`, `.dialog__form`) sin etiqueta `<form>` interna para evitar colisiones de submit con el formulario de la página padre.
* **Estructura:** Backdrop oscuro con desenfoque (`backdrop-filter: blur(4px)`), tarjeta con línea decorativa superior dorada, cabecera con botón de cierre, cuerpo scrollable y barra de acciones sticky inferior.

### 2.5 Hero Header con Navegación `/admin` (`.hero-header`)
* Toda pantalla de gestión administrativa DEBE contener el Hero Header Golden Standard:
  1. Ícono Midnight Navy de 52x52px con esquinas redondeadas de 14px y sombra suave.
  2. Breadcrumb superior compuesto por el botón badge `.btn-back-admin` con enlace a `/admin` y la categoría del módulo en tipografía monospace dorada en mayúsculas (`.hero-header__eyebrow`).
  3. Título H1 (1.7rem - 2.1rem) en tipografía de visualización (`Outfit` / `Inter`).
  4. Subtítulo descriptivo en color `$text-secondary`.

### 2.6 Patrón de Distribución Split-View (35% / 65%)
* Módulos de administración de catálogos y entidades deben priorizar el layout **Split-View**:
  - Panel Izquierdo (320px - 380px): Directorio con buscador global, filtros por chip/select, lista reactiva con estados esqueleto y contador total.
  - Panel Derecho (Flex 1): Cabecera contextual, formulario por secciones con leyendas doradas (`.section-legend`), visualizador de auditoría timeline y barra de acciones sticky.

---

## 3. Consecuencias

### Positivas
- **Homogeneidad Absoluta:** Cualquier usuario u operador experimenta una interfaz visualmente coherente, intuitiva y predecible en todo el WMS.
- **Mantenibilidad:** Los estilos residen en tokens y componentes centralizados de SCSS (`_table.scss`, `_input.scss`, `_dialog.scss`), reduciendo la duplicidad de CSS en componentes individuales.
- **Cero Efectos Colaterales:** Desaparecen los errores de recarga por submits accidentales de formularios en diálogos modales.
- **Alineación con IA:** Los asistentes de programación disponen de un estándar inequívoco y comprobable para generar o auditar código.

### Compromisos
- Exige refactorizar pantallas que aún utilizaban tablas o controles HTML planos fuera del estándar.
- Requiere pruebas de accesibilidad y contraste cromático en ambos temas (`.theme-dark` y `.theme-light`).
