# AWS Certified Solutions Architect – Associate (SAA-C03): apuntes de estudio

Apuntes propios para preparar el examen **SAA-C03**, escritos semana a semana entre octubre de 2026 y enero de 2027.

Parten de dos proyectos construidos en AWS:

- [loan-collections-platform-aws](https://github.com/CristianAndres-Arteaga/loan-collections-platform-aws): three-tier con VPC, EC2 Auto Scaling, ALB, RDS, ElastiCache y CloudFront, con resiliencia medida.
- [task-management-system](https://github.com/CristianAndres-Arteaga/task-management-system): serverless con Lambda, API Gateway, DynamoDB y Cognito.

## El examen

- 65 preguntas en 130 minutos; se aprueba con 720/1000.
- Dominios: **Secure 30%** · **Resilient 26%** · **High-Performing 24%** · **Cost-Optimized 20%**.
- Fecha objetivo: semana del **18 de enero de 2027**.

## Plan

A mi ritmo: cada semana, teoría y apuntes del tema y, al cerrarla, preguntas de práctica.

| Semana | Tema | Estado |
|---|---|---|
| 01 | [Diagnóstico y guía del examen](semana-01-diagnostico/) | ⏳ |
| 02 | [S3 a fondo + EBS](semana-02-s3-ebs/) | ⏳ |
| 03 | [EFS, FSx, Storage Gateway, DataSync, Snow](semana-03-almacenamiento-hibrido/) | ⏳ |
| 04 | [SQS, SNS, EventBridge, Step Functions, Kinesis](semana-04-desacoplamiento/) | ⏳ |
| 05 | [Aurora, DynamoDB, Redshift, bases especializadas](semana-05-bases-de-datos/) | ⏳ |
| 06 | [Route 53 y redes avanzadas](semana-06-redes/) | ⏳ |
| 07 | [Organizations, IAM avanzado, seguridad](semana-07-seguridad/) | ⏳ |
| 08 | [Compras de EC2, contenedores, serverless](semana-08-computo/) | ⏳ |
| 09 | [Analítica, migración, backup, gobierno](semana-09-analitica-migracion/) | ⏳ |
| 10–12 | [Exámenes de práctica completos](examenes-practica.md) | ⏳ |

## Cómo están organizados

- Una carpeta por semana, con la lista de temas y el registro de las preguntas del viernes.
- Un archivo por servicio o comparación, con la plantilla [`TEMPLATE.md`](TEMPLATE.md): qué es, cuándo usarlo y cuándo no, con qué se confunde y las trampas de examen.
- Los puntajes de los exámenes de práctica, en [`examenes-practica.md`](examenes-practica.md).
