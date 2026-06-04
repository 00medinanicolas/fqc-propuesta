# Dimensionamiento técnico — Fase 2 (Mapa de Argentina + extensión LATAM)

Documento de trabajo para el `index.html` de la propuesta. Reúne los números reales para presupuestar la Fase 2 (extensión a Argentina completa + Panamá, México, España, Colombia, Uruguay y otros).

Fecha de análisis: **2026-06-04**
Referencias cruzadas: ver `memory/project_fqc_pre_piloto_roadmap.md`, `memory/reference_overture_fsq.md`, `memory/reference_data_sources_free.md`.

---

## TL;DR

1. **Infra: barata.** VPS Don Web tipo Access Power resuelve el stack completo (Postgres+PostGIS+n8n+API+Nginx). USD 10–40/mes según fase.
2. **Tiempos de carga: bajos.** Bulk LATAM 6–8 hs (una vez), refresh mensual 3–4 hs (n8n cron de madrugada), queries usuario < 500 ms.
3. **Datos: disponibles y gratis.** OSM + Overture cubren AR + 5 países sin licencia paga.
4. **Lo caro son las horas humanas** de adaptar locators de marcas y curar mapeos de categorías por país.

---

## 1. Referencia viva: Access Power en Don Web

VPS productivo desde 2026-04-15, sin caídas. Estos son los recursos que ya soportan un workload mayor al que FQC requiere:

| Recurso | Valor | Carga real que aguanta hoy |
|---|---|---|
| vCPU | 4 | PostgreSQL 16 + N8N Docker + Nginx + CRM en simultáneo |
| RAM | 8 GB | 23.346 threads + 35.556 messages + 450 gaps + workflows N8N |
| SSD | 35 GB | DB completa + N8N + CRM + logs (con margen) |
| Transferencia | 2 TB/mes | Equipo Access Power usando el CRM productivo todos los días |
| OS | Ubuntu 24.04 LTS | — |
| Stack | PG 16 + N8N 2.16.1 (Docker) + Nginx + Certbot | Mismo patrón a replicar para FQC |
| Backup | Premium Diario incluido | + cron `pg_dump` 04:00 ART, 14d retention |
| Hardening | iptables DROP 5432 público / SSH solo con key / rate limit login | Pattern reutilizable |
| Costo | ~USD 5–12/mes (plan base, anual con 3 meses gratis) | — |

**Conclusión clave.** Access Power maneja mucho más workload de concurrencia humana (CRM en vivo, 5–10 agentes simultáneos, ingesta de mails, OCR + Sonnet vision) que el que necesita FQC. Lo que cambia para FQC no es la carga humana, es el **volumen de datos en reposo**.

---

## 2. Volumen real de datos por país

### 2.a Tamaños OSM .pbf (Geofabrik, snapshot 2026-06-03)

| País | OSM PBF | Overture places estimados | Filtrado a rubros FQC |
|---|---|---|---|
| Argentina | 400 MB | ~1.150.000 | ~280.000 |
| México | 598 MB | ~3.250.000 | ~810.000 |
| España | 1.4 GB | ~1.900.000 | ~370.000 |
| Colombia | 300 MB | ~1.300.000 | ~325.000 |
| Panamá | 30.5 MB | ~110.000 | ~28.000 |
| Uruguay | 53 MB | ~85.000 | ~21.000 |
| **TOTAL** | **~2.8 GB PBF** | **~7.8M places raw** | **~1.85M POIs filtrados FQC** |

Cómo se calculan los estimados: con datos reales del piloto.
- Mendoza: 21.717 places Overture / ~1M hab = **0,022 places/hab**
- Bahía Blanca: 9.077 / 336.574 = **0,027 places/hab**

Después de filtrar al diccionario de rubros FQC (gastro / retail / fitness / beauty / atractores) queda ~25–30 % del raw Overture.

### 2.b Cuánto pesa eso adentro de Postgres + PostGIS

| Componente | Tamaño |
|---|---|
| Tabla POIs (~1.85M × 3.5 KB c/u, con tags raw JSONB + procedencia) | ~9 GB |
| Índice GIST (geometría) | +4.5 GB |
| Índices btree (categoría, brand, ciudad) | +3 GB |
| **Subtotal POIs en PG** | **~17 GB** |
| Caché `places_validations` (10 KB/POI × ~200k validados) | ~2 GB |
| Históricos ohsome (17 cats × ~150 ciudades × 60 meses) | ~500 MB |
| Locators scrapeados (snapshots HTML + parseo) | ~3 GB |
| Boundaries (IGN + admin levels LATAM) | ~30 MB |
| Logs n8n + audit rolling | ~10 GB |
| Backups dump rolling (14d) | ~20 GB |
| **TOTAL operacional** | **~50–55 GB SSD** |

---

## 3. Tiempos de carga (responden a la intuición: son bajos)

### 3.a Bulk inicial (una sola vez, corre de noche)

| Tarea | Tiempo estimado | Por qué |
|---|---|---|
| Overture LATAM completa | ~20 min | DuckDB sobre S3 anónimo, ~15s por bbox grande, ~50 bboxes mayores |
| Overpass OSM por localidad (1.500–2.000 ciudades) | ~5 hs | Cortesía con la API pública (sleep 2s entre queries) |
| Boundaries IGN + Geofabrik por país | ~30 min | Una sola vez |
| Censo por país (manual XLSX / REDATAM) | 2–5 hs dev/país | Trabajo humano, no técnico |
| **Total bulk inicial técnico** | **~6–8 hs** | Corre 1 noche |

