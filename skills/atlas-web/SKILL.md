---
name: atlas-web
description: Use when the data contract is ready and you need to clone and rebrand the Nuxt web app (atlas-uraba-web base) for a new atlas territory — forking the codebase, editing the Pinia store, renaming server/api + server/utils, introducing the VOCAB vocabulary-alias layer, and wiring the public/data files. Use after tensor-atlas:atlas-scout has produced the data contract and the public/data + server/assets/data files exist. Do not use to design the data pipeline or compute scores (that is the Python build stage).
---

## Overview

Fase 4 del atlas: clonar el front Nuxt 3/4 + Pinia + MapLibre/PMTiles (Nitro `preset: vercel`) desde `atlas-uraba-web` a un territorio nuevo. La lógica de scores y los campos del contrato (`cod_manzana`, `municipio`, `atlas_score`, `zona_atlas` LISA HH/LL/HL/LH/NS, `quintil`) permanecen **1:1 e invariantes**; solo cambian datos, metadatos y **etiquetas visibles**. Esas etiquetas se centralizan en un objeto `VOCAB` — la introducción de esa capa es un **refactor único** en el repo base.

Lee primero: `${CLAUDE_PLUGIN_ROOT}/references/codebase-anatomy.md` (estructura, archivos a editar, diseño de `VOCAB`) y `${CLAUDE_PLUGIN_ROOT}/references/data-contract.md` (qué archivos consume el front/API y con qué campos).

## Cuándo usar

- Ya existe el data contract y los archivos de `public/data/` (`atlas.geojson`, `municipios.geojson`, `atlas_stats_v3.json`, `gap_analysis.json`, centroides, `atlas.pmtiles`...) están emitidos.
- Hay que rebrandear el front al territorio nuevo (title, descripción, OG, dominio `*.tensor.lat`, locale).
- Toca renombrar la unidad territorial visible (p.ej. "manzana/municipio/vereda" → "radio censal/barrio/comuna") sin tocar rutas ni campos.

## Procedimiento

1. **Fork + limpieza** (comandos verificados en anatomy §2):
   ```bash
   cp -R /Users/cristianespinal/atlas-uraba-web /Users/cristianespinal/atlas-<t>-web
   cd /Users/cristianespinal/atlas-<t>-web
   rm -rf node_modules .nuxt .vercel .git public/data/* server/assets/data/*
   git init && git add -A && git commit -m "fork: atlas-uraba-web base for Atlas <T>"
   gh repo create Cespial/atlas-<t>-web --private --source=. --remote=origin
   ```
2. **`git mv` de los nombres territoriales internos:**
   ```bash
   git mv server/api/uraba server/api/<t>
   git mv server/utils/uraba.js server/utils/<t>.js
   git mv app/components/FichaMunicipal.vue app/components/Ficha<Unidad>.vue
   ```
   Actualiza los `import` de `../../utils/uraba` → `../../utils/<t>` (y `../../../...` en handlers anidados) y el import de `FichaMunicipal` en `app/pages/index.vue`.
3. **Editar `app/stores/atlas.js`** (punto de edición central):
   - Reemplazar la lista `MUNICIPIOS` por las unidades agg1 del territorio con sus centroides (`{ nombre, lat, lng, zoom }`), primer elemento `{ nombre: 'Todos', ... }`. `lat/lng/zoom` replican `comunas_centroides.json`. **REGLA DURA:** `MUNICIPIOS[].nombre` debe coincidir EXACTAMENTE con la prop `municipio` de `atlas.geojson` y `municipios.geojson` (si no, `municipioConfig` da `undefined` y la cámara no vuela).
   - Ajustar el centro del mapa del territorio.
   - Mantener `DIMENSIONES`, `DIMENSIONES_V2` y `COLOR_EXPR_TEMPLATE` SIN cambios (lógica de scores idéntica; `key` de cada dimensión debe existir en las properties).
   - **AÑADIR el objeto `VOCAB`** (ver §3 de anatomy): claves internas `unidad_base`/`agg1`/`agg2` (invariantes: `manzana`/`municipio`/`vereda`, `cod: 'cod_manzana'`), valores = etiquetas visibles del territorio. Exportarlo (`export const VOCAB`).
