# Receta de fuentes de datos abiertos — Colombia (foco Urabá/Antioquia)

> Referencia compartida para construir un Atlas territorial por dominios.
> Anclada **solo** en `atlas-uraba-web/ROADMAP.md` y los ingestores reales de
> `atlas-uraba/src/ingestion/`. Lo no verificable contra esos archivos va marcado `[VERIFICAR]`.
> Enlázala desde una skill por ruta `${CLAUDE_PLUGIN_ROOT}/references/recipes/colombia.md` (NO uses `@`).

## Unidad base y niveles de agregación

- **Unidad base recomendada: manzana censal CNPV 2018** (manzana donde hay manzana). El núcleo del atlas son **7.028 manzanas** de Urabá con índice v1 (5 dimensiones) + v2 enriquecido.
- **Niveles de agregación:** **municipio** (8 de Urabá vía DIVIPOLA) y **vereda** (611 veredas levantadas). Regla de oro del ROADMAP: *granularidad honesta* — manzana donde existe, municipio donde el dato es solo municipal.
- **Clave de cruce territorial:** código **DIVIPOLA** de 5 dígitos (string, `zfill(5)`). Urabá = `["05045","05147","05120","05837","05544","05665","05659","05051"]` (Apartadó, Chigorodó, Carepa, Turbo, Necoclí, San Pedro de Urabá, San Juan de Urabá, Arboletes). Antioquia = depto `05`. Extensible a otras subregiones cambiando la lista DIVIPOLA.
- **Geometría manzana:** MGN/CNPV provee la columna estandarizada `cod_manzana` (ingestor `dane_loader.py`).
- **Principio:** cero geometrías inventadas; marcar proxy vs real; citar fuente por capa.

## Estado de acceso (leyenda)

- **Descarga directa** — archivo/zip/CSV/WFS sin autenticación.
- **API** — endpoint programático (puede requerir registro de llave).
- **Gestión / derecho de petición** — requiere solicitud institucional (alcaldía, ministerio, ANT).
- **Cómputo / manual** — no es una fuente sino un derivado a calcular (OSRM, geocodificación, recortes GEE).

---

## Territorio base (demografía espacial)

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| MGN 2024 manzanas censales | DANE | Shapefile (ArcGIS item `da6856c2040c4098831605384715d35b`) | Manzana | Descarga directa | Geometría base `cod_manzana` (7.028 manzanas) |
| CNPV 2018 detallado | DANE (geovisor + microdatos catalog 643) | Geovisor / microdatos | Manzana | Descarga directa | Índice v1/v2 (5 dimensiones de bienestar por manzana) |
| OSM equipamientos/red/edificaciones | OpenStreetMap (vía OSMnx) | GeoDataFrame | Punto/línea/polígono | Descarga directa | Equipamientos fallback (health, school, police, parks, shop) |

## Catastro / Ordenamiento

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| Base Catastral IGAC 2026 | IGAC (datos.icde.gov.co `2e26ee016ce14c359a5037231da25a86`) | Shapefile/zip (ArcGIS REST) | Manzana/terreno/construcción | Descarga directa | `U_MANZANA`, `U_TERRENO`, `U_CONSTRUCCION`, `R_VEREDA` — área construida, nº predios, clasificación suelo. **ADVERTENCIA:** Antioquia es catastro descentralizado y NO viene en la descarga nacional IGAC; Urabá requiere catastro de Gobernación de Antioquia / municipio |
| `U_CONSTRUCCION` (densidad) | IGAC (deriva de catastro) | Shapefile | Manzana | Descarga directa (con catastro) | Área construida real (`AREA_CONST`, `PISOS`) → densidad por manzana |
| Estrato socioeconómico | Alcaldías Apartadó/Turbo/Chigorodó (DANE) | `[VERIFICAR]` | Manzana/cuadra | Gestión / derecho de petición | Indicador urbano #1 para diferenciar cuadra a cuadra (APIs estrato dan 404) |
| POT/PBOT/EOT | SiPOT MVCT + Sec. Planeación municipal | `[VERIFICAR]` (SiPOT sin API) | Polígono uso del suelo | Gestión municipal | Uso del suelo oficial (residencial/comercial/industrial/protección) |

