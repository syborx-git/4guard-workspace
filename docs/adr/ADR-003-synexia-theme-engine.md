# ADR-003: Synexia Theme Engine (`.theme-dark` / `.theme-light`)

- **Estado:** Aceptado
- **Fecha:** 2026-07-18
- **Autores:** Equipo 4Guard WMS Frontend
- **Módulos Afectados:** Design System (`styles/themes/_dark.scss`, `admin-console`)

---

## Contexto y Problema

Los operarios y supervisores del WMS trabajan en ambientes con iluminación cambiante (oficinas iluminadas vs. pasillos oscuros de almacén). El sistema requiere un motor de temas claro/oscuro de alto contraste con estética premium (Midnight Navy `#172033` + Prestige Gold `#c5a86b`).

## Opciones Evaluadas

1. **Angular Material Theming:** Basado en Sass mixins y paletas predeterminadas.  
   *Desventaja:* Difícil personalización para lograr glassmorphism y paletas doradas específicas.
2. **Tailwind Dark Mode:** Clases `dark:` en HTML.  
   *Desventaja:* Acoplamiento alto de utility classes en plantillas HTML.
3. **Synexia Theme Engine (CSS Custom Properties):** Variables CSS globales toggled por clase `.theme-dark` / `.theme-light` en el elemento raíz `<html>` o `<body>`.

## Decisión Tomada

Se implementa **Synexia Theme Engine** utilizando CSS Custom Properties nativas. Los estilos globales definen los tokens en `_dark.scss` y los componentes adaptan su contexto mediante el selector CSS `:host-context(.theme-dark)`.

```css
:host {
  --bg-card: #ffffff;
  --text-primary: #172033;
}
:host-context(.theme-dark) {
  --bg-card: #151f35;
  --text-primary: #e6e4de;
}
```

## Consecuencias

### Positivas
- Cambio de tema instantáneo en runtime sin recalcular o recargar la página.
- Encapsulamiento limpio sin ensuciar las plantillas HTML con clases de utilidad.
- Glassmorphism nativo (`backdrop-filter: blur(20px)`).

### Negativas / Compromisos
- Es necesario mantener manualmente el bloque `:host-context(.theme-dark)` en los CSS de cada componente nuevo.
