# SDD — Blueprint Técnico: Conexión del Mapa Interactivo 2D de Nave con Backend y PostgreSQL

**Módulo:** Operación e Inventario → Mapa Interactivo 2D de Nave (`warehouse-map`)  
**HU Relacionadas:** HU-048 (Topología de Almacén) / HU-127 (Control FSM de Ubicaciones Físicas)  
**Repositorios:** `4guard_be` (Spring Boot 3 / PostgreSQL 15) · `4Guard_FE_UI` (Angular 17+ Standalone)  
**Estado:** Propuesta Técnica de Arquitectura (Spec-Driven / SDOP)  
**Versión:** 1.0.0  
**Autor:** Lead Developer — SyborX Engineering Team  
**Fecha de Emisión:** 2026-09-21  

---

## 1. Objetivo y Resumen de la Solución

El presente documento de diseño de software (SDD) establece la especificación técnica definitiva para conectar el módulo **Mapa Interactivo 2D de Nave** (`WarehouseMapComponent`), actualmente desconectado y persistido en `LocalStorage`, con la infraestructura real de backend en **Spring Boot** y la base de datos **PostgreSQL**.

### Principales Desafíos Resueltos
1. **Persistencia Espacial 2D en Base de Datos:** Incorporación de polígonos SVG (`polygonPoints`) y coordenadas de etiquetado en `wms.warehouse_sections`.
2. **Siembra Masiva de las 885 Posiciones Físicas:** Generación y normalización de los 885 slots individuales en `wms.locations` para la sucursal activa (`CDMX-01`), permitiendo control volumétrico y operativo real.
3. **Catálogo de Motivos de Bloqueo QM:** Creación de la tabla `wms.cat_block_reasons` con los 10 motivos estándar WMS, garantizando integridad referencial en las transiciones de estado FSM.
4. **Endpoint Compuesto de Alta Eficiencia:** Implementación de `GET /api/v1/warehouse-map/topology` para evitar el problema de $N+1$ requests al cargar la nave completa.
5. **Bridge Hexagonal en Frontend:** Reemplazo de `WarehouseLayoutService` por una arquitectura desacoplada basada en `WarehouseLayoutRepositoryPort` y `WarehouseLayoutHttpAdapter`, manteniendo resiliencia de fallback.
6. **Oráculo Automatizado (Playwright):** Suite de pruebas E2E que certifica la visualización 2D, el drill-down, la búsqueda reactiva y la mutación transaccional de bloqueo/liberación QM contra PostgreSQL.

---

## 2. Script Flyway: `V23__warehouse_map_2d_topology.sql`

Este script DDL/DML migra la base de datos PostgreSQL al estado requerido para dar soporte al mapa topológico 2D, normaliza las secciones existentes, agrega la sección faltante `SEC-ALM-K`, crea el catálogo de bloqueos y siembra las 885 posiciones operativas individuales.

