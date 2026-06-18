# Diseño — Plugin `tensor-atlas`

> Serie de skills empaquetadas como plugin de Claude Code para construir, paso a paso,
> cualquier Atlas de Tensor (estilo `uraba.tensor.lat`, `caba.tensor.lat`).
> Fecha: 2026-06-18 · Autor: Cristian + Claude.

---

## 1. Objetivo

Replicar de forma repetible el método con el que se construyeron el Atlas Urabá y el
Atlas CABA: un sistema territorial de bienestar humano compuesto por (a) un **pipeline
de datos en Python** que produce un contrato fijo de archivos, y (b) un **front Nuxt
data-agnóstico** que los pinta. El plugin descompone ese método en skills invocables que
permiten arrancar un atlas nuevo para cualquier territorio de LatAm.

**No-objetivos (YAGNI):** no reescribir el front desde cero; no construir un CMS; no
soportar (todavía) territorios fuera de LatAm con recetas dedicadas; no automatizar la
gestión de datos que requieren derecho de petición (se documentan como brechas, no se
resuelven).

## 2. Hallazgo clave — el contrato data-agnóstico

El front Nuxt consume archivos con nombres y forma fijos, independientes del territorio.
**Clonar un atlas = reemplazar datos + renombrar constantes + rebranding**, no reescribir
lógica. Contrato observado en `atlas-uraba-web` / `atlas-caba-web`:

| Archivo (en `public/data/` o `server/assets/data/`) | Contenido | Urabá → CABA |
|---|---|---|
| `atlas.geojson` | unidad base; props: `atlas_score`, `score_accesibilidad`, `score_ambiental`, `score_socioeconomico`, `score_seguridad`, sub-scores `score_salud/educacion/via/ndvi/calor/ndbi/viirs`, `zona` (LISA: HH/LL/HL/LH/NS), id de unidad, `municipio` (= agg nivel 1) | manzana CNPV → radio censal |
| `municipios.geojson` | agregación nivel 1 con `atlas_score` agregado | municipio → comuna |
| `veredas.geojson` | agregación nivel 2 (contexto) | vereda → barrio |
| `atlas_stats.json` | stats por agg nivel 1 que consume el FRONT (mapa/SidePanel) | idéntico |
| `atlas_stats_v3.json` | stats para la API REST Nitro | idéntico |
| `gap_analysis.json`, `comunas_centroides.json` | metadatos | idéntico |
| `*.pmtiles` | capas pesadas (tippecanoe) | idéntico |

Constantes editables en `app/stores/atlas.js`:
- `DIMENSIONES` — dimensiones expuestas en el selector del mapa (key/label/color).
- `DIMENSIONES_V2` — sub-índices mostrados en paneles.
- `MUNICIPIOS` — centroides de la agregación nivel 1 (`nombre`/`lat`/`lng`/`zoom`); el
  `nombre` DEBE coincidir con la prop `municipio` de `atlas.geojson`.
- `COLOR_EXPR_TEMPLATE` — rampa de color MapLibre (no cambia entre atlas).

## 3. Modelo de score (estable entre atlas)

Pesos: `accesibilidad 0.40 · ambiental 0.25 · socioeconómico 0.25 · seguridad 0.20`
(suma 1.10, el código normaliza). Cuando una dimensión no tiene dato real, se normaliza
sobre las dimensiones disponibles (p.ej. CABA Fase 1: `atlas_score = (0.40·acc + 0.25·soc)/0.65`).
Sub-scores construibles según dato: `score_salud`, `score_educacion`, `score_via`,
`score_ndvi`, `score_calor`/`lst`, `score_ndbi`, `score_viirs`, `score_nbi`.
Normalización por min-max o quintiles; `zona` por LISA (autocorrelación espacial local).
Referencia viva: `recalc_v3.py`, `build_atlas_caba.py`, `atlas-uraba/src/indicators/*`.

## 4. Arquitectura del plugin

```
tensor-atlas/
├── .claude-plugin/plugin.json        # manifiesto del plugin (name, version, description)
├── README.md                         # qué es, cómo instalar, mapa de skills
├── skills/
│   ├── atlas/SKILL.md                # ORQUESTADOR  /atlas <territorio>
│   ├── atlas-scout/SKILL.md          # Fase 0 · scouting+verificación de fuentes (Workflow)
│   ├── atlas-design/SKILL.md         # Fase 1 · diseño del índice → PLAN_CONSTRUCCION.md
│   ├── atlas-pipeline/SKILL.md       # Fase 2 · pipeline Python → contrato de datos + PMTiles
│   ├── atlas-satelite/SKILL.md       # Fase 3 · GEE por unidad base
│   ├── atlas-web/SKILL.md            # Fase 4 · clonar+rebrandear Nuxt + capa de alias
│   └── atlas-deploy/SKILL.md         # Fase 5 · Vercel + dominio + QA
└── references/                       # compartidas vía ${CLAUDE_PLUGIN_ROOT}
    ├── data-contract.md              # archivos exactos que el front exige (§2)
    ├── scoring-methodology.md        # dimensiones, pesos, normalización, LISA (§3)
    ├── codebase-anatomy.md           # qué editar/renombrar al clonar el Nuxt
    └── recipes/
        ├── _generico.md              # cómo investigar fuentes en cualquier territorio
        ├── colombia.md               # datos.gov.co, IGAC, DANE/CNPV, TerriData, REPS, SIMAT, GEE
        └── argentina.md              # BA Data/DGEyC, INDEC, IGN, USIG, OSRM/Geofabrik, GEE
```

Cada SKILL.md sigue el formato de `superpowers:writing-skills` (frontmatter `name` +
`description` con disparador, cuerpo accionable, gates). Las skills cargan las referencias
compartidas con `${CLAUDE_PLUGIN_ROOT}/references/...`.

