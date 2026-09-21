# INFORME DE AUDITORÍA Y DIAGNÓSTICO DE ARQUITECTURA
## Evaluación de Preparación para la Metodología SDOP (Spec-Driven Oracle-Bridge Framework)

**Organización:** SyborX  
**Ecosistema:** 4GUARD WMS 3PL (`4Guard_FE_UI` & `4guard_be`)  
**Rol:** Arquitecto de Software Principal de SyborX  
**Fecha de Auditoría:** 2026-09-20  
**Modo:** SOLO LECTURA (READ-ONLY)  
**Estado:** AUDITORÍA COMPLETADA  

---

## 1. RESUMEN EJECUTIVO Y PUNTUACIÓN DE MADUREZ SDOP

El **Framework SDOP (Spec-Driven Oracle-Bridge Framework)** exige que el software esté gobernado por especificaciones vivas e inmutables (SDD/ADR), con un desacoplamiento estricto de fuentes de datos (Oráculo y Repositorios intercambiables mediante Adaptadores), validación de esquemas inmutables y una suite de pruebas automatizadas que actúe como árbitro/oráculo de veracidad.

Tras la inspección estática y estructural de ambos proyectos (`4Guard_FE_UI` y `4guard_be`), se determina que el ecosistema cuenta con bases sólidas en Backend (DTOs, Mappers, Exception Handlers y Flyway), pero presenta **deficiencias críticas en el Frontend y en la capa de Oráculo Automatizado** para operar bajo SDOP de manera inmediata.

### Puntuación de Madurez Global: **53% / 100%** *(Nivel Moderado — Fase de Transición)*

```
[███████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░] 53%
```

### Desglose por Dimensión Crítica

| Dimensión Evaluada | Peso | Puntuación | Contribución Ponderada | Diagnóstico Rápido |
| :--- | :---: | :---: | :---: | :--- |
| **1. Estructura y Patrón Repositorio en Angular** | 25% | **25%** | 6.25% | **CRÍTICO:** Sin interfaces/puertos Repository; acoplamiento directo a servicios concretos y mocks estáticos. |
| **2. Arquitectura de Capas y DTOs en Spring Boot** | 25% | **85%** | 21.25% | **SÓLIDO:** DTOs + Mappers generalizados y `@RestControllerAdvice`. Fuga parcial de `JpaRepository` en UseCases. |
| **3. Base de Datos y Flyway** | 20% | **85%** | 17.00% | **SÓLIDO:** `ddl-auto: validate` activo en todos los perfiles y 22 migraciones Flyway. Mezcla DDL con DML seeds. |
| **4. Estructura de Workspace y Gobernanza** | 15% | **40%** | 6.00% | **PARCIAL:** Existe documentación rica (ADRs/SDDs) pero fragmentada en subcarpetas; sin gobernanza en raíz ni `.agents/skills/`. |
| **5. Pruebas Automatizadas (Oráculo)** | 15% | **15%** | 2.25% | **CRÍTICO:** Cero Playwright / E2E configurado. Solo 1 prueba unitaria en FE. No hay oráculo automatizado ejecutable. |
| **TOTAL PONDERADO** | **100%** | — | **52.75% (~53%)** | **Requiere Refactorización Previa para SDOP** |

---

## 2. EVALUACIÓN DETALLADA DE LOS 5 PUNTOS CRÍTICOS

### 2.1. Estructura y Patrón Repositorio en Angular (`4Guard_FE_UI`)

* **Estructura de Módulos:** El proyecto no utiliza la estructura tradicional `src/app/modules/`, sino una arquitectura monorepo basada en **Angular 17 Standalone Components** (`apps/admin-console/src/app/features/`, `apps/rf-terminal/src/app/features/`, y `libs/shared-core/`).
* **Llamadas a `HttpClient`:** Ningún componente (`*.component.ts`) inyecta `HttpClient` directamente. Esto cumple con la regla básica de separación visual.
* **Inexistencia del Patrón Repositorio:**
  * Todos los componentes inyectan servicios concretos (ej. `LicenseManagementComponent` inyecta directamente `LicenseManagementService` y `OrganizationService`).
  * No existen interfaces de puerto (`ILicenseRepository`, `ISupplierRepository`) ni tokens de inyección (`InjectionToken`) que permitan desacoplar los componentes de la implementación de transporte o persistencia.
* **Mocks y Adaptadores en `LocalStorage`:**
  * En `libs/shared-core/src/lib/infrastructure/services/backend.service.ts` existe un backend simulado completo que almacena colecciones enteras (`items`, `locations`, `receipts`, `transfer_orders`) en `localStorage` (`4guard_backend_db`).
  * **Alto Acoplamiento en Estado:** `InventoryState` (`libs/shared-core/src/lib/application/state/inventory.state.ts`) inyecta directamente la clase concreta `BackendService`, impidiendo alternar de forma transparente entre datos locales y el API de Spring Boot.
  * Mocks en memoria estáticos: En `admin-console`, features como `user-activity` dependen de `DEFAULT_REPORT_PROFILES` en `user-activity.mock.ts`; `business-rules` y `shifts` tienen archivos `.mock.ts` embebidos directamente en sus servicios.
