# Anatomía del codebase Atlas (Nuxt) y guía de clonación a otro territorio

Referencia técnica para clonar el codebase del Atlas de Bienestar Humano Territorial (stack **Nuxt 3/4 + Pinia + MapLibre/PMTiles + Nitro `preset: vercel`**) desde un territorio a otro, manteniendo intacta la lógica de scores y solo cambiando el vocabulario territorial, los datos y los metadatos.

> **Anclaje:** todo lo de este documento está verificado leyendo el código real de `/Users/cristianespinal/atlas-uraba-web` (estructura, `app/stores/atlas.js`, `server/utils/uraba.js`, `server/api/uraba/**`, `nuxt.config.ts`, `scripts/sync-api-assets.mjs`) y la §5 "Plan de clonación del codebase" de `/Users/cristianespinal/atlas-caba-web/docs/PLAN_CONSTRUCCION.md`. Lo no verificado va marcado `[VERIFICAR]`.

---

## 1. Mapa de carpetas del front

Estructura real observada en `atlas-uraba-web` (la raíz contiene `app/`, `server/`, `public/`, `scripts/`, `docs/`, `nuxt.config.ts`, `tailwind.config.js`, `vercel.json`, más los scripts Python `recalc_v3.py` y `script_clasificacion.py`).

| Carpeta / archivo | Qué hace |
|---|---|
| `app/stores/atlas.js` | Único store Pinia (`useAtlasStore`). Exporta las constantes de dominio `DIMENSIONES`, `DIMENSIONES_V2`, `MUNICIPIOS` (lista nombre/lat/lng/zoom) y `COLOR_EXPR_TEMPLATE` (rampa de color MapLibre). Mantiene estado UI: `dimension`, `municipioActivo`, `manzanaSeleccionada`, `stats`, filtros (`filterMin/Max`, `zonaFilter` LISA `HH/LL/HL/LH/NS`). Persiste `dimension` y `municipioActivo`. **Es el punto de edición central al clonar.** |
| `app/composables/` | Lógica reutilizable. `useAtlasMap.js` (~69 KB): instancia MapLibre, carga PMTiles, aplica `COLOR_EXPR_TEMPLATE` y capas. `useAtlasStats.js`: deriva stats agregadas del store. `useSimulador.js`: lógica de la página `/simulador`. |
| `app/components/` | ~28 componentes Vue. Mapa/capas: `AtlasMap.vue`, `LayerPanel.vue`, `LayerToggle.vue`, `MapLegend.vue`, `SatelliteToggle.vue`, `GeocoderSearch.vue`. Paneles de unidad: `FichaMunicipal.vue` (modal policy-brief por municipio, ~32 KB), `ManzanaPanel.vue`, `SidePanel.vue`, `MobileSheet.vue`, `PanoramaPanel.vue`, `DiagnosticoPanel.vue`. Util UI: `AppHeader.vue`, `AppFooter.vue`, `KPIItem.vue`, `AtlasKPIBar.vue`, `HistogramPanel.vue`, `ScoreRankingList.vue`, `FilterBar.vue`, `ShareButton.vue`, `DownloadButton.vue`, `TensorLogo.vue`, etc. Subcarpeta `comparar/` (`CompararMiniMapa.vue`, `CompararRadar.vue`). |
| `app/pages/` | 3 rutas: `index.vue` (home con mapa), `comparar.vue`, `simulador.vue`. Las dos últimas se sirven como SPA (`ssr:false`) vía `routeRules` en `nuxt.config.ts`. |
| `app/` (otros) | `app.vue` (raíz), `assets/` (CSS, incl. `assets/css/main.css`), `plugins/`. |
| `server/utils/uraba.js` | Utilidades de la API REST pública. Exporta `FUENTE` (string de atribución de fuentes), `readData(file)` (lee server assets vía `useStorage('assets')` con clave `server:data:<file>` + cache en memoria), `normalize(str)` (lower + sin acentos NFD), `setApiHeaders(event)` (CORS abierto + JSON + `Cache-Control: max-age=300`), `notFound(message)`. **Se renombra al clonar.** |
| `server/api/uraba/` | Endpoints REST (Nitro). `index.get.js` (doc de la API), `municipios.get.js`, `ranking.get.js`, `municipio/[nombre].get.js` (detalle por municipio: score, dimensiones, ranking, narrativa, top5 manzanas), `manzana/[cod].get.js` (propiedades de una manzana por `cod_manzana`, con índice cacheado). **Carpeta y rutas dinámicas se renombran al clonar.** |
| `server/assets/data/` | Sólo los 3 JSON/GeoJSON que la API lee en runtime: `atlas.geojson`, `atlas_stats_v3.json`, `gap_analysis.json`. Van DENTRO del bundle serverless (no por filesystem). |
| `public/data/` | Fuente de verdad de datos servidos al cliente: `atlas.pmtiles` (tiles del mapa) + GeoJSON ligeros (`atlas_slim.geojson`, `atlas_enriquecido.geojson`), `atlas_stats*.json`, `gap_analysis.json`, `benchmarks.json`, `editorial.json`, capas temáticas (`equipamientos.geojson`, `eva_agro.geojson`, etc.) y CSV de indicadores (`indicadores_ndvi.json`, `indicadores_luminosidad.csv`...). |
| `scripts/sync-api-assets.mjs` | Copia los archivos de la lista `FILES = ['atlas_stats_v3.json','gap_analysis.json','atlas.geojson']` desde `public/data` → `server/assets/data` para que Nitro los empaquete. **Mantener `FILES` alineada con lo que lee `server/utils/<t>.js`.** |
| `nuxt.config.ts` | Config raíz. `nitro.preset: 'vercel'` + `serverAssets: [{ baseName: 'data', dir: 'server/assets/data' }]`. `routeRules` SPA para `/comparar` y `/simulador`. Todo el bloque `app.head` (title, description, OG, Twitter, canonical, locale, theme-color). **Se edita al clonar (metadatos + dominio).** |
| `recalc_v3.py` / `script_clasificacion.py` (raíz) | Pipeline Python que recalcula scores (pesos 0.40/0.25/0.25/0.20) y la clasificación LISA/quintiles. Se adapta a los datasets del nuevo territorio conservando la fórmula. |

