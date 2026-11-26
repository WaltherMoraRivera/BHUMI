# autenticacion-roles Specification

## Purpose
En el servidor central, identificar a cada persona con correo y contraseña, aplicar en el servidor los permisos de su rol (Administrador, Planificador, Operador de bodega) y dejar registro auditable de cada escritura. No aplica a la versión de doble clic.

## Requirements

### Requirement: Contraseñas y sesiones seguras
Las contraseñas SHALL guardarse con scrypt y sal aleatoria, nunca en texto plano. La sesión MUST viajar en una cookie HttpOnly, SameSite=Lax y Secure bajo HTTPS, la base MUST guardar solo el hash del token, y la sesión vence tras 12 h sin uso o 7 días como máximo.

#### Scenario: Sesión inactiva
- **WHEN** una sesión pasa 12 h sin uso
- **THEN** la siguiente solicitud responde 401 y la interfaz vuelve al inicio de sesión con un aviso

### Requirement: Protección del inicio de sesión
El sistema SHALL responder con el mismo mensaje y el mismo tiempo para un correo inexistente o una contraseña incorrecta, bloquear un correo en una IP 15 minutos tras 5 fallos, y bloquear la IP completa solo tras 50 fallos.

#### Scenario: Oficina con IP compartida
- **WHEN** una persona falla 5 veces con su correo desde la IP de la oficina
- **THEN** solo ese correo queda bloqueado en esa IP y sus compañeros pueden entrar

### Requirement: Rutas públicas y CSRF
Solo salud, meta, login y logout SHALL ser públicas; todo lo demás MUST exigir sesión (401), y las escrituras con un Origin distinto al host MUST rechazarse.

#### Scenario: Escritura desde otro sitio
- **WHEN** llega un PUT con Origin de otro dominio y cookie válida
- **THEN** la escritura se rechaza

### Requirement: Permisos por rol en el servidor
El servidor SHALL revisar en cada solicitud el permiso del rol sobre la ruta normalizada (sin distinguir mayúsculas, sin barra final, sin barras dobles y decodificada). El Administrador puede todo; el Planificador todo menos usuarios, auditoría, respaldo, probar Google y geolocalizar la base, y los ajustes de Admin que envíe MUST ignorarse.

#### Scenario: Variante de ruta del Planificador
- **WHEN** un Planificador llama a /USUARIOS o /auditoria/
- **THEN** recibe 403

#### Scenario: Planificador envía ajustes de Admin
- **WHEN** un Planificador envía la clave de Google en PUT /settings junto con otros ajustes
- **THEN** los demás ajustes se guardan y la clave se ignora

### Requirement: Lista permitida del Operador de bodega
El rol Bodega SHALL poder solo ver despachos, entregas, clientes, flota y cubicaje, y registrar tiempos, personal del despacho, inicio y cierre de despachos, estado de entregas, km semanal y cargas de combustible. Bodega MUST NOT cancelar despachos ni acceder a Panel, Control de costos, KPI, planificación o configuración.

#### Scenario: Bodega intenta cancelar
- **WHEN** un operador de bodega intenta cancelar un despacho
- **THEN** recibe 403 y el despacho no cambia

### Requirement: Gestión de usuarios
Solo el Administrador SHALL crear usuarios, cambiar nombre o rol, desactivar y asignar contraseña; desactivar o cambiar la contraseña MUST cerrar las sesiones del afectado, y MUST quedar siempre al menos un administrador activo. No existe registro público: el primer administrador se crea por línea de comandos.

#### Scenario: Último administrador
- **WHEN** se intenta desactivar o cambiar de rol al único administrador activo
- **THEN** la operación se rechaza

#### Scenario: Usuario desactivado conectado
- **WHEN** se desactiva a un usuario con sesión abierta
- **THEN** su siguiente solicitud responde 401

### Requirement: Auditoría
Cada escritura, inicio de sesión, intento fallido y bloqueo SHALL registrarse con usuario, método, ruta, resultado, IP y hora con milisegundos, consultable solo por el Administrador.

#### Scenario: Consulta de auditoría
- **WHEN** el Administrador abre la actividad reciente
- **THEN** ve quién escribió qué y los intentos fallidos de inicio de sesión
