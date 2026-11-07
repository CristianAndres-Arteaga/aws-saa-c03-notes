# Amazon SNS y fan-out SNS → SQS

**Dominio:** RES | PERF | SEC
**Semana:** 04

## Qué es

El **megáfono** de AWS. Un publisher publica **una vez** en un **topic** y SNS **empuja** (modelo **push**) una copia a **todos** los suscriptores al instante. **No guarda nada:** si un suscriptor no estaba disponible y se agotan los reintentos, el mensaje se pierde.

```
                                     ┌──► Lambda
Publisher ──Publish──► [ Topic SNS ] ─┼──► Email / SMS
                                     └──► Endpoint HTTPS
           (cada suscriptor recibe SU copia, al instante)
```

## Cuándo usarlo

- **Notificar a varios sistemas o personas a la vez** (pub/sub): el publisher no sabe quién escucha.
- Alertas a humanos: **email, SMS, push móvil** (por ejemplo, alarmas de CloudWatch → SNS → email).
- **Fan-out SNS → SQS:** varios consumidores **y** sin perder mensajes (ver abajo).
- Un mismo evento de S3 hacia varios destinos → **S3 → SNS → varias SQS**.

## Cuándo NO usarlo (y qué usar en su lugar)

- Un consumidor que procesa **a su ritmo** y no debe perder trabajos → **SQS** (o SNS → SQS si además hay varios).
- Eventos de **SaaS** (Zendesk, Shopify…), reglas de enrutamiento complejas o tareas programadas tipo cron → **EventBridge**.
- Streaming en tiempo real con **replay** → **Kinesis Data Streams**.
- Orquestar pasos con estados y ramas → **Step Functions**.

## Datos clave

**Básicos**
- **Topic** = canal; **subscription** = destino que recibe copia.
- Tipos de suscriptor: **SQS, Lambda, HTTP/HTTPS, email, SMS, push móvil, Kinesis Data Firehose**.
- **Sin retención.** Hay **reintentos** (*delivery policy*): insistentes hacia SQS y Lambda, configurables para HTTP/S. Sirven para fallos **cortos**, no para un consumidor caído durante horas.
- Mensaje: **256 KiB** (más grande → **Extended Client Library + S3**, igual que SQS).
- Escala: hasta **12.5 M suscripciones por topic**, 100 000 topics por cuenta.
- Costo: por publicación y por entrega. SQS y Lambda baratos; **SMS y email** más caros.

**Fan-out SNS → SQS (el patrón estrella)**

```
                                ┌──► [Cola SQS A] ◄── Servicio de correos
Publisher ──► [ Topic SNS ] ────┼──► [Cola SQS B] ◄── Servicio de auditoría
                                └──► [Cola SQS C] ◄── Servicio de reportes
```

- Publicas **una vez**; cada cola guarda **su propia copia** (hasta 14 días) y cada consumidor lee a su ritmo, con sus reintentos y su DLQ.
- **Durabilidad:** si un consumidor se cae, sus mensajes esperan en su cola.
- **Independencia:** un consumidor lento no frena a los demás.
- **Extensible:** sumar un sistema nuevo = crear una cola y suscribirla, **sin tocar el publisher**.
- ⚠️ La **política de acceso de cada cola SQS** debe permitir `sqs:SendMessage` al topic. Si falta, no llega nada.

**Filtrado de mensajes (*message filtering*)**
- **Filter policy en JSON en cada suscripción** (los "audífonos con filtro"): SNS solo entrega lo que cumple el filtro.
- Por defecto filtra sobre los **message attributes**; también puede filtrar sobre el **cuerpo** del mensaje.
- Ejemplos: `{"estado": ["tarde"]}`, `{"monto": [{"numeric": [">=", 10000]}]}`.
- Evita crear un topic por tipo de mensaje y evita filtrar en el código del consumidor (que gasta procesamiento y dinero).

**SNS FIFO**
- **Orden estricto** (por `MessageGroupId`) + **deduplicación**, como SQS FIFO.
- Nombre termina en **`.fifo`**.
- Suscriptores: **solo colas SQS** (FIFO o Standard). No admite email, SMS ni HTTP.
- **SNS FIFO → varias SQS FIFO** = fan-out con orden y sin duplicados.

