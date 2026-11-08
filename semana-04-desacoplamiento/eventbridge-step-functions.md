# Amazon EventBridge y AWS Step Functions

**Dominio:** RES | PERF | COST
**Semana:** 04

## Qué es

- **EventBridge** es la **central de correos con clasificador**. Llegan eventos de servicios AWS, de tus apps y de SaaS. Unas **reglas** leen el **contenido** de cada evento y lo envían a los destinos que correspondan. Reparte eventos, pero **no sabe** qué pasó después.
- **Step Functions** es el **director de orquesta** con la partitura. Una **máquina de estados** define qué paso va primero, qué hacer si algo falla, qué corre en paralelo y cuándo **esperar**. En todo momento sabe en qué paso va cada ejecución, y la consola lo muestra **visualmente**.

```
 Fuentes                     Event Bus                    Targets
 Servicios AWS ──┐       ┌───────────────────┐         ┌──► Lambda
 Tus apps ───────┼─────► │ Regla 1: patrón   │─────────┼──► SQS / SNS
 (PutEvents)     │       │ Regla 2: patrón   │         ├──► Step Functions
 SaaS (partner) ─┘       │ Regla 3: cron     │         └──► Bus de otra cuenta
                         └───────────────────┘
```

**Pareja típica:** EventBridge **dispara** y Step Functions **orquesta**.

## Cuándo usarlo

**EventBridge**
- Eventos de **SaaS de terceros** (Shopify, Zendesk, Datadog, Auth0…) **sin servidores ni polling** → **partner event bus**.
- Reaccionar a **cambios de estado de servicios AWS** (EC2 *stopped*, objeto en S3…) o a **llamadas a la API** registradas por CloudTrail.
- **Enrutar por contenido** con reglas ricas hacia muchos tipos de servicio.
- Tareas **programadas sin servidores** (cron/rate) → **EventBridge Scheduler**.
- **Reprocesar** eventos pasados → **archive & replay**.
- Centralizar eventos de varias cuentas → **bus de otra cuenta** como target.

**Step Functions**
- Flujos de **varios pasos** con **ramas**, **paralelismo**, **reintentos** y **compensaciones**.
- Flujos **largos** (hasta 1 año) o con **aprobación humana**.
- Procesar **cada elemento** de una lista o de millones de objetos de S3 (**Map / Distributed Map**).

## Cuándo NO usarlo (y qué usar en su lugar)

- Buffer para absorber picos, consumidor a su ritmo → **SQS**.
- Fan-out masivo y simple, o email/SMS/push móvil → **SNS**.
- Streaming con **orden y replay de alto volumen** → **Kinesis Data Streams**.
- "Orquestar" con colas SQS entre pasos o con Lambdas que se invocan entre sí → frágil y mucho código: usa **Step Functions**.

## Datos clave

**EventBridge**
- **Event bus**, 3 tipos: **default** (eventos de servicios AWS, llegan solos), **custom** (tus apps) y **partner** (SaaS integrados).
- **Regla + event pattern**: filtro JSON sobre el contenido, por ejemplo `{"source": ["aws.ec2"], "detail": {"state": ["stopped"]}}`. Un evento puede cumplir varias reglas.
- **Targets:** hasta **5 por regla**, más de 20 tipos de servicio. Reintentos y **DLQ** por target.
- **EventBridge Scheduler:** cron, rate y tareas de una sola vez.
- **Archive & replay**, **schema registry**, **Pipes** (fuente → filtro → enriquecimiento → destino).
- Evento máximo: **256 KB**.
- Costo: eventos de servicios AWS al bus default **gratis**; eventos custom y de partners ~**USD 1 por millón**.

**Step Functions**
- Definición en JSON (**ASL**, Amazon States Language).

| Estado | Qué hace |
|---|---|
| **Task** | Ejecuta trabajo: Lambda o llamada **directa** a más de 200 servicios (DynamoDB, SQS, ECS, Glue…) |
| **Choice** | Un `if`: elige rama según los datos |
| **Parallel** | Varias ramas a la vez |
| **Map** | Mismos pasos por cada elemento (**Distributed Map**: millones de objetos en S3) |
| **Wait** | Espera un tiempo o hasta una fecha |
| **Pass / Succeed / Fail** | Pasar datos, terminar bien o mal |

