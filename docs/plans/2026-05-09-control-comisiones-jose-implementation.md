# Control de Comisiones Agente Jose — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file `index.html` web app for Jose Alonso to track INS commissions per fortnight, calculate the Fernando/Jose/San Gabriel split, and export an Excel backup matching the existing xlsx format.

**Architecture:** Vanilla JS app in one HTML file. Tailwind CDN for styles, Inter font, Chart.js for stats, SheetJS for Excel export. State persists in localStorage. No build step. Deploy via Netlify drag & drop.

**Tech Stack:** HTML5 + Tailwind CSS (CDN) + Vanilla JS (ES modules optional, but inline `<script>` is fine for single-file) + Chart.js (CDN) + SheetJS (CDN) + Inter font (Google Fonts CDN) + localStorage.

**Spec:** [docs/specs/2026-05-09-control-comisiones-jose-design.md](../specs/2026-05-09-control-comisiones-jose-design.md)

**Verification model:** Since this is single-file vanilla JS with no test framework, "tests" are **manual browser checks** at the end of each task. Open `index.html` in Chrome/Edge, follow the verification steps, look at devtools console for errors.

---

## File Structure

Single-file deliverable plus minimal supporting files:

| File | Purpose |
|---|---|
| `index.html` | The whole app — HTML structure + inline `<style>` for any custom CSS + inline `<script>` for all JS |
| `README.md` | How to open, deploy to Netlify, who maintains |
| `.gitignore` | Standard Node-style ignores (we don't use Node, but useful for future) |
| `LICENSE` | Optional MIT (skip unless asked) |

The `index.html` is internally organized in sections (separated by JS comments):

```
1. <head>: meta + CDN imports + custom styles
2. <body>: skeleton + tabs (Pólizas / Estadísticas)
3. Modals (Nueva póliza, Confirmar borrar, Descargar Excel)
4. <script>: 
   4a. Constants (PRODUCTOS, DUENOS, MESES)
   4b. State (localStorage helpers)
   4c. Calculations (comision, reparto, RT cap)
   4d. Date helpers (quincena utils)
   4e. Render: cabecera, tab Pólizas, tab Estadísticas
   4f. Event handlers
   4g. Excel export
   4h. Init
```

---

## Task 1: HTML skeleton + CDN imports + tabs structure

**Files:**
- Create: `index.html`

- [ ] **Step 1: Write the full HTML skeleton**

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Control de Comisiones — Agente Jose</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
  <style>
    body { font-family: 'Inter', system-ui, sans-serif; }
    .tabular { font-variant-numeric: tabular-nums; }
  </style>
</head>
<body class="bg-slate-50 text-slate-900 min-h-screen">
  <header class="bg-white border-b border-slate-200">
    <div class="max-w-7xl mx-auto px-4 py-4 flex items-center justify-between">
      <div>
        <h1 class="text-xl font-bold">Control de Comisiones</h1>
        <p class="text-sm text-slate-600">Agente Fernando Hernández · Operado por Jose Alonso</p>
      </div>
      <nav class="flex gap-2">
        <button data-tab="polizas" class="tab-btn px-4 py-2 rounded-lg font-medium bg-blue-600 text-white">Pólizas</button>
        <button data-tab="stats" class="tab-btn px-4 py-2 rounded-lg font-medium text-slate-600 hover:bg-slate-100">Estadísticas</button>
      </nav>
    </div>
  </header>

  <main class="max-w-7xl mx-auto px-4 py-6">
    <section id="tab-polizas"><!-- Cabecera + tabla irán aquí --></section>
    <section id="tab-stats" class="hidden"><!-- Stats irán aquí --></section>
  </main>

  <script>
    // Tabs
    document.querySelectorAll('.tab-btn').forEach(btn => {
      btn.addEventListener('click', () => {
        const target = btn.dataset.tab;
        document.querySelectorAll('.tab-btn').forEach(b => {
          b.classList.toggle('bg-blue-600', b === btn);
          b.classList.toggle('text-white', b === btn);
          b.classList.toggle('text-slate-600', b !== btn);
          b.classList.toggle('hover:bg-slate-100', b !== btn);
        });
        document.getElementById('tab-polizas').classList.toggle('hidden', target !== 'polizas');
        document.getElementById('tab-stats').classList.toggle('hidden', target !== 'stats');
      });
    });
  </script>
</body>
</html>
```

- [ ] **Step 2: Open `index.html` in browser to verify**

Expected:
- Header shows "Control de Comisiones" + subtitle
- Two pills: "Pólizas" (blue, active) and "Estadísticas" (gray, inactive)
- Clicking "Estadísticas" switches the active pill (visual change only — both sections empty for now)
- Devtools console: no errors. Tailwind CDN warning is OK.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: HTML skeleton + CDN imports + tabs"
```

---

## Task 2: Constants — productos, dueños, meses

**Files:**
- Modify: `index.html` (add inside `<script>` before tabs handler)

- [ ] **Step 1: Add constants block**

Add at the top of the existing `<script>`:

```javascript
// ===== CONSTANTS =====
const PRODUCTOS = [
  { id: 'AUTOS_VOL',   nombre: 'Seguro Voluntario Automóviles',     em: 0.15,  ren: 0.15,  monedas: ['CRC','USD'] },
  { id: 'HOGAR',       nombre: 'Incendio Hogar Comprensivo',        em: 0.21,  ren: 0.21,  monedas: ['CRC','USD'] },
  { id: 'INC_COM',     nombre: 'Incendio Comercial',                em: 0.16,  ren: 0.13,  monedas: ['CRC','USD'] },
  { id: 'INC_MULTI',   nombre: 'Incendio Multirriesgo',             em: 0.16,  ren: 0.13,  monedas: ['CRC','USD'] },
  { id: 'ESTUDIANTIL', nombre: 'Estudiantil',                       em: 0.185, ren: 0.185, monedas: ['CRC'] },
  { id: 'VIDA_COL',    nombre: 'Vida Colectiva',                    em: 0.20,  ren: 0.20,  monedas: ['CRC','USD'] },
  { id: 'VIAJEROS',    nombre: 'Viajeros',                          em: 0.17,  ren: null,  monedas: ['USD'] },
  { id: 'RT',          nombre: 'Riesgos del Trabajo',               em: 0.08,  ren: 0.05,  monedas: ['CRC'] },
  { id: 'EQ_ELEC',     nombre: 'Equipo Eléctrico',                  em: 0.21,  ren: 0.21,  monedas: ['CRC','USD'] },
  { id: 'PROT_CRED',   nombre: 'Protección Crediticia Colectiva',   em: 0.20,  ren: 0.20,  monedas: ['CRC','USD'] },
  { id: 'RC',          nombre: 'Responsabilidad Civil',             em: 0.21,  ren: 0.21,  monedas: ['CRC','USD'] },
  { id: 'PLENISALUD',  nombre: 'Plenisalud',                        em: 0.17,  ren: 0.17,  monedas: ['CRC'] },
];

const DUENOS = {
  FERNANDO:    { id: 'FERNANDO',    nombre: 'Fernando',     porc: 100, color: 'blue',   hex: '#2563EB' },
  JOSE:        { id: 'JOSE',        nombre: 'Jose',         porc: 100, color: 'green',  hex: '#16A34A' },
  SAN_GABRIEL: { id: 'SAN_GABRIEL', nombre: 'San Gabriel',  porc: 50,  color: 'orange', hex: '#EA580C' },
};

const MESES = ['Ene','Feb','Mar','Abr','May','Jun','Jul','Ago','Sep','Oct','Nov','Dic'];
const MESES_LARGO = ['Enero','Febrero','Marzo','Abril','Mayo','Junio','Julio','Agosto','Septiembre','Octubre','Noviembre','Diciembre'];

const RT_TOPE_ANUAL = 2_000_000; // colones
const IVA_RATE = 0.13;
```

- [ ] **Step 2: Verify in browser console**

Open devtools, type `PRODUCTOS.length` → should print `12`.
Type `DUENOS.SAN_GABRIEL.porc` → should print `50`.
Type `PRODUCTOS.find(p => p.id === 'VIAJEROS').ren` → should print `null`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: catalogo de productos, dueños y constantes"
```

---

## Task 3: localStorage state helpers

**Files:**
- Modify: `index.html` (add to `<script>` after constants)

- [ ] **Step 1: Add state module**

```javascript
// ===== STATE =====
const STORAGE_KEY = 'control_comisiones_jose_v1';

function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return { polizas: [], config: { version: 1 } };
    const parsed = JSON.parse(raw);
    if (!parsed.polizas) parsed.polizas = [];
    return parsed;
  } catch (e) {
    console.error('loadState error', e);
    return { polizas: [], config: { version: 1 } };
  }
}

