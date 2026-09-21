# SDD — Estándar de Diseño Visual & UI/UX (4GUARD WMS Light Mode)

**Proyecto:** 4GUARD WMS  
**Documento:** Especificación Técnica de Diseño Visual Homologado (Light Mode)  
**Tipo:** Estándar de Arquitectura UI/UX & Design Tokens  
**Estado:** Activo & Obligatorio para Todos los Módulos  
**Versión:** 2.0 (Homologación Unificada)  
**Ámbito:** Modo Claro (*Light Mode Enterprise Luxury / Minimalist Industrial*).  
**Regla de Portabilidad:** Prohibidas rutas absolutas del SO (`c:/Users/...`). Todos los imports y referencias deben ser relativos (`../../`) o usar alias (`@4guard/shared-core`).

---

## 1. Filosofía Visual y Principios Clave

Todo módulo en **4GUARD WMS** debe respetar una identidad visual uniforme, ergonómica y de alto impacto estético:

1. **Cero Fondos Planos:** El canvas principal utiliza un gradiente lino/marfil cálido con un brillo radial dorado muy sutil.
2. **Tipografía Jerárquica Homologada:**
   - **Display / Títulos / Cifras:** `'Outfit', 'Inter', system-ui, sans-serif`
   - **Cuerpo / Etiquetas / Tablas:** `'Inter', system-ui, sans-serif`
   - **Identificadores Técnicos:** `'JetBrains Mono', 'Fira Code', monospace` (Folios, SSCC, Lotes, UAs, Rampas, UUIDs).
3. **Ergonomía de Cabeceras (Header + Toolbar Unificado):**
   - Fila 1: Icono Midnight Navy + Eyebrow institucional con link de retroceso al Dashboard + Título H1 + Subtítulo.
   - Fila 2 (Barra de Control): **Slide de Pestañas (Pill Nav)** a la izquierda + **Botón de Acción Principal Dorado** a la derecha.
4. **Regla de Signo `+` en Botones:**  
   Los botones de acción que agreguen registros deben usar el icono Material `<span class="material-symbols-outlined">add</span>` y texto limpio sin caracteres `+` literales (`<span>Nuevo Registro</span>`), evitando el error de doble signo (`+ +`).
5. **Métricas KPI Horizontales (4 Columnas):**  
   Cuadrícula de 4 tarjetas inmediatamente debajo de la cabecera, con iconos en tonos pastel y números destacados.
6. **Layout Master-Detail de 12 Columnas:**  
   Directorio lateral en `col-span-4` y área de trabajo/detalle en `col-span-8`.

---

## 2. Paleta de Colores & Design Tokens Globales

Todos los módulos deben definir o consumir las siguientes variables CSS:

```css
:host {
  display: block;
  width: 100%;
  min-height: 100%;

  /* ── Marca Institucional ── */
  --navy:           #172033;
  --navy-mid:       #25324a;
  --navy-light:     #344463;
  --gold:           #c5a86b;
  --gold-light:     #e0c87a;
  --gold-bg:        rgba(197, 168, 107, 0.10);
  --gold-border:    rgba(197, 168, 107, 0.28);

  /* ── Canvas & Superficies ── */
  --bg-page:        #f5f4f0;
  --bg-card:        rgba(255, 255, 255, 0.92);
  --border-card:    rgba(76, 86, 105, 0.10);
  --shadow-card:    0 12px 32px rgba(36, 44, 58, 0.08), inset 0 1px 0 rgba(255, 255, 255, 0.85);

  /* ── Textos ── */
  --text-primary:   #1c2940;
  --text-secondary: #5a6477;
  --text-muted:     #8b94a3;
  --text-gold:      #9b7626;

  /* ── Estados & Semáforos ── */
  --c-success:      #208457;
  --c-success-bg:   rgba(32, 132, 87, 0.09);
  --c-success-bdr:  rgba(32, 132, 87, 0.20);
  --c-warning:      #a96b13;
  --c-warning-bg:   rgba(213, 145, 39, 0.10);
  --c-warning-bdr:  rgba(213, 145, 39, 0.22);
  --c-danger:       #c84949;
  --c-danger-bg:    rgba(200, 73, 73, 0.09);
  --c-danger-bdr:   rgba(200, 73, 73, 0.20);
  --c-info:         #2d7dd2;
  --c-info-bg:      rgba(45, 125, 210, 0.08);
  --c-info-bdr:     rgba(45, 125, 210, 0.20);

  /* ── Tipografía ── */
  --font-display:   'Outfit', 'Inter', system-ui, sans-serif;
  --font-body:      'Inter', system-ui, sans-serif;
  --font-mono:      'JetBrains Mono', 'Fira Code', monospace;

  /* ── Radios ── */
  --radius-card:    18px;
  --radius-input:   10px;
  --radius-badge:   99px;
  --radius-btn:     10px;
}
```

