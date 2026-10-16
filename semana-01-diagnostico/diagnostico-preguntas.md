# Diagnóstico SAA-C03: 20 preguntas

Preguntas de práctica originales con el estilo del examen. **No son las del Official Practice Question Set de AWS.**

**Reglas**
- Sin consultar nada: ni apuntes, ni Google, ni IA.
- Tiempo sugerido: **40 minutos** (2 minutos por pregunta, el mismo ritmo del examen real).
- Si no sabes una, descarta opciones y elige igual. En el examen no se resta por error.
- Al lado de cada respuesta marca **S** (estaba seguro) o **A** (adiviné).

Los nombres de servicios y algunos términos van en inglés, como en la versión en español del examen.

---

**1.** Una aplicación en instancias EC2 necesita leer objetos de un bucket S3. El equipo de seguridad prohíbe guardar credenciales de AWS en las instancias o en el código. ¿Qué debe hacer el arquitecto?

- A. Crear un usuario IAM y guardar sus access keys en variables de entorno de la instancia.
- B. Hacer público el bucket y restringir el acceso por la IP de la instancia.
- C. Crear un rol IAM con permiso de lectura sobre el bucket y asociarlo a las instancias mediante un instance profile.
- D. Guardar las access keys en el user data de la instancia y borrarlas después del arranque.

**2.** Una tienda online recibe pedidos en un nivel web y los procesa en un nivel de procesamiento que a veces se vuelve lento en horas pico. Hoy se pierden pedidos cuando el procesamiento no da abasto. La empresa necesita que **ningún pedido se pierda** y que cada nivel escale de forma independiente. ¿Qué solución cumple los requisitos?

- A. Poner una cola Amazon SQS entre los dos niveles y escalar los workers con un Auto Scaling group según la cantidad de mensajes en la cola.
- B. Publicar cada pedido en un tema de Amazon SNS con suscripción por correo electrónico.
- C. Cambiar las instancias del nivel de procesamiento por un tipo de instancia más grande.
- D. Hacer que el nivel web escriba los pedidos directamente en una base de datos RDS y que el procesamiento la consulte cada hora.

**3.** Una empresa guarda logs en Amazon S3. Los logs se consultan con frecuencia durante los primeros 30 días y casi nunca después. Deben conservarse 7 años por regulación. Pasado el primer año, recuperar un log en hasta 12 horas es aceptable. ¿Cuál es la solución **más económica**?

- A. Mantener todos los logs en S3 Standard durante los 7 años.
- B. Mover los logs a S3 One Zone-IA el primer día y dejarlos ahí.
- C. Usar S3 Intelligent-Tiering sin ninguna otra configuración.
- D. Crear una regla de ciclo de vida (lifecycle): S3 Standard-IA a los 30 días y S3 Glacier Deep Archive a los 365 días.

**4.** Una empresa de videojuegos tiene un juego multijugador que usa **UDP** y está desplegado en dos regiones de AWS. Los jugadores están en todo el mundo. La empresa necesita **direcciones IP estáticas** para que los clientes se conecten, menor latencia y conmutación por error rápida entre regiones. ¿Qué servicio debe usar?

- A. Amazon CloudFront
- B. AWS Global Accelerator
- C. Amazon Route 53 con política de enrutamiento simple
- D. Amazon S3 Transfer Acceleration

**5.** Una empresa tiene 40 cuentas de AWS administradas con AWS Organizations. Seguridad exige que **nadie en ninguna cuenta, ni siquiera el usuario root**, pueda desactivar AWS CloudTrail. ¿Qué debe hacer el arquitecto?

- A. Crear una política IAM que deniegue `cloudtrail:StopLogging` y asignarla a todos los usuarios de cada cuenta.
- B. Crear una regla de AWS Config que detecte cuando CloudTrail se desactiva.
- C. Aplicar una Service Control Policy (SCP) que deniegue `cloudtrail:StopLogging` y `cloudtrail:DeleteTrail` en la raíz de la organización.
- D. Asignar un permissions boundary a todos los roles de cada cuenta.

