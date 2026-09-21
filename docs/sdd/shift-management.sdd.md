# SDD — Backend: Módulo Turnos y Horarios Operativos (`4guard_be`)

> **Módulo:** `shifts`  
> **Repositorio:** `4guard_be` · **Controlador:** `ShiftController.java`  
> **Arquitectura:** Hexagonal (Ports & Adapters) — Java 21 / Spring Boot 3  
> **Estado:** 🟢 Implementado y Verificado  

---

## 1. Objetivo

Proveer el catálogo de turnos operativos, descansos y tolerancias horarias para 4GUARD WMS:
- Configuración de horas de entrada y salida con soporte para cruces de medianoche.
- Días laborales de la semana y cálculo de horas laborales efectivas descontando descansos.
- Asociación con personal operativo y terminales RF para control de jornada y productividad.

---

## 2. Esquema de Base de Datos (PostgreSQL / schema `wms`)

```sql
CREATE TABLE IF NOT EXISTS wms.shifts (
    id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id   UUID NOT NULL REFERENCES wms.organizations(id),
    code              VARCHAR(50) NOT NULL,
    name              VARCHAR(150) NOT NULL,
    start_time        TIME NOT NULL,
    end_time          TIME NOT NULL,
    tolerance_minutes INT NOT NULL DEFAULT 15,
    working_days      VARCHAR(50)[] NOT NULL,
    breaks            JSONB DEFAULT '[]'::jsonb,
    is_active         BOOLEAN NOT NULL DEFAULT TRUE,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS uk_shifts_org_code ON wms.shifts(organization_id, code);
```

---

## 3. Endpoints REST (`ShiftController`)

| Verbo | Endpoint | Request | Response | Descripción |
|---|---|---|---|---|
| `GET` | `/api/v1/shifts` | `?organizationId=` | `ApiResponse<List<ShiftResponse>>` | Lista de turnos |
| `GET` | `/api/v1/shifts/{id}` | — | `ApiResponse<ShiftResponse>` | Detalle de turno |
| `POST` | `/api/v1/shifts` | `CreateShiftRequest` | `ApiResponse<ShiftResponse>` | Alta de turno |
| `PUT` | `/api/v1/shifts/{id}` | `UpdateShiftRequest` | `ApiResponse<ShiftResponse>` | Modificación de turno |
| `PATCH`| `/api/v1/shifts/{id}/toggle` | — | `ApiResponse<ShiftResponse>` | Activa / Inactiva turno |
| `GET` | `/api/v1/shifts/{id}/audit` | — | `ApiResponse<List<AuditLogResponse>>` | Auditoría con deltas |