---

## 3. Estructura de Pantalla Estándar (Plantilla de Módulo)

### 3.1 HTML Estándar del Componente de Módulo

```html
<div class="module-page">

  <!-- ═══════════════════════════════════════════════════════════════════
       1. CABECERA PRINCIPAL & BARRA DE CONTROL
       ════════════════════════════════════════════════════════════════ -->
  <header class="module-header">
    
    <!-- Fila 1: Icono, Eyebrow, Título y Subtítulo -->
    <div class="module-header__top">
      <div class="module-header__left">
        <div class="module-header__icon-wrap">
          <span class="material-symbols-outlined module-header__icon">inventory_2</span>
        </div>

        <div class="module-header__copy">
          <div class="module-header__eyebrow">
            <a routerLink="/dashboard" class="module-header__eyebrow-tag">
              <span class="material-symbols-outlined" style="font-size: 14px;">arrow_back</span>
              <span>Dashboard</span>
            </a>
            <span class="module-header__eyebrow-sep">·</span>
            <span>Operaciones & Logística</span>
          </div>
          <h1 class="module-header__title">[Nombre del Módulo]</h1>
          <p class="module-header__subtitle">
            [Descripción funcional y concisa del módulo en tiempo real].
          </p>
        </div>
      </div>
    </div>

    <!-- Fila 2: Slide de Pestañas (Izquierda) + Botón Acción Principal (Derecha) -->
    <div class="module-toolbar-row">
      <!-- Si el módulo contiene submódulos / tabs de navegación -->
      <nav class="module-tabs-nav" aria-label="Navegación del módulo">
        <a
          routerLink="/modulo/submodulo-1"
          routerLinkActive="tab-active"
          [routerLinkActiveOptions]="{ exact: false }"
          class="module-tab-btn"
        >
          <span class="material-symbols-outlined">move_to_inbox</span>
          <span>1. Submódulo Uno</span>
        </a>

        <a
          routerLink="/modulo/submodulo-2"
          routerLinkActive="tab-active"
          [routerLinkActiveOptions]="{ exact: false }"
          class="module-tab-btn"
        >
          <span class="material-symbols-outlined">compare_arrows</span>
          <span>2. Submódulo Dos</span>
        </a>
      </nav>

      <!-- Botón de Acción Principal (Solo 1 signo '+' generado por el icono) -->
      <button
        type="button"
        (click)="startPrimaryAction()"
        class="btn-primary-gold"
        aria-label="Registrar nueva operación"
      >
        <span class="material-symbols-outlined">add</span>
        <span>Nueva Operación</span>
      </button>
    </div>

  </header>

  <!-- ═══════════════════════════════════════════════════════════════════
       2. TARJETAS DE MÉTRICAS KPI (4 COLUMNAS)
       ════════════════════════════════════════════════════════════════ -->
  <div class="module-kpi-grid">
    <!-- Total / General -->
    <div class="module-kpi-card module-kpi-card--primary">
      <div class="module-kpi-card__icon">
        <span class="material-symbols-outlined">analytics</span>
      </div>
      <div class="module-kpi-card__info">
        <span class="module-kpi-card__label">TOTAL REGISTROS</span>
        <strong class="module-kpi-card__value">{{ totalCount() }}</strong>
        <span class="module-kpi-card__sub">En sistema / General</span>
      </div>
    </div>

    <!-- Pendiente / Proceso -->
    <div class="module-kpi-card module-kpi-card--warning">
      <div class="module-kpi-card__icon">
        <span class="material-symbols-outlined">pending_actions</span>
      </div>
      <div class="module-kpi-card__info">
        <span class="module-kpi-card__label">EN PROCESO</span>
        <strong class="module-kpi-card__value">{{ pendingCount() }}</strong>
        <span class="module-kpi-card__sub">En andén / Espera</span>
      </div>
    </div>

    <!-- Completado / Exitoso -->
    <div class="module-kpi-card module-kpi-card--success">
      <div class="module-kpi-card__icon">
        <span class="material-symbols-outlined">check_circle</span>
      </div>
      <div class="module-kpi-card__info">
        <span class="module-kpi-card__label">COMPLETADOS</span>
        <strong class="module-kpi-card__value">{{ completedCount() }}</strong>
        <span class="module-kpi-card__sub">Procesados con éxito</span>
      </div>
    </div>

    <!-- Bloqueado / Cancelado -->
    <div class="module-kpi-card module-kpi-card--danger">
      <div class="module-kpi-card__icon">
        <span class="material-symbols-outlined">cancel</span>
      </div>
      <div class="module-kpi-card__info">
        <span class="module-kpi-card__label">BLOQUEADOS</span>
        <strong class="module-kpi-card__value">{{ blockedCount() }}</strong>
        <span class="module-kpi-card__sub">Rechazados / En ceros</span>
      </div>
    </div>
  </div>

  <!-- ═══════════════════════════════════════════════════════════════════
       3. ÁREA PRINCIPAL MASTER-DETAIL (12 COLUMNAS)
       ════════════════════════════════════════════════════════════════ -->
  <div class="grid grid-cols-1 lg:grid-cols-12 gap-5 items-start">
    
    <!-- Columna Izquierda: Directorio & Búsqueda (4 Columnas) -->
    <aside class="module-directory lg:col-span-4" aria-label="Directorio">
      <!-- Buscador y Filtros -->
      <div class="module-directory__filters">
        <div class="module-search">
          <span class="material-symbols-outlined module-search__icon">search</span>
          <input
            type="search"
            class="module-search__input"
            [ngModel]="searchQuery()"
            (ngModelChange)="searchQuery.set($event)"
            placeholder="Buscar por folio, cliente, lote..."
          />
        </div>
      </div>

      <!-- Lista de Items -->
      <div class="module-directory__list">
        @for (item of filteredItems(); track item.id) {
          <div
            (click)="selectItem(item)"
            [class.item--selected]="selectedItem()?.id === item.id"
            class="module-directory-item"
          >
            <!-- Detalle breve -->
          </div>
        }
      </div>
    </aside>

    <!-- Columna Derecha: Panel de Trabajo / Detalle (8 Columnas) -->
    <main class="lg:col-span-8 space-y-4">
      <!-- Contenido / Formularios / Tablas de detalle -->
    </main>

  </div>

</div>
```