**6.** Una aplicación corre en la región principal us-east-1 y tiene una copia de recuperación ante desastres en us-west-2. El tráfico debe ir **siempre** a us-east-1 y pasar a us-west-2 **solo** si la región principal deja de responder. ¿Qué configuración de Amazon Route 53 cumple el requisito?

- A. Enrutamiento ponderado (weighted) con 50% para cada región.
- B. Enrutamiento por latencia (latency-based).
- C. Enrutamiento por geolocalización.
- D. Enrutamiento de conmutación por error (failover) con health checks en el registro principal.

**7.** Una base de datos Amazon RDS for MySQL soporta una aplicación web. El equipo de finanzas ejecuta reportes pesados de solo lectura que vuelven lenta la aplicación. ¿Qué debe hacer el arquitecto para **mejorar el rendimiento** de la aplicación?

- A. Crear una read replica y apuntar los reportes a ella.
- B. Activar Multi-AZ y apuntar los reportes a la instancia standby.
- C. Aumentar el tamaño del almacenamiento de la base de datos.
- D. Tomar snapshots cada hora y ejecutar los reportes sobre ellos.

**8.** Una empresa ejecuta cada noche un procesamiento por lotes (batch) de 4 horas. El trabajo tolera interrupciones: si una instancia se detiene, retoma desde el último punto guardado. ¿Qué opción de compra de EC2 es la **más económica**?

- A. Instancias On-Demand
- B. Reserved Instances estándar a 3 años
- C. Instancias Spot
- D. Dedicated Hosts

**9.** Una empresa debe cifrar en reposo los datos de un bucket S3. Auditoría exige **registrar cada uso de la clave** de cifrado y que la empresa controle la rotación y los permisos de la clave. ¿Qué opción cumple los requisitos con el **menor esfuerzo operativo**?

- A. Cifrado del lado del servidor con claves administradas por S3 (SSE-S3).
- B. Cifrado del lado del servidor con una clave KMS administrada por el cliente (SSE-KMS).
- C. Cifrado del lado del servidor con claves proporcionadas por el cliente (SSE-C).
- D. Cifrar cada archivo en la aplicación antes de subirlo, con una clave guardada en el código.

**10.** Varias instancias EC2 con Linux, repartidas en **tres zonas de disponibilidad**, necesitan leer y escribir al mismo tiempo en un **sistema de archivos compartido** compatible con POSIX. ¿Qué servicio cumple el requisito?

- A. Un volumen Amazon EBS con Multi-Attach.
- B. Amazon S3.
- C. Instance store de cada instancia.
- D. Amazon EFS.

**11.** Una empresa recibe miles de eventos de clics por segundo desde su sitio web. **Varias aplicaciones distintas** deben procesar los mismos eventos, en orden, y poder **volver a leerlos** (replay) hasta 24 horas después. ¿Qué servicio debe usar?

- A. Amazon Kinesis Data Streams
- B. Amazon SQS cola estándar
- C. Amazon SNS
- D. Amazon S3 con notificaciones de eventos

**12.** Instancias EC2 en una subred privada necesitan subir archivos a Amazon S3. El tráfico **no debe pasar por internet** y la empresa quiere evitar el costo de un NAT Gateway. ¿Qué debe hacer el arquitecto?

- A. Crear un NAT Gateway en una subred pública.
- B. Crear un gateway VPC endpoint para S3 y agregarlo a la tabla de rutas de la subred privada.
- C. Asignar IP públicas a las instancias y agregar un Internet Gateway.
- D. Crear una conexión AWS Site-to-Site VPN.

**13.** Una startup lanza una aplicación nueva con Amazon DynamoDB. No sabe cuánto tráfico tendrá y espera picos impredecibles. Quiere **pagar solo por lo que usa** sin planificar capacidad. ¿Qué modo de capacidad debe elegir?

- A. Capacidad aprovisionada (provisioned) con un valor alto fijo.
- B. Capacidad aprovisionada con reserved capacity.
- C. Capacidad bajo demanda (on-demand).
- D. Capacidad aprovisionada con escalado manual cada mañana.

