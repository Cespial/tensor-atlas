---
name: atlas-deploy
description: Use when the atlas web app runs locally and you need to deploy it to Vercel under a tensor.lat domain with data and visual QA.
---

# Atlas Deploy (Fase 5)

## Overview
Despliega el front Nuxt del Atlas (MapLibre + PMTiles + API REST Nitro) a Vercel bajo un subdominio `*.tensor.lat`, con DOS bloqueos de calidad antes de publicar: QA de datos (los JSON/GeoJSON cumplen el contrato) y QA visual página por página (mapa y paneles). No se publica si cualquiera de los dos QA falla.

## Cuándo usar
- El atlas corre local (`npm run dev`) y toca llevarlo a producción.
- Hay que conectar/verificar el dominio (CABA usa `caba.tensor.lat`, hardcodeado en `nuxt.config.ts`).
- Tras un rebuild del pipeline (`scripts/build_atlas_caba.py`) hay que redesplegar y re-QA.
- Antes de compartir la URL con cliente: pre-publicación.

REQUIRED antes de tocar datos: si vienes de re-territorializar, confirma el contrato con la skill `tensor-atlas:atlas-scout` o lee `${CLAUDE_PLUGIN_ROOT}/references/data-contract.md`.

## Procedimiento

### 1. Build local (SSR, no estático)
El preset es `nitro.preset: 'vercel'` (SSR serverless, NO static) porque la API REST `server/api/**` debe desplegar como funciones. Por eso **NO uses `nuxt generate`** salvo que quieras un sitio sin API. El comando de Vercel es `nuxt build` (`npm run build`).

- `npm run build` corre `prebuild` → `scripts/sync-api-assets.mjs`, que copia `atlas.geojson`, `atlas_stats_v3.json`, `gap_analysis.json` de `public/data/` a `server/assets/data/` (los 3 server-assets que lee la API). `server/assets/data/` está en `.gitignore`: se regenera en cada build, no se commitea.
- Verifica que el build termine sin errores y que existan los 3 assets en `server/assets/data/`.

### 2. Deploy a Vercel
- Proyecto ya linkeado en `.vercel/project.json` (`projectName: atlas-caba-web`). `framework: "nuxtjs"` en `vercel.json`.
- Preview: `vercel`. Producción: `vercel --prod` (o usa la skill `vercel:deploy` con arg `prod`).
- `vercel.json` fija headers de `/data/*.pmtiles` (octet-stream, cache 30d immutable, CORS `*`) y `/data/*.geojson` (geo+json, cache 1d). Confirma que el PMTiles carga con esos headers en prod (sin ellos MapLibre falla con range-requests).
- Revisa build logs en Vercel; si falla, lee el log raíz antes de reintentar.

### 3. Dominio *.tensor.lat
- `nuxt.config.ts` ya referencia `https://caba.tensor.lat` en canonical, OG y twitter. El dominio del proyecto **debe** ser ese para que esas URLs no rompan.
- Añade el dominio: `vercel domains add caba.tensor.lat` y/o asígnalo al proyecto en el dashboard (Settings → Domains). Para otro territorio usa `<slug>.tensor.lat` y actualiza las URLs hardcodeadas de `nuxt.config.ts` en el mismo PR.
- `tensor.lat` es la zona DNS: crea el CNAME del subdominio apuntando a Vercel. `[VERIFICAR]` dónde está delegada la zona DNS de `tensor.lat`.

### 4. QA de datos (bloqueante)
Corre sobre `public/data/` (y confirma paridad con `server/assets/data/`). Valores de referencia CABA verificados: `atlas.geojson` = **3554** features, `municipios.geojson` = **15**, `veredas.geojson` = **48**, `atlas_stats_v3.json.municipios` = **15**.

- [ ] **Scores en [0,1]:** todo `atlas_score` y `score_*` no-null debe estar en [0,1] (la rampa `COLOR_EXPR_TEMPLATE` asume ese dominio). Fuera de rango → colores rotos.
- [ ] **Sin NaN/Infinity:** ningún campo numérico es `NaN`/`Infinity` (JSON los serializa mal y rompen el parse en cliente).
- [ ] **Conteos cuadran:** nº features de `atlas.geojson` = `_meta.radios` de `atlas_stats_v3.json`; nº `municipios.geojson` = `_meta.comunas` = `len(atlas_stats_v3.municipios)` = nº `MUNICIPIOS`−1 (sin "Todos").
- [ ] **Claves de agregación coinciden:** todo `municipio` de `atlas.geojson` existe como clave en `atlas_stats_v3.municipios` y como `MUNICIPIOS[].nombre` en `app/stores/atlas.js` (REGLA DURA del contrato — si no, la cámara no vuela y `stats[municipio]` no resuelve).
- [ ] **`zona_atlas` válido:** valores ∈ {HH,LL,HL,LH,NS}. En CABA todo es `"NS"` (LISA no calculado) — esperado, documéntalo, no es bug.
- [ ] **`quintil`:** ∈ {Q1..Q5} o null.
- [ ] **Stats coherentes:** por municipio `min ≤ p25 ≤ p75 ≤ max`, `min ≤ avg ≤ max`, y clave `"Todos"` presente en `atlas_stats.json`. `comunas_centroides.json` 1er elemento = `"Todos"`.
- [ ] **Paridad server-assets:** los 3 archivos de `server/assets/data/` son idénticos a los de `public/data/` (re-corre `npm run sync:api-assets` si dudas).