---

### 3.2 CSS Estándar del Componente de Módulo

```css
/* ── PÁGINA ── */
.module-page {
  display: flex;
  flex-direction: column;
  min-height: 100%;
  width: 100%;
  padding: 1.75rem 2rem;
  gap: 1.25rem;
  font-family: var(--font-body);
  color: var(--text-primary);
  background:
    radial-gradient(circle at 90% 5%, rgba(197, 168, 107, 0.07), transparent 30rem),
    var(--bg-page);
  overflow-x: hidden;
}

/* ── HEADER ── */
.module-header {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.module-header__top {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1.25rem;
  flex-wrap: wrap;
}

.module-header__left {
  display: flex;
  align-items: flex-start;
  gap: 1.1rem;
}

.module-header__icon-wrap {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 54px;
  height: 54px;
  border-radius: 14px;
  background: linear-gradient(145deg, var(--navy), var(--navy-mid));
  box-shadow: 0 8px 20px rgba(23, 32, 51, 0.22);
  flex-shrink: 0;
}

.module-header__icon {
  color: #ffffff;
  font-size: 26px;
}

.module-header__copy {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.module-header__eyebrow {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.68rem;
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: var(--text-gold);
}

.module-header__eyebrow-tag {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-family: var(--font-mono);
  background: var(--gold-bg);
  border: 1px solid var(--gold-border);
  border-radius: 5px;
  padding: 1px 7px;
  text-decoration: none;
  color: var(--text-gold);
  cursor: pointer;
  transition: background 0.15s;
}

.module-header__eyebrow-tag:hover {
  background: rgba(197, 168, 107, 0.20);
}

.module-header__eyebrow-sep {
  opacity: 0.4;
}

.module-header__title {
  font-family: var(--font-display);
  font-size: 1.7rem;
  font-weight: 500;
  letter-spacing: -0.025em;
  color: var(--text-primary);
  margin: 0;
  line-height: 1.15;
}

.module-header__subtitle {
  font-size: 0.82rem;
  color: var(--text-secondary);
  margin: 0;
}

/* ── BARRA DE CONTROL (SLIDE DE PESTAÑAS + BOTÓN ACCIÓN) ── */
.module-toolbar-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  flex-wrap: wrap;
}

/* ── SLIDE DE PESTAÑAS (PILL NAV) ── */
.module-tabs-nav {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  background: #ffffff;
  border: 1px solid var(--border-card);
  padding: 0.35rem;
  border-radius: 14px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
}

.module-tab-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.45rem;
  padding: 0.45rem 0.95rem;
  border-radius: 10px;
  font-family: var(--font-body);
  font-size: 0.8rem;
  font-weight: 600;
  color: var(--text-secondary);
  text-decoration: none;
  transition: all 0.18s ease;
  white-space: nowrap;
}

.module-tab-btn:hover {
  color: var(--text-primary);
  background: rgba(23, 32, 51, 0.05);
}

.module-tab-btn.tab-active {
  background: linear-gradient(145deg, var(--navy), var(--navy-mid));
  color: #ffffff;
  box-shadow: 0 4px 14px rgba(23, 32, 51, 0.22);
}

.module-tab-btn.tab-active span.material-symbols-outlined {
  color: var(--gold-light);
}

/* ── BOTÓN DE ACCIÓN PRINCIPAL (PRESTIGE GOLD) ── */
.btn-primary-gold {
  display: inline-flex;
  align-items: center;
  gap: 0.45rem;
  padding: 0.55rem 1.25rem;
  border-radius: var(--radius-btn);
  background: linear-gradient(135deg, var(--gold) 0%, #b8860b 100%);
  color: #0f172a;
  font-family: var(--font-body);
  font-weight: 700;
  font-size: 0.82rem;
  border: none;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(184, 134, 11, 0.25);
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}

.btn-primary-gold:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 16px rgba(184, 134, 11, 0.35);
  background: linear-gradient(135deg, var(--gold-light) 0%, var(--gold) 100%);
}

/* ── TARJETAS KPI HORIZONTALES (4 COLUMNAS) ── */
.module-kpi-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
}

@media (max-width: 1024px) {
  .module-kpi-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 640px) {
  .module-kpi-grid {
    grid-template-columns: 1fr;
  }
}

.module-kpi-card {
  background: #ffffff;
  border: 1px solid rgba(15, 23, 42, 0.08);
  border-radius: 18px;
  padding: 14px 18px;
  display: flex;
  align-items: center;
  gap: 16px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.03);
  transition: all 0.22s cubic-bezier(0.16, 1, 0.3, 1);
  min-height: 84px;
  box-sizing: border-box;
}

.module-kpi-card:hover {
  transform: translateY(-2px);
  border-color: rgba(197, 168, 107, 0.4);
  box-shadow: 0 12px 30px rgba(0, 0, 0, 0.06);
}

.module-kpi-card__icon {
  width: 48px;
  height: 48px;
  border-radius: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.module-kpi-card__icon .material-symbols-outlined {
  font-size: 24px !important;
  line-height: 1 !important;
}

.module-kpi-card--primary .module-kpi-card__icon {
  background: #f1f5f9; color: #1e293b; border: 1px solid #e2e8f0;
}
.module-kpi-card--warning .module-kpi-card__icon {
  background: #fefce8; color: #b45309; border: 1px solid #fef08a;
}
.module-kpi-card--success .module-kpi-card__icon {
  background: #ecfdf5; color: #059669; border: 1px solid #a7f3d0;
}
.module-kpi-card--danger .module-kpi-card__icon {
  background: #fef2f2; color: #dc2626; border: 1px solid #fecaca;
}
.module-kpi-card--info .module-kpi-card__icon {
  background: #eff6ff; color: #2563eb; border: 1px solid #bfdbfe;
}

.module-kpi-card__info {
  display: flex;
  flex-direction: column;
  justify-content: center;
  min-width: 0;
}

.module-kpi-card__label {
  font-size: 0.68rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #718096;
  line-height: 1.2;
}

.module-kpi-card__value {
  font-size: 1.85rem;
  font-weight: 700;
  font-family: var(--font-display);
  line-height: 1.1;
  color: var(--text-primary);
  margin: 2px 0;
}

.module-kpi-card__sub {
  font-size: 0.72rem;
  color: #94a3b8;
  font-weight: 500;
  line-height: 1.2;
}
```