```sql
-- =============================================================================
-- 4GUARD WMS — Flyway Migration V23
-- Archivo: V23__warehouse_map_2d_topology.sql
-- Módulo: Topología Física 2D y Mapa Interactivo de Almacén (HU-048 / HU-127)
-- Descripción:
--   1. Agrega metadatos espaciales y de estiba a wms.warehouse_sections.
--   2. Registra la sección 'SEC-ALM-K' y normaliza capacidades de naves A-L.
--   3. Crea tabla de catálogo para motivos de bloqueo QM (wms.cat_block_reasons).
--   4. Siembra las 885 posiciones operativas en wms.locations para la sucursal activa.
--   5. Crea tabla de vinculación de SKUs por sección (wms.warehouse_section_skus).
--   6. Índices compuestos para consultas espaciales y de ocupación.
-- =============================================================================

SET search_path TO wms, public;

-- ─────────────────────────────────────────────────────────────────────────────
-- 1. EXTENDER wms.warehouse_sections CON METADATOS ESPACIALES Y DE ESTIBA
-- ─────────────────────────────────────────────────────────────────────────────

ALTER TABLE wms.warehouse_sections
    ADD COLUMN IF NOT EXISTS category VARCHAR(100) DEFAULT 'General',
    ADD COLUMN IF NOT EXISTS pos_fijas INTEGER DEFAULT 0,
    ADD COLUMN IF NOT EXISTS capacidad_tarimas INTEGER DEFAULT 0,
    ADD COLUMN IF NOT EXISTS factor_estiba VARCHAR(50) DEFAULT '22 tarimas/pos',
    ADD COLUMN IF NOT EXISTS notes TEXT,
    ADD COLUMN IF NOT EXISTS polygon_points TEXT,
    ADD COLUMN IF NOT EXISTS label_x NUMERIC(6,2) DEFAULT 0,
    ADD COLUMN IF NOT EXISTS label_y NUMERIC(6,2) DEFAULT 0,
    ADD COLUMN IF NOT EXISTS sublabel_x NUMERIC(6,2) DEFAULT 0,
    ADD COLUMN IF NOT EXISTS sublabel_y NUMERIC(6,2) DEFAULT 0;

-- ─────────────────────────────────────────────────────────────────────────────
-- 2. REGISTRAR SECCIÓN K Y ACTUALIZAR COORDENADAS SVG EN SECCIONES EXISTENTES
-- ─────────────────────────────────────────────────────────────────────────────

-- 2.1 Asegurar existencia de SEC-ALM-K (Almacén K - Racks Libres & Anexo)
INSERT INTO wms.warehouse_sections (
    id, branch_id, code, name, category, pos_fijas, capacidad_tarimas, factor_estiba,
    notes, status, polygon_points, label_x, label_y, sublabel_x, sublabel_y,
    created_by, updated_by
) VALUES (
    'a83f0907-9fa5-4bdf-87db-2eb5e7683999',
    'b73f0907-9fa5-4bdf-87db-2eb5e7683936',
    'SEC-ALM-K',
    'Almacén K - Racks Libres & Anexo',
    'Almacén Anexo Exterior',
    181,
    3982,
    '22 tarimas/pos',
    'Racks libres para sobreflujo y consolidación de estiba pesada.',
    'ACTIVE',
    '600,166 734,130 734,369 600,369',
    667, 245, 667, 263,
    'SYSTEM', 'SYSTEM'
) ON CONFLICT (id) DO UPDATE SET
    code = EXCLUDED.code,
    name = EXCLUDED.name,
    category = EXCLUDED.category,
    pos_fijas = EXCLUDED.pos_fijas,
    capacidad_tarimas = EXCLUDED.capacidad_tarimas,
    factor_estiba = EXCLUDED.factor_estiba,
    polygon_points = EXCLUDED.polygon_points,
    label_x = EXCLUDED.label_x,
    label_y = EXCLUDED.label_y,
    sublabel_x = EXCLUDED.sublabel_x,
    sublabel_y = EXCLUDED.sublabel_y,
    updated_at = CURRENT_TIMESTAMP;

-- 2.2 Actualizar metadatos y polígonos de las naves activas existentes
UPDATE wms.warehouse_sections
SET category = 'Secos & Producto Terminado',
    pos_fijas = 170,
    capacidad_tarimas = 3740,
    factor_estiba = '22 tarimas/pos',
    notes = 'Área QUALAMEX en posiciones 149 a 170. Incluye Rampa 2.',
    polygon_points = '800,188 976,188 976,330 878,330 878,790 800,790',
    label_x = 865, label_y = 530, sublabel_x = 865, sublabel_y = 548,
    updated_at = CURRENT_TIMESTAMP
WHERE id = 'd7cceb3e-ad31-4ef4-bafa-f34b327ded9a'; -- SEC-ALM-A

UPDATE wms.warehouse_sections
SET category = 'Materia Prima & Insumos',
    pos_fijas = 38,
    capacidad_tarimas = 760,
    factor_estiba = '20 tarimas/pos',
    notes = 'Rampas 6, 7 y 8. Espacio para maniobra de montacargas.',
    polygon_points = '310,670 506,670 506,738 468,738 468,787 338,787 338,836 274,836 274,765 310,765',
    label_x = 412, label_y = 720, sublabel_x = 412, sublabel_y = 738,
    updated_at = CURRENT_TIMESTAMP
WHERE id = '28512354-5edf-4905-a042-61b0790c5277'; -- SEC-ALM-E

UPDATE wms.warehouse_sections
SET category = 'Empaque & Vidrio Industrial',
    pos_fijas = 120,
    capacidad_tarimas = 2640,
    factor_estiba = '22 tarimas/pos',
    notes = 'Incluye Oficinas de mantenimiento y almacén de cartón.',
    polygon_points = '600,369 722,369 722,760 635,760 635,824 597,824 597,710 600,710',
    label_x = 665, label_y = 585, sublabel_x = 665, sublabel_y = 603,
    updated_at = CURRENT_TIMESTAMP
WHERE id = '404a0fa1-fdea-4c30-932a-8e65ac6fb0a1'; -- SEC-ALM-D (F(D))

UPDATE wms.warehouse_sections
SET category = 'General Central & Palletizado',
    pos_fijas = 117,
    capacidad_tarimas = 2574,
    factor_estiba = '22 tarimas/pos',
    notes = 'Nave con columnas estructurales en cuadrícula.',
    polygon_points = '56,137 280,137 280,355 240,387 160,352 160,400 110,296 56,137',
    label_x = 195, label_y = 240, sublabel_x = 195, sublabel_y = 258,
    updated_at = CURRENT_TIMESTAMP
WHERE id = '67cb8d01-0a10-444b-b752-6ec7309b94e0'; -- SEC-ALM-G

UPDATE wms.warehouse_sections
SET category = 'Insumos Especiales',
    pos_fijas = 91,
    capacidad_tarimas = 2002,
    factor_estiba = '22 tarimas/pos',
    notes = 'Bahía longitudinal con pasillos de distribución central.',
    polygon_points = '412,298 506,252 506,670 412,670',
    label_x = 459, label_y = 470, sublabel_x = 459, sublabel_y = 488,
    updated_at = CURRENT_TIMESTAMP
WHERE id = 'd5703d81-cabd-4c62-8bbc-6ce96f7de184'; -- SEC-ALM-I

UPDATE wms.warehouse_sections
SET category = 'Granel & Tambores',
    pos_fijas = 56,
    capacidad_tarimas = 2240,
    factor_estiba = '40 tarimas/pos',
    notes = 'Alta capacidad de estiba por posición (40 tarimas/rack).',
    polygon_points = '280,323 310,323 312,298 412,298 412,670 310,670 310,642 280,642 280,494 298,494 298,355 280,355',
    label_x = 360, label_y = 470, sublabel_x = 360, sublabel_y = 488,
    updated_at = CURRENT_TIMESTAMP
WHERE id = 'f2cf444c-6b3a-4e08-bf40-f46d3b7597cf'; -- SEC-ALM-J (J(C))

UPDATE wms.warehouse_sections
SET category = 'Cuarentena & Retenidos',
    pos_fijas = 112,
    capacidad_tarimas = 2464,
    factor_estiba = '22 tarimas/pos',
    notes = '112 posiciones numeradas consecutivas 1 al 112.',
    polygon_points = '506,252 600,166 600,238 600,738 506,738',
    label_x = 553, label_y = 470, sublabel_x = 553, sublabel_y = 488,
    updated_at = CURRENT_TIMESTAMP
WHERE id = '7f028959-e3e1-4822-bf41-fe499e112c14'; -- SEC-ALM-L

UPDATE wms.warehouse_sections
SET category = 'Área Técnica',
    pos_fijas = 0,
    capacidad_tarimas = 0,
    factor_estiba = '--',
    notes = 'Pendiente de carga de archivo Excel de catálogo.',
    polygon_points = '228,17 298,17 298,87 268,91 268,135 228,135',
    label_x = 263, label_y = 60, sublabel_x = 263, sublabel_y = 75,
    updated_at = CURRENT_TIMESTAMP
WHERE id = 'ffd283a1-7568-45fb-9157-b3d96ecacdb4'; -- SEC-ALM-H

UPDATE wms.warehouse_sections
SET category = 'Área Futura',
    pos_fijas = 0,
    capacidad_tarimas = 0,
    factor_estiba = '--',
    notes = 'Pendiente de carga de archivo Excel de catálogo.',
    polygon_points = '734,130 853,52 853,188 734,188',
    label_x = 793, label_y = 125, sublabel_x = 793, sublabel_y = 140,
    updated_at = CURRENT_TIMESTAMP
WHERE id = '5e3d66ce-679b-40fd-a4c5-0a71731c1e56'; -- SEC-ALM-B

UPDATE wms.warehouse_sections
SET category = 'Área Futura',
    pos_fijas = 0,
    capacidad_tarimas = 0,
    factor_estiba = '--',
    notes = 'Pendiente de carga de archivo Excel de catálogo.',
    polygon_points = '722,369 800,369 800,760 722,760',
    label_x = 760, label_y = 585, sublabel_x = 760, sublabel_y = 600,
    updated_at = CURRENT_TIMESTAMP
WHERE id = 'b0d7b81d-2d2a-4f82-9f3e-9c90dcd7ca77'; -- SEC-ALM-C

-- ─────────────────────────────────────────────────────────────────────────────
-- 3. CATÁLOGO DE MOTIVOS DE BLOQUEO QM (wms.cat_block_reasons)
-- ─────────────────────────────────────────────────────────────────────────────

CREATE TABLE IF NOT EXISTS wms.cat_block_reasons (
    id UUID PRIMARY KEY DEFAULT wms.uuid_generate_v4(),
    code VARCHAR(50) NOT NULL UNIQUE,
    description VARCHAR(200) NOT NULL,
    category VARCHAR(50) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO wms.cat_block_reasons (code, description, category) VALUES
    ('QM_CONTAMINATION',   'Cuarentena QM — Sospecha de contaminación', 'QUALITY'),
    ('QM_INSPECTION',      'Cuarentena QM — Inspección de calidad en proceso', 'QUALITY'),
    ('QM_LAB_SAMPLE',      'Cuarentena QM — Muestra retenida para análisis de laboratorio', 'QUALITY'),
    ('MAINT_RACK_REPAIR',  'Mantenimiento — Reparación de rack o estructura', 'MAINTENANCE'),
    ('MAINT_SCHEDULED',    'Mantenimiento — Inspección técnica programada', 'MAINTENANCE'),
    ('CYCLE_COUNT',        'Inventario cíclico — Recuento en curso', 'INVENTORY'),
    ('PHYSICAL_DAMAGE',    'Daño físico — Producto con daño visible', 'SECURITY'),
    ('SPILL_HAZARD',       'Derrame o contaminación — Zona delimitada por seguridad', 'SECURITY'),
    ('ADMIN_REVIEW',       'Bloqueo administrativo — Pendiente de revisión por supervisor', 'ADMINISTRATIVE'),
    ('OVERWEIGHT_LIMIT',   'Exceso de peso — Sobrepasa límite de carga del rack', 'SECURITY')
ON CONFLICT (code) DO UPDATE SET
    description = EXCLUDED.description,
    category = EXCLUDED.category,
    updated_at = CURRENT_TIMESTAMP;

-- ─────────────────────────────────────────────────────────────────────────────
-- 4. TABLA DE ASOCIACIÓN SECCIÓN-SKUS PERMITIDOS (wms.warehouse_section_skus)
-- ─────────────────────────────────────────────────────────────────────────────

CREATE TABLE IF NOT EXISTS wms.warehouse_section_skus (
    id UUID PRIMARY KEY DEFAULT wms.uuid_generate_v4(),
    section_id UUID NOT NULL REFERENCES wms.warehouse_sections(id) ON DELETE CASCADE,
    sku_id UUID NOT NULL REFERENCES wms.products_sku(id) ON DELETE CASCADE,
    is_primary BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uk_section_sku UNIQUE (section_id, sku_id)
);

-- Vincular SKUs de Nestlé a las secciones según la pauta del demo
INSERT INTO wms.warehouse_section_skus (section_id, sku_id)
SELECT s.id, p.id
FROM wms.warehouse_sections s
CROSS JOIN wms.products_sku p
WHERE (s.code = 'SEC-ALM-A' AND p.code IN ('43211385', '43519988'))
   OR (s.code = 'SEC-ALM-E' AND p.code IN ('41165316', '44242440'))
   OR (s.code = 'SEC-ALM-D' AND p.code IN ('41165277', '44271527', '41165274'))
   OR (s.code = 'SEC-ALM-G' AND p.code IN ('44270022', '41165316', '44318043', '43140353'))
   OR (s.code = 'SEC-ALM-I' AND p.code IN ('43211385', '43759735', '44318043', '43510616'))
   OR (s.code = 'SEC-ALM-J' AND p.code IN ('41165272', '41165273', '41165275', '41165276', '43457162'))
   OR (s.code = 'SEC-ALM-L' AND p.code IN ('44271537', '43543406'))
   OR (s.code = 'SEC-ALM-K' AND p.code IN ('41165793', '43759734'))
ON CONFLICT (section_id, sku_id) DO NOTHING;

-- ─────────────────────────────────────────────────────────────────────────────
-- 5. SIEMBRA AUTOMATIZADA DE LAS 885 POSICIONES FÍSICAS EN wms.locations
-- ─────────────────────────────────────────────────────────────────────────────

CREATE OR REPLACE FUNCTION wms.fn_seed_warehouse_section_positions(
    p_branch_id UUID,
    p_section_id UUID,
    p_prefix VARCHAR,
    p_count INTEGER,
    p_capacity INTEGER
) RETURNS VOID AS $$
DECLARE
    i INTEGER;
    v_code VARCHAR(30);
    v_name VARCHAR(150);
    v_pos_str VARCHAR(10);
BEGIN
    FOR i IN 1..p_count LOOP
        v_pos_str := LPAD(i::TEXT, 3, '0');
        v_code    := 'POS-' || p_prefix || '-' || v_pos_str;
        v_name    := 'Posición ' || v_pos_str || ' — Nave ' || p_prefix;

        INSERT INTO wms.locations (
            id, branch_id, section_id, code, name, zone, aisle, rack, level, position,
            coord_x, coord_y, coord_z, type, status, capacity_units, current_occupancy,
            is_blocked, is_active, created_by, updated_by
        ) VALUES (
            wms.uuid_generate_v4(),
            p_branch_id,
            p_section_id,
            v_code,
            v_name,
            p_prefix,
            LPAD(((i - 1) / 20 + 1)::TEXT, 2, '0'), -- Pasillo simulado
            LPAD(((i - 1) % 10 + 1)::TEXT, 2, '0'), -- Rack simulado
            1,                                      -- Nivel 1 (piso/estiba)
            v_pos_str,
            ((i - 1) % 10) * 5,
            ((i - 1) / 10) * 5,
            1,
            'PALLET',
            'ACTIVE',
            p_capacity,
            0,
            FALSE,
            TRUE,
            'SYSTEM',
            'SYSTEM'
        ) ON CONFLICT (code) DO UPDATE SET
            section_id = EXCLUDED.section_id,
            capacity_units = EXCLUDED.capacity_units,
            status = 'ACTIVE',
            updated_at = CURRENT_TIMESTAMP;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Ejecutar siembra para las 8 secciones activas (Total: 885 posiciones)
DO $$
DECLARE
    v_branch UUID := 'b73f0907-9fa5-4bdf-87db-2eb5e7683936';
BEGIN
    -- Nave A: 170 posiciones (Capacidad 22)
    PERFORM wms.fn_seed_warehouse_section_positions(v_branch, 'd7cceb3e-ad31-4ef4-bafa-f34b327ded9a', 'A', 170, 22);

    -- Nave E: 38 posiciones (Capacidad 20)
    PERFORM wms.fn_seed_warehouse_section_positions(v_branch, '28512354-5edf-4905-a042-61b0790c5277', 'E', 38, 20);

    -- Nave F(D): 120 posiciones (Capacidad 22)
    PERFORM wms.fn_seed_warehouse_section_positions(v_branch, '404a0fa1-fdea-4c30-932a-8e65ac6fb0a1', 'F-D', 120, 22);

    -- Nave G: 117 posiciones (Capacidad 22)
    PERFORM wms.fn_seed_warehouse_section_positions(v_branch, '67cb8d01-0a10-444b-b752-6ec7309b94e0', 'G', 117, 22);

    -- Nave I: 91 posiciones (Capacidad 22)
    PERFORM wms.fn_seed_warehouse_section_positions(v_branch, 'd5703d81-cabd-4c62-8bbc-6ce96f7de184', 'I', 91, 22);

    -- Nave J(C): 56 posiciones (Capacidad 40)
    PERFORM wms.fn_seed_warehouse_section_positions(v_branch, 'f2cf444c-6b3a-4e08-bf40-f46d3b7597cf', 'J-C', 56, 40);

    -- Nave L: 112 posiciones (Capacidad 22)
    PERFORM wms.fn_seed_warehouse_section_positions(v_branch, '7f028959-e3e1-4822-bf41-fe499e112c14', 'L', 112, 22);

    -- Nave K: 181 posiciones (Capacidad 22)
    PERFORM wms.fn_seed_warehouse_section_positions(v_branch, 'a83f0907-9fa5-4bdf-87db-2eb5e7683999', 'K', 181, 22);
END $$;

-- Eliminar función utilitaria de migración
DROP FUNCTION IF EXISTS wms.fn_seed_warehouse_section_positions(UUID, UUID, VARCHAR, INTEGER, INTEGER);

-- ─────────────────────────────────────────────────────────────────────────────
-- 6. ÍNDICES DE RENDIMIENTO PARA ACCESO ESPACIAL Y GESTIÓN EN TIEMPO REAL
-- ─────────────────────────────────────────────────────────────────────────────

CREATE INDEX IF NOT EXISTS idx_locations_section_status
    ON wms.locations (section_id, status)
    WHERE is_deleted = FALSE;

CREATE INDEX IF NOT EXISTS idx_locations_code_search
    ON wms.locations (code varchar_pattern_ops);

CREATE INDEX IF NOT EXISTS idx_section_skus_section
    ON wms.warehouse_section_skus (section_id);
```

