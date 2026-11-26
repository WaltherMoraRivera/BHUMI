# Auditoría de arquitectura de BHUMI

Fecha: 2026-11-26 · Alcance: `main` (2be81eb) y `origin/fase-4/bodega-piloto` (7534c48, PR abierto), leída sin checkout.
Método: lectura del código, `npm test` (109/109 en 21 s, Node 24.16 local) y la suite en modo wasm (`BHUMI_LOCAL=wasm`: 82 pasan, 27 omitidas).
Las cifras de líneas son **líneas no vacías** (`Measure-Object -Line`); los números `archivo:línea` son los del archivo real.

Nota sobre la rama de la Fase 4: está cortada antes del merge de OpenSpec, así que `git diff main origin/fase-4/bodega-piloto` (dos puntos) aparenta borrar `openspec/`. Para revisar el PR hay que usar tres puntos (`main...origin/fase-4/bodega-piloto`): son 20 archivos, +478/−45.

---

## Resumen

| Decisión | Recomendación | Por qué, en una línea |
|---|---|---|
| 1. Modo doble clic junto al servidor | **Congelarlo ya** (sin funciones nuevas) y dejarlo como plan B durante el piloto. **Después del piloto, quitar el modo real** y dejar, como mucho, la demo | Cuesta poco en líneas (~880), pero amarra todo el servidor a una API de base de datos síncrona, a dependencias que se puedan empaquetar para el navegador y a 4 modos de prueba. Además expone la clave de Google sin restricción de sitio |
| 2. SQLite | **Mantener SQLite en el servidor.** Fijar la versión de Node y tener `better-sqlite3` como salida de emergencia. PostgreSQL solo si el piloto pide más de un servidor o consultas geográficas | Para menos de 10 usuarios y una bodega, SQLite + WAL + Litestream es lo adecuado. PostgreSQL obligaría a reescribir 181 llamadas síncronas |
| 3. Interfaz JS sin framework | **Seguir así durante el piloto.** Agregar `// @ts-check` + JSDoc con `tsc --noEmit` en CI y una prueba de humo con Playwright. Evaluar Preact+htm después del piloto, solo si crece | Las vistas son chicas (la mayor tiene 328 líneas) y el escape es correcto. Lo que falta son tipos y pruebas, no un framework |
| 4. Solo Google Maps | **Mantener Google** (sobre todo para geocodificar en Chile). Corregir la caché, revisar tráfico = SKU Pro y tener una salida manual cuando Google no responde. OSRM autohospedado solo si el costo real del piloto lo justifica | La dependencia es total: sin Google no se puede ni guardar un despacho en orden manual |

---

## Parte A. Decisiones de fondo

### A1. Mantener el modo doble clic (`navegador/`) junto al servidor central

**Situación actual (evidencia)**

- El código propio del modo es chico: `navegador/` suma 702 líneas no vacías en 11 archivos (`mini-express.js` 125, `google-navegador.js` 128, `almacenamiento.js` 116, `construir.mjs` 107, `entrada.js` 102, `sqlite-wasm.js` 66, shims). Con el soporte de pruebas (`test/alias-navegador*.mjs`, `test/sqljs-sustituto.js`, `test/google-navegador.test.js`) llega a ~877 líneas. Todo el proyecto tiene ~11.900 (server 5.302, public/js 3.173, test 2.697, navegador 702), así que el modo pesa ~7 %.
- **El costo real no está en esas líneas, está en lo que impone al resto del código:**
  - `server/api.js:81` recibe `express` como parámetro para poder cambiarlo por `mini-express`. `server/api.js:2-6` importa `fs`, `os` y `path`, que en el navegador se reemplazan por un módulo vacío (`navegador/construir.mjs:380`). Por eso `/respaldo.db` tiene que preguntar qué tipo de base tiene (`server/api.js:959`, `:974`).
  - Los 18 servicios y `api.js` usan la API **síncrona** de `DatabaseSync` (181 llamadas a `db.prepare`: 101 en `api.js` y 80 en servicios). `navegador/sqlite-wasm.js` imita justamente esa API. Pasar a PostgreSQL, o a cualquier base asíncrona, rompe el modo doble clic. Es un bloqueo de diseño, no de líneas.
  - Cada dependencia nueva del servidor tiene que poder empaquetarse con esbuild para `file://` (`construir.mjs:378-385`: alias de PDFKit standalone, ExcelJS min, EventEmitter propio).
  - El bundle de producción importa código de `test/`: `navegador/entrada.js:14-15` (`fake-google.js`, `demo-datos.js`).
- **Pruebas:** hay 109 pruebas. Con `test`, `test:local`, `test:navegador` y `test:wasm` se ejecutan 109 + 3×82 = 355 veces por cambio, porque 27 son solo del servidor HTTP (`test/auth.test.js:7`, `test/servidor-central.test.js:56`, `test/tiempo-real.test.js:9`). El CI de la Fase 4 corre los 4 modos más el armado (`.github/workflows/pruebas.yml` en la rama). Los scripts `test:local/navegador/wasm` usan `BHUMI_LOCAL=1 node …` (`package.json`), sintaxis POSIX que falla con npm en Windows, que es el sistema del equipo.
- **Funciones que ya divergen entre modos:**

