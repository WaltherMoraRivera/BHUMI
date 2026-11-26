## ADDED Requirements

### Requirement: Bodega no recibe costos
Toda respuesta JSON dirigida al rol Bodega, incluidas las de sus propias escrituras, MUST omitir a cualquier profundidad los campos de dinero (los que empiezan con costo, ahorro o valor_, más peajes, estacionamientos, otros_gastos, depreciacion, precio_combustible, nota_ahorro y las alternativas y la línea base del plan guardado). El precio por litro del combustible y el monto de las cargas que registra el propio operador SHALL mantenerse.

#### Scenario: Detalle de un despacho como Bodega
- **WHEN** un operador de bodega abre el detalle de un despacho planificado
- **THEN** la respuesta no trae costos, alternativas ni línea base del plan

#### Scenario: Mismo detalle como Administrador
- **WHEN** un Administrador abre el mismo despacho
- **THEN** la respuesta incluye los costos y las alternativas

#### Scenario: Carga de combustible registrada por Bodega
- **WHEN** Bodega registra una carga de combustible con su monto
- **THEN** la respuesta conserva el monto de esa carga

### Requirement: Reglas del rol para la interfaz
`GET /api/auth/yo` SHALL entregar las reglas de permisos del rol, idénticas a las que aplica el servidor, y la interfaz MUST ocultar los botones de acciones que el rol no puede usar y las columnas o tarjetas de costo para Bodega. Ocultar es solo visual: el servidor sigue decidiendo.

#### Scenario: Bodega en Vehículos
- **WHEN** un operador de bodega abre Vehículos
- **THEN** no ve "Agregar vehículo" ni "Editar datos"

#### Scenario: Llamada directa a la API
- **WHEN** Bodega llama directamente a una escritura que su rol no permite
- **THEN** el servidor responde 403 aunque el botón esté oculto
