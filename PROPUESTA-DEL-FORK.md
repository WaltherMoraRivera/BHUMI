# Propuesta del fork wmrDev/BHUMI

Documento para el autor de BHUMI y para su agente. Resume lo que agrega este fork y lo que propone, y deja al autor la decisión: **unir el fork completo, integrar solo algunas partes, o ninguna de las dos**.

- Repositorio original: `MaxisotooH/BHUMI` (en este documento, *upstream*).
- Fork: `wmrDev/BHUMI` (en este documento, *fork*).
- Fecha: 2026-11-26. Base de upstream analizada: `01a620d` (Merge PR 3, Fase 3). Rama de la Fase 4 analizada: `fase-4/bodega-piloto` @ `7534c48`, PR 4 de upstream, abierto.
- **Estado del fork: congelado.** No se escribe código de la aplicación ni se avanza en ningún issue hasta que el autor defina el contexto del negocio y el alcance.

---

## 1. Instrucciones para el agente que lea esto

Trabaja en el clon de upstream, sin modificar nada hasta que el autor decida.

1. Agrega el fork como remoto y trae sus ramas:
   ```bash
   git remote add fork https://github.com/wmrDev/BHUMI.git
   git fetch fork
   ```
2. Comprueba que el fork solo **agrega** archivos (no modifica ni borra nada de upstream):
   ```bash
   git merge-base --is-ancestor origin/main fork/main && echo "fork/main contiene a main"
   git diff --stat origin/main fork/main          # 40 archivos, +4727, 0 eliminaciones
   git diff --diff-filter=MD --name-only origin/main fork/main   # debe salir vacío
   ```
3. Lee en este orden: este archivo → `docs/AUDITORIA.md` → `docs/auditoria/*.md` → `openspec/config.yaml` → `openspec/specs/*/spec.md` → `openspec/changes/fase-4-bodega-piloto/`.
4. **No tomes los hallazgos como ciertos sin verificarlos.** Cada uno cita `archivo:línea` de `01a620d` o de la rama de la Fase 4. Comprueba al menos los 7 bloqueantes (sección 5) antes de recomendar algo.
5. Las specs de `openspec/specs/` se escribieron a partir del README, las descripciones de los PR 1 a 4 y los nombres de las pruebas. **Describen lo que el sistema hace hoy, no lo que el autor quiere.** Contrástalas con la intención del autor.
6. Responde al autor con: (a) qué partes integrar, (b) qué hallazgos confirmas o descartas, (c) respuestas o dudas sobre las preguntas de la sección 8.

---

## 2. Qué agrega el fork (todo está en `fork/main`)

| Parte | Rutas | Qué es | Depende de |
|---|---|---|---|
| A. OpenSpec | `openspec/config.yaml`, `openspec/specs/` (13 capacidades), `openspec/changes/archive/` | Especificaciones de comportamiento del sistema actual, en formato OpenSpec 2.x (`@fission-ai/openspec`), con contexto y reglas del proyecto en `config.yaml` | Nada |
| B. Cambio de la Fase 4 | `openspec/changes/fase-4-bodega-piloto/` | El PR 4 documentado como cambio OpenSpec: propuesta, diseño, tareas (11 de 15) y deltas para `autenticacion-roles` y la nueva capacidad `pantalla-hoy` | A |
| C. Skills de OpenSpec | `.claude/skills/openspec-*` (6) y `.agents/skills/` (copia genérica) | Comandos para Claude Code: `/openspec-propose`, `/openspec-apply-change`, `/openspec-archive-change`, `/openspec-explore`, `/openspec-sync-specs` y `/openspec-update-change` | A |
| D. Plan de trabajo inicial | `docs/PLAN-DE-TRABAJO.md` | Primer análisis del proyecto, el criterio de modelo y esfuerzo para agentes, y los issues 1 a 21 por hito. Lo actualizan la parte E y la sección 6 de este documento | Nada |
| E. Auditoría | `docs/AUDITORIA.md`, `docs/auditoria/arquitectura.md`, `docs/auditoria/datos-negocio.md`, `docs/auditoria/calidad-operacion.md` | Auditoría con cuatro revisores independientes: decisiones de fondo, ARQ-01 a 17, DAT-01 a 23, CAL-01 a 20 y el resumen de SEG-01 a 16 | Nada |
| F. Este documento | `PROPUESTA-DEL-FORK.md` | Resumen para decidir la integración | Nada |

