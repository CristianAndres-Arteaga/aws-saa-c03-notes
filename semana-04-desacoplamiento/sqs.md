# Amazon SQS: Standard vs FIFO, visibility timeout, DLQ y long polling

**Dominio:** RES | PERF | COST
**Semana:** 04

## Qué es

La **bandeja de pendientes** entre dos componentes. El productor deja mensajes, la cola los guarda y el consumidor los recoge **cuando puede** (modelo **pull**). Si el consumidor se cae o va lento, los mensajes esperan y no se pierden.

```
Productor ──SendMessage──► [ Cola SQS ] ◄──ReceiveMessage── Consumidor
 (web/API)                  (guarda)                       (EC2/Lambda)
                                          ──DeleteMessage──► (al terminar)
```

## Cuándo usarlo

- **Desacoplar** dos niveles que van a velocidades distintas (web → workers) y absorber picos.
- Procesamiento **asíncrono**: la web responde al instante y el trabajo pesado se hace después.
- **Escalar workers** con un Auto Scaling Group según la cantidad de mensajes en la cola.
- **FIFO:** cuando importan el **orden** y que **no haya duplicados** (pagos, pedidos, comandos secuenciales).

## Cuándo NO usarlo (y qué usar en su lugar)

- Un mensaje que deben recibir **varios sistemas a la vez** → **SNS** con fan-out a varias colas SQS (no una sola cola compartida).
- **Streaming en tiempo real**, varios consumidores leyendo el mismo dato o **replay** de datos → **Kinesis Data Streams**.
- Enrutar eventos según su contenido o reaccionar a eventos de servicios AWS o SaaS → **EventBridge**.
- Orquestar pasos con estados, ramas y reintentos → **Step Functions**.
- Migrar una app que usa **JMS/AMQP/MQTT** sin reescribirla → **Amazon MQ**.

## Datos clave

**Generales**
- Totalmente administrado, sin servidores; se cobra **por solicitud**.
- **Retención:** 4 días por defecto, configurable de **1 min a 14 días**.
- **Tamaño de mensaje:** **256 KiB** es el número clásico del examen (desde 2025 el máximo es 1 MiB). Para más grande: **Extended Client Library**, que guarda el payload en **S3** y manda el puntero por la cola.
- El consumidor **debe borrar** el mensaje (`DeleteMessage`) al terminar; si no, reaparece.
- Cifrado en reposo con **SSE-SQS** o **SSE-KMS**; acceso con IAM y **políticas de cola**.

**Standard vs FIFO**

| | Standard | FIFO |
|---|---|---|
| Throughput | Prácticamente ilimitado | **300 msg/s**, **3.000 con batching** (más con high-throughput) |
| Entrega | **At-least-once** (puede duplicar) | **Exactly-once processing** (deduplica en **5 min**) |
| Orden | Best-effort | **Estricto por `MessageGroupId`** |
| Nombre | Libre | Termina en **`.fifo`** |
| Requisito | Consumidor **idempotente** | Aceptar menos throughput |

- **`MessageGroupId`:** orden garantizado **dentro** de cada grupo; los grupos distintos van **en paralelo**. Un solo grupo para todo significa todo en fila india.
- **Deduplicación:** `MessageDeduplicationId` o **content-based deduplication** (hash del cuerpo).
- **Standard no se convierte a FIFO:** hay que crear una cola nueva.

**Visibility timeout**
- Al recibir un mensaje, se oculta por **30 s** por defecto (rango **0 s a 12 h**).
- Si no se borra antes de que venza, **reaparece** y otro consumidor lo procesa de nuevo.
- Regla: el timeout debe ser **mayor que el tiempo de procesamiento**. `ChangeMessageVisibility` lo extiende para un mensaje concreto.
- Con **Lambda** como consumidor: timeout de la cola **≥ 6 ×** el timeout de la función.