---

## 4. Garantía de Contraste, Modales y Acciones Secundarias en Modo Claro (Light Mode)

Para evitar que textos o botones aparezcan en blanco o con contraste deficiente sobre fondos claros/lino:

### 4.1 Reglas Estrictas de Tipografía y Modales
1. **Encabezados y Títulos (`h1`, `h2`, `h3`, `h4`, `.dialog__title`, `.modal-card h3`):**  
   - En Modo Claro DEBEN renderizarse SIEMPRE en color **Midnight Navy Oscuro (`#172033` / `#1c2940`)** con opacidad al 100%.  
   - Queda estrictamente prohibido el uso de colores blancos (`#ffffff`), grises deslavados o gradientes claros sobre tarjetas o modales en modo claro.
2. **Subtítulos y Textos Secundarios (`p`, `.modal-card p`, `.text-secondary`):**  
   - Deben usar color **Slate Steel (`#5a6477` / `#475569`)** garantizando un ratio de contraste WCAG AAA superior a 7:1.
3. **Encapsulamiento de Modales (`.modal-card`, `.modal-overlay`, `.dialog`):**  
   - Todo modal debe definir explícitamente `color: var(--text-primary);` y sobrescribir cualquier selector de encabezado interno:
   ```css
   .modal-card {
     background: var(--bg-card);
     color: var(--text-primary);
   }
   .modal-card h1, .modal-card h2, .modal-card h3, .modal-card h4 {
     color: var(--text-primary) !important;
   }
   .modal-card p {
     color: var(--text-secondary) !important;
   }
   ```

