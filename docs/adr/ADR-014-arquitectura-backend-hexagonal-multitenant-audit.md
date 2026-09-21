# ADR-014: Arquitectura Backend Hexagonal (Ports & Adapters), Aislamiento Multi-tenant por Organización y Motor de Auditoría con Deltas

- **Estado:** Aceptado
- **Fecha:** 2026-09-12
- **Autores:** Equipo 4GUARD WMS (Backend & Architecture)
- **Módulos Afectados:** `4guard_be`, `4Guard_FE_UI` (consumidor de APIs y auditoría)

---

## 1. Contexto y Problema

El backend de **4GUARD WMS** (`4guard_be`) administra operaciones logísticas de alta criticidad, transacciones concurrentes de almacén y datos maestros corporativos para múltiples organizaciones y clientes depositantes 3PL:

1. **Riesgo de Fuga de Datos entre Organizaciones:** Al operar en un modelo SaaS multitenant, ninguna consulta o mutación puede exponer o modificar registros pertenecientes a otra organización.
2. **Complejidad y Acoplamiento en la Lógica de Negocio:** La arquitectura tradicional de 3 capas (Controller-Service-Repository) tiende a acoplar la lógica de dominio con anotaciones de base de datos JPA y controladores web.
3. **Exigencia Regulatoria y Trazabilidad Operativa:** En operaciones logísticas, no basta con saber *quién* modificó un registro; se requiere una bitácora forense que registre con exactitud qué campos cambiaron, su valor anterior y su valor nuevo (*Deltas Diferenciales*), expuesta en tiempo real al frontend.

---

## 2. Decisión Tomada

Se formaliza la **Arquitectura Hexagonal (Ports & Adapters)** sobre **Java 21 LTS** y **Spring Boot 3.x**, con aislamiento multi-tenant estricto y un motor unificado de auditoría por deltas.

### 2.1 Estructura en Capas Hexagonales (`com.fourguard.wms`)

```
com.fourguard.wms/
├── domain/                    ← Núcleo de Negocio Puro (Sin dependencias de frameworks)
│   ├── model/                 ← Entidades de dominio, Value Objects y Reglas de Negocio
│   ├── port/
│   │   ├── in/                ← Casos de uso primarios (Input Ports)
│   │   └── out/               ← Interfaces de persistencia y servicios externos (Output Ports)
│   └── exception/             ← Excepciones de dominio tipadas
├── application/               ← Orquestación de Casos de Uso
│   ├── service/               ← Implementación de Input Ports (@Service)
│   ├── dto/                   ← Request / Response DTOs
│   └── mapper/                ← Mappers MapStruct / manuales entre Domain y DTOs
├── infrastructure/            ← Adaptadores Secundarios (Salida)
│   ├── persistence/           ← Adaptadores JPA, repositorios Spring Data y entidades de BD
│   ├── security/              ← Filtros JWT, AuthenticationProvider y UserDetails
│   └── client/                ← Clientes HTTP externos (ej. Banxico SIE REST)
└── presentation/              ← Adaptadores Primarios (Entrada)
    ├── controller/            ← Controladores REST OpenAPI (@RestController)
    ├── advice/                ← Manejador global de excepciones (GlobalExceptionHandler)
    └── filter/                ← Filtros de servlet (CORS, TenantContext, Logging)
```

### 2.2 Política de Aislamiento Multi-tenant por Organización (`organizationId`)
* **Identificador Obligatorio:** Toda tabla en PostgreSQL bajo el esquema `wms` que represente datos de negocio incluye una columna `organization_id UUID NOT NULL REFERENCES wms.organizations(id)`.
* **Inyección y Validación en Contexto:** El `organizationId` se extrae del token JWT validado en cada petición y se valida contra los parámetros de la solicitud.
* **Prohibición de Consultas Globales:** Todo método de repositorio JPA o especificación DEBE filtrar por `organizationId`. Las peticiones que intenten consultar o mutar entidades de una organización ajena retornan inmediatamente `403 Forbidden` o `404 Not Found`.

### 2.3 Motor Unificado de Auditoría con Deltas Diferenciales
* **Estructura del Registro de Auditoría:**
  Toda entidad principal cuenta con una tabla de auditoría asociada (ej. `wms_audit_logs`, `carrier_audit_log`, `client_audit_log`, etc.) con el siguiente esquema:
  ```json
  {
    "id": "UUID",
    "organizationId": "UUID",
    "entityType": "CLIENT | CARRIER | SHIFT | ALERT_CONFIG | LOCATION | ...",
    "entityId": "UUID",
    "action": "CREATE | UPDATE | STATUS_CHANGE | DELETE",
    "performedBy": "username o email del usuario autenticado",
    "performedAt": "2026-09-12T22:15:00Z",
    "ipAddress": "192.168.1.10",
    "deltas": [
      {
        "field": "businessName",
        "oldValue": "Logística San Juan S.A.",
        "newValue": "Logística San Juan S.A. de C.V."
      },
      {
        "field": "status",
        "oldValue": "SUSPENDED",
        "newValue": "ACTIVE"
      }
    ]
  }
  ```
* **Exposición REST Canónica:**
  Todo módulo administrativo expone el endpoint:
  `GET /api/v1/{entityPlural}/{id}/audit`
  consumido directamente por el componente de timeline del frontend.

---

## 3. Consecuencias

### Positivas
- **Independencia del Dominio:** Las reglas de negocio permanecen limpias de anotaciones de base de datos o dependencias web, facilitando pruebas unitarias rápidas y limpias.
- **Seguridad Multi-inquilino Robusta:** Garantía estricta de segregación de datos por organización en toda la capa de persistencia.
- **Trazabilidad Forense Completa:** Los supervisores de almacén y administradores disponen de un historial exacto y visual de todas las mutaciones del sistema.

### Compromisos
- Mayor cantidad de clases e interfaces (puertos y adaptadores) en comparación con un diseño monolítico tradicional.
- Requiere mantenimiento riguroso de mappers y sincronización de tipos entre modelos de dominio y entidades JPA.