function saveState(state) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
}

function uuid() {
  if (crypto?.randomUUID) return crypto.randomUUID();
  return 'p_' + Date.now() + '_' + Math.random().toString(36).slice(2, 9);
}

function addPoliza(state, poliza) {
  const newPoliza = {
    id: uuid(),
    createdAt: new Date().toISOString(),
    ...poliza,
  };
  state.polizas.push(newPoliza);
  saveState(state);
  return newPoliza;
}

function removePoliza(state, id) {
  state.polizas = state.polizas.filter(p => p.id !== id);
  saveState(state);
}

let STATE = loadState();
```

- [ ] **Step 2: Verify in browser console**

```javascript
addPoliza(STATE, { anio:2026, mes:5, quincena:1, moneda:'CRC', asegurado:'Test', poliza:'X1', tramite:'EMISION', producto:'AUTOS_VOL', prima:1000000, dueno:'FERNANDO' })
STATE.polizas.length  // → 1
loadState().polizas.length  // → 1 (persisted)
removePoliza(STATE, STATE.polizas[0].id)
STATE.polizas.length  // → 0
```

After verifying, clear the test: `localStorage.removeItem(STORAGE_KEY); STATE = loadState();`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: localStorage state helpers (load/save/add/remove)"
```

---

## Task 4: Calculation functions

**Files:**
- Modify: `index.html` (add to `<script>` after state)

- [ ] **Step 1: Add calculation module**

```javascript
// ===== CALCULATIONS =====
function getProducto(id) {
  return PRODUCTOS.find(p => p.id === id);
}

function getCommissionRate(productoId, tramite) {
  const p = getProducto(productoId);
  if (!p) return 0;
  return tramite === 'EMISION' ? p.em : (p.ren ?? 0);
}

function comisionBrutaINS(poliza) {
  const rate = getCommissionRate(poliza.producto, poliza.tramite);
  return poliza.prima * rate;
}

function comisionAsignadaDueno(poliza) {
  const bruta = comisionBrutaINS(poliza);
  const porc = DUENOS[poliza.dueno].porc;
  return bruta * porc / 100;
}

function repartoEntreDuenos(poliza) {
  const bruta = comisionBrutaINS(poliza);
  if (poliza.dueno === 'FERNANDO') return { fernando: bruta, jose: 0, sg: 0 };
  if (poliza.dueno === 'JOSE')     return { fernando: 0, jose: bruta, sg: 0 };
  if (poliza.dueno === 'SAN_GABRIEL') return { fernando: bruta * 0.5, jose: 0, sg: bruta * 0.5 };
  return { fernando: 0, jose: 0, sg: 0 };
}

function totalesGrupo(polizas) {
  let bruta = 0, fernando = 0, jose = 0, sg = 0;
  for (const p of polizas) {
    const r = repartoEntreDuenos(p);
    bruta += comisionBrutaINS(p);
    fernando += r.fernando;
    jose += r.jose;
    sg += r.sg;
  }
  return { bruta, fernando, jose, sg };
}

// RT acumulado anual con tope ¢2M
function rtAcumuladoAnual(polizas, anio) {
  const rt = polizas.filter(p => p.producto === 'RT' && p.anio === anio);
  const bruta = rt.reduce((sum, p) => sum + comisionBrutaINS(p), 0);
  const capeado = Math.min(bruta, RT_TOPE_ANUAL);
  return { bruta, capeado, alcanzado: bruta >= RT_TOPE_ANUAL };
}

// Formato de moneda
function fmtMoney(n, moneda) {
  const sym = moneda === 'CRC' ? '₡' : '$';
  return sym + n.toLocaleString('es-CR', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}
```

- [ ] **Step 2: Verify in browser console**

```javascript
const test = { producto:'AUTOS_VOL', tramite:'EMISION', prima:1000000, dueno:'FERNANDO' };
comisionBrutaINS(test)  // → 150000
repartoEntreDuenos(test)  // → { fernando: 150000, jose: 0, sg: 0 }

const sg = { ...test, dueno:'SAN_GABRIEL' };
repartoEntreDuenos(sg)  // → { fernando: 75000, jose: 0, sg: 75000 }

fmtMoney(150000, 'CRC')  // → "₡150,000.00"
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: funciones de calculo (comision, reparto, RT acumulado)"
```

---

## Task 5: Quincena helpers (date math)

**Files:**
- Modify: `index.html` (add to `<script>`)

- [ ] **Step 1: Add quincena helpers**

```javascript
// ===== QUINCENA HELPERS =====
function quincenaActual() {
  const now = new Date();
  return {
    anio: now.getFullYear(),
    mes: now.getMonth() + 1,                // 1..12
    quincena: now.getDate() <= 15 ? 1 : 2,
  };
}

function quincenaAnterior(q) {
  if (q.quincena === 2) return { ...q, quincena: 1 };
  if (q.mes === 1) return { anio: q.anio - 1, mes: 12, quincena: 2 };
  return { ...q, mes: q.mes - 1, quincena: 2 };
}

function quincenaSiguiente(q) {
  if (q.quincena === 1) return { ...q, quincena: 2 };
  if (q.mes === 12) return { anio: q.anio + 1, mes: 1, quincena: 1 };
  return { ...q, mes: q.mes + 1, quincena: 1 };
}

function formatQuincenaCorto(q) {
  return `Q${q.quincena} ${MESES[q.mes - 1].toUpperCase()} ${String(q.anio).slice(-2)}`;
}

function formatQuincenaLargo(q) {
  return `Quincena ${q.quincena} de ${MESES_LARGO[q.mes - 1]} ${q.anio}`;
}

function quincenaEquals(a, b) {
  return a.anio === b.anio && a.mes === b.mes && a.quincena === b.quincena;
}

function polizasDeQuincena(state, q) {
  return state.polizas.filter(p => p.anio === q.anio && p.mes === q.mes && p.quincena === q.quincena);
}
```

- [ ] **Step 2: Verify in console**