Verifica con un script Python/node ad-hoc que recorra los GeoJSON; reporta cada check como PASS/FAIL con el número real.

### 5. QA visual página por página (bloqueante)
Delega en el patrón de la skill `pdf-deliverable` (Read de cada captura), pero sobre la web en vez de un PDF. Si existe un visual companion del plugin, úsalo; si no, captura con headless browser (Playwright/`vercel screenshot` o `mcp__claude-in-chrome`) y haz **Read** de cada PNG.

Rutas a QA (de `app/pages/`): **`/`** (index: mapa + SidePanel), **`/comparar`** y **`/simulador`** (SPA, `ssr:false` en routeRules — confirma que hidratan sin pantalla en blanco). Por cada captura verifica explícitamente:
- [ ] **Mapa renderiza:** tiles base + capa `atlas.pmtiles` visible y coloreada (no gris/vacío → fallo de headers PMTiles o CORS).
- [ ] **Leyenda y rampa:** rojo→verde coherente con el selector de dimensión activo.
- [ ] **Selector de dimensiones:** las 5 (`atlas_score`, `score_accesibilidad`, `score_ambiental`, `score_socioeconomico`, `score_seguridad`) cambian el coloreo.
- [ ] **SidePanel:** al click en un radio muestra sub-scores (salud, educación, transporte, ndvi, frescura) sin `null`/`undefined`/`NaN` en pantalla.
- [ ] **Cámara vuela** al elegir un municipio (valida la coincidencia `nombre`==`municipio`).
- [ ] **Sin overflow ni cortes** en paneles/labels; sin overlays fantasma.
- [ ] **API en prod responde:** `GET https://caba.tensor.lat/api/caba` y `/api/caba/municipios` devuelven JSON, no 404/500.

Documenta cada página como "limpia" o con el hallazgo concreto. "Limpia" se dice explícito; no se infiere del silencio.

### 6. Iterar
Si QA de datos falla → arregla en el pipeline/`stores`, re-emite, re-sincroniza assets, rebuild. Si QA visual falla → arregla componente/headers, redeploy. Re-QA. No publiques con un FAIL abierto.

## Contrato de salida
- URL de producción viva en `https://<slug>.tensor.lat` con dominio asignado y SSL.
- Reporte QA datos: cada check PASS/FAIL con números reales (features, comunas, rango de scores).
- Reporte QA visual: una línea por ruta (`/`, `/comparar`, `/simulador`) con veredicto.
- Checklist pre-publicación firmado (abajo).

## Checklist pre-publicación
- [ ] `npm run build` verde y 3 server-assets presentes.
- [ ] QA de datos sin FAIL (scores [0,1], sin NaN/null, conteos cuadran, claves coinciden).
- [ ] QA visual de las 3 rutas: todas "limpias".
- [ ] PMTiles carga en prod con headers de `vercel.json` (CORS + range requests).
- [ ] API REST responde 200 en `/api/caba` y `/api/caba/municipios`.
- [ ] Dominio `*.tensor.lat` asignado, SSL activo, canonical/OG de `nuxt.config.ts` apuntan a esa URL.
- [ ] `og-image.png` (1200×630) existe en `public/`.

## Errores comunes
- **Usar `nuxt generate`** (estático): mata la API REST serverless. Usa `nuxt build` (preset vercel).
- **Olvidar sincronizar `server/assets/data/`** tras rebuild: la API sirve datos viejos o 404 (está en `.gitignore`, se regenera en `prebuild`).
- **Scores fuera de [0,1]:** la rampa MapLibre se ve toda de un color.
- **`MUNICIPIOS[].nombre` ≠ `municipio`:** cámara no vuela, `stats` no resuelve. Es la regla más quebrada al re-territorializar.
- **Headers PMTiles ausentes:** MapLibre falla silencioso (mapa gris) porque PMTiles usa HTTP range requests + CORS.
- **Cambiar de territorio sin actualizar las URLs hardcodeadas** `caba.tensor.lat` en `nuxt.config.ts`: OG/canonical apuntan al dominio equivocado.