## Ambiente

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| Deforestación SMByC | IDEAM / SMByC | Polígono | Subregión | Descarga directa `[VERIFICAR]` | Deforestación (621 Urabá + 7.930 Chocó) |
| Manglares GMW v3 | Global Mangrove Watch | Polígono (satélite) | Polígono | Descarga directa | Cobertura de manglar (1.468 polígonos) |
| Concesiones de agua | CORPOURABÁ | Tabla/geo | Punto/concesión | Gestión (enviado) `[VERIFICAR]` | 371 concesiones de agua |
| RUNAP / áreas protegidas | Parques Nacionales (`parquesnacionales.gov.co` WFS) | WFS | Polígono | API (WFS) | Todas las AP + DRMI Golfo (reemplaza SINAP 3) |
| Frontera agrícola UPRA | UPRA / SIPRA | Polígono | Polígono | Descarga directa (ya en base) | Suelo agrícola activo vs vocación; `conflicto_bananero` |

## Salud

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| REPS prestadores | MinSalud (`datos.gov.co/resource/c36g-9fc2.csv`) | CSV (Socrata) | Punto (geocodificado) | API / descarga directa | 339 prestadores Urabá; `NivelAtencion` 1/2/3, `ClasePrestador`. **No trae coordenadas** → geocodificado por dirección |
| Calidad agua potable (IRCA) | SIVICAP / INS | Reporte anual `[VERIFICAR]` | Municipio | Descarga directa `[VERIFICAR]` | Riesgo sanitario del agua |

## Educación

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| SIMAT matrícula | MinEducación (`datos.gov.co/api/views/upkm-vdjb/rows.csv`) | CSV (Socrata) | Punto (geocodificado) | API / descarga directa | 180 establecimientos Urabá; `zona`, `niveles`. **No trae coordenadas ni matrícula por nivel** (solo `matricula_Contratada` como proxy) → geocodificado por dirección |

## Demografía / Pobreza

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| TerriData | DNP (API, 800 indicadores) | JSON (API) | Municipio | API | Finanzas, gobierno, educación municipal |
| Proyecciones población 2018–2035 | DANE | Tabla | Municipio | Descarga directa | Denominador de todas las tasas per cápita |
| Resguardos indígenas oficial | ANT (shapefile) | Shapefile | Polígono | Descarga directa | Territorios étnicos con resolución legal (reemplaza OSM 35) |
| Consejos comunitarios Ley 70 | ANT | Shapefile `[VERIFICAR]` | Polígono | Descarga directa | Territorios colectivos afro — diferenciador de Urabá |
| SIEDCO criminalidad | Policía Nacional / DIJIN (datos.gov.co) | CSV | Municipio / punto | Descarga directa `[VERIFICAR]` | Índice de seguridad (grave/leve × persona/propiedad). Portal SIEDCO directo es acceso institucional |

## Transporte

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| Red vial nacional | INVÍAS (datos.gov.co shapefile) | Shapefile | Línea | Descarga directa | Estado pavimento + categoría (reemplaza OSM fallback; red vial 344) |
| Isócronas reales | OSRM API (sin key) | Cómputo | Manzana→destino | Cómputo / manual | `índice_aislamiento` en minutos reales a IPS/colegio/cabecera/Puerto/Medellín |
| Parque automotor | RUNT / Mintransporte | Tabla | Municipio | Descarga directa | Motorización por municipio |
| Concesiones viales ANI (Mar 1/2, Toyo) | ANI | Reportes | Proyecto | Cómputo / manual | Proyectos 4G/5G que transforman accesibilidad |
| Transporte fluvial (León/Atrato) | Gobernación / campo | — | Eje | Gestión / campo | Eje real sin datos digitales — vacío conocido |

