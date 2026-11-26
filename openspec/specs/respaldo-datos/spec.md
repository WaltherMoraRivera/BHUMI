# respaldo-datos Specification

## Purpose
Permitir respaldar y restaurar los datos de BHUMI para no perderlos y para moverlos entre equipos o entre la versión de doble clic y el servidor central.

## Requirements

### Requirement: Respaldo completo y respaldo JSON
El sistema SHALL ofrecer un respaldo completo (.db, incluye la clave de Google y, en servidor, usuarios y auditoría; solo Administrador) y un respaldo JSON con todos los datos de negocio que MUST NOT incluir la clave de Google, usuarios, contraseñas ni sesiones.

#### Scenario: Respaldo JSON en servidor
- **WHEN** el Administrador descarga el respaldo JSON del servidor
- **THEN** el archivo no contiene la clave de Google, usuarios ni sesiones

### Requirement: Restauración segura
Restaurar SHALL reemplazar todos los datos actuales conservando las versiones del respaldo, y un archivo que no es un respaldo válido MUST rechazarse sin tocar nada. En servidor solo se restaura JSON y no se tocan los usuarios.

#### Scenario: Archivo inválido
- **WHEN** se elige un archivo que no es un respaldo
- **THEN** se rechaza y los datos actuales quedan intactos

### Requirement: Avisos y copia interna en el navegador
En la versión de doble clic el sistema SHALL avisar cuando pasan varios días sin respaldo y guardar una copia interna automática (como mucho una por hora) que usa si la base principal no se puede abrir.

#### Scenario: Base principal dañada
- **WHEN** la base del navegador no se puede abrir
- **THEN** el sistema abre la copia interna más reciente

### Requirement: Migración automática de esquema
Una base creada con una versión anterior SHALL actualizarse sola al abrirla, sin perder datos.

#### Scenario: Base antigua
- **WHEN** se abre una base sin las columnas de una versión nueva
- **THEN** las columnas se agregan y los datos existentes se conservan
