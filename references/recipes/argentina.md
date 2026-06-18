---
name: recipe-argentina-caba
description: Úsala cuando se construya o clone un Atlas de bienestar territorial para Argentina con foco en CABA (Ciudad Autónoma de Buenos Aires), o cuando un agente necesite el catálogo de fuentes de datos abiertos argentinas por dominio (BA Data, DGEyC, INDEC, IGN, USIG, RENABAP) con sus formatos, CRS, granularidad y caveats de procesamiento para alimentar pipelines de PMTiles/MapLibre/GEE. Disparadores: "atlas caba", "argentina.tensor.lat", "radios censales caba", "datos abiertos buenos aires", "EPSG:9498 reproyectar".
---

# Receta de datos abiertos — Argentina (foco CABA)

## Overview
CABA es probablemente el caso de datos abiertos más fácil de Argentina: la geometría base y el socioeconómico salen verificados de BA Data a un `curl`. La unidad base del índice es el **radio censal** porque trae población/hogares/NBI embebidos (censo 2010, GeoJSON) sin join, igual que la "manzana censal" del Atlas Urabá. Manzana catastral = vista de zoom alto; barrio (48) / comuna (15) = niveles de agregación para KPIs y rankings.

Todo lo de abajo proviene de `/Users/cristianespinal/atlas-caba-web/docs/PLAN_CONSTRUCCION.md` (catálogo verificado en runtime 2026-06-08). El dossier `dossiers_datos_caba.json` existe en el mismo directorio pero no fue consultado para esta receta; cualquier dato no presente en el PLAN va marcado `[VERIFICAR]`.

## Cuándo usar
- Vas a clonar el Atlas Urabá para CABA (`argentina.tensor.lat`, Nuxt 4 + MapLibre + PMTiles + Nitro/Vercel).
- Necesitas decidir la unidad geográfica base o las fuentes por dominio.
- Vas a escribir el pipeline de descarga/reproyección/tippecanoe o el script GEE.

## Unidad base recomendada: RADIO CENSAL (~3.500)
Trae NBI/población/hogares embebidos (censo 2010) → desbloquea `score_socioeconomico` sin join. **Caveat clave:** la geometría viene como **WKT string en la propiedad `WKT`**, no como geometría GeoJSON nativa → hay que parsear el WKT y reconstruir GeoJSON. Manzana catastral (~14.000) no tiene atributos censales (solo id/nombre/sm). Barrio/Comuna para choropleths robustos (NBI por comuna ya viene en %).

## Catálogo por dominio
URLs verificadas (`Sí` = HTTP 200 confirmado en el PLAN). CRS default WGS84/EPSG:4326 salvo lo marcado. CRS84 = lon/lat sin reproyectar.

### Territorio base
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Radios censales 2010 (geom+NBI) | BA Data / DGEyC | GeoJSON 6,3 MB | radio (~3.500) | geom en **WKT string** (prop `WKT`) | parsear WKT → GeoJSON; unidad base del índice |
| Manzanas catastrales | BA Data / USIG | GeoJSON 7,68 MB | manzana (~14.000) | 4326 | sin atributos censales; capa de zoom alto |
| Barrios (48) | BA Data / GCBA | GeoJSON 763 KB | barrio | 4326 | agregación para filtros/rankings |
| Comunas (15) + perímetro | BA Data / GCBA | GeoJSON 587 KB | comuna | 4326 | perímetro CABA = `ST_Union` de comunas (no hay dataset de límite externo) |
| Límite oficial nacional | IGN GeoServer WFS | WFS | provincia/depto | `[VERIFICAR]` | solo GetCapabilities verificado (parcial); cross-check de baja prioridad |

### Infraestructura / Ordenamiento territorial
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Parcelas catastrales | BA Data / USIG | GeoJSON 438 MB | parcela (~350K) | 4326 | **tippecanoe → PMTiles** con `minzoom` alto antes de servir |
| Código Urbanístico (usos) | BA Data | GeoJSON ~5,8 MB | zona normativa | `[VERIFICAR]` | base para "conflicto de uso" vs RUS |
| RUS — uso real del suelo | BA Data RUS 2022-2024 | ZIP/GeoJSON | parcela/manzana | `[VERIFICAR]` | cruce con Código Urbanístico = capa conflicto de uso |
| API USIG (geocoding) | usig.buenosaires.gob.ar | API REST | dirección | — | 200 JSON; enriquecimiento/normalización de direcciones |