---

## 2. Archivos a editar al clonar y qué cambiar en cada uno

Ejemplo concreto: Urabá → CABA (Buenos Aires), tomado de §5 del PLAN_CONSTRUCCION de `atlas-caba-web`. Generaliza usando `<t>` = slug del territorio nuevo.

| Archivo (origen) | Acción | Qué cambiar |
|---|---|---|
| `app/stores/atlas.js` | EDITAR | Reemplazar la lista `MUNICIPIOS` (nombre/lat/lng/zoom) por las unidades de agregación del nuevo territorio y su centro de mapa. Para CABA: `MUNICIPIOS` → `BARRIOS` (48 barrios), centro `lat:-34.61, lng:-58.42, zoom:11`. **Mantener `DIMENSIONES`, `DIMENSIONES_V2` y `COLOR_EXPR_TEMPLATE` sin cambios** (la lógica de scores es idéntica). Añadir el objeto `VOCAB` (ver §3). |
| `server/utils/uraba.js` → `server/utils/<t>.js` | RENOMBRAR + EDITAR | `git mv` a `<t>.js`. Reescribir la constante `FUENTE` con las fuentes del territorio. Para CABA: "Atlas CABA · Tensor. Insumos: BA Data / DGEyC-GCBA (radios censales + NBI), GEE, OSRM, INDEC". Si cambian los nombres de archivo de datos, ajustar las cadenas pasadas a `readData(...)`. |
| `server/api/uraba/` → `server/api/<t>/` | RENOMBRAR carpeta | `git mv server/api/uraba server/api/<t>`. Dentro, renombrar las rutas dinámicas a la unidad del nuevo territorio: `municipio/[nombre]` → p.ej. `barrio/[nombre]`, `manzana/[cod]` → `radio/[cod]`. Actualizar los `import` que apuntan a `../../utils/uraba` → `../../utils/<t>` (y `../../../utils/...` en handlers anidados). Actualizar `index.get.js`: `nombre`, `descripcion`, `cobertura`, `formula`, lista de `endpoints` (las rutas `/api/uraba/...` → `/api/<t>/...`). |
| `server/api/.../municipio/[nombre].get.js` y `.../manzana/[cod].get.js` | EDITAR (tras renombrar) | Los handlers usan campos del GeoJSON (`municipio`, `cod_manzana`, `atlas_score_v3`, `score_*_v3`, `quintil_v3`, `zona_atlas`, `moran_i`). **Mantener los nombres de campo del GeoJSON idénticos** (mapeo 1:1 interno) para no tocar la lógica; el pipeline Python debe emitir esos mismos campos. Si se decide renombrar campos del dataset, hay que actualizarlos aquí también (no recomendado en la primera corrida). |
| `app/components/FichaMunicipal.vue` → `FichaBarrial.vue` | RENOMBRAR + EDITAR import | `git mv`. Actualizar el import en `app/pages/index.vue` (verificado: `index.vue` referencia `FichaMunicipal`). Las etiquetas de texto visibles deben salir de `VOCAB` (ver §3), no estar hardcodeadas. |
| Componentes con etiquetas literales | EDITAR | Verificado que usan literales "Municipio"/"Manzana": `AtlasKPIBar.vue`, `ScoreRankingList.vue`, `DownloadButton.vue`, `MobileSheet.vue`, `SidePanel.vue`, `HistogramPanel.vue`, `AppHeader.vue`, `EditorialTooltip.vue`, `ShareButton.vue`, `DiagnosticoPanel.vue`, `LayerPanel.vue`, `LayerToggle.vue`, `PanoramaPanel.vue`, `ManzanaPanel.vue`, `GeocoderSearch.vue`, `comparar/CompararMiniMapa.vue`, y pages `index.vue`, `comparar.vue`, `simulador.vue`. Reemplazar literales por lecturas de `VOCAB` (§3). |
| `nuxt.config.ts` | EDITAR `app.head` | `title` (p.ej. `'Atlas CABA — Bienestar Humano Territorial'`); `description` (conteos del nuevo territorio); todas las URLs OG/canonical `https://uraba.tensor.lat` → dominio nuevo (CABA: `https://argentina.tensor.lat`) en `og:url`, `og:image`, `twitter:image`, y `link rel="canonical"`; `og:title`/`twitter:title` y `og:site_name`; `og:locale` `es_CO` → `es_AR`; `theme-color` se conserva. Mantener `nitro.preset/serverAssets` y `routeRules`. |
| `scripts/sync-api-assets.mjs` | EDITAR si cambian nombres | Si los 3 archivos de la API cambian de nombre, actualizar el array `FILES`. Si conservan nombre (`atlas_stats_v3.json`, `gap_analysis.json`, `atlas.geojson`), no se toca. |
| `recalc_v3.py` / `script_clasificacion.py` | ADAPTAR | Apuntar a los GeoJSON/CSV del nuevo territorio; **conservar la fórmula de pesos (0.40/0.25/0.25/0.20) y los quintiles/LISA**, para que los campos de salida coincidan con los que esperan API y front. |

