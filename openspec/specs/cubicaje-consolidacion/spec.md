# cubicaje-consolidacion Specification

## Purpose
Calcular cuántas unidades caben en cada vehículo según medidas y peso, y sugerir entregas cercanas que conviene juntar en un solo despacho.

## Requirements

### Requirement: Cubicaje con medidas reales
El sistema SHALL acomodar cajas en el espacio útil del vehículo (largo × ancho × alto) respetando si cada producto se puede apilar o girar y el límite de peso, sin superponer cajas ni salirse del espacio, e informar volumen, peso y qué limita.

#### Scenario: Cubo perfecto
- **WHEN** se cubican cajas de 50 cm en un espacio de 1 m³
- **THEN** caben 8 y la novena no cabe

#### Scenario: Producto sin medidas
- **WHEN** un producto o vehículo no tiene medidas
- **THEN** no se cubica y el sistema pide las medidas faltantes

### Requirement: Capacidad por producto
El sistema SHALL informar, por producto y vehículo, la capacidad por volumen, por peso y el límite real (el menor).

#### Scenario: Producto pesado
- **WHEN** un producto cabe 200 veces por volumen y 120 por peso
- **THEN** el límite real informado es 120

### Requirement: Consolidación de entregas
El sistema SHALL agrupar entregas pendientes a menos del radio configurado, indicar qué vehículo las lleva completas o en cuántos despachos se reparten, y estimar el ahorro usando solo distancias de Google ya consultadas.

#### Scenario: Sin distancias en caché
- **WHEN** las distancias entre las entregas agrupadas nunca se consultaron
- **THEN** se sugiere el grupo sin estimar ahorro

### Requirement: Utilización de la carga
El sistema SHALL calcular la utilización de peso y volumen de los despachos completados por vehículo.

#### Scenario: Despacho completado
- **WHEN** un despacho completado llevó 600 kg en un vehículo de 1000 kg
- **THEN** la utilización por peso informada es 60 %
