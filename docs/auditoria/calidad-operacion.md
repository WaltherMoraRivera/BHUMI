# Auditoría de calidad, pruebas, UX y operación: BHUMI

Fecha: 2026-11-26. Rama auditada: `main` (2be81eb). Fase 4: `origin/fase-4/bodega-piloto` (7534c48, PR MaxisotooH/BHUMI#4, abierto), revisada con `git show`/`git diff` y con una copia extraída por `git archive` en el scratchpad, sin checkout.
No se modificó nada del repositorio. `dist/` se generó para medir y luego se borró (no existía antes, y está en `.gitignore`). Los scripts propios están en `scratchpad/auditoria/pruebas-calidad/`: `funciones.mjs`, `demo-con-sesion.mjs`, `crear-despacho-hoy.mjs` y `medianoche.mjs`.
Entorno: Windows 11, Node 24.16.0, npm 11.13.0. No hay Docker en el equipo, así que todo lo que depende de la imagen queda marcado "por confirmar".

---

## (A) Métricas

### Líneas por carpeta (archivos versionados)

| Carpeta | Archivos | Líneas |
|---|---|---|
| server/ (total) | 27 | 5.667 |
| ├ server/services/ | 18 | 3.385 |
| public/js/ (total) | 27 | 3.421 |
| ├ public/js/views/ | 19 | 2.289 |
| public/css/ | 1 | 367 |
| navegador/ | 11 | 776 |
| test/ | 21 | 2.943 |
| docs/ | 2 | 247 |
| openspec/ | 22 | 719 |
| .claude/skills + .agents/skills (duplicados entre sí) | 13 | 2.705 |
| package-lock.json | 1 | 3.866 |

Fase 4 frente a su base (01a620d): +478 / −45 líneas en 20 archivos. Agrega `hoy.js` (114 líneas), `test/bodega.test.js` (82), `pruebas.yml` (57) y `PILOTO.md` (70). El `git diff main origin/fase-4/...` muestra −3.563 líneas porque la rama no incluye el commit de OpenSpec de `main`; esas líneas no se borran de verdad.

### Archivos más grandes
`server/api.js` 1.001 · `server/db.js` 494 · `server/services/planner.js` 434 · `server/services/reportes.js` 423 · `public/css/app.css` 367 · `public/js/views/rutas-nueva.js` 350 · `server/auth.js` 343 · `server/services/tiempos.js` 341 · `server/services/panel.js` 314 · `public/js/views/configuracion.js` 274 · `public/js/lib.js` 273.

### Funciones más largas
Medidas con `funciones.mjs`, una heurística por llaves. No usa AST, así que los conteos son aproximados.

| Líneas | Función |
|---|---|
| 922 | `createApi` (server/api.js:81). Toda la API vive en una sola clausura |
| 341 | `pintarNueva` (public/js/views/rutas-nueva.js:10) |
| 165 | `pintarEntregas` (public/js/views/rutas-entregas.js:8) |
| 161 | `iniciarTiempoReal` (public/js/tiempo-real.js:35) |
| 146 | `pintarClientes` (public/js/views/clientes-lista.js:12) |
| 125 | `createGoogle` (server/services/google.js:73) |
| 118 | `detalle` (public/js/views/rutas-despachos.js:76) |
| 113 | `sugerirConsolidacion` (server/services/consolidacion.js:113) |
| 111 | `pintarCalculadora` (public/js/views/cubicaje-calculadora.js:7) |
| 102 | `calcularCostos` (server/services/costs.js:103) |

- 32 funciones pasan de 50 líneas.
- 673 líneas pasan de 180 caracteres. La más larga mide 773 (`rutas-despachos.js:118`), seguida de `app.js:16` con 717 y `rutas-nueva.js:305` con 661.
- Hay 331 atributos `style="…"` inline en `public/js`.
- Hay 24 bloques `catch` vacíos o silenciosos. Todos llevan un comentario que explica por qué, y la mayoría son razonables (ver CAL-12 y CAL-11 para los que no).

### Pruebas y tiempos

| Ejecución | Resultado | Tiempo |
|---|---|---|
| `npm test` en main | 109/109 ✔ | 22,5 s de reloj (21,3 s según node) |
| Igual, con `--test-concurrency=1` (en serie) | 109/109 ✔ | 40 s. No depende del orden de los archivos |
| `npm run test:local` en Windows | Falla: "BHUMI_LOCAL no se reconoce" (ya está el issue #2) | — |
| `BHUMI_LOCAL=1` desde Git Bash, con y sin alias de navegador | 82 ✔ / 27 omitidas | 3 s cada una |
| `BHUMI_LOCAL=wasm` desde Git Bash | 82 ✔ / 27 omitidas | 4 s |
| Suite de la Fase 4 (copia extraída) | 113/113 ✔ | 18 s |
| `tiempo-real` + `auth` repetidas 5 veces (dependen de esperas) | 31/31 ✔ las 5 veces | ~14 s cada vez |

Las pruebas más lentas:
- 3,6 s "50 fallos desde una IP": cada intento hace un scrypt con N=2^15.
- 2,2 s el bloqueo por correo.
- 2,2 s "canal de eventos exige sesión".
- 1,9 s la restauración de un respaldo.

La salida de `npm test` imprime un stack `Error: FOREIGN KEY constraint failed` (respaldo.js:57). Lo provoca una prueba que espera ese fallo (ver CAL-11).

### Cobertura
Comando: `node --test --experimental-test-coverage --test-coverage-exclude='test/**'`. Total de los archivos cargados: **93,44 % de líneas, 84,05 % de ramas y 90,54 % de funciones**. Ojo: solo cuenta los archivos que cargan las pruebas.

| Carpeta / archivo | Líneas | Ramas | Notas |
|---|---|---|---|
| server/services (18 archivos) | 91–100 % | 63–100 % | `ruta-conductor.js` tiene 63 % de ramas, `google.js` 69 %, `stops.js` 78 % y `reportes.js` 76 % |
| server/api.js | 86,4 % | 73,5 % | Sin probar: PUT/DELETE de entregas, cambio de estado de una entrega, PUT y DELETE de clientes, `GET /clientes/:id`, `geolocalizar-pendientes`, gastos de un despacho, `/respaldo.db` y restauración `.db` |
| server/auth.js | 98 % | 82 % | |
| server/db.js, eventos.js, concurrencia.js, config.js | 100 % | 87–93 % | |
| server/index.js | 62,9 % | 70 % | El arranque real y el cierre ordenado (SIGTERM) no se prueban |
| navegador/google-navegador.js | 98,6 % | 82,7 % | |
| navegador/mini-express.js, sqlite-wasm.js, runtime.js | 25 / 52 / 60 % | — | Solo se ejercitan en los modos local y wasm |
| **navegador/almacenamiento.js, entrada.js, construir.mjs** | **0 % (no se cargan)** | — | Aquí está la persistencia IndexedDB de la versión de doble clic |
| **public/js/** (27 archivos, 3.421 líneas) | **0 % (no se cargan)** | — | No hay ninguna prueba de interfaz |

### Rendimiento con muchos datos
Script `demo-con-sesion.mjs`: servidor real, base en memoria y sesión de administrador. Cada fila es una solicitud en milisegundos y tamaño de la respuesta.

| Endpoint | 3.000 clientes × 5 entregas | 300 clientes × 50 entregas (≈1 año) | 3.000 × 5 **con 2 índices** |
|---|---|---|---|
| GET /clientes | **8.034 ms** · 1,2 MB | 625 ms | **23 ms** |
| GET /entregas (todas) | **12.037 ms** · 12,5 MB | **14.203 ms** · 12,5 MB | 638 ms |
| GET /entregas?estado=pendientes | 2.546 ms | 240 ms | 182 ms |
| GET /panel | 7.456 ms | 712 ms | 1.091 ms |
| GET /consolidacion | 7.784 ms | 707 ms | 1.171 ms |
| GET /zonas/mapa | 1.496 ms | 24 ms | 21 ms |
| GET /respaldo.json | 131 ms · 8,4 MB | 96 ms | 161 ms |

Los dos índices de la última columna son `deliveries(client_id)` y `delivery_items(delivery_id)`.

### Tamaños de construcción e imagen
- `npm run construir` tarda 2,6 s. `Abrir BHUMI.html` pesa **4.735 KB** (1.515 KB en gzip) y `Abrir DEMO.html` **4.743 KB** (1.518 KB en gzip). Cada uno trae sql.js, Chart.js, ExcelJS, PDFKit y las fuentes en base64. `codigo-fuente/` suma 1,6 MB más.
- Carga inicial en el servidor, pantalla Panel: 22 recursos, de ellos 8 módulos JS y 6 llamadas a la API. DOMContentLoaded en 97 ms en local. Los estáticos se sirven con `Cache-Control: public, max-age=0`, o sea que se revalidan siempre.
- Imagen Docker: **por confirmar** (no hay Docker aquí). La estimación es ≈ 235 MB: `node:24-alpine` pesa ~165 MB y las dependencias de producción en node_modules ≈ 70 MB (exceljs 23 MB, chart.js 7, pdfkit 5, fuentes 4, más express y otras). En CI el job "Imagen Docker" tarda 16 s y no reporta el tamaño.
- `npm audit --omit=dev`: **2 vulnerabilidades moderadas** (uuid < 11.1.1, que llega por exceljs, GHSA-w5hq-g745-h8pq).
- `npm outdated`: express 4.22.3 frente a 5.3.0, pdfkit 0.15.2 frente a 0.21.0 y chart.js 4.6.1 frente a 4.6.2 (ya está el issue #18).

### Proceso (git y GitHub)
- Desde la importación (21:38) hasta el merge de la Fase 3 (23:25) pasaron **1 h 47 min**, el 25-11-2026. La Fase 4 se subió 10 minutos después.
- PR 1 a 4: +569, +1.118, +813 y +478 líneas, con 15 a 22 archivos cada uno. **0 revisiones en GitHub**.
- Los PR 1 a 3 los fusionó el mismo autor **entre 12 y 20 minutos después de abrirlos**, y sin CI (el CI llega recién con la Fase 4).
- La rama `main` no está protegida en el fork (la API responde "Branch not protected"). En upstream la respuesta fue 404: la protección no se pudo verificar, posiblemente por permisos.
- `upstream/main` y `origin/main` divergen: el fork lleva 2 commits más, los de OpenSpec. Los issues están en el fork y los PR en upstream, y DESPLIEGUE.md manda a clonar upstream.

---

## (B) Hallazgos

### CAL-01 · Faltan dos índices: las listas tardan segundos y congelan el servidor para todos
- **Severidad:** alta · **¿Bloquea el piloto?** no, salvo que se migre un historial grande (por confirmar el volumen real).
- **Evidencia:**
  - `server/db.js:124-218` no tiene índice en `deliveries(client_id)` ni en `delivery_items(delivery_id)`.
  - `consultaClientes` (api.js:323-327) hace 3 subconsultas correlacionadas por cliente.
  - `entregaCompleta` (api.js:476) llama a `itemsDeEntrega` una vez por fila, un patrón N+1.
  - Medido con 300 clientes × 50 entregas (≈1 año): `GET /entregas` (filtro "todas") tarda **14,2 s y devuelve 12,5 MB**, y `/clientes` 625 ms.
  - Con 3.000 × 5: `/clientes` pasa de 8,0 s a **23 ms** y `/panel` de 7,5 s a 1,1 s solo con agregar los dos índices.
- **Impacto:** `node:sqlite` es síncrono, así que mientras dura una consulta larga el proceso no atiende a nadie más. Eso incluye los pings SSE y los toques de bodega en "Hoy". Un planificador que elige "todas las entregas" congela el servidor central 14 s.
- **Recomendación:**
  - `CREATE INDEX IF NOT EXISTS` sobre `deliveries(client_id)`, `delivery_items(delivery_id)` y, de paso, `deliveries(fecha)` y `trips(fecha)`.
  - Traer los ítems de todas las entregas en una sola consulta (`WHERE delivery_id IN (…)`).
  - Agregar una prueba de rendimiento con volumen sintético que falle si `/entregas` pasa de N ms.
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### CAL-02 · Zona horaria: el servidor calcula en UTC y la operación es en Chile
- **Severidad:** alta · **¿Bloquea el piloto?** **sí**. El piloto mide justamente costos y tiempos ("¿Los costos y kilómetros de las rutas coinciden…?", PILOTO.md §3).
- **Evidencia:**
  - El `Dockerfile` y el `docker-compose.yml` no definen `TZ`, así que el contenedor corre en UTC.
  - `planner.js:32` hace `new Date(\`${fecha}T${hora}:00\`)` en hora local del servidor. Medido: con TZ=UTC, una salida "08:00" se consulta a Google como `08:00Z`, que son las **05:00 en Chile**. Con TZ=America/Santiago da `11:00Z`. El tráfico se pide para una hora 3 o 4 h antes de la real.
  - `api.js:74` `hoyStr = toISOString()` en UTC: a las 23:30 de Chile ya devuelve el día siguiente (verificado). Se usa como fecha por defecto de la entrega realizada (api.js:553), de las cargas de combustible (254) y de la vigencia de precios (194).
  - Lo mismo en `db.js:488` (precio vigente), `analytics.js:6` (`mesActual`) y `consolidacion.js:61`. Conviven además con `hoyLocal` (vehiculos-estado.js:25), que en el contenedor también da UTC.
  - En total hay 7 copias de `toISOString().slice(0, 10)` en server/.
- **Impacto:**
  - El tiempo de ruta con tráfico, el costo del conductor y los KPI de atraso se calculan con un tráfico que no corresponde.
  - Las entregas cerradas de noche quedan con fecha de mañana, y los reportes mensuales se corren de mes el último día después de las 21:00.
  - La versión de doble clic no lo sufre (el navegador está en hora de Chile), así que el piloto mostraría diferencias "misteriosas" entre una versión y otra.
- **Recomendación:**
  - `ENV TZ=America/Santiago` en el Dockerfile y `TZ` en compose. Node trae ICU completo y no necesita `tzdata`, pero hay que confirmarlo dentro de la imagen.
  - Un único módulo `fechas.js` con `hoyChile()` y `salidaChile(fecha, hora)` que use `Intl` con `timeZone: 'America/Santiago'`, independiente del TZ del proceso.
  - Correr la suite en CI con `TZ=UTC` y `TZ=America/Santiago`.
- **Esfuerzo:** bajo a medio · **Modelo:** Sonnet

### CAL-03 · Ante un corte de red, la app le dice a bodega "restaura tu último respaldo"
- **Severidad:** alta · **¿Bloquea el piloto?** **sí**, por el uso en el andén con wifi intermitente.
- **Evidencia:**
  - `public/js/lib.js:56-58`, ante cualquier fallo de `fetch`, muestra: "Recarga la página; si sigue igual, restaura tu último respaldo desde Configuración → Respaldo."
  - Reproducido en la pantalla **Hoy** de la Fase 4: se simuló un fetch fallido y se tocó "Marcar ahora", y apareció exactamente ese toast.
  - El texto viene de la versión de doble clic. En el servidor, restaurar **reemplaza los datos de todos**, y bodega ni siquiera tiene acceso (es solo de admin).
  - No hay reintento ni cola: la marca de hora se pierde y el usuario tiene que acordarse de volver a tocar.
- **Impacto:** confusión en el momento más frecuente del piloto. Si lo lee un admin, puede terminar en pérdida de datos.
- **Recomendación:**
  - Mensaje distinto por modo: en el servidor, "Sin conexión con BHUMI. Revisa el wifi; tu toque no se guardó, vuelve a intentarlo".
  - En "Hoy", reintento automático con la hora capturada en el momento del toque (no la del reintento) y un indicador de "pendiente".
- **Esfuerzo:** bajo (mensaje) / medio (reintento) · **Modelo:** Sonnet

### CAL-04 · `npm run construir` mete `.env`, respaldos y archivos locales en el paquete que se entrega
- **Severidad:** alta · **¿Bloquea el piloto?** no
- **Evidencia:**
  - `navegador/construir.mjs:115-127` copia **todo el directorio de trabajo**. Solo omite `node_modules`, `dist`, `.git`, `data` y `*.db*`; no usa `git ls-files` ni `.gitignore`.
  - Verificado en una copia con archivos falsos: `dist-prueba/codigo-fuente/` quedó con `.env` (con la `GOOGLE_SERVER_KEY` falsa adentro), `respaldo-cliente.json` y `.claude/settings.local.json`.
- **Impacto:** quien arme la versión en su equipo con un `.env` real (las claves de Google del servidor, las credenciales S3 de Litestream) o con respaldos de clientes y la entregue en un .zip filtra secretos y datos personales.
- **Recomendación:**
  - Copiar solo `git ls-files` (o una lista permitida), o como mínimo excluir `.env*` salvo `.env.example`, `respaldo*.json`, `.claude/settings.local.json` y `*.log`.
  - Agregar una prueba que arme el paquete con `--sin-sqljs` sobre un directorio que tenga un `.env` señuelo y verifique que no aparece.
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### CAL-05 · `npm run demo` está roto desde la Fase 2
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:** al correr `node test/servidor-prueba.js 3157` aborta con `Error: PUT /settings: Tu sesión terminó`. `sembrarDemo` llama a la API sin cookie y `crearApp` siempre exige sesión. Con `--vacio` arranca, pero no hay ningún usuario con el que entrar. Sigue igual en la Fase 4. README.md:174 lo recomienda "para probar sin Google".
- **Impacto:** el flujo de revisión manual que documenta el README no funciona. Un desarrollador nuevo se topa con un error en su primer comando. Además ninguna prueba ejecuta `servidor-prueba.js`, por eso nadie lo notó.
- **Recomendación:** que el servidor de demo cree un admin y un usuario de bodega de prueba, inicie sesión al sembrar e imprima las credenciales de demostración (como hace `scratchpad/.../demo-con-sesion.mjs`). Agregar una prueba de humo que lo levante y consulte `/api/salud`.
- **Esfuerzo:** bajo · **Modelo:** Haiku

### CAL-06 · "Hoy" no sirve para preparar la carga la noche anterior ni para regresos después de medianoche
- **Severidad:** media · **¿Bloquea el piloto?** por confirmar con la operación real de bodega
- **Evidencia:**
  - `hoy.js` solo pide `desde=hoy&hasta=hoy`: un despacho de mañana a las 06:00 no aparece hoy a las 22:00.
  - `tiempos.js:94` guarda todo como `${trip.fecha}THH:MM` y `validarOrden` rechaza horas que cruzan la medianoche. Probado con la API: preparación 22:30–23:15 y salida 06:00 dan **400** "La hora de salida de la base (06:00) no puede ser anterior a la hora de fin de preparación (23:15)".
- **Impacto:** si bodega prepara de noche (algo habitual en distribución), no puede usar la pantalla pensada para ella, o anota horas falsas. Los KPI de preparación quedan vacíos o distorsionados.
- **Recomendación:**
  - Mostrar en "Hoy" también los despachos de mañana que aún no salen, y los de ayer que siguen "en ruta".
  - Guardar el momento con su fecha (no solo HH:MM), o permitir el día anterior en los pasos de preparación y carga, con una prueba.
- **Esfuerzo:** medio · **Modelo:** Opus (decisión de modelo de datos)

### CAL-07 · Detalles de uso en celular y accesibilidad de "Hoy"
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia (medida en el navegador integrado a 375×812):**
  - Cada tarjeta de despacho mide **1.013 px**, más que la pantalla. Con 4 despachos hay ~4 pantallas de scroll y "Iniciar ruta" queda abajo del todo.
  - "Marcar ahora" mide 227×48, bien. "Otra hora" y "Corregir" miden **74×27**, bajo los 44×44 recomendados y pegados al botón principal: el toque equivocado es probable con guantes.
  - Al marcar, `repintar` reemplaza la tarjeta y el foco se va a `<main>`, así que el teclado y el lector de pantalla pierden su lugar.
  - El color `.tenue` (#7B8780 sobre blanco) da **contraste 3,74:1**, bajo el 4,5:1 que pide AA. Se usa en el subtítulo (13 px) y en las etiquetas de grupo de Hoy (12 px), y bajo el sol del andén se lee peor.
  - El toast del alta de un despacho dice "actualizó un despacho".
- **Lo que está bien:** formularios con `<label for>` o `aria-label` en las 13 pantallas revisadas (0 controles sin etiqueta y 0 botones sin texto), `:focus-visible`, `prefers-reduced-motion`, modal con `role=dialog`, `aria-modal` y Escape, toasts con `aria-live`, y el login con `autocomplete` correcto.
- **Recomendación:**
  - Modo compacto en "Hoy": solo el paso siguiente grande y los demás en una línea.
  - "Otra hora" de al menos 44 px de alto y separado del botón principal.
  - Devolver el foco al botón del paso siguiente.
  - Oscurecer `--tenue` hasta ≥4,5:1.
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### CAL-08 · Observabilidad casi nula: sin logs útiles, rotación, alertas ni un healthcheck que detecte disco lleno
- **Severidad:** media · **¿Bloquea el piloto?** no, pero conviene tener lo mínimo antes del día 1
- **Evidencia:**
  - Mientras corre, el servidor registra **una sola línea de error**: `console.error(err)` en api.js:994, sin ruta, usuario ni id de solicitud. No hay log de solicitudes ni de inicios de sesión fallidos (esos van solo a la tabla `auditoria`), y nada está en JSON.
  - `docker-compose.yml` no configura `logging` (por defecto json-file sin límite de tamaño).
  - `/api/salud` (index.js:37) solo hace `SELECT 1`. Si el disco se llena o el volumen queda de solo lectura, sigue diciendo `ok:true` mientras todas las escrituras fallan con 500 "error inesperado" (comportamiento con disco lleno **por confirmar**).
  - No hay monitoreo externo ni alertas. Si Litestream falla (credenciales malas, bucket lleno), solo queda en sus logs y nadie se entera. PILOTO.md descansa en que una persona revise la "Actividad reciente" cada día.
- **Recomendación:**
  - Un log por solicitud en JSON de una línea (método, ruta, estado, ms, usuario, id), y el `console.error` con ese id.
  - `logging: { driver: json-file, options: { max-size: 10m, max-file: "5" } }` en los 3 servicios.
  - Que `/api/salud` pruebe además una escritura (por ejemplo `PRAGMA quick_check` o una tabla de latido) y reporte el espacio libre del volumen.
  - Un monitor externo gratuito sobre `/api/salud` con aviso por correo.
  - Revisar `litestream snapshots` en un cron con alerta.
  - Documentar todo esto en DESPLIEGUE.
- **Esfuerzo:** medio · **Modelo:** Sonnet

### CAL-09 · La auditoría, las sesiones vencidas y `route_cache` crecen sin límite, y la auditoría guarda IPs sin plazo de retención
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:**
  - Nada borra filas de `auditoria`: `grep` encuentra solo INSERT, en auth.js:141 y crear-admin.js:50,58. Cada escritura de la API suma una fila con la IP (auth.js:252).
  - `route_cache` (planner.js:90) solo se vacía al restaurar.
  - Las sesiones se purgan solo al iniciar sesión (auth.js:207). Esa parte está bien.
- **Impacto:** el crecimiento es moderado (unas decenas de MB al año). Lo importante es el dato personal (IP + usuario + acción) sin política de retención, que choca con la Ley 19.628 y con la Ley 21.719 cuando entre en vigencia. Cada fila extra también va a parar al respaldo `.db` y a Litestream.
- **Recomendación:** purga diaria o al arrancar de la auditoría más vieja que N días (configurable, por ejemplo 365) y de `route_cache` más viejo que `cache_horas`. Documentar la retención.
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### CAL-10 · Huecos de prueba concretos (además del e2e del issue #17)
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia (cobertura, sección A):**
  - En api.js no se prueban PUT/DELETE `/entregas/:id`, `POST /entregas/:id/estado`, PUT/DELETE `/clientes/:id`, `GET /clientes/:id`, `/despachos/:id/gastos` (que recalcula `costo_total`), `/respaldo.db` ni la restauración `.db`.
  - `navegador/almacenamiento.js` y `entrada.js` (la persistencia IndexedDB y el respaldo interno de la versión de doble clic) no se cargan en ninguna prueba.
  - `public/js` está al 0 %.
  - No hay pruebas de migración desde una base con un esquema anterior, de `server/index.js` como proceso (SIGTERM, cierre ordenado), de Google real (solo un fake; `google.js` tiene 69 % de ramas) ni de carga o volumen (ver CAL-01).
  - Lo bueno: las pruebas de tiempo real resultaron estables en 5 de 5 repeticiones y no dependen del orden.
- **Recomendación:**
  - Pruebas de API para las rutas sin cubrir, que son baratas y entran con los helpers actuales.
  - Una prueba de migración que cree la base con el `SCHEMA` de la primera versión (sin `COLUMNAS_NUEVAS`) y abra con el actual.
  - Una prueba de `almacenamiento.js` con `fake-indexeddb`.
  - Medir cobertura en CI con un umbral mínimo para `server/`.
- **Esfuerzo:** medio · **Modelo:** Sonnet

### CAL-11 · Restaurar un respaldo inconsistente da "error inesperado" (500) y vuelca un stack trace
- **Severidad:** media · **¿Bloquea el piloto?** no, pero la migración al servidor del día 1 es justamente una restauración JSON
- **Evidencia:**
  - `restaurarJson` (respaldo.js:57) deja escapar el `ERR_SQLITE_ERROR FOREIGN KEY constraint failed`. El manejador de errores (api.js:993) lo convierte en 500 "Ocurrió un error inesperado" y lo imprime completo; se ve en la salida de `npm test`.
  - La prueba (respaldo.test.js:89) solo afirma `notEqual(200)`.
- **Recomendación:** capturar los errores de SQLite dentro de `restaurarJson` y lanzar `RespaldoError` (400) con la tabla y la fila que fallan, por ejemplo "el despacho 12 apunta al vehículo 999, que no está en el respaldo". La prueba debe exigir 400 y ese mensaje.
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### CAL-12 · Guardar un despacho armado con clientes es una "transacción" hecha desde el navegador
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:**
  - `rutas-nueva.js:279-289` hace N `POST /entregas` y después `POST /despachos`. Si falla, compensa con `del(...).catch(() => {})`, que traga los errores.
  - Si la red se corta (que es justo cuando falla el POST), también fallan los DELETE y quedan entregas huérfanas "pendientes" sin aviso.
  - En el servidor central cada POST genera además un evento SSE para todos los conectados y una fila de auditoría.
- **Recomendación:** un endpoint `POST /despachos` que acepte `cliente_ids` y cree las entregas dentro de la misma `tx` del servidor.
- **Esfuerzo:** medio · **Modelo:** Opus

### CAL-13 · Lo que le falta al CI de la Fase 4
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia (`.github/workflows/pruebas.yml` de la rama Fase 4):**
  - Corre bien: 4 modos de prueba, construir, imagen Docker con `/api/salud`, 401 y una salida limpia. La última corrida pasó (Node 22.13 en 1 min 1 s, Node 24 en 47 s, Docker en 16 s).
  - Falta:
    - `npm audit --omit=dev --audit-level=high` (hoy hay 2 moderadas).
    - `actions/upload-artifact` de `dist/BHUMI`: se arma y se descarta.
    - Windows en la matriz (issue #2) y lint (issue #3).
    - Cobertura con umbral.
    - `docker compose config` y `caddy validate --config deploy/Caddyfile`.
    - Medir y limitar el tamaño de la imagen (`docker image inspect`).
    - Probar que los datos sobreviven en el volumen al recrear el contenedor.
    - Correr con `TZ=UTC` (CAL-02) y una prueba de humo de `npm run demo` (CAL-05).
  - Las acciones están fijadas por tag (`@v4`) y no por SHA.
  - Los PR 1 a 3 se fusionaron **sin CI**.
- **Recomendación:** agregar los pasos de audit, artefacto, compose/Caddy y tamaño de imagen ahora, porque son baratos. Pedir el check como obligatorio con protección de rama (CAL-14).
- **Esfuerzo:** bajo · **Modelo:** Haiku (pasos) / Sonnet (volumen y TZ)

### CAL-14 · Proceso: ritmo de 2 horas, autorrevisión y dos repositorios divergentes
- **Severidad:** media · **¿Bloquea el piloto?** no (pero aumenta el riesgo de todo lo anterior)
- **Evidencia:**
  - 4 fases (~3.000 líneas) en 1 h 47 min, el mismo día.
  - Los PR tienen 0 revisiones en GitHub y los fusionó su propio autor entre 12 y 20 minutos después de abrirlos. Las correcciones aparecen como commits "Corregir hallazgos de la revisión", sin rastro de quién revisó ni de qué.
  - `main` sin protección en el fork.
  - Upstream (MaxisotooH) y el fork (WaltherMoraRivera) tienen `main` distintos: los issues están en el fork y los PR en upstream, y DESPLIEGUE.md clona upstream.
  - La rama de la Fase 4 parte de antes de OpenSpec.
  - Los commits son descriptivos, en español y sin convención.
  - Hay 2.705 líneas de skills de OpenSpec duplicadas entre `.claude/` y `.agents/`, y además entran al paquete que se entrega (CAL-04).
- **Impacto:** errores como CAL-02, CAL-05 o CAL-03 pasan porque nadie usa el sistema entre fases. Además no queda claro cuál es el repositorio "de verdad" para desplegar.
- **Recomendación:**
  - Elegir **un** repositorio canónico.
  - Proteger `main`: PR obligatorio, CI verde y 1 revisión de otra persona o, si el equipo es de uno, una lista de verificación de revisión firmada en el PR.
  - PR de menos de 400 líneas.
  - Etiquetar versiones (`v1.x`) y desplegar etiquetas, no `main`.
  - Un día de "uso real" por fase antes de fusionar.
  - Que la revisión deje un comentario en el PR.
- **Esfuerzo:** bajo (configurar) / continuo · **Modelo:** Opus para diseñar el flujo y Haiku para aplicar la configuración

### CAL-15 · Actualizar no tiene versión ni vuelta atrás
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:**
  - DESPLIEGUE "Actualizar": `git pull && docker compose up -d --build`, es decir, se compila en el servidor desde la rama, sin etiqueta de imagen.
  - No pide respaldar antes ni explica cómo volver atrás.
  - Las migraciones solo agregan columnas al arrancar (db.js:398), así que una versión vieja sobre una base migrada funciona "por suerte" (issue #13).
  - `VERSION` sale de package.json, que sigue en 1.0.0 desde la importación.
- **Recomendación:**
  - Imagen etiquetada (`bhumi:1.2.0`) construida en CI y publicada en GHCR.
  - En compose, `image:` con la etiqueta.
  - Pasos documentados: respaldo `.json` y snapshot de Litestream, cambiar la etiqueta, `up -d`, verificar `/api/salud` y la versión, y cómo volver a la etiqueta anterior.
  - Subir la versión en cada fase.
- **Esfuerzo:** medio · **Modelo:** Sonnet

### CAL-16 · Documentación: contradicciones y lo que le falta a un desarrollador nuevo
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:**
  - En README "Importante" (líneas 188-192): "Cada llamada a Google se hace desde tu navegador" y "Los datos viven en un solo navegador de un solo computador", que es falso en el servidor central descrito unas líneas antes. Falta aclarar a qué modo se refiere.
  - README:174 recomienda `npm run demo`, que está roto (CAL-05).
  - No está documentado:
    - La cadena de middlewares del servidor (`auth.publico` → `exigirSesion` → `eventos.anunciar` → `concurrencia` → `auth.privado` → `createApi`, index.js:47-58) ni dónde agregar una ruta con permisos.
    - Cómo depurar (inspector, logs, abrir la base con `sqlite3`).
    - La zona horaria.
    - Los logs y la retención.
    - Los tres modos de ejecución en un diagrama.
  - PILOTO.md no incluye: cómo cargar un despacho preparado la noche anterior (CAL-06), un "plan B" si se cae el wifi del andén ni una plantilla de reporte de incidente.
  - LEEME.txt es coherente con la versión de doble clic.
- **Recomendación:** separar el README en "Usar (doble clic)", "Servidor" y "Desarrollar". Agregar `docs/ARQUITECTURA.md`, de una página, con el diagrama de modos y de middlewares, y "Cómo depurar".
- **Esfuerzo:** bajo · **Modelo:** Sonnet (arquitectura) / Haiku (correcciones)

### CAL-17 · Listas completas sin paginación y respuestas pesadas
- **Severidad:** baja (después de CAL-01) · **¿Bloquea el piloto?** no
- **Evidencia:**
  - `/entregas` sin filtro devuelve 12,5 MB con 15.000 entregas.
  - `/clientes`, `/despachos`, `/gastos` y `/km-semanal` (este con LIMIT 200) devuelven todo.
  - La tabla de clientes y la de entregas pintan todas las filas en el DOM (clientes-lista.js y rutas-entregas.js).
  - `/zonas/incidentes` tiene LIMIT 300, pero la interfaz no lo dice.
- **Recomendación:** paginación por servidor (`limit`/`offset`, o por cursor de fecha) en entregas, despachos y gastos. Filtro por defecto de los últimos 90 días en "todas".
- **Esfuerzo:** medio · **Modelo:** Sonnet

### CAL-18 · Calidad de código: tamaño, duplicación y estilo
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:**
  - `createApi` mide 922 líneas (el issue #12 ya existe; la cifra es nueva).
  - Las vistas son funciones de 111 a 341 líneas.
  - Hay 673 líneas de más de 180 caracteres y 331 `style=` inline.
  - `v.nombre ?? \`${marca} ${modelo}\`` está repetido 13 veces aunque `nombreVehiculo` existe en lib.js.
  - El `where.length ? 'WHERE '…` aparece 3 veces en api.js.
  - Hay dos `wrap` idénticos (api.js:41 y auth.js:64).
  - 7 copias de la fecha en UTC (CAL-02).
  - Rastro del entorno del autor: `construir.mjs:16` busca esbuild en `/opt/npm-tools/...`.
  - A favor: los nombres en español son consistentes, los comentarios explican el porqué, no hay promesas sin `await` evidentes en el servidor (las rutas async pasan por `wrap`) y los catch vacíos están justificados.
- **Recomendación:** con el linter del issue #3, activar `max-len` (con excepciones para plantillas), `max-lines-per-function` en modo aviso y `no-restricted-syntax` para `toISOString().slice(0,10)`. Extraer los helpers `nombreVehiculo` (servidor) y `armarWhere`.
- **Esfuerzo:** bajo · **Modelo:** Haiku

### CAL-19 · Pruebas lentas por el scrypt real y caché de estáticos sin versión
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:**
  - Las 5 pruebas más lentas de auth suman ~9,6 s porque usan `PARAMS` de scrypt de producción (auth.js:28, N=2^15). En serie, la suite tarda 40 s.
  - Los estáticos salen con `max-age=0` y URLs sin hash. Con una pestaña abierta durante una actualización, las vistas cargadas de forma diferida pueden mezclar módulos nuevos con un `lib.js` viejo (**por confirmar**).
- **Recomendación:**
  - Parámetros de scrypt inyectables en las pruebas (N=2^10).
  - Que la versión del servidor viaje en `/api/meta`: si cambia, la interfaz pide recargar.
- **Esfuerzo:** bajo · **Modelo:** Haiku

### CAL-20 · Los archivos de doble clic pesan 4,7 MB cada uno, con todo duplicado
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:** `Abrir BHUMI.html` (4.735 KB) y `Abrir DEMO.html` (4.743 KB) traen cada uno sql.js (wasm), Chart.js, ExcelJS, PDFKit y las fuentes. Comprimido, el zip pesa ~3 MB más 1,6 MB de `codigo-fuente`. En Chrome abren bien (por confirmar en equipos lentos).
- **Recomendación:** solo si la versión de doble clic sigue después del piloto (issue #21): cargar ExcelJS y PDFKit solo al exportar y no incluir `codigo-fuente` en el zip para usuarios.
- **Esfuerzo:** medio · **Modelo:** Sonnet

---

## (C) Lo que está bien resuelto

- **Pruebas de API sólidas.** Hay 109 pruebas (113 en la Fase 4) que validan la misma API en 4 modos de ejecución, son estables (5 de 5 repeticiones) y no dependen del orden. Los servicios de negocio tienen 91–100 % de líneas cubiertas.
- **Seguridad de sesión y permisos bien pensada.**
  - Contraseñas con scrypt y parámetros versionados, y un hash señuelo para no revelar qué correos existen.
  - Bloqueo por correo+IP y por IP.
  - Cookie HttpOnly/SameSite y control de `Origin` en las escrituras.
  - Permisos validados en el servidor y auditoría de cada escritura.
- **Concurrencia.** El control de versiones con triggers (cualquier UPDATE cuenta) y el 409 con el nombre de quien cambió están bien resueltos. Hay tiempo real por SSE con reconexión, y verifiqué en el navegador que un despacho nuevo aparece en "Hoy" sin recargar.
- **Despliegue.**
  - Dockerfile mínimo, con usuario no root, `HEALTHCHECK`, `npm ci --omit=dev` y cierre ordenado con SIGTERM que cierra los SSE y la base.
  - Las claves de Google están separadas por entorno, y la configuración se valida al arrancar con mensajes claros.
  - La restauración con Litestream está documentada en detalle (generaciones, base vacía, UTC) e incluso probada.
- **Interfaz.**
  - Plantilla `html``…`` con escape automático.
  - Etiquetas en todos los controles revisados, `:focus-visible`, `prefers-reduced-motion`, modales accesibles y `aria-live`.
  - Los botones principales de "Hoy" miden 48 px.
  - Los mensajes de validación están en español claro y orientados a la acción.
- **Documentación de usuario.** PILOTO.md es concreto (lista de verificación, prueba de humo con dos personas, criterios de cierre) y LEEME.txt es claro para usuarios no técnicos.
- **Higiene del repositorio.** `.gitignore` y `.dockerignore` excluyen `.env`, las bases y los respaldos; `dist/` no se versiona y el CI arma el paquete.