## 5. Contrato de cada skill (entradas → salidas)

Todas leen/escriben un **manifiesto de estado** `atlas.manifest.json` en la raíz del
proyecto atlas, para que cada fase sepa qué produjo la anterior y el orquestador pueda
reanudar. Campos: `territorio`, `pais`, `unidad_base`, `agg1`, `agg2`, `dimensiones`,
`fuentes[]`, `fase_actual`, `outputs{}`, `gaps[]`.

1. **atlas-scout** *(Workflow)* — IN: territorio + país. Lanza fan-out de subagentes por
   dominio (territorio base, salud, educación, transporte, ambiente, satélite, riesgo,
   economía), cada uno siembra desde `recipes/<pais>.md`, descubre datasets y los
   **verifica en runtime** (HTTP 200, payload con features reales, CRS, formato, licencia).
   Triangulación adversarial de cada URL ("¿viva y con dato real?"). OUT: `dossiers/*.md`
   por dominio + `gaps[]` en el manifiesto.
2. **atlas-design** — IN: dossiers. Elige unidad base + agg1/agg2, fija dimensiones/pesos,
   marca construible-real vs proxy vs brecha. OUT: `PLAN_CONSTRUCCION.md` + bloque
   `dimensiones`/`unidad_base`/`agg*` en el manifiesto.
3. **atlas-pipeline** — IN: plan + fuentes verificadas. Genera `scripts/build_atlas_<t>.py`:
   download → reproyección CRS → parseo (WKT/otros) → joins espaciales → normalización →
   quintiles/LISA → scores → **emite el contrato** (§2) → tippecanoe → PMTiles. OUT: los
   archivos del contrato en `public/data/` + verificación (scores en [0,1], sin NaN,
   conteos cuadran).
4. **atlas-satelite** — IN: unidad base (geometría). Genera `scripts/gee/compute_satelitales_<t>.py`:
   sube unidades a `ee.FeatureCollection`, computa S2 (NDVI/NDBI), Landsat 8/9 (LST °C),
   VIIRS VNP46A2 (NTL), WorldCover (%verde) con `reduceRegions`, exporta CSV por unidad.
   OUT: `satelite_v3.csv` para merge en el pipeline.
5. **atlas-web** — IN: contrato de datos + manifiesto. Clona `atlas-uraba-web`, **introduce
   la capa de alias de vocabulario** en `stores/atlas.js` (mapa `unidad_base`/`agg1`/`agg2`
   → etiquetas, consumido por componentes/labels — refactor único), edita `DIMENSIONES`,
   `MUNICIPIOS` (centroides agg1), centro del mapa, `nuxt.config.ts`
   (title/description/OG/canonical/dominio/locale), y conecta los datos nuevos.
   OUT: app corriendo (`npm run dev`) pintando el territorio.
6. **atlas-deploy** — IN: app + datos. `npm run generate` + deploy Vercel + dominio
   `*.tensor.lat`, y QA: consistencia de datos + QA visual página-por-página (delega en el
   patrón de `pdf-deliverable`/visual companion). OUT: URL productiva + reporte QA.
7. **atlas** *(orquestador)* — `/atlas <territorio>`. Lee/crea el manifiesto, corre 0→5 con
   gates (no avanza sin que scout y el contrato de datos pasen verificación), e informa
   estado. No llama Workflow directamente; invoca sub-skills y ellas deciden si abren uno.

## 6. Decisiones de diseño tomadas

- **Granularidad:** 7 skills (granular por fase). Cada paso es reusable e invocable solo.
- **Alcance geo:** núcleo agnóstico de territorio + recetas por país (Colombia, Argentina);
  `_generico.md` permite cualquier territorio investigando en runtime.
- **Orquestación:** Workflow multi-agente solo en pasos pesados (scout + verificación de
  datos). El resto secuencial.
- **Home:** repo git nuevo `~/tensor-atlas`, estructura de plugin, publicable a marketplace.
- **Vocabulario de unidad:** **capa de alias configurable** en `stores/atlas.js`. Internamente
  las rutas/variables del Nuxt pueden seguir siendo `manzana/municipio/vereda` (mapeo 1:1),
  pero las etiquetas visibles salen de un mapa de vocabulario por atlas. La primera corrida
  de `atlas-web` hace el refactor único que introduce esa capa; atlas futuros solo definen
  el mapa, sin tocar componentes.

## 7. Riesgos / cuestiones abiertas

- **Refactor de la capa de alias:** introducirla es un cambio transversal en el front.
  Mitigar haciéndolo una sola vez sobre una copia limpia de `atlas-uraba-web` y validando
  que Urabá/CABA siguen renderizando idénticos (regresión visual).
- **CRS y formatos raros** (EPSG:9498, WKT embebido, RAR) varían por país: viven en las
  recetas y en notas de `atlas-pipeline`, no en el núcleo.
- **GEE auth** asumido configurado (cuenta del usuario). `atlas-satelite` verifica auth
  antes de computar.
- **Verificación de skills:** usar `superpowers:writing-skills` para crear y probar cada
  SKILL.md; el orquestador se prueba con un dry-run sobre un territorio chico.

## 8. Plan de construcción del plugin (resumen; el detalle va en el plan)

1. Esqueleto del plugin (`.claude-plugin/plugin.json`, README) + `references/` compartidas
   (data-contract, scoring, codebase-anatomy, recipes CO/AR/genérico) — extraídas de los
   atlas reales, no inventadas.
2. Skills hoja en orden de dependencia: scout → design → pipeline → satelite → web → deploy.
3. Orquestador `atlas` + manifiesto.
4. Verificación: dry-run del orquestador y revisión de cada skill con `writing-skills`.
