# ADR-005: Encapsulamiento de Estilos per-Componente con Variables `:host`

- **Estado:** Aceptado
- **Fecha:** 2026-07-22
- **Autores:** Equipo 4Guard WMS Frontend
- **Módulos Afectados:** UI Components (`admin-console`)

---

## Contexto y Problema

En proyectos grandes de Angular con CSS global masivo, las reglas de un componente frecuentemente rompen el diseño de otros componentes debido a la colisión de nombres de clases CSS.

## Opciones Evaluadas

1. **CSS Global BEM Monolítico:** Todo el CSS se escribe en `styles.scss`.  
   *Desventaja:* Difícil mantenimiento y archivos CSS gigantescos.
2. **Estilos Aislados por Componente con Variables en `:host`:** Cada componente Standalone mantiene su propio archivo `.css` o `.scss` dedicado (`styleUrl: './component.css'`), definiendo sus tokens locales dentro de `:host`.

## Decisión Tomada

Se adopta la estrategia de **Estilos Aislados por Componente**. Las reglas de diseño globales (paleta de colores, tipografía `Outfit`/`Inter`, mixins) se importan como abstracts, mientras que el diseño de la pantalla reside de forma encapsulada en su propio archivo de estilos.

## Consecuencias

### Positivas
- Cero colisión de clases CSS entre pantallas.
- Mantenibilidad aislada: cambiar el CSS de `user-activity` jamás romperá `carrier-management`.
- Aprovecha la encapsulación de emulación de DOM de Angular (`ViewEncapsulation.Emulated`).

### Negativas / Compromisos
- Es necesario repetir la definición de variables locales en `:host` para nuevos componentes.
