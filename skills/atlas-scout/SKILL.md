---
name: atlas-scout
description: "Use when starting a new Tensor Atlas for a territory and you need to discover and verify open-data sources before designing the index. Triggers: 'arrancar atlas para <territorio>', 'qué datos abiertos hay para X', 'fase 0 del atlas', 'verificar fuentes', 'dossiers por dominio', cuando aún no existen dossiers/ ni el bloque fuentes[]/gaps[] del manifiesto. NO la uses para diseñar el índice (eso es atlas-design) ni para emitir el contrato de datos (eso es atlas-pipeline)."
---

## Overview

Fase 0 de un Tensor Atlas: dado `territorio` + `pais`, **descubrir y VERIFICAR en runtime** las fuentes de datos abiertos por dominio, sembrando desde la receta-país, y producir un dossier por dominio + los bloques `fuentes[]` y `gaps[]` del `atlas.manifest.json`. Principio innegociable: nada se da por vivo sin un `curl` que lo pruebe; lo no verificable es brecha (`gaps[]`), no dato. Esto alimenta a atlas-design (fija `unidad_base`, `dimensiones`, pesos) y luego a atlas-pipeline (emite el contrato de `${CLAUDE_PLUGIN_ROOT}/references/data-contract.md`).

## Cuándo usar

- Inicias un Atlas para un territorio nuevo y no existen `dossiers/<dominio>.md`.
- El manifiesto no tiene aún `fuentes[]` (verificadas) ni `gaps[]`.
- Necesitas saber qué datos abiertos reales existen y cuáles están vivos por dominio antes de diseñar el índice o escribir el pipeline.

## Procedimiento (orquestación pesada: fan-out de subagentes)

Esta skill **lanza un Workflow / fan-out**, un subagente por dominio, en paralelo (sin estado compartido). Dominios: **territorio base, salud, educación, transporte, ambiente, satélite, riesgo, economía** (mapea al checklist de `${CLAUDE_PLUGIN_ROOT}/references/recipes/_generico.md`; añade socioeconómico/NBI dentro de territorio base).

1. **Sembrar desde la receta-país.** Cada subagente abre `${CLAUDE_PLUGIN_ROOT}/references/recipes/<pais>.md` (p.ej. `colombia.md`, `argentina.md`) por su dominio. Si no existe receta-país, usa `${CLAUDE_PLUGIN_ROOT}/references/recipes/_generico.md` (respaldo agnóstico + fuentes universales: GEE, OSM/Geofabrik, límites administrativos). La receta da candidatos (proveedor, formato, granularidad, acceso), NO verdad: hay que verificar.
2. **Verificar cada URL en runtime** (método de `_generico.md` §"Cómo verificar"):
   - **HTTP vivo:** `curl -sI <url>` → `200` (o `302→200`; `401` = existe pero requiere key; `404`/SSL roto = brecha).
   - **Features reales:** descargar un trozo y confirmar que `features` NO está vacío (`features:[]` = descartar, como "espacio verde privado" en CABA).
   - **CRS:** detectar proyección; el contrato exige **EPSG:4326**. Reproyectar locales: `ogr2ogr -s_srs EPSG:<src> -t_srs EPSG:4326 out.geojson in.geojson` (CABA: hospitales/educativos EPSG:9498, techos verdes EPSG:5347).
   - **Formato real:** GeoJSON nativo vs **WKT en una propiedad** (radios CABA traen geom en prop `WKT`); `.zip` que es **RAR**; separador/decimal en CSV.
   - **Licencia:** abierta reutilizable (CC-BY o similar); anotarla.
3. **Triangulación adversarial** ("¿viva y con dato real?"): cada fuente candidata enfrenta a escépticos independientes que intentan refutar que esté viva y con dato real. Marca cada una `verificado=true/false` con verdicto por mayoría. (Patrón: skill `propuestas-publicas:triangulacion-adversarial`.)
4. **Elegir unidad base y agregaciones.** Unidad base = la más fina que trae población/NBI **embebidos sin join** (Urabá: manzana censal CNPV 2018; CABA: radio censal 2010). Agg1 = administrativa intermedia (`municipios.geojson`/`municipio`: municipio en Urabá, comuna en CABA); Agg2 = contexto (`veredas.geojson`: vereda/barrio). Perímetro = `ST_Union`/dissolve de agg1, no dataset externo.
5. **Mapear cada dominio a dimensión/sub-score** según la tabla de `_generico.md` (salud→`score_salud`, educación→`score_educacion`, transporte→`score_via`, NDVI/verde→`score_ndvi`/`score_ndbi`, LST→`score_calor`, VIIRS→`score_seguridad`/`score_viirs` proxy, NBI→`score_nbi`). Pesos de referencia: accesibilidad 0.40 · ambiental 0.25 · socioeconómico 0.25 · seguridad 0.20 (normalizar sobre dimensiones con dato real; no inventar).

## Contrato de salida

- **`dossiers/<dominio>.md`** por dominio: fuentes `verificado=true` con `url`, `formato`, `granularidad`, `crs`, `licencia`, `mapea_a` (dimensión/sub-score); unidad base propuesta + agg1/agg2; y `gaps[]` con proxy aceptable si existe.
- **Bloque `fuentes[]` y `gaps[]` en `atlas.manifest.json`**: consolidado de los dossiers. `fuentes[]` = todo lo verificado; `gaps[]` = lo que cuelga, requiere login/derecho de petición, o no se pudo probar (con el proxy que lo cubre, p.ej. VIIRS para seguridad).

## Errores comunes

- Confiar en una URL de catálogo sin `curl`: muchas dan `404`, `features:[]` o SSL roto.
- Elegir como unidad base la capa más fina aunque NO traiga censo: fuerza joins frágiles y pierde `score_socioeconomico`.
- Olvidar reproyectar capas en CRS local (9498/5347/Gauss-Krüger): desplazan toda la geometría; el contrato exige EPSG:4326.
- Asumir GeoJSON nativo cuando la geometría viene como **WKT** en una propiedad, o `.zip` que es **RAR**.
- Inventar `score_seguridad` con dato directo: el delito a unidad fina casi siempre es brecha → usar VIIRS como **proxy explícito**.
- Colombia: asumir que catastro IGAC nacional cubre Antioquia (es descentralizado); esperar coordenadas en REPS/SIMAT (hay que geocodificar por dirección); olvidar `zfill(5)` en DIVIPOLA.
- Sumar pesos sin normalizar cuando falta una dimensión.