## Riesgo

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| Inundación por cuencas | IDEAM | Polígono | Cuenca | Descarga directa `[VERIFICAR]` | Amenaza de inundación (18 cuencas Urabá) |

## Satélite (GEE — reemplaza proxies)

> GEE ya autenticado en el repo (`gee_downloader.py`). Recorte por DIVIPOLA Urabá, periodos climáticos Colombia: cálido (dic–mar = meses 12,1,2,3) y frío (may–oct).

| Fuente | Colección GEE | Banda/índice | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| NDVI | `COPERNICUS/S2_SR_HARMONIZED` (Sentinel-2) | NDVI (B8/B4) | Píxel→manzana | API (GEE) | Vegetación; base de cobertura |
| LST | `LANDSAT/LC08/C02/T1_L2` (Landsat 8/9) | `ST_B10` (escala `0.00341802`, offset `149.0`, K→°C) | Píxel→manzana | API (GEE) | Isla de calor por manzana (indicador NUEVO) |
| NDBI | Sentinel-2 `[VERIFICAR colección exacta]` | NDBI | Píxel→manzana | API (GEE) | Impermeabilización real (reemplaza proxy GHSL/área) |
| Luminosidad nocturna | VIIRS Black Marble `[VERIFICAR id]` | radiancia | Píxel→manzana | API (GEE) | Actividad económica + cobertura eléctrica de facto (reemplaza `proxy_luminosidad`) |
| Cobertura del suelo 10m | ESA WorldCover / Dynamic World `[VERIFICAR id]` | clase | Píxel | API (GEE) | Cobertura del suelo independiente de OSM |

---

## Infraestructura / servicios (FRENTE 2 del ROADMAP)

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| Cobertura servicios públicos | SUI Superservicios | API c/registro | Municipio | API (registro) | % acueducto/alcantarillado/energía/aseo |
| Cobertura eléctrica / ZNI | UPME ICEE + IPSE | Excel/API | Municipio/ZNI | API / descarga | Zonas No Interconectadas — déficit energético |
| Banda ancha fija | MinTIC SIUST / Atlas TIC | API | Municipio | API | Brecha digital — velocidad media |

## Agro / cadena de valor (FRENTE 3)

| Fuente | Proveedor | Formato | Granularidad | Acceso | Indicador que desbloquea |
|---|---|---|---|---|---|
| EVA producción agrícola | MADR (Socrata `uejq-wxrr`) | CSV (API) | Municipio/cultivo | API | Área/producción/rendimiento por cultivo y año |
| Precios mayoristas SIPSA | DANE (Socrata) | CSV (API) | Producto | API | Precio banano/plátano — margen del productor |
| Exportaciones FOB | DANE-DIAN EXPO (catálogo 472) | Tabla | Producto/país | Descarga directa | Banano Antioquia: toneladas + país + USD |
| Registros sanitarios / Foc TR4 | ICA | — | Predio | Gestión / derecho de petición | Sanidad vegetal — riesgo biológico geolocalizado |

---

## Errores comunes

- Asumir catastro IGAC nacional cubre Antioquia: **no lo hace** (catastro descentralizado). Para Urabá ir a Gobernación de Antioquia / municipio.
- Esperar coordenadas en REPS o SIMAT: **no las traen** — hay que geocodificar por dirección (`reps_uraba_geo.gpkg`, `simat_uraba_geo.gpkg`).
- Tratar `impermeabilizacion`, `proxy_luminosidad`, `score_via`, `clasificacion_suelo` como dato real: son **proxies** pendientes de reemplazo (NDBI, VIIRS, OSRM, catastro IGAC).
- Usar "distancia euclidiana" como aislamiento: el ROADMAP exige **isócronas OSRM reales** (minutos), no distancia.
- Olvidar `zfill(5)` en DIVIPOLA al cruzar tablas → joins rotos por códigos `05045` vs `5045`.
- Buscar API de estrato o POT: estrato da 404 y SiPOT no tiene API → ambos van por gestión institucional.
