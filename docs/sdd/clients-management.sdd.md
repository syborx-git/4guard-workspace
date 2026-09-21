# SDD — Backend: Gestión de Clientes, Contactos Corporativos y Destinos Físicos

> **Módulo:** `clients`
> **Repositorio:** `4guard_be` · **Rama:** `develop` (MERGED)
> **Arquitectura:** Hexagonal (Ports & Adapters)
> **Versión:** 2.0 (Multi-Contacto & Multi-Bodega / Ship-to Locations)
> **Estado:** 🟢 Implementado y Verificado (10/10 Tests PASS)

---

## 1. Objetivo

Proveer la capa de persistencia, validaciones de negocio y API REST para la gestión integral de **Clientes Depositantes / Owners 3PL** en el WMS 4GUARD, con soporte nativo para:

1. **Datos Corporativos y Fiscales:** Razón Social, RFC/Tax ID, Dirección Fiscal, Teléfono, Email y Contraseña Portal de Autoservicio (`webPortalPassword`).
2. **Matriz Dinámica de Contactos Corporativos (1:N):** Múltiples contactos por cliente categorizados por área (Logística, Finanzas, Calidad, Compras, etc.) con indicador de contacto principal (`isPrimary`).
3. **Direcciones Físicas de Destino / Bodegas / Plantas (1:N):** *Ship-to Locations* utilizadas por los módulos de Despacho/Outbound.
4. **Auditoría Transaccional Relacional:** Bitácora detallada de cambios antes/después (*delta*) por usuario, IP y transacción.

---

## 2. Esquema de Base de Datos (PostgreSQL / schema `wms`)

### Migración Flyway: `V5__client_destinations_and_contacts.sql`

```sql
-- 1. AMPLIAR TABLA MAESTRA DE CLIENTES
ALTER TABLE wms.clients
    ADD COLUMN IF NOT EXISTS address             VARCHAR(300),
    ADD COLUMN IF NOT EXISTS phone               VARCHAR(50),
    ADD COLUMN IF NOT EXISTS email               VARCHAR(150),
    ADD COLUMN IF NOT EXISTS web_portal_password VARCHAR(255);

CREATE UNIQUE INDEX IF NOT EXISTS uk_clients_org_external_id
    ON wms.clients (organization_id, external_id)
    WHERE external_id IS NOT NULL;

-- 2. TABLA DE MATRIZ DE CONTACTOS CORPORATIVOS (1:N)
CREATE TABLE IF NOT EXISTS wms.client_contacts (
    id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    client_id   UUID        NOT NULL REFERENCES wms.clients(id) ON DELETE CASCADE,
    name        VARCHAR(150) NOT NULL,
    department  VARCHAR(100) NOT NULL,
    phone       VARCHAR(50)  NOT NULL,
    email       VARCHAR(150) NOT NULL,
    is_primary  BOOLEAN      DEFAULT FALSE,
    created_at  TIMESTAMPTZ  DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  DEFAULT NOW(),
    created_by  VARCHAR(36),
    updated_by  VARCHAR(36)
);

CREATE UNIQUE INDEX IF NOT EXISTS uk_client_contacts_primary
    ON wms.client_contacts (client_id)
    WHERE is_primary = TRUE;

CREATE INDEX IF NOT EXISTS idx_client_contacts_client_id
    ON wms.client_contacts (client_id);

-- 3. TABLA DE DIRECCIONES FÍSICAS DE DESTINO (1:N / Multi-Bodega)
CREATE TABLE IF NOT EXISTS wms.client_destinations (
    id                  UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    client_id           UUID        NOT NULL REFERENCES wms.clients(id) ON DELETE CASCADE,
    destination_code    VARCHAR(50)  NOT NULL,
    plant_name          VARCHAR(200) NOT NULL,
    full_address        VARCHAR(500) NOT NULL,
    contact_person      VARCHAR(150) NOT NULL,
    phone               VARCHAR(50)  NOT NULL,
    status              VARCHAR(20)  DEFAULT 'ACTIVO',
    notes               TEXT,
    version             BIGINT       DEFAULT 1,
    created_at          TIMESTAMPTZ  DEFAULT NOW(),
    updated_at          TIMESTAMPTZ  DEFAULT NOW(),
    created_by          VARCHAR(36),
    updated_by          VARCHAR(36),
    CONSTRAINT uk_client_destination_code UNIQUE (client_id, destination_code)
);

CREATE INDEX IF NOT EXISTS idx_client_destinations_client_id
    ON wms.client_destinations (client_id);

CREATE INDEX IF NOT EXISTS idx_client_destinations_active
    ON wms.client_destinations (client_id, status)
    WHERE status = 'ACTIVO';
```

