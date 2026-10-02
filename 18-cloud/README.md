# Cloud

## Conceptos previos

- Servidor: una computadora que ofrece un servicio (una web, una base de datos) a otras por la red.
- Centro de datos (data center): edificio con miles de servidores, energía redundante, refrigeración y seguridad física.
- Máquina virtual (VM): computadora hecha de software que corre sobre un hypervisor; la nube entera está construida sobre esto (ver [`06-virtualizacion`](../06-virtualizacion/)).
- API: interfaz por la que un programa le pide cosas a otro; en la nube todo (crear un servidor, borrar un disco) es una llamada a una API.
- IAM (Identity and Access Management): el sistema que decide quién puede hacer qué sobre qué recurso (ver [`08-autenticacion`](../08-autenticacion/)).
- Política (policy): documento, normalmente JSON, que dice qué acciones están permitidas o denegadas a una identidad o sobre un recurso.
- CapEx / OpEx: gasto de capital (comprar equipos de una vez) frente a gasto operativo (pagar mes a mes por uso).
- IaaS / PaaS / SaaS: niveles de servicio en la nube; en IaaS alquilas máquinas virtuales, en PaaS una plataforma donde subes tu código, en SaaS usas una aplicación terminada. Solo importan aquí porque mueven la línea de responsabilidad compartida.
- Región y zona de disponibilidad (AZ): una región es un área geográfica del proveedor (por ejemplo `us-east-1`, Virginia); cada región tiene varias AZ, que son centros de datos separados para que un incendio en uno no tumbe todo.
- Misconfiguration: un error de configuración (un permiso de más, un recurso público por descuido); es la causa número uno de incidentes en la nube.

## Understand the Concept of Security in the Cloud

La **seguridad en la nube** es el conjunto de controles, procesos y configuraciones que protegen los datos, las aplicaciones y la infraestructura que una organización tiene en un proveedor cloud. Existe como disciplina propia porque las reglas cambian: ya no hay un perímetro físico ni un cable que desconectar, todo se controla por API e identidad, y el cliente no puede tocar el hardware. Lo que sí sigue igual son los objetivos de la tríada CIA (ver [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/)).

El concepto central es el **modelo de responsabilidad compartida**: un acuerdo que reparte la seguridad entre el proveedor y el cliente según qué capa controla cada uno. AWS lo resume en dos frases: el proveedor es responsable de la seguridad *de* la nube (security of the cloud: edificios, hardware, red física, hypervisor) y el cliente de la seguridad *en* la nube (security in the cloud: sus datos, sus identidades, su configuración, su sistema operativo si alquila VMs).

Analogía: alquilar un apartamento. El dueño del edificio responde por los cimientos, el ascensor, la cerradura de la entrada principal y el guardia de la puerta. Tú respondes por cerrar tu puerta con llave, a quién le das copia de la llave y qué dejas encima de la mesa. Si te roban porque dejaste la puerta abierta, no es culpa del edificio.

La línea se mueve según el nivel de servicio:

```
                        On-premises   IaaS (EC2)    PaaS (RDS,    SaaS (correo
                                                    App Engine)   en la nube)
Datos y quién accede    cliente       cliente       cliente       cliente
Identidades y permisos  cliente       cliente       cliente       cliente
Aplicación              cliente       cliente       cliente       proveedor
Sistema operativo       cliente       cliente       proveedor     proveedor
Hypervisor / red física cliente       proveedor     proveedor     proveedor
Edificio y hardware     cliente       proveedor     proveedor     proveedor
```

Las dos primeras filas son del cliente en todas las columnas: nunca se delegan.

Ejemplo concreto: una empresa usa una base de datos gestionada (RDS). AWS parchea el motor de la base de datos y el sistema operativo debajo; pero si la empresa deja el puerto 5432 abierto a `0.0.0.0/0` (todo internet) en el grupo de seguridad, con la contraseña `admin123`, la brecha es suya.

### La nube no te quita responsabilidad, te la cambia de sitio

> [!IMPORTANT]
> El proveedor casi nunca es la causa de una brecha en la nube: la causa típica es una configuración del cliente (un bucket público, una clave filtrada, un permiso `*`). Lo que el cliente siempre conserva son sus datos y sus identidades.

La responsabilidad compartida es un reparto de tareas, no una transferencia del riesgo: subir algo a la nube no lo vuelve seguro, solo cambia qué partes cuidas tú. Gartner proyectó que hasta 2025 el 99 % de los fallos de seguridad en la nube serían culpa del cliente. El caso clásico es Capital One (2019): una aplicación web con un firewall mal configurado permitió un ataque SSRF (hacer que el servidor haga peticiones en nombre del atacante) contra el servicio de metadatos de la VM, que entregó credenciales temporales de un rol con permisos de lectura sobre buckets S3; se filtraron datos de más de 100 millones de personas. Ninguna pieza de AWS falló: falló la configuración del cliente.

```
Capa del cliente:   WAF mal configurado -> SSRF -> 169.254.169.254 (metadatos, IMDSv1)
                    -> credenciales del rol -> rol con permiso s3:* demasiado amplio
                    -> descarga de buckets
Capa del proveedor: hypervisor, red, hardware -> nada comprometido
```

La defensa estándar hoy es exigir IMDSv2 (los metadatos solo responden a quien primero pidió un token con un `PUT`, cosa que un SSRF simple no puede hacer) y dar a cada rol solo los permisos mínimos.

