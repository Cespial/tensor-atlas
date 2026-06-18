# Metodología de score del Atlas (replicable)

Referencia técnica del índice compuesto de bienestar territorial a nivel de manzana / radio censal. Inspirada en la MBHT chilena (Matriz de Bienestar Humano Territorial, MINVU) y en la arquitectura Indicator/CompositeIndicator de CityScope (MIT Media Lab). Documenta dimensiones, pesos, sub-scores, normalización (min-max, winsorize, quintiles), renormalización por datos faltantes y zonas LISA.

Fuentes verificadas (única base de los hechos aquí descritos):
- `atlas-uraba-web/recalc_v3.py` — recálculo v3 del Atlas Urabá con insumos reales (GEE `satelite_v3`, isócronas OSRM).
- `atlas-caba-web/scripts/build_atlas_caba.py` — build por fases del Atlas CABA (Buenos Aires).
- `atlas-uraba/src/composite/{normalizer,aggregator,spatial_clusters,pca_index}.py` — motor de composición canónico.
- `atlas-uraba/src/indicators/{accesibilidad,ambiental,seguridad,socioeconomico}/*.py` — indicadores raw.

> Hay DOS implementaciones del peso de dimensiones. El **default de `aggregator.py`** (`DIMENSION_WEIGHTS`) es `acc 0.35 / amb 0.20 / soc 0.30 / seg 0.15` (suma 1.0). Las builds **de producción Urabá v3 y CABA** usan `acc 0.40 / amb 0.25 / soc 0.25 / seg 0.20` (suma 1.10, se renormaliza). Esta referencia documenta los pesos de producción y marca la divergencia donde aplica.

---

## 1. Las 4 dimensiones y sus pesos

Pesos de producción (Urabá v3 y CABA):

| Dimensión | Peso | Variable | Archivo / línea |
|---|---|---|---|
| Accesibilidad | **0.40** | `W_ACC` | `recalc_v3.py:58` · `build_atlas_caba.py:254` |
| Ambiental | **0.25** | `W_AMB` | `recalc_v3.py:58` · `build_atlas_caba.py:254` |
| Socioeconómico | **0.25** | `W_SOC` | `recalc_v3.py:58` · `build_atlas_caba.py:255` |
| Seguridad | **0.20** | `W_SEG` | `recalc_v3.py:58` · `build_atlas_caba.py:255` |
| **Suma** | **1.10** | `W_SUM` | `recalc_v3.py:59` — la suma es 1.10 a propósito y **se normaliza** dividiendo por `W_SUM`. |

Fórmula canónica (literal de `recalc_v3.py:144` y del `_meta.formula` en `recalc_v3.py:219`):

```
atlas_score_v3 = (0.40·acc + 0.25·amb + 0.25·soc + 0.20·seg) / 1.10
```

Definición exacta en código (`recalc_v3.py:58-59`):
```python
W_ACC, W_AMB, W_SOC, W_SEG = 0.40, 0.25, 0.25, 0.20
W_SUM = W_ACC + W_AMB + W_SOC + W_SEG  # 1.10 -> se normaliza
```

En CABA el mismo diccionario de pesos vive en `build_atlas_caba.py:254-255`:
```python
W = {"score_accesibilidad": 0.40, "score_ambiental": 0.25,
     "score_socioeconomico": 0.25, "score_seguridad": 0.20}
```

> Divergencia anclada: el default de `aggregator.py:38-43` (`DIMENSION_WEIGHTS`) es `acc 0.35 / amb 0.20 / soc 0.30 / seg 0.15` y suma 1.0; el aggregator igual reescala dividiendo por la suma de pesos activos (`aggregator.py:238-239`). Si se replica con `aggregator.py` SIN pasar `dimension_weights`, se obtienen los pesos 0.35/0.20/0.30/0.15, NO los de producción.

---

## 2. Sub-scores y de qué insumo sale cada uno

### Accesibilidad
Se compone de sub-scores por equipamiento; el agregado de la dimensión es su media o media ponderada según la implementación.

**Urabá v3 (isócronas OSRM reales, `recalc_v3.py:91-108`).** Sub-scores derivados de minutos de conducción por carretera (`isocronas_osrm_real.csv`):

| Sub-score | Insumo (columna CSV) | CAP (min) | Peso interno |
|---|---|---|---|
| acceso a salud/IPS | `min_ips` | `CAP_IPS = 60.0` | 0.30 |
| acceso a colegio | `min_colegio` | `CAP_COLEGIO = 30.0` | 0.25 |
| acceso a cabecera municipal | `min_cabecera` | `CAP_CABECERA = 45.0` | 0.30 |
| acceso a Puerto Antioquia | `min_puerto` | `CAP_PUERTO = 120.0` | 0.15 |

