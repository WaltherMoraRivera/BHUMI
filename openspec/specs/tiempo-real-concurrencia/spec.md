# tiempo-real-concurrencia Specification

## Purpose
En el servidor central, mostrar al instante a cada persona lo que cambian las demás y evitar que una edición simultánea pise el trabajo de otra.

## Requirements

### Requirement: Canal de avisos en tiempo real
El servidor SHALL anunciar por un canal de eventos que exige sesión, tras cada escritura exitosa, qué recurso cambió, su id, la acción, el usuario y la pestaña de origen, sin incluir los datos. Los cálculos que no guardan y las escrituras con error MUST NOT anunciarse.

#### Scenario: Escritura rechazada
- **WHEN** una escritura responde con error
- **THEN** no se envía ningún aviso

### Requirement: Vigencia de la sesión en el canal
El canal SHALL comprobar periódicamente la sesión sin renovarla, y MUST cortar la conexión con un aviso "sin_sesion" si el usuario se desactiva o la sesión vence.

#### Scenario: Pestaña abierta sin uso
- **WHEN** una pestaña queda abierta 12 h sin que la persona haga nada
- **THEN** la sesión vence aunque el canal siga conectado

### Requirement: Refresco sin interrumpir
La interfaz SHALL recargar la sección visible cuando un aviso la afecta, y MUST NOT recargar si hay un diálogo abierto, un formulario con cambios sin guardar o el cursor en un campo; en ese caso muestra una barra "Actualizar". Los avisos de la propia pestaña y usuario se ignoran.

#### Scenario: Formulario abierto
- **WHEN** otra persona guarda el vehículo que Ana está editando
- **THEN** Ana ve la barra "Actualizar" y no pierde lo que escribió

### Requirement: Control de versiones optimista
Los registros editables SHALL llevar una versión que sube en cada cambio; al guardar con una versión vieja el servidor MUST responder 409 con el nombre de quien hizo el cambio (o "Otra persona" si no hay coincidencia segura, o la misma persona en otra pestaña) sin pisar el valor guardado. Los tiempos de un despacho no cuentan como cambio del despacho; sus paradas sí.

#### Scenario: Conflicto entre dos personas
- **WHEN** el Planificador guarda rendimiento 14 y luego Ana guarda 9 con la versión anterior
- **THEN** Ana recibe el aviso de conflicto con el nombre del Planificador y el servidor conserva 14

#### Scenario: Anotar tiempos mientras otro cierra
- **WHEN** Bodega anota horas de un despacho y luego el Planificador lo cierra
- **THEN** el cierre no genera conflicto

### Requirement: Una conexión por navegador
Las pestañas de un mismo navegador SHALL compartir una sola conexión al canal y otra pestaña MUST tomarla si la primera se cierra; ante errores, la reconexión reintenta con esperas crecientes de hasta 30 s.

#### Scenario: Reinicio del servidor
- **WHEN** el servidor se reinicia con la página abierta
- **THEN** la conexión vuelve sola y la pantalla se recarga por si se perdió algún cambio
