# Contrato de datos — Front Nuxt de un Atlas de Tensor

Este documento describe **EXACTAMENTE** el contrato data-agnóstico de archivos que consume el
front Nuxt de un "Atlas de Bienestar Humano Territorial" (Tensor). El contrato está verificado
contra la implementación de referencia **Atlas CABA** (`atlas-caba-web`), que reusa el mismo
contrato que "Atlas Urabá". Si reproduces estos archivos para otro territorio respetando los
nombres de campo y rutas exactos, el front Nuxt + la API REST Nitro funcionan sin tocar código.

> **Vocabulario data-agnóstico (heredado de Urabá).** El front habla en tres niveles genéricos:
> - **manzana** = unidad base del índice. En CABA es el **radio censal** (INDEC/DGEyC 2010).
> - **municipio** = agregación nivel 1. En CABA es la **comuna** (15).
> - **vereda** = capa de contexto nivel 2. En CABA es el **barrio** (48).
>
> Los nombres de campo `cod_manzana`, `municipio`, `veredas.geojson` se mantienen literales
> aunque el territorio no tenga manzanas ni veredas — son el contrato, no etiquetas de UI.

Pipeline de referencia que **emite** estos archivos:
`atlas-caba-web/scripts/build_atlas_caba.py`. Todos los archivos se escriben en `public/data/`.

---

## 1) Tabla de archivos de salida

Todos viven en `public/data/` (servidos como estáticos al cliente). Un subconjunto se copia
además a `server/assets/data/` para que la API REST Nitro los lea como server-asset empaquetado
(ver §4).

| Archivo | Qué representa | Granularidad | Emitido en build_atlas_caba.py |
|---|---|---|---|
| `atlas.geojson` | Unidades base con índice + dimensiones + sub-scores. Es el corazón del contrato. | Una feature por **manzana** (radio censal). En CABA ~3.554. | Sí (líneas 288-289) |
| `municipios.geojson` | Agregación nivel 1 con `atlas_score` pop-weighted por unidad. | Una feature por **municipio** (comuna, 15). | Sí (líneas 317-318) |
| `veredas.geojson` | Capa de contexto (sin score), solo `nombre` + `comuna`. | Una feature por **vereda** (barrio, 48). | Sí (líneas 331-332) |
| `comunas_centroides.json` | Centroides + zoom por municipio para volar la cámara del mapa. Primer elemento es `"Todos"`. | Un objeto por municipio + 1 "Todos". | Sí (líneas 320-321) |
| `atlas_stats.json` | Stats por municipio que consume el **FRONT** (mapa/SidePanel): `count`, `avg`, `min`, `max`, `p25`, `p75` por dimensión. Incluye clave `"Todos"`. | Por municipio + "Todos". | Sí (líneas 377-378) |
| `atlas_breaks.json` | Cortes por cuantiles (7 anclas) por dimensión para coloreo de alto contraste en MapLibre. | Por dimensión. | Sí (líneas 399-400) |
| `atlas_stats_v3.json` | Stats que consume la **API REST**: `_meta`, `ranking_municipios_v3`, `municipios`. | Por municipio + meta global. | Sí (líneas 421-422) |
| `gap_analysis.json` | Narrativa/fortaleza/debilidad/gaps por municipio. En CABA es **placeholder** (`municipios: {}`). | Por municipio. | Sí (líneas 423-424) |
| `barrios_populares.geojson` | Capa de contexto adicional (asentamientos), copiada cruda. | Polígonos. | Sí (líneas 333-335) |
| `atlas.pmtiles` | Tile vectorial PMTiles de `atlas.geojson` para render de muchas features en MapLibre. | Igual que `atlas.geojson`. | **NO** lo emite el script Python — se genera aparte con **tippecanoe** (no presente en este pipeline). `[VERIFICAR]` el comando exacto. |
| `codigo_urbanistico.pmtiles` | Capa PMTiles de contexto urbanístico (CABA-específica). | Polígonos. | **NO** lo emite el script. `[VERIFICAR]` origen. |

