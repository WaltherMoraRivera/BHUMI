# integracion-google Specification

## Purpose
Usar Google Maps Platform (Maps JavaScript, Geocoding y Routes) como única fuente de mapa, coordenadas, distancias y tiempos, controlando el gasto y protegiendo las claves.

## Requirements

### Requirement: Google como única fuente
Distancias, tiempos y coordenadas SHALL provenir de Google; el sistema MUST NOT calcular distancias de respaldo inventadas fuera del modo demostración.

#### Scenario: Google no responde
- **WHEN** la consulta a Routes falla
- **THEN** el plan informa el error de Google clasificado y no calcula con distancias aproximadas

### Requirement: Consultas eficientes y caché
Las matrices de rutas SHALL dividirse en llamadas de a lo más 625 elementos, considerar tráfico y hora de salida, y guardarse en caché para no pagar dos veces la misma consulta.

#### Scenario: Consulta repetida
- **WHEN** se planifica dos veces la misma ruta con la misma hora
- **THEN** la segunda vez se usan las distancias en caché

### Requirement: Claves separadas en el servidor central
Con claves por variables de entorno, la clave del servidor (Geocoding y Routes) MUST NOT enviarse al navegador y la del navegador SHALL limitarse al mapa; si se define solo una de las dos, el servidor MUST NOT arrancar y explica el problema. Sin claves en variables se usa la guardada en Configuración.

#### Scenario: Una sola clave definida
- **WHEN** se define GOOGLE_SERVER_KEY sin GOOGLE_BROWSER_KEY
- **THEN** el servidor no arranca y explica qué falta

### Requirement: Probar conexión
El sistema SHALL permitir probar la clave e informar por separado si Geocoding y Routes responden.

#### Scenario: Routes desactivada
- **WHEN** la clave tiene Geocoding activa pero no Routes
- **THEN** la prueba informa Geocoding correcto y Routes con error