---

## 3. Capas de la Arquitectura Hexagonal

### 3.1 Domain Layer

| Componente | Ruta | Descripción |
|---|---|---|
| `Client` | `domain/model/Client.java` | Entidad raíz de dominio con colecciones de contactos y destinos |
| `ClientContact` | `domain/model/ClientContact.java` | Modelo de dominio para contactos corporativos |
| `ClientDestination` | `domain/model/ClientDestination.java` | Modelo de dominio para destinos físicos |
| `ClientUseCase` | `domain/ports/in/ClientUseCase.java` | Puerto de entrada (casos de uso de cliente y destinos) |
| `ClientRepositoryPort` | `domain/ports/out/ClientRepositoryPort.java` | Puerto de persistencia de clientes |
| `ClientDestinationRepositoryPort` | `domain/ports/out/ClientDestinationRepositoryPort.java` | Puerto de persistencia de destinos |

### 3.2 Application Layer

| Componente | Ruta | Descripción |
|---|---|---|
| `ClientContactDto` | `application/dto/common/ClientContactDto.java` | DTO unificado de contactos (requests y response) |
| `PhysicalDestinationDto` | `application/dto/common/PhysicalDestinationDto.java` | DTO unificado de destinos físicos |
| `CreateClientRequest` | `application/dto/request/CreateClientRequest.java` | DTO de creación con validaciones `@Valid`, `@NotBlank`, `@Email` |
| `UpdateClientRequest` | `application/dto/request/UpdateClientRequest.java` | DTO de actualización con soporte de sincronización completa |
| `ClientResponse` | `application/dto/response/ClientResponse.java` | Response con listas embebidas `contacts[]` y `destinations[]` |
| `ClientMapper` | `application/mapper/ClientMapper.java` | MapStruct mapper para transformaciones DTO ↔ Entidad |
| `ClientService` | `application/usecase/ClientService.java` | Orquestador de casos de uso con sincronización inteligente |

### 3.3 Infrastructure Layer

| Componente | Ruta | Descripción |
|---|---|---|
| `ClientEntity` | `persistence/entity/ClientEntity.java` | JPA con `@OneToMany(cascade = ALL, orphanRemoval = true)` |
| `ClientContactEntity` | `persistence/entity/ClientContactEntity.java` | JPA para tabla `wms.client_contacts` |
| `ClientDestinationEntity` | `persistence/entity/ClientDestinationEntity.java` | JPA para tabla `wms.client_destinations` con `@Version` |
| `ClientJpaRepository` | `persistence/repository/ClientJpaRepository.java` | Queries de unicidad por RFC/External ID |
| `ClientContactJpaRepository` | `persistence/repository/ClientContactJpaRepository.java` | Queries de contactos por cliente |
| `ClientDestinationJpaRepository` | `persistence/repository/ClientDestinationJpaRepository.java` | Queries de destinos por cliente y estatus |
| `ClientPersistenceAdapter` | `persistence/adapter/ClientPersistenceAdapter.java` | Implementación del puerto `ClientRepositoryPort` |
| `ClientDestinationPersistenceAdapter` | `persistence/adapter/ClientDestinationPersistenceAdapter.java` | Implementación del puerto `ClientDestinationRepositoryPort` |

### 3.4 Presentation Layer

| Componente | Ruta | Descripción |
|---|---|---|
| `ClientController` | `presentation/controller/ClientController.java` | REST Controller bajo `/clients` con 11 endpoints OpenAPI |

---

## 4. Endpoints REST (OpenAPI 3.0)

