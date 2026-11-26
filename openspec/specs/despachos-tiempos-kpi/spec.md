# despachos-tiempos-kpi Specification

## Purpose
Registrar las horas reales y el resultado de cada despacho y entrega, y calcular con ellos los KPI del personal (preparación, carga, ruta, entrega y cumplimiento) frente a los objetivos definidos.

## Requirements

### Requirement: Ciclo de vida del despacho
Un despacho SHALL pasar por los estados planificado, en ruta y completado, o cancelado; al cerrarlo MUST registrarse el resultado de cada entrega (realizada, no realizada con su motivo, o reprogramada) y los km recorridos se suman al vehículo.

#### Scenario: Cierre con una entrega fallida
- **WHEN** se cierra un despacho con una entrega no realizada por "cliente ausente"
- **THEN** esa entrega guarda su propio motivo y el despacho queda completado

### Requirement: Tiempos reales en orden cronológico
El sistema SHALL registrar inicio y fin de preparación, inicio y fin de carga, salida, llegada y fin en cada cliente, y regreso, y MUST rechazar horas que rompan el orden cronológico.

#### Scenario: Hora de carga anterior a la preparación
- **WHEN** se anota un inicio de carga anterior al inicio de preparación
- **THEN** el sistema rechaza la hora y explica el orden esperado

### Requirement: Cálculo de KPI del personal
El sistema SHALL calcular preparación = fin − inicio de preparación, carga = fin − inicio de carga, tránsito = salida de un punto a llegada al siguiente, entrega = fin − llegada al cliente, ruta completa = salida → regreso y cumplimiento = entregadas ÷ programadas, por conductor y por mes, usando solo horas anotadas.

#### Scenario: Indicadores de un mes
- **WHEN** se consultan los KPI de marzo
- **THEN** solo se cuentan los despachos de marzo

#### Scenario: Sin objetivo definido
- **WHEN** no hay objetivo de carga configurado
- **THEN** el KPI de carga se muestra solo con los valores reales, sin comparación

### Requirement: Personal del despacho editable
El conductor y el personal de bodega asignados a un despacho SHALL poder cambiarse antes del cierre.

#### Scenario: Cambio de conductor
- **WHEN** se cambia el conductor de un despacho en ruta
- **THEN** los KPI de ese despacho se atribuyen al nuevo conductor