- **Retry** (con *backoff*) y **Catch** (ir a otra rama si falla) declarados en cada estado, sin código propio.
- **`.waitForTaskToken`**: el flujo se **pausa** y entrega un token; continúa cuando alguien lo devuelve. Sirve para **aprobación humana** o sistemas externos.
- **Seguridad:** la state machine asume un **rol IAM**. Mínimo privilegio, solo los servicios que invoca.

**Standard vs Express** ⭐

| | **Standard** | **Express** |
|---|---|---|
| Duración máx. | **1 año** | **5 minutos** |
| Semántica | **Exactly-once** | **At-least-once** (async) / at-most-once (sync) |
| Volumen | Moderado | **Altísimo** (más de 100 000 ejecuciones/s) |
| Costo | Por **transición de estado** (~USD 25 por millón) | Por **ejecución + duración** |
| Historial | Visual en consola, 90 días | **CloudWatch Logs** |
| Típico | Pedidos, aprobaciones, ETL largos | IoT, streaming, microservicios de alto volumen |

Regla: **largo, crítico o con humanos → Standard**. **Corto, masivo, barato y con pasos idempotentes → Express**.

## Se confunde con…

### SNS vs EventBridge
| Si el escenario dice… | Respuesta |
|---|---|
| Fan-out simple, muchísimo volumen, email/SMS/push | **SNS** |
| Eventos de **SaaS** | **EventBridge** (partner bus) |
| Filtrar por **contenido** con reglas ricas, muchos tipos de destino | **EventBridge** |
| Cambios de estado de servicios AWS / llamadas a la API | **EventBridge** |
| Cron sin servidores | **EventBridge Scheduler** |
| Reprocesar eventos pasados | **EventBridge archive & replay** |

### Step Functions vs Lambdas encadenadas vs SQS
| | Step Functions | Lambdas encadenadas | SQS entre pasos |
|---|---|---|---|
| Esperar días | Sí (Standard, task token) | **No** (Lambda máx. 15 min) | Sí, pero sin noción de flujo |
| Reintentos / errores | Declarativos (Retry/Catch) | En tu código | Por cola (DLQ), sin ramas |
| Ver en qué paso va | **Visual** | No | No |
| Pagas esperas | No (Standard cobra por transición) | **Sí** (una Lambda esperando a otra) | No |

## Trampas de examen

**Frases → respuesta**
- "SaaS", "sin servidores ni polling" → **EventBridge partner event bus + regla**
- "cron", "cada noche", "sin servidores" → **EventBridge Scheduler** (no EC2 con crontab)
- "reaccionar cuando una instancia/recurso cambia de estado" → **regla de EventBridge**
- "reprocesar eventos" → **archive & replay**
- "varios pasos", "ramas", "reintentos", "ver en qué paso está" → **Step Functions**
- "aprobación manual", "puede tardar días" → **Standard + `.waitForTaskToken`**
- "alto volumen", "pasos cortos", "menor costo", "idempotente / duplicados aceptables" → **Express**

**Opciones trampa**
- **EC2 con cron que consulta una API** → servidor + polling.
- **SNS para recibir eventos de un SaaS** → no hay integración nativa; SNS publica hacia endpoints, no los recibe.
- **Lambda que recibe todo y descarta en código** → filtra tarde; la regla debe filtrar antes de invocar.
- **Express para algo que espera horas o días** → muere a los 5 minutos (la trampa más común).
- **Standard para decenas de miles de eventos/s** → funciona, pero el costo por transición se dispara.
- **Lambdas que se invocan entre sí** → límite de 15 min, lógica de errores en código, pagas esperas.
- **SQS + EC2 coordinador** → orquestador "hecho a mano".

**Método (Día 12)**
1. ¿Es **repartir** un evento (EventBridge/SNS) o **coordinar pasos** (Step Functions)?
2. Si es Step Functions, busca **números**: duración (> 5 min → Standard) y volumen (masivo y corto → Express).
3. "Idempotente" o "duplicados aceptables" en el enunciado es la pista de **at-least-once → Express**.

## Lo vi en mis proyectos

- **TMS:** no usó EventBridge ni Step Functions. Un buen añadido sería un **EventBridge Scheduler** que dispare cada noche una Lambda de reportes, en lugar de depender de una invocación manual.
- **loan-collections:** candidato a Step Functions Standard: `PagoRecibido` → validar pago → (Choice) si el monto supera un umbral, aprobación del supervisor con task token → Parallel: actualizar cuota | enviar recibo. El evento de pago podría llegar por EventBridge desde la pasarela de pagos.