```
Seguridad DE la nube → proveedor: edificios, hardware, red física, hypervisor.
Seguridad EN la nube → cliente: datos, identidades, permisos, configuración, SO en IaaS.
Datos e identidades → siempre del cliente, en IaaS, PaaS y SaaS.
```

## Understand the differences between cloud and on-premises

**On-premises** (on-prem) es un modelo de infraestructura en el que la organización compra, instala y opera sus propios servidores en sus instalaciones o en un centro de datos que controla; la **nube** es el modelo en el que alquila esos recursos a un proveedor y los obtiene por API, pagando por uso. La diferencia existe porque cambian quién compra el hardware, quién lo opera, cuánto tarda conseguir capacidad y dónde está la frontera de seguridad.

Analogía: tener carro propio frente a usar taxis o carros de alquiler. Con el carro propio pagas todo de golpe, haces tú el mantenimiento y lo tienes siempre, aunque esté parado. Con el alquiler pagas por kilómetro, alguien más lo mantiene, y si mañana necesitas diez carros los tienes en una hora; pero si te olvidas el contrato abierto, sigues pagando.

Diferencias que importan:

- Costo: on-prem es CapEx (comprar 10 servidores a 8.000 dólares cada uno = 80.000 de entrada, se amortizan en 3-5 años); cloud es OpEx (pagar por hora de uso; una VM pequeña cuesta centavos la hora).
- Elasticidad: on-prem tarda semanas en comprar, recibir e instalar un servidor; en la nube una VM nueva está lista en un minuto y se apaga cuando no hace falta.
- Control físico: on-prem controlas la puerta, los discos y puedes llevarte un disco para forense; en la nube no tocas el hardware y la evidencia sale de logs (CloudTrail, Azure Activity Log, Cloud Audit Logs) y snapshots.
- Perímetro: on-prem la frontera es la red (firewall en la entrada, ver [`14-defensa-y-hardening`](../14-defensa-y-hardening/)); en la nube la frontera real es la identidad: quien tiene una clave de acceso válida puede llegar al recurso desde cualquier lugar del mundo.
- Velocidad de error: on-prem, exponer un servidor exige tocar varios equipos; en la nube, un clic o una línea de código lo publica a internet en segundos.
- Cumplimiento y residencia de datos: on-prem sabes en qué edificio están los datos; en la nube eliges región (por ejemplo, una región europea para datos sujetos a GDPR, ver [`17-estandares-y-cumplimiento`](../17-estandares-y-cumplimiento/)).

Ejemplo: una tienda en línea que vende 10 veces más en Black Friday. On-prem tendría que comprar servidores para el pico y dejarlos ociosos 11 meses. En la nube escala de 4 a 40 instancias ese fin de semana y vuelve a 4 el lunes.

> [!WARNING]
> En la nube la identidad es el nuevo perímetro: una clave de acceso subida por error a un repositorio público de GitHub suele ser encontrada y usada por bots en minutos, sin pasar por ningún firewall.

```
On-premises → hardware propio, CapEx, control físico, perímetro de red.
Cloud → recursos alquilados por API, OpEx, elasticidad, perímetro de identidad.
Evidencia on-prem → discos y equipos físicos.
Evidencia cloud → logs de API (CloudTrail, Activity Log, Audit Logs) y snapshots.
```

## Understand the concept of Infrastructure as Code

**Infrastructure as Code** (IaC) es una práctica que define servidores, redes, bases de datos y permisos en archivos de texto que una herramienta lee para crear o modificar esa infraestructura automáticamente, en lugar de configurarla a mano en una consola web. Resuelve tres problemas de la configuración manual: no se puede repetir igual (dos entornos "iguales" acaban distintos), no deja rastro de quién cambió qué, y no se puede revisar antes de aplicar.

Analogía: una receta de cocina frente a cocinar "a ojo". Con la receta, cualquiera produce el mismo pastel, puedes ver qué cambió entre la versión de la abuela y la tuya, y si algo sale mal vuelves a la versión anterior.

Cómo funciona por dentro. Hay dos estilos:

- Declarativo: describes el estado final ("quiero un bucket privado y dos VMs") y la herramienta calcula qué crear, cambiar o borrar. Terraform/OpenTofu (multi-nube), AWS CloudFormation, Azure ARM/Bicep.
- Imperativo/configuración: describes pasos a ejecutar sobre máquinas existentes. Ansible, scripts.

Terraform guarda un archivo de estado (`terraform.tfstate`) con lo que cree que existe; en cada `plan` compara código, estado y realidad, y muestra la diferencia antes de aplicarla.

```
  main.tf (código en git) --> terraform plan --> diff (+ crear, ~ cambiar, - borrar)
                                                    |
                                     revisión humana / escáner de seguridad
                                                    |
                                             terraform apply --> API del proveedor
                                                    |
                                            terraform.tfstate actualizado
```

Ejemplo concreto: un bucket S3 privado con cifrado, y la salida del plan:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "acme-logs-2026"
}