| Función | Servidor central | Doble clic |
|---|---|---|
| Login, roles, auditoría | Sí (`server/index.js:47-58`) | No (`navegador/runtime.js:7` monta solo `createApi`) |
| Control de versiones 409 | Sí (`server/concurrencia.js`) | No; lo reemplaza el candado de una pestaña (`almacenamiento.js:53-63`) |
| Tiempo real (SSE) | Sí (`server/eventos.js`) | No |
| Bodega sin costos (Fase 4) | Sí (`auth.js` en la rama) | No aplica |
| Cliente de Google | REST (`server/services/google.js`) | Biblioteca Maps JS (`navegador/google-navegador.js`), con otros campos (`durationMillis`), polilínea calculada localmente y errores detectados con expresiones regulares sobre el texto (`google-navegador.js:16-24`) |
| Clave de Google | Dos claves: servidor por IP y mapa por dominio (`config.js:15-21`) | Una sola clave **sin restricción de sitio web** (README, paso 5) |
| Restaurar `.db` | No (`api.js:974`) | Sí |
| Zona horaria | La del contenedor (UTC, ver ARQ-01) | La del PC (Chile) |
| `/api/salud` | Sí | No |

- **El cliente de Google del navegador no lo prueba la suite de la API.** En modo local las pruebas usan `createGoogle` (REST) con un Google simulado (`test/helpers.js:37`), no `createGoogleNavegador`. Ese adaptador solo tiene sus propias pruebas con mocks (`test/google-navegador.test.js`, 117 líneas).
- **Riesgos de datos del modo real:**
  - La base completa se exporta y se reescribe en IndexedDB 400 ms después de cada cambio (`almacenamiento.js:80-116`). El guardado al cerrar es "best effort" (`pagehide`/`visibilitychange`).
  - Borrar "Cookies y otros datos de sitios" elimina todo (lo dice el README). `navigator.storage.persist()` se pide, pero no se sabe si Chrome lo concede para `file://` (por confirmar).
  - Una sola pestaña (Web Locks).
  - La copia interna anterior se guarda como mucho una vez por hora.
  - Hay comentarios contradictorios sobre el origen `file://`: `almacenamiento.js:2` dice que los datos quedan "asociados a la carpeta" y el README dice que mover la carpeta no los pierde. Si el origen es común a todos los `file://`, una copia **más antigua** de BHUMI abriría la base nueva y correría su `prepararDb` viejo, sin ninguna protección contra versiones anteriores (por confirmar en Chrome/Edge).
- **¿El piloto lo necesita?** Solo como vuelta atrás. `docs/PILOTO.md` (rama Fase 4), sección 4, lo usa como plan B y la sección 5 dice: "Con el piloto aprobado, la versión de doble clic queda solo para el modo demostración".

**Costo/riesgo:** medio en mantenimiento (4 modos, scripts que no corren en Windows, dos clientes de Google). Alto como límite estratégico (impide una base asíncrona y obliga a todo a ser empaquetable). Alto en seguridad de costos por la clave sin restricción.

**Alternativas**

| Opción | Pros | Contras |
|---|---|---|
| a) Mantener los dos modos con igualdad de funciones | No hay que decidir nada ahora | Cada función nueva vale x2 en diseño y pruebas; las divergencias ya existentes crecen; bloquea PostgreSQL |
| b) Congelar el doble clic ahora y quitar el modo real tras el piloto, dejando solo la demo (o una demo servida por `npm run demo`) | Simplifica `api.js`, permite `async`, deja 1-2 modos de prueba y elimina el riesgo de la clave | Hay que migrar a quien hoy use el doble clic con datos reales (el camino JSON → servidor ya existe y está documentado) |
| c) Reemplazarlo por un servidor local (`npm start` en 127.0.0.1, como antes) para quien no tenga red | Una sola pila (Express + node:sqlite) | Vuelve a pedir Node en el PC, que fue justo lo que se quiso evitar |

**Recomendación: b), decidida después del piloto, con congelamiento desde hoy.** Durante el piloto no se agregan funciones al doble clic, y solo `npm test` + `test:wasm` son obligatorios para tocar `server/`. Con el piloto aprobado: quitar el modo real, dejar la demo y retirar `express` inyectable, los shims y los modos `local`/`navegador`.

### A2. SQLite (`node:sqlite` + `sql.js`) frente a PostgreSQL

**Situación actual**

- `server/db.js:2` usa `DatabaseSync` de `node:sqlite`, todavía experimental en Node 22/24 (por eso los scripts y el `CMD` llevan `--disable-warning=ExperimentalWarning`: `package.json`, `Dockerfile` en su última línea).
- La imagen usa una etiqueta flotante, `FROM node:24-alpine` (Dockerfile, línea 2). Cualquier menor nuevo de Node puede cambiar el comportamiento de un módulo experimental.
- `PRAGMA journal_mode = WAL; foreign_keys = ON` (`db.js:347`) y `busy_timeout = 5000` solo en disco (`db.js:340`). Hay Litestream 0.3.13 con 30 días de retención (`deploy/litestream.yml`).
- Todo es un solo proceso: los SSE viven en memoria (`eventos.js:30`) y los intentos de login en un `Map` (`auth.js:136`). Eso ya supone una sola instancia, lo que calza con SQLite.
- El SQL propio de SQLite está acotado: `AUTOINCREMENT` 18, `CURRENT_TIMESTAMP` 10, `PRAGMA` 5, `julianday` 4, `lastInsertRowid` 4, `strftime` 2, `INSERT OR` 2 y `VACUUM INTO` 1. El problema para migrar no es el dialecto, es la API síncrona (181 llamadas) y los triggers de versión (`db.js:380-396`).

