# SDD — Backend: Módulo Reglas y Configuración de Alertas (`4guard_be`)

> **Módulo:** `alerts-config`  
> **Repositorio:** `4guard_be` · **Controlador:** `AlertConfigController.java`  
> **Arquitectura:** Hexagonal (Ports & Adapters) — Java 21 / Spring Boot 3  
> **Estado:** 🟢 Implementado y Verificado  

---

## 1. Objetivo

Proveer el motor de reglas, evaluación de eventos y despacho de notificaciones multicanal para 4GUARD WMS:
- Almacenamiento de reglas por evento operativo y severidad (`INFO`, `WARNING`, `CRITICAL`, `EMERGENCY`).
- Destinatarios heterogéneos (Roles, Usuarios individuales, Teléfonos para SMS/WhatsApp, Webhooks).
- Evaluación con acelerador de frecuencia (*throttle*) para prevenir tormentas de alertas.
- Endpoint de prueba en vivo (`POST /{id}/test`) que despacha notificaciones mock/sandbox.

---

## 2. Esquema de Base de Datos (PostgreSQL / schema `wms`)

```sql
CREATE TABLE IF NOT EXISTS wms.alert_configs (
    id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id   UUID NOT NULL REFERENCES wms.organizations(id),
    code              VARCHAR(50) NOT NULL,
    name              VARCHAR(150) NOT NULL,
    description       VARCHAR(300),
    event_type        VARCHAR(100) NOT NULL,
    severity          VARCHAR(30) NOT NULL,
    channels          JSONB NOT NULL DEFAULT '[]'::jsonb,
    recipients        JSONB NOT NULL DEFAULT '[]'::jsonb,
    message_template  TEXT NOT NULL,
    throttle_minutes  INT NOT NULL DEFAULT 5,
    is_active         BOOLEAN NOT NULL DEFAULT TRUE,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS uk_alert_configs_org_code ON wms.alert_configs(organization_id, code);
```

---

## 3. Endpoints REST (`AlertConfigController`)

| Verbo | Endpoint | Request | Response | Descripción |
|---|---|---|---|---|
| `GET` | `/api/v1/alert-configs` | `?organizationId=` | `ApiResponse<List<AlertConfigResponse>>` | Lista de reglas de alerta |
| `GET` | `/api/v1/alert-configs/{id}` | — | `ApiResponse<AlertConfigResponse>` | Detalle de regla |
| `POST` | `/api/v1/alert-configs` | `CreateAlertConfigRequest` | `ApiResponse<AlertConfigResponse>` | Creación de regla |
| `PUT` | `/api/v1/alert-configs/{id}` | `UpdateAlertConfigRequest` | `ApiResponse<AlertConfigResponse>` | Actualización de regla |
| `PATCH`| `/api/v1/alert-configs/{id}/toggle`| — | `ApiResponse<AlertConfigResponse>` | Alterna activación |
| `POST` | `/api/v1/alert-configs/{id}/test` | `?channel=` | `ApiResponse<AlertTestResult>` | Envío de alerta de prueba |
| `GET` | `/api/v1/alert-configs/{id}/audit`| — | `ApiResponse<List<AuditLogResponse>>` | Historial de auditoría |
