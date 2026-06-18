---
name: receta-generica-latam
description: Úsala cuando atlas-scout o atlas-design necesiten investigar y verificar fuentes de datos abiertos para un territorio de LatAm que NO tiene receta-país dedicada (no existe references/recipes/<pais>.md), o cuando una receta-país no cubre algún dominio. Es la receta de respaldo agnóstica de país.
---

## Overview

Cuando no hay receta-país, el método es el mismo que produjo Atlas Urabá y Atlas CABA: descubrir datasets por dominio, **verificarlos en runtime** (HTTP 200 + features reales + CRS + licencia) antes de confiar en ellos, y mapear cada dominio a una dimensión/sub-score del contrato data-agnóstico del Atlas. Nada se da por vivo sin un `curl` que lo pruebe. Lo no verificable es brecha, no dato.

## Cuándo usar

- No existe `${CLAUDE_PLUGIN_ROOT}/references/recipes/<pais>.md` para el territorio.
- La receta-país existe pero un dominio (p.ej. riesgo o seguridad) no está cubierto.
- Necesitas el método universal de verificación de fuentes o las fuentes independientes de país (GEE, OSM/Geofabrik, límites administrativos).

## Checklist de dominios a cubrir

Lanza un descubrimiento por cada dominio (en atlas-scout, uno por subagente):

1. **Territorio base** — geometría de la unidad más fina CON población/NBI embebidos; y 2 niveles de agregación.
2. **Salud** — efectores (hospitales, atención primaria) + áreas de cobertura/catchment.
3. **Educación** — establecimientos + educación superior.
4. **Transporte** — paradas/estaciones, recorridos, red ruteable (OSM) para isócronas.
5. **Ambiente** — verde público (m²), arbolado, + verdad-terreno para validar satélite.
6. **Satélite** — GEE (NDVI/NDBI/LST/NTL/cobertura); ver §Fuentes universales.
7. **Riesgo** — hábitat/asentamientos informales, isla de calor, amenazas.
8. **Socioeconómico** — NBI/pobreza/hogares/población a la unidad base.

## Cómo elegir la unidad base y las agregaciones

Regla: **la unidad base es la más fina que traiga población/NBI embebidos en sus propiedades, SIN join.** Es el equivalente al `atlas.geojson` del contrato (ver `${CLAUDE_PLUGIN_ROOT}/references/data-contract.md`). En Urabá fue la manzana censal (CNPV); en CABA, el radio censal 2010 (geometría + población + hogares + NBI en un solo archivo).

- Si la capa más fina (manzana/parcela catastral) NO trae atributos censales y exige join espacial pesado → resérvala como capa visual de zoom alto, no como unidad del índice.
- **Agregación nivel 1** (`municipios.geojson`, prop `municipio`): unidad administrativa intermedia para KPIs, rankings y filtros (municipio en Urabá, comuna en CABA).
- **Agregación nivel 2** (`veredas.geojson`): contexto más fino que agg1 (vereda en Urabá, barrio en CABA).
- El perímetro del territorio se deriva por `ST_Union`/dissolve de agg1; no busques un dataset de límite externo si ya tienes las unidades.

## Cómo verificar una fuente en runtime (antes de confiar)

NO confíes en un catálogo ni en una URL "documentada". Verifica cada una:

1. **HTTP vivo:** `curl -sI <url>` → debe dar `200` (o `302→200`; un `401` significa que existe pero requiere key; `60` = SSL roto = brecha).
2. **Features reales:** descarga un trozo y confirma que `features` NO está vacío (`features:[]` = descartar, como pasó con "espacio verde privado" en CABA).
3. **CRS:** detecta la proyección. Muchos GeoJSON gubernamentales vienen en Gauss-Krüger / locales (EPSG:9498, EPSG:5347, etc.) y requieren reproyección: `ogr2ogr -s_srs EPSG:<src> -t_srs EPSG:4326 out.geojson in.geojson`. El contrato exige EPSG:4326.
4. **Formato real:** verifica que el `.geojson` sea GeoJSON nativo y no WKT embebido en una propiedad (CABA traía la geometría como string `WKT` que hubo que parsear). Verifica que un `.zip` no sea en realidad RAR. Verifica separador/decimal en CSV.
5. **Licencia:** confirma licencia abierta reutilizable (CC-BY u similar) y anótala.

