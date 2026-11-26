## Why

Bodega veía el costo de cada despacho y botones que el servidor le rechazaba con 403, y no tenía una forma cómoda de anotar horas desde el andén. Además, el repositorio no tenía pruebas automáticas. Esta fase deja BHUMI listo para una semana de piloto con datos reales en el servidor central.

> Cambio documentado a posteriori: corresponde al PR MaxisotooH/BHUMI#4 (rama `fase-4/bodega-piloto`), abierto y pendiente de integrar. Este documento describe su contrato para revisarlo y archivarlo con OpenSpec una vez integrado.

## What Changes

- El servidor quita los campos de dinero de toda respuesta JSON dirigida al rol Bodega, a cualquier profundidad.
- `GET /api/auth/yo` entrega las reglas de permisos del rol; la interfaz oculta los botones que el rol no puede usar y las columnas o tarjetas de costo.
- Nueva pantalla "Hoy": despachos del día con un botón grande por cada momento (preparación, carga, salida, regreso), pensada para celular o tablet. Es la sección inicial de Bodega.
- Integración continua en GitHub Actions: las cuatro formas de prueba en Node 22.13 y 24, armado de la versión de doble clic y verificación de la imagen Docker.
- Guía del piloto (`docs/PILOTO.md`).

## Capabilities

### New Capabilities
- `pantalla-hoy`: registro rápido de los momentos del despacho del día desde el andén.

### Modified Capabilities
- `autenticacion-roles`: Bodega no recibe costos y la interfaz recibe las reglas del rol para ocultar acciones.

## Impact

- Modos: solo servidor central (la versión de doble clic no tiene roles; "Hoy" también está disponible ahí para el único usuario).
- Roles: Bodega (principal), Planificador y Administrador pueden abrir "Hoy".
- Código: `server/auth.js`, `public/js/sesion.js`, `public/js/views/hoy.js`, vistas con `data-requiere`/`data-costo`, `.github/workflows/pruebas.yml`, `docs/PILOTO.md`.
- Fuera de alcance: filtrar por rol los avisos de tiempo real; probar la reconexión ante un 502 de Caddy (queda en la prueba de humo del piloto).