### Renombres semánticos globales (buscar/reemplazar con criterio)
- `uraba` / `Urabá` → `<t>` / `<Territorio>` (no romper imports hasta hacer `git mv`).
- Cadenas de cobertura ("7,028 manzanas · 8 municipios") → conteos reales del nuevo territorio.
- Citas de fuente ("DANE CNPV 2018") → fuente del territorio (CABA: "INDEC Censo 2010 / DGEyC-GCBA").

### Comandos de fork (verificados en §5)
```bash
cp -R /Users/cristianespinal/atlas-uraba-web /Users/cristianespinal/atlas-<t>-web
cd /Users/cristianespinal/atlas-<t>-web
rm -rf node_modules .nuxt .vercel .git public/data/* server/assets/data/*
git init && git add -A && git commit -m "fork: atlas-uraba-web base for Atlas <T>"
gh repo create Cespial/atlas-<t>-web --private --source=. --remote=origin
git mv server/api/uraba server/api/<t>
git mv server/utils/uraba.js server/utils/<t>.js
git mv app/components/FichaMunicipal.vue app/components/Ficha<Unidad>.vue
npm install
# editar atlas.js / nuxt.config.ts / <t>.js / VOCAB según §2 y §3
# correr pipeline de datos FASE 1 → poblar public/data + server/assets/data
npm run dev
```
Deploy: nuevo proyecto Vercel apuntado al dominio del territorio. El preset `vercel` y `serverAssets` ya están listos; sólo cambia el dominio.