```javascript
quincenaActual()  // → { anio: 2026, mes: 5, quincena: 1 } (varies by today)
quincenaSiguiente({ anio: 2026, mes: 12, quincena: 2 })  // → { anio: 2027, mes: 1, quincena: 1 }
quincenaAnterior({ anio: 2026, mes: 1, quincena: 1 })    // → { anio: 2025, mes: 12, quincena: 2 }
formatQuincenaCorto({ anio: 2026, mes: 3, quincena: 1 })  // → "Q1 MAR 26"
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: quincena helpers (actual, anterior, siguiente, format)"
```

---

## Task 6: Cabecera con selectores y flechas (Pólizas tab)

**Files:**
- Modify: `index.html` (replace `<section id="tab-polizas">` content + add JS)

- [ ] **Step 1: Replace `tab-polizas` HTML**

Replace `<section id="tab-polizas"><!-- ... --></section>` with:

```html
<section id="tab-polizas">
  <!-- Cabecera de selección de quincena -->
  <div class="bg-white rounded-xl border border-slate-200 shadow-sm p-4 mb-6 flex flex-wrap items-center gap-3">
    <div class="flex items-center gap-2">
      <button id="btn-prev-quincena" class="p-2 rounded-lg hover:bg-slate-100 text-slate-600" title="Quincena anterior">←</button>
      <select id="sel-anio" class="px-3 py-2 rounded-lg border border-slate-300 font-medium"></select>
      <select id="sel-mes" class="px-3 py-2 rounded-lg border border-slate-300 font-medium"></select>
      <select id="sel-quincena" class="px-3 py-2 rounded-lg border border-slate-300 font-medium">
        <option value="1">Quincena 1 (1-15)</option>
        <option value="2">Quincena 2 (16-fin)</option>
      </select>
      <button id="btn-next-quincena" class="p-2 rounded-lg hover:bg-slate-100 text-slate-600" title="Quincena siguiente">→</button>
    </div>
    <div class="flex-1"></div>
    <button id="btn-nueva-poliza" class="px-4 py-2 rounded-lg bg-blue-600 text-white font-medium hover:bg-blue-700">+ Nueva póliza</button>
    <button id="btn-descargar-excel" class="px-4 py-2 rounded-lg border border-slate-300 hover:bg-slate-50 font-medium">📥 Descargar Excel</button>
  </div>

  <h2 id="lbl-quincena-actual" class="text-2xl font-bold mb-4"></h2>

  <div id="contenedor-tablas">
    <!-- Tablas colones y dólares irán aquí, render dinámico -->
  </div>
</section>
```

- [ ] **Step 2: Add cabecera state and render**

Add to `<script>`:

```javascript
// ===== UI STATE =====
let UI = {
  quincena: quincenaActual(),
};

// ===== CABECERA RENDER =====
function renderCabecera() {
  const selAnio = document.getElementById('sel-anio');
  const selMes = document.getElementById('sel-mes');
  const selQ = document.getElementById('sel-quincena');

  // Años: 2024..2030 por ahora
  selAnio.innerHTML = '';
  for (let y = 2024; y <= 2030; y++) {
    selAnio.innerHTML += `<option value="${y}">${y}</option>`;
  }
  selAnio.value = UI.quincena.anio;

  // Meses
  selMes.innerHTML = '';
  for (let m = 1; m <= 12; m++) {
    selMes.innerHTML += `<option value="${m}">${MESES_LARGO[m - 1]}</option>`;
  }
  selMes.value = UI.quincena.mes;
  selQ.value = UI.quincena.quincena;

  document.getElementById('lbl-quincena-actual').textContent = formatQuincenaLargo(UI.quincena);
}

document.getElementById('sel-anio').addEventListener('change', e => {
  UI.quincena = { ...UI.quincena, anio: parseInt(e.target.value, 10) };
  renderAll();
});
document.getElementById('sel-mes').addEventListener('change', e => {
  UI.quincena = { ...UI.quincena, mes: parseInt(e.target.value, 10) };
  renderAll();
});
document.getElementById('sel-quincena').addEventListener('change', e => {
  UI.quincena = { ...UI.quincena, quincena: parseInt(e.target.value, 10) };
  renderAll();
});
document.getElementById('btn-prev-quincena').addEventListener('click', () => {
  UI.quincena = quincenaAnterior(UI.quincena);
  renderAll();
});
document.getElementById('btn-next-quincena').addEventListener('click', () => {
  UI.quincena = quincenaSiguiente(UI.quincena);
  renderAll();
});

function renderAll() {
  renderCabecera();
  renderTablas();   // implementado en Task 8
  renderStats();    // implementado en Task 9
}

// stub temporal hasta Task 8
function renderTablas() {
  document.getElementById('contenedor-tablas').innerHTML = '<p class="text-slate-500">Tabla por construir (Task 8)</p>';
}

function renderStats() { /* placeholder Task 9 */ }

renderAll();
```

- [ ] **Step 3: Verify in browser**

