# SDD — Backend: Módulo Divisas y Tipos de Cambio (`4guard_be`)

> **Módulo:** `currency-exchange`  
> **Repositorio:** `4guard_be` · **Controladores:** `CurrencyController.java`, `ExchangeRateController.java`  
> **Arquitectura:** Hexagonal (Ports & Adapters) — Java 21 / Spring Boot 3  
> **Estado:** 🟢 Implementado y Verificado  

---

## 1. Objetivo

Proveer el motor contable multimoneda para el WMS, incluyendo:
- Gestión de monedas con una única moneda base contable por organización (`is_base = true`).
- Mapeo y consumo en vivo de la API de Banco Central (Banxico SIE REST) para paridades oficiales (USD, EUR).
- Persistencia de tasas históricas con cálculo instantáneo de tasa inversa (`inverse_rate = 1 / rate`).
- Bitácora forense de auditoría para ajustes cambiarios manuales.

---

## 2. Esquema de Base de Datos (PostgreSQL / schema `wms`)

```sql
CREATE TABLE IF NOT EXISTS wms.currencies (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES wms.organizations(id),
    code            VARCHAR(3) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    symbol          VARCHAR(10) NOT NULL,
    decimal_places  INT NOT NULL DEFAULT 2,
    is_base         BOOLEAN NOT NULL DEFAULT FALSE,
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS uk_currencies_org_code ON wms.currencies(organization_id, code);
CREATE UNIQUE INDEX IF NOT EXISTS uk_currencies_org_base ON wms.currencies(organization_id) WHERE is_base = TRUE;

CREATE TABLE IF NOT EXISTS wms.exchange_rates (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id     UUID NOT NULL REFERENCES wms.organizations(id),
    base_currency_id    UUID NOT NULL REFERENCES wms.currencies(id),
    target_currency_id  UUID NOT NULL REFERENCES wms.currencies(id),
    rate                NUMERIC(18, 6) NOT NULL,
    inverse_rate        NUMERIC(18, 6) NOT NULL,
    effective_date      DATE NOT NULL,
    source_type         VARCHAR(50) NOT NULL DEFAULT 'MANUAL',
    source_series_id    VARCHAR(50),
    notes               TEXT,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 3. Endpoints REST (`CurrencyController` & `ExchangeRateController`)

| Verbo | Endpoint | Request | Response | Descripción |
|---|---|---|---|---|
| `GET` | `/api/v1/currencies` | `?organizationId=` | `ApiResponse<List<CurrencyResponse>>` | Lista de monedas del catálogo |
| `POST` | `/api/v1/currencies` | `CreateCurrencyRequest` | `ApiResponse<CurrencyResponse>` | Alta de divisa |
| `POST` | `/api/v1/currencies/{id}/set-base`| — | `ApiResponse<CurrencyResponse>` | Designa divisa base contable |
| `GET` | `/api/v1/exchange-rates` | `?currencyId=` | `ApiResponse<List<ExchangeRateResponse>>` | Historial de paridades |
| `POST` | `/api/v1/exchange-rates` | `CreateExchangeRateRequest` | `ApiResponse<ExchangeRateResponse>` | Registro de nueva paridad |
| `GET` | `/api/v1/exchange-rates/banxico/live/{seriesId}` | — | `ApiResponse<BanxicoLiveRateData>` | Cotización en vivo de Banxico SIE |