**14.** Una aplicación web corre en **una sola** instancia EC2 en una zona de disponibilidad. La empresa necesita que sea **altamente disponible**. ¿Qué **DOS** acciones debe tomar? (Elige dos.)

- A. Crear un Auto Scaling group que lance instancias en al menos dos zonas de disponibilidad.
- B. Cambiar la instancia por un tipo más grande.
- C. Asociar una Elastic IP a la instancia.
- D. Poner un Application Load Balancer delante de las instancias, con subredes en varias zonas de disponibilidad.
- E. Lanzar las instancias en un placement group de tipo cluster.

**15.** Una empresa tiene servidores de archivos en su oficina. Quiere guardar los archivos en Amazon S3 pero que los usuarios sigan accediendo **por SMB/NFS desde la oficina**, con una **caché local** para los archivos más usados. ¿Qué servicio debe usar?

- A. AWS DataSync
- B. AWS Storage Gateway con S3 File Gateway
- C. AWS Snowball Edge
- D. AWS Direct Connect

**16.** Una aplicación web detrás de un Application Load Balancer recibe ataques de **inyección SQL** y **cross-site scripting (XSS)**. ¿Qué servicio debe usar el arquitecto para bloquearlos?

- A. AWS Shield Standard
- B. Security groups de las instancias
- C. Amazon GuardDuty
- D. AWS WAF

**17.** Una tabla de Amazon DynamoDB recibe muchísimas lecturas de los mismos elementos. La aplicación necesita latencia de **microsegundos** en las lecturas con **cambios mínimos** en el código. ¿Qué debe hacer el arquitecto?

- A. Poner Amazon ElastiCache for Redis delante de la tabla y reescribir la lógica de caché.
- B. Cambiar la tabla a capacidad bajo demanda (on-demand).
- C. Agregar DynamoDB Accelerator (DAX).
- D. Crear una tabla global (global table) en otra región.

**18.** Cuando un cliente hace un pedido, **tres sistemas independientes** deben procesarlo: facturación, inventario y envío. Cada sistema debe recibir su propia copia y procesarla a su ritmo, sin perder mensajes si uno está caído. ¿Qué arquitectura cumple los requisitos?

- A. Publicar el pedido en un tema de Amazon SNS con tres colas Amazon SQS suscritas, una por sistema (fan-out).
- B. Una sola cola Amazon SQS de la que leen los tres sistemas.
- C. Que el sistema de pedidos llame directamente a la API de cada sistema.
- D. Guardar el pedido en S3 y que cada sistema revise el bucket cada 10 minutos.

**19.** Una empresa tiene cargas de trabajo en EC2 que corren **24/7** y seguirán así **3 años**. Quiere el mayor ahorro posible, pero en ese tiempo planea **cambiar de familia de instancias** y quizás mover parte de la carga a AWS Fargate o Lambda. ¿Qué opción de compra debe elegir?

- A. Instancias On-Demand
- B. Reserved Instances estándar para la familia actual
- C. Compute Savings Plans
- D. Instancias Spot

**20.** Una aplicación se conecta a Amazon RDS con usuario y contraseña. Seguridad exige guardar las credenciales fuera del código y **rotarlas automáticamente cada 30 días**, con el **menor esfuerzo operativo**. ¿Qué debe hacer el arquitecto?

- A. Guardar las credenciales en un archivo cifrado en S3 y rotarlas con un script manual.
- B. Guardar las credenciales en AWS Secrets Manager con rotación automática activada.
- C. Guardar las credenciales en variables de entorno de la instancia.
- D. Guardar las credenciales en el user data de la instancia.

---

## Hoja de respuestas

| # | Respuesta | S / A |
|---|---|---|
| 1 | | |
| 2 | | |
| 3 | | |
| 4 | | |
| 5 | | |
| 6 | | |
| 7 | | |
| 8 | | |
| 9 | | |
| 10 | | |
| 11 | | |
| 12 | | |
| 13 | | |
| 14 | (dos letras) | |
| 15 | | |
| 16 | | |
| 17 | | |
| 18 | | |
| 19 | | |
| 20 | | |

Tiempo usado: ____ minutos