`acc_v3 = Σ(vᵢ·wᵢ)/Σwᵢ` sobre los componentes con dato (`recalc_v3.py:99-103`). CAPs en `recalc_v3.py:36-37`.

**CABA (distancia euclídea a equipamientos, `build_atlas_caba.py:183-192`).** Sub-scores:

| Sub-score (campo) | Insumo | CAP (m) | Peso |
|---|---|---|---|
| `score_salud` | distancia a CeSAC + hospitales (`cesac.geojson`, `hospitales_4326.geojson`) | 1500 | — |
| `score_educacion` | distancia a establecimientos educativos | 800 | — |
| `score_via` | transporte masivo (subte/metrobús/FFCC), colectivo, ecobici | mass 1000 / bus 400 / bici 500 | masivo 0.5, bus 0.3, bici 0.2 |

`score_via = 0.5·s_mass + 0.3·s_bus + 0.2·s_bici` (`build_atlas_caba.py:188`).
`score_accesibilidad = (score_salud + score_educacion + score_via)/3` (`build_atlas_caba.py:189`).

**Catálogo de indicadores Urabá** (mapeo indicador→dimensión en `aggregator.py:48-71`): accesibilidad = `IAV, IDEP, ICUL, ISAL, ISER, ISE`.
- `ISAL` (salud): m² de equipamiento de salud por habitante, radio 1500 m, tags OSM `hospital/clinic/doctors/health_post/pharmacy`; `invert=False` (`indicators/accesibilidad/salud.py:18-23`).
- `ISE` (educación): matrículas disponibles por niño/a 4–18 años, radio 1000 m; usa `matriculas_disponibles` real o estima `n_sedes × 300 cupos`; demanda = `poblacion_4_18` o `poblacion × 0.28`; `invert=False` (`indicators/accesibilidad/educacion.py:24-32,108`).

### Ambiental
Sub-scores satelitales (GEE):

| Sub-score (campo) | Insumo | Cómo |
|---|---|---|
| `score_ndvi` | NDVI Sentinel-2 (`COPERNICUS/S2_SR_HARMONIZED`, B8/B4) | más vegetación = mejor (`recalc_v3.py:116-117`, `ndvi.py:19,48`) |
| `score_calor` (LST) | LST Landsat 8/9 (`LANDSAT/LC08/C02/T1_L2`, `ST_B10`) | `1 − norm(LST)` → fresco=1, caliente=0 (`recalc_v3.py:85-86`) |

Conversión LST→°C: `LST_C = ST_B10·0.00341802 + 149.0 − 273.15` (`indicators/ambiental/temperatura.py:28,66-72`).
Dimensión ambiental = media ponderada NDVI 0.6 + calor 0.4 (`recalc_v3.py:120-126`); en CABA es media simple de los disponibles (`build_atlas_caba.py:246-247`).
Indicador alternativo `IATA` (amplitud térmica anual, `LST_cálido − LST_frío`, `invert=True`) en `temperatura.py:31-40` (cálido dic–mar, frío may–oct).

### Socioeconómico
| Sub-score (campo) | Insumo | Cómo |
|---|---|---|
| `score_socioeconomico` (CABA) `score_nbi` | NBI del censo (`H_CON_NBI`, `T_HOGAR`) | `nbi_rate = NBI/hogares`; `socio = clamp01(1 − nbi_rate)` (`build_atlas_caba.py:114-117`) |
| Urabá v3 | `score_economico_v2` (TerriData/SUI/EVA) | sin insumo nuevo → se hereda v2 (`recalc_v3.py:135`, `_meta` `recalc_v3.py:227`) |

Indicadores Urabá socioeconómico (`aggregator.py:60-65`): `IVI, ISV, IEJ, IRH, IEM, IPJ`.
- `IVI` calidad de vivienda: `pct_mat_inadecuada` (CNPV 2018), `invert=True` (`vivienda.py:31-34`).
- `ISV` hacinamiento: `0.6·pct_hacinamiento + 0.4·pct_hacinamiento_severo`, `invert=True` (`vivienda.py:80-81,113-116`).
- `IEJ` escolaridad jefe de hogar: `escolaridad_ponderada`/`esc_jefe_hogar`, `invert=False` (`escolaridad.py:32-35`).
- `IEM` empleo: `tasa_ocupacion` sobre PEA, `invert=False` (`empleo.py:46-49`).
- `IPJ` participación juvenil 15–24: `pct_jovenes_activos`, `invert=False` (`empleo.py:72-76`).
- `IRH` resiliencia: `pct_monoparental`, `invert=True` (`resiliencia.py:34-37`).