Marca cada fuente verificada como `verificado=true` con su `url`, `formato`, `granularidad`, `crs`, `licencia` y `mapea_a`. Lo que cuelgue, requiera login por navegador o derecho de petición → `gaps[]`, con proxy aceptable si existe.

## Fuentes universales (independientes de país)

Siempre disponibles, NO requieren receta-país:

- **GEE** (cuenta del usuario autenticada; ver skill `tensor-atlas:atlas-satelite`):
  - `COPERNICUS/S2_SR_HARMONIZED` → NDVI, NDBI (10 m).
  - `LANDSAT/LC09/C02/T1_L2` + `LANDSAT/LC08/C02/T1_L2` → LST / isla de calor (banda térmica).
  - `NASA/VIIRS/002/VNP46A2` → luces nocturnas (NTL, ~500 m): proxy de actividad/iluminación.
  - `ESA/WorldCover/v200` → % cobertura/verde (10 m). `GOOGLE/DYNAMICWORLD/V1` para tendencia.
  - Patrón: subir la unidad base a `ee.FeatureCollection` → `reduceRegions` → exportar CSV por unidad → merge en el pipeline.
- **OSM / Geofabrik** para ruteo OSRM (isócronas de accesibilidad): descargar `<region>-latest.osm.pbf` de `download.geofabrik.de`, recortar a la bbox del territorio, levantar OSRM (perfiles foot/bike/car).
- **Límites administrativos** (cross-check, no fuente primaria): WFS del instituto geográfico nacional o GADM. La geometría base autoritativa sale del censo, no de aquí.

## Cómo mapear cada dominio a una dimensión / sub-score

Mapea contra el modelo estable (`${CLAUDE_PLUGIN_ROOT}/references/scoring-methodology.md`). Pesos: `accesibilidad 0.40 · ambiental 0.25 · socioeconómico 0.25 · seguridad 0.20`.

| Dominio | Dimensión | Sub-score |
|---|---|---|
| Salud (efectores + áreas) | `score_accesibilidad` | `score_salud` |
| Educación (estab. + universidades) | `score_accesibilidad` | `score_educacion` |
| Transporte (paradas + red OSRM) | `score_accesibilidad` | `score_via` |
| NBI / hogares / población | `score_socioeconomico` | `score_nbi` |
| VIIRS (NTL, proxy actividad) | `score_socioeconomico` | `score_viirs` |
| NDVI / verde / WorldCover | `score_ambiental` | `score_ndvi`, `score_ndbi` |
| LST (Landsat) | `score_ambiental` | `score_calor`/`lst` |
| Iluminación nocturna (VIIRS) | `score_seguridad` | (proxy explícito) |
| Hábitat / asentamientos informales | `score_socioeconomico` / riesgo | overlay narrativo |

Sub-scores construibles según el dato real disponible: `score_salud`, `score_educacion`, `score_via`, `score_ndvi`, `score_calor`/`lst`, `score_ndbi`, `score_viirs`, `score_nbi`. Si una dimensión no tiene dato real, normalízala sobre las dimensiones disponibles (no la inventes). `score_seguridad` casi siempre arranca como proxy VIIRS + equipamiento; el delito georreferenciado a unidad fina suele ser brecha de gestión.

## Contrato de salida

Por dominio, un dossier con: fuentes `verificado=true` (url/formato/granularidad/crs/licencia/`mapea_a`), unidad base propuesta + agg1/agg2, y `gaps[]` con proxy. Eso alimenta `atlas-design` para fijar `unidad_base`, `dimensiones` y pesos en el manifiesto, y `atlas-pipeline` para emitir el contrato de datos (ver `${CLAUDE_PLUGIN_ROOT}/references/data-contract.md`).

## Errores comunes

- Confiar en una URL de catálogo sin `curl`: muchas dan `404`, `features:[]` o SSL roto.
- Elegir como unidad base la capa más fina aunque NO traiga censo: obliga a joins frágiles; pierde el `score_socioeconomico` sin join.
- Olvidar reproyectar: capas en CRS local desplazan toda la geometría; el contrato exige EPSG:4326.
- Asumir GeoJSON nativo cuando la geometría viene como WKT en una propiedad, o `.zip` que es RAR.
- Inventar un `score_seguridad` con dato directo cuando casi siempre es proxy: márcalo como proxy explícito.
- Sumar pesos sin normalizar cuando falta una dimensión: normaliza sobre las disponibles.
