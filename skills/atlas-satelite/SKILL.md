---
name: atlas-satelite
description: Use when computing satellite indicators (NDVI, NDBI, LST, VIIRS, land cover) per base unit via Google Earth Engine for the atlas.
---

## Overview
Fase 3 del atlas: computa indicadores satelitales agregados por unidad base con Google Earth Engine (GEE) y exporta un CSV (una fila por `cod_unidad`) para hacer merge en el pipeline de composición. Plantilla real: `atlas-caba-web/scripts/gee/compute_satelitales_caba.py`.

## Cuándo usar
- Necesitas las capas ambientales/seguridad del atlas: NDVI, NDBI, LST (°C), VIIRS (luces nocturnas), % verde.
- Vas a calcular los sub-scores `score_ndvi`, `score_calor` (LST), `score_viirs`, `score_ndbi` definidos en la metodología de scoring.
- Tienes las unidades base (radios censales / manzanas) como GeoJSON y aún no existe `satelite_*.csv`.

## Procedimiento
1. **Auth GEE.** `import ee; ee.Initialize()`. Si falla con error de credenciales, correr `ee.Authenticate()` una vez y reintentar `ee.Initialize()`. (El script real solo llama `ee.Initialize()`; asume auth previa.)
2. **AOI.** Define el bounding box de la zona con `ee.Geometry.Rectangle([w, s, e, n])`. Acota toda colección con `.filterBounds(AOI)`.
3. **Ventanas temporales.** Define la ventana de verano (más reciente, según hemisferio) para Sentinel-2/Landsat y una ventana anual para VIIRS.
4. **Sentinel-2 → NDVI, NDBI.** Colección `COPERNICUS/S2_SR_HARMONIZED`, enmascarada con Cloud Score+ (`GOOGLE/CLOUD_SCORE_PLUS/V1/S2_HARMONIZED`, banda `cs_cdf >= 0.6`) vía `linkCollection`, `.divide(10000)`, `.median()`. Luego `normalizedDifference(["B8","B4"])` → `ndvi`; `normalizedDifference(["B11","B8"])` → `ndbi`.
5. **Landsat 8/9 → LST °C.** Colecciones `LANDSAT/LC08/C02/T1_L2` y `LANDSAT/LC09/C02/T1_L2`, `.merge()`. Máscara nubes/sombra con bits 3 y 4 de `QA_PIXEL`. Conversión: `lst_c = ST_B10·0.00341802 + 149.0 − 273.15`. `.median().select("lst_c")`.
6. **VIIRS → luces nocturnas.** Colección `NASA/VIIRS/002/VNP46A2`, banda `Gap_Filled_DNB_BRDF_Corrected_NTL`, `.median()` → `viirs`.
7. **% verde (ESA WorldCover).** [VERIFICAR] La plantilla CABA NO computa % verde con ESA WorldCover. Si se requiere `pct_verde`: usar `ESA/WorldCover/v200` (o `v100`), aislar la clase Tree cover (valor 10) con `.eq(10)`, multiplicar por 100 y agregar con `ee.Reducer.mean()` para obtener % de cobertura arbórea por unidad. [VERIFICAR] el dataset y valor de clase contra el catálogo GEE antes de correr.
8. **Stack + reduceRegions.** `ee.Image.cat([ndvi, ndbi, lst, viirs])`. Carga las unidades base desde el GeoJSON como `ee.FeatureCollection` (una `ee.Feature` por geometría con propiedad `cod`). Procesa **por chunks** (CHUNK ≈ 250 features) para evitar límites de tamaño de request; cada chunk: `stack.reduceRegions(collection=fc, reducer=ee.Reducer.mean(), scale=30, tileScale=4)` y `.getInfo()` con reintentos (3 intentos, `sleep` entre fallos).
9. **Exportar CSV.** Una fila por unidad con `cod_unidad, ndvi, ndbi, lst_c, viirs_rad, pct_verde`. (En la plantilla CABA la columna se llama `cod_manzana` y `viirs`; renombra a `cod_unidad`/`viirs_rad`/añade `pct_verde` según el contrato de salida de este atlas.)

## Contrato de salida
CSV con header `cod_unidad,ndvi,ndbi,lst_c,viirs_rad,pct_verde`, una fila por unidad base. Valores faltantes quedan vacíos (None). Este CSV se mergea por `cod_unidad` en la fase de composición; ver la metodología en `${CLAUDE_PLUGIN_ROOT}/references/scoring-methodology.md` (sub-scores `score_ndvi`, `score_calor`/LST, `score_viirs`, `score_ndbi`).

## Errores comunes
- **Auth no resuelta:** `ee.Initialize()` falla si nunca se corrió `ee.Authenticate()`; resolver auth antes de iterar chunks.
- **Request demasiado grande:** no llamar `reduceRegions` sobre todas las unidades de golpe; usar chunks + reintentos como la plantilla.
- **`scale` incorrecta:** la plantilla usa `scale=30` (resolución Landsat/NDBI); no bajar a 10 sin necesidad (encarece y puede romper el request).
- **Bandas Sentinel-2:** NDVI usa B8/B4; NDBI usa B11/B8. No invertir el orden de `normalizedDifference`.
- **LST sin conversión:** `ST_B10` crudo es escala de sensor; aplicar `·0.00341802 + 149.0 − 273.15` para °C.
- **VIIRS sin normalizar luego:** el CSV exporta radiancia cruda (`viirs_rad`); la normalización log (`log(viirs+1)`) ocurre en la fase de scoring, no aquí.
- **% verde [VERIFICAR]:** ESA WorldCover no está en la plantilla real; confirma dataset y código de clase en el catálogo GEE antes de afirmar resultados.
