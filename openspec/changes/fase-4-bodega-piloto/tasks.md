# Tasks

Estado según el PR MaxisotooH/BHUMI#4. Las tareas marcadas ya están implementadas en la rama `fase-4/bodega-piloto`.

## 1. Bodega sin costos

- [x] 1.1 Filtro `sinCostos` en `server/auth.js` para toda respuesta JSON del rol Bodega
- [x] 1.2 Excepciones `precio` y `monto`; quitar `alternativas`, `baseline` y `baseline_total` del plan guardado
- [x] 1.3 Corregir pantallas que leían costos anidados sin `?.` (Vehículos, ficha de cliente)
- [x] 1.4 Pruebas en `test/bodega.test.js`

## 2. Interfaz según el rol

- [x] 2.1 `GET /api/auth/yo` entrega `permisos` serializados
- [x] 2.2 `data-requiere` en botones de escritura y `MutationObserver` que los oculta
- [x] 2.3 `data-costo` en columnas y tarjetas de costo, ocultas con `rol-bodega`

## 3. Pantalla "Hoy"

- [x] 3.1 Vista `public/js/views/hoy.js` con botones de al menos 48 px y una columna en pantallas chicas
- [x] 3.2 Sección inicial de Bodega y refresco en vivo

## 4. Integración continua y piloto

- [x] 4.1 Workflow `.github/workflows/pruebas.yml` (Node 22.13 y 24, construir, imagen Docker)
- [x] 4.2 Guía `docs/PILOTO.md`
- [ ] 4.3 Revisar el resultado de la primera corrida de CI con Node 22.13
- [ ] 4.4 Revisión de código del PR e integración a `main`
- [ ] 4.5 Correr las cuatro formas de prueba y `npm run construir` sobre `main` ya integrado
- [ ] 4.6 Archivar este cambio con `openspec archive fase-4-bodega-piloto`
