# Control de Comisiones — Agente Jose

App web single-file para que **Jose Alonso Hernández** lleve el control de pólizas vendidas bajo la licencia del agente titular **Fernando Hernández** (INS Costa Rica).

## Cómo usar

1. Abrir `index.html` en cualquier navegador moderno (Chrome, Edge, Firefox, Safari)
2. La app guarda los datos localmente en el navegador (`localStorage`)
3. Para enviarle un reporte a Fernando: botón **📥 Descargar Excel**

## Cómo desplegar (Netlify)

1. Ir a https://app.netlify.com/drop
2. Arrastrar la carpeta del proyecto
3. Listo — Netlify da una URL pública

O conectar el repo de GitHub a Netlify para deploy automático en cada push.

## Modelo de negocio

- **Fernando Hernández** — agente titular INS, dueño de la licencia. Recibe las comisiones del INS.
- **Jose Alonso Hernández** — administrativo, vende bajo la licencia de Fernando. Único usuario de la app.
- **Cooperativa San Gabriel** — cliente especial con quien Fernando comparte 50/50 las comisiones.

## Reparto por dueño

- 🔵 Fernando dueño → 100% para Fernando
- 🟢 Jose dueño → 100% para Jose (Fernando le paga lo que Jose vende)
- 🟠 San Gabriel dueño → 50% Fernando + 50% San Gabriel

## Documentación

- Spec: [docs/specs/2026-05-09-control-comisiones-jose-design.md](docs/specs/2026-05-09-control-comisiones-jose-design.md)
- Plan de implementación: [docs/plans/2026-05-09-control-comisiones-jose-implementation.md](docs/plans/2026-05-09-control-comisiones-jose-implementation.md)

## Notas

- **No tiene login.** Cualquiera con acceso al navegador ve los datos.
- **No sincroniza entre dispositivos.** Si Jose cambia de PC, los datos no migran (puede exportar el respaldo Excel para mantener su archivo).
- Los porcentajes de comisión INS están **hardcoded en `index.html`**. Si cambian, editar la constante `PRODUCTOS`.
- El RT tiene tope anual de ¢2.000.000 — la app avisa cuando se alcanza.