### Medio ambiente
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Espacios verdes públicos | BA Data / Desarrollo Urbano | GeoJSON 18 MB | polígono | `[VERIFICAR]` | m² verde/hab cruzando con población |
| Arbolado público lineal | BA Data / MEPHU | GeoJSON 232 MB | punto (árbol) | `[VERIFICAR]` | **tippecanoe → PMTiles** por tamaño; verdad-terreno de NDVI |
| Espacios verdes catastrales | BA Data | GeoJSON 6 MB | polígono | `[VERIFICAR]` | — |
| Techos verdes | BA Data / APRA | GeoJSON | punto | **EPSG:5347** | **reproyectar 5347→4326** |
| Puntos verdes (reciclaje) | BA Data | GeoJSON | punto | `[VERIFICAR]` | contexto |
| Arbolado en espacios verdes | BA Data | **RAR (no ZIP)** | punto | `[VERIFICAR]` | descomprimir con `unrar`/`unar` antes de `ogr2ogr` |
| ~~Espacio verde privado~~ | BA Data | GeoJSON | — | — | DESCARTAR: responde `features:[]` vacío |

### Salud
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Hospitales | BA Data / Min. Salud | GeoJSON | punto | **EPSG:9498 (Gauss-Krüger)** | **reproyectar 9498→4326** |
| CeSAC (atención primaria) | BA Data / Min. Salud | GeoJSON | punto | CRS84 | listo, sin reproyectar |
| Áreas hospitalarias (catchment) | BA Data / Min. Salud | GeoJSON 492 KB | polígono | `[VERIFICAR]` | distancia al efector del área para `score_salud` |
| Centros médicos barriales | BA Data / Min. Salud | GeoJSON | punto | `[VERIFICAR]` | — |
| Centros de salud privados | BA Data / Min. Salud | CSV | punto | lat/lon | separador `;`, coma decimal |
| Farmacias | BA Data / Min. Salud | GeoJSON 512 KB | punto | `[VERIFICAR]` | — |
| Geriátricos | BA Data / Min. Salud | GeoJSON | punto | `[VERIFICAR]` | equipamiento |

### Educación
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Establecimientos educativos | BA Data / Min. Educación | GeoJSON | punto | **EPSG:9498** | **reproyectar 9498→4326** |
| Universidades | BA Data / Min. Educación | GeoJSON | punto | CRS84 | listo |
| Padrón establecimientos | BA Data / Min. Educación | CSV | establecimiento | — | separador `;`; enriquece score_educacion |
| Escuelas verdes | BA Data / Min. Educación | GeoJSON (wgs84) | punto | 4326 | cruza Educación + Ambiente |
| Asistencia escolar (serie) | BA Data / DGEyC | CSV | ciudad (proxy) | — | proxy socioeconómico, no es a radio |

### Demografía / Pobreza
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Info censal por radio 2010 | BA Data / DGEyC | GeoJSON 6,3 MB / CSV | radio | geom en **WKT** | misma fuente base; CSV requiere join al GeoJSON |
| NBI por comuna (%) | BA Data / IVC | CSV 144 B | comuna | — | ya viene en %, panel comunal |
| Barrios Populares CABA | BA Data / Hábitat | GeoJSON 251 KB | polígono | `[VERIFICAR]` | overlay hábitat/vulnerabilidad |
| RENABAP nacional | infraestructura.gob.ar (SSISU) | GeoJSON 9,4 MB / CSV | polígono | `[VERIFICAR]` | filtrar a CABA |
| INDEC WFS radios censales | INDEC GeoNode | WFS | radio (nacional) | EPSG:4326 (srsName) | `typeName=geonode:radios_censales2`; CQL a CABA; alt. programática |
| Censo 2022 a radio | INDEC REDATAM + cartografía | — | radio | `[VERIFICAR]` | NO descarga directa; REDATAM 200 pero no archivo único; resolver en navegador (fase 3). Usar **censo 2010** para el índice |

### Transporte
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Estaciones + líneas de Subte | BA Data / SBASE | GeoJSON | punto/línea | `[VERIFICAR]` | — |
| Paradas de colectivo (~9.000) | BA Data | GeoJSON 3,3 MB | punto | `[VERIFICAR]` | conteo por radio → score_accesibilidad |
| Recorridos de colectivo | BA Data | GeoJSON 21,8 MB | línea | `[VERIFICAR]` | **tippecanoe → PMTiles** |
| Metrobús (estaciones + recorridos) | BA Data | GeoJSON | punto/línea | `[VERIFICAR]` | BRT |
| Estaciones de FFCC | BA Data | GeoJSON | punto | `[VERIFICAR]` | — |
| Ecobici (estaciones) | BA Data | CSV | punto | lat/lon | micromovilidad |
| Ciclovías | BA Data | GeoJSON 960 KB | línea | `[VERIFICAR]` | infra ciclista |
| Grafo ruteable (isócronas) | OSM Geofabrik | OSM PBF 420 MB | red | — | `argentina-latest.osm.pbf`; recortar bbox CABA `-58.53,-34.71,-58.33,-34.53`; OSRM perfiles foot/bike |
| API tiempo real GTFS-RT/GBFS | apitransporte GCBA | API | vehículo/estación | — | 401: requiere `client_id/secret`; GTFS estático para frecuencias |
| Matriz OD SUBE / Viajes-etapas AMBA | BA Data | PDF (metadatos) | viaje/etapa | — | solo metadatos verificados; URL del CSV `[VERIFICAR]` |

