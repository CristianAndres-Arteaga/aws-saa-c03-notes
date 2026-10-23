# S3: cifrado (SSE-S3, SSE-KMS, SSE-C, cliente), bucket policies y Block Public Access

**Dominio:** SEC
**Semana:** 02

## Qué es

- **Cifrado en reposo:** los datos se guardan cifrados en el disco de AWS (analogía: el banco guarda tu caja en una caja fuerte). Hay 4 opciones según **quién cifra** y **quién gestiona la clave**.
- **Cifrado en tránsito:** los datos viajan cifrados por la red (HTTPS/TLS), como un sobre sellado en vez de una postal.
- **Bucket policy:** un JSON pegado al bucket que dice quién puede hacer qué en él (analogía: el reglamento pegado en la puerta del banco). La IAM policy, en cambio, es la credencial del empleado.
- **Block Public Access (BPA):** el candado general del edificio. Aunque una policy o una ACL diga "entrada libre", BPA no deja pasar el acceso público.

## Cuándo usarlo

- **SSE-S3** → cifrado sin esfuerzo ni costo extra; AWS gestiona todo. Es el **valor por defecto** de todo bucket desde 2023.
- **SSE-KMS** → cuando piden **auditar** cada uso de la clave (CloudTrail), **controlar quién la usa** (key policy) o **rotarla**.
- **DSSE-KMS** → cuando un cumplimiento exige **doble capa** de cifrado.
- **SSE-C** → la empresa quiere **gestionar sus propias claves fuera de AWS**, pero que S3 haga el cifrado.
- **Client-side** → **AWS nunca debe ver los datos sin cifrar**, ni un instante (ej. hospital con historiales médicos).
- **Bucket policy con Deny + `aws:SecureTransport`** → forzar HTTPS.
- **Bucket policy con Deny + `s3:x-amz-server-side-encryption`** → forzar SSE-KMS en las subidas.
- **BPA a nivel de cuenta** → garantizar que **ningún** bucket, actual o futuro, sea público.
- **CloudFront + OAC** → sitio web estático con HTTPS, baja latencia global y **bucket privado**.
- **Presigned URL** → compartir un objeto privado de forma **temporal** con alguien sin usuario IAM.
- **IAM Access Analyzer for S3** → detectar buckets públicos o compartidos con otras cuentas.

## Cuándo NO usarlo (y qué usar en su lugar)

- **SSE-S3** cuando piden auditar el uso de la clave → usar **SSE-KMS**, porque SSE-S3 no deja un registro en CloudTrail de cada uso de la clave.
- **SSE-C** cuando AWS no debe ver nunca el texto plano → usar **client-side**, porque con SSE-C S3 ve los datos un instante mientras cifra.
- **SSE-C** si la empresa no quiere responsabilizarse de custodiar la clave → usar **SSE-KMS**, porque con SSE-C, si pierdes la clave, pierdes los datos.
- **Cifrado** para "evitar que un bucket sea público" → usar **BPA**, porque cifrar no es controlar el acceso: S3 descifra para cualquiera que tenga permiso de lectura.
- **Allow** para "forzar" algo → usar **Deny + condición negada**, porque un Allow no bloquea a quien ya tiene otro Allow.
- **Desactivar BPA + website endpoint de S3** para un sitio con HTTPS → usar **CloudFront + OAC**, porque el website endpoint solo funciona con HTTP y obliga a hacer público el bucket.
- **IAM user por cliente** u **objetos públicos con nombres raros** para descargas temporales → usar **presigned URLs**.

## Datos clave

**Las 4 opciones de cifrado en reposo**
| | SSE-S3 | SSE-KMS | SSE-C | Client-side |
|---|---|---|---|---|
| ¿Quién cifra? | S3 | S3 | S3 | Tú, antes de subir |
| ¿Quién gestiona la clave? | AWS | KMS (AWS managed o customer managed) | Tú, la envías en cada petición | Tú (o KMS desde tu app) |
| Auditoría en CloudTrail | No | **Sí** | No | — |
| ¿S3 ve el texto plano? | Sí | Sí | Sí, un instante | **Nunca** |
| Cabecera | `AES256` | `aws:kms` | `x-amz-server-side-encryption-customer-*` | — |
| Costo extra | No | Sí (llamadas a KMS) | No | Lo que cueste tu solución |
| Consola | Sí | Sí | **No** (solo CLI/SDK/API) | — |