resource "aws_s3_bucket_public_access_block" "logs" {
  bucket                  = aws_s3_bucket.logs.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```
$ terraform plan
  + resource "aws_s3_bucket" "logs" { + bucket = "acme-logs-2026" ... }
  + resource "aws_s3_bucket_public_access_block" "logs" { ... }
Plan: 2 to add, 0 to change, 0 to destroy.
```

Por qué importa en seguridad: como la infraestructura es texto, se puede escanear antes de que exista. Herramientas como Checkov o tfsec revisan el código y marcan un bucket sin bloqueo público o un grupo de seguridad con `0.0.0.0/0` en el puerto 22 antes del `apply`. El riesgo inverso: el archivo de estado guarda en claro valores sensibles (contraseñas de bases de datos), así que debe vivir en un backend remoto cifrado y con acceso restringido, nunca en el repositorio.

> [!TIP]
> Regla práctica: si un cambio de infraestructura no pasó por código, revisión y `plan`, es una deriva (drift) que nadie va a recordar; los escáneres de IaC encuentran los errores donde es más barato corregirlos, antes de desplegar.

```
IaC → infraestructura definida en archivos de texto versionados y aplicada por herramienta.
Declarativo (Terraform, CloudFormation, Bicep) → describes el estado final.
Imperativo (Ansible, scripts) → describes los pasos.
terraform plan / apply → mostrar la diferencia / aplicarla.
tfstate → estado conocido; puede contener secretos; backend remoto cifrado.
Drift → diferencia entre el código y lo que realmente existe.
```

## Understand the Concept of Serverless

**Serverless** es un modelo de ejecución en la nube en el que el desarrollador sube solo funciones o contenedores y el proveedor se encarga de los servidores, el sistema operativo, el escalado y el apagado, cobrando únicamente por el tiempo real de ejecución. Los servidores existen, pero el cliente no los ve ni los administra. Resuelve el desperdicio de pagar una VM encendida 24 horas para un código que se ejecuta 3 minutos al día.

Cómo funciona por dentro. Una función (AWS Lambda, Azure Functions, Google Cloud Run functions) se dispara por un evento: una petición HTTP, un archivo nuevo en un bucket, un mensaje en una cola, una hora del día. El proveedor arranca un entorno aislado (en Lambda, una microVM Firecracker), ejecuta la función y lo reutiliza o lo destruye. La primera ejecución tras un rato sin uso tarda más porque hay que crear el entorno ("cold start"). Límites típicos de Lambda: hasta 15 minutos por ejecución y entre 128 MB y 10.240 MB de memoria.

```
  Evento (subida a S3, petición HTTP, cola)
        |
        v
  Proveedor: ¿hay un entorno caliente? --no--> crea microVM (cold start)
        |                                           |
        +-------------------sí----------------------+
        v
  Ejecuta la función con el ROL IAM asignado --> lee/escribe otros servicios
        |
        v
  Cobra por milisegundos de ejecución; entorno se recicla o destruye
```

Analogía: un taxi frente a un carro propio. No compras, no aseguras ni estacionas nada; pagas solo el trayecto. Pero no eliges el motor, y si hace mucho que nadie lo pidió, tarda un poco más en llegar.

Ejemplo concreto: una función que genera miniaturas cuando alguien sube una foto a un bucket. Con 10.000 fotos al mes de 2 segundos cada una, son unos 20.000 segundos de cómputo; una VM equivalente estaría encendida 2,6 millones de segundos al mes.

Seguridad en serverless: desaparece el parcheo del SO (lo hace el proveedor), pero la superficie se mueve a tres sitios: el código de la función (inyección a través de los datos del evento, por ejemplo el nombre de un archivo subido), las dependencias que empaqueta, y sobre todo el rol IAM de la función. Una función con permiso `s3:*` sobre `*` que sufre una inyección le da al atacante todos los buckets de la cuenta.

```
Serverless → el proveedor gestiona servidores y escalado; pagas por ejecución.
Función (Lambda, Azure Functions, Cloud Run functions) → código disparado por eventos.
Cold start → retraso de la primera ejecución al crear el entorno.
Rol de la función → define el daño máximo si la función es comprometida.
```

## Understand the basics and general flow of deploying in the cloud

**Desplegar en la nube** es el proceso de llevar el código de una aplicación desde el repositorio hasta un entorno de producción en un proveedor cloud, normalmente de forma automática mediante un pipeline de CI/CD. CI (integración continua) es la parte que compila y prueba cada cambio; CD (entrega o despliegue continuo) es la parte que lo publica. Existe para que los despliegues sean repetibles, revisables y rápidos, en lugar de alguien copiando archivos a mano a un servidor un viernes por la noche.

Analogía: una línea de montaje de fábrica. Cada pieza pasa por estaciones fijas (ensamblado, inspección, pintura, control de calidad) y solo lo que pasa todas sale por la puerta.

Flujo general:

```
  1. Desarrollador hace push a git (rama + pull request revisado)
        |
  2. CI: compilar -> pruebas -> escaneo (SAST, dependencias, secretos, IaC)
        |
  3. Artefacto: imagen de contenedor o paquete, firmado -> registro (ECR, ACR, Artifact Registry)
        |
  4. IaC: terraform plan/apply crea o ajusta red, VMs, funciones, permisos
        |
  5. Despliegue: staging -> pruebas -> producción (blue/green o canary)
        |
  6. Operación: logs, métricas, alertas, auditoría de API (CloudTrail)
```

Términos del flujo:

- Entornos: dev, staging y producción separados, idealmente en cuentas o suscripciones distintas para que un error en dev no toque producción.
- Blue/green: se levanta la versión nueva (green) al lado de la vieja (blue) y se cambia el tráfico de golpe; si falla, se vuelve a blue.
- Canary: se manda primero un 5-10 % del tráfico a la versión nueva y se sube si no hay errores.
- Rollback: volver a la versión anterior.

Seguridad del pipeline: el pipeline tiene permisos para cambiar producción, así que es un objetivo de alto valor (ataques a la cadena de suministro). Buenas prácticas: credenciales de corta duración mediante federación OIDC entre el sistema de CI y la nube en lugar de claves de acceso fijas guardadas como secreto, secretos en un gestor (AWS Secrets Manager, Azure Key Vault, Google Secret Manager) y nunca en el código, y revisión obligatoria antes de producción.

Ejemplo concreto: un equipo hace 20 despliegues por semana. Con canary al 10 %, una versión que rompe el pago afecta a 1 de cada 10 clientes durante 5 minutos antes del rollback automático, en lugar de a todos durante una hora.

```
CI → compilar, probar y escanear cada cambio.
CD → publicar automáticamente lo que pasó CI.
Artefacto → la imagen o paquete exacto que se despliega.
Blue/green → dos versiones lado a lado; cambio de tráfico de golpe.
Canary → un porcentaje pequeño de tráfico primero.
OIDC en CI → credenciales temporales en lugar de claves fijas.
```

## Cloud Models

Un **modelo de despliegue cloud** es la forma en que se organiza quién es dueño de la infraestructura y quién puede usarla. NIST SP 800-145 define cuatro (privada, comunitaria, pública e híbrida); el roadmap pide las tres principales. La comunitaria es una nube compartida por organizaciones con requisitos comunes (por ejemplo, varias agencias de gobierno).

### Private

Una **nube privada** es una infraestructura cloud usada exclusivamente por una sola organización, operada por ella misma o por un tercero, en sus instalaciones o en un centro de datos dedicado. Ofrece la experiencia de nube (autoservicio, VMs por API, escalado dentro de lo comprado) sin compartir hardware con otros clientes. Se construye con plataformas como VMware vSphere, OpenStack o Proxmox en clúster.

Por qué existe: requisitos regulatorios o de soberanía (bancos, gobierno, salud), datos que no pueden salir de un país, o cargas estables donde el hardware propio sale más barato a largo plazo.

Analogía: una piscina en tu casa. Solo la usas tú, la mantienes tú y su tamaño es el que construiste.

Ejemplo: un banco monta OpenStack sobre 50 servidores propios; los equipos internos piden VMs desde un portal como en AWS, pero los datos de clientes nunca salen del edificio.

### Public

Una **nube pública** es una infraestructura cloud propiedad de un proveedor que la ofrece por internet a cualquier cliente que pague, compartiendo el hardware entre muchos clientes (multi-tenancy) y aislándolos por virtualización e identidad. Es AWS, Azure, Google Cloud, Oracle Cloud y similares.

Por qué existe: economía de escala. El proveedor compra hardware por millones y lo reparte; el cliente obtiene capacidad casi infinita en minutos, sin inversión inicial, en regiones de todo el mundo.

Analogía: una piscina municipal. Cualquiera entra pagando la entrada, es enorme, alguien más la mantiene, y cada quien está en su carril.

Ejemplo: una startup de 5 personas lanza su aplicación en `us-east-1` y `eu-west-1` en una tarde, con un gasto de 200 dólares el primer mes.

El riesgo propio de la nube pública es que el recurso está a un permiso de distancia de internet: un bucket, una base de datos o un panel puede quedar expuesto a todo el mundo con un solo cambio de configuración.

### Hybrid

Una **nube híbrida** es una arquitectura que combina infraestructura privada (u on-premises) con una o más nubes públicas, conectadas para que datos y aplicaciones se muevan entre ellas. Es el modelo más común en empresas grandes: lo sensible o lo heredado se queda en casa y lo elástico va a la pública.

Cómo se conecta: VPN site-to-site cifrada o enlaces dedicados (AWS Direct Connect, Azure ExpressRoute, Google Cloud Interconnect), y una identidad unificada (por ejemplo, el Active Directory local sincronizado con Microsoft Entra ID).

Analogía: tener piscina en casa y además abono de la municipal para los días que vienen 40 invitados (a esto se le llama "cloud bursting").

Ejemplo: un hospital guarda las historias clínicas en su centro de datos y usa la nube pública para la web de citas y para procesar en lotes imágenes anonimizadas.

> [!NOTE]
> El punto débil de la híbrida es la unión: el túnel y la identidad compartida. Un atacante que compromete la nube puede saltar a la red interna por la VPN, y al revés; por eso el enlace se filtra como si fuera internet.

```
Private → una sola organización; máximo control y cumplimiento; capacidad limitada a lo comprado.
Public → proveedor compartido por muchos clientes; elasticidad y pago por uso; riesgo de exposición por configuración.
Hybrid → privada + pública conectadas (VPN, Direct Connect, ExpressRoute); el enlace es el punto débil.
Community → compartida por organizaciones con requisitos comunes (NIST SP 800-145).
```

## Common Cloud Environments

Los tres grandes proveedores públicos ofrecen servicios equivalentes con nombres distintos. Conocer la equivalencia permite leer cualquier informe de incidente o documentación sin importar la nube:

```
Función              AWS                    Azure                       GCP
VMs                  EC2                    Virtual Machines            Compute Engine
Almacén de objetos   S3                     Blob Storage                Cloud Storage
Identidad / permisos IAM                    Entra ID + Azure RBAC       Cloud IAM
Serverless           Lambda                 Azure Functions             Cloud Run functions
IaC nativo           CloudFormation         ARM / Bicep                 Infrastructure Manager
Auditoría de API     CloudTrail             Activity Log                Cloud Audit Logs
Red privada          VPC                    VNet                        VPC
CLI                  aws                    az                          gcloud
```

### AWS

**AWS** (Amazon Web Services) es la nube pública de Amazon, lanzada en 2006 con S3 y EC2, y el proveedor con mayor cuota de mercado (alrededor de un 30 %). Su modelo de seguridad gira alrededor de IAM: usuarios, grupos, roles (identidades que se "asumen" y entregan credenciales temporales) y políticas JSON. La unidad de aislamiento es la cuenta; las empresas usan muchas cuentas agrupadas en AWS Organizations.

Analogía: el supermercado más grande y antiguo del barrio: tiene de todo, a veces cuesta encontrarlo, y casi todos los manuales asumen que compras ahí.

Ejemplo: comprobar con qué identidad estás operando, lo primero que hace un atacante con una clave robada y lo primero que debería hacer un defensor:

```
$ aws sts get-caller-identity
{
    "UserId": "AIDAEXAMPLE1234567890",
    "Account": "111122223333",
    "Arn": "arn:aws:iam::111122223333:user/dev-juan"
}
```

### GCP

**GCP** (Google Cloud Platform) es la nube pública de Google, fuerte en datos, analítica, contenedores (Kubernetes nació en Google) e IA, con alrededor de un 12 % del mercado. Organiza los recursos en una jerarquía organización → carpetas → proyectos, y los permisos (roles de Cloud IAM) se heredan hacia abajo. Las aplicaciones usan cuentas de servicio (service accounts) como identidad.

Analogía: una tienda especializada: menos pasillos que el supermercado, pero los mejores en lo suyo.

Ejemplo:

```
$ gcloud projects list
PROJECT_ID        NAME          PROJECT_NUMBER
acme-prod-4821    acme-prod     123456789012
$ gcloud projects get-iam-policy acme-prod-4821 --format=json | grep -A2 '"role": "roles/owner"'
```

Un error típico en GCP: crear claves JSON descargables para cuentas de servicio y dejarlas en repositorios; la recomendación es evitarlas y usar federación de identidad.

### Azure

**Azure** es la nube pública de Microsoft, con alrededor de un 20 % del mercado y muy fuerte en empresas que ya usan Windows, Active Directory y Microsoft 365. La identidad vive en Microsoft Entra ID (antes Azure AD) y los permisos sobre recursos en Azure RBAC; los recursos se organizan en grupos de administración → suscripciones → grupos de recursos.

Analogía: la tienda de la marca que ya tienes en casa: si toda tu oficina usa Microsoft, todo encaja con lo que ya conoces.

Ejemplo:

```
$ az account show --query "{sub:name, user:user.name}"
{ "sub": "Acme-Produccion", "user": "ana@acme.com" }
$ az role assignment list --assignee ana@acme.com --output table
Principal      Role     Scope
ana@acme.com   Owner    /subscriptions/0000-.../resourceGroups/web-prod
```

Por qué importa la mezcla en seguridad: comprometer Entra ID suele dar acceso a la vez al correo, a los documentos y a la infraestructura de Azure, porque es la misma identidad.

```
AWS → mayor cuota; cuentas + IAM con roles y políticas JSON; CloudTrail.
GCP → organización > carpetas > proyectos; Cloud IAM heredado; service accounts.
Azure → Entra ID + RBAC; suscripciones y grupos de recursos; ligado a Microsoft 365.
```

## Common Cloud Storage

### S3

**Amazon S3** (Simple Storage Service) es un servicio de almacenamiento de objetos que guarda archivos de cualquier tamaño (hasta 5 TB por objeto) dentro de contenedores llamados buckets, accesibles por HTTPS a través de una API. Almacenamiento de objetos significa que no hay discos ni carpetas reales: cada archivo es un objeto con una clave (su nombre completo, como `facturas/2026/enero.pdf`), sus datos y sus metadatos. Se usa para backups, logs, sitios web estáticos, datos de análisis y archivos de aplicaciones.

Analogía: un guardarropa de un teatro con millones de ganchos. Entregas el abrigo, te dan un ticket (la clave) y con ese ticket lo recuperas; no te importa en qué estante está.

Buckets. Cada bucket tiene un nombre único en todo AWS (de 3 a 63 caracteres, minúsculas, números, puntos y guiones) y vive en una región. Ese nombre forma parte de una URL pública:

```
https://<bucket>.s3.<region>.amazonaws.com/<clave>
https://acme-backups.s3.us-east-1.amazonaws.com/db/2026-10-01.sql.gz
```

Como el nombre es global y predecible (`acme-backups`, `acme-logs`, `acme-dev`), un atacante puede adivinar buckets de una empresa probando nombres.

Políticas: quién puede hacer qué se decide combinando varias capas, y una denegación explícita siempre gana:

- Políticas IAM: pegadas a un usuario o rol ("Juan puede leer el bucket X").
- Bucket policy: pegada al bucket, en JSON ("cualquiera puede leer los objetos de este bucket").
- ACLs: el mecanismo antiguo por objeto; desde abril de 2023 los buckets nuevos las tienen desactivadas (Object Ownership = Bucket owner enforced).
- Block Public Access: un interruptor de cuatro opciones, a nivel de bucket y de cuenta, que anula cualquier política o ACL que haría público el bucket. Desde abril de 2023 viene activado por defecto en buckets nuevos.

Una bucket policy que hace público el bucket entero; la clave es `"Principal": "*"`, que significa "cualquiera, sin autenticarse":

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::acme-backups", "arn:aws:s3:::acme-backups/*"]
  }]
}
```

Buckets públicos mal configurados. Es la causa clásica de filtraciones en la nube: listas de clientes, backups de bases de datos y credenciales encontradas en buckets abiertos. Los errores típicos son:

- `"Principal": "*"` con `s3:ListBucket`: cualquiera puede listar todo el contenido.
- Dar permiso a "Authenticated Users" (cualquier cuenta de AWS del mundo, no solo las tuyas).
- Permitir `s3:PutObject` público: un atacante puede subir o reemplazar archivos (por ejemplo, el JavaScript de tu sitio web).
- Desactivar Block Public Access "para probar" y olvidarse.

Ejemplo concreto en un laboratorio legal (flaws.cloud, nivel 1): un bucket que permite listar sin credenciales se enumera así:

```
$ aws s3 ls s3://flaws.cloud/ --no-sign-request --region us-west-2
2017-03-14 03:00:38       2575 hint1.html
2017-03-03 04:05:17       1707 hint2.html
2017-03-03 04:05:11       1101 hint3.html
2024-02-22 02:32:41       2861 index.html
2018-07-10 16:47:16      15979 logo.png
2017-02-27 01:59:28         46 robots.txt
2017-02-27 01:59:30       1051 secret-xxxxxxx.html     <- nombre ocultado para no resolverte el nivel
```

`--no-sign-request` significa "sin credenciales": si esto devuelve un listado en vez de `AccessDenied`, el bucket es público para todo internet.

Defensa, del lado del dueño del bucket:

```
$ aws s3api get-public-access-block --bucket acme-backups
{ "PublicAccessBlockConfiguration": { "BlockPublicAcls": true, "IgnorePublicAcls": true,
  "BlockPublicPolicy": true, "RestrictPublicBuckets": true } }
```

Además: activar Block Public Access a nivel de cuenta, cifrado por defecto (SSE-S3 está activo en todos los buckets nuevos desde enero de 2023), versionado para recuperarse de borrados, registro de accesos y CloudTrail, y revisar con herramientas como IAM Access Analyzer o Prowler qué buckets son accesibles desde fuera. Si un sitio web necesita servir archivos públicos, se ponen detrás de CloudFront en lugar de abrir el bucket.

### Un bucket público es público para todo internet

> [!IMPORTANT]
> "Público" en S3 no significa "visible para mi empresa" sino "legible por cualquier persona del mundo sin credenciales"; y los atacantes no buscan tu bucket, escanean nombres de todos los buckets constantemente.

Un bucket público es un recurso cuyo contenido se puede leer, y a veces escribir, sin ninguna autenticación, y por eso su exposición se mide en minutos, no en meses. La confusión nace porque en una red interna "accesible" implica "dentro de la oficina"; en la nube pública no existe ese "dentro".

```
                    Red interna on-prem       Bucket S3 público
Quién puede llegar  quien está en la red      cualquiera con internet
Qué hace falta      estar en la oficina/VPN   conocer o adivinar el nombre
Dónde mirarlo       firewall perimetral       bucket policy + Block Public Access
```

La segunda fila es la que engaña: en la nube no hay paso previo, basta con el nombre. Límite de la idea: un bucket con `"Principal": "*"` sigue cerrado mientras Block Public Access esté activo, porque este anula la política; el día que alguien lo apaga, la política olvidada se vuelve una fuga. Por eso se revisan las dos capas juntas.

```
S3 → almacenamiento de objetos por HTTPS; objetos de hasta 5 TB en buckets.
Bucket → contenedor de nombre único global; forma parte de una URL pública.
Política IAM → permisos pegados a la identidad.
Bucket policy → permisos pegados al bucket; "Principal": "*" = cualquiera.
ACL → mecanismo antiguo por objeto; desactivado por defecto desde 2023.
Block Public Access → anula políticas/ACL públicas; activado por defecto desde 2023.
--no-sign-request → probar acceso anónimo; listado = bucket público.
```

## Recursos para aprender y practicar

### Videos

- [AWS Compliance - The Shared Responsibility Model](https://www.youtube.com/watch?v=U632-ND7dKQ) — Amazon Web Services; la explicación oficial de "of the cloud" frente a "in the cloud" (Security in the Cloud).
- [Cloud Infrastructures - CompTIA Security+ SY0-701 - 3.1](https://www.youtube.com/watch?v=8qpQ8Q6xxiU) — Professor Messer; responsabilidad compartida, nube híbrida, IaC y serverless desde la óptica de Security+ (varios nodos).
- [Cloud Models - CompTIA Network+ N10-009 - 1.3](https://www.youtube.com/watch?v=wI2x6eGF1Yg) — Professor Messer; público, privado, híbrido (Cloud Models).
- [Public Cloud Explained](https://www.youtube.com/watch?v=KaCyfQ7luVY) y [What is Hybrid Cloud?](https://www.youtube.com/watch?v=sUoeVhbp4cQ) — IBM Technology; modelos público e híbrido (Public, Hybrid).
- [What is Infrastructure as Code?](https://www.youtube.com/watch?v=zWw2wuiKd5o) — IBM Technology; declarativo frente a imperativo (Infrastructure as Code).
- [Terraform in 100 Seconds](https://www.youtube.com/watch?v=tomUWcQ0P3k) — Fireship; Terraform en dos minutos (Infrastructure as Code).
- [What is Serverless?](https://www.youtube.com/watch?v=vxJobGtqKVM) — IBM Technology; y [Serverless Computing in 100 Seconds](https://www.youtube.com/watch?v=W_VV2Fx32_Y) — Fireship (Serverless).
- [DevOps CI/CD Explained in 100 Seconds](https://www.youtube.com/watch?v=scEDHsr3APg) — Fireship; el pipeline de despliegue (deploying in the cloud).
- [Amazon S3 Access Control - IAM Policies, Bucket Policies and ACLs](https://www.youtube.com/watch?v=xFzJw6wJ8eY) — Digital Cloud Training; las capas de permisos de S3 (S3).
- [Dumping S3 Buckets | Exploiting S3 Bucket Misconfigurations](https://www.youtube.com/watch?v=ITSZ8743MUk) — HackerSploit; enumeración de buckets mal configurados (S3).
- [A Beginner's Guide To Exploiting AWS Misconfigurations | Flaws.Cloud Full Walkthrough](https://www.youtube.com/watch?v=Suxqxd74a_Q) — CYBERWOX; resolución completa de flaws.cloud (S3, AWS; verlo después de intentarlo).
- [AWS Certified Cloud Practitioner Certification Course (CLF-C02)](https://www.youtube.com/watch?v=NhDYbskXRgc) — freeCodeCamp; curso largo de AWS desde cero (AWS, cloud vs on-prem).

### Lectura y documentación

- [AWS Shared Responsibility Model](https://aws.amazon.com/compliance/shared-responsibility-model/), [Azure: Shared responsibility in the cloud](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility) y [Google Cloud: Shared responsibilities and shared fate](https://cloud.google.com/architecture/framework/security/shared-responsibility-shared-fate) — el modelo en las tres nubes (Security in the Cloud, Common Cloud Environments).
- [NIST SP 800-145, The NIST Definition of Cloud Computing](https://csrc.nist.gov/pubs/sp/800/145/final) — definición oficial de modelos de despliegue y servicio (Cloud Models).
- [NIST SP 800-144, Guidelines on Security and Privacy in Public Cloud Computing](https://csrc.nist.gov/pubs/sp/800/144/final) — riesgos de la nube pública (Security in the Cloud, Public).
- [AWS Well-Architected: Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html) — prácticas de seguridad de referencia en AWS (AWS, deploying).
- [S3: Blocking public access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html), [Bucket policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html), [Object Ownership](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html) y [Security best practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html) — documentación oficial (S3).
- [AWS Lambda: What is Lambda?](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) — modelo serverless de AWS (Serverless).
- [Terraform: What is Terraform?](https://developer.hashicorp.com/terraform/intro) — introducción oficial (Infrastructure as Code).
- [Checkov](https://github.com/bridgecrewio/checkov) y [Prowler](https://github.com/prowler-cloud/prowler) — escáner de IaC y auditor de configuración cloud (IaC, S3).
- [IMDS en EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) y [Krebs: What We Can Learn from the Capital One Hack](https://krebsonsecurity.com/2019/08/what-we-can-learn-from-the-capital-one-hack/) — el caso Capital One y la defensa IMDSv2 (Security in the Cloud).
- [MITRE ATT&CK Cloud Matrix](https://attack.mitre.org/matrices/enterprise/cloud/) y [T1530, Data from Cloud Storage](https://attack.mitre.org/techniques/T1530/) — técnicas de ataque en la nube (ver [`13-frameworks-de-amenazas`](../13-frameworks-de-amenazas/)).
- [CSA Top Threats](https://cloudsecurityalliance.org/research/topics/top-threats) — amenazas principales según la Cloud Security Alliance (Security in the Cloud).
- [Hacking The Cloud](https://hackingthe.cloud/) — enciclopedia de técnicas ofensivas y defensivas en AWS, Azure y GCP (todos los nodos).

### Práctica

- [flaws.cloud](http://flaws.cloud/) — 6 niveles sobre errores reales de AWS: buckets públicos, claves filtradas, snapshots públicos, metadatos; sin cuenta de AWS necesaria para empezar (S3, Security in the Cloud).
- [flaws2.cloud](http://flaws2.cloud/) — continuación con camino de atacante y camino de defensor (análisis de CloudTrail) (AWS, Serverless, deploying).
- [CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat) — Rhino Security Labs; despliega con Terraform escenarios vulnerables en tu propia cuenta de AWS (IAM, Lambda, S3). Practica IaC y AWS a la vez. Usa una cuenta dedicada, pon una alerta de facturación y ejecuta `cloudgoat destroy` al terminar. Complemento: [Pacu](https://github.com/RhinoSecurityLabs/pacu), el framework de explotación de AWS de los mismos autores.
- [TryHackMe: Cloud Security Pitfalls](https://tryhackme.com/room/cloudsecuritypitfalls) — gratis; errores típicos de configuración en la nube (Security in the Cloud).
- [TryHackMe: First Steps Into AWS](https://tryhackme.com/room/awsfirststeps) — consola y CLI de AWS en un entorno provisto (AWS).
- [TryHackMe: AWS Security - S3cret Santa](https://tryhackme.com/room/cloudenum-aoc2025-y4u7i0o3p6) — gratis; enumerar y asegurar buckets S3 mal configurados (S3).
- [TryHackMe: Intro to IaC](https://tryhackme.com/room/introtoiac) — gratis; Infrastructure as Code y sus riesgos (IaC). Para serverless, el escenario de Lambda de [CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat) se monta en tu propia cuenta.
- Ejercicio en casa: con la capa gratuita de [AWS](https://aws.amazon.com/free/), crea con Terraform un bucket con `aws_s3_bucket_public_access_block`, pasa `checkov -d .`, luego quita el bloque y compara el informe. Destruye todo con `terraform destroy` (IaC, S3).
- Ejercicio en casa: desde una máquina sin credenciales ejecuta `aws s3 ls s3://<tu-bucket> --no-sign-request` antes y después de poner una bucket policy con `"Principal": "*"` y desactivar Block Public Access; vuelve a activarlo y confirma el `AccessDenied` (S3).

## Cuadro resumen

Todo lo visto, en una línea por término.

Understand the Concept of Security in the Cloud

```
Seguridad en la nube → controles y configuraciones que protegen lo que una organización tiene en un proveedor.
Seguridad DE la nube → proveedor: edificios, hardware, red física, hypervisor.
Seguridad EN la nube → cliente: datos, identidades, permisos, configuración, SO en IaaS.
Datos e identidades → siempre del cliente, en IaaS, PaaS y SaaS.
Misconfiguration → causa principal de brechas cloud (Capital One: SSRF + IMDSv1 + rol amplio).
IMDSv2 → metadatos solo tras un PUT con token; frena el SSRF simple.
```

Understand the differences between cloud and on-premises

```
On-premises → hardware propio, CapEx, control físico, perímetro de red.
Cloud → recursos alquilados por API, OpEx, elasticidad, perímetro de identidad.
Evidencia on-prem → discos y equipos físicos.
Evidencia cloud → logs de API (CloudTrail, Activity Log, Audit Logs) y snapshots.
```

Understand the concept of Infrastructure as Code

```
IaC → infraestructura definida en archivos de texto versionados y aplicada por herramienta.
Declarativo (Terraform, CloudFormation, Bicep) → describes el estado final.
Imperativo (Ansible, scripts) → describes los pasos.
terraform plan / apply → mostrar la diferencia / aplicarla.
tfstate → estado conocido; puede contener secretos; backend remoto cifrado.
Drift → diferencia entre el código y lo que realmente existe.
Checkov / tfsec → escanean IaC antes de desplegar.
```

Understand the Concept of Serverless

```
Serverless → el proveedor gestiona servidores y escalado; pagas por ejecución.
Función (Lambda, Azure Functions, Cloud Run functions) → código disparado por eventos.
Cold start → retraso de la primera ejecución al crear el entorno.
Límites Lambda → hasta 15 min por ejecución; 128 MB a 10.240 MB.
Rol de la función → define el daño máximo si la función es comprometida.
```

Understand the basics and general flow of deploying in the cloud

```
CI → compilar, probar y escanear cada cambio.
CD → publicar automáticamente lo que pasó CI.
Artefacto → la imagen o paquete exacto que se despliega.
Blue/green → dos versiones lado a lado; cambio de tráfico de golpe.
Canary → un porcentaje pequeño de tráfico primero.
OIDC en CI → credenciales temporales en lugar de claves fijas.
```

Cloud Models

```
Private → una sola organización; máximo control y cumplimiento; capacidad limitada a lo comprado.
Public → proveedor compartido por muchos clientes; elasticidad y pago por uso; riesgo de exposición por configuración.
Hybrid → privada + pública conectadas (VPN, Direct Connect, ExpressRoute); el enlace es el punto débil.
Community → compartida por organizaciones con requisitos comunes (NIST SP 800-145).
```

Common Cloud Environments

```
AWS → mayor cuota; cuentas + IAM con roles y políticas JSON; CloudTrail.
GCP → organización > carpetas > proyectos; Cloud IAM heredado; service accounts.
Azure → Entra ID + RBAC; suscripciones y grupos de recursos; ligado a Microsoft 365.
```

Common Cloud Storage

```
S3 → almacenamiento de objetos por HTTPS; objetos de hasta 5 TB en buckets.
Bucket → contenedor de nombre único global; forma parte de una URL pública.
Política IAM → permisos pegados a la identidad.
Bucket policy → permisos pegados al bucket; "Principal": "*" = cualquiera.
ACL → mecanismo antiguo por objeto; desactivado por defecto desde 2023.
Block Public Access → anula políticas/ACL públicas; activado por defecto desde 2023.
--no-sign-request → probar acceso anónimo; listado = bucket público.
Bucket público → legible por cualquiera en internet sin credenciales; se descubre adivinando el nombre.
```