* **Feature Flags y `environment.ts`:**
  * Los archivos `environment.ts` y `environment.develop.ts` únicamente definen `production`, `envName`, `apiBaseUrl`, `publicAppUrl` y `appVersion`.
  * **No existen Feature Flags** ni selectores como `dataSource: 'API' | 'MOCK' | 'INDEXEDDB'` o `features: { mockInventory: boolean }` para alternar orígenes de datos sin editar código fuente.

---

### 2.2. Arquitectura de Capas y DTOs en Spring Boot (`4guard_be`)

* **Controllers REST y Exposición de Entidades:**
  * Se inspeccionaron los **25 Controllers REST** en `presentation/controller/` (`WmsLicenseController`, `WarehouseReceptionController`, `SupplierController`, `WarehouseOutboundController`, `UserController`, etc.).
  * **100% de cumplimiento:** Ningún controlador expone ni recibe entidades JPA (`*Entity`). Todas las operaciones reciben Request DTOs validados con `@Valid` y devuelven Response DTOs dentro del estándar `ResponseEntity<ApiResponse<T>>`.
* **Capa de Mappers:**
  * Existen **23 clases Mapper** en `application/mapper/` (ej. `WmsLicenseMapper`, `SupplierMapper`, `WarehouseReceptionMapper`, `WarehouseTransferMapper`, etc.) que transforman de forma bidireccional y controlada entre Modelos/Entidades y DTOs.
* **Manejador Global de Excepciones:**
  * Implementado con éxito en `presentation/advice/GlobalExceptionHandler.java` y `DomainExceptionHandler.java` con `@RestControllerAdvice`.
  * Captura tanto excepciones de dominio (`EntityNotFoundException`, `ConflictException`, `ValidationException`) como de validación/seguridad de Spring (`MethodArgumentNotValidException`, `AccessDeniedException`), retornando siempre `ApiResponse.error(...)` con `stacktrace` inhabilitado.
* **Separación de Capas (Controller -> Service -> Repository -> Entity):**
  * La separación teórica es Hexagonal (Controller -> UseCase Port In -> UseCase Impl -> Repository Port Out -> Adapter -> JPA Repo).
  * **Hallazgo / Fuga de Abstracción:** En varias implementaciones de use cases (ej. `WarehouseTransferService`, `WarehouseReceptionService`, `WarehouseOutboundService`, `SupplierService`, `SecurityGateService` e incluso `WmsLicenseService`), el servicio inyecta directamente `*JpaRepository` (ej. `OrganizationJpaRepository`, `InventoryItemJpaRepository`, `SecurityPreCheckinJpaRepository`) y manipula entidades JPA en lugar de utilizar puertos de dominio (`RepositoryPort`). Esto vulnera la inmutabilidad de la capa de aplicación hacia la infraestructura.

---

### 2.3. Base de Datos y Flyway (`4guard_be`)

* **Configuración Hibernate `ddl-auto`:**
  * Inspeccionados `application-local.yml`, `application-dev.yml` y `application-prod.yml`:
    ```yaml
    spring:
      jpa:
        open-in-view: false
        hibernate:
          ddl-auto: validate # Cumplimiento 100%
    ```
  * En todos los entornos se encuentra en `validate`, garantizando que Hibernate nunca modifique el esquema de PostgreSQL en tiempo de ejecución.
* **Flyway y Scripts Inmutables:**
  * Existen **22 scripts de migración** en `/src/main/resources/db/migration/` (`V1__initial_schema.sql` hasta `V22__add_security_gate_checkout_fields.sql`).
  * Todos operan bajo el schema `wms` y coinciden exactamente con las 45 entidades JPA en `infrastructure/persistence/entity`.
* **Observación de Deuda Técnica (Gobernanza Flyway):**
  * Scripts como `V2__initial_test_data.sql`, `V8__seed_forklift_operators.sql`, `V9__seed_real_client_data.sql` y `V13..V16` mezclan DDL estructural con DML de seeding (inserciones masivas de prueba). Bajo SDOP, las migraciones de esquema deben ser puras (DDL), mientras que los seeds de datos deben gobernarse mediante perfiles controlados o scripts reproducibles independientes.

---

### 2.4. Estructura del Workspace y Gobernanza