**Dead-letter queue (DLQ)**
- **Redrive policy** + **`maxReceiveCount`**: después de N recepciones sin borrar, el mensaje se mueve a la DLQ.
- Aísla las **poison messages** (mal formadas, siempre fallan) sin perderlas.
- Debe ser **del mismo tipo** (FIFO → FIFO) y tener **más retención** que la cola origen (en Standard la expiración cuenta desde que el mensaje entró a la cola original).
- **DLQ redrive:** devolver los mensajes a la cola origen cuando se arregla el bug.
- Alarma de CloudWatch sobre los mensajes visibles de la DLQ.

**Polling**
- **Short polling** (por defecto, 0 s): responde al instante consultando solo parte de los servidores y puede devolver respuestas vacías que también se cobran.
- **Long polling:** `WaitTimeSeconds` de **1 a 20 s**. Menos respuestas vacías, **menos costo**, menos latencia real. A nivel de cola: `ReceiveMessageWaitTimeSeconds`.

**Delay queues**
- `DelaySeconds` de **0 a 15 min**: el mensaje no es visible hasta que pasa ese tiempo.
- Por mensaje se llama **message timer** (solo en Standard).

**Escalar con la cola**
- ASG de workers con la métrica de CloudWatch **`ApproximateNumberOfMessagesVisible`**, idealmente como **backlog por instancia** (target tracking).

## Se confunde con…

### Visibility timeout vs delay queue
| | Visibility timeout | Delay queue |
|---|---|---|
| Cuándo oculta | **Después** de que alguien lo lee | **Antes** de la primera lectura |
| Para qué | Evitar que dos consumidores lo procesen a la vez | Posponer el procesamiento |
| Rango | 0 s a 12 h (defecto 30 s) | 0 a 15 min |

### SQS vs SNS (adelanto del tema 2)
| | SQS | SNS |
|---|---|---|
| Modelo | **Pull**: el consumidor pregunta | **Push**: SNS entrega a los suscriptores |
| Persistencia | Guarda el mensaje hasta 14 días | No guarda: entrega y listo |
| Consumidores | Cada mensaje lo procesa **uno** | **Todos** los suscriptores lo reciben |

## Trampas de examen

**Frases → respuesta**
- "desacoplar", "absorber picos", "procesamiento asíncrono" → **SQS**
- "orden" + "sin duplicados" → **FIFO**; si además dice "en paralelo por cliente/cuenta" → **`MessageGroupId` por cliente**
- "se procesa varias veces" + el proceso tarda más de 30 s → **subir el visibility timeout**
- "mensajes que siempre fallan", "apartarlos para analizarlos", "sin perderlos" → **DLQ + `maxReceiveCount`**
- "muchas respuestas vacías", "reducir costo de polling" → **long polling (20 s)**
- "esperar X minutos antes de procesar" → **delay queue** (≤ 15 min)
- "escalar workers según la carga de la cola" → **ASG + métrica de backlog de SQS**
- "mensajes de más de 256 KiB" → **Extended Client Library + S3**

**Opciones trampa**
- FIFO con **un solo grupo** cuando piden paralelismo → orden perfecto pero sin paralelismo.
- **FIFO para arreglar reprocesos** por visibility timeout → la deduplicación actúa al enviar, no evita que el mensaje reaparezca.
- **DLQ con `maxReceiveCount` = 1** → manda a cuarentena mensajes sanos al primer timeout.
- **Visibility timeout gigante** (12 h) para poison messages → solo los esconde; vuelven a fallar.
- **Una sola cola** para que varios sistemas reciban el mismo mensaje → cada mensaje lo procesa uno solo; eso es **SNS fan-out**.

**Método (Día 10)**
1. Explica la respuesta con la **causa y sus números**, no con el síntoma: "30 s por defecto < 2 min de proceso" es más sólido que "porque se procesan varias veces".
2. En FIFO, revisa **las tres condiciones**: orden, duplicados y paralelismo.

## Lo vi en mis proyectos

- Ninguno lo usó todavía. TMS llamaba a Lambda de forma síncrona desde API Gateway y loan-collections procesaba todo dentro de la misma petición. Un buen candidato habría sido generar reportes o enviar recordatorios de cobranza de forma asíncrona con SQS + workers.