**SSE-KMS**
- Cada subida = 1 llamada `GenerateDataKey`; cada descarga = 1 llamada `Decrypt` → cuentan contra la **cuota de peticiones de KMS** por región (puede dar throttling con mucho tráfico).
- **S3 Bucket Keys** → S3 reutiliza una clave a nivel de bucket durante un tiempo → **muchas menos llamadas a KMS** → menos costo y menos throttling.
- Para leer un objeto SSE-KMS hace falta `s3:GetObject` **y** `kms:Decrypt`. El permiso de KMS se da en la **key policy** (o en IAM, solo si la key policy habilita IAM para la cuenta).
- Las claves KMS son **regionales**: para usar otra región hay que usar una clave de esa región.

**SSE-C**
- **HTTPS obligatorio** → la clave viaja en las cabeceras de cada petición; por HTTP iría legible y cualquiera que la intercepte podría descifrar tus objetos. S3 rechaza las peticiones SSE-C por HTTP.
- AWS solo guarda un **HMAC con salt** de la clave (una huella dactilar): sirve para comprobar que traes la misma clave, pero no para reconstruirla.
- **Clave perdida = datos perdidos.** AWS Support no puede recuperarla. Prevención: guardar y respaldar la clave tú mismo, o usar SSE-KMS.

**Bucket policies: cómo se evalúan**
1. **Deny explícito siempre gana**, aunque haya un Allow (incluso `s3:*` o `AdministratorAccess`).
2. Sin ningún Allow → **deny implícito** (por defecto todo está prohibido).
3. **Misma cuenta:** basta un Allow en la IAM policy **o** en la bucket policy (OR).
4. **Cross-account:** hace falta un Allow en **ambas**, la bucket policy de la cuenta A **y** la IAM policy de la cuenta B (AND).

**Patrón: forzar HTTPS**
```json
{
  "Sid": "DenyInsecureTransport",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::mi-bucket", "arn:aws:s3:::mi-bucket/*"],
  "Condition": { "Bool": { "aws:SecureTransport": "false" } }
}
```