* **Revisión en la Raíz del Workspace (`/workspace`):**
  * `/docs/adr/`: **NO existe** en la raíz.
  * `/docs/sdd/`: **NO existe** en la raíz.
  * `/docs/gap-analysis/`: **NO existe** en la raíz (se formaliza con este informe).
  * `/.agents/skills/`: **NO existe** en la raíz ni en subcarpetas.
* **Documentación Existente en Subproyectos:**
  * `4Guard_FE_UI/docs/adr/`: Contiene 20 ADRs (`ADR-001` a `ADR-020`).
  * `4Guard_FE_UI/docs/sdd/`: Contiene 19 SDDs funcionales.
  * `4guard_be/docs/adr/`: Contiene 12 ADRs (parcialmente duplicados del frontend).
  * `4guard_be/docs/sdd/`: Contiene 10 SDDs.
* **Diagnóstico de Gobernanza:**
  * La documentación arquitectónica es de alta calidad conceptual, pero está **descentralizada y duplicada** entre repositorios/carpetas, careciendo de un contrato único de oráculo en la raíz del workspace que gobierne a los agentes de IA (`.agents/skills/`).

---

### 2.5. Pruebas Automatizadas como Oráculo

* **Playwright / Pruebas E2E:**
  * **Completamente ausente.** No existe archivo `playwright.config.ts` ni en la raíz ni en subcarpetas.
  * En `4Guard_FE_UI/package.json` no está instalada la dependencia `@playwright/test` ni ningún runner E2E alternativo. Solo figuran `karma` y `jasmine`.
* **Estado de Pruebas Unitarias / Integración:**
  * **Frontend:** Deplorable para oráculo. Solo existe **1 archivo de prueba** (`login.component.spec.ts`) en todo el proyecto Angular. El resto de las 19 features carece de tests automatizados.
  * **Backend:** Cuenta con 20 pruebas unitarias y de integración de use cases/controllers (`MockMvc`), pero no cuenta con un arnés de validación end-to-end con base de datos real en contenedor (Testcontainers) accesible como oráculo determinista.
* **Impacto para SDOP:**
  * El pilar "Oracle" de SDOP no puede operar hoy: no hay ningún mecanismo automatizado que valide si los cambios del agente respetan las especificaciones sin intervención manual del usuario.

---

## 3. MATRIZ DE HALLAZGOS ARQUITECTÓNICOS

| Componente / Clase | Estado Actual | Cambio Requerido para SDOP | Nivel de Riesgo |
| :--- | :---: | :---: | :---: |
| **Angular Feature Components** (`apps/admin-console/src/app/features/*/*.component.ts`) | Inyectan servicios de clase concreta directamente (`LicenseManagementService`, etc.). | Implementar abstracción de interfaces `Repository` e inyectar mediante `InjectionToken` o clases base abstractas. | 🔴 Alto |
| **Backend Mock Service** (`libs/shared-core/.../backend.service.ts`) | Simulación acoplada en `localStorage` invocada directamente por `InventoryState`. | Extraer interfaz `InventoryRepository` y crear `LocalStorageInventoryAdapter` e `HttpInventoryAdapter`. | 🔴 Alto |
| **Feature Data Sources** (`user-activity.service.ts`, `business-rules.*`, etc.) | Mocks hardcodeados en archivos `*.mock.ts` dentro de los servicios. | Desacoplar en adaptadores de oráculo (`MockAdapter` vs `HttpAdapter`) activables por configuración. | 🟡 Medio |
| **Angular Environments** (`environments/environment*.ts`) | Solo alojan URLs base del backend sin flags de alternancia. | Incorporar Feature Flags (`featureFlags: { useMockData: boolean, dataSource: 'API' \| 'MOCK' }`). | 🟡 Medio |
| **Frontend Test Suite** (`4Guard_FE_UI`) | 1 solo test (`login.component.spec.ts`), cero Playwright. | Instalar Playwright E2E y configurar suites de prueba por módulo como oráculo SDOP. | 🔴 Crítico |
| **Spring Boot Use Cases** (`WarehouseTransferService`, `SupplierService`, etc.) | Inyectan directamente `*JpaRepository` y entidades JPA en la capa `application`. | Redirigir el acceso a través de Domain Out-Ports (`*RepositoryPort`) implementados por adaptadores en `infrastructure`. | 🟡 Medio |
| **Spring Boot Controllers** (`presentation/controller/*`) | 100% DTOs, Mappers y `@RestControllerAdvice`. | Mantener intacto. Cumple 100% con los estándares SDOP. | 🟢 Óptimo |
| **Flyway Migrations** (`resources/db/migration/*`) | 22 migraciones activas con `ddl-auto: validate`, pero DML de pruebas mezclado con DDL. | Aislar scripts DML de test/seed en perfiles de inicialización independientes. | 🟢 Bajo |
| **Gobernanza Monorepo** (`/workspace`) | Docs fragmentados en `4Guard_FE_UI/docs/` y `4guard_be/docs/`. Sin `/.agents/skills/`. | Centralizar catálogo `/docs/` en la raíz y desplegar las skills SDOP en `/.agents/skills/`. | 🟡 Medio |