### Seguridad
| Sub-score (campo) | Insumo | Cómo |
|---|---|---|
| `score_viirs` / `score_seguridad` (CABA) | VIIRS `avg_rad` (luces nocturnas, GEE) | proxy luminosidad, normalizada en log: `norm(log(viirs+1))` (`build_atlas_caba.py:225-226,240`) |
| `score_ndbi` (display) | NDBI Sentinel-2 | impermeabilización `norm(ndbi)`, NO entra al score (`build_atlas_caba.py:239`; `recalc_v3.py:77`) |
| Urabá v3 | `score_seguridad` v2 | sin insumo nuevo → se hereda (`recalc_v3.py:139`) |

Indicadores Urabá seguridad (`aggregator.py:67-71`): `IGPE, IGPR, ILPE, ILPR` — densidad de delitos por **KDE** (`scipy.stats.gaussian_kde`) sobre puntos en **EPSG:3116**, bandwidth 500 m, evaluado en centroides; `invert=True` (más delitos = peor). Categorías `grave_persona/grave_propiedad/leve_persona/leve_propiedad`. Fallback sin eventos → 1.0 (seguridad máxima). (`indicators/seguridad/delitos.py:35-163,179`).

> Nota: en CABA `score_viirs` se usa como proxy de seguridad; el NDBI es solo para display. `score_calor` (LST) y `score_ndbi` son campos derivados de satélite.

---

## 3. Normalización — funciones reales

### `clamp01` (idéntica en ambos archivos)
```python
def clamp01(x):
    return max(0.0, min(1.0, x))
```
`recalc_v3.py:40-41` · `build_atlas_caba.py:52-53`.

### `norm` — min-max lineal (`recalc_v3.py:44-47`)
```python
def norm(x, lo, hi):
    if x is None or hi == lo:
        return None
    return clamp01((x - lo) / (hi - lo))
```
Rangos `lo/hi` desde min/max reales de los datos (`recalc_v3.py:28-30`: NDBI, LST, VIIRS).

### `norm_robust` — min-max con rango percentil p2–p98 (`build_atlas_caba.py:212-215`)
```python
def norm_robust(x, lo, hi):
    if x is None or hi == lo:
        return None
    return clamp01((x - lo) / (hi - lo))
```
Los límites se toman de los percentiles 2 y 98 (`build_atlas_caba.py:218-220`), no del min/max, para resistir outliers. VIIRS se normaliza en escala log: `log(viirs+1)` (`build_atlas_caba.py:225-226`).

### `acc_from_minutes` — accesibilidad desde minutos con CAP (`recalc_v3.py:50-54`)
```python
def acc_from_minutes(mins, cap):
    """0 min -> 1.0 ; cap o mas -> 0.0 (lineal). None -> None."""
    if mins is None:
        return None
    return clamp01(1.0 - (mins / cap))
```

### `acc_from_dist` — accesibilidad desde distancia con CAP (`build_atlas_caba.py:142-143`)
```python
def acc_from_dist(dist, cap):
    return clamp01(1.0 - dist / cap)
```

### Pipeline canónico `composite/normalizer.py`
Orden: **winsorize → normalize_minmax**.
- `winsorize(series, lower=0.01, upper=0.99)`: recorta a percentiles p1/p99 vía `scipy.stats.mstats.winsorize`; conserva NaN (`normalizer.py:14-46`).
- `normalize_minmax(series, invert=False)`: imputa NaN con **mediana**; si `min==max` retorna **0.5** constante; min-max a [0,1]; si `invert` → `1 − x` (`normalizer.py:49-85`).
- `Indicator.normalize` (base, `indicators/base.py:33-39`): min-max simple; `0.5` si `min==max`; `1−x` si `invert`.

### Quintiles
Breaks en los cuantiles 0.2/0.4/0.6/0.8 del `atlas_score`:
```python
qbreaks = [scores[int(n*q)] for q in (0.2, 0.4, 0.6, 0.8)]
```
`recalc_v3.py:150-152` → etiquetas `Q1-Crítico … Q5-Óptimo` (`recalc_v3.py:155-164`). CABA usa `Q1…Q5` con la misma lógica de cuantiles (`build_atlas_caba.py:277-284`).

