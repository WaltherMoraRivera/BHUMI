# Auditoría de lógica de negocio e integridad de datos · BHUMI

- **Alcance:** rama `main` (2be81eb). De la Fase 4 (`origin/fase-4/bodega-piloto`) solo cambia `server/auth.js` en el servidor (filtro `sinCostos` para Bodega y `reglasPara`); el resto son pantallas, docs y CI. Lo revisé con `git show`/`git diff` sin checkout.
- **Método:** leí los servicios, `db.js`, `api.js`, `concurrencia.js`, el adaptador `navegador/sqlite-wasm.js` y las specs `openspec/specs/*`. Cada sospecha la probé con scripts en `auditoria/pruebas-datos\`. Los scripts importan los módulos del repo con rutas absolutas y corren la API completa en proceso (enrutador local + node:sqlite en memoria + el Google simulado del repo). Al lado de cada script queda su salida (`NN-salida.txt`).
- **No modifiqué el repo.** Lo que no pude verificar va marcado **por confirmar**.

## Resumen

| ID | Título | Severidad | ¿Bloquea piloto? |
|---|---|---|---|
| DAT-01 | El servidor Docker corre en UTC: el tráfico se consulta 3 h antes y "hoy" pasa a mañana después de las 21:00 | alta | **sí** |
| DAT-02 | `hoyStr()` y `precioCombustible()` usan la fecha UTC en los dos modos: fechas por defecto y precio vigente de "mañana" desde las 21:00 | media | no (se corrige junto con DAT-01) |
| DAT-03 | Dos solicitudes simultáneas meten la misma entrega en dos despachos | alta | no (conviene antes) |
| DAT-04 | Restaurar un JSON parcial o equivocado borra todos los datos | alta | no (conviene antes) |
| DAT-05 | Basta una línea sin peso para que se apague el control de capacidad: se guarda una carga de 1500 kg en un vehículo de 1000 kg | alta | no (conviene antes) |
| DAT-06 | Los gastos agregados al despacho descuadran el reparto por cliente, se cuentan como "ahorro potencial de rutas" y se aceptan negativos | media | no |
| DAT-07 | Un despacho se guarda con el costo a medias (desgaste y conductor en $0) y después se trata como completo | media | no |
| DAT-08 | Consolidación: grupos encadenados de cientos de entregas, "despachos evitables" inflados, se ignora el límite de 25 paradas y el cálculo bloquea el servidor | media | no |
| DAT-09 | No se pueden anotar horas que cruzan la medianoche ni una preparación hecha el día anterior | media | no |
| DAT-10 | El consumo real "estanque lleno" no sabe qué cargas fueron a estanque lleno ni filtra odómetros mal digitados | media | no |
| DAT-11 | Falta el índice `deliveries(client_id)`: GET /clientes crece en forma cuadrática (15 s con 1000 clientes) | media | no |
| DAT-12 | Al cerrar un despacho sus km planificados se suman al odómetro aunque la lectura real ya los incluía | baja | no |
| DAT-13 | El km semanal registrado no suma los km de los despachos; el promedio semanal los ignora | baja | no |
| DAT-14 | Validación de entradas: fechas sin formato, montos negativos, vehículos con rendimiento o capacidad negativos, error 500 por clave foránea | baja | no |
| DAT-15 | Se guardan despachos con vehículos "fuera de servicio" o desactivados | baja | no |
| DAT-16 | Borrar un vehículo sin despachos elimina en cascada sus cargas de combustible y km (costo real histórico) | baja | no |
| DAT-17 | La tabla por vehículo de Control de costos omite los vehículos desactivados y no cuadra con el total | baja | no |
| DAT-18 | Las horas extra se calculan por despacho, no por la jornada diaria del conductor | baja | no |
| DAT-19 | Fase 4: `sinCostos` deja pasar el desgaste en $/km del vehículo hacia Bodega | baja | no |
| DAT-20 | Respaldo: los ajustes que no trae el archivo no se reemplazan, un error de clave foránea responde con un 500 genérico, y el .db del navegador acepta cualquier SQLite | baja | no |
| DAT-21 | Supuestos y valores por defecto poco visibles (08:00, 2 semanas, radio 5 km, promedio histórico, `vida_util_anios` ignorada, "orden de selección") | baja | no |
| DAT-22 | Cantidades no enteras siempre dan "no cabe" en el cubicaje | baja | no |
| DAT-23 | El personal de un despacho se puede cambiar después del cierre | baja | no |

---

## Hallazgos

### DAT-01 · El servidor Docker corre en UTC: el tráfico se consulta 3 h antes y "hoy" pasa a mañana desde las 21:00
- **Severidad:** alta · **Bloquea el piloto:** sí
- **Evidencia:**
  - `Dockerfile:2-4`: `FROM node:24-alpine` y `ENV NODE_ENV=production`. No hay `TZ`, ni en `Dockerfile` ni en `docker-compose.yml`, así que el proceso corre en UTC.
  - `server/services/planner.js:32` arma la salida con `new Date(\`${fecha}T${hora || '08:00'}:00\`)`, que se interpreta en la zona del proceso.
  - `server/services/vehiculos-estado.js:25` calcula `hoyLocal` con el desfase del proceso.
  - Script `04-fechas-tz.mjs`, con el reloj fijado en 26-11-2026 22:30 hora de Chile:
    - con `TZ=UTC`: `hoyLocal() = 2026-11-27`, y al planificar para el 28-11 a las 08:00 se envía `departureTime = 2026-11-28T08:00:00Z`, que son las **05:00** en Chile;
    - con la zona de Chile: `hoyLocal() = 2026-11-26` y `departureTime = …T11:00:00Z`, que son las 08:00 (correcto).
- **Impacto en el negocio:**
  - En el servidor del piloto, las rutas con tráfico se calculan con la congestión de las 05:00 y no con la de la hora real de salida. Salen subestimados el tiempo de ruta, el costo del conductor (que es proporcional a los minutos), la alternativa B "menor tiempo" y la "ruta estimada" de los KPI. La magnitud depende de Google: **por confirmar** con la clave real.
  - Entre las 21:00 y las 24:00 (horario de verano; desde las 20:00 en invierno), `hoyLocal` devuelve la fecha de mañana. Eso afecta:
    - la fecha por defecto de los gastos (`api.js:770`);
    - la mantención "vencida por fecha";
    - el mes por defecto del Panel, Control de costos y KPI: el 30 de noviembre a las 22:00 se abre diciembre vacío;
    - la semana actual del domingo en la noche.
  - El piloto pregunta justamente "¿los costos y kilómetros coinciden con lo real?", y con esto no pueden coincidir.
- **Recomendación:**
  - Agregar `TZ: America/Santiago` en `docker-compose.yml` (o `ENV TZ=America/Santiago` en el `Dockerfile`). Node trae los datos ICU, así que no debería necesitar `tzdata`; **por confirmar** con `docker compose exec app node -e "console.log(new Date().getTimezoneOffset())"`, que debe dar 180.
  - Mejor aún, no depender de la zona del proceso: calcular "hoy" y la hora de salida con `Intl.DateTimeFormat('en-CA', { timeZone: 'America/Santiago' })` y una función común `fechaChile()`.
  - Agregar una prueba con `TZ=UTC`.
- **Esfuerzo:** bajo · **Modelo sugerido:** Sonnet
- **Specs afectadas:** `planificacion-rutas` (Cuatro alternativas), `integracion-google`, `flota-vehiculos` (Mantención preventiva), `panel-analisis`, `modos-ejecucion`.

### DAT-02 · `hoyStr()` y `precioCombustible()` toman la fecha UTC también en el navegador
- **Severidad:** media · **Bloquea el piloto:** no (va con la corrección de DAT-01)
- **Evidencia:**
  - `server/api.js:74`: `const hoyStr = () => new Date().toISOString().slice(0, 10);`. Se usa como fecha por defecto en la vigencia de un precio (`:194`), en la fecha de una carga de combustible (`:254`) y en la fecha de una entrega realizada (`:553`).
  - `server/db.js:488`: en `precioCombustible`, `const f = fecha ?? new Date().toISOString().slice(0, 10);`.
  - Script `04-fechas-tz.mjs`, en la zona de Chile (modo doble clic) a las 22:30 del 26-11: `hoyStr = 2026-11-27`. Con un precio nuevo vigente desde el 27, el "precio vigente hoy" ya devuelve **1400 en vez de 1300**.
- **Impacto en el negocio:**
  - Las cargas de combustible se suelen anotar al final del día. Una carga hecha el 30 a las 21:30 sin fecha explícita queda con fecha del día 1 y su monto pasa al mes siguiente: el combustible real del mes cierra más bajo y el del siguiente más alto.
  - Durante 3 horas al día, el costo/km de cada vehículo, la flota y la sugerencia de vehículo económico usan el precio de mañana.
- **Recomendación:** reemplazar `hoyStr` y el valor por defecto de `precioCombustible` por la `fechaChile()` de DAT-01. Además, `analytics.js:6` (`mesActual`) y `public/js/componentes.js:62` usan `toISOString` para el mes y para "hasta".
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Specs afectadas:** `costos-transporte` (Precio de combustible con vigencia, Control de costos).

### DAT-03 · Dos solicitudes simultáneas meten la misma entrega en dos despachos
- **Severidad:** alta · **Bloquea el piloto:** no (conviene corregirlo antes, porque el piloto es justamente multiusuario)
- **Evidencia:**
  - `server/api.js:653-655` revisa que la entrega esté pendiente y sin despacho. Después, `await planner.crearDespacho(...)` (`:666`) espera a Google (matriz y `computeRoute`).
  - La transacción de `planner.js:385-403` inserta y hace `UPDATE deliveries SET trip_id` sin volver a comprobar el estado.
  - Script `10-integridad.mjs`: dos `POST /despachos` en paralelo con la entrega 1 responden **201 y 201**. La entrega queda como parada de los despachos [1, 2] con `trip_id = 2`. Si se cancela el despacho 2, la entrega vuelve a "pendiente" aunque sigue siendo parada del despacho 1, y se puede planificar una tercera vez.
- **Impacto en el negocio:**
  - El mismo pedido sale en dos camiones, con km y costo contados dos veces en Control de costos y en el Panel.
  - Al cerrar el despacho 1, esa parada queda sin resultado (`api.js:688` solo recorre las entregas cuyo `trip_id` es ese despacho), así que el cumplimiento queda mal.
  - El botón de la interfaz se desactiva (`rutas-nueva.js:273`), pero eso no evita el caso de dos planificadores trabajando a la vez.
- **Recomendación:** dentro de la transacción de `crearDespacho`, volver a leer las entregas y abortar con 409 si alguna ya tiene `trip_id` o no está pendiente. Opcional: un índice único parcial `trip_stops(delivery_id)` limitado a despachos no cancelados, o hacer el control con `UPDATE deliveries … WHERE trip_id IS NULL` y comparar `changes`.
- **Esfuerzo:** bajo · **Modelo sugerido:** Sonnet
- **Specs afectadas:** `tiempo-real-concurrencia`, `planificacion-rutas`, `despachos-tiempos-kpi` (Ciclo de vida).

### DAT-04 · Restaurar un JSON parcial o equivocado borra todos los datos
- **Severidad:** alta · **Bloquea el piloto:** no (pero la migración del piloto se hace justamente con un JSON)
- **Evidencia:**
  - `server/services/respaldo.js:37` acepta cualquier objeto que tenga **una** tabla conocida como arreglo: `!TABLAS_RESPALDO.some((t) => Array.isArray(datos[t]))`.
  - Después borra todas las tablas (`:43`) y trata como vacías las que faltan (`:46`).
  - Script `05-respaldo.mjs`:
    - `{"settings": []}` responde 200 y deja **vehicles 0, clients 0, deliveries 0, trips 0, fuel_prices 0…**. Hasta se pierden los dos vehículos base.
    - `{"clients":[{"nombre":"X"}]}` responde 200 y deja todo vacío salvo un cliente.
- **Impacto en el negocio:** elegir por error otro JSON (un export a medias, otro archivo con la clave `clients`) borra la base entera. En el servidor solo se recupera con Litestream, que es opcional; en el navegador, solo si hay un respaldo descargado. Además contradice la spec ("un archivo que no es un respaldo válido MUST rechazarse sin tocar nada").
- **Recomendación:**
  - Exigir la firma del archivo (`version` entero y `generado`) y la presencia de las tablas núcleo (`vehicles`, `clients`, `trips`, `deliveries`).
  - Rechazar si faltan tablas que la versión del archivo debería traer.
  - Mostrar antes de reemplazar un resumen de conteos ("se reemplazarán 0 vehículos por…") y pedir confirmación.
- **Esfuerzo:** bajo · **Modelo sugerido:** Sonnet
- **Spec afectada:** `respaldo-datos` (Restauración segura).

### DAT-05 · Basta una línea sin peso para que se apague el control de capacidad
- **Severidad:** alta · **Bloquea el piloto:** no (se mitiga si todos los productos tienen peso y medidas)
- **Evidencia:**
  - `server/services/stops.js:10`: si algún ítem no tiene peso, la entrega queda con `peso_kg = null`.
  - `planner.js:137,187,192`: `pesoConocido` exige que todas las paradas tengan peso; si no, `excede: null`, y `crearDespacho` (`:355`) solo bloquea cuando `excede === true`.
  - El cubicaje (`api.js:659-663`) solo bloquea si todos los productos tienen medidas (`cub.ok`).
  - Script `12-capacidad-parcial.mjs`, con 30 sacos de 50 kg (1500 kg conocidos) más 1 accesorio sin peso en un vehículo de 1000 kg: **201, despacho guardado con `peso_kg = null`**. La misma carga sin el accesorio responde 422 ("supera la capacidad").
- **Impacto en el negocio:** un despacho con 50 % de sobrecarga pasa sin aviso de bloqueo, con riesgo de seguridad vial y de multas. Tampoco entra a la utilización por peso del Panel.
- **Recomendación:**
  - Llevar el **peso mínimo conocido** (la suma de lo que tiene peso) y bloquear cuando ese mínimo ya excede la capacidad.
  - Cuando falte algún peso, exigir confirmación explícita ("peso incompleto: N kg conocidos de …").
  - Hacer lo mismo en `cubicar()`, que hoy cuenta los productos sin peso como 0 (`cubicaje.js:152`).
- **Esfuerzo:** bajo · **Modelo sugerido:** Sonnet
- **Specs afectadas:** `planificacion-rutas` (Bloqueo por capacidad), `cubicaje-consolidacion`.

### DAT-06 · Los gastos agregados al despacho descuadran el reparto, inflan el "ahorro de rutas" y aceptan negativos
- **Severidad:** media · **Bloquea el piloto:** no
- **Evidencia:**
  - `server/api.js:726-737` (`POST /despachos/:id/gastos`) recalcula `costo_otros` y `costo_total`, pero **no** vuelve a repartir `trip_stops.costo_asignado`. Además usa `n()`, que acepta negativos.
  - `server/services/panel.js:182-187` calcula el ahorro de rutas como `t.costo_total - mejor.total`, donde `mejor.total` se calculó al planificar, sin esos gastos.
  - Script `02-despacho-costos.mjs`:
    - (A) al crear: total 23 982 y suma asignada 23 982, cuadra;
    - (B) con peajes 4500 y estacionamientos 1500: total 29 982 y suma asignada 23 982, **una diferencia de $6000**;
    - (C) `peajes: -20000` responde 200 y deja un total de **$3 982**;
    - (F) el Panel muestra un "ahorro potencial de rutas" de **$6000** en un despacho donde las cuatro alternativas costaban lo mismo que el orden manual (ahorro real $0).
- **Impacto en el negocio:**
  - El costo por cliente y por sector (Panel, ficha del cliente, concentración por comuna) queda bajo en exactamente los peajes y estacionamientos. Esto rompe "la suma asignada = total".
  - El ahorro potencial de rutas se infla con cada peaje registrado después de planificar (en el ejemplo, 100 % falso).
  - Un monto negativo puede dejar costos casi en cero.
- **Recomendación:**
  - Al guardar gastos, volver a repartir con `repartirCostoPorCliente`: los gastos van al "resto" que se divide en partes iguales.
  - En `ahorroRutas`, comparar costos sin extras (`costo_total - peajes - estacionamientos - otros_gastos` contra el total de la alternativa sin los extras planificados), o guardar en `plan_json` los extras usados.
  - Rechazar montos negativos.
- **Esfuerzo:** bajo · **Modelo sugerido:** Sonnet
- **Specs afectadas:** `costos-transporte` (Asignación de costos a clientes), `panel-analisis`.

### DAT-07 · Un despacho se guarda con el costo a medias y después se trata como completo
- **Severidad:** media · **Bloquea el piloto:** no (si la configuración está completa, no ocurre)
- **Evidencia:**
  - `costs.js:121-126,147-148`: solo el rendimiento y el precio hacen `calculable = false`. Los componentes de desgaste que faltan suman 0 (`:52`) y sin costo por hora el conductor queda en 0 (`:154-156`).
  - `planner.js:352` solo bloquea si `!calculable`.
  - En `/setup`, el desgaste y el costo del conductor aparecen como "opcional" (`api.js:169,172`).
  - Script `03-costos-puros.mjs` (3): un vehículo sin desgaste y sin costo por hora responde **201** y queda guardado con `costo_total 6957`, `costo_desgaste 0` y `costo_otros 0`. Con los datos completos, la misma ruta cuesta 23 982: **el total guardado es 71 % más bajo**.
- **Impacto en el negocio:** Control de costos, el Panel (costo por km y por entrega) y la proyección suman esos despachos como si fueran costos completos. Los avisos solo aparecen en la respuesta al planificar. Esto contradice "si falta un dato, el indicador queda en blanco".
- **Recomendación:**
  - Guardar en `trips` una marca `costo_completo` (o la lista de faltantes).
  - En los totales, mostrar "N despachos con costo incompleto", o excluirlos del costo por km "real vs modelado".
  - O bien, exigir los componentes requeridos antes de guardar, que es lo que la spec "Bloqueo por… costo incalculable" sugiere.
- **Esfuerzo:** medio · **Modelo sugerido:** Sonnet
- **Specs afectadas:** `datos-reales` (Sin cifras inventadas), `costos-transporte`, `planificacion-rutas` (Bloqueo por costo incalculable).

### DAT-08 · Consolidación: grupos encadenados, "evitables" inflados, sin límite de 25 paradas y cálculo que bloquea el servidor
- **Severidad:** media · **Bloquea el piloto:** no (depende del volumen de entregas pendientes)
- **Evidencia:**
  - `optimizer.js:151-157`: `agruparCercanos` usa componentes conectados (enlace simple). Script `01-optimizador.mjs` (7): con puntos en línea cada 4,03 km y un radio de 5 km sale **1 grupo con extremos a 28,2 km**.
  - `consolidacion.js:336`: `evitables = ids.length - 1` supone que cada entrega sería un despacho propio.
  - `despachosNecesarios` (`:283-298`) solo controla volumen y peso, nunca las 25 paradas.
  - Script `07-consolidacion-rendimiento.mjs`:
    - 200 pendientes (radio 3 km): un grupo de **188 entregas** repartido en "6 despachos" (unas 31 paradas cada uno, más que el límite), con **191 "despachos evitables"** de 200;
    - 500 pendientes y 40 despachos planificados: `/consolidacion` tarda **9,8 s** y `/panel` **15,7 s**, síncronos, lo que bloquea a todos los usuarios del servidor.
  - `panel.js:231-235` suma la consolidación de las pendientes **de hoy** al ahorro potencial de cualquier período, también de meses pasados.
- **Impacto en el negocio:** en zonas densas la sugerencia deja de servir ("junta 188 entregas"), la cifra de despachos evitables del Panel es engañosa y, con volumen, el Panel congela el servidor central para todos.
- **Recomendación:**
  - Agrupar con un diámetro máximo (cada par a menos del radio, o un radio alrededor de un centro) y un tope de 25 paradas por grupo.
  - Calcular "evitables" contra los despachos que realmente se habrían hecho (por ejemplo, los agrupados por fecha), o rotularlo como "máximo teórico".
  - Sacar la consolidación del período del Panel, o filtrar las entregas por la fecha del período.
  - Limitar o cachear el cálculo de `un_producto` (hoy son entregas × despachos × cubicaje).
- **Esfuerzo:** medio · **Modelo sugerido:** Opus
- **Specs afectadas:** `cubicaje-consolidacion` (Consolidación de entregas), `panel-analisis`.

### DAT-09 · No se pueden anotar horas que cruzan la medianoche ni una preparación del día anterior
- **Severidad:** media · **Bloquea el piloto:** no
- **Evidencia:**
  - `tiempos.js:159` guarda cada hora como `${trip.fecha}T${hora}`, siempre con la fecha del despacho.
  - `validarOrden` (`:126-135`) compara textos.
  - Script `08-validaciones-tiempos.mjs`:
    - (6) salida 22:00 y regreso 00:40 responde **400** ("regreso… no puede ser anterior a… 23:50");
    - (6b) preparación de 18:00 a 19:00 la tarde anterior y carga a las 07:30 responde **400**.
- **Impacto en el negocio:** si Bodega prepara los pedidos la tarde anterior, o una ruta vuelve después de medianoche, el operador no puede anotar la hora real. Pierde el registro (los KPI quedan con menos datos) o anota una hora falsa para que lo acepte, y los KPI salen distorsionados. Es precisamente lo que mide el piloto ("¿Bodega anota las horas?").
- **Recomendación:** aceptar fecha y hora completas (`AAAA-MM-DDTHH:MM`) en cada evento, con un selector de "día anterior / día siguiente" en la pantalla Hoy, o deducir +1 día cuando la hora es menor que la anterior y la diferencia es plausible (menos de 12 h), avisando al usuario.
- **Esfuerzo:** medio · **Modelo sugerido:** Sonnet
- **Spec afectada:** `despachos-tiempos-kpi` (Tiempos reales en orden cronológico).

### DAT-10 · El consumo real "estanque lleno" no distingue las cargas parciales ni filtra errores de odómetro
- **Severidad:** media · **Bloquea el piloto:** no
- **Evidencia:**
  - `costs.js:234-241` ordena por odómetro y suma los litros de todas las cargas salvo la primera. `fuel_logs` no tiene una columna "estanque lleno" (`db.js:90-98`).
  - Script `03-costos-puros.mjs`:
    - (6) con la última carga parcial, da **16 km/L**, cuando el vehículo rinde unos 10;
    - (6b) con un odómetro digitado de más (104000 en vez de 10400), da **2350 km/L**.
- **Impacto en el negocio:** el Panel deja de detectar consumos altos (falsos negativos), "litros extra" y "costo extra" quedan mal, y un solo error de tipeo arruina el rendimiento real del vehículo para siempre, porque se usa todo el historial.
- **Recomendación:**
  - Agregar `lleno INTEGER` a `fuel_logs` (vía `COLUMNAS_NUEVAS`) y calcular solo entre cargas llenas consecutivas.
  - Usar una ventana (por ejemplo, los últimos 90 días o las últimas N cargas).
  - Rechazar o advertir saltos de odómetro imposibles (por ejemplo, más de 2000 km entre cargas, o menos de 2 o más de 30 km/L).
- **Esfuerzo:** medio · **Modelo sugerido:** Sonnet
- **Spec afectada:** `costos-transporte` (Consumo real por estanque lleno).

### DAT-11 · Falta el índice `deliveries(client_id)`: GET /clientes crece en forma cuadrática
- **Severidad:** media · **Bloquea el piloto:** no (con el volumen de un piloto no se nota)
- **Evidencia:**
  - `api.js:323-327` hace 3 subconsultas por cliente sobre `deliveries` filtrando por `client_id`, y no hay índice para esa columna. Solo existen `idx_deliveries_estado` e `idx_deliveries_trip` (`db.js:184-185`).
  - Lo mismo ocurre en `/zonas/mapa` (`api.js:879`).
  - Script `09-indices.mjs`:

    | Clientes / entregas | Sin índice | Con índice `deliveries(client_id, estado)` |
    |---|---|---|
    | 200 / 2 000 | 55 ms | 1 ms |
    | 500 / 5 000 | 338 ms | — |
    | 1 000 / 20 000 | **15 082 ms** | **61 ms** |

  - `EXPLAIN` muestra `SCAN` en estas consultas: `trips` por estado y fecha o por vehículo, `fuel_logs` por vehículo, `fuel_prices` vigente y `trip_events` por `stop_id` (este último en el borrado en cascada).
- **Impacto en el negocio:** con un año de historial, la lista de clientes y el mapa tardan segundos, y en el servidor bloquean a todos (node:sqlite es síncrono).
- **Recomendación:** crear los índices en `SCHEMA` con `CREATE INDEX IF NOT EXISTS` (así se aplican también a bases existentes):
  - `deliveries(client_id, estado)`;
  - `trips(estado, fecha)`;
  - `trips(vehicle_id, fecha)`;
  - `fuel_logs(vehicle_id, fecha)`;
  - `fuel_prices(combustible, vigente_desde)`;
  - `trip_events(stop_id)`;
  - `gastos(fecha)`.
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Specs afectadas:** `clientes-zonas`, `tiempo-real-concurrencia` (rendimiento del servidor compartido).

### DAT-12 · Al cerrar un despacho sus km planificados se suman al odómetro aunque la lectura real ya los incluía
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:**
  - `api.js:701`: `UPDATE vehicles SET km_actual = km_actual + ?` con `t.distancia_km` (los km de Google, no los reales).
  - Las lecturas reales solo suben `km_actual` si son mayores (`api.js:255,766,806`).
  - Script `02-despacho-costos.mjs` (D): `km_actual` en 10 000, carga con el odómetro real 10 054 y después el cierre: `km_actual` queda en **10 107,5** (53,5 km de más). Las lecturas reales posteriores que sean menores se ignoran (`vehiculos-estado.js:49` toma el máximo).
- **Impacto en el negocio:** el odómetro queda adelantado más o menos en los km de un despacho. La mantención aparece "próxima" o "vencida" antes de tiempo y el % de neumáticos sale un poco más alto. Es conservador, pero no es un dato real.
- **Recomendación:** no sumar km al odómetro al cerrar, o sumar solo si no hay una lectura real posterior a la salida del despacho. Mostrar el odómetro "estimado" aparte del "leído".
- **Esfuerzo:** bajo · **Modelo sugerido:** Sonnet
- **Spec afectada:** `flota-vehiculos` (Odómetro solo aumenta, Mantención preventiva).

### DAT-13 · El km semanal registrado no suma los km de los despachos
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:**
  - Al cerrar un despacho no se escribe `km_semanal` (`api.js:685-701`).
  - `estadoFlota` muestra `km_despachos` aparte y calcula `km_prom_semanal` solo con `km_registrado` (`vehiculos-estado.js:121-122`).
  - Script `02-despacho-costos.mjs` (E): en la semana del 23-11, `km_registrado = null` y `km_despachos = 53,5`, con `km_prom_semanal = null`. Por eso tampoco hay "fecha estimada" de mantención.
- **Impacto en el negocio:** si no se anota a mano el km semanal, el aviso de mantención por ritmo de uso ("menos de 2 semanas") nunca se activa, aunque haya despachos.
- **Recomendación:** usar `km_registrado ?? km_despachos` para el promedio (indicando la fuente), o aclarar en la spec que "sumar" significa mostrar al lado.
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Spec afectada:** `flota-vehiculos` (Kilometraje semanal).

### DAT-14 · Validación de entradas incompleta
- **Severidad:** baja (la interfaz usa campos de fecha y número, así que pega sobre todo a la API y a la importación) · **Bloquea el piloto:** no
- **Evidencia** (script `08-validaciones-tiempos.mjs`):
  - `POST /despachos` con `fecha: "25/11/2026"` responde **201**. El despacho completado **no aparece** en el Control de costos de noviembre (`api.js:651` solo exige que no esté vacía).
  - `POST /vehiculos/:id/cargas` con `fecha: "ayer"` y `monto: -13000` responde **201** y queda guardado tal cual (`api.js:251-254`).
  - Una carga a un vehículo inexistente responde **500 error_interno** (clave foránea).
  - Un precio con `vigente_desde: "1/12/2026"` responde 201 y nunca rige (`api.js:194`).
  - `nueva_fecha: "pronto"` al cerrar responde 200 y la entrega queda con `fecha = "pronto"` (`api.js:687,695`).
  - `POST /vehiculos` con rendimiento -8, capacidad -100 y ajuste -50 % responde **201**. `PUT` sí valida el rendimiento, `POST` no (`api.js:223-231` frente a `:237`). Con `ajuste_condicion_pct = -50` el rendimiento efectivo sube a 15 km/L (`03` (8)).
  - El combustible del vehículo es texto libre ("bencina 93"): el vehículo deja de encontrar su precio (`03`/`08` (4)).
- **Impacto en el negocio:** registros que "desaparecen" de los reportes mensuales (las fechas no ISO quedan fuera de los rangos), costos reales negativos y rendimiento inflado por un ajuste mal ingresado.
- **Recomendación:**
  - Un validador común `fechaISO()` para `fecha`, `vigente_desde`, `nueva_fecha` y `semana_inicio`, y `positivo()` para montos, litros, rendimiento, capacidad y peajes.
  - Limitar los ajustes a 0-100 %.
  - Validar `combustible` contra `COMBUSTIBLES`.
  - Comprobar que el vehículo exista antes de insertar una carga (404 en vez de 500).
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Specs afectadas:** `costos-transporte`, `flota-vehiculos`, `despachos-tiempos-kpi`.

### DAT-15 · Se guardan despachos con vehículos "fuera de servicio" o desactivados
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:** `planner.js:273-276`: cuando se pasan ids, `vehiculosPorId` no filtra por `activo` ni por `estado`. Script `08` (7): un vehículo "fuera de servicio" responde **201**.
- **Impacto en el negocio:** se puede asignar un vehículo en taller y su costo y km entran a la flota activa.
- **Recomendación:** en `crearDespacho`, exigir `activo = 1 AND estado = 'activo'`. Mantener la planificación libre si se quiere comparar.
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Spec afectada:** `planificacion-rutas`.

### DAT-16 · Borrar un vehículo sin despachos elimina en cascada sus cargas y km semanales
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:**
  - `db.js:92` y `:239` tienen `ON DELETE CASCADE` en `fuel_logs` y `km_semanal`.
  - `api.js:241-245` solo desactiva el vehículo si tiene despachos.
  - Script `10-integridad.mjs` (2): el combustible real de noviembre baja de **52 000 a 0** al borrar un vehículo arrendado que tenía cargas. Sus gastos quedan sin vehículo.
- **Impacto en el negocio:** cambia en silencio el costo real de meses ya cerrados.
- **Recomendación:** desactivar también cuando haya cargas, gastos o km registrados, o cambiar la cascada por `RESTRICT` y mostrar un aviso.
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Spec afectada:** `costos-transporte` (Control de costos).

### DAT-17 · La tabla por vehículo de Control de costos omite los vehículos desactivados
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:** `control-costos.js:269` filtra `WHERE activo = 1`, y lo mismo hacen `utilizacionCarga` (`consolidacion.js:422`) y `rendimientoVehiculos`. Script `10` (3): total modelado **23 982** contra una suma de la tabla por vehículo de **0**.
- **Impacto en el negocio:** la tabla no cuadra con el total del período cuando se da de baja un vehículo a mitad de mes.
- **Recomendación:** incluir los vehículos con despachos o gastos en el período aunque estén inactivos (marcados como "dado de baja").
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Spec afectada:** `costos-transporte`, `panel-analisis`.

### DAT-18 · Las horas extra se calculan por despacho, no por la jornada del conductor
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:** `costs.js:153-161` compara las horas de **un** despacho con `jornadaHoras`. Script `03` (4): dos despachos de 6 h el mismo día con una jornada de 8 h dan 0 horas extra cada uno.
- **Impacto en el negocio:** el costo del conductor queda bajo cuando hace dos o más rutas al día.
- **Recomendación:** calcular el recargo al cerrar o en Control de costos, acumulando por conductor y día; o documentar que es por despacho.
- **Esfuerzo:** medio · **Modelo sugerido:** Sonnet
- **Spec afectada:** `costos-transporte` (Costo modelado: horas extra con recargo).

### DAT-19 · Fase 4: `sinCostos` deja pasar el desgaste en $/km hacia Bodega
- **Severidad:** baja · **Bloquea el piloto:** no (el criterio del piloto "ningún rol pudo ver… lo que no le correspondía" se cumple en pantalla, no en la API)
- **Evidencia:**
  - En `origin/fase-4/bodega-piloto`, `server/auth.js:118-120` filtra `/^(costo|ahorro|valor_)/` y una lista de nombres.
  - `api.js:210` devuelve `desgaste: ck.desgaste`, con `total` y `componentes.neumaticos` y `mantencion`.
  - Script `sincostos-fase4.mjs` (código copiado tal cual de la rama): lo que ve Bodega incluye `"desgaste":{"total":100,"componentes":{…"neumaticos":20,…"mantencion":30}…}`.
- **Impacto en el negocio:** Bodega puede leer, desde la API, el costo por km de desgaste de cada vehículo.
- **Recomendación:** agregar `desgaste` a `CAMPOS_COSTO` (o renombrarlo `costo_desgaste_detalle`), y agregar una prueba que recorra todas las rutas que Bodega puede leer y busque números junto a claves de dinero.
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Spec afectada:** `autenticacion-roles` (delta de fase 4).

### DAT-20 · Respaldo: ajustes que no se reemplazan, error 500 genérico y un .db sin validar
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:**
  - `respaldo.js:43` no borra `settings`, y `:54-55` hace upsert. Script `05` (4): un `costo_hora_conductor` que no venía en el respaldo queda en el valor anterior (99 999).
  - Script `05` (5): un respaldo con una clave foránea rota responde **500 "error inesperado"**. Los datos quedan intactos (la transacción funciona), pero el usuario no sabe qué fila falla.
  - `navegador/sqlite-wasm.js:74-80` (`cargarBytes`) acepta cualquier SQLite válido. Después `prepararDb` crea tablas vacías, así que un `.db` ajeno deja BHUMI vacío. **Por confirmar** en el navegador.
  - **Por confirmar:** si alguna base vieja del navegador tiene filas huérfanas (antes de que `exportBytes` reactivara `foreign_keys`). En ese caso, su JSON no se podría restaurar en el servidor (500). La historia de git no lo muestra, porque todo entró en un solo commit.
- **Impacto en el negocio:** una restauración que "reemplaza todo" deja ajustes mezclados, y si la migración al servidor falla no se sabe por qué.
- **Recomendación:**
  - Restaurar `settings` borrando antes todo salvo `google_api_key`.
  - Atrapar los errores de restricción y responder 400 indicando la tabla y la fila.
  - En `cargarBytes`, exigir las tablas `trips`, `vehicles` y `settings` antes de reemplazar.
  - Antes del piloto, correr `PRAGMA foreign_key_check` sobre el JSON migrado.
- **Esfuerzo:** bajo · **Modelo sugerido:** Sonnet
- **Spec afectada:** `respaldo-datos`.

### DAT-21 · Supuestos y valores por defecto poco visibles
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:**
  - Hora de salida por defecto 08:00 cuando hay fecha y no hora (`planner.js:32`).
  - Mantención "próxima" si faltan 2 semanas o menos (`vehiculos-estado.js:69`).
  - Valores iniciales en `SETTINGS_DEFAULTS`: `alerta_mantencion_pct 10`, `radio_cercania_km 5`, `umbral_consumo_pct 10`, `cache_horas 12` y pesos 1 (`db.js:26-32`).
  - La proyección valora las entregas sin ruta al costo promedio **histórico** y cuenta las que no tienen fecha en **cualquier** mes (`analytics.js:55-60`).
  - La depreciación ignora `vida_util_anios` (`costs.js:30-33`; script `03` (7) da depreciación null aunque se ingresaron los años).
  - El "ahorro" de cada despacho se mide contra el orden de selección (`planner.js:368`). "Seleccionar todas" usa el orden de la lista (`rutas-nueva.js:18`), así que el ahorro puede salir inflado frente al orden que realmente usaba la empresa.
  - No encontré factores de carga, 35 km/h ni 1,3 fuera del modo demo y del Google simulado: los ajustes de rendimiento solo se aplican si el usuario los ingresa.
- **Impacto en el negocio:** cifras que parecen datos reales pero dependen de supuestos (proyección, ahorro, alertas).
- **Recomendación:** mostrar en pantalla la fuente de cada supuesto ("con hora 08:00 por defecto", "estimado con el promedio histórico"), permitir indicar un "orden tradicional" explícito y usar `vida_util_anios` (o quitar el campo).
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Spec afectada:** `datos-reales`.

### DAT-22 · Las cantidades no enteras siempre dan "no cabe"
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:** `cubicaje.js:52` coloca `Math.floor(cantidad)`, pero `unidades_total` suma la cantidad tal cual (`:147`). `delivery_items.cantidad` es REAL. Script `06-cubicaje.mjs` (4): 2,5 cajas de 10 cm en 1 m³ dan `cabe=false`, 2 de 2,5 colocadas, y `limitante=volumen`.
- **Impacto en el negocio:** una entrega con una cantidad decimal (una unidad por kg, media caja) bloquea el despacho por "cubicaje".
- **Recomendación:** exigir cantidades enteras para productos con medidas, o redondear hacia arriba de forma explícita.
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Spec afectada:** `cubicaje-consolidacion`.

### DAT-23 · El personal del despacho se puede cambiar después del cierre
- **Severidad:** baja · **Bloquea el piloto:** no
- **Evidencia:** `api.js:716-724` no revisa el estado. Script `02` (I): un `PUT /despachos/:id/personal` sobre un despacho completado responde **200**.
- **Impacto en el negocio:** los KPI de un mes cerrado se pueden reatribuir a otro conductor. También puede ser útil para corregir errores.
- **Recomendación:** permitirlo solo a Administrador después del cierre, con auditoría, o seguir la spec ("antes del cierre").
- **Esfuerzo:** bajo · **Modelo sugerido:** Haiku
- **Spec afectada:** `despachos-tiempos-kpi` (Personal del despacho editable).

---

## Lo que está bien resuelto

- **Optimizador exacto (Held-Karp) correcto:** coincide con la fuerza bruta en 280 casos asimétricos (n de 2 a 8, con y sin regreso), con 0 fallas. Con paradas duplicadas (distancia 0) también da el óptimo.
  - Tiempos: 22 ms con n=15 y 163 ms en el peor caso del plan (2 vehículos × 4 alternativas).
  - La heurística (vecino más cercano desde cada inicio + 2-opt + or-opt) siempre incluye todas las paradas. Con n=25 tarda 53 ms. Frente al óptimo, la brecha es de 0,32 % como máximo en matrices simétricas y de 3,3 % en asimétricas, y encuentra el óptimo en 59 de 60 y 55 de 60 casos (`01b-salida.txt`).
  - `validarMatriz` impide que lleguen tramos nulos al optimizador.
- **Fórmulas del README:** se verificaron con el caso de 100 km: combustible 13 000, desgaste 10 000, conductor 10 000, total 33 000. El desgaste derivado (neumáticos por vida útil, mantención por intervalo) funciona. Los montos en CLP se redondean a enteros por componente y el total es su suma exacta.
- **Reparto a clientes:** al crear el despacho, el reparto suma exactamente el total y en enteros, incluso con montos que no se dividen (1000 / 3 = 333 + 333 + 334). El problema aparece solo al editar gastos después (DAT-06).
- **Datos reales:** sin rendimiento o sin precio el costo queda en blanco y el despacho no se guarda. El precio vigente se toma por la fecha del despacho y queda congelado en `trips.precio_combustible`. Solo se usa Google para distancias y tiempos: la recta × 1,3 a 35 km/h existe solo en `test/fake-google.js` y en el modo demo. Haversine se usa solo para agrupar y filtrar por radio.
- **Cubicaje:**
  - Es geométricamente seguro: 0 cajas fuera del espacio y 0 solapes en 40 cargas aleatorias mezcladas. El cubo perfecto da 8 sí y 9 no, como pide la spec. Respeta rotable y no rotable y el límite real (el menor entre volumen y peso).
  - Rinde bien: 500 productos distintos en 13 ms y una caja de 1 cm³ en 0 ms. No hay riesgo de bloquear el servidor por el cubicaje mismo, solo por la consolidación (DAT-08).
- **Integridad:**
  - `PRAGMA foreign_keys = ON` en node:sqlite y en sql.js (se reactiva después de `export` y `cargarBytes`).
  - Las operaciones compuestas (cerrar, cancelar, eliminar un despacho, importar Excel, restaurar JSON, guardar tiempos) van en `tx()`. Una restauración con error revierte todo: lo comprobé (`05` (5), datos intactos).
  - Los triggers de `version` suben con cualquier UPDATE y la restauración conserva las versiones.
  - Las migraciones con `COLUMNAS_NUEVAS` tienen prueba propia. El resultado por parada en bases antiguas se deduce solo cuando no hay ambigüedad.
- **Fechas puras:** `lunesDe`, `sumarDias`, `resolverPeriodo` y `ultimosMeses` calculan sobre la fecha ISO en UTC, así que no dependen de la zona del proceso. Las semanas parten en lunes. Los minutos de los KPI se calculan como horas de reloj (`Date.parse(...Z)`), así que el cambio de horario no altera las diferencias.
- **Odómetro y mantención:**
  - Las cargas, los gastos y el km semanal solo suben `km_actual`, y el odómetro mostrado es la mayor lectura conocida.
  - La próxima mantención indicada en el registro manda sobre el intervalo, y una fecha cumplida la deja vencida aunque falten km, como pide la spec.
  - El desgaste de neumáticos queda en blanco cuando no hay registro.
- **Respaldo JSON:** no incluye la clave de Google, ni usuarios, ni sesiones, ni auditoría. Respeta el orden de dependencias al borrar y al insertar.
- **Enlaces para el conductor:** se encadenan en tramos de 9 paradas intermedias (12 paradas con regreso dan 2 enlaces).

## Por confirmar

- La magnitud real del error de tráfico de DAT-01 con la clave de Google del piloto, y que `TZ=America/Santiago` funcione en `node:24-alpine` sin `tzdata`.
- Si hay filas huérfanas en las bases del navegador que se van a migrar (DAT-20).
- El comportamiento de `cargarBytes` con un `.db` ajeno en el navegador real (DAT-20). Lo deduje leyendo el código.