**Lo que el fork no tiene:**
- **Cambios en el código de la aplicación.** `server/`, `public/`, `navegador/`, `test/`, `package.json` y el despliegue están idénticos a upstream.
- **La Fase 4.** El fork tiene la rama `fase-4/bodega-piloto` igual que upstream, sin cambios.
- **El detalle de seguridad.** Los pasos para explotar SEG-01 a 16 no se publicaron, porque el repositorio es público y los hallazgos siguen abiertos. El dueño del fork se los entrega al autor por un canal privado.

Fuera del repositorio, en GitHub (fork):
- 43 issues abiertos, con hitos y etiquetas (sección 6).
- Etiquetas `modelo:*`, `esfuerzo:*`, `tipo:*`, `area:*`, `prioridad:*`, `origen:auditoria` y `estado:en-espera`.
- Cinco hitos: M0 Bases, M1 Fase 4, M2 Piloto, M3 Calidad y M4 Escalabilidad.

---

## 3. Opciones de integración

| Opción | Cómo | A favor | En contra |
|---|---|---|---|
| 1. Unir el fork completo | `git merge fork/main` en upstream (o un PR del fork a upstream) | Sin conflictos: solo agrega archivos. Trae todo el contexto | Incluye `.agents/` (duplica las skills, unas 1.350 líneas) y documentos que hablan de "el fork" |
| 2. Integrar por partes | `git checkout fork/main -- <rutas>` y un commit propio por parte | El autor elige qué adoptar, y el historial queda a su nombre | Hay que elegir con criterio (ver abajo) |
| 3. Solo como referencia | Leer y no integrar nada | Upstream no cambia | Se pierde el marco de specs para los próximos cambios |

**Recomendación del fork: opción 2.**
1. Partes **A + B + C** sin `.agents/`, si el autor quiere trabajar con OpenSpec. Antes, revisar las specs (sección 1, punto 5).
2. **E** (la auditoría), aunque no se adopte OpenSpec, porque es independiente.
3. **D** es opcional: los issues viven en el fork y no en upstream.

Comandos de ejemplo para la opción 2:
```bash
git switch -c integrar-fork
git checkout fork/main -- openspec .claude/skills docs/AUDITORIA.md docs/auditoria
git commit -m "Integrar OpenSpec y la auditoría desde el fork"
```

**Ojo con la Fase 4.** `fase-4/bodega-piloto` parte de `01a620d`, antes de todo lo del fork. Si se integra el fork antes que la Fase 4, conviene rebasar la Fase 4 sobre el nuevo `main`. Al revisarla, usar `git diff main...fase-4/bodega-piloto` (tres puntos), porque con dos puntos parece borrar `openspec/`. Al integrar la Fase 4, archivar el cambio con `openspec archive fase-4-bodega-piloto`.

---

## 4. Decisiones de fondo que se proponen

| Decisión actual | Propuesta | Cuándo |
|---|---|---|
| Mantener el modo doble clic junto al servidor | **Congelarlo ya.** Después del piloto, retirar el modo real y dejar solo la demo. Ocupa ~880 líneas, pero ata el servidor a una API de base de datos síncrona (181 llamadas), obliga a 4 modos de prueba, empaqueta código de `test/` y usa una clave de Google sin restricción de sitio | Congelar hoy; retirar tras el piloto (issue 21) |
| SQLite | Mantenerlo. Fijar la versión de Node (`node:sqlite` sigue experimental) y documentar `better-sqlite3` como reemplazo. PostgreSQL sería sobre-ingeniería a esta escala | Ahora (issues 33 y 20, este en espera) |
| JS sin compilación ni framework | Mantenerlo y agregar `@ts-check` + JSDoc, ESLint, Playwright y la trampa de foco en los modales | Ahora (issues 3, 17 y 19) |
| Solo Google Maps | Mantenerlo, con caché por filas, tráfico solo al guardar, medición del gasto y un despacho de contingencia con km informados. Gasto estimado: US$130 a 210 al mes, aproximado | Durante y después del piloto (issues 36 y 40) |