---

## 3. Backend (Spring Boot): Arquitectura Hexagonal

### 3.1 Entidades JPA Modificadas y Nuevas

#### `WarehouseSectionEntity.java` (Ampliación)
```java
package com.fourguard.wms.infrastructure.persistence.entity;

import com.fourguard.wms.domain.enums.WarehouseSectionStatus;
import com.fourguard.wms.shared.audit.BaseVersionedEntity;
import jakarta.persistence.*;
import lombok.*;
import lombok.experimental.SuperBuilder;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@Entity
@Table(name = "warehouse_sections", schema = "wms")
@Getter
@Setter
@NoArgsConstructor
@SuperBuilder(toBuilder = true)
public class WarehouseSectionEntity extends BaseVersionedEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(updatable = false, nullable = false, columnDefinition = "UUID")
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "branch_id", nullable = false)
    private BranchEntity branch;

    @Column(nullable = false, length = 20)
    private String code;

    @Column(length = 100)
    private String name;

    @Column(length = 100)
    private String category;

    @Column(name = "pos_fijas")
    private Integer posFijas;

    @Column(name = "capacidad_tarimas")
    private Integer capacidadTarimas;

    @Column(name = "factor_estiba", length = 50)
    private String factorEstiba;

    @Column(columnDefinition = "TEXT")
    private String notes;

    @Column(name = "polygon_points", columnDefinition = "TEXT")
    private String polygonPoints;

    @Column(name = "label_x", precision = 6, scale = 2)
    private BigDecimal labelX;

    @Column(name = "label_y", precision = 6, scale = 2)
    private BigDecimal labelY;

    @Column(name = "sublabel_x", precision = 6, scale = 2)
    private BigDecimal sublabelX;

    @Column(name = "sublabel_y", precision = 6, scale = 2)
    private BigDecimal sublabelY;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    @Builder.Default
    private WarehouseSectionStatus status = WarehouseSectionStatus.ACTIVE;

    @OneToMany(mappedBy = "section", fetch = FetchType.LAZY)
    @Builder.Default
    private List<LocationEntity> locations = new ArrayList<>();
}
```

