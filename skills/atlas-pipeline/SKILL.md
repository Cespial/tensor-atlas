---
name: atlas-pipeline
description: Use when the index is designed and the agent needs to build the Python data pipeline that turns verified raw data into the Atlas data contract (atlas.geojson, municipios.geojson, veredas.geojson, atlas_stats*.json, gap_analysis.json) plus atlas.pmtiles via tippecanoe. Triggers on requests to "build the pipeline", "emit the data contract", "generate atlas.geojson / PMTiles", reproject CRS, run tippecanoe, compute scores/quintiles/LISA, or wire raw GeoJSON/CSV into the Nuxt + MapLibre front.
---

## Overview
Pipeline data-agnóstico que, desde datos crudos verificados, emite EXACTAMENTE el contrato que consume el front Nuxt + API Nitro. Reusa la plantilla real `build_atlas_caba.py` (build por fases) y `recalc_v3.py` (recálculo Urabá). El front NO se toca si respetas nombres de campo y rutas.

## Cuándo usar
- El índice ya está diseñado (dimensiones, pesos, sub-scores definidos) y toca producir los archivos de `public/data/`.
- Necesitas reproyectar CRS, parsear WKT embebido, hacer joins espaciales por distancia, normalizar (min-max/quintiles) o calcular LISA.
- Vas a generar `atlas.pmtiles` con tippecanoe.
- Antes de empezar: lee ${CLAUDE_PLUGIN_ROOT}/references/data-contract.md (campos exactos) y ${CLAUDE_PLUGIN_ROOT}/references/scoring-methodology.md (pesos y fórmulas). Plantillas reales: `/Users/cristianespinal/atlas-caba-web/scripts/build_atlas_caba.py` y `/Users/cristianespinal/atlas-caba-web/recalc_v3.py`.

## Procedimiento

| # | Paso | Mecánica verificada en la plantilla |
|---|---|---|
| 1 | **Download** | Datos crudos a `data/raw/` (no a `public/data/`). El script lee de `data/raw/` y `data/raw/fase2/`. Salidas a `public/data/` (`OUT`, `build_atlas_caba.py:31`). |
| 2 | **Reproyección CRS** | Varios GeoJSON GCBA vienen en **EPSG:9498 / Gauss-Krüger** (hospitales, establecimientos educativos) → reproyectar a **EPSG:4326** con `ogr2ogr -t_srs EPSG:4326` (GDAL) o `pyproj`. Todo el pipeline trabaja en 4326. `[VERIFICAR]` el EPSG origen de cada dataset nuevo. |
| 3 | **Parseo (WKT embebido)** | La geometría de radios 2010 llega como **string WKT en la propiedad `WKT`**, no como geometría GeoJSON nativa → parsear (p.ej. `shapely.wkt.loads`) antes de calcular centroides. `[VERIFICAR]` si tu fuente trae WKT o geometría nativa. |
| 4 | **Joins espaciales (centroide/distancia)** | Centroide por unidad (`poly_centroid`, `build_atlas_caba.py:60`). Proyección equirectangular local a metros (`to_xy`, `:56`) o reproyección a CRS métrico. Vecino más cercano por categoría con `scipy.spatial.cKDTree` (`:149-160`): salud/educación/transporte. CAPs en metros: salud 1500, educación 800, masivo 1000, bus 400, bici 500 (`:183-187`). Urabá usa minutos OSRM con CAPs en min (IPS 60, colegio 30, cabecera 45, puerto 120; `recalc_v3.py:34-37`). |
| 5 | **Normalización** | `clamp01` (`:52`); `norm` min-max (`recalc_v3.py:44`); `norm_robust` percentil **p2–p98** anti-outliers (`build_atlas_caba.py:212-220`). VIIRS se normaliza en **log**: `norm(log(viirs+1))` (`:225-226,240`). `acc_from_dist = clamp01(1 - dist/cap)` (`:142`); `acc_from_minutes` análogo (`recalc_v3.py:50`). |
| 6 | **LISA / zona** | El template CABA NO calcula LISA: deja `zona_atlas="NS"` (`:134`). Para zonas reales: reproyectar a **EPSG:3116** (MAGNA-SIRGAS), pesos **Queen** row-standardized, `Moran` global → `moran_i`, `Moran_Local` → cuadrante `lisa.q` mapeado a HH(1)/LH(2)/LL(3)/HL(4)/NS (`p_sim≥0.05`) con `libpysal`+`esda` (ver `spatial_clusters.py`, scoring-methodology §5). `[VERIFICAR]` si corres esta capa. |
| 7 | **Scores con pesos** | Pesos producción: `W={"score_accesibilidad":0.40,"score_ambiental":0.25,"score_socioeconomico":0.25,"score_seguridad":0.20}` (suma 1.10) (`:254-255`). **Renormaliza por dims disponibles**: solo suma pesos de dims con valor, divide por `wsum` (`:265-273`). `atlas_score=round(num/wsum,4)`. Quintiles Q1–Q5 sobre cuantiles 0.2/0.4/0.6/0.8 (`:276-286`). |
| 8 | **Emitir el contrato** | Ver "Contrato de salida". Campos EXACTOS de cada feature en data-contract §2. |
| 9 | **tippecanoe → PMTiles** | Comando EXACTO (extraído de la metadata del `atlas.pmtiles` real de CABA): ver abajo. |
| 10 | **Copia a server-assets** | `node scripts/sync-api-assets.mjs` copia **3** archivos (`atlas_stats_v3.json`, `gap_analysis.json`, `atlas.geojson`) de `public/data/` → `server/assets/data/`. Corre como `prebuild`/`predev` en package.json. La API Nitro solo funciona con esas copias. |

