---
name: atlas
description: Use when the user wants to build a complete Tensor Atlas for a territory end to end (e.g. "/atlas <territorio>", "armá el atlas de X", "construye un atlas de bienestar territorial"), or to resume an in-progress atlas build from its manifest. Entry point of the tensor-atlas plugin.
---

# atlas — Orquestador del plugin tensor-atlas

## Overview
`/atlas <territorio>` construye un Atlas de Bienestar Humano Territorial (estilo `uraba.tensor.lat`, `caba.tensor.lat`) corriendo las fases 0→5 **en orden, con gates entre fases**. Es el único punto de entrada del plugin. NO ejecuta lógica de datos ni de front por sí mismo: lee/escribe un manifiesto de estado e **invoca cada sub-skill** (`tensor-atlas:atlas-scout`, `-design`, `-pipeline`, `-satelite`, `-web`, `-deploy`); cada sub-skill decide si abre un Workflow multi-agente. El orquestador NUNCA llama Workflow directamente.

## Cuándo usar
- "Construí/armá/empieza el atlas de <territorio>" (end to end).
- "Reanudá el atlas de <territorio>" / "¿en qué fase quedó?" → reanuda desde `fase_actual`.
- El usuario nombra un territorio LatAm y quiere el sistema completo (pipeline de datos Python + front Nuxt desplegado).
- NO usar para una sola fase: si el usuario solo quiere scoutear fuentes, diseñar el índice, correr el pipeline, computar satélite, clonar el front o desplegar, invocá directamente esa sub-skill.

## Manifiesto de estado: `atlas.manifest.json`
Vive en la **raíz del proyecto atlas** (`~/atlas-<territorio>-web/atlas.manifest.json` o donde el usuario lo ubique). Es el contrato de estado entre fases y la fuente de verdad para reanudar.

Campos (definidos en el design spec, §5):
```json
{
  "territorio": "",          // p.ej. "Urabá", "CABA"
  "pais": "",                // selecciona la receta: references/recipes/<pais>.md
  "unidad_base": "",         // = vocabulario "manzana" del contrato (radio censal, manzana CNPV…)
  "agg1": "",                // = "municipio" (comuna, municipio…)
  "agg2": "",                // = "vereda" (barrio, vereda…)
  "dimensiones": [],         // [{key,label,color,estado}] estado: real|proxy|brecha
  "fuentes": [],             // [{dominio,url,formato,crs,licencia,verificada}]
  "fase_actual": "scout",    // scout|design|pipeline|satelite|web|deploy|done
  "outputs": {},             // rutas producidas por cada fase (dossiers, PLAN, contrato, csv, url)
  "gaps": []                 // brechas (datos por derecho de petición, dims sin dato real)
}
```
Al arrancar: si existe el manifiesto, **léelo y reanuda** desde `fase_actual`; si no, créalo con `territorio`/`pais` del argumento y `fase_actual: "scout"`.

## Procedimiento — fases y gates
Corre en este orden. **No avances de fase hasta que el gate pase.** Tras cada fase, persiste lo producido en `outputs{}` y avanza `fase_actual` en el manifiesto.