---

## 4. Dimensión sin dato real → renormalizar sobre las disponibles

Regla: **solo se suman los pesos de las dimensiones con valor**; el numerador se divide por la **suma de pesos activos** (`wsum`), no por 1.10. Así el score sigue en [0,1] aunque falten dimensiones.

CABA — bucle de composición (`build_atlas_caba.py:265-273`):
```python
for p in atlas_feats:
    pr = p["properties"]
    num = wsum = 0.0
    for dim, w in W.items():
        v = pr.get(dim)
        if v is not None:
            num += v * w
            wsum += w
    pr["atlas_score"] = round(num / wsum, 4) if wsum > 0 else None
```

Fórmula según fase activa (literal de `build_atlas_caba.py:414-416`):
- **Fase 3** (todas las dims): `atlas_score = (0.40·acc + 0.25·amb + 0.25·soc + 0.20·seg)/1.10`
- **Fase 2** (solo accesibilidad + socioeconómico): `atlas_score = (0.40·acc + 0.25·soc)/0.65`
- **Fase 1** (solo socioeconómico): `atlas_score = score_socioeconomico = 1 − NBI`

Ejemplo CABA Fase 1→2: con solo accesibilidad (0.40) y socioeconómico (0.25) activos, `wsum = 0.65` y `atlas_score = (0.40·acc + 0.25·soc)/0.65`.

En el aggregator canónico la misma idea: `weight_used += w` solo si la dimensión no es toda-NaN, y `atlas_raw /= weight_used` si `|weight_used−1|>1e-6` (`aggregator.py:223-239`). Las dimensiones faltantes a nivel manzana se imputan con la mediana de esa dimensión antes de ponderar (`aggregator.py:230`).

> `recalc_v3.py` siempre divide por `W_SUM=1.10` fijo (`recalc_v3.py:144`) porque allí todas las dimensiones tienen valor (Urabá hereda v2 donde no hay insumo nuevo, ver §2). La renormalización dinámica por pesos activos es propia de la build por fases de CABA y del aggregator.

---

## 5. Zona (LISA / autocorrelación espacial local)

Implementado en `composite/spatial_clusters.py:17-64` con **PySAL** (`libpysal.weights`, `esda.moran`).

Procedimiento (`calcular_zonas_atlas`):
1. Reproyectar a **EPSG:3116** (MAGNA-SIRGAS Bogotá) — `spatial_clusters.py:42`.
2. `y = atlas_score` con NaN imputados a la media (`:43`).
3. Matriz de pesos espaciales **Queen** (default) o **Rook**, estandarizada por fila `W.transform="r"` (`:45-46`).
4. **Moran global** `Moran(y, W)` → campo `moran_i` (`:49,55`).
5. **LISA local** `Moran_Local(y, W)` → cuadrante `lisa.q` y `p_sim` (`:52,56-57`).
6. Clasificación de cuadrantes a `zona_atlas` (`:54,58-62`):

| `lisa.q` | Etiqueta | Significado |
|---|---|---|
| 1 | **HH** | alto bienestar rodeado de alto (zona próspera) |
| 2 | **LH** | bajo rodeado de alto (rezago en zona próspera) |
| 3 | **LL** | bajo rodeado de bajo (zona crítica) |
| 4 | **HL** | alto rodeado de bajo (isla de bienestar) |
| — | **NS** | `p_sim ≥ significance` → no significativo |

`significance = 0.05` por defecto (`:21,58-62`). Campos de salida: `moran_i`, `lisa_q`, `lisa_p`, `zona_atlas`. El resultado se devuelve en el CRS original (`:64`). En CABA, sin esta capa corrida, `zona_atlas` se inicializa en `"NS"` (`build_atlas_caba.py:134`).

---

## Apéndice — método alternativo PCA (no es el de producción)
`composite/pca_index.py` ofrece `PCAtlasIndex`: estandariza (StandardScaler), toma **PC1**, lo invierte si `loadings.mean()<0`, y lo escala min-max a [0,1] (`pca_index.py:98-125`). Es una alternativa a la media ponderada, no el score canónico del Atlas.

## Palabras clave
LISA, Moran, PySAL, esda, Queen, Rook, EPSG:3116, MAGNA-SIRGAS, min-max, winsorize, quintiles, NDVI, NDBI, LST, VIIRS, NBI, KDE, GEE, reduceRegions, Sentinel-2, Landsat, OSRM, isócronas, clamp01, normalización robusta, renormalización por dimensiones disponibles.