#### `CatBlockReasonEntity.java` (Nueva)
```java
package com.fourguard.wms.infrastructure.persistence.entity;

import jakarta.persistence.*;
import lombok.*;

import java.time.OffsetDateTime;
import java.util.UUID;

@Entity
@Table(name = "cat_block_reasons", schema = "wms")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CatBlockReasonEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(updatable = false, nullable = false, columnDefinition = "UUID")
    private UUID id;

    @Column(nullable = false, unique = true, length = 50)
    private String code;

    @Column(nullable = false, length = 200)
    private String description;

    @Column(nullable = false, length = 50)
    private String category;

    @Column(name = "is_active", nullable = false)
    @Builder.Default
    private Boolean isActive = true;

    @Column(name = "created_at")
    @Builder.Default
    private OffsetDateTime createdAt = OffsetDateTime.now();
}
```

---

### 3.2 Repositorios JPA

#### `WarehouseMapSectionJpaRepository.java`
```java
package com.fourguard.wms.infrastructure.persistence.repository;

import com.fourguard.wms.infrastructure.persistence.entity.WarehouseSectionEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.UUID;

@Repository
public interface WarehouseMapSectionJpaRepository extends JpaRepository<WarehouseSectionEntity, UUID> {

    @Query("""
        SELECT s FROM WarehouseSectionEntity s
        LEFT JOIN FETCH s.locations l
        WHERE s.branch.id = :branchId AND s.isDeleted = false
        ORDER BY s.code ASC
    """)
    List<WarehouseSectionEntity> findSectionsWithLocationsByBranchId(@Param("branchId") UUID branchId);
}
```

#### `WarehouseMapLocationJpaRepository.java`
```java
package com.fourguard.wms.infrastructure.persistence.repository;

import com.fourguard.wms.infrastructure.persistence.entity.LocationEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Repository
public interface WarehouseMapLocationJpaRepository extends JpaRepository<LocationEntity, UUID> {

    @Query("""
        SELECT l FROM LocationEntity l
        WHERE l.section.id = :sectionId
          AND l.isDeleted = false
          AND (:status IS NULL OR l.status = :status)
          AND (:search IS NULL OR LOWER(l.code) LIKE LOWER(CONCAT('%', :search, '%'))
                               OR LOWER(l.name) LIKE LOWER(CONCAT('%', :search, '%')))
        ORDER BY l.code ASC
    """)
    List<LocationEntity> findFilteredPositions(
        @Param("sectionId") UUID sectionId,
        @Param("status") String status,
        @Param("search") String search
    );

    @Query("SELECT COUNT(l) FROM LocationEntity l WHERE l.section.id = :sectionId AND l.status = 'BLOCKED'")
    long countBlockedBySection(@Param("sectionId") UUID sectionId);

    @Query("SELECT COUNT(l) FROM LocationEntity l WHERE l.section.id = :sectionId AND l.currentOccupancy > 0")
    long countOccupiedBySection(@Param("sectionId") UUID sectionId);
}
```

---

### 3.3 DTOs (Request / Response)

#### `WarehouseTopologyResponse.java`
```java
package com.fourguard.wms.application.dto.response.map;

import lombok.Builder;
import lombok.Getter;

import java.time.OffsetDateTime;
import java.util.List;
import java.util.UUID;

@Getter
@Builder
public class WarehouseTopologyResponse {
    private final UUID branchId;
    private final String branchName;
    private final WarehouseMapStatsResponse globalStats;
    private final List<WarehouseSectionMapResponse> sections;
    private final OffsetDateTime generatedAt;
}
```

#### `WarehouseSectionMapResponse.java`
```java
package com.fourguard.wms.application.dto.response.map;

import lombok.Builder;
import lombok.Getter;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

@Getter
@Builder
public class WarehouseSectionMapResponse {
    private final UUID id;
    private final String code;
    private final String name;
    private final String category;
    private final Integer posFijas;
    private final Integer capacidadTarimas;
    private final String factorEstiba;
    private final List<String> materials;
    private final String notes;
    private final String status; // LOADED | PENDING
    private final String polygonPoints;
    private final CoordinateResponse labelPosition;
    private final CoordinateResponse sublabelPosition;
    private final Integer occupiedPositions;
    private final Integer availablePositions;
    private final Integer blockedPositions;
    private final Integer occupancyPercentage;
}
```

#### `PositionMapDetailResponse.java`
```java
package com.fourguard.wms.application.dto.response.map;

import lombok.Builder;
import lombok.Getter;

import java.util.UUID;

@Getter
@Builder
public class PositionMapDetailResponse {
    private final UUID id;
    private final Integer positionNumber;
    private final String code;
    private final UUID sectionId;
    private final String sectionName;
    private final String skuCode;
    private final String skuDescription;
    private final String status; // AVAILABLE | OCCUPIED | BLOCKED | MAINTENANCE
    private final Integer capacityTarimas;
    private final Integer currentTarimas;
    private final String batchNumber;
    private final String lastMovement;
    private final String blockReason;
    private final String statusReason;
    private final Boolean isBlocked;
}
```