---

## 3. DISEÑO — Capa de alias de vocabulario (`VOCAB`)

> **Esta es la guía a implementar. Refactor único en la primera corrida de `atlas-web`** (el repo base genérico). Hoy las etiquetas territoriales están hardcodeadas en ~20 componentes y pages (verificado en §1 del listado de consumidores); el objetivo es centralizarlas en un solo objeto para que clonar a un territorio sea editar UN bloque, no buscar/reemplazar literales en toda la UI.

### Principio
Las **rutas, variables, campos de GeoJSON y claves de API permanecen 1:1 e invariantes** entre territorios: internamente siempre `manzana` / `municipio` / `vereda` (la unidad base, la agregación media y la agregación gruesa). Lo único que cambia por territorio son las **etiquetas visibles**. Esas etiquetas salen de un objeto `VOCAB` definido en `app/stores/atlas.js` y exportado, consumido por componentes y labels.

Esto desacopla:
- **Capa interna (estable):** `unidad_base` = `manzana`, `agg1` = `municipio`, `agg2` = `vereda`. Mismas rutas (`/api/<t>/manzana/[cod]`, `/api/<t>/municipio/[nombre]`), mismos campos (`cod_manzana`, `municipio`, `atlas_score_v3`).
- **Capa visible (por territorio):** Urabá muestra "manzana / municipio / vereda"; CABA muestra "radio censal / barrio / comuna".

### Forma del objeto (a añadir en `app/stores/atlas.js`)
```js
// Capa de alias de vocabulario territorial.
// Las CLAVES (unidad_base/agg1/agg2) son INTERNAS e invariantes y mapean 1:1
// a las rutas/variables/campos del codebase (manzana/municipio/vereda).
// Los VALORES son las etiquetas VISIBLES, editables por territorio al clonar.
export const VOCAB = {
  unidad_base: {            // interno: "manzana" (rutas /manzana/[cod], campo cod_manzana)
    singular: 'manzana',
    plural:   'manzanas',
    cod:      'cod_manzana', // nombre del campo identificador (no cambiar entre territorios)
  },
  agg1: {                   // interno: "municipio" (ruta /municipio/[nombre], campo municipio)
    singular: 'municipio',
    plural:   'municipios',
  },
  agg2: {                   // interno: "vereda" (agregación gruesa)
    singular: 'vereda',
    plural:   'veredas',
  },
  // Etiqueta del territorio para títulos/atribución de UI (no para nuxt.config).
  territorio: { nombre: 'Urabá', gentilicio: 'urabaense' }, // [VERIFICAR] gentilicio no requerido
}
```

