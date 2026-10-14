# Ecosistema de Automatización IA para un estudio contable

Proyecto final del curso **Automatización con IA** (Coderhouse). Automatiza el circuito de vencimientos impositivos de un estudio contable argentino: lee comprobantes de pago (VEP de ARCA), extrae los datos con IA, los registra en Airtable, **pide aprobación humana** al contador y recién entonces avisa al cliente por **Telegram**.

> Todos los datos (clientes, CUIT, montos) son de prueba e inventados.

## Cómo funciona

```
Google Drive (carpeta ENTRADA VEP)
  └─ E1 · Extracción (Make.com)
       Download → convertir a Google Doc (carpeta Procesados) → Get Content → Set variable
       → HTTP POST Cohere (command-r-plus-08-2024, prompt dinámico) → Parse JSON → Router
          ├─ Ruta 1 (datos completos): Search Clientes por CUIT → Create Vencimiento
          │        (Estado = Procesado por IA) → Gmail al contador   ← HITL #1
          └─ Ruta 2 (requiere revisión): Create Errores                ← HITL #2
       Error handler en HTTP/Cohere → Create Errores

El contador revisa el mail y cambia Estado a «Aprobado por Humano» en Airtable
  └─ E2 · Envío al cliente (Make.com)
       Airtable Watch ({Estado}='Aprobado por Humano') → Get Client → Telegram Send Message
       → Update Vencimiento (Estado = Enviado al Cliente, Fecha_Envio = hoy)
       Error handler en Telegram → Create Errores
```

Flujo de estados: `Pendiente → Procesado por IA → Aprobado por Humano → Enviado al Cliente → Pagado`.

## Entregables

| # | Entregable | Archivo |
|---|-----------|---------|
| 1 | Diagrama de arquitectura | [`docs/1_Diagrama_Arquitectura.pdf`](docs/1_Diagrama_Arquitectura.pdf) |
| 2 | Manual de datos (tablas vinculadas + esquemas JSON) | [`docs/2_Manual_Datos.pdf`](docs/2_Manual_Datos.pdf) |
| 3 | Matriz de optimización de costos | [`docs/3_Matriz_Costos.pdf`](docs/3_Matriz_Costos.pdf) |
| 4 | Seguridad y resiliencia (incluye 6 pruebas de estrés) | [`docs/4_Seguridad_Resiliencia.pdf`](docs/4_Seguridad_Resiliencia.pdf) |
| 5 | Dashboard público (vista compartida de Airtable) | https://airtable.com/appHRyWw9WN7XRVTP/shrU1jx7ZrOl2LLwB |
| – | Base de datos (solo lectura) | _(link incluido en la entrega)_ |
| – | Blueprints de Make | [`blueprints/`](blueprints) |
| – | Capturas de las pruebas | [`screenshots/`](screenshots) |
| – | Video (3 min) | _(link incluido en la entrega)_ |

## Pruebas realizadas

| # | Caso | Resultado | Evidencia |
|---|------|-----------|-----------|
| 1 | VEP F931 válido | Ruta 1 → Vencimiento ID 8 + mail de aprobación | `test1_*` |
| 2 | Comprobante de pago (no es VEP) | Ruta 2 → Errores, sin vencimiento | `test2_*` |
| 3 | Clave de API inválida | Error handler → registro «API IA» | `test3_*` |
| 4 | PDF sin texto | Ruta 2 → Errores (ID 14) | `test4_*` |
| 5 | VEP F931 repetido | Ruta 1 → Vencimiento ID 9 | `test5_*` |
| 6 | E2 con Vencimiento 8 aprobado | Mensaje de Telegram + Estado «Enviado al Cliente» | `e2_*` |

## Decisiones de diseño

- **IA propone, humano aprueba:** ningún cliente recibe mensajes sin que el contador pase el Estado a «Aprobado por Humano».
- **Anti-bucle:** el Google Doc convertido se guarda en una carpeta fuera de la vigilada; en E2, al enviar, el Estado deja de cumplir la fórmula del trigger.
- **Filtros numéricos y de patrón:** `monto > 0` (número contra número) y CUIT con patrón `^[0-9]{11}$`.
- **Prompt dinámico:** una sola variable (`texto_limpio`) sobre un prompt con reglas de dominio (SICOSS/SUSS/Empleadores = F931, formato de período, importes en formato argentino).
- **Modelo por tarea:** modelo grande solo donde un error cuesta (identificar al cliente); ver matriz de costos.

## Limitaciones conocidas

Detalladas en `docs/4_Seguridad_Resiliencia.pdf`: sin deduplicación de VEP repetidos, los registros nuevos no se vinculan solos a la tabla Indicadores (dashboard), el VEP no trae la fecha de vencimiento del impuesto (se calcularía con Calendario_Fiscal) y el informe mensual quedó fuera de esta entrega.

## Nota de seguridad

El blueprint de E1 incluye la clave de Cohere usada en las pruebas (el docente indicó que redactarla es opcional). La clave se regenera al cerrar la entrega. El token del bot de Telegram vive en la conexión de Make y no está en los blueprints.