**Costo/riesgo:** bajo para la escala (menos de 10 usuarios, 2 vehículos, decenas de despachos por semana; las consultas síncronas de reportes bloquean el event loop por milisegundos). Riesgo medio de que se rompa una API experimental al actualizar Node.

**Alternativas**

| Opción | Pros | Contras |
|---|---|---|
| a) Seguir con `node:sqlite`, fijando la versión de Node (imagen con digest o `node:24.x.y-alpine`) | Sin dependencias nativas, la misma API que el shim del navegador, cero trabajo | Experimental; los cambios de Node hay que validarlos a mano |
| b) `better-sqlite3` | Estable y muy usado; API síncrona casi idéntica (`prepare/get/all/run`); el cambio queda en `db.js` y en los parámetros con nombre | Binario nativo (compila en alpine o hay que usar una imagen con prebuilds) |
| c) PostgreSQL (+PostGIS) | Concurrencia real, varias instancias, consultas geográficas, herramientas de operación | Reescribir los 18 servicios a `async`, matar el doble clic, cambiar Litestream por `pg_dump`/WAL-G, una pieza más que operar. Es sobre-ingeniería para esta escala |

**Recomendación: a) ahora** (fijar la versión es un cambio de una línea) **y b) como plan de contingencia documentado.** PostgreSQL (issue #20) solo si el piloto muestra una necesidad concreta: más de una sede o servidor, consultas espaciales que `haversineKm` no cubra, o más de ~50 usuarios concurrentes. Hoy no hay evidencia de ninguna.

### A3. Interfaz en JS sin compilación ni framework

**Situación actual**

- 3.173 líneas no vacías en `public/js`. Hay una vista por archivo y la mayor es `rutas-nueva.js` (328), seguida de `configuracion.js` (264), `rutas-despachos.js` (181) y `rutas-entregas.js` (159). El tamaño es manejable.
- **Escape:** la plantilla `html\`\`` escapa `& < > " '` en cada interpolación (`public/js/lib.js:4-12`), y `raw()` se usa 29 veces, casi siempre con literales (`'selected'`, `'required'`). Los `innerHTML` directos (16) usan `esc()` o texto fijo (`app.js:132`, `maps.js:41`). `modal()` solo acepta HTML confiable si recibe un `Seguro` (`lib.js:108`), y todas las llamadas encontradas pasan `html\`\``. Está **bien resuelto**.
- **Estado:** hay un objeto `estado` compartido en `app.js` y cada vista vuelve a pedir sus datos y se repinta entera. El mapa recurso → secciones para el tiempo real está escrito a mano (`tiempo-real.js:9-22`; la Fase 4 tuvo que agregar `'hoy'` en tres lugares).
- **Tipos y calidad:** no hay `jsconfig.json`, ni ESLint, ni `@ts-check`. Hay apenas 11 anotaciones JSDoc en todo `server/` y `public/js/`.
- **Pruebas de interfaz:** no hay ninguna (pendiente en el issue #17).
- **Accesibilidad:** hay 93 atributos `aria-`/`role`, `campo()` asocia siempre `<label for>` (`lib.js:255-266`) y los avisos usan `role="status" aria-live` (`index.html`). Falta una trampa de foco en `modal()`: solo maneja Escape y el foco inicial (`lib.js:117-125`), así que Tab se escapa del diálogo.
- **Matiz importante:** "sin compilación" solo vale para el servidor. El doble clic **ya tiene un paso de esbuild** (`navegador/construir.mjs:391-407`), así que el argumento de "no queremos build" hoy no se sostiene del todo.

**Alternativas**

| Opción | Pros | Contras |
|---|---|---|
| a) Seguir así + `// @ts-check`/JSDoc + `tsc --noEmit --checkJs` en CI + ESLint | Cero cambios en tiempo de ejecución; atrapa errores de forma de datos entre la API y las vistas | Las anotaciones JSDoc son verbosas |
| b) Preact + htm (sin build, vía módulo ES) | Componentes, estado local y repintado parcial; `htm` también escapa | Una dependencia más; reescribir las vistas de a poco; el aporte es bajo con vistas de este tamaño |
| c) Lit o Alpine | Lit: componentes estándar. Alpine: interactividad declarativa | Lit usa su propio modelo de plantillas y Alpine choca con el repintado por `innerHTML` de hoy |
| d) TypeScript con build | Tipos completos | Agrega build al servidor; ganancia marginal sobre a) |