**Principios nuevos que se proponen**, para agregar a los del proyecto en `openspec/config.yaml`:
1. Hora de negocio = `America/Santiago`, en una sola función, y CI también con `TZ=UTC`.
2. Lo que ve cada rol se define por **lista permitida**, en JSON, PDF, Excel y avisos.
3. Un dato informado es real: sin Google, el despacho se guarda con km informados y marcados como tales. Un costo incompleto se muestra como incompleto.
4. Las operaciones destructivas (restaurar, borrar, cerrar) se validan antes de borrar.
5. Nada entra a `main` sin uso real y sin revisión de otra persona.

---

## 5. Los 7 bloqueantes del piloto

Todos son de esfuerzo bajo. Hay que verificarlos primero.

| # | Problema | Evidencia | Issue |
|---|---|---|---|
| 1 | El contenedor corre en UTC: el tráfico se consulta 3 h antes y desde las 21:00 "hoy" pasa a mañana | Sin `TZ` en `Dockerfile` ni en `docker-compose.yml`; `server/services/planner.js:32`; `server/api.js:74` (`hoyStr` con `toISOString`) | 27 |
| 2 | El PDF de la hoja de ruta muestra los costos a Bodega (Fase 4) | Rama de la Fase 4: `server/auth.js` permite `/reportes/despacho/:id.pdf` a Bodega; `server/services/reportes.js:408-418` | 25 |
| 3 | La lista negra deja pasar `desgaste.*` ($/km) a Bodega (Fase 4) | Rama de la Fase 4: `sinCostos` en `server/auth.js` | 16 |
| 4 | El login no limita el largo del correo: se puede llenar la base sin autenticarse | `server/auth.js:188-202` (`emailValido` solo se usa al crear usuarios) | 28 |
| 5 | Ante un corte de red, la interfaz recomienda "restaura tu último respaldo" | `public/js/lib.js:57` | 29 |
| 6 | Restaurar un JSON como `{"settings": []}` vacía toda la base | `server/services/respaldo.js:36-43` | 30 |
| 7 | Una misma entrega puede quedar en dos despachos, y un producto sin peso apaga el control de capacidad | `server/api.js:653-666` + `server/services/planner.js:397`; `server/services/stops.js:10` + `planner.js:137,187,355` | 31 |

**Hallazgo de proceso (CAL-14).**
- Las fases 1 a 4 se escribieron en 1 h 47 min.
- Ningún PR tuvo revisión de otra persona, y cada uno se fusionó entre 12 y 20 minutos después de abrirse.
- Los PR 1 a 3 entraron sin CI, y `main` no está protegida.
- Nadie usa el sistema entre fases, y por eso nadie notó la demo rota desde la Fase 2 (CAL-05), la zona horaria (CAL-02) ni el mensaje de corte de red (CAL-03).

La primera etapa del plan es acordar el flujo de trabajo (issue 24).

---

## 6. Plan de acción e issues

Todas las acciones están **en espera** (`estado:en-espera`). Cada issue trae contexto, tareas, criterios de aceptación, la spec afectada y una recomendación de **modelo** (Opus, Sonnet o Haiku) y **esfuerzo** del agente (bajo, medio o alto).

**Etapas y puertas:**

| Etapa | Objetivo | Puerta para pasar a la siguiente | Issues |
|---|---|---|---|
| 0 · Acordar el flujo (1 a 2 días) | Repositorio canónico, `main` protegida, revisión de otra persona, PR de menos de 400 líneas, versiones | Flujo acordado con el autor | 24, 1, 2, 4 |
| 1 · Corregir la Fase 4 (1 día) | PDF sin costos, lista permitida por rol, prueba de Bodega real, rebase | Bodega no ve montos, y las pruebas lo comprueban | 6, 16, 25, 26, 7 |
| 2 · Bloqueantes del piloto (2 a 3 días) | Hora de Chile, login, aviso de red, restauración, despachos, índices, `construir`, demo, DoS, validación | Prueba de humo de `docs/PILOTO.md` aprobada | 27 a 35 |
| 3 · Piloto (1 semana) | Infraestructura, gasto de Google, observabilidad, despliegue por versiones, preparación nocturna | Criterios de cierre de `docs/PILOTO.md` | 8 a 11, 36 a 39 |
| 4 · Decisiones con datos | Retirar el doble clic, contingencia sin Google, costos incompletos, odómetro, consolidación, API, migraciones, CSP, e2e | Por definir con el autor | 3, 5, 12 a 15, 17 a 21, 40 a 45 |

