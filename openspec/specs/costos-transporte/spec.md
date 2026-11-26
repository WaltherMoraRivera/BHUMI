# costos-transporte Specification

## Purpose
Calcular el costo modelado de cada ruta y despacho (combustible, desgaste y otros) y compararlo con el costo real gastado por la empresa, por concepto, por km y por entrega.

## Requirements

### Requirement: Costo modelado de una ruta
El sistema SHALL calcular el costo de una ruta como combustible + desgaste + otros, donde combustible = km ÷ rendimiento × precio por litro vigente, desgaste = km × suma de componentes por km (neumáticos, mantención, depreciación, otros) y otros = peajes + estacionamientos + costo del conductor (horas normales y horas extra con recargo).

#### Scenario: Ruta con datos completos
- **WHEN** un vehículo con rendimiento y componentes de desgaste recorre 100 km con precio de combustible vigente
- **THEN** el sistema entrega combustible, desgaste, otros y total, y el costo por km resultante

#### Scenario: Sin precio de combustible
- **WHEN** no hay precio de combustible registrado para la fecha del despacho
- **THEN** el costo de combustible y el total quedan en blanco y se indica que falta el precio

### Requirement: Desgaste derivado de los datos del vehículo
Cuando un componente de desgaste no tiene valor manual, el sistema SHALL calcularlo a partir de los datos del vehículo si es posible (por ejemplo, precio y vida útil de neumáticos).

#### Scenario: Componente vacío calculable
- **WHEN** el vehículo no tiene costo de neumáticos por km pero sí precio del juego y vida útil en km
- **THEN** el costo por km de neumáticos se deriva de esos datos

### Requirement: Precio de combustible con vigencia
El precio del combustible SHALL guardarse con fecha de vigencia, y cada cálculo MUST usar el precio vigente en la fecha del despacho.

#### Scenario: Cambio de precio a mitad de mes
- **WHEN** se registra un nuevo precio con vigencia desde el día 15
- **THEN** los despachos anteriores al 15 usan el precio anterior y los posteriores el nuevo

### Requirement: Control de costos real frente a modelado
El sistema SHALL comparar, por período y por concepto, el costo modelado con el costo real (cargas de combustible + gastos registrados de mantención, neumáticos, seguros, permisos y peajes + conductor modelado), y MUST mostrar la diferencia y el costo real por km y por entrega.

#### Scenario: Sin cargas de combustible
- **WHEN** en el período no hay cargas de combustible registradas
- **THEN** el sistema no calcula un total real de combustible y lo informa como dato faltante

### Requirement: Asignación de costos a clientes
El costo de un despacho asignado a sus clientes SHALL sumar exactamente el total del despacho.

#### Scenario: Reparto con redondeo
- **WHEN** el costo de un despacho de 3 clientes no se divide en partes enteras
- **THEN** la suma de los montos asignados es igual al total del despacho

### Requirement: Consumo real por estanque lleno
El sistema SHALL calcular el consumo real de un vehículo por el método de estanque lleno a partir de las cargas de combustible y el odómetro.

#### Scenario: Dos cargas a estanque lleno
- **WHEN** existen dos cargas a estanque lleno con sus odómetros
- **THEN** el consumo real es los litros de la segunda carga divididos por los km recorridos entre ambas
