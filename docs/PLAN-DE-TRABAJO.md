# Análisis y plan de trabajo de BHUMI

Fecha: 2026-11-26 · Base analizada: `main` (fases 1 a 3) y la rama `fase-4/bodega-piloto` (PR 4 del repositorio original, abierto).
Los issues viven en el fork `WaltherMoraRivera/BHUMI`.

## De qué trata

BHUMI es un sistema para controlar y reducir los costos de transporte y distribución de una empresa chilena. Calcula cuánto cuesta realmente cada entrega, recomienda rutas con Google Maps, mide el desempeño de bodega y conductores, controla la flota, cubica la carga y consolida despachos. Su regla central: **solo datos reales**. Si falta un dato, el indicador queda en blanco y el sistema dice qué falta.

| Pieza | Qué hay |
|---|---|
| Servidor | Node ≥ 22.13, Express 4, `node:sqlite`; 82 rutas en `server/api.js` y 18 servicios de negocio |
| Interfaz | JavaScript sin compilación, una vista por archivo en `public/js/views/` |
| Navegador | `navegador/` simula Express, SQLite (sql.js/WASM), PDFKit y ExcelJS para el modo de doble clic |
| Despliegue | Docker + Caddy (HTTPS) + Litestream (respaldo continuo a S3) |
| Pruebas | 109 pruebas (`node --test`) en cuatro modos: HTTP, local, navegador y wasm |

### Tres modos de ejecución, un solo código
1. **Doble clic** (`Abrir BHUMI.html`): todo dentro del navegador, un usuario, sin login.
2. **Demostración**: datos y Google simulados, en memoria.
3. **Servidor central**: multiusuario con roles (Administrador, Planificador, Bodega), auditoría, tiempo real (SSE) y control de versiones optimista (409).

### Evolución
| Fase | Estado | Contenido |
|---|---|---|
| 0 | En `main` | Versión de navegador completa (7 pilares) |
| 1 | Integrada | Servidor central, Docker, claves de Google separadas |
| 2 | Integrada | Login, roles, auditoría |
| 3 | Integrada | Tiempo real y edición simultánea sin pisar cambios |
| 4 | **PR abierto** | Pantalla "Hoy" para bodega, Bodega sin costos, CI, guía del piloto |
| 5 | Idea | PostgreSQL + PostGIS si el piloto lo justifica |

## Diagnóstico

**Fortalezas**
- Documentación y PR muy completos, con decisiones y pendientes explícitos.
- Buena cobertura de la API y de las reglas de negocio, con pruebas que verifican "no inventar datos".
- Seguridad pensada: scrypt, cookies seguras, normalización de rutas antes de los permisos, CSRF por Origin, bloqueo por intentos que no deja sin acceso a una oficina.

**Riesgos y deuda**
- `server/api.js` concentra ~1000 líneas y 82 rutas.
- El esquema crece con `COLUMNAS_NUEVAS`, sin migraciones versionadas.
- El filtro de costos para Bodega (Fase 4) es una lista negra por nombre de campo: falla abierto.
- Sin CSP; sin pruebas automáticas de la interfaz; sin linter; sin licencia.
- Los scripts `test:local`, `test:navegador` y `test:wasm` no corren en Windows.
- Mantener el modo navegador duplica la superficie de pruebas (cuatro modos por cambio).
- La reconexión del tiempo real detrás de Caddy (502) nunca se probó.

## OpenSpec

- `openspec/config.yaml`: contexto del proyecto y reglas para propuestas, specs y tareas.
- `openspec/specs/`: 13 capacidades con el comportamiento actual de `main` (datos-reales, costos-transporte, planificacion-rutas, despachos-tiempos-kpi, flota-vehiculos, clientes-zonas, cubicaje-consolidacion, panel-analisis, autenticacion-roles, tiempo-real-concurrencia, respaldo-datos, integracion-google, modos-ejecucion).
- `openspec/changes/fase-4-bodega-piloto/`: la Fase 4 documentada como cambio en curso (11/15 tareas).
- Skills para Claude Code en `.claude/skills/openspec-*`: `/openspec-propose`, `/openspec-apply-change`, `/openspec-archive-change`, etc.

Ciclo para cada issue con cambio de comportamiento: `/openspec-propose` → revisión → `/openspec-apply-change` → PR → `openspec archive`.

