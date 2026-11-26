# clientes-zonas Specification

## Purpose
Administrar clientes georreferenciados, la concentración de entregas por comuna y zona, y la evaluación de seguridad de cada sector para planificar entregas con menor riesgo.

## Requirements

### Requirement: Clientes y catálogo de Chile
El sistema SHALL guardar clientes con nombre, dirección y comuna validada contra el catálogo de 16 regiones y 346 comunas, permitir búsqueda e importación desde Excel con plantilla, y MUST rechazar filas inválidas indicando el motivo.

#### Scenario: Importación con una comuna inexistente
- **WHEN** se importa un Excel con una fila cuya comuna no existe
- **THEN** esa fila se informa como inválida y las demás se importan

### Requirement: Geolocalización con Google
Las coordenadas de clientes y del punto de partida SHALL obtenerse de Google Geocoding con restricción a Chile; al guardar la clave de Google se geolocaliza automáticamente solo el punto de partida.

#### Scenario: Guardar la clave
- **WHEN** el administrador guarda la clave de Google
- **THEN** se geolocaliza el punto de partida y no se geolocaliza ningún otro registro

### Requirement: Seguridad de sectores
Cada sector (comuna) SHALL tener una evaluación de seguridad (sin evaluar, normal, con precaución, alto riesgo) con su fuente y horario recomendado, e incidentes por tipo; el sistema MUST mostrar los incidentes de los últimos 90 días.

#### Scenario: Ruta por un sector con precaución
- **WHEN** una ruta incluye paradas en un sector con precaución, con incidentes o nunca evaluado
- **THEN** el plan muestra un aviso con esos sectores

### Requirement: Concentración de entregas
El sistema SHALL calcular la concentración de entregas por comuna y por zona contando solo despachos completados.

#### Scenario: Despacho cancelado
- **WHEN** un despacho fue cancelado
- **THEN** sus entregas no cuentan en la concentración
