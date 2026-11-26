# datos-reales Specification

## Purpose
Principio transversal de BHUMI: todo indicador, costo o distancia se calcula con datos reales ingresados por la empresa u obtenidos de Google; si falta un dato, el sistema lo dice en lugar de inventarlo.

## Requirements

### Requirement: Sin cifras inventadas
El sistema SHALL dejar en blanco cualquier indicador cuyo cálculo dependa de un dato faltante, y MUST indicar qué dato falta y dónde se ingresa.

#### Scenario: Vehículo sin rendimiento
- **WHEN** se calcula el costo de una ruta con un vehículo sin rendimiento km/L
- **THEN** el costo de combustible queda en blanco y el sistema indica que falta el rendimiento del vehículo

#### Scenario: Panel con datos incompletos
- **WHEN** el usuario abre el Panel de análisis y faltan datos de configuración
- **THEN** el panel muestra "Faltan N datos…" con la lista de datos pendientes y el lugar donde se ingresa cada uno

### Requirement: Arranque sin datos de ejemplo
El sistema SHALL iniciar en modo real solo con la flota base de la empresa (Chevrolet N400 y KYC X5) sin rendimiento ni costos, y MUST NOT precargar clientes, despachos ni cifras de ejemplo.

#### Scenario: Primera apertura
- **WHEN** se abre una base nueva en modo real
- **THEN** existen los dos vehículos base sin rendimiento ni costos y no existen clientes ni despachos

### Requirement: Promedios con su base de cálculo
Todo promedio de desempeño SHALL indicar entre cuántos datos registrados se calculó.

#### Scenario: KPI con pocos registros
- **WHEN** solo 3 de 10 despachos tienen horas de carga anotadas
- **THEN** el promedio de carga se calcula con esos 3 y se informa que se basa en 3 datos