- Reload `index.html`
- See cabecera with 3 selectors + flechas + botones
- Click ← → and watch the selectors and label update
- Change a selector and watch the label update
- Console: no errors

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: cabecera con selectores año/mes/quincena + flechas"
```

---

## Task 7: Modal Nueva póliza

**Files:**
- Modify: `index.html` (add modal HTML + open/close + form handling)

- [ ] **Step 1: Add modal HTML at the end of `<body>` before `<script>`**

```html
<!-- Modal Nueva póliza -->
<div id="modal-nueva" class="hidden fixed inset-0 bg-slate-900/50 z-50 flex items-center justify-center p-4">
  <div class="bg-white rounded-xl shadow-xl w-full max-w-lg p-6 max-h-[90vh] overflow-y-auto">
    <h3 class="text-xl font-bold mb-4">Nueva póliza</h3>
    <p class="text-sm text-slate-600 mb-4" id="lbl-nueva-quincena"></p>

    <form id="form-nueva" class="space-y-4">
      <div>
        <label class="block text-sm font-medium text-slate-700 mb-1">Asegurado</label>
        <input name="asegurado" required class="w-full px-3 py-2 border border-slate-300 rounded-lg" />
      </div>
      <div>
        <label class="block text-sm font-medium text-slate-700 mb-1">Número de póliza</label>
        <input name="poliza" required class="w-full px-3 py-2 border border-slate-300 rounded-lg" />
      </div>
      <div>
        <label class="block text-sm font-medium text-slate-700 mb-1">Producto</label>
        <select name="producto" required class="w-full px-3 py-2 border border-slate-300 rounded-lg" id="sel-producto-nueva">
          <option value="">— Seleccione —</option>
        </select>
      </div>
      <div>
        <label class="block text-sm font-medium text-slate-700 mb-1">Trámite</label>
        <div class="flex gap-2" id="grp-tramite">
          <label class="flex-1 border border-slate-300 rounded-lg px-3 py-2 cursor-pointer has-[:checked]:bg-blue-50 has-[:checked]:border-blue-600">
            <input type="radio" name="tramite" value="EMISION" required class="mr-2" /> Emisión
          </label>
          <label class="flex-1 border border-slate-300 rounded-lg px-3 py-2 cursor-pointer has-[:checked]:bg-blue-50 has-[:checked]:border-blue-600">
            <input type="radio" name="tramite" value="RENOVACION" class="mr-2" /> Renovación
          </label>
        </div>
      </div>
      <div>
        <label class="block text-sm font-medium text-slate-700 mb-1">Moneda</label>
        <div class="flex gap-2" id="grp-moneda">
          <label class="flex-1 border border-slate-300 rounded-lg px-3 py-2 cursor-pointer has-[:checked]:bg-green-50 has-[:checked]:border-green-600">
            <input type="radio" name="moneda" value="CRC" required class="mr-2" /> Colones ₡
          </label>
          <label class="flex-1 border border-slate-300 rounded-lg px-3 py-2 cursor-pointer has-[:checked]:bg-blue-50 has-[:checked]:border-blue-600">
            <input type="radio" name="moneda" value="USD" class="mr-2" /> Dólares $
          </label>
        </div>
      </div>
      <div>
        <label class="block text-sm font-medium text-slate-700 mb-1">Prima</label>
        <input type="number" name="prima" required min="0" step="0.01" class="w-full px-3 py-2 border border-slate-300 rounded-lg tabular" />
      </div>
      <div>
        <label class="block text-sm font-medium text-slate-700 mb-2">Dueño de la venta</label>
        <div class="grid grid-cols-3 gap-2">
          <label class="border border-slate-300 rounded-lg px-3 py-3 text-center cursor-pointer has-[:checked]:bg-blue-50 has-[:checked]:border-blue-600 has-[:checked]:text-blue-700 font-medium">
            <input type="radio" name="dueno" value="FERNANDO" required class="hidden" /> 🔵 Fernando
          </label>
          <label class="border border-slate-300 rounded-lg px-3 py-3 text-center cursor-pointer has-[:checked]:bg-green-50 has-[:checked]:border-green-600 has-[:checked]:text-green-700 font-medium">
            <input type="radio" name="dueno" value="JOSE" class="hidden" /> 🟢 Jose
          </label>
          <label class="border border-slate-300 rounded-lg px-3 py-3 text-center cursor-pointer has-[:checked]:bg-orange-50 has-[:checked]:border-orange-600 has-[:checked]:text-orange-700 font-medium">
            <input type="radio" name="dueno" value="SAN_GABRIEL" class="hidden" /> 🟠 San Gabriel
          </label>
        </div>
      </div>

      <div id="rt-warning" class="hidden p-3 rounded-lg bg-yellow-50 border border-yellow-300 text-sm text-yellow-900"></div>
      <div id="form-error" class="hidden p-3 rounded-lg bg-red-50 border border-red-300 text-sm text-red-900"></div>

      <div class="flex gap-2 pt-2">
        <button type="button" id="btn-cancelar-nueva" class="flex-1 px-4 py-2 rounded-lg border border-slate-300 hover:bg-slate-50 font-medium">Cancelar</button>
        <button type="submit" class="flex-1 px-4 py-2 rounded-lg bg-blue-600 text-white font-medium hover:bg-blue-700">Guardar</button>
      </div>
    </form>
  </div>
</div>
```

- [ ] **Step 2: Add modal logic to `<script>`**

```javascript
// ===== MODAL NUEVA POLIZA =====
const modalNueva = document.getElementById('modal-nueva');
const formNueva = document.getElementById('form-nueva');

function openModalNueva() {
  // Llenar selector de productos
  const selProd = document.getElementById('sel-producto-nueva');
  selProd.innerHTML = '<option value="">— Seleccione —</option>' +
    PRODUCTOS.map(p => `<option value="${p.id}">${p.nombre}</option>`).join('');
  formNueva.reset();
  document.getElementById('lbl-nueva-quincena').textContent =
    'Se guardará en: ' + formatQuincenaLargo(UI.quincena);
  document.getElementById('rt-warning').classList.add('hidden');
  document.getElementById('form-error').classList.add('hidden');
  modalNueva.classList.remove('hidden');
}

function closeModalNueva() {
  modalNueva.classList.add('hidden');
}

document.getElementById('btn-nueva-poliza').addEventListener('click', openModalNueva);
document.getElementById('btn-cancelar-nueva').addEventListener('click', closeModalNueva);
modalNueva.addEventListener('click', e => { if (e.target === modalNueva) closeModalNueva(); });

// Reactividad: producto cambia → restringir trámite y moneda
document.getElementById('sel-producto-nueva').addEventListener('change', e => {
  const prod = getProducto(e.target.value);
  if (!prod) return;

  // Trámite
  const radioRenov = formNueva.querySelector('input[name="tramite"][value="RENOVACION"]');
  if (prod.ren === null) {
    radioRenov.checked = false;
    radioRenov.disabled = true;
    radioRenov.closest('label').classList.add('opacity-50','pointer-events-none');
    formNueva.querySelector('input[name="tramite"][value="EMISION"]').checked = true;
  } else {
    radioRenov.disabled = false;
    radioRenov.closest('label').classList.remove('opacity-50','pointer-events-none');
  }

  // Moneda
  ['CRC','USD'].forEach(m => {
    const r = formNueva.querySelector(`input[name="moneda"][value="${m}"]`);
    const allowed = prod.monedas.includes(m);
    r.disabled = !allowed;
    r.closest('label').classList.toggle('opacity-50', !allowed);
    r.closest('label').classList.toggle('pointer-events-none', !allowed);
    if (!allowed) r.checked = false;
  });
  // Si solo hay una moneda permitida, marcarla
  if (prod.monedas.length === 1) {
    formNueva.querySelector(`input[name="moneda"][value="${prod.monedas[0]}"]`).checked = true;
  }

  // Aviso RT tope
  if (prod.id === 'RT') {
    const acum = rtAcumuladoAnual(STATE.polizas, UI.quincena.anio);
    if (acum.alcanzado) {
      const w = document.getElementById('rt-warning');
      w.textContent = `⚠️ El acumulado RT del año ${UI.quincena.anio} ya alcanzó el tope ¢2.000.000. Puedes seguir registrando, pero el sistema mostrará el tope en reportes.`;
      w.classList.remove('hidden');
    }
  }
});

formNueva.addEventListener('submit', e => {
  e.preventDefault();
  const fd = new FormData(formNueva);
  const obj = Object.fromEntries(fd);

  const errBox = document.getElementById('form-error');
  errBox.classList.add('hidden');

  // Validar producto
  const prod = getProducto(obj.producto);
  if (!prod) { errBox.textContent = 'Selecciona un producto.'; errBox.classList.remove('hidden'); return; }
  if (prod.ren === null && obj.tramite === 'RENOVACION') {
    errBox.textContent = `${prod.nombre} solo admite Emisión.`;
    errBox.classList.remove('hidden'); return;
  }
  if (!prod.monedas.includes(obj.moneda)) {
    errBox.textContent = `${prod.nombre} no admite la moneda seleccionada.`;
    errBox.classList.remove('hidden'); return;
  }

  const prima = parseFloat(obj.prima);
  if (!Number.isFinite(prima) || prima <= 0) {
    errBox.textContent = 'La prima debe ser mayor a 0.';
    errBox.classList.remove('hidden'); return;
  }

  addPoliza(STATE, {
    anio: UI.quincena.anio,
    mes: UI.quincena.mes,
    quincena: UI.quincena.quincena,
    moneda: obj.moneda,
    asegurado: obj.asegurado.trim(),
    poliza: obj.poliza.trim(),
    tramite: obj.tramite,
    producto: obj.producto,
    prima,
    dueno: obj.dueno,
  });

  closeModalNueva();
  renderAll();
});
```

- [ ] **Step 3: Verify in browser**

- Click "+ Nueva póliza" → modal opens with current quincena label
- Select "Viajeros" → trámite Renovación se desactiva, moneda Colones se desactiva, marca Dólares
- Select "Estudiantil" → moneda Dólares se desactiva
- Llenar form con AUTOS_VOL, Emisión, ¢, prima 1000000, dueño Fernando → Guardar
- Modal se cierra. Console: `STATE.polizas` muestra 1 entrada
- Recargar página → la póliza persiste

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: modal Nueva poliza con validaciones por producto"
```