### 4.2 Botones de Acción Secundaria (Píldoras, Contornos y Acciones de Modal)
Los botones de acción secundaria como *Nuevo QR*, *Copiar Enlace*, *Cancelar*, *Refrescar* o *Filtros*:
- **Fondo:** `#ffffff` (en Light Mode) / `rgba(23, 35, 50, 0.6)` (en Dark Mode).
- **Borde:** `1px solid rgba(76, 86, 105, 0.22)` (en Light Mode) / `1px solid rgba(255, 255, 255, 0.12)` (en Dark Mode).
- **Texto e Icono:** `#1c2940` (en Light Mode) / `#edf1f5` (en Dark Mode).
- **Hover:** Borde iluminado con el dorado institucional (`rgba(197, 168, 107, 0.45)`) y elevación sutil `translateY(-1px)`.

### 4.3 Homologación de Tarjetas, Títulos y Badges Oficiales
- **Variables SCSS Globales:** `$text-primary`, `$text-secondary`, `$surface` DEBEN instanciarse como `var(--text-primary, #172033)` para prevenir evaluación estática a color blanco en tiempo de compilación.
- **Regla de Encabezados con Alta Prioridad:** Todo archivo `.component.css` debe forzar `h1, h2, h3, h4, h5, h6 { font-family: var(--font-display) !important; color: var(--text-primary) !important; }` y los títulos internos `.security-card h2, .security-card h3, .receiving-card h2, .receiving-card h3, etc.` a `color: var(--text-primary) !important;`.
- **Badge Institucional Dorado (`.badge-gold`, `.badge--gold`):** Usar siempre `background: var(--gold-bg); border: 1px solid var(--gold-border); color: var(--text-gold); font-family: var(--font-mono); font-size: 10px; font-weight: 800; text-transform: uppercase;` asegurando que "FORMATO OFICIAL", "FORMATO F01", etc. tengan máxima legibilidad y jerarquía visual.

---

## 5. Checklist Obligatorio para Nuevos Módulos

Al crear o refactorizar un módulo, verificar estrictamente los siguientes puntos antes del commit:

- [ ] **TypeScript Imports:** Si se usan `routerLinkActive` y `[routerLinkActiveOptions]`, incluir tanto `RouterLink` como `RouterLinkActive` en `imports: [...]`.
- [ ] **Sin Rutas Absolutas:** Usar siempre imports relativos (`../../`) o alias (`@4guard/shared-core`).
- [ ] **Estructura de Cabecera:**
  - Fila 1: Icono Navy (`54x54px`), Eyebrow con link de retorno, Título H1 (`Outfit 1.7rem`), Subtítulo (`0.82rem`).
  - Fila 2: Slide de pestañas a la izquierda + Botón dorado a la derecha.
- [ ] **Contraste en Modo Claro:** Verificar que en Light Mode no existan textos en blanco (`#fff`) sobre fondos blancos o grises claros en títulos, modales o botones secundarios.
- [ ] **Botones de Registro:** El icono `add` es el único que proporciona el signo `+`. El texto del botón no debe tener `+` redundante.
- [ ] **KPIs:** Disposición de 4 tarjetas horizontales con sus 4 variantes de color.
- [ ] **Compilación Limpia:** Ejecutar `npm run build:admin` o `npx ng build admin-console --configuration development` y confirmar **0 errores**.
