# DataSync, Transfer Family y familia Snow

**Dominio:** RES | PERF | COST
**Semana:** 03

## Qué es

- **DataSync:** el **camión de mudanza programable**. Mueve datos de A a B por la red, una vez o con una frecuencia programada.
- **Transfer Family:** el **buzón para tus socios**. Es un servidor SFTP/FTPS/FTP/AS2 administrado que guarda en S3 o EFS lo que suben terceros.
- **Familia Snow:** cuando la red no alcanza, **AWS te envía un dispositivo físico**. Lo llenas, lo devuelves por courier y los datos llegan a S3. También sirve para procesar datos en sitios sin conectividad (**edge computing**).

## Cuándo usarlo

- **DataSync:** migrar un NAS (NFS/SMB) o HDFS a S3, EFS o FSx; sincronizar cada noche; copiar entre servicios de AWS (S3 → EFS, EFS → EFS en otra región).
- **Transfer Family:** socios externos (bancos, proveedores) que ya envían archivos por SFTP/FTP y **no van a cambiar sus clientes**.
- **Snowball Edge Storage Optimized:** migraciones de decenas a cientos de TB cuando la red tardaría más de una semana.
- **Snowball Edge Compute Optimized:** procesar o filtrar datos en sitios **sin red** (barcos, minas, plataformas petroleras, zonas militares).

## Cuándo NO usarlo (y qué usar en su lugar)

- **DataSync** para que los servidores on-prem accedan siempre a los datos con caché local → **Storage Gateway**.
- **Storage Gateway** para una migración de una sola vez → **DataSync**.
- **SFTP en EC2** cuando piden "sin administrar servidores" → **Transfer Family**.
- **DataSync / S3 Transfer Acceleration** con un enlace lento y mucho volumen → **Snowball Edge** (la red sigue siendo el cuello de botella).
- **Direct Connect** para "migrar en 2 semanas" → instalarlo tarda semanas o meses → **Snowball Edge**.
- **Snowball** "directo a Glacier" → no existe: importa a **S3** y luego una **lifecycle policy** lo mueve a Glacier.
- **Storage Optimized** cuando piden procesar en sitio → **Compute Optimized**.

## Datos clave

**DataSync**
- **Agente** (VM) on-prem cerca de los datos. Lee **NFS, SMB, HDFS** o almacenamiento compatible con S3.
- Destinos: **S3** (cualquier clase, incluida Glacier), **EFS**, **FSx** (los cuatro).
- **AWS ↔ AWS sin agente.**
- Tareas **programadas**, **verificación de integridad**, **conserva metadatos** (permisos, timestamps, ACLs), **límite de ancho de banda**.
- Cifrado en tránsito con TLS; por internet o Direct Connect. Mucho más rápido que `cp`/`rsync` porque paraleliza.

**Transfer Family**
- Protocolos: **SFTP, FTPS, FTP, AS2**. Destino: **S3** o **EFS**.
- Autenticación: usuarios del servicio (claves SSH), Active Directory o un proveedor de identidad propio (Lambda).
- Costo: **por hora de endpoint activo + GB transferidos**.

**Familia Snow**
- Cálculo mental: **100 Mbps ≈ 1 TB/día** · **1 Gbps ≈ 10 TB/día**. Si por red tarda **más de una semana** → Snow.

| | Storage Optimized | Compute Optimized |
|---|---|---|
| Para qué | **Migrar** muchos TB | **Procesar en sitio** (edge) |
| Cómputo | Básico | Más vCPU, GPU opcional |
| Ejemplo | 200 TB con un enlace de 100 Mbps | Barco sin red que filtra datos con ML |

- Corre **EC2** (desde tus AMIs) y **Lambda** en el dispositivo.
- Cifrado obligatorio con **KMS**; resistente a manipulación; AWS lo borra al terminar.
- Se administra con **AWS OpsHub** (aplicación de escritorio).
- **Actualidad:** Snowcone y Snowmobile fueron retirados y la familia Snow ya no está disponible para clientes nuevos, pero el SAA-C03 todavía pregunta por **Snowball Edge**.

## Se confunde con…

### Árbol de decisión: todo lo híbrido (semana 03)
```
¿Qué necesitas?
├─ Acceso CONTINUO desde on-prem, con caché local ... Storage Gateway
│     (archivos → File GW · bloques → Volume GW · cintas → Tape GW)
├─ MOVER datos por la red (una vez o programado) .... DataSync
├─ Socios externos con SFTP/FTP/AS2 ................. Transfer Family
├─ Red demasiado lenta (> 1 semana) o sin red ....... Snowball Edge
│     (migrar → Storage Optimized · procesar en sitio → Compute Optimized)
└─ Subir a S3 rápido desde lejos (usuarios globales)  S3 Transfer Acceleration
```

### Storage Gateway vs DataSync vs Transfer Family
| | Storage Gateway | DataSync | Transfer Family |
|---|---|---|---|
| Para qué | Acceso **continuo** on-prem ↔ AWS | **Mover / migrar** datos | **Recibir o enviar** archivos de terceros |
| Quién lo usa | Tus servidores | Tú (migración / sync) | Socios externos |
| Protocolo | NFS, SMB, iSCSI, VTL | NFS, SMB, HDFS, S3 | SFTP, FTPS, FTP, AS2 |
| Caché local | Sí | No | No |

## Trampas de examen

**Frases → respuesta**
- "migrar NAS", "NFS/SMB a EFS/S3/FSx", "sincronizar cada noche", "verificar integridad" → **DataSync**
- "socios externos", "SFTP", "sin cambiar sus clientes", "sin administrar servidores" → **Transfer Family**
- "cientos de TB", "enlace lento", "plazo de semanas" → **Snowball Edge Storage Optimized**
- "sin conectividad", "procesar / filtrar en sitio", "edge" → **Snowball Edge Compute Optimized**

**Opciones trampa**
- Direct Connect para una migración urgente → tarda semanas o meses en instalarse.
- S3 Transfer Acceleration con poco ancho de banda → no aumenta tu enlace.
- Snowball directo a Glacier → llega a S3 y luego lifecycle.
- Storage Gateway o DataSync en un sitio sin red → ambos necesitan conexión.

**Método (Día 9)**
1. ¿**Continuo, mudanza, terceros u offline**? → te da el servicio.
2. Si es mudanza: **haz el cálculo** (TB ÷ TB/día) y compáralo con el plazo. "Grande" no basta: 20 TB con 10 Gbps es menos de un día → DataSync.

## Lo vi en mis proyectos

- Ninguno lo usó todavía. Las dos apps nacieron en AWS, sin datos on-premises que migrar ni socios externos enviando archivos.
