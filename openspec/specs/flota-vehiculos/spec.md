# flota-vehiculos Specification

## Purpose
Llevar el kilometraje semanal, las cargas de combustible, el desgaste y la mantención preventiva de cada vehículo de la flota.

## Requirements

### Requirement: Kilometraje semanal
El sistema SHALL registrar los km de cada vehículo por semana (semana que parte el lunes) y MUST sumar automáticamente a la semana correspondiente los km de los despachos completados.

#### Scenario: Despacho completado
- **WHEN** se completa un despacho de 80 km el miércoles
- **THEN** los 80 km se suman a la semana que comienza el lunes anterior

### Requirement: Odómetro solo aumenta
El odómetro de un vehículo MUST NOT disminuir al registrar gastos, cargas o km.

#### Scenario: Odómetro menor al actual
- **WHEN** se registra un gasto con un odómetro menor al último conocido
- **THEN** el odómetro del vehículo conserva el valor mayor

### Requirement: Mantención preventiva
El sistema SHALL clasificar la mantención de cada vehículo como al día, próxima o vencida según el odómetro, el intervalo y los km por semana; una próxima mantención indicada en el registro MUST mandar sobre el intervalo, y una fecha cumplida la deja vencida aunque falten km.

#### Scenario: Fecha cumplida
- **WHEN** la fecha de la próxima mantención ya pasó y faltan km para el intervalo
- **THEN** la mantención figura como vencida

### Requirement: Desgaste de neumáticos
El sistema SHALL informar el porcentaje de vida consumido de los neumáticos desde el último cambio.

#### Scenario: Vehículo sin registros
- **WHEN** un vehículo no tiene cambio de neumáticos registrado
- **THEN** el desgaste queda en blanco
