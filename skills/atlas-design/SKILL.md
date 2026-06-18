---
name: atlas-design
description: Use when sources are scouted and you need to design the atlas index (base unit, dimensions, weights) before building the data pipeline. Úsala cuando ya tengas los dossiers de fuentes/datasets de un territorio y debas decidir unidad base, 2 niveles de agregación, las 4 dimensiones y sus pesos, y clasificar cada dimensión como dato-real/proxy/brecha antes de tocar el pipeline de datos. NO la uses para scoutear fuentes (eso es atlas-scout) ni para construir GeoJSON/PMTiles.
---

## Overview

Fase 1 del Atlas de Tensor: traducir los dossiers de fuentes en un diseño de índice replicable. Eliges la **unidad base** + 2 niveles de agregación, fijas las **4 dimensiones y sus pesos**, marcas qué es construible con dato real vs proxy vs brecha, y produces `PLAN_CONSTRUCCION.md` + el bloque de unidad/dimensiones en `atlas.manifest.json`. No descargas ni transformas datos todavía.

LEE primero (rutas, no `@`):
- `${CLAUDE_PLUGIN_ROOT}/references/scoring-methodology.md` — pesos, sub-scores, normalización, renormalización por dimensiones faltantes, LISA.
- `${CLAUDE_PLUGIN_ROOT}/references/data-contract.md` — nombres EXACTOS de campos y archivos que el front Nuxt + API Nitro consumen.
- Ejemplo de salida: estructura secciones 1–4 de `/Users/cristianespinal/atlas-caba-web/docs/PLAN_CONSTRUCCION.md` (resumen ejecutivo, diseño del índice, catálogo de capas, secuencia priorizada).

REQUIRED previo: usa la skill `tensor-atlas:atlas-scout` para tener los dossiers de fuentes verificadas. Esta skill consume esos dossiers, no los genera.

## Cuándo usar

- Ya hay dossiers de datasets verificados (URL responde 200, formato, granularidad) y aún NO existe `PLAN_CONSTRUCCION.md`.
- Hay que decidir cuál es la unidad base (manzana/radio censal/sección) y qué da el socioeconómico sin join.
- Hay que fijar pesos de las 4 dimensiones y justificarlos contra la metodología canónica.
- Hay que clasificar cada dimensión como construible-real / proxy / brecha antes de comprometer trabajo de pipeline.

## Procedimiento

### 1. Elegir unidad base (tabla de decisión)

La unidad base = `manzana` en el vocabulario data-agnóstico del contrato (`data-contract.md` §intro): nivel 1 = `municipio`, nivel 2 = `veredas.geojson`. Los nombres de campo `cod_manzana`, `municipio`, `veredas` se mantienen LITERALES aunque el territorio no tenga manzanas ni veredas.

Construye una tabla `Opción | A favor | En contra | Veredicto` (como §2 del ejemplo CABA). Criterio decisor: **la unidad base debe ser el grano que entrega `score_socioeconomico` real sin join espacial** (población/hogares/NBI embebidos), porque es la dimensión que ancla el índice en Fase 1. En CABA ganó el radio censal (NBI embebido); la manzana catastral quedó como capa visual de zoom alto. Niveles 1/2 son agregaciones para filtros, rankings y paneles (comuna=15, barrio=48).

### 2. Asignar las 4 dimensiones y sus pesos

Pesos de PRODUCCIÓN (cita `scoring-methodology.md` §1, anclados en `recalc_v3.py:58` y `build_atlas_caba.py:254-255`):

| Dimensión | Campo del contrato | Peso |
|---|---|---|
| Accesibilidad | `score_accesibilidad` | **0.40** |
| Ambiental | `score_ambiental` | **0.25** |
| Socioeconómico | `score_socioeconomico` | **0.25** |
| Seguridad | `score_seguridad` | **0.20** |

La suma es **1.10 a propósito** y se normaliza dividiendo por `W_SUM`: `atlas_score = (0.40·acc + 0.25·amb + 0.25·soc + 0.20·seg)/1.10`. NO la cambies a 1.0 para que la lógica de `recalc_v3.py` se copie tal cual.

ADVERTENCIA de divergencia (scoring-methodology §1): el default de `aggregator.py` (`DIMENSION_WEIGHTS`) es `0.35/0.20/0.30/0.15` (suma 1.0). Si replicas con `aggregator.py` SIN pasar `dimension_weights`, obtienes esos pesos, NO los de producción. Documenta cuáles usas.

### 3. Tabla de dimensiones: construible-real / proxy / brecha

Para cada dimensión, marca el estado contra los dossiers (como §2 del ejemplo CABA):

