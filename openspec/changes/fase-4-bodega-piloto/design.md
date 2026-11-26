## Context

Tras las fases 1 a 3, el servidor central ya aplica permisos por rol, pero las respuestas a Bodega incluían montos y la interfaz mostraba acciones que el servidor rechazaba. El piloto exige que Bodega anote horas en el momento desde el andén.

## Goals / Non-Goals

**Goals:**
- Ningún monto en pesos llega al navegador de Bodega, aunque una pantalla nueva olvide ocultarlo.
- La interfaz oculta lo que el rol no puede hacer usando las mismas reglas del servidor.
- Anotar una hora con un toque desde un celular.
- CI que corra las cuatro formas de prueba y verifique la imagen Docker.

**Non-Goals:**
- Cambiar la lógica de permisos del servidor (solo se serializa).
- Filtrar los avisos SSE por rol.

## Decisions

- **Filtro de costos en el servidor (`sinCostos`)** en lugar de en cada endpoint: se aplica a toda respuesta JSON para Bodega, por nombre de campo (`costo*`, `ahorro*`, `valor_*`, `peajes`, `estacionamientos`, `otros_gastos`, `depreciacion`, `precio_combustible`, `nota_ahorro`, y `alternativas`/`baseline`/`baseline_total` del plan). Ventaja: defensa por defecto. Riesgo: un campo de costo con nombre nuevo se filtraría solo si sigue la convención; las pantallas deben leer campos de costo con `?.`.
- **Excepciones explícitas**: `precio` del combustible (público) y `monto` de las cargas que el operador registra.
- **Reglas serializadas en `/auth/yo`** y `data-requiere="MÉTODO /ruta"` en cada botón, con un `MutationObserver` que oculta también lo que aparece después. La interfaz solo oculta; el servidor decide.
- **CI con Node 22.13 y 24**: la mínima declarada en `engines` y la de la imagen Docker.

## Risks / Trade-offs

- [El filtro por nombre puede dejar pasar un campo de dinero mal nombrado] → prueba que recorre las 20 respuestas legibles por Bodega con datos de demostración y falla si aparece un campo de costo.
- [La primera corrida con Node 22.13 ocurre recién en CI] → revisar el resultado del workflow antes de integrar.
- [La reconexión ante un 502 de Caddy no se probó en desarrollo] → paso obligatorio de la prueba de humo del piloto.