**Recomendación: a) ahora** (issue #19) **y una prueba de humo con Playwright** (login, crear despacho, marcar "Hoy", conflicto 409). Corregir la trampa de foco. Revisar b) después del piloto solo si "Hoy" y la planificación crecen en interacción (por ejemplo, edición en línea o arrastrar paradas).

### A4. Solo Google Maps como fuente de distancias

**Situación actual**

- **Dependencia total.** `planner.crearDespacho` siempre llama a `prepararPuntos` → `getMatrix` (`server/services/planner.js:349`), también para `alternativa = 'manual'` (`api.js:666-669`). Sin Google, y sin caché vigente, **no se puede guardar ningún despacho**. En el servidor, un fallo de Google se devuelve como 502 (`api.js:991`). Es coherente con la regla "solo datos reales", pero no hay modo de contingencia.
- **Caché (`route_cache`):**
  - La clave es lat/lng con 5 decimales, más un *bucket* (`planner.js:24`, `:62`).
  - Con tráfico (`usar_trafico = '1'` por defecto, `db.js:17`) el bucket es **la hora UTC de salida** (`planner.js:37`) y el TTL es de 12 h (`:63`). Al planificar "hoy", la salida se ajusta a ahora + 1 min (`:34`), así que el bucket cambia cada hora.
  - Sin tráfico, el bucket es fijo y el TTL es de 30 días.
  - **Si falta un solo par, se vuelve a comprar la matriz completa n×n** (`planner.js:74-88`), aunque el comentario de la línea 57 dice "consulta a Google solo lo que falta".
- **Facturación** (fuentes: https://developers.google.com/maps/billing-and-pricing/pricing y https://developers.google.com/maps/documentation/routes/usage-and-billing, consultadas el 2026-11-26). La matriz se cobra **por elemento** (orígenes × destinos), y `TRAFFIC_AWARE` la pasa al **SKU Pro**: 5.000 elementos gratis al mes y luego US$10 por 1.000. Essentials: 10.000 gratis y US$5 por 1.000. Compute Routes Pro: 5.000 gratis y US$10 por 1.000. Geocoding: 10.000 gratis y US$5 por 1.000. Dynamic Maps: 10.000 gratis y US$7 por 1.000.
  - *Estimación aproximada, depende del uso:* un plan de 20 paradas + base pide 21×21 = 441 elementos. Con tráfico, eso agota la cuota gratis en unas 11 matrices nuevas al mes. 2 a 3 planificaciones diarias "frescas" en 20 días hábiles serían ~18.000-26.000 elementos, del orden de **US$130-210 al mes** solo en matrices. Sin tráfico, y con la caché de 30 días, el costo bajaría mucho. Conviene medir el uso real en el piloto.
- **Sin internet:** en el servidor, sin internet no hay nada (pero el servidor siempre está en la nube). En el doble clic, la interfaz funciona y la planificación no.

**Alternativas**

| Opción | Pros | Contras |
|---|---|---|
| a) Google, con la caché corregida (pedir solo las filas u orígenes que faltan), tráfico opcional o solo al confirmar, y un presupuesto con alerta | Mejor geocodificación de direcciones chilenas, tráfico real, cero infraestructura | Costo variable; un fallo de facturación o de clave detiene la operación |
| b) OSRM, Valhalla o GraphHopper autohospedado con el extracto de Chile de OpenStreetMap (OSM), para matrices y rutas; Google solo para geocodificar | Matrices sin costo por elemento, funciona aunque Google caiga | Un servicio más que operar y actualizar; no tiene tráfico en tiempo real; el preprocesamiento de Chile exige varios GB de RAM (por confirmar según el motor) y es probable que no quepa en la VM de 2 GB del piloto |
| c) OpenRouteService o Mapbox (servicios gestionados) | Sin infraestructura | Otro proveedor y otras cuotas (no verificadas aquí); la calidad de las direcciones chilenas está por evaluar |
| d) Contingencia manual: guardar un despacho "sin cálculo de Google" con los km que ingrese el usuario, marcado como tal | Cumple "solo datos reales" (son km informados) y la operación no se detiene | Indicadores incompletos para ese despacho; requiere diseño (Opus) |

**Recomendación: a) + d).** Corregir la caché, dar a Admin un interruptor de tráfico (que ya existe y es solo de Admin: `auth.js:60`), poner una alerta de presupuesto (ya está documentada) y diseñar la contingencia d). Dejar b) para después del piloto, si la factura real supera unos US$50-100 al mes de forma sostenida (umbral a definir por la empresa).

### A5. Estructura de `server/`, límites entre servicios y modelo de datos

**Lo que está bien resuelto**
- Los servicios de cálculo son funciones puras, sin base de datos: `costs.js`, `optimizer.js`, `cubicaje.js`, `polilinea.js` y `periodo.js` tienen 0 llamadas a `db.prepare`, y por eso tienen buenas pruebas unitarias.
- La versión de los registros la mantienen triggers (`db.js:380-396`), no el código.
- Cierre ordenado y endpoint de salud (`index.js:38-46`, `:91-104`).
- Configuración validada (`config.js`).
- Dos claves de Google en el servidor.
- Transacciones sin `await` adentro (ninguna `tx(db, async …)`).

**Problemas**
- **`server/api.js`** tiene 1.002 líneas, 84 rutas y 101 sentencias SQL (más que todos los servicios juntos, que suman 80). Hay reglas de negocio dentro de los handlers:
  - el cierre de despacho, con la reprogramación y el odómetro (`api.js:673-705`);
  - el recálculo de costos al registrar gastos del despacho (`:726-737`);
  - la lista de datos faltantes (`:151-182`).
  - Además, la validación de que las entregas estén libres está separada de la escritura por un `await` (ARQ-02).