### Comando tippecanoe (verificado)
```bash
tippecanoe -o public/data/atlas.pmtiles -l manzanas -z14 -Z9 \
  --drop-densest-as-needed --no-tile-size-limit -f public/data/atlas.geojson
```
Capa = `manzanas`, zoom 9–14, `-f` sobrescribe. (tippecanoe v2.79.0). No lo emite el script Python; córrelo tras escribir `atlas.geojson`.

## Contrato de salida (a `public/data/`)
- `atlas.geojson` — 1 feature/manzana; props mínimas: `cod_manzana`, `municipio`, `atlas_score`, los `score_*` expuestos, `zona_atlas`, `quintil` (data-contract §2).
- `municipios.geojson` — `municipio` + `atlas_score` pop-weighted; `veredas.geojson` — `nombre`+`comuna`.
- `atlas_stats.json` (front: `count/avg/min/max/p25/p75` por dim, clave `"Todos"`) · `atlas_breaks.json` (7 anclas) · `atlas_stats_v3.json` (API: `_meta`+`ranking_municipios_v3`+`municipios`) · `comunas_centroides.json` (1er elem `"Todos"`) · `gap_analysis.json` (placeholder OK).
- `atlas.pmtiles` (tippecanoe).

## Verificación de salida (obligatoria)
- Todos los `score_*` y `atlas_score` ∈ **[0,1]**; **cero NaN** (usar `None`, nunca `NaN` en JSON).
- Conteos cuadran: nº features `atlas.geojson` == radios cargados; `count` por municipio suma a "Todos".
- **REGLA DURA**: `MUNICIPIOS[].nombre` en `stores/atlas.js` == `municipio` en atlas/municipios.geojson (idéntico, ej. "Comuna 7"). Si no, la cámara no vuela y `stats[municipio]` no resuelve.
- Claves de `avg`/`dimensiones` == nombres de campo `score_*` exactos (la API hace lookup directo).
- `quintil` ∈ {Q1..Q5}; `zona_atlas` ∈ {HH,LL,HL,LH,NS}.

## Errores comunes
- Olvidar reproyectar 9498→4326 → puntos caen lejos, distancias absurdas.
- Dejar geometría como WKT string sin parsear → sin centroides.
- Dividir el índice por 1.10 fijo cuando faltan dims (rompe [0,1]); usa `wsum` dinámico (CABA), salvo Urabá v3 donde todas las dims existen y sí divide por `W_SUM=1.10`.
- Emitir `NaN`/`Infinity` en JSON → el front rompe al parsear; emite `null`.
- No correr `sync-api-assets.mjs` tras el rebuild → la API sirve datos viejos.
- `MUNICIPIOS[].nombre` no idéntico a `municipio` → `municipioConfig` devuelve `undefined`.
- No confundir esta skill con el diseño del índice: REQUIRED haber pasado por la fase de diseño/scout antes.