#### `UpdatePositionStatusMapRequest.java`
```java
package com.fourguard.wms.application.dto.request.map;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotNull;
import lombok.Builder;
import lombok.Value;

@Value
@Builder
public class UpdatePositionStatusMapRequest {

    @NotNull(message = "El campo 'targetAction' es obligatorio")
    @Schema(description = "Acción operativa: BLOCK, RELEASE u OCCUPY", example = "BLOCK")
    String targetAction; // 'BLOCK' | 'RELEASE' | 'OCCUPY'

    @Schema(description = "Código o descripción del motivo de bloqueo QM", example = "QM_CONTAMINATION")
    String reasonCode;

    @Schema(description = "Comentarios u observaciones adicionales del supervisor", example = "Inspección técnica programada")
    String comment;
}
```

---

### 3.4 Service / UseCase Layer

#### Puerto Inbound: `WarehouseMapUseCase.java`
```java
package com.fourguard.wms.domain.ports.in;

import com.fourguard.wms.application.dto.request.map.UpdatePositionStatusMapRequest;
import com.fourguard.wms.application.dto.response.map.CatBlockReasonResponse;
import com.fourguard.wms.application.dto.response.map.PositionMapDetailResponse;
import com.fourguard.wms.application.dto.response.map.WarehouseTopologyResponse;

import java.util.List;
import java.util.UUID;

public interface WarehouseMapUseCase {
    WarehouseTopologyResponse getTopology(UUID branchId);
    List<PositionMapDetailResponse> getPositionsBySection(UUID sectionId, String status, String search);
    PositionMapDetailResponse updatePositionStatus(UUID positionId, UpdatePositionStatusMapRequest request, String username);
    List<CatBlockReasonResponse> getActiveBlockReasons();
}
```

#### Implementación: `WarehouseMapService.java`
```java
package com.fourguard.wms.application.usecase;

import com.fourguard.wms.application.dto.request.map.UpdatePositionStatusMapRequest;
import com.fourguard.wms.application.dto.response.map.*;
import com.fourguard.wms.domain.enums.LocationStatus;
import com.fourguard.wms.domain.ports.in.WarehouseMapUseCase;
import com.fourguard.wms.infrastructure.persistence.entity.LocationEntity;
import com.fourguard.wms.infrastructure.persistence.entity.WarehouseSectionEntity;
import com.fourguard.wms.infrastructure.persistence.repository.CatBlockReasonJpaRepository;
import com.fourguard.wms.infrastructure.persistence.repository.WarehouseMapLocationJpaRepository;
import com.fourguard.wms.infrastructure.persistence.repository.WarehouseMapSectionJpaRepository;
import jakarta.persistence.EntityNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.OffsetDateTime;
import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class WarehouseMapService implements WarehouseMapUseCase {

    private final WarehouseMapSectionJpaRepository sectionRepository;
    private final WarehouseMapLocationJpaRepository locationRepository;
    private final CatBlockReasonJpaRepository blockReasonRepository;

    @Override
    @Transactional(readOnly = true)
    public WarehouseTopologyResponse getTopology(UUID branchId) {
        List<WarehouseSectionEntity> sections = sectionRepository.findSectionsWithLocationsByBranchId(branchId);

        int totalPositions = 0;
        int totalCapacity = 0;
        int totalOccupied = 0;
        int totalBlocked = 0;

        List<WarehouseSectionMapResponse> sectionResponses = new java.util.ArrayList<>();

        for (WarehouseSectionEntity sec : sections) {
            int posCount = sec.getLocations().size();
            long blocked = sec.getLocations().stream().filter(l -> l.getStatus() == LocationStatus.BLOCKED).count();
            long occupied = sec.getLocations().stream().filter(l -> l.getCurrentOccupancy() != null && l.getCurrentOccupancy() > 0).count();
            int available = (int) (posCount - blocked - occupied);

            totalPositions += posCount;
            totalCapacity += (sec.getCapacidadTarimas() != null ? sec.getCapacidadTarimas() : 0);
            totalOccupied += occupied;
            totalBlocked += blocked;

            int pct = posCount > 0 ? (int) Math.round(((double) occupied / posCount) * 100) : 0;

            sectionResponses.add(WarehouseSectionMapResponse.builder()
                .id(sec.getId())
                .code(sec.getCode())
                .name(sec.getName())
                .category(sec.getCategory())
                .posFijas(sec.getPosFijas())
                .capacidadTarimas(sec.getCapacidadTarimas())
                .factorEstiba(sec.getFactorEstiba())
                .notes(sec.getNotes())
                .status(sec.getPosFijas() != null && sec.getPosFijas() > 0 ? "LOADED" : "PENDING")
                .polygonPoints(sec.getPolygonPoints())
                .labelPosition(CoordinateResponse.builder().x(sec.getLabelX()).y(sec.getLabelY()).build())
                .sublabelPosition(CoordinateResponse.builder().x(sec.getSublabelX()).y(sec.getSublabelY()).build())
                .occupiedPositions((int) occupied)
                .availablePositions(Math.max(0, available))
                .blockedPositions((int) blocked)
                .occupancyPercentage(pct)
                .build());
        }

        WarehouseMapStatsResponse stats = WarehouseMapStatsResponse.builder()
            .totalSections(sections.size())
            .loadedSections((int) sections.stream().filter(s -> s.getPosFijas() != null && s.getPosFijas() > 0).count())
            .pendingSections((int) sections.stream().filter(s -> s.getPosFijas() == null || s.getPosFijas() == 0).count())
            .totalPositions(totalPositions)
            .totalCapacityTarimas(totalCapacity)
            .occupiedPositions(totalOccupied)
            .blockedPositions(totalBlocked)
            .build();

        return WarehouseTopologyResponse.builder()
            .branchId(branchId)
            .globalStats(stats)
            .sections(sectionResponses)
            .generatedAt(OffsetDateTime.now())
            .build();
    }

    @Override
    @Transactional(readOnly = true)
    public List<PositionMapDetailResponse> getPositionsBySection(UUID sectionId, String status, String search) {
        List<LocationEntity> locations = locationRepository.findFilteredPositions(sectionId, status, search);
        return locations.stream().map(this::mapLocationToPositionDetail).toList();
    }

    @Override
    @Transactional
    public PositionMapDetailResponse updatePositionStatus(UUID positionId, UpdatePositionStatusMapRequest request, String username) {
        LocationEntity loc = locationRepository.findById(positionId)
            .orElseThrow(() -> new EntityNotFoundException("Ubicación no encontrada con ID: " + positionId));

        switch (request.getTargetAction().toUpperCase()) {
            case "BLOCK" -> {
                String fullReason = request.getReasonCode() + (request.getComment() != null && !request.getComment().isBlank() 
                    ? " — " + request.getComment().trim() : "");
                loc.setStatus(LocationStatus.BLOCKED);
                loc.setIsBlocked(true);
                loc.setStatusReason(fullReason);
                loc.setBlockReason(fullReason);
                loc.setUpdatedBy(username);
            }
            case "RELEASE" -> {
                loc.setStatus(LocationStatus.ACTIVE);
                loc.setIsBlocked(false);
                loc.setStatusReason(null);
                loc.setBlockReason(null);
                loc.setCurrentOccupancy(0);
                loc.setUpdatedBy(username);
            }
            case "OCCUPY" -> {
                loc.setStatus(LocationStatus.ACTIVE);
                loc.setIsBlocked(false);
                loc.setBlockReason(null);
                loc.setCurrentOccupancy(loc.getCapacityUnits());
                loc.setUpdatedBy(username);
            }
            default -> throw new IllegalArgumentException("Acción targetAction no soportada: " + request.getTargetAction());
        }

        LocationEntity saved = locationRepository.save(loc);
        return mapLocationToPositionDetail(saved);
    }

    @Override
    @Transactional(readOnly = true)
    public List<CatBlockReasonResponse> getActiveBlockReasons() {
        return blockReasonRepository.findByIsActiveTrueOrderByDescriptionAsc().stream()
            .map(r -> new CatBlockReasonResponse(r.getCode(), r.getDescription(), r.getCategory()))
            .toList();
    }

    private PositionMapDetailResponse mapLocationToPositionDetail(LocationEntity l) {
        String visualStatus = "AVAILABLE";
        if (l.getStatus() == LocationStatus.BLOCKED || Boolean.TRUE.equals(l.getIsBlocked())) {
            visualStatus = "BLOCKED";
        } else if (l.getStatus() == LocationStatus.MAINTENANCE) {
            visualStatus = "MAINTENANCE";
        } else if (l.getCurrentOccupancy() != null && l.getCurrentOccupancy() > 0) {
            visualStatus = "OCCUPIED";
        }

        int posNum = 1;
        try {
            if (l.getPosition() != null) posNum = Integer.parseInt(l.getPosition());
        } catch (NumberFormatException ignored) {}

        return PositionMapDetailResponse.builder()
            .id(l.getId())
            .positionNumber(posNum)
            .code(l.getCode())
            .sectionId(l.getSection() != null ? l.getSection().getId() : null)
            .sectionName(l.getSection() != null ? l.getSection().getName() : "")
            .status(visualStatus)
            .capacityTarimas(l.getCapacityUnits())
            .currentTarimas(l.getCurrentOccupancy() != null ? l.getCurrentOccupancy() : 0)
            .blockReason(l.getBlockReason())
            .statusReason(l.getStatusReason())
            .isBlocked(l.getIsBlocked())
            .build();
    }
}
```

