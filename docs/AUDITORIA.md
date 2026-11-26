# Auditoría de BHUMI (2026-11-26)

Alcance: `main` (fases 0 a 3 y OpenSpec) y la rama `fase-4/bodega-piloto`, que sigue abierta. Cuatro revisores independientes leyeron el código en paralelo, uno por área, y comprobaron cada hallazgo leyendo el código o con una prueba. Supuestos: empresa pequeña (menos de 10 usuarios, 2 vehículos, una bodega) y piloto en las próximas semanas.

**Veredicto:** BHUMI está bien diseñado para su escala, pero todavía no está listo para el piloto. Siete fallas lo bloquean, todas de poco esfuerzo, y ningún hallazgo es crítico. La causa de fondo es el proceso: cuatro fases en menos de dos horas, sin revisión de otra persona ni uso real entre una fase y otra.

Informes detallados:
- [Arquitectura y decisiones de fondo](auditoria/arquitectura.md) (ARQ-01 a ARQ-17)
- [Datos y lógica de negocio](auditoria/datos-negocio.md) (DAT-01 a DAT-23)
- [Calidad, pruebas, interfaz y operación](auditoria/calidad-operacion.md) (CAL-01 a CAL-20)
- Seguridad (SEG-01 a SEG-16): aquí va solo el resumen. El detalle, con los pasos para explotar cada falla, se comparte aparte hasta que estén corregidas.

## Decisiones de fondo

| Decisión | Recomendación |
|---|---|
| Modo doble clic | Congelarlo ya. Después del piloto, retirar el modo real y dejar solo la demo |
| SQLite | Mantenerlo, fijar la versión de Node y documentar `better-sqlite3` como reemplazo |
| JS sin compilación ni framework | Mantenerlo, con `@ts-check` + JSDoc, ESLint y pruebas con Playwright |
| Solo Google Maps | Mantenerlo, con caché por filas, tráfico solo al guardar y un despacho de contingencia con km informados |

## Bloqueantes del piloto

| # | Problema | Hallazgos |
|---|---|---|
| 1 | El servidor calcula en UTC: el tráfico se consulta 3 h antes y desde las 21:00 "hoy" pasa a mañana | ARQ-01, DAT-01, DAT-02, CAL-02 |
| 2 | La hoja de ruta en PDF muestra los costos a Bodega (Fase 4) | SEG-01, ARQ-03 |
| 3 | La lista negra de campos deja pasar el desgaste en $/km a Bodega (Fase 4) | SEG-02, DAT-19 |
| 4 | El login no limita el largo del correo y permite llenar la base sin autenticarse | SEG-03, SEG-12 |
| 5 | Ante un corte de red, la interfaz manda a "restaurar el último respaldo" | CAL-03 |
| 6 | Restaurar un JSON parcial vacía toda la base | DAT-04, CAL-11 |
| 7 | Una misma entrega puede quedar en dos despachos, y un producto sin peso apaga el control de capacidad | ARQ-02, DAT-03, DAT-05 |

## Resumen de seguridad

No se encontró nada crítico: ningún bypass de permisos (se probaron 30 variantes de ruta), ni inyección SQL, ni XSS explotable, ni escalada de rol.

| ID | Tema | Severidad |
|---|---|---|
| SEG-03 | Largo del correo en el login | Alta |
| SEG-01, SEG-02 | Costos visibles para Bodega (Fase 4) | Media |
| SEG-04, SEG-05 | Consumo de recursos al importar Excel y en cubicaje | Media |
| SEG-06 | Clave de Google visible si se despliega sin las variables de entorno | Media |
| SEG-07 | Límite de intentos de login por cuenta | Media |
| SEG-08 a SEG-16 | Endurecimiento: restauración, CSP, CSRF, SSE, tamaño de cuerpos, permisos menores, dependencias, Docker | Baja |

## Plan de acción

0. **Acordar el flujo de trabajo** (1 a 2 días): repositorio canónico; `main` protegida con CI y revisión de otra persona; PR de menos de 400 líneas; versiones etiquetadas. *Puerta:* flujo acordado con el autor.
1. **Corregir la Fase 4 antes de integrarla** (1 día): PDF sin costos, lista permitida de campos por rol y una prueba de Bodega que pruebe de verdad. *Puerta:* Bodega no ve montos, y las pruebas lo comprueban.
2. **Bloqueantes del piloto** (2 a 3 días): los de la tabla anterior, además de índices, `construir` sin secretos, la demo reparada, la imagen de Node fijada y límites contra DoS. *Puerta:* prueba de humo aprobada.
3. **Piloto de una semana:** medir el gasto de Google, observabilidad mínima, restauración probada y despliegue por versiones. *Puerta:* criterios de cierre de `docs/PILOTO.md`.
4. **Decisiones con los datos del piloto:** retirar el doble clic, contingencia sin Google, dividir la API, migraciones numeradas, CSP y pruebas e2e. PostgreSQL queda en espera.

Las acciones y su modelo y esfuerzo recomendados se cargan como issues en el fork una vez que el autor apruebe el plan.
