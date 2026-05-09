# Control de Comisiones Agente INS — Jose Alonso

**Fecha:** 2026-05-09
**Repo destino:** `jhernandez-vibecode/control-comisiones-agente-ins-jose`
**Path local:** `C:\Users\segur\Desktop\control-comisiones-agente-ins-jose\`
**Solicitante:** Juan Carlos Hernández
**Usuario final:** Jose Alonso Hernández (administrativo)

---

## 1. Propósito

App web single-file para que **Jose Alonso Hernández** (administrativo de Fernando Hernández, agente titular INS) lleve el control de las pólizas vendidas y calcule la comisión que corresponde a cada dueño de venta. La app reemplaza un Excel manual existente.

Al final de cada quincena Jose exporta un respaldo en Excel para enviárselo a Fernando.

---

## 2. Modelo de negocio

### Roles

- **Fernando Hernández** — agente titular INS, dueño de la licencia. Recibe las comisiones del INS.
- **Jose Alonso Hernández** — administrativo, vende bajo la licencia de Fernando. Único usuario de la app.
- **Cooperativa San Gabriel** — cliente especial con quien Fernando comparte 50/50 las comisiones.

### Reglas de "dueño de venta" (% participación del dueño)

- **Fernando dueño** → 100% de la comisión bruta es para Fernando
- **Jose dueño** → 100% de la comisión bruta es para Jose (Fernando le paga lo que Jose vendió)
- **San Gabriel dueño** → 50% para Fernando + 50% para San Gabriel

---

## 3. Catálogo de productos

13 productos con porcentajes de comisión INS (emisión / renovación) y % de IVA en la prima. Los % están hardcoded.

| ID | Nombre | Em | Ren | IVA | Moneda |
|---|---|---:|---:|---:|:---:|
| `AUTOS_VOL` | Seguro Voluntario Automóviles | 15% | 15% | 13% | ¢ / $ |
| `HOGAR` | Incendio Hogar Comprensivo | 21% | 21% | 13% | ¢ / $ |
| `INC_COM` | Incendio Comercial | 16% | 13% | 13% | ¢ / $ |
| `INC_MULTI` | Incendio Multirriesgo | 16% | 13% | 13% | ¢ / $ |
| `ESTUDIANTIL` | Estudiantil | 18.5% | 18.5% | **0%** | ¢ |
| `VIDA_COL` | Vida Colectiva | 20% | 20% | **2%** | ¢ / $ |
| `VIAJEROS` | Viajeros | 17% | — | **0%** | $ obligatorio |
| `RT` | Riesgos del Trabajo | 8% | 5% | **0%** | ¢ |
| `EQ_ELEC` | Equipo Eléctrico | 21% | 21% | 13% | ¢ / $ |
| `PROT_CRED` | Protección Crediticia Colectiva | 20% | 20% | 13% | ¢ / $ |
| `RC` | Responsabilidad Civil | 21% | 21% | 13% | ¢ / $ |
| `PLENISALUD` | Plenisalud | 17% | 17% | 13% | ¢ |
| `INS_MEDICAL` | INS Medical | 24% | 16% | 13% | $ obligatorio |

**Reglas especiales:**

- `VIAJEROS` solo permite **Emisión** (no renovación) y **moneda $**. La UI lo fuerza.
- `ESTUDIANTIL`, `RT` y `PLENISALUD` solo permiten **moneda ¢**. La UI lo fuerza.
- **Coberturas C/D del Hogar Comprensivo y D del Incendio Comercial son ignoradas** — se usa siempre el % base del producto.
- **Riesgos del Trabajo tope anual ¢2.000.000**: ver §6.

---

## 4. Stack y arquitectura

- **Single-file `index.html`**
- **Tailwind CSS** vía CDN
- **Chart.js** vía CDN para los gráficos de la pestaña Estadísticas
- **SheetJS** (`xlsx.full.min.js`) vía CDN para generar el respaldo Excel
- **localStorage** como base de datos (sin backend, sin login)
- **Sin build step**, sin dependencias instalables

Deploy: Netlify (drag & drop o conectado al repo de GitHub).

### Schema localStorage

```jsonc
{
  "polizas": [
    {
      "id": "uuid-v4",
      "anio": 2026,
      "mes": 3,                // 1..12
      "quincena": 1,           // 1 (días 1-15) o 2 (días 16-fin de mes)
      "moneda": "CRC",         // "CRC" | "USD"
      "asegurado": "string",
      "poliza": "string",
      "tramite": "EMISION",    // "EMISION" | "RENOVACION"
      "producto": "AUTOS_VOL", // ID del catálogo
      "prima": 1000000,        // número
      "dueno": "FERNANDO",     // "FERNANDO" | "JOSE" | "SAN_GABRIEL"
      "createdAt": "2026-05-09T..."
    }
  ],
  "config": {
    "version": 1
  }
}
```

---

## 5. UI — Pestaña "Pólizas"

### Cabecera fija

```
[Año 2026 ▾]  [Mes Mar ▾]  [Quincena 1 ▾]   ← →    [+ Nueva póliza]   [📥 Descargar Excel]
```

- **Al cargar la app**: la quincena queda en la actual según fecha de hoy.
- **← →**: saltan ±1 quincena, cruzando meses y años automáticamente.
- **Pestañas globales** (arriba del todo): `Pólizas` | `Estadísticas`.

### Modal "Nueva póliza"

Campos:

1. **Asegurado** (texto, obligatorio)
2. **# Póliza** (texto, obligatorio)
3. **Producto** (dropdown 12 productos)
4. **Trámite** (radio Emisión / Renovación) — bloqueado a Emisión si producto = Viajeros
5. **Moneda** (radio ¢ / $) — bloqueada según producto si aplica
6. **Prima** (numérico, obligatorio, > 0)
7. **Dueño** (radio grande con colores: 🔵 Fernando · 🟢 Jose · 🟠 San Gabriel)

Botones: **Guardar** · **Cancelar**.

### Tabla principal

Dos secciones verticales (siempre visibles, aunque vacías):

```
═══ COLONES ═══
Asegurado | Póliza | Trámite | Producto | Prima ¢ | %Com | Dueño | Comisión ¢ | 🗑