---

## 4. PLAN DE ACCIÓN INMEDIATO (PREVIO A LA MIGRACIÓN DEL PRIMER MÓDULO)

Para alcanzar el **100% de preparación SDOP** de manera segura y sin disrupciones en producción, se debe ejecutar el siguiente plan ordenado por fases de refactorización menor:

### Fase 1: Gobernanza del Workspace y Configuración de Agentes (Esfuerzo: 0.5 días)
1. **Consolidación de Documentación en la Raíz:**
   * Unificar los directorios `docs/adr/` y `docs/sdd/` de ambos subproyectos en `/docs/adr/` y `/docs/sdd/` en la raíz del workspace para crear la Fuente Única de Verdad (SSOT).
2. **Creación de Gobernanza de Agentes:**
   * Crear la estructura `/.agents/skills/` y configurar los skills de oráculo y especificación para Antigravity.

### Fase 2: Habilitación del Oráculo Automatizado (Playwright) (Esfuerzo: 1 día)
1. **Configuración de Playwright:**
   * Inicializar Playwright en la raíz del frontend (`4Guard_FE_UI`):
     ```bash
     npm install -D @playwright/test
     npx playwright install --with-deps
     ```
   * Crear `playwright.config.ts` configurado para interactuar contra el puerto local de Angular (4200) y API (8080).
2. **Oráculo Base de Regresión:**
   * Crear la primera suite de pruebas E2E inmutable para el flujo de autenticación y carga de contexto (`tests/e2e/auth.spec.ts`).

### Fase 3: Arquitectura Hexagonal y Repositorios en Angular (Esfuerzo: 1.5 días)
1. **Diseño del Patrón Repositorio en Angular:**
   * Definir interfaces abstractas de repositorio (Ports) en `shared-core` o en cada módulo (ej. `LicenseRepository`, `SupplierRepository`).
2. **Implementación de Adaptadores:**
   * Crear dos adaptadores por cada dominio:
     * `HttpLicenseRepositoryAdapter` (consume backend real con `HttpClient`).
     * `MockLicenseRepositoryAdapter` (consume datos controlados de prueba).
3. **Feature Flags en `environment.ts`:**
   * Agregar a `environment.ts`:
     ```typescript
     export const environment = {
       production: false,
       dataSource: 'API', // 'API' | 'MOCK'
       featureFlags: {
         useMockAdapters: false,
       }
     };
     ```
4. **Provider Factory Dinámico:**
   * Configurar `app.config.ts` para proveer el adaptador adecuado según el flag del entorno mediante `useClass` o `useFactory`.

### Fase 4: Desacoplamiento de Mocks y LocalStorage (Esfuerzo: 1 día)
1. **Refactorizar `BackendService`:**
   * Reemplazar la inyección directa de `BackendService` en `InventoryState` por la interfaz `InventoryRepository`.
   * Convertir `BackendService` en un `LocalStorageInventoryAdapter` que implemente dicha interfaz.
2. **Limpiar Dependencias Estáticas de Mocks:**
   * Extraer `user-activity.mock.ts` y `business-rules.mock.ts` fuera de la lógica operativa de los servicios hacia adaptadores mock exclusivos de pruebas.

### Fase 5: Saneamiento Menor en Backend (Esfuerzo: 1 día)
1. **Blindar UseCases contra Dependencia Directa de `JpaRepository`:**
   * Asegurar que `WarehouseTransferService`, `WarehouseReceptionService` y `SupplierService` dependan exclusivamente de sus `*RepositoryPort` de dominio, delegando las llamadas a Spring Data JPA a sus respectivos adaptadores de persistencia.

---

## 5. CONCLUSIÓN DEL ARQUITECTO

El proyecto **4Guard** se encuentra a mitad de camino en su preparación metodológica para SDOP (**53%**). Cuenta con una base de datos excelente, gobernada por Flyway e inmutable con Hibernate `validate`, y una capa de Controllers REST que respeta estrictamente los DTOs y el tratamiento de errores.

Sin embargo, **no es recomendable iniciar la migración de módulos hacia SDOP de manera inmediata** sin antes:
1. Implementar la capa de abstracción de **Repositorios y Adaptadores** en Angular.
2. Establecer **Playwright** como Oráculo determinista automatizado.
3. Centralizar las especificaciones en la raíz del workspace (`/docs/`) y crear `/.agents/skills/`.

Una vez ejecutadas las Fases 1 a 3 del Plan de Acción Inmediato, el nivel de madurez superará el **90%**, permitiendo que los agentes de SyborX apliquen la metodología SDOP con total precisión, cero alucinaciones y verificación automática continua.