---

### 3.5 Controller REST: `WarehouseMapController.java`

```java
package com.fourguard.wms.presentation.controller;

import com.fourguard.wms.application.dto.request.map.UpdatePositionStatusMapRequest;
import com.fourguard.wms.application.dto.response.map.CatBlockReasonResponse;
import com.fourguard.wms.application.dto.response.map.PositionMapDetailResponse;
import com.fourguard.wms.application.dto.response.map.WarehouseTopologyResponse;
import com.fourguard.wms.domain.ports.in.WarehouseMapUseCase;
import com.fourguard.wms.shared.response.ApiResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/warehouse-map")
@RequiredArgsConstructor
@Tag(name = "Mapa de Nave 2D", description = "Endpoints de topología espacial, blueprint SVG y gestión de posiciones en tiempo real")
public class WarehouseMapController {

    private final WarehouseMapUseCase mapUseCase;

    @GetMapping("/topology")
    @PreAuthorize("hasAuthority('INVENTORY_READ') or hasRole('OPERATIONS_MANAGER')")
    @Operation(summary = "Obtener topología 2D completa", description = "Retorna las naves del almacén con coordenadas SVG y métricas agregadas de ocupación.")
    public ResponseEntity<ApiResponse<WarehouseTopologyResponse>> getTopology(
            @RequestParam UUID branchId) {
        WarehouseTopologyResponse response = mapUseCase.getTopology(branchId);
        return ResponseEntity.ok(ApiResponse.ok("Topología del almacén recuperada con éxito", response));
    }

    @GetMapping("/sections/{sectionId}/positions")
    @PreAuthorize("hasAuthority('INVENTORY_READ') or hasRole('OPERATIONS_MANAGER')")
    @Operation(summary = "Consultar posiciones de una sección", description = "Retorna la cuadrícula de posiciones de una nave con filtros de estado y búsqueda.")
    public ResponseEntity<ApiResponse<List<PositionMapDetailResponse>>> getPositionsBySection(
            @PathVariable UUID sectionId,
            @RequestParam(required = false) String status,
            @RequestParam(required = false) String search) {
        List<PositionMapDetailResponse> response = mapUseCase.getPositionsBySection(sectionId, status, search);
        return ResponseEntity.ok(ApiResponse.ok("Posiciones recuperadas con éxito", response));
    }

    @PatchMapping("/positions/{positionId}/status")
    @PreAuthorize("hasAuthority('LOCATIONS_UPDATE') or hasRole('OPERATIONS_MANAGER')")
    @Operation(summary = "Actualizar estado operativo de posición", description = "Ejecuta transiciones FSM: BLOCK (bloqueo QM), RELEASE (liberación) u OCCUPY.")
    public ResponseEntity<ApiResponse<PositionMapDetailResponse>> updatePositionStatus(
            @PathVariable UUID positionId,
            @Valid @RequestBody UpdatePositionStatusMapRequest request,
            Authentication authentication) {
        String username = authentication != null ? authentication.getName() : "SYSTEM";
        PositionMapDetailResponse response = mapUseCase.updatePositionStatus(positionId, request, username);
        return ResponseEntity.ok(ApiResponse.ok("Estado de la posición actualizado con éxito", response));
    }

    @GetMapping("/catalogs/block-reasons")
    @PreAuthorize("hasAuthority('INVENTORY_READ') or hasRole('OPERATIONS_MANAGER')")
    @Operation(summary = "Obtener motivos de bloqueo QM", description = "Retorna el catálogo normalizado de causas de bloqueo para inspección de calidad.")
    public ResponseEntity<ApiResponse<List<CatBlockReasonResponse>>> getBlockReasons() {
        List<CatBlockReasonResponse> response = mapUseCase.getActiveBlockReasons();
        return ResponseEntity.ok(ApiResponse.ok("Catálogo de motivos de bloqueo recuperado con éxito", response));
    }
}
```

---

## 4. Frontend (Angular): Patrón Bridge (SDOP Pilar 3)

### 4.1 Puerto de Repositorio: `warehouse-layout.repository.port.ts`

```typescript
import { InjectionToken } from '@angular/core';
import { Observable } from 'rxjs';
import {
  WarehouseSection,
  PositionDetail,
  PositionStatus,
  WarehouseLayoutStats
} from '../models/warehouse-layout.models';

export interface WarehouseLayoutRepositoryPort {
  getSections(): Observable<WarehouseSection[]>;
  getPositionsForSection(sectionId: string, status?: string, query?: string): Observable<PositionDetail[]>;
  updatePositionStatus(
    positionId: string,
    action: 'BLOCK' | 'RELEASE' | 'OCCUPY',
    meta?: { reasonCode?: string; comment?: string }
  ): Observable<PositionDetail>;
  getBlockReasons(): Observable<string[]>;
}

export const WAREHOUSE_LAYOUT_REPOSITORY = new InjectionToken<WarehouseLayoutRepositoryPort>(
  'WAREHOUSE_LAYOUT_REPOSITORY'
);
```