### 3.b Refresh mensual (n8n cron)

| Tarea | Tiempo |
|---|---|
| Overture release nuevo (sobrescribe diff) | ~20 min |
| Diff OSM (ohsome o re-query selectivo) | ~1 hora |
| Locators marcas (paralelizable) | ~2–3 horas |
| **Total refresh mensual** | **~3–4 horas, 02:00–06:00 ART** |

### 3.c Latencia que percibe el usuario en el mapa

| Acción | Tiempo |
|---|---|
| Query POIs radio 5 km (ciudad chica) | < 100 ms (GIST index) |
| Query POIs radio 10 km (ciudad grande, 5k POIs) | < 500 ms |
| Reporte radial completo (POIs + censo + GAF) | 1–3 s |
| Validación Places ad hoc (no cacheada) | 1–2 s (call externa) |

---

## 4. Disponibilidad de datos por país

| Fuente | AR | MX | ES | CO | PA | UY |
|---|---|---|---|---|---|---|
| OSM (gratis) | ✅ | ✅ | ✅ 1.4 GB, muy maduro | ✅ | ✅ | ✅ |
| Overture (gratis) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Google Places (cache) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Censo nacional | INDEC | INEGI | INE | DANE | INEC | INE |
| Asoc. franquicias | GAF/AAMF (178) | AMF (~750) | AEF (~350) | Colfranquicias (~200) | ~30–50 | ~20–30 |

**Las 4 primeras filas: data libre, mismo pipeline, escala lineal con bbox.** Censo y franquicias requieren trabajo manual por país (cada organismo con su formato), pero es trabajo humano puntual, no de infraestructura.

---

## 5. Scrapeos ad hoc de cadenas (lo más caro en horas humanas)

Approach validado en pre-piloto: **~10–15 min/marca con LLM-asistido + dev que testea**.

| Mercado | Marcas estimadas | Setup inicial (AI-asistido) | Refresh mensual con n8n |
|---|---|---|---|
| AR (GAF) | 178 | ✅ Ya scrapeado | 0–5 h (alertas mail si rompe) |
| MX | ~750 | ~125 hs | 5–15 h |
| ES | ~350 | ~60 hs | 3–8 h |
| CO | ~200 | ~35 hs | 2–5 h |
| PA + UY | ~70 | ~12 hs | 1–2 h |
| **Total expansión LATAM+ES** | **~1.370 marcas nuevas** | **~230 hs setup** | **~12–30 hs/mes** |

**Estrategia clave**: no es necesario scrapear las 1.700 marcas el día 1. Operamos **on-demand por cliente comercial**: cada cliente que entra tracciona el scrapeo de las 50–100 marcas que le importan (su categoría + competencia directa). Convierte el costo en variable, no fijo.

---

## 6. Dimensionamiento VPS Don Web Fase 2

| Plan | vCPU | RAM | SSD | Transf | Costo aprox | Cuándo usarlo |
|---|---|---|---|---|---|---|
| **Mínimo viable** | 4 | 8 GB | 80 GB | 2 TB | ~USD 10–15/mes | Piloto AR completa + extensión iniciada |
| **Recomendado** | 8 | 16 GB | 120 GB | 4 TB | ~USD 25–40/mes | LATAM en producción, 2–3 clientes activos |
| **Holgado** | 8 | 32 GB | 240 GB | 4 TB | ~USD 50–80/mes | LATAM completa, +10 clientes, tile cache MapLibre |

Don Web tiene escalado vertical en clicks: **arrancás con el mínimo y escalás cuando entran clientes**, sin migrar nada. Mismo nodo, mismo SSH, mismo dominio.

---

## 7. Resumen para presupuesto

1. **Infra: barata.** VPS Don Web tipo Access Power, USD 10–40/mes según fase. NO es el costo dominante.
2. **Tiempos: bajos.** Bulk LATAM 6–8 hs (una vez), refresh mensual 3–4 hs (n8n cron), queries en vivo < 500 ms. Sensación del usuario: la app responde rápido.
3. **Datos: disponibles y gratis.** OSM + Overture cubren AR + 5 países sin licencia paga.
4. **Censo + franquicias por país: variable.** ~2–5 hs dev/país para censo, ~12–125 hs/país para locators (puede ser on-demand por cliente).
5. **Google Places: ad hoc cacheado.** n8n controla cuota y permisos. USD 0–50/mes recurrente con caché agresivo en Postgres.

> **El factor dominante del presupuesto Fase 2 NO es la infraestructura, son las HORAS HUMANAS** de adaptar locators por país y curar mapeos de categorías Overture → FQC por mercado.

---

## Fuentes consultadas (2026-06-04)

- [Don Web Cloud Servers](https://donweb.com/es-ar/hosting-cloud-servers-vps)
- [Geofabrik South America (AR, CO, UY)](https://download.geofabrik.de/south-america.html)
- [Geofabrik Europe — España](https://download.geofabrik.de/europe/spain.html)
- [Geofabrik Central America — Panamá](https://download.geofabrik.de/central-america.html)
- [Geofabrik North America — México](https://download.geofabrik.de/north-america.html)
- [Overture Places Guide](https://docs.overturemaps.org/guides/places/)
- [PostGIS storage sizing — Dan Baston](http://www.danbaston.com/posts/2016/11/28/what-is-the-maximum-size-of-a-postgis-geometry.html)
- Memoria interna Access Power: VPS #5875378, hardening Fase 1+2, arquitectura consolidada Don Web.