(filas)

Total bruto ¢:  Fernando ¢X   Jose ¢Y   San Gabriel ¢Z   |   TOTAL ¢W

═══ DÓLARES ═══
(misma estructura)

Total bruto $:  Fernando $X   Jose $Y   San Gabriel $Z   |   TOTAL $W
```

- "Comisión ¢/$" en cada fila = **comisión asignada al dueño** (ver §7), no la bruta INS.
- "Total bruto" al pie es la suma de comisiones brutas INS (lo que Fernando recibe del INS).
- "Fernando / Jose / San Gabriel" al pie son los 3 totales asignados (ver §7).
- Los chips de dueño usan el código de color (🔵🟢🟠).
- El icono 🗑 elimina la fila con confirmación.

---

## 6. UI — Pestaña "Estadísticas"

### Cabecera

- Selector de **año** (default: año actual)
- KPIs: `Total año ¢` · `Total año $` · `Para Fernando` · `Para Jose` · `Para San Gabriel`
- Banner amarillo si la comisión RT acumulada del año iguala o supera ¢2.000.000

### 4 gráficos (grid 2×2)

1. **Barras** — Comisión total por mes (12 barras). Toggle ¢ / $.
2. **Pie** — Comisión por dueño (Fernando · Jose · San Gabriel). Toggle ¢ / $.
3. **Barras horizontales** — Top productos del año por comisión generada.
4. **Barras** — Comisión por quincena (24 barras del año).

---

## 7. Cálculos

### Términos

- **Comisión bruta INS** = lo que Fernando (titular) recibe del INS por la póliza
- **% participación dueño** = porcentaje que le toca al dueño identificado en la fila (100 ó 50)
- **Comisión asignada al dueño** = lo que el dueño identificado en la fila recibe
- **Para Fernando / Jose / San Gabriel** = totales por persona después de aplicar las reglas de reparto

### Por póliza

La prima que se ingresa es la **prima total** (lo que paga el asegurado, puede incluir IVA según producto). La comisión se calcula sobre la **prima sin impuesto**:

```
%comINS              = catálogo[producto][tramite]   // ej. AUTOS_VOL emisión = 15%
ivaProducto          = catálogo[producto][iva]       // 0, 0.02 o 0.13 según producto
primaSinImpuesto     = primaTotal / (1 + ivaProducto)
comisionBrutaINS     = primaSinImpuesto × %comINS