- **Acoplamiento a la forma de las rutas:** tres módulos interpretan las URL con sus propias listas de expresiones regulares: `auth.js:58-76` (permisos), `concurrencia.js:10-21` (qué tabla edita cada ruta) y `eventos.js:8-10` (qué recurso cambió). Agregar un endpoint exige tocar hasta 4 lugares, y ninguna prueba verifica que estén completos.
- **Modelo de datos:**
  - Las tablas en inglés (`vehicles`, `drivers`, `clients`, `trips`, `deliveries`…) conviven con tablas en español (`gastos`, `km_semanal`, `sectores`, `incidentes`, `usuarios`, `sesiones`, `auditoria`), y las columnas están casi todas en español (`db.js:36-318`).
  - `drivers` guarda también al personal de bodega (`rol` = `'preparador'`, `db.js:357`), y los `usuarios` del login no se relacionan con el personal.
  - `regiones` y `comunas` se siembran pero nunca se consultan (la única lectura es el `COUNT` de `db.js:439`; el catálogo sale de `server/data/chile.js`). `clients.comuna`, `sectores.comuna` e `incidentes.comuna` son texto libre, sin clave foránea.
  - Faltan índices en `deliveries(client_id)` (usada en 3 subconsultas por cliente, `api.js:323-327`), `trips(fecha)`, `trips(vehicle_id)`, `fuel_logs(vehicle_id)`, `delivery_items(delivery_id)` y `trip_events(stop_id)`.
  - Los estados (`estado`, `resultado`) no tienen `CHECK`.
  - Las fechas usan cuatro representaciones: UTC con texto `CURRENT_TIMESTAMP`, UTC con milisegundos (`auditoria.ts`), hora local `'AAAA-MM-DDTHH:MM'` (`trip_events.momento`) y milisegundos desde la época (`sesiones`, `route_cache`).
  - A esta escala nada de esto es urgente, pero conviene ordenarlo antes de que el piloto genere historia.
- **Migraciones:** `COLUMNAS_NUEVAS` (`db.js:354-373`) solo puede agregar columnas. No hay número de versión de esquema (no se usa `PRAGMA user_version`), los rellenos de datos son ad hoc (`faltabaResultado`, `db.js:399-417`), los triggers se recrean en cada arranque y no hay protección ante una base más nueva que el programa. Con el doble clic, esas migraciones corren en un número desconocido de navegadores, sin ninguna visibilidad.
- **¿Escala a lo que se necesita?** Sí. Un proceso, SQLite y SSE en memoria alcanzan para menos de 10 usuarios. No hay sobre-ingeniería en el servidor. La complejidad que sobra está en mantener dos runtimes, no en la arquitectura del servidor.

