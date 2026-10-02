# Ejercicios: Cloud

Ejercicios guiados para hacer en tu propia cuenta de AWS. Cada uno dice qué vas a
lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

> [!WARNING]
> Todo se hace en una cuenta de AWS tuya y con recursos tuyos, dentro de la capa
> gratuita. S3 entra en la capa gratuita con límites (los primeros 12 meses: 5 GB de
> almacenamiento, 2.000 peticiones PUT y 20.000 GET al mes); estos ejercicios usan unos
> pocos KB y un puñado de peticiones, así que el coste esperado es cero o céntimos. Aun
> así, activa una alerta de facturación antes de empezar y ejecuta la limpieza
> (`terraform destroy` y borrado del bucket) al terminar para no dejar nada corriendo.
> Nunca uses buckets, cuentas ni dominios que no sean tuyos.

Antes de empezar, pon un presupuesto de aviso en AWS Billing (`Billing and Cost
Management > Budgets > Create budget`), tipo "Cost budget", importe 1 USD, con alerta
por correo al 80 %. Así te avisa si algo se dispara.

## Ejercicio 1: Bucket S3 con Terraform y escaneo con Checkov

Nodo: [Infrastructure as Code](README.md#understand-the-concept-of-infrastructure-as-code)
y [S3](README.md#s3).

Objetivo: crear con Terraform un bucket S3 privado con Block Public Access, comprobar
que Checkov lo aprueba, quitar luego el bloqueo público, ver que el informe de Checkov
pasa a fallar, y destruir todo con `terraform destroy`.

Necesitas: una cuenta de AWS propia con la CLI configurada (`aws configure` con una
clave de un usuario IAM tuyo con permisos sobre S3); Terraform u OpenTofu; Checkov.
Tiempo: 45-60 min. Instalación:
- Debian/Ubuntu: `sudo apt install awscli` y `pipx install checkov`; Terraform desde el
  [repositorio de HashiCorp](https://developer.hashicorp.com/terraform/install).
- Arch: `sudo pacman -S aws-cli-v2 terraform` y `pipx install checkov`.

### Pasos

1. Crea una carpeta de trabajo y comprueba con qué identidad operas:
   ```bash
   mkdir ~/lab-s3 && cd ~/lab-s3
   aws sts get-caller-identity
   ```
   - `mkdir ~/lab-s3` → crea la carpeta de trabajo.
   - `&& cd ~/lab-s3` → si lo anterior tuvo éxito, entra en ella.
   - `aws sts get-caller-identity` → consulta al servicio STS y muestra la identidad (cuenta, usuario o rol) con la que la CLI está operando.

   El `Arn` debe ser tu usuario; si da error, revisa `aws configure`.
2. Crea `main.tf` con un bucket privado y su bloqueo de acceso público. Sustituye el
   nombre por uno único tuyo (minúsculas, sin puntos):
   ```hcl
   terraform {
     required_providers {
       aws = {
         source  = "hashicorp/aws"
         version = "~> 5.0"
       }
     }
   }

   provider "aws" {
     region = "us-east-1"
   }

   resource "aws_s3_bucket" "lab" {
     bucket = "lab-s3-tuyo-2026-0001"
   }

   resource "aws_s3_bucket_public_access_block" "lab" {
     bucket                  = aws_s3_bucket.lab.id
     block_public_acls       = true
     block_public_policy     = true
     ignore_public_acls      = true
     restrict_public_buckets = true
   }
   ```
   - `terraform { required_providers { ... } }` → bloque que declara qué proveedores necesita el módulo.
   - `aws = { source = "hashicorp/aws", version = "~> 5.0" }` → declara el proveedor AWS; `source` es su origen en el registro y `version = "~> 5.0"` acepta cualquier versión 5.x pero no la 6.0.
   - `provider "aws" { region = "us-east-1" }` → configura el proveedor AWS; `region` fija la región donde se crean los recursos.
   - `resource "aws_s3_bucket" "lab" { bucket = "..." }` → declara un bucket S3; `aws_s3_bucket` es el tipo, `lab` el nombre local con el que lo referencia Terraform y `bucket` el nombre real y único del bucket.
   - `resource "aws_s3_bucket_public_access_block" "lab" { ... }` → declara el bloqueo de acceso público del bucket.
   - `bucket = aws_s3_bucket.lab.id` → enlaza este recurso al bucket anterior por su id.
   - `block_public_acls = true` → bloquea la creación de ACL públicas nuevas.
   - `block_public_policy = true` → impide aplicar políticas de bucket que sean públicas.
   - `ignore_public_acls = true` → ignora cualquier ACL pública ya existente.
   - `restrict_public_buckets = true` → limita el acceso al bucket solo a principales autorizados de la cuenta.

3. Inicializa y revisa el plan antes de crear nada:
   ```bash
   terraform init
   terraform plan
   ```
   - `terraform init` → descarga los proveedores e inicializa el directorio de trabajo.
   - `terraform plan` → calcula y muestra los cambios que aplicaría, sin ejecutarlos todavía.

   Debe decir `Plan: 2 to add, 0 to change, 0 to destroy`.
4. Escanea el código con Checkov antes de aplicar:
   ```bash
   checkov -d .
   ```
   - `checkov` → analizador estático de infraestructura como código que busca malas configuraciones.
   - `-d .` → analiza el directorio indicado; `.` es el directorio actual.

   Fíjate en los controles de S3 (como `CKV_AWS_53` a `CKV_AWS_56`, el Block Public
   Access): deben salir en `PASSED`. Anota cuántos checks pasan.
5. Aplica y comprueba en AWS que el bucket existe y es privado:
   ```bash
   terraform apply        # escribe "yes" para confirmar
   aws s3api get-public-access-block --bucket lab-s3-tuyo-2026-0001
   ```
   - `terraform apply` → aplica de verdad los cambios del plan; pide confirmación escribiendo "yes".
   - `aws s3api get-public-access-block` → consulta la configuración de bloqueo de acceso público de un bucket.
   - `--bucket lab-s3-tuyo-2026-0001` → el bucket consultado.

   Las cuatro opciones deben estar en `true`.
6. Ahora introduce el fallo a propósito: borra el recurso
   `aws_s3_bucket_public_access_block` de `main.tf` (o pon sus cuatro valores a
   `false`) y vuelve a escanear:
   ```bash
   checkov -d .
   ```
   - `checkov -d .` → vuelve a analizar el directorio actual (ver paso 4).

   Los mismos controles de Block Public Access pasan ahora a `FAILED`. Compara el conteo
   de PASSED/FAILED con el del paso 4.

### Resultado esperado

Dos informes de Checkov: el primero con los controles de acceso público en PASSED y el
segundo con esos mismos en FAILED, y un bucket privado creado y luego destruido.

### Comprueba que lo lograste

- ¿La diferencia entre los dos informes de Checkov está exactamente en los controles de
  Block Public Access? Esa es la prueba de que el escáner detecta el error en el código,
  antes de desplegar.
- ¿`get-public-access-block` devolvió las cuatro opciones en `true` mientras el bloqueo
  estaba puesto?

### Limpieza

```bash
cd ~/lab-s3
terraform destroy      # escribe "yes"; deja la cuenta como estaba
```

- `cd ~/lab-s3` → entra en la carpeta del laboratorio.
- `terraform destroy` → elimina todos los recursos que Terraform creó; pide confirmación escribiendo "yes".
Confirma que el bucket ya no aparece: `aws s3 ls | grep lab-s3-tuyo`. Borra la carpeta
`~/lab-s3` si quieres. Revisa en Billing que no quede nada el día siguiente.

## Ejercicio 2: Observa cuándo un bucket se vuelve público y ciérralo

Nodo: [S3](README.md#s3) y
[Un bucket público es público para todo internet](README.md#un-bucket-público-es-público-para-todo-internet).

Objetivo: comprobar en tu propio bucket que el acceso anónimo (`--no-sign-request`)
devuelve `AccessDenied` mientras está protegido, que se vuelve legible al poner una
bucket policy con `"Principal": "*"` y desactivar Block Public Access, y que vuelve a
denegar en cuanto reactivas el bloqueo.

Necesitas: la misma cuenta de AWS propia y CLI del ejercicio 1; un bucket propio (puedes
reutilizar el del ejercicio 1 antes de destruirlo, o crear uno rápido). Tiempo: 30 min.
Solo tu bucket; nunca pruebes esto contra buckets ajenos.

> [!WARNING]
> Este ejercicio deja tu bucket público durante unos minutos. Úsalo con un bucket vacío
> o con un archivo de prueba sin valor, no con datos reales, y vuelve a cerrarlo en
> cuanto termines el paso 5.

### Pasos

1. Crea un bucket de prueba y sube un archivo intrascendente:
   ```bash
   aws s3 mb s3://lab-pub-tuyo-2026-0001 --region us-east-1
   echo "archivo de prueba de laboratorio" > prueba.txt
   aws s3 cp prueba.txt s3://lab-pub-tuyo-2026-0001/
   ```
   - `aws s3 mb s3://lab-pub-tuyo-2026-0001` → crea el bucket (`mb` = make bucket).
   - `--region us-east-1` → la región donde se crea.
   - `echo "archivo de prueba de laboratorio" > prueba.txt` → crea un archivo local con ese texto.
   - `aws s3 cp prueba.txt s3://lab-pub-tuyo-2026-0001/` → copia (sube) el archivo al bucket.

2. Intenta listarlo sin credenciales. Debe denegar, porque los buckets nuevos vienen con
   Block Public Access activado:
   ```bash
   aws s3 ls s3://lab-pub-tuyo-2026-0001/ --no-sign-request --region us-east-1
   ```
   - `aws s3 ls s3://lab-pub-tuyo-2026-0001/` → lista el contenido del bucket.
   - `--no-sign-request` → no firma la petición con tus credenciales: la CLI actúa como un usuario anónimo de internet.
   - `--region us-east-1` → la región del bucket.

   Salida esperada: un error `AccessDenied`.
3. Desactiva Block Public Access (a nivel de bucket) para poder aplicar una política
   pública:
   ```bash
   aws s3api delete-public-access-block --bucket lab-pub-tuyo-2026-0001
   ```
   - `aws s3api delete-public-access-block` → elimina la configuración de bloqueo de acceso público del bucket.
   - `--bucket lab-pub-tuyo-2026-0001` → el bucket afectado.
4. Crea `policy.json` con una bucket policy abierta (sustituye el nombre del bucket) y
   aplícala:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": "*",
       "Action": ["s3:GetObject", "s3:ListBucket"],
       "Resource": [
         "arn:aws:s3:::lab-pub-tuyo-2026-0001",
         "arn:aws:s3:::lab-pub-tuyo-2026-0001/*"
       ]
     }]
   }
   ```
   ```bash
   aws s3api put-bucket-policy --bucket lab-pub-tuyo-2026-0001 \
     --policy file://policy.json
   ```
   Campos de `policy.json`:
   - `"Version": "2012-10-17"` → versión del lenguaje de políticas de IAM.
   - `"Statement"` → lista de reglas de la política.
   - `"Effect": "Allow"` → la regla concede el permiso (sería `Deny` para negarlo).
   - `"Principal": "*"` → a quién aplica; `*` significa cualquiera, incluido el acceso anónimo.
   - `"Action": ["s3:GetObject", "s3:ListBucket"]` → acciones permitidas: leer objetos y listar el bucket.
   - `"Resource": [...]` → sobre qué recursos aplica: el bucket (`arn:aws:s3:::nombre`) y todos sus objetos (`.../*`).

   El comando que la aplica:
   - `aws s3api put-bucket-policy` → asigna una bucket policy al bucket.
   - `--bucket lab-pub-tuyo-2026-0001` → el bucket afectado.
   - `--policy file://policy.json` → lee la política del archivo local; `file://` indica que el valor es una ruta a un archivo, no el JSON en línea.
5. Vuelve a listar sin credenciales. Ahora sí debe devolver el contenido:
   ```bash
   aws s3 ls s3://lab-pub-tuyo-2026-0001/ --no-sign-request --region us-east-1
   ```
   - `aws s3 ls ... --no-sign-request --region us-east-1` → vuelve a listar el bucket como anónimo (ver paso 2).

   Salida esperada: la línea de `prueba.txt`. Eso significa que el bucket es legible por
   cualquiera en internet.
6. Cierra de inmediato: quita la política y reactiva Block Public Access, luego confirma
   que vuelve a denegar:
   ```bash
   aws s3api delete-bucket-policy --bucket lab-pub-tuyo-2026-0001
   aws s3api put-public-access-block --bucket lab-pub-tuyo-2026-0001 \
     --public-access-block-configuration \
     BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
   aws s3 ls s3://lab-pub-tuyo-2026-0001/ --no-sign-request --region us-east-1
   ```
   - `aws s3api delete-bucket-policy --bucket ...` → quita la bucket policy aplicada en el paso 4.
   - `aws s3api put-public-access-block` → vuelve a poner el bloqueo de acceso público en el bucket.
   - `--public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true` → las cuatro opciones del bloqueo, todas activadas: bloquea e ignora las ACL públicas, bloquea las políticas públicas y restringe el bucket a principales autorizados.
   - `aws s3 ls ... --no-sign-request --region us-east-1` → vuelve a listar como anónimo para comprobar que ya deniega (ver paso 2).

   Debe volver el `AccessDenied`.

### Resultado esperado

Tres salidas de la misma orden `aws s3 ls --no-sign-request`: denegado, luego listado,
luego denegado otra vez. Es la demostración de que Block Public Access anula la política
pública.

### Comprueba que lo lograste

- ¿Viste el contenido sin credenciales solo en el paso 5, con la política abierta y el
  bloqueo quitado a la vez?
- ¿Entiendes por qué con Block Public Access activo la política `"Principal": "*"` no
  hace nada? Porque el bloqueo tiene prioridad sobre la política.

### Limpieza

```bash
aws s3 rm s3://lab-pub-tuyo-2026-0001/prueba.txt
aws s3 rb s3://lab-pub-tuyo-2026-0001
rm -f prueba.txt policy.json
```

- `aws s3 rm s3://lab-pub-tuyo-2026-0001/prueba.txt` → borra el objeto del bucket (`rm` = remove).
- `aws s3 rb s3://lab-pub-tuyo-2026-0001` → elimina el bucket, que debe estar vacío (`rb` = remove bucket).
- `rm -f prueba.txt policy.json` → borra los archivos locales; `-f` no se queja si alguno ya no existe.
Si creaste el bucket con Terraform en el ejercicio 1, usa `terraform destroy` en su
lugar. Confirma con `aws s3 ls` que ya no está y revisa Billing al día siguiente.