| # | Fase | Sub-skill a invocar (REQUIRED) | Gate para avanzar |
|---|------|--------------------------------|-------------------|
| 0 | scout | `tensor-atlas:atlas-scout` | **Fuentes core verificadas** en runtime (HTTP 200, payload con features reales, CRS/formato/licencia anotados) para los dominios mínimos (territorio base + ≥1 de salud/educación/transporte). Brechas → `gaps[]`, no bloquean si hay núcleo. |
| 1 | design | `tensor-atlas:atlas-design` | **`PLAN_CONSTRUCCION.md` aprobado** (unidad_base/agg1/agg2 fijados, dimensiones marcadas real\|proxy\|brecha, pesos definidos). Confirmar con el usuario antes de seguir. |
| 2 | pipeline | `tensor-atlas:atlas-pipeline` | **Contrato de datos pasa verificación**: emite `atlas.geojson`, `municipios.geojson`, `veredas.geojson`, `atlas_stats.json`, `atlas_stats_v3.json`, `comunas_centroides.json`, `atlas_breaks.json`, `gap_analysis.json` en `public/data/`; scores en [0,1], sin NaN, conteos cuadran, claves de agg coinciden. Ver `${CLAUDE_PLUGIN_ROOT}/references/data-contract.md`. |
| 3 | satelite | `tensor-atlas:atlas-satelite` *(opcional)* | Solo si el PLAN usa dims satelitales (NDVI/NDBI/LST/VIIRS via GEE `reduceRegions`). Gate: `satelite_v3.csv` por unidad_base, GEE auth verificada. Si las dims ambiental/seguridad no son satelitales, **saltar** y registrar en `outputs{}`. |
| 4 | web | `tensor-atlas:atlas-web` | App Nuxt corriendo (`npm run dev`) pintando el territorio: clon rebrandeado, capa de alias de vocabulario en `stores/atlas.js`, `DIMENSIONES`/`MUNICIPIOS` (regla dura: `MUNICIPIOS[].nombre == municipio`), datos del contrato conectados. |
| 5 | deploy | `tensor-atlas:atlas-deploy` | `npm run generate` OK + deploy Vercel + dominio `*.tensor.lat` + QA (consistencia de datos + QA visual página-por-página). Gate final: URL productiva responde y QA sin bloqueantes → `fase_actual: "done"`. |

Regla de oro de los gates: si un gate **falla**, NO avances. Reporta qué falló, deja `fase_actual` en la fase actual, registra el problema en `gaps[]` u `outputs{}` y devuelve el control al usuario.

## Cómo reanudar
1. Lee `atlas.manifest.json` de la raíz del proyecto.
2. Salta a `fase_actual` y **re-verifica el gate de la fase anterior** antes de continuar (los outputs pueden haber cambiado o quedado incompletos).
3. Continúa la secuencia 0→5 desde ahí. Si `fase_actual: "done"`, solo reporta estado (no reconstruyas).

## Invocación de sub-skills (regla crítica)
- Invoca cada fase con la skill nombrada: **REQUIRED: usa la skill `tensor-atlas:atlas-scout`** (y análogas). Nunca por @ruta ni leyendo su SKILL.md a mano.
- El orquestador **no** abre Workflow. Las fases pesadas (scout y la verificación de datos del pipeline) son las que internamente lanzan el fan-out multi-agente; el orquestador solo las llama y lee su salida.
- Las referencias compartidas viven en `${CLAUDE_PLUGIN_ROOT}/references/` (`data-contract.md`, `scoring-methodology.md`, `codebase-anatomy.md`, `recipes/<pais>.md`, `recipes/_generico.md`). Enlázalas por esa ruta; no las cargues con @.

## Contrato de salida
- `atlas.manifest.json` actualizado con `fase_actual` y `outputs{}` reales de cada fase.
- Al terminar: URL productiva `*.tensor.lat`, reporte QA, e inventario de `gaps[]` pendientes (incluidos los que requieren derecho de petición — se documentan, no se resuelven).

## Errores comunes
- **Saltarse un gate.** No avances "porque parece listo": verifica el contrato/PLAN/fuentes reales.
- **Llamar Workflow desde el orquestador.** No. Invoca la sub-skill; ella decide.
- **Inventar la receta de país.** Si `pais` no tiene receta dedicada, usa `recipes/_generico.md` (investiga fuentes en runtime); no asumas datasets.
- **Romper la regla dura del front.** `MUNICIPIOS[].nombre` debe coincidir EXACTAMENTE con la prop `municipio` de `atlas.geojson`/`municipios.geojson`, o la cámara no vuela y los stats no resuelven.
- **Perder el manifiesto.** Siempre persistir tras cada fase; sin él no hay reanudación.
- **[VERIFICAR] sub-skills en disco.** Las hojas (`atlas-scout`…`atlas-deploy`) son hermanas en `skills/` según el design spec (`docs/superpowers/specs/2026-06-18-tensor-atlas-plugin-design.md`). Si alguna aún no existe al invocarla, detente y repórtalo; no improvises su trabajo dentro del orquestador.
