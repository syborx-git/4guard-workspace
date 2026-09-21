---
name: playwright-oracle
description: Instrucciones y mejores prácticas para la ejecución y mantenimiento del Oráculo de Pruebas Automatizadas con Playwright en el ecosistema 4GUARD.
---

# Skill: Oráculo de Pruebas Playwright (SyborX)

Este skill define la interacción con el oráculo E2E de Playwright configurado en el frontend de 4GUARD.

## Comandos Principales

Ejecutar desde el directorio del frontend (`4Guard_FE_UI`):

- **Ejecutar suite completa en modo headless:**
  ```powershell
  npx playwright test
  ```
- **Ejecutar solo el oráculo de autenticación:**
  ```powershell
  npx playwright test e2e/auth/
  ```
- **Ejecutar con interfaz gráfica interactiva:**
  ```powershell
  npm run test:e2e:ui
  ```
- **Ver reporte HTML del último resultado:**
  ```powershell
  npx playwright show-report
  ```

## Convenciones de Escritura de Pruebas Oráculo

1. **Determinismo:** Las pruebas de oráculo no deben depender de retardos arbitrarios (`page.waitForTimeout`). Deben esperar selectores de estado (`toBeVisible()`, `toHaveURL()`, `toHaveText()`).
2. **Resiliencia ante Mocks y Entornos:** Las pruebas deben poder ejecutarse tanto contra backend real como contra adaptadores mock locales (`dataSource: 'MOCK'`).
3. **Selectores Basados en Accesibilidad o Roles:** Usar `page.getByRole()`, `page.getByLabel()`, o selectores con atributos `data-testid` en lugar de clases CSS volátiles.