---

## Task 8: Tablas de pólizas (colones / dólares) con totales

**Files:**
- Modify: `index.html` (replace `renderTablas` stub)

- [ ] **Step 1: Replace `renderTablas` stub**

```javascript
// ===== RENDER TABLAS POLIZAS =====
function renderTablas() {
  const cont = document.getElementById('contenedor-tablas');
  const polizasQ = polizasDeQuincena(STATE, UI.quincena);
  const colones = polizasQ.filter(p => p.moneda === 'CRC');
  const dolares = polizasQ.filter(p => p.moneda === 'USD');

  cont.innerHTML = `
    ${renderTablaMoneda(colones, 'CRC')}
    <div class="h-6"></div>
    ${renderTablaMoneda(dolares, 'USD')}
  `;

  // Adjuntar handlers de borrar
  cont.querySelectorAll('[data-borrar]').forEach(btn => {
    btn.addEventListener('click', () => {
      const id = btn.dataset.borrar;
      const poliza = STATE.polizas.find(p => p.id === id);
      if (!poliza) return;
      if (confirm(`¿Borrar la póliza de ${poliza.asegurado} (${poliza.poliza})?`)) {
        removePoliza(STATE, id);
        renderAll();
      }
    });
  });
}

function renderTablaMoneda(polizas, moneda) {
  const tit = moneda === 'CRC' ? 'COLONES ₡' : 'DÓLARES $';
  const sym = moneda === 'CRC' ? '₡' : '$';
  const totales = totalesGrupo(polizas);
  const acum = moneda === 'CRC' ? rtAcumuladoAnual(STATE.polizas, UI.quincena.anio) : null;
  const rtBanner = acum?.alcanzado ? `<div class="px-4 py-2 bg-yellow-50 border-b border-yellow-300 text-sm text-yellow-900">⚠️ Tope RT anual ¢2M alcanzado (acumulado: ${fmtMoney(acum.bruta,'CRC')}). En reportes se muestra el tope.</div>` : '';

  if (polizas.length === 0) {
    return `
    <div class="bg-white rounded-xl border border-slate-200 shadow-sm overflow-hidden">
      <div class="px-4 py-3 bg-slate-50 border-b border-slate-200 font-semibold text-sm tracking-wide">${tit}</div>
      <div class="p-8 text-center text-slate-500 text-sm">Sin pólizas en esta moneda para esta quincena.</div>
    </div>`;
  }

  const filas = polizas.map(p => {
    const prod = getProducto(p.producto);
    const rate = getCommissionRate(p.producto, p.tramite);
    const bruta = comisionBrutaINS(p);
    const asignada = comisionAsignadaDueno(p);
    const dueno = DUENOS[p.dueno];
    return `
      <tr class="border-b border-slate-100 hover:bg-slate-50">
        <td class="px-4 py-2 text-sm">${escapeHtml(p.asegurado)}</td>
        <td class="px-4 py-2 text-sm font-mono text-slate-600">${escapeHtml(p.poliza)}</td>
        <td class="px-4 py-2 text-sm">${p.tramite === 'EMISION' ? 'Emisión' : 'Renovación'}</td>
        <td class="px-4 py-2 text-sm">${prod?.nombre ?? p.producto}</td>
        <td class="px-4 py-2 text-sm tabular text-right">${fmtMoney(p.prima, moneda)}</td>
        <td class="px-4 py-2 text-sm tabular text-right">${(rate*100).toFixed(1)}%</td>
        <td class="px-4 py-2 text-sm">
          <span class="inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-medium" style="background-color:${dueno.hex}1a;color:${dueno.hex}">
            ● ${dueno.nombre}
          </span>
        </td>
        <td class="px-4 py-2 text-sm tabular text-right font-medium">${fmtMoney(asignada, moneda)}</td>
        <td class="px-4 py-2 text-right">
          <button data-borrar="${p.id}" class="text-slate-400 hover:text-red-600" title="Eliminar">🗑</button>
        </td>
      </tr>`;
  }).join('');

  return `
    <div class="bg-white rounded-xl border border-slate-200 shadow-sm overflow-hidden">
      <div class="px-4 py-3 bg-slate-50 border-b border-slate-200 font-semibold text-sm tracking-wide">${tit}</div>
      ${moneda==='CRC' ? rtBanner : ''}
      <div class="overflow-x-auto">
        <table class="w-full text-left">
          <thead class="bg-slate-50 text-xs uppercase tracking-wide text-slate-600">
            <tr>
              <th class="px-4 py-2 font-medium">Asegurado</th>
              <th class="px-4 py-2 font-medium">Póliza</th>
              <th class="px-4 py-2 font-medium">Trámite</th>
              <th class="px-4 py-2 font-medium">Producto</th>
              <th class="px-4 py-2 font-medium text-right">Prima</th>
              <th class="px-4 py-2 font-medium text-right">% Com</th>
              <th class="px-4 py-2 font-medium">Dueño</th>
              <th class="px-4 py-2 font-medium text-right">Comisión dueño</th>
              <th class="px-4 py-2 font-medium"></th>
            </tr>
          </thead>
          <tbody>${filas}</tbody>
        </table>
      </div>
      <div class="px-4 py-3 bg-slate-50 border-t border-slate-200 grid grid-cols-2 sm:grid-cols-4 gap-3 text-sm">
        <div>
          <div class="text-xs text-slate-500">Total bruto INS</div>
          <div class="font-bold tabular">${fmtMoney(totales.bruta, moneda)}</div>
        </div>
        <div>
          <div class="text-xs text-blue-600">Para Fernando</div>
          <div class="font-bold tabular text-blue-700">${fmtMoney(totales.fernando, moneda)}</div>
        </div>
        <div>
          <div class="text-xs text-green-600">Para Jose</div>
          <div class="font-bold tabular text-green-700">${fmtMoney(totales.jose, moneda)}</div>
        </div>
        <div>
          <div class="text-xs text-orange-600">Para San Gabriel</div>
          <div class="font-bold tabular text-orange-700">${fmtMoney(totales.sg, moneda)}</div>
        </div>
      </div>
    </div>`;
}

function escapeHtml(s) {
  return String(s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
}
```

- [ ] **Step 2: Verify in browser**

- Reload. La quincena actual muestra 2 cards (¢ y $) con estado vacío
- Crear 3 pólizas:
  1. AUTOS_VOL Emisión ¢1.000.000 Fernando
  2. HOGAR Renovación ¢500.000 Jose
  3. AUTOS_VOL Emisión ¢2.000.000 San Gabriel
- Tabla colones muestra las 3 filas con totales:
  - Total bruto: ¢150.000 + ¢105.000 + ¢300.000 = ¢555.000
  - Para Fernando: ¢150.000 + ¢150.000 (50% SG) = ¢300.000
  - Para Jose: ¢105.000
  - Para San Gabriel: ¢150.000
  - Suma 300+105+150 = 555 ✓