Para CABA, al clonar, el mismo objeto pasa a:
```js
unidad_base: { singular: 'radio censal', plural: 'radios censales', cod: 'cod_manzana' },
agg1:        { singular: 'barrio',       plural: 'barrios' },
agg2:        { singular: 'comuna',       plural: 'comunas' },
territorio:  { nombre: 'CABA', gentilicio: 'porteño' },
```
Nótese: `cod` sigue siendo `cod_manzana` (mapeo interno 1:1), aunque la etiqueta diga "radio censal". Las rutas siguen siendo `/api/caba/manzana/[cod]` salvo que se decida también renombrarlas (decisión independiente; no es requisito del VOCAB de UI).

### Dónde se consume

| Consumidor | Uso de `VOCAB` |
|---|---|
| `app/components/Ficha*.vue` (Municipal/Barrial) | Título y encabezados: `VOCAB.agg1.singular`. |
| `ManzanaPanel.vue`, `SidePanel.vue`, `MobileSheet.vue` | Etiqueta de la unidad base seleccionada: `VOCAB.unidad_base.singular`. |
| `AtlasKPIBar.vue`, `ScoreRankingList.vue`, `HistogramPanel.vue` | Conteos plurales: `${n} ${VOCAB.unidad_base.plural}`, `${m} ${VOCAB.agg1.plural}`. |
| `AppHeader.vue`, `AppFooter.vue`, `index.vue` | Copys de cabecera/footer y descripción del territorio. |
| `comparar.vue`, `CompararMiniMapa.vue`, `CompararRadar.vue` | Selector y leyendas "comparar {agg1.plural}". |
| `LayerPanel.vue`, `LayerToggle.vue`, `MapLegend.vue`, `GeocoderSearch.vue` | Labels de capas y placeholder del buscador. |
| `DownloadButton.vue`, `ShareButton.vue` | Texto de descarga/compartir por unidad. |

Acceso en componentes vía el store (`useAtlasStore` ya es importado en casi todos): exportar `VOCAB` desde el store e importarlo directamente, p.ej.
```js
import { VOCAB } from '~/stores/atlas'
// template: {{ VOCAB.unidad_base.plural }}
```
o exponerlo como getter (`getters: { vocab: () => VOCAB }`) si se prefiere acceder vía `store.vocab` reactivo. **[VERIFICAR]** cuál convención sigue el repo base `atlas-web` cuando se cree; recomendado: import directo de la constante (no necesita reactividad porque es estático por build).

### Regla de oro del refactor
1. **No tocar** rutas de API, nombres de archivo de datos, ni campos de GeoJSON: siguen siendo `manzana/municipio/...` y `cod_manzana/atlas_score_v3/...`.
2. **Sí reemplazar** cada literal de UI ("Municipio", "Manzana", "manzanas", etc.) por la lectura de `VOCAB`.
3. Tras el refactor, clonar a un territorio nuevo = editar `MUNICIPIOS`/centro + el bloque `VOCAB` + `FUENTE` + `nuxt.config.ts head`, sin recorrer 20 archivos de UI.

---

## 4. Checklist de clonación (resumen accionable)
1. `cp -R` + limpiar (`node_modules/.nuxt/.vercel/.git/public-data/server-assets`).
2. `git init` + repo remoto.
3. `git mv` carpeta API, util y componente Ficha.
4. Editar `stores/atlas.js`: `MUNICIPIOS`→unidades + centro + bloque `VOCAB`.
5. Editar `server/utils/<t>.js`: `FUENTE` (+ nombres de archivo si cambian).
6. Editar `server/api/<t>/index.get.js`: nombre/descripción/cobertura/fórmula/endpoints.
7. Reemplazar literales UI por `VOCAB` en componentes/pages.
8. Editar `nuxt.config.ts`: title/description/OG/canonical/dominio/locale.
9. Alinear `scripts/sync-api-assets.mjs` (`FILES`) si cambian nombres.
10. Adaptar `recalc_v3.py`/`script_clasificacion.py` a datasets nuevos, conservando fórmula y campos de salida.
11. Poblar `public/data` + `server/assets/data` con el pipeline de datos.
12. `npm install` → `npm run dev` → deploy Vercel con dominio nuevo.
