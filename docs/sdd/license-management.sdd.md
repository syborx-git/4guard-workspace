# SDD — Backend: Módulo Gestión de Licencias WMS (`4guard_be`)

> **Módulo:** `license-management`  
> **Repositorio:** `4guard_be` · **Controlador:** `WmsLicenseController.java`  
> **Arquitectura:** Hexagonal (Ports & Adapters) — Java 21 / Spring Boot 3  
> **Estado:** 🟢 Implementado y Verificado  

---

## 1. Objetivo

Controlar y auditar la activación y límites de uso de licencias de 4GUARD WMS:
- Validación de claves criptográficas y verificación de huella digital de hardware.
- Control de cuotas de dispositivos handheld RF, usuarios web y almacenes permitidos.
- Monitoreo de vigencia, expiración y períodos de gracia.

---

## 2. Esquema de Base de Datos (PostgreSQL / schema `wms`)

```sql
CREATE TABLE IF NOT EXISTS wms.wms_licenses (
    id                    UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id       UUID NOT NULL REFERENCES wms.organizations(id),
    license_key           VARCHAR(100) NOT NULL UNIQUE,
    tier                  VARCHAR(30) NOT NULL,
    status                VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    issued_to             VARCHAR(200) NOT NULL,
    max_rf_terminals      INT NOT NULL DEFAULT 10,
    active_rf_terminals   INT NOT NULL DEFAULT 0,
    max_web_users         INT NOT NULL DEFAULT 25,
    active_web_users      INT NOT NULL DEFAULT 0,
    max_warehouses        INT NOT NULL DEFAULT 3,
    issued_at             TIMESTAMPTZ NOT NULL,
    expires_at            TIMESTAMPTZ NOT NULL,
    hardware_fingerprint  VARCHAR(255),
    features_enabled      JSONB NOT NULL DEFAULT '[]'::jsonb,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 3. Endpoints REST (`WmsLicenseController`)

| Verbo | Endpoint | Request | Response | Descripción |
|---|---|---|---|---|
| `GET` | `/api/v1/licenses` | `?organizationId=` | `ApiResponse<List<WmsLicenseResponse>>` | Lista de licencias |
| `GET` | `/api/v1/licenses/active` | — | `ApiResponse<WmsLicenseResponse>` | Licencia vigente actual |
| `POST` | `/api/v1/licenses/activate`| `ActivateLicenseRequest` | `ApiResponse<WmsLicenseResponse>` | Activación de nueva clave |
| `POST` | `/api/v1/licenses/{id}/revoke` | `RevokeLicenseRequest` | `ApiResponse<WmsLicenseResponse>` | Revocación manual |
| `GET` | `/api/v1/licenses/{id}/audit` | — | `ApiResponse<List<AuditLogResponse>>` | Trazabilidad de licencias |