Archivos crudos adicionales en `public/data/` (`caba_cesac.geojson`, `caba_educacion.geojson`,
`caba_subte.geojson`, `caba_metrobus.geojson`, `caba_ffcc.geojson`, `caba_hospitales.geojson`,
`caba_ciclovias.geojson`, `censo2022_*.geojson`, `conflicto_uso*.geojson`) son **capas de display
opcionales**, no parte del contrato núcleo. El script Python **no** los escribe (su entrada vive en
`data/raw/`, no en `public/data/`); se sirven como overlays. `[VERIFICAR]` quién los copia.

> `server/assets/data/` solo contiene los **tres** archivos que necesita la API: `atlas.geojson`,
> `atlas_stats_v3.json`, `gap_analysis.json` (verificado por `ls`). Son copias de los de
> `public/data/`. Mantener ambas copias sincronizadas es parte del contrato (ver §4).

---

## 2) Esquema de propiedades de cada feature de `atlas.geojson`

Cada feature es `{"type":"Feature","geometry":<Polygon|MultiPolygon>,"properties":{...}}`.
Las `properties` se construyen en `build_atlas_caba.py` (líneas 119-135, rellenadas en fases).
**Nombres EXACTOS** tal como aparecen en el código:

### Identidad y agregación
| Campo | Tipo | Significado | Origen en el script |
|---|---|---|---|
| `_fid` | int | Índice incremental de la feature. | `len(atlas_feats)` (l.120) |
| `cod_manzana` | string | **ID de la unidad base**. Clave de lookup de la API `/manzana/{cod}`. En CABA = `CO_FRAC_RA` del radio. | l.121 |
| `municipio` | string | **Prop de agregación nivel 1**. Debe coincidir con `MUNICIPIOS[].nombre` del store y con la clave de `atlas_stats*.json`. Formato CABA: `"Comuna N"` o `"Sin comuna"`. | l.110, l.123 |
| `comuna` | string | Alias de `municipio` (mismo valor en CABA). | l.123 |

### Población base
| Campo | Tipo | Origen |
|---|---|---|
| `total_pob` | int | `TOTAL_POB` del radio. Usado como peso en agregados pop-weighted. |
| `total_hogar` | int | `T_HOGAR`. |
| `total_vivienda` | int | `T_VIVIENDA`. |
| `hogares_nbi` | int | `H_CON_NBI`. |
| `nbi_rate` | float\|null | `hogares_nbi / total_hogar`. |

### Índice y dimensiones principales (las 4 del front)
| Campo | Tipo | Significado | Fase |
|---|---|---|---|
| `atlas_score` | float\|null | **Índice compuesto** en [0,1]. Media ponderada de las dims disponibles, normalizada por el peso activo. | §3 del script (l.265-273) |
| `score_socioeconomico` | float\|null | `1 - nbi_rate`, clamp [0,1]. | FASE 1 (l.117) |
| `score_accesibilidad` | float\|null | `media(salud, educación, transporte)`. | FASE 2 (l.193) |
| `score_ambiental` | float\|null | `media(score_ndvi, score_calor)`. | FASE 3 (l.247) |
| `score_seguridad` | float\|null | Luminosidad nocturna VIIRS (log-normalizada) como proxy. | FASE 3 (l.244) |

### Sub-scores (paneles, `DIMENSIONES_V2`)
| Campo | Tipo | Significado | Fase |
|---|---|---|---|
| `score_salud` | float\|null | Cercanía a CeSAC + hospitales (cap 1500 m). | FASE 2 (l.190) |
| `score_educacion` | float\|null | Cercanía a establecimientos educativos (cap 800 m). | FASE 2 (l.191) |
| `score_via` | float\|null | Transporte: `0.5*masivo + 0.3*colectivo + 0.2*bici`. Mapea a label "Transporte". | FASE 2 (l.192) |
| `score_ndvi` | float\|null | Vegetación (NDVI normalizado robusto p2-p98). | FASE 3 (l.241) |
| `score_calor` | float\|null | Frescura = `1 - LST_norm` (anti isla de calor). | FASE 3 (l.242) |
| `score_impermeabilizacion` | float\|null | NDBI normalizado (solo display). | FASE 3 (l.243) |

