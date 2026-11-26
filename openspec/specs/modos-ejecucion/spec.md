# modos-ejecucion Specification

## Purpose
Definir los tres modos en que corre BHUMI con el mismo código: doble clic en el navegador (un usuario), demostración (datos y Google simulados) y servidor central (multiusuario con Docker).

## Requirements

### Requirement: Versión de doble clic autosuficiente
El archivo "Abrir BHUMI.html" SHALL contener el programa completo y funcionar sin instalar nada en Chrome o Edge, guardando los datos en el navegador (no en la carpeta), sin inicio de sesión y para una sola persona.

#### Scenario: Mover la carpeta
- **WHEN** el usuario mueve o renombra la carpeta de BHUMI y vuelve a abrir el archivo en el mismo navegador
- **THEN** sus datos siguen disponibles

### Requirement: Modo demostración aislado
El modo demostración SHALL usar datos de prueba, un mapa simulado y distancias simuladas (línea recta × 1,3 a 35 km/h), vivir solo en memoria, MUST NOT tocar los datos reales e identificarse con la etiqueta "Modo demostración".

#### Scenario: Cerrar la demostración
- **WHEN** el usuario cierra la pestaña de demostración
- **THEN** todo lo hecho se borra y los datos reales no cambian

### Requirement: Configuración del servidor central
El servidor SHALL leer HOST, PORT, DB_FILE, TRUST_PROXY y las claves de Google de variables de entorno; sin variables MUST quedar como antes (solo 127.0.0.1:3000 y data/bhumi.db), rechazar un puerto inválido y advertir si escucha en otros equipos sin claves de Google por variables.

#### Scenario: Puerto inválido
- **WHEN** PORT no es un número de puerto válido
- **THEN** el servidor no arranca y explica el problema

### Requirement: Salud y cierre ordenado
El servidor SHALL exponer un endpoint de salud que responde ok con la versión mientras la base funciona (503 si no), y ante SIGTERM o SIGINT MUST dejar de aceptar solicitudes, cerrar las conexiones en vivo y cerrar la base.

#### Scenario: docker stop
- **WHEN** Docker detiene el contenedor con conexiones en vivo abiertas
- **THEN** el proceso termina con código 0 en pocos segundos

### Requirement: Base compartida con espera
En disco, la base SHALL esperar hasta 5 s si otro proceso la tiene tomada en lugar de fallar de inmediato.

#### Scenario: Litestream replicando
- **WHEN** Litestream replica mientras llegan escrituras concurrentes
- **THEN** las escrituras se completan sin "database is locked"