**Issues del fork**

| # | Título | Hito | Modelo | Esfuerzo | Origen |
|---|---|---|---|---|---|
| 1 | Integrar OpenSpec: specs base del sistema y flujo de cambios | M0 | Sonnet | Bajo | Plan inicial |
| 2 | Los scripts test:local, test:navegador y test:wasm fallan en Windows | M0 | Haiku | Bajo | Plan inicial |
| 3 | Agregar linter y formateador con verificación en CI | M0 | Sonnet | Medio | Plan inicial |
| 4 | Plantillas de issue y PR y guía CONTRIBUTING con el flujo OpenSpec | M0 | Haiku | Bajo | Plan inicial |
| 5 | Definir la licencia del proyecto | M0 | Haiku | Bajo | Plan inicial |
| 6 | Revisión de código de la Fase 4 | M1 | Opus | Alto | Plan inicial |
| 7 | Cerrar la Fase 4: CI con Node 22.13, integrar y archivar el cambio OpenSpec | M1 | Haiku | Bajo | Plan inicial |
| 8 | Preparar la infraestructura del piloto | M2 | Sonnet | Medio | Plan inicial |
| 9 | Prueba de humo del piloto, incluida la reconexión tras un 502 de Caddy | M2 | Sonnet | Medio | Plan inicial |
| 10 | Simulacro de restauración desde Litestream | M2 | Sonnet | Medio | Plan inicial |
| 11 | Evaluar el piloto y priorizar lo siguiente | M2 | Opus | Medio | Plan inicial |
| 12 | Dividir server/api.js en routers por dominio | M3 | Opus | Alto | Plan inicial + ARQ-07 |
| 13 | Migraciones de esquema versionadas | M3 | Opus | Alto | Plan inicial + ARQ-08 |
| 14 | Content-Security-Policy y encabezados de seguridad | M3 | Sonnet | Medio | Plan inicial + SEG-09 |
| 15 | Filtrar por rol los avisos de tiempo real | M3 | Sonnet | Medio | Plan inicial + SEG-11 |
| 16 | Bodega sin costos: lista permitida por respuesta | **M1** | Opus | Medio | Plan inicial + SEG-02 |
| 17 | Pruebas end-to-end de la interfaz en navegador | M3 | Sonnet | Alto | Plan inicial + CAL-10 |
| 18 | Actualizar dependencias: Express 5, PDFKit 0.21 y Chart.js | M3 | Sonnet | Medio | Plan inicial |
| 19 | Tipado gradual con JSDoc y checkJs | M3 | Sonnet | Medio | Plan inicial |
| 20 | Evaluar PostgreSQL + PostGIS (en espera sin evidencia) | M4 | Opus | Alto | Plan inicial |
| 21 | Decidir el futuro de la versión de doble clic | M4 | Opus | Medio | Plan inicial + A1 |
| 24 | Flujo de trabajo: repositorio canónico, main protegida, revisión y versiones | M0 | Opus | Bajo | CAL-14, CAL-15 |
| 25 | Hoja de ruta PDF sin costos para Bodega | M1 | Sonnet | Bajo | SEG-01, ARQ-03 |
| 26 | La prueba de Bodega sobre km semanal no prueba lo que dice | M1 | Haiku | Bajo | ARQ-14 |
| 27 | Usar la hora de Chile en todo el sistema | M2 | Sonnet | Bajo | ARQ-01, DAT-01, DAT-02, CAL-02 |
| 28 | Acotar el login y el tamaño de las solicitudes | M2 | Sonnet | Bajo | SEG-03, SEG-07, SEG-12 |
| 29 | Aviso de corte de red sin recomendar restaurar, y reintento en Hoy | M2 | Sonnet | Bajo | CAL-03 |
| 30 | Restauración segura del respaldo JSON | M2 | Sonnet | Medio | DAT-04, CAL-11, SEG-08, DAT-20 |
| 31 | Despachos: evitar la carrera y controlar capacidad con productos sin peso | M2 | Sonnet | Bajo | ARQ-02, DAT-03, DAT-05, DAT-15 |
| 32 | Agregar los índices faltantes | M2 | Haiku | Bajo | CAL-01, DAT-11, ARQ-13 |
| 33 | construir sin secretos, demo reparada e imagen de Node fijada | M2 | Haiku | Bajo | CAL-04, CAL-05, ARQ-09 |
| 34 | Límites contra consumo excesivo: Excel, cubicaje y tiempo real | M2 | Sonnet | Bajo | SEG-04, SEG-05, SEG-11 |
| 35 | Validación común de fechas, montos y datos de vehículos | M2 | Haiku | Bajo | DAT-14 |
| 36 | Medir el gasto de Google y pedir a la caché solo lo que falta | M2 | Sonnet | Medio | ARQ-04 |
| 37 | Observabilidad mínima y retención de auditoría, sesiones y caché | M2 | Sonnet | Medio | CAL-08, CAL-09 |
| 38 | Despliegue por versiones etiquetadas con vuelta atrás | M2 | Sonnet | Medio | CAL-15, CAL-13 |
| 39 | Decidir cómo registrar la preparación nocturna y los regresos tras medianoche | M2 | Opus | Medio | CAL-06, DAT-09 |
| 40 | Despacho de contingencia sin Google, con km informados | M3 | Opus | Medio | ARQ-05 |
| 41 | Costos incompletos marcados y gastos del despacho bien repartidos | M3 | Opus | Medio | DAT-06, DAT-07, DAT-21 |
| 42 | Odómetro con lecturas reales y consumo con cargas parciales | M3 | Sonnet | Medio | ARQ-06, DAT-10, DAT-12, DAT-13 |
| 43 | Consolidación sin grupos encadenados y sin bloquear el servidor | M3 | Opus | Medio | DAT-08 |
| 44 | Guardar en una sola transacción del servidor el despacho armado con clientes | M3 | Sonnet | Medio | CAL-12 |
| 45 | Ajustes menores de la auditoría | M3 | Sonnet | Medio | Hallazgos de severidad baja |