### Riesgo territorial
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Barrios populares (hábitat) | BA Data / RENABAP nacional | GeoJSON/CSV | polígono | `[VERIFICAR]` | vulnerabilidad de hábitat |
| LST (isla de calor) | Landsat 8/9 (GEE) | raster | radio | EPSG asset GEE | ver Satélite |

### Satélite (GEE)
Colecciones confirmadas; se computan sobre el polígono CABA con `reduceRegions` por radio → CSV → join → PMTiles. Empezar por **LST** (mayor impacto en CABA).
| Capa | Colección GEE | Resolución | Granularidad | Caveat |
|---|---|---|---|---|
| NDVI / NDBI | `COPERNICUS/S2_SR_HARMONIZED` | 10 m | radio/manzana | score_ambiental |
| LST (isla de calor) | `LANDSAT/LC09/C02/T1_L2` + `LANDSAT/LC08/C02/T1_L2` | 30/100 m | radio | banda térmica `ST_B10`; ventana verano; °C |
| VIIRS luces nocturnas | `NASA/VIIRS/002/VNP46A2` (Black Marble) | ~500 m | radio (proxy) | proxy socioeconómico/seguridad |
| Cobertura del suelo | `ESA/WorldCover/v200` | 10 m | %clase/radio | %verde/impermeable |
| Cobertura dinámica | `GOOGLE/DYNAMICWORLD/V1` | 10 m | radio (tendencia) | versión dinámica de WorldCover |

### Economía / Seguridad
| Fuente | Proveedor | Formato | Granularidad | CRS | Caveat |
|---|---|---|---|---|---|
| Valor del suelo (oferta) | BA Data Terrenos Valor Oferta 2020 | CSV/GeoJSON | parcela/zona | `[VERIFICAR]` | CDN directo; doc OpenAPI catastro NO confirmada |
| Actividad económica (proxy) | VIIRS VNP46A2 (GEE) | GEE | radio | — | proxy |
| Iluminación nocturna (proxy seguridad) | VIIRS (GEE) | GEE | radio | — | proxy de seguridad |
| Delito georreferenciado a radio | — | — | — | — | **BRECHA**: no verificado en abierto a grano fino; usar VIIRS como proxy explícito + investigar mapa del delito GCBA |

## Procedimiento (pipeline FASE 1, demo en horas)
1. Descargar radios censales 2010 GeoJSON → `data/raw/`.
2. **Parsear WKT** (prop `WKT`) → reconstruir GeoJSON nativo.
3. Calcular `score_socioeconomico = 1 − norm(hogares_NBI/hogares_total)`; quintiles.
4. Comunas → `ST_Union` = perímetro/máscara CABA. Barrios → filtros/rankings.
5. Reproyectar lo que esté en 9498 o 5347: `ogr2ogr -s_srs EPSG:9498 -t_srs EPSG:4326 out.geojson in.geojson` (hospitales, establecimientos educativos = 9498; techos verdes = 5347).
6. Pesados a tiles: `tippecanoe` → `public/data/atlas.pmtiles` (parcelas 438 MB, arbolado 232 MB, recorridos colectivo 21,8 MB).
7. `npm run dev` → el front de Urabá pinta CABA.

## Contrato de salida
- `public/data/atlas.pmtiles` (radios con `atlas_score` + sub-scores) + GeoJSON ligeros.
- CSV de satelitales por radio (GEE) joinados a la geometría.
- Capas vector overlay (salud, educación, transporte, ambiente) reproyectadas a 4326.

## Errores comunes
- Tratar la geometría de radios como GeoJSON nativo: viene en **WKT string**, hay que parsear.
- Olvidar reproyectar hospitales/educativos (**EPSG:9498**) y techos verdes (**EPSG:5347**): caen fuera de CABA.
- Descomprimir "arbolado en espacios verdes" como ZIP: es **RAR**.
- Servir parcelas/arbolado/recorridos crudos sin **tippecanoe**: rompe el navegador.
- Incluir "espacio verde privado" (`features:[]` vacío) o asumir delito a radio en abierto.
- Asumir CRS no marcados arriba sin verificar el archivo real (`[VERIFICAR]`).

## Notas de formato de skill
Las URLs exactas verificadas viven en la "Tabla maestra de descargas" del PLAN (`/Users/cristianespinal/atlas-caba-web/docs/PLAN_CONSTRUCCION.md` §7). No se replican aquí para evitar copia divergente; ese PLAN es la fuente de verdad de las URLs (BA Data usa dos hosts: `cdn.buenosaires.gob.ar/datosabiertos/...` y `data.buenosaires.gob.ar/dataset/.../resource/.../download`).