### 4.2 Adaptador HTTP Real: `warehouse-layout-http.adapter.ts`

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, catchError, map, of, throwError } from 'rxjs';
import { environment } from '../../../../../environments/environment';
import { WarehouseLayoutRepositoryPort } from '../ports/warehouse-layout.repository.port';
import {
  WarehouseSection,
  PositionDetail,
  PositionStatus
} from '../models/warehouse-layout.models';

interface ApiResponse<T> {
  success: boolean;
  message: string;
  data: T;
}

@Injectable({
  providedIn: 'root'
})
export class WarehouseLayoutHttpAdapter implements WarehouseLayoutRepositoryPort {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiBaseUrl}/api/v1/warehouse-map`;

  // Fallback cache local en caso de desconexión de red
  private readonly STORAGE_KEY = '4guard_warehouse_layout_resilience_cache';

  getSections(): Observable<WarehouseSection[]> {
    const branchId = environment.defaultBranchId || 'b73f0907-9fa5-4bdf-87db-2eb5e7683936';
    const params = new HttpParams().set('branchId', branchId);

    return this.http.get<ApiResponse<{ sections: any[] }>>(`${this.baseUrl}/topology`, { params }).pipe(
      map(res => res.data.sections.map(s => this.mapSectionFromBackend(s))),
      catchError(err => {
        console.warn('Fallo conexión HTTP con Backend, recuperando cache de resiliencia...', err);
        const cached = localStorage.getItem(this.STORAGE_KEY);
        return cached ? of(JSON.parse(cached)) : throwError(() => err);
      })
    );
  }

  getPositionsForSection(sectionId: string, status?: string, query?: string): Observable<PositionDetail[]> {
    let params = new HttpParams();
    if (status && status !== 'ALL') params = params.set('status', status);
    if (query && query.trim()) params = params.set('search', query.trim());

    return this.http.get<ApiResponse<any[]>>(`${this.baseUrl}/sections/${sectionId}/positions`, { params }).pipe(
      map(res => res.data.map(p => this.mapPositionFromBackend(p)))
    );
  }

  updatePositionStatus(
    positionId: string,
    action: 'BLOCK' | 'RELEASE' | 'OCCUPY',
    meta: { reasonCode?: string; comment?: string } = {}
  ): Observable<PositionDetail> {
    const payload = {
      targetAction: action,
      reasonCode: meta.reasonCode,
      comment: meta.comment
    };

    return this.http.patch<ApiResponse<any>>(`${this.baseUrl}/positions/${positionId}/status`, payload).pipe(
      map(res => this.mapPositionFromBackend(res.data))
    );
  }

  getBlockReasons(): Observable<string[]> {
    return this.http.get<ApiResponse<{ code: string; description: string }[]>>(`${this.baseUrl}/catalogs/block-reasons`).pipe(
      map(res => res.data.map(r => r.description)),
      catchError(() => of([
        'Cuarentena QM — Sospecha de contaminación',
        'Cuarentena QM — Inspección de calidad en proceso',
        'Mantenimiento — Reparación de rack o estructura',
        'Bloqueo administrativo — Pendiente de revisión por supervisor'
      ]))
    );
  }

  private mapSectionFromBackend(raw: any): WarehouseSection {
    return {
      id: raw.id,
      code: raw.code.replace('SEC-ALM-', ''),
      name: raw.name,
      category: raw.category || 'General',
      posFijas: raw.posFijas || 0,
      capacidadTarimas: raw.capacidadTarimas || 0,
      factorEstiba: raw.factorEstiba || '22 tarimas/pos',
      materials: raw.materials || [],
      notes: raw.notes || '',
      status: raw.status === 'LOADED' ? 'LOADED' : 'PENDING',
      polygonPoints: raw.polygonPoints,
      labelPosition: { x: Number(raw.labelPosition?.x || 0), y: Number(raw.labelPosition?.y || 0) },
      sublabelPosition: { x: Number(raw.sublabelPosition?.x || 0), y: Number(raw.sublabelPosition?.y || 0) }
    };
  }

  private mapPositionFromBackend(raw: any): PositionDetail {
    return {
      id: raw.id,
      positionNumber: raw.positionNumber,
      code: raw.code,
      sectionId: raw.sectionId,
      sectionName: raw.sectionName,
      skuCode: raw.skuCode,
      skuDescription: raw.skuDescription || 'Sin Material Asignado',
      status: raw.status as PositionStatus,
      capacityTarimas: raw.capacityTarimas,
      currentTarimas: raw.currentTarimas,
      batchNumber: raw.batchNumber || 'N/A',
      lastMovement: raw.lastMovement || 'Sin movimientos',
      blockReason: raw.blockReason
    };
  }
}
```

### 4.3 Configuración de Inyección de Dependencias (Bridge Provider)
En `app.config.ts` o en los providers del módulo `inventory`:
```typescript
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './core/interceptors/auth.interceptor';
import { WAREHOUSE_LAYOUT_REPOSITORY } from './features/inventory/ports/warehouse-layout.repository.port';
import { WarehouseLayoutHttpAdapter } from './features/inventory/services/warehouse-layout-http.adapter';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor])),
    {
      provide: WAREHOUSE_LAYOUT_REPOSITORY,
      useClass: WarehouseLayoutHttpAdapter // Intercambiable por WarehouseLayoutLocalStorageAdapter en tests unitarios aislados
    }
  ]
};
```

---

## 5. Criterio de Validación con Playwright: Oracle E2E

Para certificar el cumplimiento del **Pilar 2 (Oracle)** de la metodología SDOP de SyborX, el comportamiento se modela a través de la siguiente suite automatizada de pruebas en Playwright (`warehouse-map.spec.ts`).

### Suite de Pruebas: `tests/e2e/warehouse-map.spec.ts`