**Entregas fallidas**
- Cuando se agotan los reintentos: **DLQ (cola SQS) configurada en la suscripción**, no en el topic.

**Seguridad**
- En tránsito: HTTPS. En reposo: **SSE con KMS**.
- **IAM policies:** quién de tu cuenta puede publicar o suscribirse.
- **Topic policy** (política de recurso, como una bucket policy): acceso **entre cuentas** o de **servicios de AWS** (S3, CloudWatch…) que publican en el topic.
- ⚠️ Topic cifrado con **clave KMS administrada por el cliente** + publisher que es un servicio de AWS → ese servicio necesita `kms:GenerateDataKey*` y `kms:Decrypt` en la **key policy**. La topic policy sola no alcanza. La clave `aws/sns` no sirve para esto porque su key policy no se puede editar.

## Se confunde con…

### SQS vs SNS vs EventBridge
| | SQS | SNS | EventBridge |
|---|---|---|---|
| Analogía | Buzón | Megáfono | Central de correos inteligente |
| Modelo | **Pull** | **Push** | Push con **reglas** |
| ¿Guarda? | Sí (1 min a 14 días) | **No** | No (sí permite archivo y replay) |
| ¿Quién recibe? | **Un** consumidor por mensaje | **Todos** los suscriptores (con filtro opcional) | Los destinos cuyas reglas coinciden |
| Fuerte en | Absorber picos, trabajo asíncrono | Notificar a muchos, email/SMS | Eventos de AWS y SaaS, cron, enrutar por contenido |

### Filter policy vs FIFO
| | Filter policy | SNS FIFO |
|---|---|---|
| Problema que resuelve | El suscriptor **recibe mensajes que no le sirven** | Mensajes **desordenados o duplicados** |
| Dónde se configura | En la **suscripción** | En el **topic** (`.fifo`) |
| `MessageGroupId` | No aplica | Agrupa para mantener el orden; **no excluye** mensajes |

## Trampas de examen

**Frases → respuesta**
- "notificar a varios sistemas/personas a la vez", "email/SMS" → **SNS**
- "varios consumidores" + "sin perder mensajes" / "a velocidades distintas" → **SNS → SQS fan-out**
- "menor acoplamiento", "agregar consumidores sin cambiar la app" → **fan-out** (no que la app escriba en N colas)
- "un consumidor solo debe recibir ciertos mensajes" → **filter policy en la suscripción**
- "fan-out con orden y sin duplicados" → **SNS FIFO → SQS FIFO**
- "mismo evento de S3 a varios destinos" → **S3 → SNS → varias SQS** (o EventBridge)
- "activaron cifrado KMS y dejaron de llegar las alarmas" → **permisos en la key policy** para el servicio que publica

**Opciones trampa**
- **Una sola cola SQS** para varios sistemas → cada mensaje lo procesa **uno**; los demás no lo ven.
- **SNS → HTTPS directo** cuando se pide no perder mensajes → no hay retención.
- **La app escribe en N colas** → funciona, pero acopla y puede dejar envíos parciales.
- **FIFO / `MessageGroupId` para filtrar** → FIFO ordena, no filtra.
- **Lambda intermedia para filtrar** → más código y costo para algo nativo de SNS.
- **Quitar el cifrado o pasar a `aws/sns`** para arreglar permisos → baja la seguridad y no lo resuelve.

**Método (Día 11)**
1. Primero clasifica el problema: ¿**se pierden** mensajes (falta SQS), **llegan los que no sirven** (filter policy) o llegan **desordenados o duplicados** (FIFO)?
2. En errores de cifrado, pregunta **quién publica** y si esa identidad tiene permiso en la **key policy**.

## Lo vi en mis proyectos

- **TMS:** el módulo 11 (monitoring) usó **CloudWatch Alarms → SNS → email**, el caso de uso clásico de notificar a personas.
- **loan-collections:** buen candidato para un fan-out: `PagoRecibido` → topic `pagos` → colas de actualizar cuota, recibo por correo y auditoría; y una filter policy `{"estado": ["tarde"]}` para la cola del equipo de cobranza.
