# EFS (Elastic File System) vs EBS vs S3

**Dominio:** RES | PERF | COST
**Semana:** 03

## Qué es

- **EFS:** un sistema de archivos **NFS** administrado que muchas instancias **Linux**, en **varias AZs**, montan a la vez y en el que leen y escriben **los mismos archivos** (analogía: la **biblioteca compartida** del edificio). Crece y se reduce sola, y pagas por los GB que usas.
- Comparado con los otros: **EBS** es el cuaderno personal de un salón (1 instancia, 1 AZ) y **S3** es un depósito de cajas (objetos por HTTP, no es un sistema de archivos).

## Cuándo usarlo

- Varias EC2 (por ejemplo un **ASG Multi-AZ**) que comparten archivos: contenido web, subidas de usuarios, CMS (WordPress).
- **Home directories** compartidos de Linux.
- Almacenamiento compartido para **ECS/EKS/Lambda**.
- Servidores **on-premises Linux** que montan el mismo sistema de archivos (vía Direct Connect o VPN).
- Necesitas que la capacidad **crezca sola**, sin reservar tamaño.

## Cuándo NO usarlo (y qué usar en su lugar)

- Servidores **Windows** (SMB, Active Directory) → **FSx for Windows File Server**.
- HPC / machine learning con altísimo rendimiento → **FSx for Lustre**.
- Una sola instancia con una base de datos o un volumen de arranque → **EBS** (más barato y menor latencia).
- Objetos accedidos por HTTP/API, data lake, sitios estáticos → **S3**.
- **EBS Multi-Attach** para compartir entre AZs → no se puede (solo en 1 AZ) → **EFS**.
- **Instance store** para compartir archivos → cada instancia tiene el suyo y es efímero → **EFS**.
- **Script en cron** para mover archivos fríos → **lifecycle policy** de EFS.
- **Archivos de relleno** para subir el caudal en Bursting → **Provisioned** o **Elastic** throughput.

## Datos clave

**Básicos**
- Protocolo **NFS v4.1**, compatible con **POSIX** → **solo Linux**.
- **Mount target** en cada AZ, con su **Security Group** → permitir el puerto **2049 (NFS)** desde el SG de las EC2.
- Escala hasta **miles de clientes** simultáneos y petabytes.
- ~**3 veces más caro por GB que gp2**, pero se paga solo lo usado.

**Clases de almacenamiento**

| Clase | Uso | Clave |
|---|---|---|
| **Standard** | Acceso frecuente | Multi-AZ, la más cara |
| **Infrequent Access (IA)** | Acceso ocasional | Más barata por GB, **cobra por cada lectura** |
| **Archive** | Casi nunca se accede | La más barata |
| **One Zone / One Zone-IA** | Dev/test, datos recreables | **1 sola AZ**, más barata, menos disponibilidad |

- **Lifecycle policy:** mueve los archivos no leídos en *N* días (por ejemplo 30) a IA y después a Archive. La **ruta no cambia** y la app no se entera. Opcionalmente, devuelve a Standard un archivo cuando se vuelve a leer.

**Performance mode** (se elige al crear)
- **General Purpose** (por defecto): baja latencia. Web, CMS, home directories.
- **Max I/O:** más paralelismo, **más latencia**. Big data con miles de clientes (hoy casi legacy).

**Throughput mode**
- **Bursting:** el caudal **depende del tamaño**. Con un sistema pequeño se acaban los créditos.
- **Provisioned:** caudal **fijo sin importar el tamaño**. Para cargas **constantes y predecibles**.
- **Elastic:** se ajusta solo y pagas por uso. Para cargas **impredecibles o con picos**.

**Seguridad**
- Cifrado en reposo con KMS **solo al crear** el sistema de archivos. Para cifrar uno existente: crear uno nuevo cifrado y copiar los datos (por ejemplo con **DataSync**).
- Cifrado en tránsito con **TLS** usando el mount helper (`amazon-efs-utils`).
- Control de acceso con Security Groups (red), políticas IAM / file system policy y **Access Points** (ruta y usuario POSIX por aplicación).

## Se confunde con…

### EBS vs EFS vs S3
| | EBS | EFS | S3 |
|---|---|---|---|
| Tipo | Bloques | Archivos (NFS) | Objetos |
| Acceso | 1 instancia | Miles, varias AZs | HTTP/API |
| Alcance | 1 AZ | Regional (o One Zone) | Regional |
| Capacidad | Reservada | Automática | Ilimitada |
> Pregúntate: **¿cuántas instancias** y **en cuántas AZs** usan los mismos archivos?

### Performance mode vs throughput mode
| | Performance mode | Throughput mode |
|---|---|---|
| Qué controla | Latencia vs paralelismo | MB/s |
| Opciones | General Purpose / Max I/O | Bursting / Provisioned / Elastic |
> Si el problema es "**se queda sin créditos**" o "**MB/s**" → cambia el **throughput mode**.

### Provisioned vs Elastic
| | Provisioned | Elastic |
|---|---|---|
| Carga | Constante y predecible | Impredecible, con picos |
| Pagas por | El caudal que fijas | El caudal que usas |

### EFS vs FSx for Windows
| | EFS | FSx for Windows |
|---|---|---|
| Protocolo | NFS | SMB |
| Sistema operativo | Linux | Windows (+ Active Directory) |
> Si el enunciado dice **Windows**, **SMB** o **Active Directory** → no es EFS.

## Trampas de examen

**Frases → respuesta**
- "varias instancias Linux en varias AZs comparten archivos" → **EFS**
- "crece automáticamente", "sin gestionar capacidad" → **EFS**
- "archivos que no se abren después de 30 días", "menor esfuerzo" → **lifecycle policy a IA**
- "dev/test", "datos que se pueden recrear", "más barato" → **EFS One Zone**
- "sistema de archivos pequeño pero necesita mucho caudal constante" → **Provisioned throughput**
- "carga impredecible", "picos" → **Elastic throughput**
- "las EC2 no pueden montar el EFS" → revisar el **SG del mount target** (puerto **2049**)
- "cifrar un EFS existente" → **nuevo EFS cifrado + copiar los datos**

**Opciones trampa**
- EFS para servidores Windows → es NFS, solo Linux.
- EBS Multi-Attach entre AZs → solo en 1 AZ.
- S3 "montado como disco" para una app que escribe archivos → no es POSIX.
- Max I/O para tener más MB/s → eso es throughput mode, no performance mode.
- One Zone para ahorrar cuando piden alta disponibilidad → pierdes Multi-AZ.
- Activar el cifrado en un EFS existente → solo al crear.

**Método (Día 8)**
1. ¿**Windows**? → FSx. ¿**Linux compartido**? → EFS. ¿**Una instancia**? → EBS. ¿**HTTP/objetos**? → S3.
2. Si el problema es de **costo** con archivos fríos → lifecycle. Si es de **caudal** → throughput mode.
3. "Menor esfuerzo operativo" → la función nativa, nunca el script.

## Lo vi en mis proyectos

- Ninguno lo usó todavía. En la plataforma three-tier las EC2 no compartían archivos: los datos iban a RDS y la caché a ElastiCache. Si la app hubiera guardado subidas de usuarios en disco, EFS habría sido la opción para compartirlas entre las instancias del ASG (o mejor aún, S3 con presigned URLs).