### Variables crudas satelitales (display, no normalizadas)
| Campo | Tipo | Origen |
|---|---|---|
| `ndvi` | float\|null | NDVI crudo. |
| `lst_c` | float\|null | Temperatura superficie (°C). |
| `viirs` | float\|null | Radiancia nocturna VIIRS cruda. |

### Clasificación territorial
| Campo | Tipo | Valores | Origen |
|---|---|---|---|
| `quintil` | string\|null | `"Q1"`..`"Q5"` sobre `atlas_score`. | l.278-286 |
| `zona_atlas` | string | **Zona LISA / Moran local**. Valores del contrato: `"HH"`, `"LL"`, `"HL"`, `"LH"`, `"NS"`. | Inicializado a `"NS"` (l.134) |

> **IMPORTANTE — `zona_atlas` (LISA HH/LL/HL/LH/NS).** El front filtra por estas cinco clases
> (`zonaFilter: ['HH','LL','HL','LH','NS']` en el store). El script de CABA inicializa
> `zona_atlas = "NS"` (l.134) pero **NO calcula LISA / Moran's I** en esta versión: todas las
> features quedan en `"NS"`. La API expone también `moran_i` (l.38 de `manzana/[cod]`), campo que
> **el script Python NO escribe**. Para reproducir con clústeres reales debes computar LISA
> (p.ej. con `libpysal`/`esda`) y poblar `zona_atlas` ∈ {HH,LL,HL,LH,NS} más `moran_i` por unidad.
> Marcado `[VERIFICAR]` el origen de LISA en otros territorios.

> **Campos `*_v3` que la API lee pero el script CABA no escribe.** La API hace fallback
> (`p.atlas_score_v3 ?? p.atlas_score`, `p.quintil_v3 ?? p.quintil`,
> `p.score_accesibilidad_v3 ?? p.score_accesibilidad`, etc.). En CABA solo existen las versiones
> sin sufijo `_v3` a nivel de feature; los `_v3` son un hook para una variante "v3" del índice.

---

## 3) Constantes editables de `app/stores/atlas.js`

Estas constantes son **lo único** que hay que editar para re-territorializar la UI. Verificadas
en `atlas-caba-web/app/stores/atlas.js`.

### `DIMENSIONES` (selector del mapa + pills del sheet)
Array de `{ key, label, color }`. `key` **debe** ser un nombre de campo presente en las
properties de `atlas.geojson`. CABA expone:

| key | label | color |
|---|---|---|
| `atlas_score` | Índice Atlas | `#1B6B6D` |
| `score_accesibilidad` | Accesibilidad | `#60a5fa` |
| `score_ambiental` | Ambiental | `#34d399` |
| `score_socioeconomico` | Socioeconómico (NBI) | `#a78bfa` |
| `score_seguridad` | Seguridad (NTL) | `#fbbf24` |

### `DIMENSIONES_V2` (sub-índices en paneles)
`{ key, label, shortLabel, color }`: `score_salud`, `score_educacion`, `score_via`,
`score_ndvi`, `score_calor`.

### `MUNICIPIOS` (regla de coincidencia crítica)
Array de `{ nombre, lat, lng, zoom, barrios }`. Primer elemento siempre `{ nombre: 'Todos', ... }`.