**Patrón: forzar SSE-KMS en las subidas**
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::mi-bucket/*",
  "Condition": { "StringNotEquals": { "s3:x-amz-server-side-encryption": "aws:kms" } }
}
```
- **Default encryption** del bucket = cifra lo que llega **sin** cabecera de cifrado. **Policy con Deny** = **rechaza** lo que no pide KMS. Si piden "garantizar / rechazar" → policy.

**Block Public Access: las 4 opciones**
| Opción | Qué hace | Afecta a |
|---|---|---|
| `BlockPublicAcls` | Rechaza **nuevas** ACLs públicas | ACLs |
| `IgnorePublicAcls` | **Ignora** las ACLs públicas existentes | ACLs |
| `BlockPublicPolicy` | Rechaza **nuevas** policies públicas | Policies |
| `RestrictPublicBuckets` | Con una policy pública existente, solo deja entrar a principals de AWS autorizados | Policies |
- Se aplica a nivel de **cuenta** (todos los buckets, incluso los futuros) o de **bucket**. **Gana el más restrictivo.**
- Desde 2023, todo bucket nuevo trae **BPA activado** y las **ACLs desactivadas** (Object Ownership = Bucket owner enforced).
- Las **presigned URLs** no se bloquean por BPA, porque van firmadas por un principal autorizado y caducan.

## Se confunde con…

### SSE-C vs client-side
| | SSE-C | Client-side |
|---|---|---|
| Para qué sirve | Claves propias fuera de AWS, cifrado hecho por S3 | Que AWS nunca vea el texto plano |
| Diferencia clave | S3 ve los datos un instante | S3 solo recibe datos ya cifrados |
> Decide si AWS **puede o no** ver los datos sin cifrar, aunque sea un momento.

### SSE-S3 vs SSE-KMS
| | SSE-S3 | SSE-KMS |
|---|---|---|
| Para qué sirve | Cifrado por defecto, sin esfuerzo | Auditoría, control de uso y rotación de la clave |
| Diferencia clave | Sin CloudTrail ni key policy, sin costo | CloudTrail + key policy, cuota y costo de KMS |
> Decide si piden **auditar o controlar** quién usa la clave.

### IAM policy vs bucket policy
| | IAM policy | Bucket policy |
|---|---|---|
| Para qué sirve | Qué puede hacer una identidad en cualquier recurso | Quién puede hacer qué en este bucket |
| Diferencia clave | Identity-based (va en el usuario o rol) | Resource-based (va en el bucket, permite cross-account) |
> En cross-account hacen falta **las dos**.

### Cifrado vs control de acceso
| | Cifrado | Control de acceso (policies + BPA) |
|---|---|---|
| Protege contra | Robo del disco, lectura sin la clave | Que alguien sin permiso pida el objeto |
| ¿Evita que un bucket sea público? | **No** | **Sí** |

## Trampas de examen

**Frases → respuesta**
- "auditar quién usa la clave / cada uso" → SSE-KMS (CloudTrail)
- "throttling de KMS / reducir costo de KMS con muchos objetos" → S3 Bucket Keys
- "AccessDenied leyendo un objeto KMS con s3:GetObject" → falta `kms:Decrypt` en la key policy
- "nuestras propias claves, fuera de AWS, que S3 cifre" → SSE-C
- "AWS nunca debe ver los datos sin cifrar" → client-side
- "forzar HTTPS" → Deny + `aws:SecureTransport: false`
- "rechazar subidas que no usen KMS" → Deny PutObject + `StringNotEquals s3:x-amz-server-side-encryption aws:kms`
- "usuario de otra cuenta recibe AccessDenied con IAM correcta" → falta Allow en la bucket policy de la cuenta dueña
- "ningún bucket, actual o futuro, puede ser público" → BPA a nivel de cuenta
- "sitio estático, HTTPS, global, bucket privado" → CloudFront + OAC
- "acceso temporal a un objeto privado, sin usuarios IAM" → presigned URL
- "detectar buckets públicos o compartidos" → IAM Access Analyzer for S3

**Opciones trampa**
- Cifrar para evitar acceso público → el cifrado no controla el acceso.
- SSE-KMS "para cifrar en tránsito" → SSE-* es en reposo; en tránsito es HTTPS/TLS.
- Allow para "garantizar" → solo un Deny explícito garantiza.
- Más permisos en la IAM de la cuenta B → en cross-account falta la bucket policy de A.
- Desactivar BPA en un bucket cuando la cuenta lo tiene activado → no hace nada, gana el más restrictivo.
- Website endpoint de S3 con HTTPS → solo funciona con HTTP; hace falta CloudFront.
- Pedir a AWS Support una clave SSE-C perdida → AWS solo tiene el HMAC; los datos se pierden.

**Método (lo que aprendí el Día 4-5)**
1. Subrayar las palabras que deciden: **"auditar"**, **"nunca"**, **"garantizar"**, **"futuros"**, **"otra cuenta"**, **"menor esfuerzo"**.
2. Responder con **letra + S/A + un porqué corto** de la opción elegida.
3. Si el porqué solo cubre un requisito, revisar que la opción cumpla **todos**.
4. Leer los descartes de cada opción en la revisión para reconocer las trampas.

## Lo vi en mis proyectos

**loan-collections, Módulo 11: snapshot RDS copiado a otra región**
- La copia en us-west-2 necesitó una clave KMS de esa región, porque las claves KMS son **regionales**.

**loan-collections, Módulo 12a: bucket del frontend**
- Bucket **privado** con **BPA activado**, cifrado **SSE-S3** (por defecto), servido por **CloudFront + OAC**. Es justo la respuesta de la Q2 de escenarios.
- Bucket policy con **DenyInsecureTransport**: el patrón para forzar HTTPS.
