# planificacion-rutas Specification

## Purpose
Planificar despachos a partir de entregas o clientes: calcular alternativas de ruta con Google Maps, recomendar la más económica frente al orden tradicional y entregar al conductor el recorrido georreferenciado.

## Requirements

### Requirement: Cuatro alternativas de ruta
Al planificar, el sistema SHALL calcular con distancias y tiempos de Google cuatro alternativas (menor distancia, menor tiempo, menor costo y balanceada), compararlas con el orden tradicional ingresado y recomendar la más económica.

#### Scenario: Planificación completa
- **WHEN** el planificador selecciona entregas pendientes y un vehículo con datos completos
- **THEN** el sistema muestra las cuatro alternativas con km, tiempo y costo, el ahorro frente al orden tradicional y la recomendada

#### Scenario: Sin clave de Google
- **WHEN** no hay clave de Google configurada
- **THEN** el sistema no planifica, explica que falta la clave y no usa distancias aproximadas

### Requirement: Optimización del orden de paradas
El sistema SHALL encontrar el orden óptimo exacto hasta 15 paradas y MUST usar una heurística que incluya todas las paradas cuando haya más, con o sin regreso a la base.

#### Scenario: Más de 15 paradas
- **WHEN** se planifica una ruta de 20 paradas
- **THEN** la ruta incluye las 20 paradas y su costo queda cerca del óptimo

### Requirement: Límite de paradas y división en despachos
Una ruta SHALL tener a lo más 25 paradas (límite de Google Routes); con más paradas o con carga sobre la capacidad del vehículo, el sistema MUST dividir la selección en varios despachos.

#### Scenario: Selección de 30 entregas
- **WHEN** se seleccionan 30 entregas para un mismo vehículo
- **THEN** el sistema propone dividirlas en despachos de a lo más 25 paradas

### Requirement: Bloqueo por capacidad y costo incalculable
El sistema MUST NOT guardar un despacho cuya carga no cabe en el vehículo (peso o volumen) ni uno cuyo costo no se puede calcular por datos faltantes.

#### Scenario: Carga excede la capacidad
- **WHEN** la carga de las entregas supera la capacidad en kg del vehículo
- **THEN** el plan avisa el exceso y no permite guardar el despacho

### Requirement: Ruta para el conductor
El sistema SHALL entregar el recorrido del despacho en orden, partiendo desde la base, como enlaces de Google Maps encadenados en tramos de a lo más 9 paradas intermedias, como texto para WhatsApp y como hoja de ruta en PDF.

#### Scenario: Despacho de 12 paradas
- **WHEN** se genera la ruta para un despacho de 12 paradas
- **THEN** se entregan dos enlaces de Google Maps encadenados que recorren las 12 paradas en orden