%dueno               = 50 si dueño = San Gabriel, sino 100
comisionAsignadaDueno = comisionBrutaINS × %dueno / 100
```

**IVA por producto:**
- 0% (exento): Viajeros, Estudiantil, Riesgos del Trabajo
- 2%: Vida Colectiva
- 13% (default): los demás

**Aclaración: dos 13% distintos en el flujo:**
1. **IVA sobre la prima** (variable por producto) — la app lo extrae para calcular la comisión
2. **Retención del 13% al agente** sobre la comisión que recibe del INS — aparece al pie del Excel respaldo, siempre 13% sin importar el producto

### Reparto a las 3 personas (al sumar todas las pólizas)

| Dueño de la póliza | Para Fernando | Para Jose | Para San Gabriel |
|---|---:|---:|---:|
| Fernando | bruta | 0 | 0 |
| Jose | 0 | bruta | 0 |
| San Gabriel | bruta × 0.5 | 0 | bruta × 0.5 |

**Validación interna por póliza:** Para Fernando + Para Jose + Para San Gabriel = comisionBrutaINS.
**Validación al pie del Excel:** suma de los 3 totales = total bruto INS = total que Fernando recibe del INS.

### Totales

Totales por moneda separadamente. **No hay conversión ¢↔$**.

### Impuesto en el Excel respaldo

El xlsx actual aplica una línea **"Impuesto 13%"** al pie de la hoja, calculada sobre el **total bruto INS**:

- `Total sin impuesto` = suma de comisiones brutas INS de la hoja
- `Impuesto` = `Total sin impuesto × 0.13`
- `Total neto` = `Total sin impuesto − Impuesto`

*(Esto representa la retención de IVA del INS al agente titular.)*

---

## 8. Riesgos del Trabajo — Tope ¢2.000.000

Reglas:

- La app calcula RT al 8% emisión / 5% renovación normalmente.
- Lleva un **acumulado anual de comisión bruta RT** (suma de todas las pólizas RT del año en curso).
- **Cuando el acumulado anual supera ¢2.000.000**:
  - El **valor mostrado** se capea visualmente a ¢2.000.000 (en tablas, KPIs y Excel)
  - Aparece un **badge amarillo "Tope ¢2M alcanzado"** junto al valor
  - El modal "Nueva póliza" muestra una advertencia si el dueño selecciona un RT que llevará al tope
  - En el Excel: nota al pie de la hoja que contenga RT

El cálculo individual de cada póliza RT no se altera; solo el agregado anual mostrado.

---

## 9. Excel respaldo

### Botón

`📥 Descargar Excel` en la cabecera de la pestaña Pólizas. Abre un modal con 2 grupos de opciones:

**Rango:**
- **Esta quincena** (default)
- **Este mes** (las 2 quincenas del mes seleccionado)
- **Todo el año** (24 quincenas)

**Agente:**
- **Todos** (default — incluye todas las pólizas con las 3 filas Para X al pie)
- **Fernando** (filtra por dueno=Fernando)
- **Jose** (filtra por dueno=Jose)
- **San Gabriel** (filtra por dueno=San Gabriel)

Cuando se filtra por agente: fila 1 muestra título `REPORTE: <AGENTE>`. El nombre del archivo agrega sufijo `_<AGENTE>`.

### Estructura del archivo .xlsx

Una hoja por **quincena × moneda** que tenga datos:

- Nombre hoja: `Q1 MAR 26 ¢` o `Q2 MAR 26 $` (corto, máx 31 caracteres)
- Si una quincena no tiene pólizas en ¢ o en $, esa hoja se omite

### Formato de cada hoja

Inspirado en el xlsx actual de COOPESANGABRIEL, ampliado con las columnas necesarias para distinguir productos y dueños:

```
Fila 1-3: vacías (igual al original)

Fila 4: encabezados (11 columnas)
ASEGURADO | POLIZA | TRAMITE | PRODUCTO | PRIMA TOTAL ¢|$ | PRIMA SIN IMP ¢|$ | %COM INS | COMISION INS | DUEÑO | %DUEÑO | COMISION DUEÑO

