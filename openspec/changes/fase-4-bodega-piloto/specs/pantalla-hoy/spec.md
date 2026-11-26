## Purpose
Permitir que el personal de bodega anote desde el celular o la tablet del andén, con un toque, cada momento de los despachos del día.

## ADDED Requirements

### Requirement: Despachos del día con un botón por momento
La pantalla "Hoy" SHALL listar los despachos del día con un botón por cada momento (preparación, carga, salida y regreso; este último solo si el despacho vuelve a la base) y resaltar el siguiente paso.

#### Scenario: Despacho sin regreso a la base
- **WHEN** un despacho del día no vuelve a la base
- **THEN** su tarjeta no muestra el botón de regreso

### Requirement: Anotar con un toque
Un toque en un momento SHALL anotar la hora actual, y "Otra hora" o "Corregir" MUST permitir poner otra hora o quitarla, respetando el orden cronológico de los tiempos.

#### Scenario: Marcar el inicio de la preparación
- **WHEN** Bodega toca "Empieza la preparación"
- **THEN** se guarda la hora actual y el siguiente paso queda resaltado

### Requirement: Uso móvil y en vivo
La pantalla SHALL tener botones de al menos 48 px, una sola columna en pantallas chicas, actualizarse en vivo cuando otra persona marca algo, y ser la sección inicial de Bodega (los demás roles también pueden abrirla).

#### Scenario: Otro operador marca la carga
- **WHEN** otro operador marca el fin de carga de un despacho visible
- **THEN** la pantalla muestra la hora sin recargar manualmente

#### Scenario: Entrada de Bodega
- **WHEN** un operador de bodega inicia sesión
- **THEN** llega directamente a "Hoy"
