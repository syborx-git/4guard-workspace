# SDD — Backend: Gestión de Transportistas y Líneas Fleteras (`4guard_be`)

> **Módulo:** `carriers`  
> **Repositorio:** `4guard_be` · **Controlador:** `CarrierController.java`  
> **Arquitectura:** Hexagonal (Ports & Adapters) — Java 21 / Spring Boot 3  
> **Estado:** 🟢 Implementado y Verificado  

---

## 1. Objetivo

Proveer el núcleo transaccional, reglas de negocio y API REST para la administración del catálogo de **Líneas Transportistas (Carriers)** en 4GUARD WMS, garantizando:
- Control de unicidad de RFC y códigos CAAT/SCAC por organización (`organization_id`).
- FSM transaccional para estados `ACTIVE`, `SUSPENDED` e `INACTIVE`.
- Bitácora forense de auditoría con almacenamiento de deltas antes/después.

---

## 2. Esquema de Base de Datos (PostgreSQL / schema `wms`)

```sql
CREATE TABLE IF NOT EXISTS wms.carriers (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id     UUID NOT NULL REFERENCES wms.organizations(id),
    code                VARCHAR(50) NOT NULL,
    legal_name          VARCHAR(200) NOT NULL,
    commercial_name     VARCHAR(200) NOT NULL,
    rfc                 VARCHAR(20) NOT NULL,
    caat_code           VARCHAR(50),
    scac_code           VARCHAR(50),
    carrier_type        VARCHAR(50) NOT NULL DEFAULT 'THIRD_PARTY_3PL',
    status              VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    contact_name        VARCHAR(150),
    contact_email       VARCHAR(150),
    contact_phone       VARCHAR(50),
    address             VARCHAR(300),
    city                VARCHAR(100),
    state               VARCHAR(100),
    postal_code         VARCHAR(20),
    authorized_services JSONB DEFAULT '[]'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS uk_carriers_org_rfc ON wms.carriers(organization_id, rfc);
CREATE UNIQUE INDEX IF NOT EXISTS uk_carriers_org_code ON wms.carriers(organization_id, code);
```

---

## 3. Puertos y Adaptadores Hexagonales

```
com.fourguard.wms/
├── domain/model/Carrier.java
├── domain/port/in/CarrierUseCase.java
├── domain/port/out/CarrierRepositoryPort.java
├── application/service/CarrierService.java
├── application/dto/request/CreateCarrierRequest.java
├── application/dto/response/CarrierResponse.java
├── infrastructure/persistence/adapter/CarrierPersistenceAdapter.java
└── presentation/controller/CarrierController.java
```

---

## 4. Contrato de Endpoints REST

| Verbo | Endpoint | Request DTO | Response DTO | Descripción |
|---|---|---|---|---|
| `GET` | `/api/v1/carriers` | `?organizationId=&status=&search=` | `ApiResponse<List<CarrierResponse>>` | Lista filtrada de transportistas |
| `GET` | `/api/v1/carriers/{id}` | — | `ApiResponse<CarrierResponse>` | Detalle de transportista |
| `POST` | `/api/v1/carriers` | `CreateCarrierRequest` | `ApiResponse<CarrierResponse>` | Alta de transportista |
| `PUT` | `/api/v1/carriers/{id}` | `UpdateCarrierRequest` | `ApiResponse<CarrierResponse>` | Actualización de datos |
| `PATCH` | `/api/v1/carriers/{id}/status` | `UpdateCarrierStatusRequest` | `ApiResponse<CarrierResponse>` | Transición de estado FSM |
| `GET` | `/api/v1/carriers/{id}/audit` | — | `ApiResponse<List<AuditLogResponse>>` | Historial de deltas de auditoría |
