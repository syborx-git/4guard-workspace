# Documento de Diseño de Software (SDD)
## Módulo Unificado de Topología y Catálogo de Almacén (SDOP Framework)
**Ruta Frontend:** `/catalogs/warehouse`  
**Módulo Destino:** `apps/admin-console/src/app/features/catalogs/pages/warehouse-catalog`  
**Estado:** Propuesta de Arquitectura e Implementación Fullstack  
**Fecha:** 2026-09-22  

---

## 1. Contexto y Justificación de Negocio

Actualmente existían dos interfaces desconectadas y dispares dentro de la consola WMS:
1. **`/catalogs/warehouse` (Catálogo por Almacén / Topología):** Contenía datos 100% simulados en memoria (6 bodegas ficticias: A, APC, AT, B, BPC, BT y 282 posiciones mock en `CatalogsService`), sin persistencia ni integración con Spring Boot. No obstante, posee una valiosa **Consulta de Bahías** en formato tabular con filtros.
2. **`/inventory/map` (Mapa Interactivo de Nave):** Módulo 100% integrado al Backend (`/api/v1/warehouse-map/*`) y a PostgreSQL (`wms.warehouse_sections`, `wms.locations`), con el plano interactivo SVG 2D de la nave real de 885-915 posiciones y 11 naves operativas, capacidades de zoom, pan, drill-down y transiciones FSM (Liberar, Ocupar, Bloqueo QM).

### Objetivo
Unificar ambas pantallas en una sola experiencia de primer nivel en **`/catalogs/warehouse`**, compuesta por dos pestañas homologadas con la identidad visual del proyecto (*Midnight Navy & Prestige Gold*):
* **Pestaña 1: "1. Mapa Visual de Topología":** Plano interactivo 2D SVG con navegación espacial, métricas en vivo y panel drill-down/inspector FSM.
* **Pestaña 2: "2. Consulta de Bahías":** Tabla de búsqueda, auditoría y filtrado global de bahías/posiciones reales conectada al Backend.

Adicionalmente, se elimina el módulo huérfano `/inventory` y su entrada en el menú lateral para erradicar código muerto.

---

## 2. Arquitectura de Backend (Spring Boot + PostgreSQL)

### 2.1 Modelo de Datos en PostgreSQL
Se reutiliza y aprovecha el esquema existente sin requerir nuevas migraciones DDL:
* **`wms.warehouse_sections`:** 11 naves reales (`SEC-ALM-A`, `SEC-ALM-D`, `SEC-ALM-E`, `SEC-ALM-G`, `SEC-ALM-I`, `SEC-ALM-J`, `SEC-ALM-K`, `SEC-ALM-L`, etc.) con coordenadas SVG y capacidades de estiba.
* **`wms.locations`:** 915 ubicaciones físicas con código, pasillo, rack, nivel, posición, coordenadas espaciales, estiba y estado (`ACTIVE`, `BLOCKED`, etc.).
* **`wms.cat_block_reasons`:** Catálogo estandarizado de motivos de bloqueo de calidad (QM).
* **`wms.warehouse_section_skus` + `wms.products_sku`:** Relación de materiales y productos asignados por nave.

### 2.2 Ampliación de Endpoints REST (`WarehouseMapController`)
Se mantiene la suite existente y se añade la consulta global de bahías para alimentar la Pestaña 2:

| Método | Endpoint | Descripción | Parámetros |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/v1/warehouse-map/topology` | Topología SVG 2D completa y KPIs globales | `branchId` (UUID) |
| `GET` | `/api/v1/warehouse-map/positions` | **[NUEVO]** Consulta global de bahías para tabla | `branchId`, `sectionId` (opt), `status` (opt), `search` (opt) |
| `GET` | `/api/v1/warehouse-map/sections/{id}/positions` | Posiciones de una sección específica (drill-down) | `sectionId` (UUID), `status`, `search` |
| `PATCH` | `/api/v1/warehouse-map/positions/{id}/status` | Transición FSM (`BLOCK`, `RELEASE`, `OCCUPY`) | `positionId` (UUID), Body JSON |
| `GET` | `/api/v1/warehouse-map/catalogs/block-reasons` | Catálogo de motivos de bloqueo QM | Ninguno |

---

## 3. Arquitectura de Frontend (Angular 17 Standalone)

### 3.1 Estructura de Directorios
```text
apps/admin-console/src/app/features/catalogs/
├── models/
│   ├── warehouse-catalog.models.ts        # Modelos de topología y posiciones unificadas
│   ├── users-catalog.models.ts
│   ├── clients-catalog.models.ts
│   └── ...
├── ports/
│   └── warehouse-layout.repository.port.ts # Contrato Hexagonal (Port)
├── services/
│   ├── catalogs.service.ts                # Catálogos maestros (limpio de mocks de almacén)
│   ├── warehouse-layout-http.adapter.ts   # Adaptador HTTP REST (Cero localStorage)
│   └── warehouse-layout.service.ts        # Servicio reactivo con Signals
└── pages/
    └── warehouse-catalog/
        ├── warehouse-catalog.component.ts # Controlador de la vista unificada (2 pestañas)
        ├── warehouse-catalog.component.html # Template con Mapa SVG y Tabla de Bahías
        └── warehouse-catalog.component.css  # Estilos homologados Midnight Navy / Gold
```

### 3.2 Interfaz Unificada (`WarehouseCatalogComponent`)
* **Pestaña 1: 1. Mapa Visual de Topología:**
  * Renderizado vectorial SVG (1000x870 px) con zoom (+/- / fit) y paneo (drag).
  * Hover con tooltip reactivo de estiba y materiales.
  * Clic en zona abre Drawer lateral de posiciones.
  * Inspector Modal FSM para Liberar / Ocupar / Bloquear QM.
* **Pestaña 2: 2. Consulta de Bahías:**
  * Barra de filtros: Buscador reactivo (Código / SKU / Lote), selector de Almacén Real (11 secciones de BD), selector de Estado (*Disponible, Ocupada, Bloqueada QM*).
  * Tabla con diseño de alta densidad: Código, Almacén, Tarimas ocupadas/capacidad, SKU, Lote, Píldora de Estado, Último movimiento y botón de acción rápida para inspeccionar.
* **Sistema de Diseño:**
  * Fondo: `#0b132b` / `#0f172a` (Midnight Navy).
  * Bordes y Acentos: `#d97706` / `#f59e0b` (Prestige Gold) y Cyan WMS `#00E5FF`.
  * Estados: Verde Esmeralda (`#4DB6AC`), Azul Cobalto (`#2E4A7A`), Rojo Alerta (`#FF6B6B`).

---

## 4. Plan de Limpieza de Componentes Muertos

1. **Rutas:**
   * En `app.routes.ts`: Redirigir `inventory` e `inventory/map` hacia `/catalogs/warehouse`.
   * En `shell.component.ts`: Retirar el item de navegación `{ label: 'Inventario', route: '/inventory' }` dejando exclusivamente `{ label: 'Almacén / Topología', route: '/catalogs/warehouse', icon: 'warehouse', module: 'catalogs' }`.
2. **Archivos a Eliminar:**
   * `apps/admin-console/src/app/features/inventory/` (directorio completo con `inventory.routes.ts`, `warehouse-map/`, etc.), trasladando su lógica probada a `catalogs/`.
3. **Validación de Compilación:**
   * `npx ng build admin-console --configuration development` con 0 errores y 0 dependencias rotas.
