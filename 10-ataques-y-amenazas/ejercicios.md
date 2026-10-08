# Ejercicios: Ataques y amenazas

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se hace sobre equipos, cuentas y logs tuyos. Los intentos fallidos se generan contra tu
propia máquina o VM, nunca contra equipos de terceros.

## Ejercicio 1: Distingue fuerza bruta de password spray en tus logs

Nodo: Brute Force vs Password Spray ([README](README.md#brute-force-vs-password-spray)).

Objetivo: a partir de los logs de autenticación de tu propio equipo o VM, contar fallos por
cuenta y cuentas distintas por IP de origen, y decidir con esos dos números si un patrón se
parece más a fuerza bruta (muchos fallos en una cuenta) o a password spray (un fallo en cada
una de muchas cuentas desde el mismo origen).

Necesitas: una VM o equipo Linux con `sshd` corriendo y acceso a `journalctl` (con `sudo` o
siendo miembro del grupo `systemd-journal`). 30 minutos.

### Pasos

1. Genera unos pocos fallos de autenticación contra tu propia máquina, para tener datos
   reproducibles. Simula primero un patrón de fuerza bruta: varios intentos fallidos contra una
   sola cuenta. Responde con una contraseña incorrecta cada vez.
   ```bash
   for i in 1 2 3 4 5; do ssh -4 ana@localhost -o PreferredAuthentications=password \
     -o PubkeyAuthentication=no -o NumberOfPasswordPrompts=1 true; done
   ```
   - `for i in 1 2 3 4 5; do ...; done` → bucle de la shell que repite el comando cinco veces; `i` solo cuenta las vueltas.
   - `\` → al final de la línea, continúa el comando en la línea siguiente.
   - `ssh` → cliente SSH que abre una sesión en otra máquina (aquí, la tuya).
   - `-4` → fuerza IPv4. Sin esto `localhost` suele resolver a `::1` (IPv6) y el origen que registra sshd es `::1`, que la expresión regular IPv4 del paso 3 no captura; con `-4` el origen es `127.0.0.1`.
   - `ana@localhost` → usuario `ana` en el equipo `localhost` (127.0.0.1, tu propia máquina).
   - `-o` → pasa una opción de configuración del cliente (las mismas de `ssh_config`) solo para esta conexión.
   - `PreferredAuthentications=password` → orden de métodos de autenticación a probar; aquí solo contraseña.
   - `PubkeyAuthentication=no` → no intenta autenticarse con clave pública, para que el fallo sea de contraseña.
   - `NumberOfPasswordPrompts=1` → pide la contraseña una sola vez antes de rendirse (por defecto son 3), así cada vuelta deja un fallo.
   - `true` → comando remoto que se ejecutaría si el login funcionara; no hace nada y termina.

   (La cuenta `ana` puede no existir o puedes usar una tuya; lo que importa es que falle y quede
   registrado.)

2. Simula ahora un patrón de spray: un intento fallido contra varias cuentas distintas.
   ```bash
   for u in ana bruno carla diego elena; do ssh -4 "$u@localhost" \
     -o PreferredAuthentications=password -o PubkeyAuthentication=no \
     -o NumberOfPasswordPrompts=1 true; done
   ```
   - `for u in ana bruno carla diego elena; do ...; done` → bucle que recorre los cinco nombres y guarda cada uno en la variable `u`.
   - `"$u@localhost"` → usuario de la vuelta actual en tu propia máquina; las comillas evitan que la shell parta el valor.
   - `ssh`, `-4`, `\`, `-o PreferredAuthentications=password`, `-o PubkeyAuthentication=no`, `-o NumberOfPasswordPrompts=1`, `true` → (ver paso 1).

   > [!NOTE]
   > OpenSSH 9.8 y posteriores traen `PerSourcePenalties` activado por defecto: tras varios
   > fallos seguidos desde la misma IP, el servidor penaliza ese origen y rechaza o retrasa los
   > intentos siguientes (verás `banner exchange ... Connection closed`). Es justo una defensa
   > contra lo que estás simulando. Si los intentos empiezan a cortarse, espera unos minutos, o
   > baja el ritmo, o en una VM de laboratorio desactívalo temporalmente con
   > `PerSourcePenalties no` en `/etc/ssh/sshd_config` y `sudo systemctl restart sshd`. En equipos
   > reales se deja activado.

3. Saca de los logs los pares "usuario origen" de cada fallo.
   ```bash
   sudo journalctl -u ssh -u sshd --no-pager 2>/dev/null | \
     sed -nE 's/.*Failed password for (invalid user )?([^ ]+) from ([0-9a-fA-F.:]+).*/\2 \3/p' > ~/fallos.txt
   wc -l ~/fallos.txt
   ```
   - `sudo` → leer los mensajes de autenticación del journal requiere privilegios: hazlo con `sudo` o siendo miembro del grupo `systemd-journal` (en ese caso puedes quitar el `sudo`).
   - `journalctl` → lee el journal de systemd, donde quedan los logs del sistema.
   - `-u ssh -u sshd` → muestra los mensajes de esa unidad; la unidad es `ssh` en Debian/Ubuntu y `sshd` en Arch, así que se piden las dos y la que no exista simplemente no aporta líneas. `2>/dev/null` descarta el aviso por la unidad ausente.
   - `--no-pager` → vuelca la salida entera sin abrir el paginador (`less`), necesario para pasarla a otro comando.
   - `|` → tubería: la salida del comando de la izquierda entra como entrada del de la derecha.
   - `sed` → editor de flujo que transforma el texto línea a línea.
   - `-n` → no imprime nada salvo lo que se pida explícitamente con `p`.
   - `-E` → usa expresiones regulares extendidas (paréntesis y `?` sin escapar).
   - `s/patrón/reemplazo/p` → sustituye lo que casa con el patrón por el reemplazo; la `p` final imprime la línea solo si hubo sustitución, así quedan solo las líneas de fallo.
   - `.*Failed password for` → casa cualquier texto hasta el mensaje de contraseña fallida de `sshd`.
   - `(invalid user )?` → grupo 1, opcional: aparece cuando el usuario no existe en el sistema.
   - `([^ ]+)` → grupo 2: el nombre de usuario (caracteres hasta el siguiente espacio).
   - `from ([0-9a-fA-F.:]+)` → grupo 3: la IP de origen; acepta IPv4 (`127.0.0.1`) e IPv6 (`::1`), por si algún intento entró sin `-4`.
   - `.*` → el resto de la línea (puerto, protocolo), que se descarta.
   - `\2 \3` → reemplazo: deja solo "usuario IP" separados por un espacio.
   - `> ~/fallos.txt` → redirige la salida al archivo `fallos.txt` de tu carpeta personal, sobrescribiéndolo.
   - `wc` → cuenta líneas, palabras o bytes de un archivo.
   - `-l` → cuenta solo líneas: el número de fallos extraídos.
   - `~/fallos.txt` → archivo que se cuenta.

4. Cuenta los fallos por cuenta. Si una cuenta acumula muchos fallos, huele a fuerza bruta.
   ```bash
   awk '{print $1}' ~/fallos.txt | sort | uniq -c | sort -rn
   ```
   - `awk '{print $1}' ~/fallos.txt` → `awk` procesa el archivo por campos separados por espacios; `{print $1}` imprime solo el primer campo de cada línea, el usuario.
   - `|` → (ver paso 3).
   - `sort` → ordena las líneas alfabéticamente, para que los usuarios repetidos queden juntos.
   - `uniq` → colapsa líneas consecutivas iguales en una sola.
   - `-c` → antepone a cada línea cuántas veces se repetía: fallos por cuenta.
   - `sort -rn` → ordena de nuevo la salida: `-n` compara como número (el recuento) y `-r` invierte el orden, de mayor a menor.

   Esperado del paso 1: la cuenta atacada (p. ej. `ana`) con 5 fallos destacando sobre el resto.

5. Cuenta cuántas cuentas distintas falla cada IP de origen. Si una IP toca muchas cuentas con
   uno o pocos fallos cada una, huele a spray.
   ```bash
   awk '{print $2, $1}' ~/fallos.txt | sort -u | awk '{print $1}' | sort | uniq -c | sort -rn
   ```
   - `awk '{print $2, $1}' ~/fallos.txt` → invierte los campos y escribe "IP usuario"; la coma de `print` los separa con un espacio.
   - `sort -u` → ordena y `-u` deja una sola copia de cada línea repetida: cada par IP-usuario cuenta una vez, aunque fallara varias veces.
   - `awk '{print $1}'` → (ver paso 4); aquí el primer campo es la IP.
   - `sort | uniq -c | sort -rn` → (ver paso 4); ahora el recuento es de cuentas distintas por IP.

   Esperado del paso 2: `127.0.0.1` (o tu IP) con 5 cuentas distintas.

### Resultado esperado

Dos recuentos sacados de tus propios logs: uno que muestra una cuenta con muchos fallos (fuerza
bruta) y otro que muestra una IP tocando muchas cuentas distintas (spray). Con eso puedes
etiquetar cada patrón como dice la nota.

### Comprueba que lo lograste

- ¿Cuál es la señal de la fuerza bruta y cuál la del spray? Fuerza bruta: número alto de fallos
  por cuenta. Spray: número alto de cuentas distintas falladas desde un mismo origen, aunque
  cada cuenta tenga un solo fallo.
- ¿Por qué una alerta que solo cuenta fallos por cuenta no ve nunca un spray? Porque en el spray
  cada cuenta falla una sola vez; hay que contar por origen y por número de cuentas distintas.
- ¿Qué defensa deja inútil la contraseña adivinada en ambos casos? MFA; además, bloqueo por
  cuenta contra la fuerza bruta y detección por origen contra el spray.

### Limpieza

```bash
rm -f ~/fallos.txt
```
- `rm` → borra archivos.
- `-f` → no pide confirmación y no da error si el archivo ya no existe.
- `~/fallos.txt` → el archivo de trabajo creado en el paso 3.

## Ejercicio 2: Triage de un correo de phishing

Nodo: Phishing ([README](README.md#phishing)).

Objetivo: analizar un correo de phishing de muestra como lo haría un analista de SOC: leer la
cadena `Received`, comparar `Return-Path`, `From` y `Reply-To`, interpretar el
`Authentication-Results` (SPF, DKIM, DMARC), comprobar tú mismo los registros DNS del dominio
suplantado, extraer y "defangear" las URLs, calcular el hash del adjunto y emitir un veredicto
con una plantilla de informe.

Necesitas: una máquina Linux tuya. `dig` (paquete `dnsutils` en Debian/Ubuntu, `bind` en Arch),
`sha256sum` (viene en coreutils) y `sed`/`grep`. Conexión a internet solo para las consultas
DNS (no se envía nada a nadie). 40 minutos. El correo es una muestra inventada con dominios
reservados por RFC 2606 (`.example`, `.invalid`), así que no apunta a nadie real.

- Debian/Ubuntu: `sudo apt install dnsutils`
- Arch: `sudo pacman -S bind`

### Pasos

1. Guarda el correo de muestra tal cual (es un `.eml`: cabeceras, una línea en blanco y el
   cuerpo). Es un phishing realista pero falso.
   ```bash
   mkdir -p ~/phish-lab && cd ~/phish-lab
   cat > muestra.eml <<'EOF'
   Return-Path: <bounce@mail.account-verify.invalid>
   Received: from mail.account-verify.invalid (unknown [203.0.113.77])
       by mx.example.com (Postfix) with ESMTP id 4A1F2C0A3
       for <cliente@example.com>; Mon, 06 Oct 2025 03:14:52 +0000 (UTC)
   Authentication-Results: mx.example.com;
       spf=fail (sender IP is 203.0.113.77) smtp.mailfrom=mail.account-verify.invalid;
       dkim=none;
       dmarc=fail (p=reject) header.from=bancoejemplo.example
   From: "Seguridad BancoEjemplo" <seguridad@bancoejemplo.example>
   Reply-To: <soporte@account-verify.invalid>
   To: <cliente@example.com>
   Subject: [Accion requerida] Verifique su cuenta en 24h o sera suspendida
   Date: Mon, 06 Oct 2025 03:14:50 +0000
   MIME-Version: 1.0
   Content-Type: text/plain; charset=UTF-8

   Estimado cliente:

   Detectamos un acceso inusual. Verifique su identidad antes de 24 horas
   o su cuenta sera suspendida:

       http://bancoejemplo.example.account-verify.invalid/login?id=8812

   Adjuntamos el formulario de verificacion.

   Atentamente,
   Departamento de Seguridad
   EOF
   # adjunto simulado (un binario cualquiera hace de "formulario")
   printf 'formulario-falso-de-laboratorio' > verificacion.html
   ```
   - `mkdir -p ~/phish-lab && cd ~/phish-lab` → crea y entra en el directorio de trabajo.
   - `cat > muestra.eml <<'EOF' ... EOF` → guarda el correo completo; las comillas en `'EOF'` impiden que la shell toque `$` o comillas del texto.
   - `printf '...' > verificacion.html` → crea un archivo que hace de adjunto para poder hashearlo en el paso 6.

2. Lee la cadena `Received` de abajo hacia arriba: el salto de más abajo es el primero (el que
   originó el correo). Fíjate en la IP y el nombre de quien lo entregó.
   ```bash
   grep -i '^Received:' muestra.eml
   ```
   - `grep -i '^Received:'` → muestra las líneas de recepción; `-i` ignora mayúsculas y `^` ancla al inicio de línea. Aquí el correo llegó desde `mail.account-verify.invalid` (IP `203.0.113.77`), no desde la infraestructura del banco.

3. Compara las tres direcciones que deberían ser coherentes en un correo legítimo:
   `Return-Path` (a dónde van los rebotes), `From` (quién dice ser) y `Reply-To` (a dónde va tu
   respuesta).
   ```bash
   grep -iE '^(Return-Path|From|Reply-To):' muestra.eml
   ```
   - `grep -iE '^(Return-Path|From|Reply-To):'` → `-E` activa la alternancia `( | )`. El `From` dice `bancoejemplo.example`, pero el `Return-Path` y el `Reply-To` apuntan a `account-verify.invalid`: tu respuesta no iría al banco, sino al atacante. Esa discordancia es una señal fuerte.

4. Interpreta el `Authentication-Results`, donde el servidor de correo anotó el resultado de las
   tres comprobaciones de autenticidad.
   ```bash
   grep -iA3 '^Authentication-Results:' muestra.eml
   ```
   - `grep -iA3 '^Authentication-Results:'` → muestra esa línea y las 3 siguientes (`-A3`). Lee:
     `spf=fail` (el servidor que envió no está autorizado por el dominio del `mailfrom`),
     `dkim=none` (el correo no va firmado) y `dmarc=fail (p=reject)` (la política del dominio
     `bancoejemplo.example` pide **rechazar** lo que no autentique). Tres fallos a la vez: casi
     con seguridad suplantación.

5. Verifica por tu cuenta los registros DNS del dominio que el correo dice ser (`From`). Un
   dominio serio publica SPF y DMARC; compáralos con lo que dijo la cabecera.
   ```bash
   dig +short TXT bancoejemplo.example            # SPF: busca una cadena "v=spf1 ..."
   dig +short TXT _dmarc.bancoejemplo.example     # DMARC: busca "v=DMARC1; p=..."
   ```
   - `dig +short TXT <dominio>` → consulta los registros TXT y muestra solo el valor. El SPF es el que empieza por `v=spf1` y lista los servidores autorizados a enviar; si la IP del `Received` no está, el correo es sospechoso.
   - `dig +short TXT _dmarc.<dominio>` → el registro DMARC vive en el subdominio `_dmarc`; su `p=` (none/quarantine/reject) dice qué hacer con lo que falla. Nota: `bancoejemplo.example` es inventado y no resuelve; repite estos dos comandos sustituyéndolo por el dominio real que un correo diga suplantar (por ejemplo el de tu banco) para ver cómo se ve un SPF y un DMARC publicados de verdad.

6. Extrae las URLs del cuerpo y "defangéalas" para poder anotarlas en un informe sin riesgo de
   que alguien haga clic. Calcula también el hash del adjunto.
   ```bash
   grep -oE 'https?://[^ ]+' muestra.eml | sed -E 's#^http#hxxp#; s#\.#[.]#g'
   sha256sum verificacion.html
   ```
   - `grep -oE 'https?://[^ ]+'` → `-o` imprime solo lo que casa (la URL, no la línea entera); el patrón acepta `http` o `https` seguido de caracteres hasta el primer espacio.
   - `sed -E 's#^http#hxxp#; s#\.#[.]#g'` → "defang": cambia `http` por `hxxp` y cada `.` por `[.]`, para que la URL no sea clicable ni resoluble por accidente. Resultado: `hxxp://bancoejemplo[.]example[.]account-verify[.]invalid/login?id=8812`. El dominio real es el que va justo antes del primer `/`: el dueño es `account-verify.invalid`, y `bancoejemplo.example` es solo un subdominio suyo para engañar.
   - `sha256sum verificacion.html` → huella del adjunto; es lo que se busca en VirusTotal o en el EDR sin tener que abrirlo. Nunca abras el adjunto: solo lo identificas por su hash.

### Resultado esperado

Un análisis escrito del correo con: la IP de origen de la cadena `Received`, la discordancia
entre `From`, `Return-Path` y `Reply-To`, el triple fallo SPF/DKIM/DMARC, la URL defangeada con
su dominio real identificado, y el hash del adjunto. Con todo eso, un veredicto: es phishing.

Plantilla de informe (rellénala con tus hallazgos):

```
Asunto analizado:  [Accion requerida] Verifique su cuenta en 24h ...
Remitente (From):  seguridad@bancoejemplo.example
Return-Path:       bounce@mail.account-verify.invalid   (no coincide con From)
Reply-To:          soporte@account-verify.invalid       (no coincide con From)
IP de origen:      203.0.113.77  (mail.account-verify.invalid)
SPF / DKIM / DMARC: fail / none / fail (p=reject)
URL (defang):      hxxp://bancoejemplo[.]example[.]account-verify[.]invalid/login?id=8812
Dominio real URL:  account-verify.invalid
Hash adjunto:      <sha256 de verificacion.html>
Veredicto:         PHISHING (suplantacion de BancoEjemplo)
Accion recomendada: bloquear dominio account-verify.invalid y la IP 203.0.113.77,
                    borrar el correo de todos los buzones, avisar al usuario, no hacer clic.
```

### Comprueba que lo lograste

- ¿Qué tres direcciones comparaste y qué delató la suplantación? `From`, `Return-Path` y
  `Reply-To`: el `From` dice el banco pero las otras dos apuntan a `account-verify.invalid`.
- ¿Por qué `dmarc=fail (p=reject)` es tan concluyente? Porque el propio dominio suplantado pide
  que se rechace cualquier correo que no supere la autenticación; el mensaje no debería haberse
  entregado, y que llegara indica un filtro mal configurado o evadido.
- ¿Cuál es el dominio real de la URL y cómo lo supiste? `account-verify.invalid`: es la parte
  de registro que va justo antes de la primera barra; `bancoejemplo.example` es solo un
  subdominio puesto delante para engañar al ojo.
- ¿Por qué se hashea el adjunto en vez de abrirlo? Para identificarlo y buscarlo en fuentes de
  inteligencia sin ejecutarlo y sin arriesgarse a infectar el equipo de análisis.

### Limpieza

```bash
rm -rf ~/phish-lab
```
- `rm -rf` → borra el directorio del laboratorio con la muestra y el adjunto.