- Validar suma manual con calculadora
- Click 🗑 en una fila → confirm → desaparece

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: tablas de polizas por moneda con totales por dueño"
```

---

## Task 9: Pestaña Estadísticas (KPIs + 4 gráficos)

**Files:**
- Modify: `index.html` (replace tab-stats content + replace renderStats)

- [ ] **Step 1: Replace `<section id="tab-stats">` HTML**

```html
<section id="tab-stats" class="hidden">
  <div class="bg-white rounded-xl border border-slate-200 shadow-sm p-4 mb-6 flex items-center gap-3">
    <label class="text-sm font-medium">Año:</label>
    <select id="sel-anio-stats" class="px-3 py-2 rounded-lg border border-slate-300 font-medium"></select>
  </div>

  <div id="stats-rt-banner" class="hidden mb-4 p-3 rounded-lg bg-yellow-50 border border-yellow-300 text-sm text-yellow-900"></div>

  <div id="stats-kpis" class="grid grid-cols-2 md:grid-cols-5 gap-3 mb-6"></div>

  <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
    <div class="bg-white rounded-xl border border-slate-200 shadow-sm p-4">
      <h3 class="font-semibold mb-3">Comisión total por mes</h3>
      <canvas id="chart-mes"></canvas>
    </div>
    <div class="bg-white rounded-xl border border-slate-200 shadow-sm p-4">
      <h3 class="font-semibold mb-3">Por dueño</h3>
      <canvas id="chart-dueno"></canvas>
    </div>
    <div class="bg-white rounded-xl border border-slate-200 shadow-sm p-4">
      <h3 class="font-semibold mb-3">Top productos</h3>
      <canvas id="chart-producto"></canvas>
    </div>
    <div class="bg-white rounded-xl border border-slate-200 shadow-sm p-4">
      <h3 class="font-semibold mb-3">Por quincena</h3>
      <canvas id="chart-quincena"></canvas>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Replace `renderStats` stub**

```javascript
// ===== RENDER STATS =====
let CHART_INSTANCES = {};
let UI_STATS = { anio: new Date().getFullYear() };

function renderStats() {
  const sel = document.getElementById('sel-anio-stats');
  if (!sel.options.length) {
    for (let y = 2024; y <= 2030; y++) {
      sel.innerHTML += `<option value="${y}">${y}</option>`;
    }
    sel.value = UI_STATS.anio;
    sel.addEventListener('change', e => { UI_STATS.anio = parseInt(e.target.value, 10); renderStats(); });
  }

  const polizas = STATE.polizas.filter(p => p.anio === UI_STATS.anio);
  const totalesCRC = totalesGrupo(polizas.filter(p => p.moneda === 'CRC'));
  const totalesUSD = totalesGrupo(polizas.filter(p => p.moneda === 'USD'));

  // KPIs
  document.getElementById('stats-kpis').innerHTML = `
    ${kpi('Total año ¢', fmtMoney(totalesCRC.bruta,'CRC'), 'slate')}
    ${kpi('Total año $', fmtMoney(totalesUSD.bruta,'USD'), 'slate')}
    ${kpi('Para Fernando ¢', fmtMoney(totalesCRC.fernando,'CRC'), 'blue')}
    ${kpi('Para Jose ¢', fmtMoney(totalesCRC.jose,'CRC'), 'green')}
    ${kpi('Para San Gabriel ¢', fmtMoney(totalesCRC.sg,'CRC'), 'orange')}
  `;

  // RT banner
  const acum = rtAcumuladoAnual(STATE.polizas, UI_STATS.anio);
  const banner = document.getElementById('stats-rt-banner');
  if (acum.alcanzado) {
    banner.textContent = `⚠️ Acumulado RT año ${UI_STATS.anio}: ${fmtMoney(acum.bruta,'CRC')}. Tope ¢2.000.000 alcanzado.`;
    banner.classList.remove('hidden');
  } else {
    banner.classList.add('hidden');
  }

  drawChartMes(polizas);
  drawChartDueno(polizas);
  drawChartProducto(polizas);
  drawChartQuincena(polizas);
}

function kpi(label, value, color) {
  const colorMap = {
    slate: 'text-slate-900',
    blue: 'text-blue-700',
    green: 'text-green-700',
    orange: 'text-orange-700',
  };
  return `
    <div class="bg-white rounded-xl border border-slate-200 shadow-sm p-4">
      <div class="text-xs text-slate-500 mb-1">${label}</div>
      <div class="text-lg font-bold tabular ${colorMap[color]}">${value}</div>
    </div>`;
}

function destroyChart(key) {
  if (CHART_INSTANCES[key]) { CHART_INSTANCES[key].destroy(); delete CHART_INSTANCES[key]; }
}

function drawChartMes(polizas) {
  destroyChart('mes');
  const ctx = document.getElementById('chart-mes');
  const dataCRC = Array(12).fill(0);
  const dataUSD = Array(12).fill(0);
  for (const p of polizas) {
    const bruta = comisionBrutaINS(p);
    if (p.moneda === 'CRC') dataCRC[p.mes - 1] += bruta;
    else dataUSD[p.mes - 1] += bruta;
  }
  CHART_INSTANCES.mes = new Chart(ctx, {
    type: 'bar',
    data: {
      labels: MESES,
      datasets: [
        { label: '₡ Colones', data: dataCRC, backgroundColor: '#15803D' },
        { label: '$ Dólares', data: dataUSD, backgroundColor: '#1D4ED8' },
      ],
    },
    options: { responsive: true, plugins: { legend: { position: 'bottom' } } },
  });
}

function drawChartDueno(polizas) {
  destroyChart('dueno');
  const ctx = document.getElementById('chart-dueno');
  const tot = totalesGrupo(polizas.filter(p => p.moneda === 'CRC'));
  CHART_INSTANCES.dueno = new Chart(ctx, {
    type: 'doughnut',
    data: {
      labels: ['Fernando','Jose','San Gabriel'],
      datasets: [{
        data: [tot.fernando, tot.jose, tot.sg],
        backgroundColor: ['#2563EB','#16A34A','#EA580C'],
      }],
    },
    options: { responsive: true, plugins: { legend: { position: 'bottom' }, title: { display: true, text: 'Distribución ¢' } } },
  });
}

function drawChartProducto(polizas) {
  destroyChart('producto');
  const ctx = document.getElementById('chart-producto');
  const map = {};
  for (const p of polizas) {
    const bruta = comisionBrutaINS(p);
    const key = getProducto(p.producto)?.nombre ?? p.producto;
    map[key] = (map[key] || 0) + bruta;
  }
  const sorted = Object.entries(map).sort((a, b) => b[1] - a[1]).slice(0, 12);
  CHART_INSTANCES.producto = new Chart(ctx, {
    type: 'bar',
    data: {
      labels: sorted.map(s => s[0]),
      datasets: [{ label: 'Comisión bruta', data: sorted.map(s => s[1]), backgroundColor: '#2563EB' }],
    },
    options: { indexAxis: 'y', responsive: true, plugins: { legend: { display: false } } },
  });
}

function drawChartQuincena(polizas) {
  destroyChart('quincena');
  const ctx = document.getElementById('chart-quincena');
  const labels = [];
  const data = [];
  for (let m = 1; m <= 12; m++) {
    for (let q = 1; q <= 2; q++) {
      labels.push(`${MESES[m-1]} Q${q}`);
      const sum = polizas
        .filter(p => p.mes === m && p.quincena === q)
        .reduce((s, p) => s + comisionBrutaINS(p), 0);
      data.push(sum);
    }
  }
  CHART_INSTANCES.quincena = new Chart(ctx, {
    type: 'bar',
    data: { labels, datasets: [{ label: 'Bruta', data, backgroundColor: '#2563EB' }] },
    options: { responsive: true, plugins: { legend: { display: false } } },
  });
}
```