> **REGLA DURA:** `MUNICIPIOS[].nombre` **debe coincidir EXACTAMENTE** con la propiedad
> `municipio` de `atlas.geojson` **y** de `municipios.geojson`. Está documentado en el propio
> store (líneas 24-26: *"El campo `nombre` debe coincidir exactamente con la propiedad `municipio`
> de atlas.geojson y municipios.geojson"*). En CABA ambos lados usan el literal `"Comuna N"`.
> Si no coinciden: `municipioConfig` (getter) devuelve `undefined`, la cámara no vuela, y
> `stats[municipio]` no resuelve. `lat`/`lng`/`zoom` deben replicar `comunas_centroides.json`.

### `COLOR_EXPR_TEMPLATE` (rampa de color MapLibre)
Expresión `interpolate / linear` de MapLibre GL. El placeholder `__FIELD__` se reemplaza en
runtime por la `key` de la dimensión activa (`['get','__FIELD__']`). Rampa rojo→verde anclada en
`0.0 #d73027 … 1.0 #1a9850`. El dominio asume scores normalizados en **[0,1]**.

### State / getters / actions relevantes
- **state:** `dimension` (default `'atlas_score'`), `municipioActivo` (`'Todos'`),
  `manzanaSeleccionada`, `stats` (`{}`), `filterMin` (0), `filterMax` (1),
  `zonaFilter` (`['HH','LL','HL','LH','NS']`).
- **getters:** `dimensionActual`, `municipioConfig` (match por `nombre`), `hasFilters`,
  `regionScore` (media pop-weighted de `avg.atlas_score` sobre `stats`).
- **persist:** solo persiste `['dimension','municipioActivo']`.

---

## 4) Contrato de la API REST (Nitro `server/api/caba/`)

Utilidades en `server/utils/caba.js`:
- `readData(file)` lee desde el storage de server-assets con clave `server:data:<archivo>`
  (montado desde `server/assets/data/`, baseName `data`), cacheado en memoria. Por eso esos
  3 archivos deben existir en `server/assets/data/` además de `public/data/`.
- `normalize(str)` → minúsculas, sin acentos, trim (búsqueda case/acentos-insensible).
- `setApiHeaders(event)` → CORS abierto `*`, `Content-Type: application/json; charset=utf-8`,
  `Cache-Control: public, max-age=300`.
- `notFound(message)` → error 404 con cuerpo JSON `{error:'not_found', mensaje, fuente}`.
- `FUENTE` → string de atribución de fuentes (constante).

| Ruta | Lee | Devuelve |
|---|---|---|
| `GET /api/caba` | (ninguno) | Documento índice: `nombre`, `version`, `descripcion`, `cobertura` (comunas/barrios/radios/dimensiones/subindices), `formula`, `endpoints[]`, `fuente`. Todo estático/hardcodeado. |
| `GET /api/caba/municipios` | `atlas_stats_v3.json` | `{ total, municipios[], fuente }`. Cada municipio: `comuna`, `atlas_score` (=`atlas_score_v3`), `ranking`, `radios` (=`count`), `dimensiones{accesibilidad,ambiental,socioeconomico,seguridad,salud,educacion,transporte,ndvi,frescura}`. Ordenado desc por `atlas_score`. |
| `GET /api/caba/municipio/{nombre}` | `atlas_stats_v3.json` + `gap_analysis.json` + `atlas.geojson` | `{ comuna, ranking, total_comunas, atlas_score, radios, dimensiones{...9}, top5_radios[], fuente }`. `{nombre}` resuelto con `normalize` contra claves de `stats.municipios`. `top5_radios` = 5 features de ese municipio con mayor `atlas_score`. 404 si no resuelve. |
| `GET /api/caba/ranking` | `atlas_stats_v3.json` | `{ total, formula, ranking[], fuente }`. Cada item: `ranking`, `municipio`, `atlas_score_v3`, `dimensiones{accesibilidad,ambiental,socioeconomico,seguridad}`. Toma `ranking_municipios_v3` y lo enriquece desde `municipios`. |
| `GET /api/caba/manzana/{cod}` | `atlas.geojson` (índice `cod_manzana→properties` cacheado a nivel módulo) | `{ cod_manzana, municipio, atlas_score_v3 (fallback a atlas_score), quintil (quintil_v3 ‖ quintil), zona_atlas, moran_i, dimensiones{accesibilidad,ambiental,socioeconomico,seguridad} (cada una con fallback `_v3 ?? base`), propiedades (objeto completo), fuente }`. 404 si `cod` no existe. |

> **Nota de mapeo de dimensiones en la API.** La API renombra los campos del feature a etiquetas
> "humanas": `score_via → transporte`, `score_calor → frescura`. El campo crudo del contrato sigue
> siendo `score_via`/`score_calor` en `atlas.geojson`; el rename ocurre solo en la capa REST.

> **Sincronía de assets (contrato operativo).** La API **solo** funciona si `atlas.geojson`,
> `atlas_stats_v3.json` y `gap_analysis.json` están copiados en `server/assets/data/`. Tras cada
> rebuild del pipeline hay que re-copiar esos tres a `server/assets/data/` (el `ls` confirma que
> son los únicos tres presentes ahí). `[VERIFICAR]` si existe un paso de copia automatizado.

---

## Estructura de `atlas_stats_v3.json` (referencia para la API)

```
{
  "_meta": { "version", "generado", "unidad_base", "formula",
             "dimensiones": [...], "quintil_breaks": [...], "radios", "comunas" },
  "ranking_municipios_v3": [ { "municipio", "atlas_score_v3" }, ... ],   // desc
  "municipios": {
    "Comuna N": { "count", "atlas_score_v3", "dimensiones": { <avg por dim> } }
  }
}
```

`dimensiones` dentro de `municipios[*]` proviene de `front_stats[name].avg` (las claves son los
nombres de campo `score_*`). La API lee `d.dimensiones?.score_accesibilidad`, etc. — por eso las
claves de `avg` deben ser los **mismos nombres exactos** que en las features.

---

## Checklist de reproducción para otro territorio

1. Emitir `atlas.geojson` con properties de §2 (mínimo: `cod_manzana`, `municipio`, `atlas_score`,
   los `score_*` que vayas a exponer, `zona_atlas`, `quintil`).
2. Emitir `municipios.geojson` con `municipio` + `atlas_score` agregado, y `veredas.geojson`
   (`nombre`, `comuna`).
3. Emitir `atlas_stats.json` (front) y `atlas_stats_v3.json` (API) con la estructura de arriba;
   las claves de `municipios` y de `avg`/`dimensiones` = nombres exactos.
4. Emitir `comunas_centroides.json` (1er elemento `"Todos"`), `atlas_breaks.json`, `gap_analysis.json`.
5. Copiar `atlas.geojson`, `atlas_stats_v3.json`, `gap_analysis.json` a `server/assets/data/`.
6. Editar `stores/atlas.js`: `DIMENSIONES`, `DIMENSIONES_V2`, `MUNICIPIOS` (¡`nombre` == `municipio`!),
   centroides. No tocar `COLOR_EXPR_TEMPLATE` salvo cambiar la rampa.
7. (Opcional) Generar `atlas.pmtiles` con tippecanoe si hay demasiadas features para GeoJSON crudo.
8. (Opcional/recomendado) Calcular LISA/Moran's I real para poblar `zona_atlas` y `moran_i`.

## Anclajes marcados [VERIFICAR]
- Comando **tippecanoe** que genera `atlas.pmtiles` (no está en el script Python).
- Origen de `codigo_urbanistico.pmtiles` y de las capas overlay `caba_*.geojson` / `censo2022_*`.
- Paso de copia que sincroniza `public/data/` → `server/assets/data/`.
- Cálculo de `zona_atlas` (LISA) y `moran_i`: ausente en el script CABA (todo queda `"NS"`).
- Campos `*_v3` a nivel feature: la API los espera con fallback, pero el script CABA no los emite.