4. **Editar `server/utils/<t>.js`:** reescribir `FUENTE` con las fuentes del territorio; ajustar los strings de `readData(...)` solo si cambian nombres de archivo. No tocar `normalize`/`setApiHeaders`/`notFound`.
5. **Editar `server/api/<t>/index.get.js`:** `nombre`, `descripcion`, `cobertura`, `formula`, y la lista `endpoints[]` (rutas `/api/uraba/...` → `/api/<t>/...`). Renombrar rutas dinámicas si aplica (`municipio/[nombre]`, `manzana/[cod]`); mantener los campos de GeoJSON que leen los handlers (`atlas_score_v3`, `quintil_v3`, `zona_atlas`, `moran_i`) idénticos.
6. **REFACTOR VOCAB (único):** reemplazar literales de UI ("Municipio"/"Manzana"/plurales) por lecturas de `VOCAB` en los ~20 componentes/pages listados en anatomy §1/§3 (`Ficha*.vue`, `ManzanaPanel`, `SidePanel`, `MobileSheet`, `AtlasKPIBar`, `ScoreRankingList`, `HistogramPanel`, `AppHeader/Footer`, `index.vue`, `comparar.vue`, `simulador.vue`, `LayerPanel/Toggle`, `MapLegend`, `GeocoderSearch`, `Download/ShareButton`, `comparar/*`). Acceso recomendado: `import { VOCAB } from '~/stores/atlas'` (estático por build; no requiere reactividad). [VERIFICAR] convención del repo base.
7. **Editar `nuxt.config.ts` → `app.head`:** `title`, `description` (conteos reales), todas las URLs OG/canonical `https://uraba.tensor.lat` → dominio nuevo `*.tensor.lat` (`og:url`, `og:image`, `twitter:image`, `link rel=canonical`), `og:title`/`twitter:title`/`og:site_name`, `og:locale` (`es_CO` → locale nuevo). Mantener `nitro.preset:'vercel'`, `serverAssets` y `routeRules` (SPA `/comparar` y `/simulador`).
8. **Conectar datos del contrato:** poblar `public/data/` con los archivos emitidos; copiar los **tres** que lee la API (`atlas.geojson`, `atlas_stats_v3.json`, `gap_analysis.json`) a `server/assets/data/`. Alinear el array `FILES` de `scripts/sync-api-assets.mjs` solo si cambian nombres.
9. **Arrancar:** `npm install` → `npm run dev`.

## Contrato de salida

- Repo `atlas-<t>-web` que levanta con `npm run dev` mostrando el territorio nuevo.
- `app/stores/atlas.js` con `MUNICIPIOS`/centro del territorio + bloque `VOCAB` exportado; `DIMENSIONES`/`COLOR_EXPR_TEMPLATE` intactos.
- `server/api/<t>/` y `server/utils/<t>.js` renombrados, imports y `FUENTE`/`index.get.js` ajustados.
- UI sin literales territoriales hardcodeados (todos vía `VOCAB`).
- `nuxt.config.ts` con metadatos/dominio/locale del territorio.
- Los tres assets de API presentes en `server/assets/data/`.

## Validación de regresión

- El refactor `VOCAB` es **único**: tras introducirlo, clonar a un territorio futuro = editar `MUNICIPIOS`/centro + `VOCAB` + `FUENTE` + `nuxt.config.ts head`, sin recorrer 20 archivos de UI.
- Validar **regresión visual** de Urabá y CABA tras el refactor: con sus respectivos `VOCAB`, la UI debe mostrar exactamente las etiquetas previas ("manzana/municipio/vereda" en Urabá; "radio censal/barrio/comuna" en CABA). Cualquier divergencia = bug del refactor.

## Errores comunes

- **Renombrar campos del GeoJSON o rutas de API** al cambiar etiquetas: NO. `cod_manzana`/`municipio`/`atlas_score_v3` y `/manzana/[cod]` son contrato, no UI. `VOCAB.unidad_base.cod` sigue siendo `cod_manzana` aunque la etiqueta diga "radio censal".
- **`MUNICIPIOS[].nombre` ≠ prop `municipio`** del GeoJSON → cámara no vuela, `stats[municipio]` no resuelve.
- **Olvidar copiar los 3 assets** a `server/assets/data/` → la API REST falla (lee server-asset empaquetado, no filesystem).
- **Tocar `COLOR_EXPR_TEMPLATE` / pesos** (0.40/0.25/0.25/0.20): la lógica de scores no cambia entre territorios.
- **Dejar literales sueltos** tras el refactor → la regresión visual de Urabá/CABA los detecta.