- [ ] **Step 3: Verify in browser**

- Click "Estadísticas" tab
- KPIs aparecen con totales del año
- 4 gráficos renderizan
- Cambiar selector de año → todo se actualiza
- No errores en consola

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: pestaña Estadisticas con KPIs y 4 graficos Chart.js"
```

---

## Task 10: Excel respaldo (modal + generación con SheetJS)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add Excel modal HTML before `<script>`**

```html
<!-- Modal Descargar Excel -->
<div id="modal-excel" class="hidden fixed inset-0 bg-slate-900/50 z-50 flex items-center justify-center p-4">
  <div class="bg-white rounded-xl shadow-xl w-full max-w-md p-6">
    <h3 class="text-xl font-bold mb-4">Descargar respaldo Excel</h3>
    <p class="text-sm text-slate-600 mb-4">¿Qué rango quieres exportar?</p>
    <div class="space-y-2 mb-4">
      <label class="flex items-center gap-2 border border-slate-300 rounded-lg px-3 py-2 cursor-pointer has-[:checked]:bg-blue-50 has-[:checked]:border-blue-600">
        <input type="radio" name="rango" value="quincena" checked /> Esta quincena (<span id="lbl-rango-q"></span>)
      </label>
      <label class="flex items-center gap-2 border border-slate-300 rounded-lg px-3 py-2 cursor-pointer has-[:checked]:bg-blue-50 has-[:checked]:border-blue-600">
        <input type="radio" name="rango" value="mes" /> Este mes (<span id="lbl-rango-m"></span>)
      </label>
      <label class="flex items-center gap-2 border border-slate-300 rounded-lg px-3 py-2 cursor-pointer has-[:checked]:bg-blue-50 has-[:checked]:border-blue-600">
        <input type="radio" name="rango" value="anio" /> Todo el año <span id="lbl-rango-a"></span>
      </label>
    </div>
    <div class="flex gap-2">
      <button id="btn-cancelar-excel" class="flex-1 px-4 py-2 rounded-lg border border-slate-300 hover:bg-slate-50 font-medium">Cancelar</button>
      <button id="btn-confirmar-excel" class="flex-1 px-4 py-2 rounded-lg bg-blue-600 text-white font-medium hover:bg-blue-700">Descargar</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Add Excel logic to `<script>`**

```javascript
// ===== EXCEL EXPORT =====
const modalExcel = document.getElementById('modal-excel');

document.getElementById('btn-descargar-excel').addEventListener('click', () => {
  document.getElementById('lbl-rango-q').textContent = formatQuincenaCorto(UI.quincena);
  document.getElementById('lbl-rango-m').textContent = `${MESES_LARGO[UI.quincena.mes-1]} ${UI.quincena.anio}`;
  document.getElementById('lbl-rango-a').textContent = String(UI.quincena.anio);
  modalExcel.classList.remove('hidden');
});

document.getElementById('btn-cancelar-excel').addEventListener('click', () => modalExcel.classList.add('hidden'));
modalExcel.addEventListener('click', e => { if (e.target === modalExcel) modalExcel.classList.add('hidden'); });

document.getElementById('btn-confirmar-excel').addEventListener('click', () => {
  const rango = document.querySelector('input[name="rango"]:checked').value;
  generarExcel(rango);
  modalExcel.classList.add('hidden');
});

function generarExcel(rango) {
  const wb = XLSX.utils.book_new();
  const quincenas = listarQuincenas(rango);
  const acumRT = rtAcumuladoAnual(STATE.polizas, UI.quincena.anio);

  for (const q of quincenas) {
    for (const moneda of ['CRC','USD']) {
      const polizasQM = polizasDeQuincena(STATE, q).filter(p => p.moneda === moneda);
      if (polizasQM.length === 0) continue;
      const ws = construirHojaXLSX(polizasQM, moneda, acumRT);
      const nombre = `${formatQuincenaCorto(q)} ${moneda === 'CRC' ? '¢' : '$'}`.slice(0, 31);
      XLSX.utils.book_append_sheet(wb, ws, nombre);
    }
  }

  if (wb.SheetNames.length === 0) {
    alert('No hay pólizas en el rango seleccionado.');
    return;
  }

  const ts = new Date().toISOString().slice(0,10).replace(/-/g,'');
  let suffix;
  if (rango === 'quincena') suffix = formatQuincenaCorto(UI.quincena).replaceAll(' ','-');
  else if (rango === 'mes') suffix = `${MESES[UI.quincena.mes-1].toUpperCase()}-${UI.quincena.anio}`;
  else suffix = `ANO-${UI.quincena.anio}`;

  XLSX.writeFile(wb, `comisiones_${suffix}_${ts}.xlsx`);
}

function listarQuincenas(rango) {
  if (rango === 'quincena') return [UI.quincena];
  if (rango === 'mes') return [
    { anio: UI.quincena.anio, mes: UI.quincena.mes, quincena: 1 },
    { anio: UI.quincena.anio, mes: UI.quincena.mes, quincena: 2 },
  ];
  // anio
  const out = [];
  for (let m = 1; m <= 12; m++) for (let q = 1; q <= 2; q++) out.push({ anio: UI.quincena.anio, mes: m, quincena: q });
  return out;
}

function construirHojaXLSX(polizas, moneda, acumRT) {
  const symbol = moneda === 'CRC' ? '¢' : '$';
  const headerRow = ['ASEGURADO','POLIZA','TRAMITE','PRODUCTO',`PRIMA ${symbol}`,'%COM INS',`COMISION INS ${symbol}`,'DUEÑO','%DUEÑO',`COMISION DUEÑO ${symbol}`];

  const rows = [];
  rows.push([]); // fila 1 vacía
  rows.push([]); // fila 2 vacía
  rows.push([]); // fila 3 vacía
  rows.push(headerRow); // fila 4

  let totalBruta = 0, paraFernando = 0, paraJose = 0, paraSG = 0;
  let tieneRT = false;

  for (const p of polizas) {
    const prod = getProducto(p.producto);
    const rate = getCommissionRate(p.producto, p.tramite);
    const bruta = comisionBrutaINS(p);
    const dueno = DUENOS[p.dueno];
    const asignada = bruta * dueno.porc / 100;
    rows.push([
      p.asegurado,
      p.poliza,
      p.tramite === 'EMISION' ? 'EMISION' : 'RENOVACION',
      prod?.nombre ?? p.producto,
      p.prima,
      rate,            // decimal: 0.21 (igual al xlsx original)
      bruta,
      dueno.nombre,
      dueno.porc,
      asignada,
    ]);
    totalBruta += bruta;
    const r = repartoEntreDuenos(p);
    paraFernando += r.fernando;
    paraJose += r.jose;
    paraSG += r.sg;
    if (p.producto === 'RT') tieneRT = true;
  }

  rows.push([]);
  rows.push(['','','','','','','','','Para Fernando', paraFernando]);
  rows.push(['','','','','','','','','Para Jose', paraJose]);
  rows.push(['','','','','','','','','Para San Gabriel', paraSG]);
  rows.push([]);
  rows.push(['','','','','','','','','Total sin impuesto', totalBruta]);
  rows.push(['','','','','','','','','Impuesto (13%)', totalBruta * IVA_RATE]);
  rows.push(['','','','','','','','','Total neto', totalBruta * (1 - IVA_RATE)]);

  if (tieneRT && moneda === 'CRC' && acumRT.alcanzado) {
    rows.push([]);
    rows.push([`Nota: Acumulado RT año = ${acumRT.bruta.toFixed(2)}. Tope anual ¢2.000.000 alcanzado. Reportes capean al tope.`]);
  }

  const ws = XLSX.utils.aoa_to_sheet(rows);
  // Ancho de columnas aproximado
  ws['!cols'] = [
    { wch: 30 }, { wch: 18 }, { wch: 12 }, { wch: 32 }, { wch: 14 },
    { wch: 9 },  { wch: 14 }, { wch: 14 }, { wch: 8 },  { wch: 14 },
  ];
  return ws;
}
```