## Criterio de modelo y esfuerzo

| Modelo | Cuándo |
|---|---|
| **Opus** | Diseño, arquitectura, seguridad, revisiones, decisiones con datos reales en juego |
| **Sonnet** | Implementación estándar con especificación clara |
| **Haiku** | Tareas mecánicas, acotadas y con verificación obvia |

Esfuerzo **Bajo / Medio / Alto** = nivel de razonamiento del agente (low / medium / high), según incertidumbre y radio de impacto, no según el tamaño del diff.

## Plan por hitos

| # | Issue | Hito | Modelo | Esfuerzo |
|---|---|---|---|---|
| [1](https://github.com/WaltherMoraRivera/BHUMI/issues/1) | Integrar OpenSpec: specs base y flujo de cambios | M0 | Sonnet | Bajo |
| [2](https://github.com/WaltherMoraRivera/BHUMI/issues/2) | Scripts de prueba fallan en Windows | M0 | Haiku | Bajo |
| [3](https://github.com/WaltherMoraRivera/BHUMI/issues/3) | Linter y formateador con CI | M0 | Sonnet | Medio |
| [4](https://github.com/WaltherMoraRivera/BHUMI/issues/4) | Plantillas de issue/PR y CONTRIBUTING | M0 | Haiku | Bajo |
| [5](https://github.com/WaltherMoraRivera/BHUMI/issues/5) | Definir la licencia | M0 | Haiku | Bajo |
| [6](https://github.com/WaltherMoraRivera/BHUMI/issues/6) | Revisión de código de la Fase 4 | M1 | Opus | Alto |
| [7](https://github.com/WaltherMoraRivera/BHUMI/issues/7) | Cerrar la Fase 4 (CI 22.13, integrar, archivar) | M1 | Haiku | Bajo |
| [8](https://github.com/WaltherMoraRivera/BHUMI/issues/8) | Infraestructura del piloto | M2 | Sonnet | Medio |
| [9](https://github.com/WaltherMoraRivera/BHUMI/issues/9) | Prueba de humo (reconexión tras 502) | M2 | Sonnet | Medio |
| [10](https://github.com/WaltherMoraRivera/BHUMI/issues/10) | Simulacro de restauración Litestream | M2 | Sonnet | Medio |
| [11](https://github.com/WaltherMoraRivera/BHUMI/issues/11) | Evaluar el piloto y priorizar | M2 | Opus | Medio |
| [12](https://github.com/WaltherMoraRivera/BHUMI/issues/12) | Dividir `server/api.js` | M3 | Opus | Alto |
| [13](https://github.com/WaltherMoraRivera/BHUMI/issues/13) | Migraciones versionadas | M3 | Opus | Alto |
| [14](https://github.com/WaltherMoraRivera/BHUMI/issues/14) | CSP y encabezados de seguridad | M3 | Sonnet | Medio |
| [15](https://github.com/WaltherMoraRivera/BHUMI/issues/15) | Filtrar avisos SSE por rol | M3 | Sonnet | Medio |
| [16](https://github.com/WaltherMoraRivera/BHUMI/issues/16) | Bodega sin costos con lista permitida | M3 | Opus | Medio |
| [17](https://github.com/WaltherMoraRivera/BHUMI/issues/17) | Pruebas end-to-end en navegador | M3 | Sonnet | Alto |
| [18](https://github.com/WaltherMoraRivera/BHUMI/issues/18) | Actualizar Express 5, PDFKit, Chart.js | M3 | Sonnet | Medio |
| [19](https://github.com/WaltherMoraRivera/BHUMI/issues/19) | Tipado con JSDoc + checkJs | M3 | Sonnet | Medio |
| [20](https://github.com/WaltherMoraRivera/BHUMI/issues/20) | Evaluar PostgreSQL + PostGIS | M4 | Opus | Alto |
| [21](https://github.com/WaltherMoraRivera/BHUMI/issues/21) | Futuro de la versión de doble clic | M4 | Opus | Medio |

**Ruta crítica al piloto:** 6 → 7 → 8 → 9 → 11. Lo de M0 puede avanzar en paralelo; M3 conviene después de integrar la Fase 4 para no generar conflictos con su PR.
