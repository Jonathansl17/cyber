# Ejercicios: Cloud

Ejercicios guiados para hacer en tu propia cuenta de AWS. Cada uno dice qué vas a
lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

> [!WARNING]
> Todo se hace en una cuenta de AWS tuya y con recursos tuyos, dentro de la capa
> gratuita. Desde el 15 de julio de 2025 las cuentas nuevas ya no tienen la antigua capa
> gratuita "de 12 meses": eliges entre un plan Free (hasta 200 USD en créditos, 100 al
> abrir la cuenta y hasta 100 más completando actividades, durante un máximo de 6 meses;
> la cuenta se cierra sola al acabarse los créditos o el plazo, y no te cobran salvo que
> pases al plan Paid) y un plan Paid (pago por uso, también con créditos). Las cuentas
> anteriores a esa fecha conservan las condiciones viejas. Los detalles cambian:
> compruébalos en [aws.amazon.com/free](https://aws.amazon.com/free/). Estos ejercicios
> usan unos pocos KB en S3, un puñado de peticiones y servicios sin coste (IAM, STS,
> historial de eventos de CloudTrail, validación de políticas de Access Analyzer), así
> que el coste esperado es cero o céntimos de crédito. Aun
> así, activa una alerta de facturación antes de empezar y ejecuta la limpieza
> (`terraform destroy` y borrado del bucket) al terminar para no dejar nada corriendo.
> Nunca uses buckets, cuentas ni dominios que no sean tuyos.

Antes de empezar, pon un presupuesto de aviso en AWS Billing (`Billing and Cost
Management > Budgets > Create budget`), tipo "Cost budget", importe 1 USD, con alerta
por correo al 80 %. Así te avisa si algo se dispara.

## Preparación: credenciales de la CLI sin claves permanentes

Los tres ejercicios usan la AWS CLI v2. No uses la cuenta root para trabajar y evita las
claves de acceso de larga duración de un usuario IAM (`AKIA...`): si se filtran (un
`git push` descuidado, un backup) sirven hasta que alguien las borre, y son una de las
causas más comunes de cuentas comprometidas. Prefiere credenciales temporales:

- Opción recomendada, IAM Identity Center: en la consola activa IAM Identity Center,
  crea un usuario para ti y asígnale tu cuenta con un permission set (para estos
  laboratorios vale `AdministratorAccess`; en un entorno real, uno más acotado). Luego:
  ```bash
  aws configure sso
  aws sso login --profile lab
  export AWS_PROFILE=lab
  ```
  - `aws configure sso` → asistente que pide la URL de inicio de tu portal de Identity Center (`https://...awsapps.com/start`), la región, la cuenta y el rol, y guarda un perfil en `~/.aws/config`; cuando pregunte el nombre del perfil, escribe `lab`.
  - `aws sso login --profile lab` → abre el navegador para iniciar sesión y deja en caché credenciales temporales (caducan en horas).
  - `export AWS_PROFILE=lab` → hace que todos los comandos `aws` de esta terminal usen ese perfil sin repetir `--profile`.
- Alternativa sencilla: con una versión reciente de la CLI v2, `aws login` usa la sesión
  de la consola (con un usuario IAM tuyo, no con root) para obtener credenciales
  temporales que se renuevan solas.
  - `aws login` → abre el navegador, inicias sesión en la consola y la CLI recibe credenciales temporales y un token de refresco.
- Último recurso, clave de usuario IAM (`aws configure`): si la usas, que sea de un
  usuario IAM (nunca de root), no la copies en ningún archivo del proyecto y bórrala al
  terminar los ejercicios (ver la limpieza del ejercicio 3).

Comprueba siempre con `aws sts get-caller-identity` con qué identidad estás operando.

## Ejercicio 1: Bucket S3 con Terraform y escaneo con Checkov

Nodo: [Infrastructure as Code](README.md#understand-the-concept-of-infrastructure-as-code)
y [S3](README.md#s3).

Objetivo: crear con Terraform un bucket S3 privado con Block Public Access, comprobar
que Checkov lo aprueba, quitar luego el bloqueo público, ver que el informe de Checkov
pasa a fallar, y destruir todo con `terraform destroy`.

Necesitas: una cuenta de AWS propia con la CLI configurada como en la
[preparación](#preparación-credenciales-de-la-cli-sin-claves-permanentes), con permisos
sobre S3; Terraform u OpenTofu; Checkov. Tiempo: 45-60 min. Instalación:
- Debian/Ubuntu: Ubuntu 24.04 ya no trae el paquete `awscli` en apt. Usa el instalador
  oficial de la CLI v2 (o `sudo snap install aws-cli --classic`):
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
  unzip awscliv2.zip
  sudo ./aws/install
  ```
  - `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip` → descarga el instalador oficial para x86_64 (en ARM la URL termina en `aarch64.zip`); `-o` lo guarda con ese nombre.
  - `unzip awscliv2.zip` → descomprime el instalador en la carpeta `aws/`.
  - `sudo ./aws/install` → instala la CLI en `/usr/local/aws-cli` y crea el enlace `/usr/local/bin/aws`.

  Luego `pipx install checkov`, y Terraform desde el
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

   El `Arn` debe ser tu identidad: `arn:aws:sts::...:assumed-role/AWSReservedSSO_...`
   si usas Identity Center, o `arn:aws:iam::...:user/...` si usas un usuario IAM. Nunca
   debe terminar en `:root`. Si da error, repite la
   [preparación](#preparación-credenciales-de-la-cli-sin-claves-permanentes).
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

   Lo que vas a ver: los cuatro controles del recurso de bloqueo (`CKV_AWS_53` a
   `CKV_AWS_56`, uno por cada opción) y `CKV2_AWS_6` ("el bucket tiene un Public Access
   block") salen en `PASSED`. Pero el informe no sale limpio: varios controles del
   bucket salen en `FAILED` porque este bucket mínimo no tiene registro de accesos
   (`CKV_AWS_18`), versionado (`CKV_AWS_21`), replicación entre regiones
   (`CKV_AWS_144`), cifrado con KMS (`CKV_AWS_145`), reglas de ciclo de vida
   (`CKV2_AWS_61`) ni notificaciones de eventos (`CKV2_AWS_62`). La lista exacta depende
   de la versión de Checkov. Es normal: Checkov aplica un catálogo amplio de buenas
   prácticas, y no todas tienen sentido para un laboratorio. Para centrarte en el acceso
   público, repite el escaneo solo con esos controles:
   ```bash
   checkov -d . --check CKV_AWS_53,CKV_AWS_54,CKV_AWS_55,CKV_AWS_56,CKV2_AWS_6
   ```
   - `--check CKV_AWS_53,...,CKV2_AWS_6` → ejecuta solo los controles indicados (lista separada por comas) e ignora el resto.

   Debe dar `Passed checks: 5, Failed checks: 0`. Lo contrario sería `--skip-check`, que
   ejecuta todo menos los controles que listes; en un proyecto real se usa para descartar,
   de forma explícita y documentada, controles que no aplican, en vez de ignorar el
   informe entero. Anota los números de las dos ejecuciones.
5. Aplica y comprueba en AWS que el bucket existe y es privado:
   ```bash
   terraform apply        # escribe "yes" para confirmar
   aws s3api get-public-access-block --bucket lab-s3-tuyo-2026-0001
   ```
   - `terraform apply` → aplica de verdad los cambios del plan; pide confirmación escribiendo "yes".
   - `aws s3api get-public-access-block` → consulta la configuración de bloqueo de acceso público de un bucket.
   - `--bucket lab-s3-tuyo-2026-0001` → el bucket consultado.

   Las cuatro opciones deben estar en `true`.
6. Ahora introduce el fallo a propósito: en `main.tf`, pon a `false` los cuatro valores
   del recurso `aws_s3_bucket_public_access_block` (no lo apliques; solo vas a escanear
   el código) y vuelve a escanear con el filtro del paso 4:
   ```bash
   checkov -d . --check CKV_AWS_53,CKV_AWS_54,CKV_AWS_55,CKV_AWS_56,CKV2_AWS_6
   ```
   - `checkov -d . --check ...` → vuelve a analizar el directorio actual solo con los controles de acceso público (ver paso 4).

   Debe dar `Passed checks: 0, Failed checks: 5`: los cuatro `CKV_AWS_53` a `CKV_AWS_56`
   y también `CKV2_AWS_6`, que exige que `block_public_acls` y `block_public_policy`
   estén en `true`. Variante: si en vez de poner `false` borras el recurso entero, los
   controles `CKV_AWS_53` a `CKV_AWS_56` desaparecen del informe (ya no hay recurso que
   revisar) y el único que falla es `CKV2_AWS_6`. Por eso conviene tener los dos tipos de
   control. Al terminar, vuelve a poner los cuatro valores en `true` antes del
   `terraform destroy`.

### Resultado esperado

Tres informes de Checkov: el completo del paso 4 (acceso público en PASSED, varios
controles de logging, versionado, cifrado y similares en FAILED), el filtrado del paso 4
(5 PASSED, 0 FAILED) y el filtrado del paso 6 (0 PASSED, 5 FAILED), y un bucket privado
creado y luego destruido.

### Comprueba que lo lograste

- ¿La diferencia entre los dos informes filtrados está exactamente en los controles de
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

   Si en el paso 4 `put-bucket-policy` da `AccessDenied` aunque seas administrador, es
   que tu cuenta tiene Block Public Access activado también a nivel de cuenta (lo
   recomendable). Compruébalo con
   `aws s3control get-public-access-block --account-id <tu-id-de-cuenta>`; en ese caso
   puedes quedarte en este punto: ya has visto que el bloqueo impide abrir el bucket.
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
6. Cierra de inmediato reactivando solo Block Public Access, sin tocar la política
   pública, y confirma que vuelve a denegar:
   ```bash
   aws s3api put-public-access-block --bucket lab-pub-tuyo-2026-0001 \
     --public-access-block-configuration \
     BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
   aws s3api get-bucket-policy --bucket lab-pub-tuyo-2026-0001
   aws s3 ls s3://lab-pub-tuyo-2026-0001/ --no-sign-request --region us-east-1
   ```
   - `aws s3api put-public-access-block` → vuelve a poner el bloqueo de acceso público en el bucket.
   - `--public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true` → las cuatro opciones del bloqueo, todas activadas: bloquea e ignora las ACL públicas, bloquea las políticas públicas nuevas y restringe el bucket a principales autorizados aunque tenga una política pública.
   - `aws s3api get-bucket-policy --bucket ...` → muestra la bucket policy actual; confirma que la política `"Principal": "*"` sigue puesta.
   - `aws s3 ls ... --no-sign-request --region us-east-1` → vuelve a listar como anónimo (ver paso 2).

   Debe volver el `AccessDenied` aunque la política abierta sigue ahí: esa es la prueba
   de que Block Public Access (en concreto `RestrictPublicBuckets`) anula la política
   pública. Si el cambio tarda unos segundos en propagarse, repite el `ls`.
7. Ahora quita también la política, que ya no hace falta, y comprueba que no queda
   ninguna:
   ```bash
   aws s3api delete-bucket-policy --bucket lab-pub-tuyo-2026-0001
   aws s3api get-bucket-policy --bucket lab-pub-tuyo-2026-0001
   ```
   - `aws s3api delete-bucket-policy --bucket ...` → quita la bucket policy aplicada en el paso 4.
   - `aws s3api get-bucket-policy --bucket ...` → vuelve a consultarla.

   La segunda orden debe fallar con `NoSuchBucketPolicy`: el bucket ya no tiene política.

### Resultado esperado

Tres salidas de la misma orden `aws s3 ls --no-sign-request`: denegado, luego listado,
luego denegado otra vez con la política pública todavía puesta (paso 6). Es la
demostración de que Block Public Access anula la política pública.

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

## Ejercicio 3: Mínimo privilegio con IAM e investigación en CloudTrail

Nodo: [Understand the Concept of Security in the Cloud](README.md#understand-the-concept-of-security-in-the-cloud)
y [AWS](README.md#aws).

Objetivo: crear un rol que solo puede leer un bucket concreto (mínimo privilegio),
validar su política con IAM Access Analyzer, comprobar con `simulate-principal-policy`
qué permite y qué no, usar el rol de verdad, y luego investigar en CloudTrail quién hizo
qué, cuándo y desde dónde, como lo harías en un incidente.

Necesitas: la cuenta y la CLI de la
[preparación](#preparación-credenciales-de-la-cli-sin-claves-permanentes), con permisos
de administrador; `jq` para leer JSON (`sudo apt install jq` o `sudo pacman -S jq`).
Tiempo: 45-60 min.

Coste: cero. IAM, STS, el historial de eventos de CloudTrail (los últimos 90 días de
eventos de administración, activado por defecto sin crear ningún trail) y la validación
de políticas de Access Analyzer no tienen coste; el bucket guarda un archivo de pocos
bytes. No crees un trail ni el analizador de accesos no utilizados: esos sí pueden
cobrarse.

### Pasos

1. Guarda tu número de cuenta y el nombre del bucket en variables, y crea el bucket con
   un archivo de prueba:
   ```bash
   CUENTA=$(aws sts get-caller-identity --query Account --output text)
   BUCKET=lab-iam-tuyo-2026-0001
   aws s3 mb s3://$BUCKET --region us-east-1
   echo "dato de laboratorio" > prueba.txt
   aws s3 cp prueba.txt s3://$BUCKET/
   ```
   - `CUENTA=$(...)` → guarda en la variable `CUENTA` la salida del comando entre paréntesis.
   - `aws sts get-caller-identity --query Account --output text` → pide tu identidad y, con `--query Account`, extrae solo el número de cuenta; `--output text` lo imprime sin comillas ni JSON.
   - `BUCKET=lab-iam-tuyo-2026-0001` → nombre de tu bucket; cámbialo por uno único tuyo.
   - `aws s3 mb s3://$BUCKET --region us-east-1` → crea el bucket en `us-east-1`.
   - `echo ... > prueba.txt` y `aws s3 cp prueba.txt s3://$BUCKET/` → crean un archivo local y lo suben.
2. Crea la política de confianza del rol, `trust.json`, que dice quién puede asumirlo
   (identidades de tu propia cuenta con permiso para ello) y créalo:
   ```bash
   cat > trust.json <<JSON
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": {"AWS": "arn:aws:iam::$CUENTA:root"},
       "Action": "sts:AssumeRole"
     }]
   }
   JSON
   aws iam create-role --role-name lab-lectura-s3 \
     --assume-role-policy-document file://trust.json
   ```
   - `cat > trust.json <<JSON ... JSON` → escribe el texto entre las dos marcas en `trust.json`, sustituyendo `$CUENTA` por tu número de cuenta.
   - `"Principal": {"AWS": "arn:aws:iam::$CUENTA:root"}` → confía en tu cuenta; no significa "el usuario root", sino que cualquier identidad de tu cuenta a la que su propia política le permita `sts:AssumeRole` puede asumir el rol.
   - `"Action": "sts:AssumeRole"` → la acción que se permite: asumir el rol.
   - `aws iam create-role --role-name lab-lectura-s3` → crea el rol con ese nombre.
   - `--assume-role-policy-document file://trust.json` → le asigna la política de confianza del archivo.

   Salida esperada: un JSON con `"Arn": "arn:aws:iam::<cuenta>:role/lab-lectura-s3"`.
3. Escribe la política de permisos, `perm.json`, con lo mínimo para leer ese bucket: listar
   el bucket y leer sus objetos, nada más. Valídala antes de aplicarla:
   ```bash
   cat > perm.json <<JSON
   {
     "Version": "2012-10-17",
     "Statement": [
       {"Effect": "Allow", "Action": "s3:ListBucket",
        "Resource": "arn:aws:s3:::$BUCKET"},
       {"Effect": "Allow", "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::$BUCKET/*"}
     ]
   }
   JSON
   aws accessanalyzer validate-policy --policy-type IDENTITY_POLICY \
     --policy-document file://perm.json --region us-east-1
   ```
   - `"Action": "s3:ListBucket"` sobre `arn:aws:s3:::$BUCKET` → permite listar el contenido de ese bucket, y solo de ese.
   - `"Action": "s3:GetObject"` sobre `arn:aws:s3:::$BUCKET/*` → permite leer los objetos de ese bucket; `/*` significa "todos los objetos dentro".
   - `aws accessanalyzer validate-policy` → IAM Access Analyzer revisa la política contra más de cien comprobaciones de sintaxis, seguridad y buenas prácticas.
   - `--policy-type IDENTITY_POLICY` → indica que es una política de identidad (para un usuario o rol), no de recurso.
   - `--policy-document file://perm.json` → la política a revisar.
   - `--region us-east-1` → región del servicio Access Analyzer.

   Salida esperada: `{"findings": []}`, sin hallazgos. Para ver cómo se ve un error,
   cambia `s3:GetObject` por `s3:GetObjects` en el archivo, repite la validación y
   verás un hallazgo con `"findingType": "ERROR"`, `"issueCode": "INVALID_ACTION"` y un
   mensaje que sugiere la acción correcta. Corrígelo antes de seguir.
4. Asigna la política al rol y simula qué puede hacer:
   ```bash
   aws iam put-role-policy --role-name lab-lectura-s3 \
     --policy-name lectura-bucket --policy-document file://perm.json
   aws iam simulate-principal-policy \
     --policy-source-arn arn:aws:iam::$CUENTA:role/lab-lectura-s3 \
     --action-names s3:GetObject s3:PutObject s3:DeleteObject \
     --resource-arns arn:aws:s3:::$BUCKET/prueba.txt \
     --query 'EvaluationResults[].[EvalActionName,EvalDecision]' --output table
   ```
   - `aws iam put-role-policy` → añade una política en línea (inline) al rol.
   - `--policy-name lectura-bucket` → nombre de la política dentro del rol.
   - `aws iam simulate-principal-policy` → evalúa las políticas de una identidad contra acciones concretas, sin ejecutarlas.
   - `--policy-source-arn arn:aws:iam::$CUENTA:role/lab-lectura-s3` → la identidad cuyas políticas se simulan: el rol.
   - `--action-names s3:GetObject s3:PutObject s3:DeleteObject` → las acciones a probar: leer, escribir y borrar un objeto.
   - `--resource-arns arn:aws:s3:::$BUCKET/prueba.txt` → el recurso sobre el que se prueban.
   - `--query 'EvaluationResults[].[EvalActionName,EvalDecision]'` → de cada resultado, extrae solo la acción y la decisión.
   - `--output table` → lo muestra como tabla.

   Salida esperada: `s3:GetObject` → `allowed`; `s3:PutObject` y `s3:DeleteObject` →
   `implicitDeny` (denegado porque ninguna política lo permite; en IAM todo lo que no se
   permite explícitamente está denegado).
5. Crea un perfil de la CLI que asume el rol y úsalo: una lectura permitida y dos
   operaciones que deben fallar:
   ```bash
   aws configure set role_arn arn:aws:iam::$CUENTA:role/lab-lectura-s3 --profile lab-lectura
   aws configure set source_profile lab --profile lab-lectura
   aws configure set role_session_name investigacion-lab --profile lab-lectura
   aws configure set region us-east-1 --profile lab-lectura
   aws sts get-caller-identity --profile lab-lectura
   aws s3 cp s3://$BUCKET/prueba.txt - --profile lab-lectura
   aws s3 cp prueba.txt s3://$BUCKET/intruso.txt --profile lab-lectura
   aws s3 ls --profile lab-lectura
   ```
   - `aws configure set <clave> <valor> --profile lab-lectura` → escribe un ajuste en el perfil `lab-lectura` de `~/.aws/config` (lo crea si no existe).
   - `role_arn ...` → el rol que asumirá el perfil.
   - `source_profile lab` → el perfil con cuyas credenciales se asume el rol; pon el nombre de tu perfil de la preparación (`default` si usaste `aws configure` o `aws login` sin nombre).
   - `role_session_name investigacion-lab` → nombre de la sesión; aparecerá en CloudTrail y te permitirá encontrarla.
   - `region us-east-1` → región por defecto del perfil.
   - `aws sts get-caller-identity --profile lab-lectura` → muestra la identidad con la que operas a través del perfil.
   - `aws s3 cp s3://$BUCKET/prueba.txt -` → descarga el objeto y, con `-` como destino, lo escribe en pantalla.
   - `aws s3 cp prueba.txt s3://$BUCKET/intruso.txt` → intenta subir un archivo.
   - `aws s3 ls` → intenta listar todos los buckets de la cuenta (acción `s3:ListAllMyBuckets`).

   Salida esperada, en orden: un `Arn` `arn:aws:sts::<cuenta>:assumed-role/lab-lectura-s3/investigacion-lab`;
   el texto `dato de laboratorio`; un error `AccessDenied` al subir; y otro
   `AccessDenied` al listar los buckets. El rol hace lo que necesita y nada más.
6. Espera entre 5 y 15 minutos (CloudTrail tarda en mostrar los eventos) e investiga.
   Primero, ¿quién asumió el rol?
   ```bash
   aws cloudtrail lookup-events --region us-east-1 \
     --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
     --max-items 5 --query 'Events[].CloudTrailEvent' --output json \
     | jq -r '.[] | fromjson | {quien: .userIdentity.arn, que: .eventName,
       cuando: .eventTime, desde: .sourceIPAddress, agente: .userAgent,
       rol: .requestParameters.roleArn, sesion: .requestParameters.roleSessionName}'
   ```
   - `aws cloudtrail lookup-events` → busca en el historial de eventos de CloudTrail (últimos 90 días, solo eventos de administración).
   - `--region us-east-1` → el historial es por región; los eventos de esta práctica están en `us-east-1`.
   - `--lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole` → filtra por nombre de evento; solo admite un atributo por búsqueda (otros: `Username`, `EventSource`, `ResourceName`, `AccessKeyId`, `ReadOnly`).
   - `--max-items 5` → devuelve como mucho los 5 eventos más recientes.
   - `--query 'Events[].CloudTrailEvent'` → extrae de cada evento el registro completo, que viene como texto JSON dentro del JSON.
   - `--output json` → lo entrega como lista JSON para pasarla a `jq`.
   - `jq -r '.[] | fromjson | {...}'` → recorre la lista (`.[]`), convierte cada texto en JSON (`fromjson`) y construye un objeto solo con los campos que importan.
   - `.userIdentity.arn` → quién hizo la llamada; `.eventName` → qué hizo; `.eventTime` → cuándo (UTC); `.sourceIPAddress` → desde qué IP; `.userAgent` → con qué herramienta (aquí verás `aws-cli/2...`).
   - `.requestParameters.roleArn` y `.roleSessionName` → qué rol asumió y con qué nombre de sesión.

   Salida esperada: un evento con `quien` = tu identidad de la preparación, `rol` =
   `arn:aws:iam::<cuenta>:role/lab-lectura-s3`, `sesion` = `investigacion-lab`, tu IP
   pública en `desde` y la hora de tu paso 5.
7. Ahora, ¿qué intentó hacer esa sesión y qué se le denegó?
   ```bash
   aws cloudtrail lookup-events --region us-east-1 \
     --lookup-attributes AttributeKey=EventName,AttributeValue=ListBuckets \
     --max-items 5 --query 'Events[].CloudTrailEvent' --output json \
     | jq -r '.[] | fromjson | {quien: .userIdentity.arn, que: .eventName,
       cuando: .eventTime, desde: .sourceIPAddress, error: .errorCode}'
   ```
   - `AttributeValue=ListBuckets` → busca las llamadas a `ListBuckets`, la que hizo `aws s3 ls` en el paso 5.
   - `error: .errorCode` → añade el código de error; si la llamada se denegó, aparece aquí.

   Salida esperada: un evento con `quien` =
   `arn:aws:sts::<cuenta>:assumed-role/lab-lectura-s3/investigacion-lab` y
   `error` = `AccessDenied`. Fíjate en lo que no aparece: ni el `GetObject` permitido ni
   el `PutObject` denegado del paso 5. Son eventos de datos de S3, y el historial gratuito
   solo guarda eventos de administración; para registrarlos hace falta un trail con
   eventos de datos, que es de pago. Es una limitación real que debes conocer al
   investigar. Repite la búsqueda con `AttributeValue=CreateRole` para ver quién creó el
   rol y cuándo.

### Resultado esperado

Un rol que solo puede listar y leer un bucket, una validación sin hallazgos, una
simulación con `allowed` solo para `GetObject`, dos `AccessDenied` reales, y dos
búsquedas en CloudTrail que responden quién, qué, cuándo y desde dónde.

### Comprueba que lo lograste

- ¿Puedes explicar por qué `PutObject` salió `implicitDeny` sin que escribieras ningún
  `Deny`?
- En el evento de `ListBuckets`, ¿el `quien` es el rol con el nombre de sesión, y no tu
  usuario? Así se distingue en una investigación qué hizo cada sesión.
- ¿Sabes qué tipo de eventos no verás en el historial gratuito de CloudTrail?

### Limpieza

```bash
aws iam delete-role-policy --role-name lab-lectura-s3 --policy-name lectura-bucket
aws iam delete-role --role-name lab-lectura-s3
aws s3 rm s3://$BUCKET/prueba.txt
aws s3 rb s3://$BUCKET
rm -f prueba.txt trust.json perm.json
```

- `aws iam delete-role-policy --role-name lab-lectura-s3 --policy-name lectura-bucket` → borra la política en línea del rol (un rol no se puede borrar mientras tenga políticas).
- `aws iam delete-role --role-name lab-lectura-s3` → borra el rol.
- `aws s3 rm s3://$BUCKET/prueba.txt` y `aws s3 rb s3://$BUCKET` → vacían y borran el bucket.
- `rm -f prueba.txt trust.json perm.json` → borra los archivos locales.

Borra también la sección `[profile lab-lectura]` de `~/.aws/config` con un editor; ya no
apunta a nada. Si en la preparación creaste una clave de acceso de usuario IAM, este es
el momento de eliminarla:

```bash
aws iam list-access-keys --user-name <tu-usuario>
aws iam delete-access-key --user-name <tu-usuario> --access-key-id <AKIA...>
```

- `aws iam list-access-keys --user-name <tu-usuario>` → lista las claves de acceso del usuario con su `AccessKeyId`.
- `aws iam delete-access-key --user-name <tu-usuario> --access-key-id <AKIA...>` → borra la clave indicada. Puedes ejecutarlo con esa misma clave: la orden funciona y, a partir de ahí, la CLI ya no tiene credencial válida (que es lo que quieres).

Revisa Billing al día siguiente.