- [ ] **Step 3: Verify in browser**

- Crear 2-3 pólizas en la quincena actual (mezcladas Fernando/Jose/SG, en ¢ y $)
- Click "Descargar Excel" → seleccionar "Esta quincena" → Descargar
- Abrir el .xlsx en Excel/Sheets
- Hojas: `Q1 MAR 26 ¢`, `Q1 MAR 26 $` (las que tengan datos)
- Cada hoja: header en fila 4, filas de datos, separador, 3 filas Para X, separador, Total sin impuesto / Impuesto / Neto
- Verificar suma manual: Para Fernando + Jose + SG = Total sin impuesto ✓

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: exportar respaldo Excel con SheetJS"
```

---

## Task 11: README + .gitignore

**Files:**
- Create: `README.md`
- Create: `.gitignore`

- [ ] **Step 1: Write `README.md`**

```markdown
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

## Documentación

- Spec: [docs/specs/2026-05-09-control-comisiones-jose-design.md](docs/specs/2026-05-09-control-comisiones-jose-design.md)
- Plan de implementación: [docs/plans/2026-05-09-control-comisiones-jose-implementation.md](docs/plans/2026-05-09-control-comisiones-jose-implementation.md)

## Notas

- **No tiene login.** Cualquiera con acceso al navegador ve los datos.
- **No sincroniza entre dispositivos.** Si Jose cambia de PC, los datos no migran (pero puede exportar Excel y reimportar manualmente — el botón de importación es V2).
- Los porcentajes de comisión INS están **hardcoded en `index.html`**. Si cambian, editar la constante `PRODUCTOS`.
```

- [ ] **Step 2: Write `.gitignore`**

```
# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/
*.swp

# Test exports (xlsx generados en local)
*.xlsx
!docs/**/*.xlsx
```

- [ ] **Step 3: Commit**

```bash
git add README.md .gitignore
git commit -m "docs: README + gitignore"
```

---

## Task 12: Verificación end-to-end + ajustes finales

**Files:**
- Modify: `index.html` (correcciones que surjan)

- [ ] **Step 1: Flujo de prueba completo**

Borrar localStorage primero: en consola → `localStorage.removeItem('control_comisiones_jose_v1'); location.reload();`

Crear las siguientes pólizas (todas en quincena actual):

| # | Asegurado | Producto | Trámite | Moneda | Prima | Dueño | Comisión esperada |
|---|---|---|---|---|---|---|---|
| 1 | Carlos López | Seguro Voluntario Automóviles | Emisión | ¢ | 1,000,000 | Fernando | ₡150,000 |
| 2 | María Rojas | Incendio Hogar Comprensivo | Renovación | ¢ | 500,000 | Jose | ₡105,000 |
| 3 | Coopesg M | Vida Colectiva | Emisión | ¢ | 2,000,000 | San Gabriel | ₡200,000 (50%) |
| 4 | Juan Pérez | Viajeros | Emisión | $ | 500 | Fernando | $85 |
| 5 | Mall Rotonda | Riesgos del Trabajo | Emisión | ¢ | 30,000,000 | Fernando | ₡2,400,000 (RT tope) |

- [ ] **Step 2: Verificar visualmente**

- Tabla colones: 4 filas, total bruto INS = 150,000 + 105,000 + 400,000 + 2,400,000 = ₡3,055,000
- Para Fernando = 150,000 (suya) + 200,000 (50% SG) + 2,400,000 (RT) = ₡2,750,000
- Para Jose = 105,000
- Para San Gabriel = 200,000 (50% del bruto SG = 400,000)
- Suma: 2,750,000 + 105,000 + 200,000 = ₡3,055,000 ✓
- Banner amarillo "Tope RT alcanzado" visible en sección colones (RT acumulado = 2,400,000 > 2,000,000)
- Tabla dólares: 1 fila, Para Fernando = $85
- Estadísticas: KPIs muestran totales correctos, banner RT visible
- 4 gráficos renderizan sin errores

- [ ] **Step 3: Verificar Excel respaldo**

- Descargar "Esta quincena" → 2 hojas (¢ y $)
- Abrir y verificar:
  - Hoja ¢ tiene 4 filas + 3 Para X + Total sin impuesto + Impuesto 13% + Neto + nota RT
  - Hoja $ tiene 1 fila + reparto + impuesto
  - Suma de Para Fernando + Jose + SG = Total sin impuesto ✓

- [ ] **Step 4: Verificar persistencia**

- Cerrar pestaña del navegador
- Reabrir `index.html`
- Datos persisten ✓

- [ ] **Step 5: Verificar mobile**

- Abrir DevTools → Toggle Device Toolbar (responsive)
- Tamaño 375px (iPhone)
- Cabecera se reorganiza, tabla scrollea horizontalmente
- Modal se ve OK
- Gráficos responsive

- [ ] **Step 6: Si surge algún bug, fixear y commit**

Hacer un commit por bug fix con mensaje descriptivo.

- [ ] **Step 7: Commit final E2E pass**

```bash
git add -A
git commit --allow-empty -m "test: E2E manual pass exitoso (5 polizas, multi-moneda, RT tope)"
```

---

## Self-Review Checklist (al terminar la implementación)

- [ ] **Cobertura del spec**:
  - §3 12 productos con % correctos → Task 2
  - §4 stack y schema localStorage → Tasks 1, 3
  - §5 cabecera + selectores + tabla + colores dueño → Tasks 6, 7, 8
  - §6 estadísticas (KPIs + 4 gráficos) → Task 9
  - §7 cálculos (bruta, asignada, reparto) → Task 4
  - §8 RT tope ¢2M → Tasks 4, 7, 8, 9
  - §9 Excel respaldo (formato exacto, 3 rangos) → Task 10
  - §10 identidad visual Clean Minimal Pro → integrado en cada task
  - §11 validaciones → Task 7
- [ ] **No placeholders**: cada step tiene código completo o comando exacto
- [ ] **Type consistency**: todos los `getProducto()`, `getCommissionRate()`, `comisionBrutaINS()` referenciados existen y firman igual

---

## Notas de implementación

- **Sin TDD formal** porque no hay test framework. La verificación es visual + consola + Excel abierto en Sheets/Excel.
- **Frecuencia de commit**: uno por task. Si un task tiene un bug, commit del fix con mensaje claro.
- **Sin worktree**: el plan ya está en una carpeta dedicada del proyecto. Ejecución directa.
- **Tailwind CDN**: usa v3 con la sintaxis `has-[:checked]` que es válida. Si en algún navegador fallan los radios estilo card, usar JS para toggle de clases.
