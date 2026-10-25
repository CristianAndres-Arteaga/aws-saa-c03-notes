# S3: Object Lock, Glacier Vault Lock, presigned URLs, Transfer Acceleration y multipart

**Dominio:** SEC | PERF
**Semana:** 02

## Qué es

- **Object Lock:** convierte objetos de S3 en **WORM** (*Write Once, Read Many*): se escriben una vez y no se pueden borrar ni sobrescribir hasta una fecha (analogía: un documento ante notario en una caja sellada; el modo decide quién puede romper el sello).
- **Glacier Vault Lock:** lo mismo, pero para los **vaults** del servicio Glacier original, mediante una política que se vuelve inmutable.
- **Presigned URL:** un enlace temporal, firmado con los permisos de quien lo genera, para descargar o subir un objeto privado sin credenciales AWS.
- **Transfer Acceleration:** subir al edge location más cercano y viajar por la red troncal de AWS hasta el bucket (analogía: entrar a la autopista privada de AWS en el peaje más cercano a tu casa).
- **Multipart upload:** partir un archivo grande en trozos que se suben en paralelo y se reintentan por separado.
- **Byte-range fetch:** un GET que descarga solo una parte del objeto (cabecera `Range`).

## Cuándo usarlo

- **Object Lock Compliance** → regulación estricta (SEC 17a-4, FINRA, salud): **nadie, ni root**, puede borrar durante la retención.
- **Object Lock Governance** → proteger contra borrados accidentales, dejando una vía de escape a **unos pocos admins** con permiso de bypass.
- **Legal Hold** → conservar objetos **sin fecha de fin conocida** (litigio, investigación).
- **Glacier Vault Lock** → WORM sobre **vaults de Glacier**, con 24 h para probar la política antes de que sea permanente.
- **Presigned URL (GET)** → descarga temporal de un objeto privado (facturas, informes).
- **Presigned URL (PUT)** → que los clientes suban **directo a S3** sin pasar por tus servidores y sin credenciales AWS.
- **Transfer Acceleration** → usuarios repartidos por el mundo que suben a **un único bucket** lejano.
- **Multipart** → archivos grandes, redes inestables, subidas en paralelo.
- **Byte-range fetch** → descargas en paralelo más rápidas, o leer solo una parte del archivo.

## Cuándo NO usarlo (y qué usar en su lugar)

- **Versioning + MFA Delete** cuando piden WORM o "ni root" → usar **Object Lock Compliance**, porque el root puede desactivar MFA Delete y subir versiones nuevas encima.
- **Bucket policy o IAM policy con Deny** para inmutabilidad → usar **Object Lock**, porque un admin o el root pueden editar o borrar cualquier política.
- **Governance** cuando piden "nadie, ni root" → usar **Compliance**, porque Governance se puede saltar con `s3:BypassGovernanceRetention`.
- **Alargar la retención con un script** o **copiar a otro bucket** cuando el fin del litigio es desconocido → usar **Legal Hold**.
- **Object Lock** sobre un vault de Glacier → usar **Glacier Vault Lock**; Object Lock es solo para buckets de S3.
- **Vault access policy** para WORM → usar la **política de Vault Lock**, porque la de acceso es editable.
- **Bucket público para escritura** o **un usuario IAM por cliente** → usar **presigned URLs PUT**.
- **CloudFront con caché** para acelerar subidas únicas → usar **Transfer Acceleration**; la caché solo ayuda a descargar contenido repetido.
- **Global Accelerator** delante de S3 → usar **Transfer Acceleration**; Global Accelerator es para apps TCP/UDP e IPs estáticas.
- **Cross-Region Replication** para acelerar subidas → usar **Transfer Acceleration**; CRR copia desde un bucket que ya existe.
- **Lambda que lista objetos** para limpiar multipart a medias → usar **lifecycle AbortIncompleteMultipartUpload**; las partes incompletas ni siquiera aparecen al listar objetos.

## Datos clave

**Object Lock**
- Requiere **Versioning** (se activa solo). El bloqueo se aplica a **cada versión**, no a la key.
- Se puede activar al crear el bucket o en buckets existentes.
- **Retención por defecto** del bucket → se aplica a todos los objetos nuevos.

| Modo | ¿Quién puede borrar o acortar la retención? | ¿Se puede alargar? |
|---|---|---|
| **Governance** | Quien tenga `s3:BypassGovernanceRetention` y envíe la cabecera de bypass | Sí |
| **Compliance** | **Nadie, ni root**. No se puede bajar a Governance | Sí, es lo único que se puede hacer |

| | Retention period | Legal Hold |
|---|---|---|
| Fecha de fin | Sí | **No** |
| Se gestiona con | Modo + fecha | Permiso `s3:PutObjectLegalHold` (on/off) |
| Combinación | El objeto está protegido mientras **cualquiera de los dos** siga activo | |