**Recomendación:**
1. Después de integrar la Fase 4, dividir `api.js` en routers por recurso, moviendo la lógica de los handlers a servicios (issue #12).
2. Centralizar en una sola tabla, junto a cada ruta, sus metadatos de permiso, versión y recurso.
3. Migraciones numeradas con `user_version` (issue #13), fijando a la vez la convención de nombres para lo nuevo, sin renombrar lo existente durante el piloto.
4. Los índices, ahora mismo (bajo costo).

---

## Parte B. Hallazgos

Cada hallazgo se verificó en el código. Lo que no se pudo confirmar se marca "por confirmar".

### ARQ-01 · El servidor en Docker calcula fechas y horas de salida en UTC
- **Severidad:** alta · **¿Bloquea el piloto?** sí
- **Evidencia:**
  - El `Dockerfile` y el `docker-compose.yml` no definen `TZ`.
  - `server/services/vehiculos-estado.js:25` `hoyLocal = … getTimezoneOffset()…` da UTC dentro del contenedor.
  - `server/api.js:74` `const hoyStr = () => new Date().toISOString().slice(0, 10);` es UTC **en todos los modos**. Se usa en `:194` (vigencia del precio), `:254` (carga de combustible) y `:553` (`fecha_realizada`).
  - `server/db.js:488` `precioCombustible` también usa la fecha UTC.
  - `server/services/planner.js:32` `new Date(\`${fecha}T${hora || '08:00'}:00\`)` interpreta la hora en la zona del proceso.
- **Impacto:**
  - En el servidor, una salida a las "08:00" se consulta a Google como 08:00 UTC (05:00 en Chile en verano), con lo que el tráfico, los tiempos y el costo del conductor quedan mal.
  - Desde las 21:00 hora de Chile (UTC−3), las fechas "de hoy" pasan al día siguiente: entregas realizadas, cargas, gastos, la semana de `km_semanal` y el precio vigente.
  - La pantalla "Hoy" (fecha local del navegador) puede no coincidir con lo que guarda el servidor.
  - El CI en ubuntu (UTC) no lo detecta.
- **Recomendación:**
  - Agregar `ENV TZ=America/Santiago` en el Dockerfile y en compose. Los binarios oficiales de Node traen ICU completo; verificar en alpine que `new Date().getTimezoneOffset()` dé 180.
  - Reemplazar `hoyStr` por `hoyLocal`.
  - Agregar una prueba con `TZ=UTC` y otra con `TZ=America/Santiago`.
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### ARQ-02 · Carrera al crear despachos: la misma entrega puede quedar en dos despachos
- **Severidad:** alta · **¿Bloquea el piloto?** no (es barato, conviene hacerlo antes)
- **Evidencia:**
  - `server/api.js:653-655` revisa que la entrega esté libre (`!s.trip_id`).
  - Luego `:666` hace `await planner.crearDespacho(...)`, que espera a Google (`planner.js:349` y `:365`).
  - Recién después `planner.js:397` ejecuta `UPDATE deliveries SET trip_id = ?, orden = ? WHERE id = ?`, **sin** `AND trip_id IS NULL`.
- **Impacto:** dos solicitudes (dos planificadores, o un doble envío) pasan la validación durante la espera a Google. Se crean dos `trips` con `trip_stops` de la misma entrega y la entrega queda apuntando al último. Los costos y KPI quedan duplicados y el primer despacho queda inconsistente. Afecta a ambos modos.
- **Recomendación:** dentro de la transacción de `planner.js:385`, volver a validar y hacer `UPDATE … WHERE id = ? AND trip_id IS NULL AND estado IN ('pendiente','reprogramada')`. Si `changes` = 0, abortar. Agregar una prueba con dos `POST /despachos` simultáneos.
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### ARQ-03 · (Fase 4) La hoja de ruta en PDF muestra los costos a Bodega
- **Severidad:** alta · **¿Bloquea el piloto?** sí (contradice el criterio de `docs/PILOTO.md`: "Bodega no ve montos en pesos en ninguna pantalla")
- **Evidencia:**
  - En la rama, `server/auth.js` incluye `/^\/reportes\/despacho\/\d+\.pdf$/` en `BODEGA_LEE`.
  - El filtro solo envuelve `res.json` (`res.json = (cuerpo) => json(sinCostos(cuerpo))`).
  - `server/services/reportes.js:412-417` escribe en el PDF "Costo total", "Costo por kilómetro", "Combustible (precio $…/L)", etc.
  - `test/bodega.test.js` (en la rama) no revisa los PDF.
- **Impacto:** Bodega descarga la hoja de ruta y ve todos los costos del despacho. Es el mismo problema de fondo que ya se anota en el plan: una lista negra que falla abierta (issue #16).
- **Recomendación:** pasar `{ conCostos: rol !== 'bodega' }` a `pdfHojaDeRuta` y omitir esa sección. A mediano plazo, una lista permitida de campos por rol para JSON, PDF y Excel. Agregar una prueba que extraiga el texto del PDF como Bodega.
- **Esfuerzo:** bajo · **Modelo:** Sonnet (Opus para el diseño de la lista permitida)

### ARQ-04 · La caché de rutas vuelve a comprar la matriz completa y, con tráfico, cambia cada hora
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:**
  - `planner.js:78` `if (!row || now - row.fetched_at > ttl) { incompleto = true; break; }` y `:88` `google.routeMatrix(puntos, …)` con **todos** los puntos.
  - El bucket con tráfico es `t${iso.slice(0, 13)}` (`:37`) y la salida se fuerza a ahora + 1 min (`:34`).
  - El tráfico está activado por defecto (`db.js:17` `usar_trafico: '1'`).
  - El comentario de `:57` dice "consulta a Google solo lo que falta".
- **Impacto:** agregar una parada a un plan de 20 cobra de nuevo ~441 elementos en el SKU Pro (5.000 gratis al mes y luego US$10 por 1.000, según el sitio de Google al 2026-11-26). Replanificar el mismo día en otra hora vuelve a cobrar. Ver la estimación aproximada en A4.
- **Recomendación:** pedir solo los orígenes o destinos que faltan (por filas). Usar tráfico solo al guardar el despacho, o dejarlo como una opción explícita. Registrar en una tabla de uso los elementos pedidos por día, para medir el gasto en el piloto.
- **Esfuerzo:** medio · **Modelo:** Sonnet

### ARQ-05 · Sin Google no se puede guardar ningún despacho, ni siquiera en orden manual
- **Severidad:** media · **¿Bloquea el piloto?** no (sí es un riesgo de operación)
- **Evidencia:** `api.js:666` → `planner.js:349` `await prepararPuntos(stops, …)` siempre, sin camino alternativo. Un `GoogleError` se devuelve como 502 (`api.js:991`).
- **Impacto:** si la clave se rechaza, la facturación falla, se acaba la cuota o Google cae, bodega se queda sin despachos registrados ese día y los tiempos y KPI se pierden.
- **Recomendación:** diseñar un "despacho sin cálculo de Google" con km y tiempo informados por el usuario y marcados como tales (sin costo calculado si faltan datos), coherente con la regla de datos reales. Documentarlo en `PILOTO.md`.
- **Esfuerzo:** medio · **Modelo:** Opus

### ARQ-06 · El odómetro mezcla kilómetros planificados con lecturas reales
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:**
  - `server/api.js:701` `UPDATE vehicles SET km_actual = km_actual + ? …` usa `t.distancia_km` (km estimados por Google) al cerrar el despacho.
  - Las lecturas reales solo suben el valor si es mayor (`api.js:255`, `:766`, `:806`: `km_actual < ?`).
  - `vehiculos-estado.js:45-49` toma el máximo de las lecturas y llama a `km_actual` "Kilometraje informado en el vehículo".
- **Impacto:** si los km reales son menos que los estimados, el odómetro queda inflado y las lecturas reales se ignoran hasta superarlo. La mantención preventiva y el desgaste se adelantan, lo que contradice "solo datos reales".
- **Recomendación:** no sumar km estimados a `km_actual`. Si se quiere una proyección, mostrarla aparte como "estimado desde la última lectura".
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### ARQ-07 · `server/api.js` concentra rutas, SQL y reglas de negocio
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:** 1.002 líneas, 84 rutas (`r.get/post/put/delete`) y 101 `db.prepare` (contra 80 en los 18 servicios). Lógica de dominio en `:673-705` (cierre), `:726-737` (gastos del despacho) y `:151-182` (datos faltantes). Las reglas por ruta están repetidas en `auth.js:58-76`, `concurrencia.js:10-21` y `eventos.js:8-10`.
- **Impacto:** los cambios chocan en un solo archivo (el PR de la Fase 4 ya toca rutas, permisos y tiempo real por separado). Cuesta probar las reglas sin HTTP y es fácil olvidar una de las cuatro listas.
- **Recomendación:** después de integrar la Fase 4, crear routers por recurso (`rutas/vehiculos.js`, `rutas/despachos.js`…) y mover `cerrarDespacho` y `registrarGastosDespacho` a servicios. Definir junto a cada ruta su metadato `{ permiso, tabla_version, recurso }` y generar desde ahí las tres listas. Agregar una prueba que falle si una ruta de escritura no tiene metadatos.
- **Esfuerzo:** alto · **Modelo:** Opus

### ARQ-08 · Migraciones sin versión, solo `ADD COLUMN`
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:** `db.js:354-373` (`COLUMNAS_NUEVAS`), `:398-418` (relleno condicionado por `faltabaResultado`) y `:380-396` (triggers que se borran y recrean en cada arranque). No se usa `PRAGMA user_version`.
- **Impacto:** no se puede renombrar ni borrar columnas, cambiar tipos ni agregar `CHECK` o índices únicos con datos. No se sabe en qué versión está una base. Una versión vieja del programa puede abrir una base nueva sin aviso. En el doble clic, cada navegador migra por su cuenta.
- **Recomendación:** migraciones numeradas e idempotentes con `user_version`, todas en una transacción, y negarse a abrir una base con versión mayor que la del programa. Litestream o un `VACUUM INTO` antes de migrar en el servidor (issue #13).
- **Esfuerzo:** medio · **Modelo:** Opus

### ARQ-09 · `node:sqlite` experimental con imagen Docker sin fijar
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:** `Dockerfile:2` `FROM node:24-alpine`. `package.json` y el `CMD` usan `--disable-warning=ExperimentalWarning`. `db.js:2` `import { DatabaseSync } from 'node:sqlite'`.
- **Impacto:** un `docker compose up --build` en medio del piloto puede traer un Node con cambios en un módulo experimental, sin que nadie haya corrido pruebas con esa versión.
- **Recomendación:** fijar `node:24.x.y-alpine` (o un digest) y actualizarlo a propósito, con el CI de por medio. Documentar `better-sqlite3` como reemplazo si `node:sqlite` cambia.
- **Esfuerzo:** bajo · **Modelo:** Haiku

### ARQ-10 · El doble clic multiplica las pruebas y deja su cliente de Google fuera de la suite
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:**
  - En modo local, `test/helpers.js:37` usa `createGoogle` (REST) y no `createGoogleNavegador`.
  - 27 de 109 pruebas se omiten fuera del modo HTTP (`auth.test.js:7`, `servidor-central.test.js:56`, `tiempo-real.test.js:9`).
  - Los scripts `test:local/navegador/wasm` usan `BHUMI_LOCAL=…` (`package.json`), que no funciona en Windows (issue #2).
  - `navegador/entrada.js:14-15` empaqueta `test/fake-google.js` y `test/demo-datos.js` en el producto.
- **Impacto:** se corren 355 pruebas por cambio y aun así el camino real de Google del doble clic (`google-navegador.js`, con errores detectados por regex en `:16-24`) solo tiene pruebas con mocks. El equipo en Windows no puede correr 3 de los 4 modos.
- **Recomendación:** congelar el doble clic (ver A1). Mientras exista, usar `cross-env` o un `node --env-file` en los scripts y mover el Google simulado y los datos demo a `navegador/demo/`.
- **Esfuerzo:** bajo · **Modelo:** Haiku (scripts), Sonnet (mover módulos)

### ARQ-11 · La clave de Google del doble clic no tiene restricción de sitio y viaja en el respaldo `.db`
- **Severidad:** alta (si el doble clic real sigue en uso) · **¿Bloquea el piloto?** no (el servidor usa dos claves restringidas)
- **Evidencia:**
  - README, "Configurar Google Maps", paso 5: "No uses restricción por sitio web (HTTP referrer)".
  - `google-navegador.js:18` exige que la clave no tenga esa restricción.
  - La clave se guarda en `settings.google_api_key` (`db.js:15`) dentro de IndexedDB y en el respaldo `.db` (README, "Respaldos").
- **Impacto:** cualquiera que vea la clave (en el tráfico del navegador o en un `.db` compartido) puede consumir Maps, Routes y Geocoding a cuenta de la empresa. Solo la limitan las cuotas que se configuren en Google Cloud.
- **Recomendación:** pasar a la empresa al servidor central (que ya resuelve esto). Mientras el doble clic se use con datos reales: cuotas diarias por API en Google Cloud Console y alerta de presupuesto obligatoria (hoy es una recomendación). Decidir el retiro del modo real tras el piloto.
- **Esfuerzo:** bajo (cuotas) / medio (retiro) · **Modelo:** Opus (decisión), Haiku (documentación)

### ARQ-12 · Persistencia del doble clic: reescritura completa y guardado al cerrar sin garantía
- **Severidad:** media · **¿Bloquea el piloto?** no
- **Evidencia:**
  - `navegador/almacenamiento.js:80-116`: `exportar()` serializa toda la base en cada guardado (400 ms después del último cambio).
  - Al cerrar, `pagehide`/`visibilitychange` inician una transacción que el navegador puede cortar.
  - La copia anterior se guarda como mucho una vez por hora.
  - `:2` y el README se contradicen sobre si los datos dependen de la carpeta.
- **Impacto:** si se cierra la pestaña o se cae el PC dentro de esa ventana, se pierden los últimos cambios. Si el navegador limpia el almacenamiento, se pierde todo. Por confirmar: si el origen `file://` es común a todas las copias, una carpeta vieja de BHUMI abriría y migraría a medias la base nueva.
- **Recomendación:** congelar el modo (A1). Si se mantiene, guardar una "versión del programa" en la meta y negarse a abrir con una versión anterior. Probar en Chrome y Edge qué origen usa `file://`.
- **Esfuerzo:** medio · **Modelo:** Sonnet

### ARQ-13 · Deuda del modelo de datos: nombres, tablas sin uso, índices y relaciones
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:**
  - Tablas en inglés y en español (`db.js:36-318`).
  - `regiones` y `comunas` sin lecturas (la única es `db.js:439`).
  - `comuna` como texto libre en `clients`, `sectores` e `incidentes`.
  - `drivers.rol = 'preparador'` (`db.js:357`) para el personal de bodega.
  - `usuarios` sin relación con `drivers`.
  - Sin índices en `deliveries(client_id)`, `trips(fecha)`, `trips(vehicle_id)`, `fuel_logs(vehicle_id)`, `delivery_items(delivery_id)` y `trip_events(stop_id)`.
  - Estados sin `CHECK`.
  - Cuatro formatos de fecha y hora.
- **Impacto:** a esta escala, el rendimiento no se nota. Sí afecta la legibilidad y lo que se puede analizar después: no se puede saber qué usuario de bodega corresponde a qué persona del KPI.
- **Recomendación:**
  - Ahora: agregar los índices (sin riesgo, con `CREATE INDEX IF NOT EXISTS`).
  - Después del piloto, con migraciones versionadas: relacionar `usuarios` con el personal, borrar `regiones`/`comunas` o usarlas como clave foránea, agregar `CHECK` a los estados y fijar una convención única de nombres para lo nuevo.
- **Esfuerzo:** bajo (índices) / medio (resto) · **Modelo:** Haiku (índices), Opus (rediseño)

### ARQ-14 · (Fase 4) Una prueba de Bodega pasa sin probar lo que dice
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:**
  - `test/bodega.test.js` (en la rama) envía `PUT /km-semanal` con `{ vehicle_id, semana: '2026-11-23', km_inicio: 1000, km_fin: 1250 }`.
  - La API espera `semana_inicio`/`fecha` y `km` (`server/api.js:798-801`), así que responde 400 ("Indica la semana…").
  - La prueba solo revisa que el error no traiga campos de costo.
- **Impacto:** el escenario "se quitan los costos de lo que devuelve una escritura de Bodega" queda sin probar de verdad.
- **Recomendación:** usar los campos correctos y comprobar `status === 200`.
- **Esfuerzo:** bajo · **Modelo:** Haiku

### ARQ-15 · Interfaz sin tipos, lint ni pruebas, y modales sin trampa de foco
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:**
  - No hay `jsconfig.json` ni configuración de ESLint, y hay 11 anotaciones JSDoc en todo el código.
  - No hay pruebas de `public/js`.
  - `lib.js:117-125`: el modal maneja Escape y el foco inicial, pero no el Tab.
  - El mapa recurso → secciones de `tiempo-real.js:9-22` está escrito a mano.
- **Impacto:** los errores de forma de datos entre la API y las vistas solo aparecen en uso. Con teclado, el foco se sale del diálogo.
- **Recomendación:** `// @ts-check` + JSDoc en `lib.js` y en los tipos de respuesta clave, `tsc --noEmit` y ESLint en CI (issues #3 y #19), una prueba de humo con Playwright (issue #17) y una trampa de foco en `modal()`.
- **Esfuerzo:** medio · **Modelo:** Sonnet

### ARQ-16 · Los avisos de tiempo real llegan a todos los roles
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:** `server/eventos.js:62-65` `emitir` escribe a todos los clientes, con `usuario.nombre`, recurso y acción. También emite cambios de `usuarios` y `settings` (`eventos.js:75`) a Bodega.
- **Impacto:** Bodega se entera de que hubo cambios en usuarios o en la configuración, y de quién los hizo. No recibe datos, solo metadatos. Ya está anotado en el issue #15.
- **Recomendación:** filtrar por `permitido(rol, 'GET', …)` del recurso antes de escribir a cada cliente.
- **Esfuerzo:** bajo · **Modelo:** Sonnet

### ARQ-17 · El respaldo `.db` del servidor incluye usuarios, sesiones y auditoría
- **Severidad:** baja · **¿Bloquea el piloto?** no
- **Evidencia:** `server/api.js:965-967` `VACUUM INTO` copia la base completa. Solo Admin puede descargarla (`auth.js:58`).
- **Impacto:** el archivo descargado contiene los hashes scrypt, los hashes SHA-256 de las sesiones activas y la auditoría. Si se filtra, permite intentar romper las contraseñas por fuerza bruta fuera de línea. Las sesiones no se pueden reusar, porque se guarda el hash y no el token.
- **Recomendación:** en el servidor, borrar `sesiones` en la copia temporal antes de entregarla, o depender solo de Litestream y del JSON. Advertirlo en la interfaz.
- **Esfuerzo:** bajo · **Modelo:** Haiku

---

### Orden sugerido

1. **Antes del piloto:** ARQ-01, ARQ-03, ARQ-02, ARQ-09 y ARQ-14. Todos son de esfuerzo bajo.
2. **Durante el piloto:** medir el gasto de Google (ARQ-04), diseñar la contingencia ARQ-05 y corregir ARQ-06.
3. **Después del piloto:** decidir el retiro del doble clic (A1, ARQ-10, ARQ-11, ARQ-12). Si se retira, ARQ-07, ARQ-08 y ARQ-13 se vuelven más simples, porque se puede pasar a una API `async` y a migraciones con un solo destino.