```typescript
import { test, expect } from '@playwright/test';

test.describe('Oráculo E2E — Mapa Interactivo 2D de Almacén (HU-048 / HU-127)', () => {

  test.beforeEach(async ({ page }) => {
    // 1. Iniciar sesión con perfil SUPERVISOR / OPERATIONS_MANAGER
    await page.goto('/login');
    await page.fill('#username', 'supervisor_cdmx');
    await page.fill('#password', 'SyborX@2026');
    await page.click('button[type="submit"]');
    await page.waitForURL('/dashboard');

    // 2. Navegar al módulo de Mapa Interactivo
    await page.goto('/inventory/map');
    await expect(page.locator('.wmap__title')).toHaveText('Mapa Interactivo de Nave');
  });

  test('TC-01: Renderizado Topológico Inicial y KPIs de Almacén', async ({ page }) => {
    // Interceptar y validar contrato HTTP de topología
    const topologyResponsePromise = page.waitForResponse(resp => 
      resp.url().includes('/api/v1/warehouse-map/topology') && resp.status() === 200
    );

    await page.reload();
    const topologyResponse = await topologyResponsePromise;
    const body = await topologyResponse.json();

    // Verificaciones de estructura
    expect(body.success).toBe(true);
    expect(body.data.sections.length).toBeGreaterThanOrEqual(11);
    expect(body.data.globalStats.totalPositions).toBe(885);

    // Verificaciones visuales en el DOM
    await expect(page.locator('.wmap__kpi-value').first()).toContainText('885');
    await expect(page.locator('.wmap__live-badge')).toBeVisible();

    // Verificar presencia del SVG y de los polígonos de naves activas
    const svg = page.locator('svg.wmap__svg');
    await expect(svg).toBeVisible();
    await expect(page.locator('.wmap__poly--active')).toHaveCount(8); // Naves A, E, F(D), G, I, J(C), L, K
  });

  test('TC-02: Hover en Polígono de Nave y Drill-Down Lateral', async ({ page }) => {
    // Localizar Nave A en el plano SVG
    const polyNaveA = page.locator('polygon[aria-label*="Almacén A"]');
    await expect(polyNaveA).toBeVisible();

    // Hover para desplegar tooltip
    await polyNaveA.hover();
    const tooltip = page.locator('.wmap__tooltip');
    await expect(tooltip).toBeVisible();
    await expect(tooltip).toContainText('Almacén A');
    await expect(tooltip).toContainText('170'); // Posiciones fijas

    // Clic para abrir el panel lateral de drill-down
    await polyNaveA.click();
    const drawer = page.locator('.wmap__detail');
    await expect(drawer).toBeVisible();
    await expect(drawer.locator('.wmap__detail-title')).toContainText('Almacén A');

    // Verificar tarjetas de posiciones dentro del panel
    const posCards = page.locator('.wmap__pos');
    await expect(posCards.first()).toBeVisible();
  });

  test('TC-03: Filtrado Reactivo y Búsqueda en Cuadrícula de Posiciones', async ({ page }) => {
    // Abrir Nave A
    await page.locator('polygon[aria-label*="Almacén A"]').click();

    // Ingresar búsqueda en el input
    const searchInput = page.locator('.wmap__search-input');
    await searchInput.fill('POS-001');

    // Comprobar que solo la posición coincidente permanezca visible
    const filteredCards = page.locator('.wmap__pos');
    await expect(filteredCards).toHaveCount(1);
    await expect(filteredCards.first()).toContainText('POS-001');

    // Limpiar búsqueda
    await page.locator('.wmap__search-clear').click();
    await expect(filteredCards).toHaveCount(170);

    // Cambiar a pestaña "Disponibles"
    await page.locator('button.wmap__tab:has-text("Disponibles")').click();
    await expect(page.locator('.wmap__pos--occupied')).toHaveCount(0);
  });

  test('TC-04: Transición FSM — Bloqueo QM de Posición con Validación Transaccional', async ({ page }) => {
    // Abrir Nave E
    await page.locator('polygon[aria-label*="Almacén E"]').click();

    // Seleccionar la primera posición disponible
    const targetPos = page.locator('.wmap__pos--available').first();
    const posCode = await targetPos.locator('.wmap__pos-code').innerText();
    await targetPos.click();

    // Validar apertura del modal inspector
    const modal = page.locator('.wmap__modal');
    await expect(modal).toBeVisible();
    await expect(modal.locator('.wmap__modal-title')).toContainText(posCode);

    // Clic en "Bloquear QM"
    await modal.locator('button:has-text("Bloquear QM")').click();

    // Seleccionar motivo y agregar comentario
    await modal.locator('select.wmap__form-select').selectOption({ label: 'Cuarentena QM — Sospecha de contaminación' });
    await modal.locator('textarea.wmap__form-textarea').fill('Bloqueo automatizado ejecutado por Playwright Oracle');

    // Interceptar mutación PATCH hacia Spring Boot
    const patchPromise = page.waitForResponse(resp =>
      resp.url().includes('/api/v1/warehouse-map/positions/') &&
      resp.request().method() === 'PATCH' &&
      resp.status() === 200
    );

    // Confirmar Bloqueo
    await modal.locator('button:has-text("Confirmar Bloqueo QM")').click();
    const patchResponse = await patchPromise;
    const patchBody = await patchResponse.json();

    // Validaciones del Oráculo
    expect(patchBody.success).toBe(true);
    expect(patchBody.data.status).toBe('BLOCKED');
    expect(patchBody.data.isBlocked).toBe(true);

    // Comprobar reflejo visual inmediato en el inspector modal
    await expect(modal.locator('.wmap__status-badge')).toHaveText('Bloqueada QM');
    await expect(modal.locator('.wmap__block-alert')).toContainText('Cuarentena QM — Sospecha de contaminación');

    // Cerrar inspector
    await modal.locator('button[title="Cerrar"]').click();

    // Validar que la tarjeta en la cuadrícula ahora tenga clase blocked
    const updatedCard = page.locator(`.wmap__pos:has-text("${posCode}")`);
    await expect(updatedCard).toHaveClass(/wmap__pos--blocked/);
  });

  test('TC-05: Transición FSM — Liberación de Posición Bloqueada', async ({ page }) => {
    // Abrir Nave E
    await page.locator('polygon[aria-label*="Almacén E"]').click();

    // Filtrar por Bloqueadas
    await page.locator('button.wmap__tab:has-text("Bloqueadas")').click();
    const blockedCard = page.locator('.wmap__pos--blocked').first();
    await blockedCard.click();

    const modal = page.locator('.wmap__modal');
    await expect(modal).toBeVisible();

    // Interceptar llamada PATCH
    const releasePromise = page.waitForResponse(resp =>
      resp.url().includes('/api/v1/warehouse-map/positions/') &&
      resp.request().method() === 'PATCH' &&
      resp.status() === 200
    );

    // Clic en "Liberar"
    await modal.locator('button:has-text("Liberar")').click();
    const releaseResponse = await releasePromise;
    const releaseBody = await releaseResponse.json();

    expect(releaseBody.data.status).toBe('AVAILABLE');
    expect(releaseBody.data.isBlocked).toBe(false);

    // Verificar en el modal
    await expect(modal.locator('.wmap__status-badge')).toHaveText('Disponible');
  });

});
```

---

## 6. Matriz de Trazabilidad y Criterios de Aceptación (SDOP Certificación)

| Requerimiento / GAP | Componente / DDL | Prueba Oracle (Playwright) | Criterio de Éxito |
|---|---|---|---|
| **M-01** (Geometría SVG en BD) | `V23: ALTER TABLE wms.warehouse_sections` | `TC-01` | Polígonos SVG cargados dinámicamente desde BD sin hardcoding |
| **M-02** (885 Posiciones Físicas) | `V23: fn_seed_warehouse_section_positions` | `TC-01`, `TC-02` | 885 registros en `wms.locations`; KPI bar muestra 885 |
| **M-03** (API Compuesta Topología) | `WarehouseMapController.getTopology()` | `TC-01` | 1 sola llamada HTTP retorna naves, coordenadas y métricas |
| **I-01** (Normalización Secciones A-L) | `V23: SEC-ALM-K + Naves A-L` | `TC-01` | 8 naves activas reconocidas en el canvas |
| **P-01** (Catálogo de Bloqueos QM) | `wms.cat_block_reasons` + API | `TC-04` | Selector de motivos alimentado por base de datos |
| **D-01** (Eliminación de LocalStorage) | `WarehouseLayoutHttpAdapter` | `TC-04`, `TC-05` | Mutaciones persisten en PostgreSQL y superan recarga de página |

---

*Documento técnico de diseño de software generado bajo la metodología SDOP de SyborX.*  
*Estado: Aprobado para implementación. Ningún código fuente ha sido alterado durante esta fase de diseño.*
