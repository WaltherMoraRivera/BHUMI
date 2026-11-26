# panel-analisis Specification

## Purpose
Comparar en un solo lugar rutas, conductores, clientes y vehículos de un período (costo total, por km y por entrega, cumplimiento, utilización y ahorro potencial) y exportarlo.

## Requirements

### Requirement: Período de análisis
Todas las vistas de análisis SHALL filtrarse por un período (mes o rango de fechas) con un valor por defecto.

#### Scenario: Sin período indicado
- **WHEN** se abre el panel sin elegir período
- **THEN** se usa el período por defecto

### Requirement: Vistas del panel
El panel SHALL ofrecer las vistas de rutas, conductores, clientes, vehículos, costos, cumplimiento y ahorro potencial con los datos del período, y MUST NOT inventar cifras cuando no hay datos.

#### Scenario: Período sin despachos
- **WHEN** el período elegido no tiene despachos
- **THEN** el panel muestra las vistas vacías sin cifras

### Requirement: Exportación
El panel, el control de costos, los KPI, la flota y las zonas SHALL exportarse a Excel y PDF con los mismos datos que muestra la pantalla.

#### Scenario: Exportar el panel
- **WHEN** el usuario exporta el panel a Excel
- **THEN** el archivo contiene las mismas cifras del período en pantalla