Los números 22 y 23 son los PR que integraron las partes A a E en el fork. Si el autor prefiere tener los issues en upstream, se pueden transferir o recrear allá.

**Criterio de modelo y esfuerzo:**
- **Opus:** diseño, arquitectura, seguridad, revisiones y decisiones con datos reales en juego.
- **Sonnet:** implementación con especificación clara.
- **Haiku:** tareas mecánicas, con verificación obvia.

El esfuerzo es el nivel de razonamiento del agente y depende de la incertidumbre y del impacto, no del tamaño del cambio.

---

## 7. Lo que está bien y no se propone cambiar

- **Seguridad:** sin bypass de permisos en 30 variantes de ruta, sin inyección SQL, sin XSS explotable y sin escalada de rol. scrypt con parámetros guardados, tokens guardados como hash, mismo mensaje y mismo tiempo de respuesta ante correos inexistentes, y dos claves de Google en el servidor.
- **Cálculo:** las fórmulas del README cuadran. El optimizador exacto coincide con la fuerza bruta en 280 casos, y la heurística resuelve 25 paradas en 53 ms, a no más de 3,3 % del óptimo. El cubicaje no produce solapes.
- **Datos:** las claves foráneas están activas y una restauración fallida revierte todo. Fuera del modo demo no se inventan distancias.
- **Código y documentación:** 109 pruebas estables con 93 % de cobertura en el servidor; escape correcto en la plantilla `html`; cierre ordenado del servidor; documentación de despliegue y del piloto detallada.

---

## 8. Preguntas para el autor

Definen el contexto y el alcance. Mientras no se respondan, el fork sigue congelado.

1. ¿Alguien usa hoy el modo doble clic con datos reales? ¿Desde cuándo y con cuántos despachos?
2. ¿Bodega prepara la carga la noche anterior? ¿Hay despachos que vuelven después de la medianoche?
3. ¿Cuál es el "orden tradicional" contra el que se mide el ahorro? Hoy es el orden en que se seleccionan las entregas.
4. ¿Qué gasto mensual en Google acepta la empresa?
5. ¿Bodega puede cancelar entregas sueltas y cambiar el personal de despachos cerrados?
6. ¿Cuál es el repositorio canónico y quién revisa los PR?
7. ¿Hay fecha para el piloto y una persona responsable en la empresa?
8. ¿Qué licencia tiene el proyecto, o debe ser privado? La documentación incluye la dirección real de la bodega.
9. ¿Qué partes de este fork se integran (sección 3)?