| Dimensión | Estado | Sub-scores construibles | Dataset que lo da | Insumo de score |
|---|---|---|---|---|
| Accesibilidad | real/proxy/brecha | `score_salud`, `score_educacion`, `score_via` | salud + educación + transporte | distancia/isócrona con CAP → `acc_from_dist`/`acc_from_minutes` |
| Ambiental | real/proxy/brecha | `score_ndvi`, `score_calor`, `score_impermeabilizacion` | Sentinel-2 (NDVI/NDBI), Landsat (LST) GEE | `norm`/`norm_robust`; calor = `1 − norm(LST)` |
| Socioeconómico | real/proxy/brecha | `score_socioeconomico` (= `1 − nbi_rate`) | censo a unidad base | `clamp01(1 − NBI/hogares)` |
| Seguridad | real/proxy/brecha | `score_seguridad` | delito georreferenciado (real) o VIIRS (proxy) | KDE delitos `invert=True`, o `norm(log(viirs+1))` proxy |

Reglas de clasificación (scoring-methodology §2, §4):
- **real**: hay dataset verificado a la granularidad de la unidad base.
- **proxy**: no hay dato directo pero existe sustituto defendible (p.ej. VIIRS como proxy de seguridad/iluminación; conteo de paradas como proxy de demanda de transporte). Etiqueta el sub-score como proxy EXPLÍCITO.
- **brecha**: no hay dato ni proxy aceptable a esa granularidad. Una dimensión en brecha NO bloquea: la renormalización por pesos activos (§4) suma solo los pesos de las dimensiones con valor y divide por `wsum` (p.ej. Fase 1 con solo socioeconómico → `atlas_score = score_socioeconomico`; Fase 2 acc+soc → `/0.65`).

### 4. Secuencia priorizada Fase1/Fase2/Fase3

Ordena por quick-wins (como §4 del ejemplo CABA):
- **Fase 1 (demo en horas):** geometría base + `score_socioeconomico` de un solo archivo (censo a la unidad base), agregaciones (nivel 1/2), capa de vulnerabilidad. Resultado: mapa coroplético con `atlas_score` real y panel lateral. Pipeline: descargar → (parsear geometría si viene en WKT) → `1 − norm(NBI)` → quintiles → tippecanoe a `atlas.pmtiles`.
- **Fase 2 (enriquecer):** salud + educación (reproyectar a EPSG:4326 si vienen en Gauss-Krüger/otro CRS) → `score_salud`/`score_educacion`; transporte + isócronas OSRM → `score_accesibilidad`; satélite GEE (`reduceRegions` por unidad: NDVI/NDBI/LST/VIIRS/WorldCover) → `score_ambiental`.
- **Fase 3 (avanzado/institucional):** ordenamiento/conflicto de uso, parcelas en PMTiles (minzoom alto), censo más reciente a la unidad base, APIs tiempo real, y **LISA/Moran's I real** para poblar `zona_atlas` ∈ {HH,LL,HL,LH,NS} + `moran_i` (PySAL/esda, EPSG local, Queen, `significance=0.05`; ver scoring-methodology §5).

## Contrato de salida

1. **`PLAN_CONSTRUCCION.md`** con, como mínimo, secciones 1–4 del ejemplo CABA: (1) resumen ejecutivo + caveats honestos, (2) diseño del índice = tabla de decisión de unidad base + tabla dimensiones/pesos/estado, (3) catálogo de capas mapeado a grupos del LayerPanel, (4) secuencia priorizada Fase1/2/3. Toda afirmación cuantitativa anclada a un dossier verificado; lo no verificado va como `[VERIFICAR]`.
2. **Bloque unidad/dimensiones en `atlas.manifest.json`**: la unidad base elegida, los nombres-contrato (`cod_manzana`, `municipio`, niveles de agregación reales), y las 4 dimensiones con sus pesos (`score_accesibilidad` 0.40, `score_ambiental` 0.25, `score_socioeconomico` 0.25, `score_seguridad` 0.20) más el estado real/proxy/brecha de cada una. Las `key` de dimensión DEBEN ser nombres de campo que existirán en las properties de `atlas.geojson` (data-contract §2, §3).

## Errores comunes

- Usar pesos `0.35/0.20/0.30/0.15` (default de `aggregator.py`) creyendo que son los de producción. Producción = `0.40/0.25/0.25/0.20` (suma 1.10).
- Normalizar la suma de pesos a 1.0 y romper la copia directa de `recalc_v3.py`. Déjala en 1.10; el código divide por `W_SUM`.
- Elegir la unidad base por estética (manzana catastral más fina) en vez de por el grano que da el socioeconómico sin join.
- Renombrar los campos del contrato (`cod_manzana`, `municipio`, `score_*`) a etiquetas del territorio: rompe el front Nuxt y la API Nitro. El rename a etiquetas humanas (`score_via→transporte`, `score_calor→frescura`) ocurre SOLO en la capa REST.
- Tratar una dimensión en brecha como bloqueante: la renormalización por pesos activos permite arrancar con menos dimensiones.
- Marcar un proxy como dato real. Etiqueta VIIRS-como-seguridad y conteo-de-paradas-como-demanda explícitamente como proxy.
- Olvidar que `zona_atlas` arranca en `"NS"` y `moran_i` no se emite hasta correr LISA (Fase 3): no prometas clústeres reales en Fase 1.