| Método | Path | Permiso | Descripción |
|---|---|---|---|
| `POST` | `/api/v1/clients` | `CLIENTS_CREATE` | Registrar cliente con contactos y destinos |
| `PUT` | `/api/v1/clients` | `CLIENTS_UPDATE` | Actualizar cliente (sincronización de colecciones) |
| `GET` | `/api/v1/clients/{id}` | `CLIENTS_READ` | Consultar cliente por UUID |
| `GET` | `/api/v1/clients?organizationId=` | `CLIENTS_READ` | Listar clientes por organización |
| `DELETE` | `/api/v1/clients/{id}` | `CLIENTS_DELETE` | Eliminación física del cliente |
| `PATCH` | `/api/v1/clients/{id}/status` | `CLIENTS_UPDATE` | Baja lógica (toggle ACTIVE ↔ INACTIVE) |
| `GET` | `/api/v1/clients/{id}/audit` | `CLIENTS_READ` | Bitácora de auditoría del cliente |
| `GET` | `/api/v1/clients/{id}/destinations` | `CLIENTS_READ` | Listar destinos físicos de un cliente |
| `POST` | `/api/v1/clients/{id}/destinations` | `CLIENTS_UPDATE` | Agregar destino físico individual |
| `PUT` | `/api/v1/clients/{id}/destinations/{destId}` | `CLIENTS_UPDATE` | Actualizar destino físico individual |
| `DELETE` | `/api/v1/clients/{id}/destinations/{destId}` | `CLIENTS_UPDATE` | Eliminar destino físico individual |

---

## 5. Reglas de Negocio Implementadas

| ID | Regla | Implementación Técnica |
|---|---|---|
| **RN-CLI-001** | External ID / Código ERP único por organización | `validateExternalIdUniqueness()` en `ClientService` + índice `uk_clients_org_external_id` |
| **RN-CLI-002** | Tax ID / RFC único por organización (excepto genéricos `XAXX010101000` / `XEXX010101000`) | `validateTaxIdUniqueness()` en `ClientService` |
| **RN-CLI-003** | Código de destino (`destinationCode`) único por cliente | `validateDestinationCodeForNew()` + constraint `uk_client_destination_code` |
| **RN-CLI-004** | Sincronización inteligente de contactos | `syncContacts()` compara IDs existentes; agrega nuevos (`id=null`), actualiza modificados y elimina huérfanos vía `orphanRemoval = true` |
| **RN-CLI-005** | Sincronización inteligente de destinos | `syncDestinations()` con misma estrategia transaccional |
| **RN-CLI-006** | Baja Lógica operativa | `PATCH /{id}/status` cambia estado sin romper integridad referencial en inventario/pedidos |
| **RN-CLI-007** | Consumo Outbound exclusivo de destinos activos | `ClientDestinationJpaRepository.findByClientIdAndStatus(clientId, "ACTIVO")` |

---

## 6. Sistema de Auditoría Transaccional

* **Tabla Principal:** `wms.audit_logs` (registra `organization_id`, `user_id`, `action`, `entity_type = 'CLIENT'`, `entity_id`, `ip_address`, `user_agent`, `created_at`).
* **Tabla Detalle:** `wms.audit_log_details` (registra `field_name`, `old_value`, `new_value`).
* **Acciones Registradas:**
  - `CLIENT_CREATED`: Registro inicial con todos los campos y conteos iniciales.
  - `CLIENT_UPDATED`: Delta exacto campo por campo de los atributos modificados.
  - `CLIENT_STATUS_CHANGED`: Registro del cambio `ACTIVE ↔ INACTIVE`.
  - `CLIENT_DELETED`: Registro del borrado y estado final previo.
  - `CLIENT_DESTINATION_ADDED`: Alta granular de bodega/planta.
  - `CLIENT_DESTINATION_UPDATED`: Modificación granular de bodega/planta.
  - `CLIENT_DESTINATION_DELETED`: Baja granular de bodega/planta.

---

## 7. Pruebas Unitarias (`ClientServiceTest.java`)

```
Tests run: 10, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS
```

1. ✅ `whenCreateClient_withValidData_thenSuccess`
2. ✅ `whenCreateClient_withDuplicateExternalId_thenThrowValidationException`
3. ✅ `whenCreateClient_withGenericTaxId_thenSuccessEvenIfDuplicate`
4. ✅ `whenUpdateClient_withValidData_thenSuccess`
5. ✅ `whenDeleteClient_withExistingId_thenSuccess`
6. ✅ `whenGetClientById_withInvalidId_thenThrowEntityNotFoundException`
7. ✅ `whenToggleStatus_fromActive_thenBecomesInactive`
8. ✅ `whenGetClientAuditLogs_withExistingId_thenReturnLogs`
9. ✅ `whenAddDestination_withUniqueCode_thenSuccess`
10. ✅ `whenAddDestination_withDuplicateCode_thenThrowValidationException`