**Glacier Vault Lock**
1. **Initiate** → la política queda *in-progress* **24 h** para probarla.
2. **Complete** → la política queda **inmutable para siempre**.
- Si no se completa en 24 h, caduca y hay que empezar de nuevo.
- Un vault puede tener a la vez una vault access policy (editable) y una política de Vault Lock (permanente).

**Presigned URLs**
- Llevan los **permisos de quien las firma**: si el firmante pierde el permiso, la URL deja de funcionar.
- Caducidad: **consola hasta 12 h**, **CLI/SDK hasta 7 días**. Si se firmó con credenciales temporales (rol/STS), caduca cuando caducan esas credenciales.
- Funcionan para **GET** y **PUT**. BPA no las bloquea.

**Transfer Acceleration**
- Endpoint: `bucket.s3-accelerate.amazonaws.com`.
- El nombre del bucket debe ser compatible con DNS y **sin puntos**.
- Coste extra, pero **solo se cobra si la transferencia fue más rápida**.

**Multipart upload**
- **Recomendado > 100 MB**, **obligatorio > 5 GB** (un PUT simple admite como máximo 5 GB).
- Objeto máximo **5 TB**, hasta **10.000 partes**.
- Las subidas incompletas **ocupan espacio y se cobran** → regla de lifecycle **AbortIncompleteMultipartUpload**.

## Se confunde con…

### Governance vs Compliance
| | Governance | Compliance |
|---|---|---|
| Para qué sirve | Evitar borrados accidentales | Cumplimiento normativo estricto |
| Diferencia clave | Hay bypass con permiso | Sin bypass, ni root |
> Decide si **alguien** debe poder borrar antes de tiempo.

### Object Lock vs Glacier Vault Lock
| | Object Lock | Glacier Vault Lock |
|---|---|---|
| Para qué sirve | WORM en buckets de S3 (cualquier storage class, incluidas Glacier) | WORM en vaults del servicio Glacier |
| Diferencia clave | Retención por objeto/versión | Política del vault, 24 h de prueba y luego inmutable |
> Decide si el enunciado habla de **bucket** o de **vault**.

### Transfer Acceleration vs CloudFront vs Global Accelerator
| | Transfer Acceleration | CloudFront | Global Accelerator |
|---|---|---|---|
| Para qué sirve | Acelerar subidas/descargas a un bucket lejano | Caché de contenido para usuarios globales | Apps TCP/UDP, IPs estáticas, failover |
| Diferencia clave | Sin caché, red troncal hasta S3 | Caché en el edge | No va delante de S3 |
> Decide si el contenido es **único** (subidas → TA), **repetido** (descargas → CloudFront) o **no HTTP** (GA).

## Trampas de examen

**Frases → respuesta**
- "WORM", "regulación", "nadie, ni siquiera root" → Object Lock **Compliance**
- "solo ciertos admins podrán borrar antes de tiempo" → Object Lock **Governance**
- "conservar hasta que termine el juicio" / "indefinidamente" → **Legal Hold**
- "vault de Glacier", "probar la política antes de que sea permanente" → **Glacier Vault Lock**
- "acceso temporal a un objeto privado, sin usuarios IAM" → **presigned URL GET**
- "los clientes suben directo a S3, sin credenciales, sin sobrecargar las EC2" → **presigned URL PUT**
- "usuarios en varios continentes suben a un bucket lento" → **Transfer Acceleration**
- "archivos grandes", "fallan a mitad", "subidas en paralelo" → **multipart**
- "archivo > 5 GB" → multipart **obligatorio**
- "la factura sube pero los objetos visibles no cambian" → **AbortIncompleteMultipartUpload**
- "descargar más rápido en paralelo" / "leer solo la cabecera" → **byte-range fetch**

**Opciones trampa**
- MFA Delete como WORM → el root lo desactiva.
- Bucket policy o IAM policy "para que nadie borre" → son editables.
- Cambiar de Compliance a Governance → no se puede.
- Object Lock en un vault → Object Lock es solo para buckets.
- CloudFront con caché para subidas únicas → no hay nada que cachear.
- Global Accelerator delante de S3 → no aplica.
- Presigned URL de 30 días desde la CLI → máximo 7 días.

**Método (Día 6)**
1. Buscar **quién** puede borrar ("ni root", "solo admins", "nadie hasta el juicio").
2. Si hay **dos problemas** en el enunciado ("lento" y "falla"), elegir la opción que resuelva los dos.
3. "Menor esfuerzo operativo" descarta scripts y Lambdas cuando hay una función nativa (Legal Hold, lifecycle).

## Lo vi en mis proyectos

- Ninguno lo usó todavía. Pendiente si un proyecto futuro guarda backups regulados o subidas de usuarios.