Filas 5..N: una fila por póliza
- PRIMA y COMISION en la moneda de la hoja (¢ o $)
- %COM INS escrito como decimal (0.21) tal como el xlsx original
- %DUEÑO en número entero (100 ó 50)
- DUEÑO en texto: Fernando / Jose / San Gabriel

Fila N+2: separador

Fila N+3: Para Fernando      = suma de COMISION DUEÑO donde dueño=Fernando + bruta×0.5 donde dueño=SanGabriel
Fila N+4: Para Jose          = suma de COMISION DUEÑO donde dueño=Jose
Fila N+5: Para San Gabriel   = suma de COMISION DUEÑO donde dueño=SanGabriel

Fila N+7: Total sin impuesto = suma de COMISION INS (bruta) — debe coincidir con suma de las 3 anteriores
Fila N+8: Impuesto (13%)     = Total sin impuesto × 0.13
Fila N+9: Total neto         = Total sin impuesto − Impuesto

Si la hoja contiene RT y se aplicó el tope ¢2M: nota al pie.
```

### Nombre del archivo

`comisiones_<rango>_<timestamp>.xlsx`

Ejemplos:
- `comisiones_Q1-MAR-2026_20260509.xlsx`
- `comisiones_MAR-2026_20260509.xlsx`
- `comisiones_AÑO-2026_20260509.xlsx`

---

## 10. Identidad visual

**Estilo:** Clean Minimal Pro (Linear / Notion / Stripe Dashboard)

| Token | Valor |
|---|---|
| Fondo página | `#F8FAFC` (slate-50) |
| Fondo tarjetas | `#FFFFFF` |
| Texto principal | `#0F172A` (slate-900) |
| Texto secundario | `#475569` (slate-600) |
| Borde | `#E2E8F0` (slate-200) |
| Acento principal | `#2563EB` (blue-600) |
| Hover acento | `#1D4ED8` (blue-700) |
| Éxito | `#16A34A` (green-600) |
| Advertencia | `#CA8A04` (yellow-600) |
| Error | `#DC2626` (red-600) |
| Sombra tarjetas | `0 1px 3px rgba(0,0,0,0.05)` |
| Radius tarjetas | `12px` |
| Radius inputs/botones | `8px` |

**Colores semánticos por dueño** (consistentes en toda la app — chips, gráficos, totales):

- 🔵 Fernando — `#2563EB` (blue-600)
- 🟢 Jose — `#16A34A` (green-600)
- 🟠 San Gabriel — `#EA580C` (orange-600)

**Colores semánticos por moneda:**

- ¢ Colones — `#15803D` (green-700, fondo `#DCFCE7`)
- $ Dólares — `#1D4ED8` (blue-700, fondo `#DBEAFE`)

**Tipografía:** Inter (Google Fonts), pesos 400/500/600/700.

- Display/títulos: 24-32px, weight 600-700
- Body: 14-16px, weight 400
- Labels: 12-14px, weight 500
- **Datos numéricos**: usar `tabular-nums` (font-variant-numeric) para evitar saltos de columna

---

## 11. UX — Validaciones y reglas de captura

- Prima > 0 obligatoria
- Asegurado y póliza no vacíos
- Si producto = Viajeros y trámite = Renovación → error "Viajeros solo permite Emisión"
- Si producto requiere ¢ y se intenta guardar en $ → error con mensaje claro
- Confirmación antes de eliminar póliza
- localStorage corruption → mensaje + opción "Importar respaldo Excel" (futuro, no en V1)

---

## 12. Out of scope V1

- Login / multiusuario
- Sincronización entre dispositivos
- Importar Excel histórico (Jose ingresa manualmente la quincena en curso, no migra histórico)
- Conversión ¢↔$
- Reportes PDF
- Coberturas C/D especiales

---

## 13. Criterios de éxito

- Jose puede ingresar 10 pólizas en menos de 5 minutos
- El Excel respaldo abre sin errores en Excel/Google Sheets
- El Excel respaldo replica visualmente el formato del xlsx actual
- Los totales por dueño cuadran con la suma manual
- Funciona en Chrome / Edge / Safari de desktop y celular
- No hay pérdida de datos al cerrar y reabrir la app
