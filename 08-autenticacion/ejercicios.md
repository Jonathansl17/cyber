# Ejercicios: Autenticación

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

## Ejercicio 1: Reproduce y genera códigos TOTP

Nodo: MFA & 2FA, sección [TOTP](README.md#totp).

Objetivo: reproducir con `oathtool` el vector de prueba `94287082` de RFC 6238, comprobar que es solo un HMAC del reloj recalculándolo en Python, y después generar un secreto propio, cargarlo en una app autenticadora y verificar que la app y tu terminal dan el mismo código en cada ventana de 30 segundos.

Necesitas: un Linux con `oathtool` y `qrencode` (Debian/Ubuntu: `sudo apt install oathtool qrencode`; Arch: `sudo pacman -S oath-toolkit qrencode`), `python3`, y un teléfono con una app TOTP (Aegis, 2FAS, Google Authenticator o la de tu gestor de contraseñas). El reloj del equipo debe estar sincronizado (`timedatectl` debe decir `System clock synchronized: yes`). Tiempo: 20 minutos.

### Pasos

1. Pasa el secreto del RFC a hexadecimal. El vector usa como secreto el texto ASCII `12345678901234567890`, y `oathtool` espera la clave en hex por defecto.
   ```bash
   printf 12345678901234567890 | od -An -tx1 | tr -d ' \n'; echo
   ```
   - `printf 12345678901234567890` → escribe el texto tal cual, sin salto de línea final (a diferencia de `echo`), para que no se cuele un `\n` en la clave.
   - `|` → pasa la salida de un comando como entrada del siguiente.
   - `od` → "octal dump": muestra bytes en distintos formatos.
   - `-An` → `-A` fija la base de la columna de direcciones; `n` (none) la quita, así solo salen los bytes.
   - `-tx1` → `-t` elige el formato de salida; `x1` es hexadecimal, un byte por grupo.
   - `tr -d ' \n'` → `tr` traduce o borra caracteres; `-d` borra todos los espacios y saltos de línea que pone `od`, dejando una sola cadena hex.
   - `;` → separa comandos: ejecuta el siguiente al terminar el anterior.
   - `echo` → sin argumentos imprime solo un salto de línea, para que el prompt no quede pegado a la salida.

   Salida: `3132333435363738393031323334353637383930` (cada `3x` es un dígito ASCII).
2. Reproduce el vector: TOTP con 8 dígitos en el segundo 59 de la época Unix.
   ```bash
   oathtool --totp -d 8 --now "1970-01-01 00:00:59 UTC" 3132333435363738393031323334353637383930
   ```
   - `oathtool` → genera y valida contraseñas de un solo uso OATH (HOTP y TOTP).
   - `--totp` → usa el modo TOTP, basado en la hora; sin valor usa HMAC-SHA1 (admite `--totp=SHA256` o `SHA512`). Sin esta opción, `oathtool` calcula HOTP por contador.
   - `-d 8` → (`--digits`) número de dígitos del código; los vectores del RFC usan 8, las apps suelen usar 6.
   - `--now "1970-01-01 00:00:59 UTC"` → (`-N`) usa esa fecha como hora actual en lugar del reloj del sistema; aquí, 59 segundos después de la época Unix.
   - `3132...3930` → la clave secreta (KEY), en hexadecimal, que es el formato por defecto.

   Salida esperada: `94287082`.
3. Reproduce otros dos vectores de la misma tabla del RFC para confirmar que no es casualidad.
   ```bash
   oathtool --totp -d 8 --now "2005-03-18 01:58:29 UTC" 3132333435363738393031323334353637383930
   oathtool --totp -d 8 --now "2009-02-13 23:31:30 UTC" 3132333435363738393031323334353637383930
   ```
   - `--totp`, `-d 8`, `--now` y la clave hex → (ver paso 2); solo cambia la fecha pasada a `--now`, que da otro contador `T`.

   Salidas esperadas: `07081804` y `89005924`.
4. Pide a `oathtool` que explique el cálculo del primer vector.
   ```bash
   oathtool --totp -v -d 8 --now "1970-01-01 00:00:59 UTC" 3132333435363738393031323334353637383930
   ```
   - `-v` → (`--verbose`) explica lo que hace: imprime los datos intermedios del cálculo además del código.
   - `--totp`, `-d 8`, `--now` y la clave hex → (ver paso 2).

   Verás el secreto, el tamaño de ventana (`Step size (seconds): 30`), la hora usada y `Counter: 0x1 (1)`: 59 / 30 = 1, redondeado hacia abajo, es la `T` de la fórmula del README.
5. Recalcula el mismo código a mano con Python, siguiendo la fórmula `T`, HMAC-SHA1 y truncado.
   ```bash
   python3 - <<'PY'
   import hmac, hashlib, struct
   secreto = b"12345678901234567890"
   t = 59 // 30
   h = hmac.new(secreto, struct.pack(">Q", t), hashlib.sha1).digest()
   o = h[-1] & 0x0F
   codigo = (struct.unpack(">I", h[o:o + 4])[0] & 0x7FFFFFFF) % 10**8
   print(f"{codigo:08d}")
   PY
   ```
   - `python3 -` → ejecuta Python leyendo el programa desde la entrada estándar (`-`) en vez de un archivo.
   - `<<'PY' ... PY` → heredoc: pasa como entrada todas las líneas hasta la línea `PY`; las comillas en `'PY'` impiden que la shell expanda `$` o comillas dentro del código.
   - `import hmac, hashlib, struct` → módulos para HMAC, funciones hash y empaquetado de enteros en bytes.
   - `secreto = b"12345678901234567890"` → la clave del RFC como bytes.
   - `t = 59 // 30` → contador de ventana: segundos desde la época divididos entre el periodo de 30 s, con división entera (vale 1).
   - `struct.pack(">Q", t)` → convierte `t` en 8 bytes: `>` big-endian (orden de red), `Q` entero sin signo de 64 bits, como pide el RFC.
   - `hmac.new(secreto, ..., hashlib.sha1).digest()` → calcula HMAC-SHA1 del contador con la clave y devuelve los 20 bytes del resultado.
   - `o = h[-1] & 0x0F` → truncado dinámico: los 4 bits bajos del último byte dan el desplazamiento (0 a 15).
   - `struct.unpack(">I", h[o:o + 4])[0]` → lee 4 bytes desde ese desplazamiento como entero sin signo de 32 bits big-endian (`I`).
   - `& 0x7FFFFFFF` → borra el bit más alto para que el número sea positivo.
   - `% 10**8` → se queda con los últimos 8 dígitos decimales.
   - `print(f"{codigo:08d}")` → imprime el código con 8 cifras, rellenando con ceros a la izquierda.

   Salida: `94287082`. No hay red ni servidor de por medio: solo el secreto y la hora.
6. Genera tu propio secreto de 160 bits en base32 (el formato que usan las apps) y guárdalo en una variable, sin que pase por el historial de la shell.
   ```bash
   SECRETO=$(head -c 20 /dev/urandom | base32)
   echo "$SECRETO"
   ```
   - `SECRETO=$(...)` → guarda en la variable `SECRETO` la salida del comando entre `$(` y `)` (sustitución de comandos).
   - `head -c 20` → lee solo los primeros 20 bytes (`-c` cuenta bytes, no líneas): 20 bytes son 160 bits.
   - `/dev/urandom` → fuente de bytes aleatorios del kernel, apta para claves.
   - `base32` → codifica esos bytes en base32 (alfabeto `A-Z` y `2-7`), el formato que esperan las apps TOTP.
   - `echo "$SECRETO"` → imprime el valor; las comillas evitan que la shell lo parta o interprete.

   Saldrán 32 caracteres de `A-Z` y `2-7`, por ejemplo `QC3Y2BARCFMGMAAJKSORTEPRSFF2OGQ7`.
7. Muestra el secreto como código QR en la terminal con una URI `otpauth`, el formato estándar que leen las apps.
   ```bash
   qrencode -t ansiutf8 "otpauth://totp/Lab:jony?secret=$SECRETO&issuer=Lab&algorithm=SHA1&digits=6&period=30"
   ```
   - `qrencode` → genera un código QR con el texto que recibe como argumento.
   - `-t ansiutf8` → (`--type`) tipo de salida: dibuja el QR en la terminal con caracteres UTF-8 y colores ANSI, en lugar del PNG por defecto.
   - `"otpauth://..."` → el texto a codificar, entre comillas para que la shell no interprete los `&`:
   - `otpauth://totp/` → esquema de URI para cuentas OTP, en modo TOTP.
   - `Lab:jony` → etiqueta de la cuenta: emisor y nombre de usuario que mostrará la app.
   - `secret=$SECRETO` → el secreto compartido en base32.
   - `issuer=Lab` → nombre del servicio emisor.
   - `algorithm=SHA1` → función hash del HMAC.
   - `digits=6` → longitud del código.
   - `period=30` → duración de cada ventana en segundos.

   Abre la app autenticadora del teléfono, elige añadir cuenta por QR y escanéalo. Aparecerá una entrada "Lab (jony)".
8. Genera el código actual y el de la siguiente ventana desde la terminal.
   ```bash
   oathtool --totp -b -w 1 "$SECRETO"
   ```
   - `--totp` → (ver paso 2).
   - `-b` → (`--base32`) indica que la clave está en base32 en vez de hexadecimal.
   - `-w 1` → (`--window`) genera 1 código adicional, el de la ventana siguiente.
   - `"$SECRETO"` → la clave, tomada de la variable del paso 6.

   La primera línea debe coincidir con la app; la segunda es el que mostrará la app cuando su contador de 30 s llegue a cero. Espera a ese cambio y compruébalo.
9. Simula un teléfono con el reloj 30 segundos adelantado: el código que calcula es el de la ventana siguiente, el que el servidor solo acepta si tolera una ventana de desfase.
   ```bash
   oathtool --totp -b --now "$(date -u -d '+30 sec' '+%F %T UTC')" "$SECRETO"
   ```
   - `--totp`, `--now` → (ver paso 2); `-b` → (ver paso 8).
   - `date` → muestra o calcula fechas.
   - `-u` → trabaja en UTC.
   - `-d '+30 sec'` → (`--date`) en vez de la hora actual, usa la hora actual más 30 segundos.
   - `'+%F %T UTC'` → formato de salida: `%F` es `AAAA-MM-DD`, `%T` es `HH:MM:SS`, y `UTC` se escribe literal para que `oathtool` sepa la zona.
   - `"$SECRETO"` → (ver paso 8).

   Debe coincidir con la segunda línea del paso 8.
10. Muestra el código y su cuenta atrás cada segundo, para compararlo un par de ciclos con la app. Sal con `Ctrl+C`.
    ```bash
    while true; do printf '\r%s  quedan %2ds ' "$(oathtool --totp -b "$SECRETO")" $((30 - $(date +%s) % 30)); sleep 1; done
    ```
    - `while true; do ...; done` → bucle infinito: repite el cuerpo hasta que lo cortes con `Ctrl+C`.
    - `printf '\r%s  quedan %2ds ' A B` → imprime con formato: `\r` vuelve al inicio de la línea para sobrescribirla, `%s` se sustituye por el código y `%2d` por los segundos, en un campo de 2 caracteres.
    - `"$(oathtool --totp -b "$SECRETO")"` → el código actual (opciones en pasos 2 y 8).
    - `$((...))` → expansión aritmética de la shell.
    - `date +%s` → segundos desde la época Unix.
    - `30 - ... % 30` → `%` es el resto de la división: segundos que faltan para que acabe la ventana de 30 s.
    - `sleep 1` → espera un segundo antes de la siguiente vuelta.

### Resultado esperado

Los tres vectores del RFC reproducidos con `oathtool` (y el primero también con Python), y una cuenta TOTP en tu teléfono cuyos códigos coinciden con los de la terminal en cada ventana de 30 segundos.

### Comprueba que lo lograste

- ¿Qué dos datos necesita la app para calcular el código? Respuesta: el secreto compartido y la hora actual; no necesita internet.
- ¿Por qué el contador del vector `94287082` vale 1? Respuesta: `floor(59 / 30) = 1`.
- Si alguien fotografía tu QR del paso 7, ¿qué obtiene? Respuesta: el secreto; podrá generar tus códigos para siempre. El QR es tan sensible como una contraseña.
- ¿Te protege TOTP de una página de phishing que reenvía el código al sitio real? Respuesta: no; el código vale unos 30 segundos y el proxy lo usa en ese tiempo. Solo FIDO2/passkeys corta ese ataque.

### Limpieza

Borra la entrada "Lab" de la app autenticadora y la variable de la shell: `unset SECRETO`.

## Ejercicio 2: Autentica contra FreeRADIUS y captura el tráfico

Nodo: [RADIUS](README.md#radius).

Objetivo: levantar FreeRADIUS en un contenedor con un usuario propio, obtener un `Access-Accept` y un `Access-Reject` con `radtest`, capturar los paquetes UDP 1812 y comprobar qué campos viajan en claro, cuál va ofuscado y que el shared secret basta para recuperar la contraseña.

Necesitas: un host Linux con Docker (Debian/Ubuntu: `sudo apt install docker.io`; Arch: `sudo pacman -S docker`; después `sudo systemctl start docker`), el cliente `radtest` (Debian/Ubuntu: `sudo apt install freeradius-utils`; Arch: `sudo pacman -S freeradius`), `tcpdump` y `tshark` (Debian/Ubuntu: `sudo apt install tcpdump tshark`; Arch: `sudo pacman -S tcpdump wireshark-cli`). Todo ocurre dentro de tu equipo, entre el host y el contenedor. Si tu usuario no está en el grupo `docker`, antepón `sudo` a los comandos `docker`. Tiempo: 30 minutos.

### Pasos

1. Crea una carpeta de trabajo.
   ```bash
   mkdir -p ~/lab/radius && cd ~/lab/radius
   ```
   - `mkdir` → crea directorios.
   - `-p` → crea también los directorios padre que falten (`~/lab`) y no da error si ya existen.
   - `~/lab/radius` → la ruta a crear; `~` es tu directorio personal.
   - `&&` → ejecuta el segundo comando solo si el primero terminó bien.
   - `cd ~/lab/radius` → entra en esa carpeta.
2. Escribe `clients.conf`, que define qué equipos (NAS) pueden preguntar al servidor y con qué shared secret. El host llega al contenedor desde la red de Docker (172.16.0.0/12), por eso hay dos clientes.
   ```bash
   cat > clients.conf <<'EOF'
   client localhost {
   	ipaddr = 127.0.0.1
   	secret = testing123
   }

   client dockernet {
   	ipaddr = 172.16.0.0/12
   	secret = testing123
   }
   EOF
   ```
   - `cat > clients.conf` → `cat` copia su entrada a la salida, y `>` envía esa salida al archivo `clients.conf` (lo crea o lo sobrescribe).
   - `<<'EOF' ... EOF` → heredoc: la entrada de `cat` son las líneas hasta `EOF`; con comillas, la shell no expande nada dentro.
   - `client localhost { ... }` → define un cliente RADIUS (un NAS autorizado a consultar) con el nombre interno `localhost`.
   - `ipaddr = 127.0.0.1` → dirección de origen desde la que se aceptan peticiones de ese cliente.
   - `secret = testing123` → shared secret que comparten ese cliente y el servidor; sirve para ofuscar User-Password y firmar las respuestas.
   - `client dockernet { ... }` → segundo cliente, con nombre `dockernet`.
   - `ipaddr = 172.16.0.0/12` → acepta como cliente cualquier IP de ese rango (la red de Docker); se admite notación CIDR.
3. Escribe el archivo de usuarios (`authorize`, el antiguo `users`) con una usuaria `ana`. La segunda línea, con tabulador al inicio, es un atributo que el servidor devolverá en el Accept.
   ```bash
   printf 'ana\tCleartext-Password := "Secreto123"\n\tReply-Message := "Bienvenida, ana"\n' > authorize
   cat authorize
   chmod 644 clients.conf authorize
   ```
4. Arranca FreeRADIUS en modo depuración (`-X`), montando tus dos archivos encima de los de la imagen oficial y publicando los puertos UDP.
   ```bash
   docker run --rm -d --name radius-lab -p 1812-1813:1812-1813/udp \
     -v "$PWD/clients.conf:/etc/raddb/clients.conf:ro" \
     -v "$PWD/authorize:/etc/raddb/mods-config/files/authorize:ro" \
     freeradius/freeradius-server:latest -X
   docker logs radius-lab 2>&1 | tail -3
   ```
   La última línea debe ser `Ready to process requests`. Si ves un error de sintaxis, revisa que el archivo `authorize` tenga el tabulador.
5. En otra terminal, deja una captura del puerto de autenticación en todas las interfaces.
   ```bash
   sudo tcpdump -i any -nn -w ~/lab/radius/radius.pcap 'udp port 1812'
   ```
6. En la primera terminal, prueba con la contraseña correcta. Los argumentos son usuario, contraseña, servidor, número de puerto NAS (0) y shared secret.
   ```bash
   radtest ana Secreto123 127.0.0.1 0 testing123
   ```
   Salida esperada (resumida):
   ```
   Sent Access-Request Id 120 from 0.0.0.0:41234 to 127.0.0.1:1812 length 73
   	User-Name = "ana"
   	User-Password = "Secreto123"
   ...
   Received Access-Accept Id 120 from 127.0.0.1:1812 to 127.0.0.1:41234 length 38
   	Reply-Message = "Bienvenida, ana"
   ```
7. Prueba con una contraseña incorrecta.
   ```bash
   radtest ana otraCosa 127.0.0.1 0 testing123
   ```
   Llega `Received Access-Reject` (tras una espera de alrededor de un segundo que el servidor añade a propósito para frenar ataques de fuerza bruta).
8. Mira qué decidió el servidor y por qué en su log de depuración.
   ```bash
   docker logs radius-lab 2>&1 | grep -E 'Sent Access-(Accept|Reject)|pap:'
   ```
   Para la primera petición verás `pap: User authenticated successfully` seguido de `Sent Access-Accept`; para la segunda, un `pap: ERROR: ... password does not match "known good" password` seguido de `Sent Access-Reject`. El módulo `pap` es el que comparó la contraseña recibida con la del archivo `authorize`.
9. Detén tcpdump con `Ctrl+C`. Mira el contenido en hexadecimal y ASCII.
   ```bash
   cd ~/lab/radius && sudo chown "$USER" radius.pcap
   tcpdump -nn -X -r radius.pcap | head -40
   ```
   En la columna ASCII de la derecha se lee `ana` (el User-Name) y, en la respuesta, `Bienvenida, ana`; `Secreto123` no aparece. Con `-i any` cada paquete puede salir dos veces (por `lo` y por la interfaz de Docker): es el mismo paquete antes y después de la traducción de puertos.
10. Decodifica los campos con tshark, sin el secreto.
    ```bash
    tshark -r radius.pcap -Y radius -T fields -e radius.code -e radius.User_Name -e radius.User_Password
    ```
    Código 1 es Access-Request, 2 Access-Accept, 3 Access-Reject. `User_Name` sale como `ana`; `User_Password` sale como bytes ilegibles (16 bytes en hex): está ofuscado con MD5 y el shared secret.
11. Ahora dale a tshark el shared secret y mira el campo de contraseña de los Access-Request.
    ```bash
    tshark -r radius.pcap -o radius.shared_secret:testing123 -Y 'radius.code == 1' -V | grep -i -A1 'User-Password'
    ```
    Junto al valor cifrado aparece la contraseña descifrada: `Secreto123` en el primer intento y `otraCosa` en el segundo. Quien conozca el secreto y capture el tráfico obtiene las contraseñas.

### Resultado esperado

Un `Access-Accept` con su `Reply-Message` y un `Access-Reject`, el log del servidor explicando cada decisión, y una captura `radius.pcap` en la que has comprobado tres cosas: el usuario y los atributos viajan en claro, la contraseña va ofuscada, y con el shared secret se descifra.

### Comprueba que lo lograste

- ¿Qué campo de un Access-Request protege el shared secret? Respuesta: solo User-Password; el resto del paquete va en claro.
- ¿Por qué un shared secret como `testing123` es un problema aunque la red sea interna? Respuesta: quien lo adivine o lo lea de un NAS y capture tráfico descifra todas las contraseñas.
- ¿Qué devuelve el servidor además de "sí" en el Access-Accept, y qué A de AAA es eso? Respuesta: atributos (aquí Reply-Message; en producción VLAN o tiempo de sesión); es autorización.
- ¿Qué dos mitigaciones da el README frente a ataques como Blast-RADIUS? Respuesta: exigir Message-Authenticator en todos los paquetes, o llevar RADIUS dentro de TLS (RadSec, TCP 2083).

### Limpieza

```bash
docker stop radius-lab
docker rmi freeradius/freeradius-server:latest
rm -r ~/lab/radius
```

## Ejercicio 3: Monta OpenLDAP y observa un bind simple en claro

Nodo: [LDAP](README.md#ldap).

Objetivo: levantar un directorio OpenLDAP propio con dos OUs y tres usuarios, autenticarte con un bind simple sin TLS desde otra máquina mientras capturas con Wireshark, y localizar en la captura el DN y la contraseña en texto claro; además, distinguir en la captura un bind correcto de uno fallido.

Necesitas: el laboratorio del [ejercicio 1 de Virtualización](../06-virtualizacion/ejercicios.md#ejercicio-1-rompe-y-restaura-una-vm-con-un-snapshot): VM `victima` con Debian 12 o Ubuntu (será el servidor LDAP) y VM `atacante` con Kali (cliente y Wireshark), en la red interna `labnet`. Antes de pasar la víctima a `labnet`, instala en ella `sudo apt install slapd ldap-utils`; en Kali, si falta, `sudo apt install ldap-utils`. Sustituye `10.10.10.101` por la IP de tu víctima. Referencia: [OpenLDAP Administrator's Guide](https://www.openldap.org/doc/admin26/). Tiempo: 45 minutos.

### Pasos

1. En la víctima, configura la base del directorio. El instalador ya pidió una contraseña de administrador; `dpkg-reconfigure` te deja fijar el dominio.
   ```bash
   sudo dpkg-reconfigure slapd
   ```
   Respuestas: "Omit OpenLDAP server configuration?" → No; "DNS domain name" → `lab.local`; "Organization name" → `Lab`; "Administrator password" → una contraseña que recuerdes (dos veces); "Do you want the database to be removed when slapd is purged?" → No; "Move old database?" → Yes. Con `lab.local` la raíz del árbol es `dc=lab,dc=local` y el administrador es `cn=admin,dc=lab,dc=local`.
2. Comprueba que el servidor responde y escucha en el 389 de todas las interfaces.
   ```bash
   ss -tln | grep ':389'
   ldapsearch -x -H ldap://localhost -b dc=lab,dc=local -LLL dn
   ```
   `ss` muestra `0.0.0.0:389`; `ldapsearch` devuelve `dn: dc=lab,dc=local`. Esa consulta fue anónima (`-x` sin `-D`): anótalo, el directorio deja leer a cualquiera.
3. Crea el archivo de las dos OUs.
   ```bash
   cat > ous.ldif <<'EOF'
   dn: ou=Contabilidad,dc=lab,dc=local
   objectClass: organizationalUnit
   ou: Contabilidad

   dn: ou=TI,dc=lab,dc=local
   objectClass: organizationalUnit
   ou: TI
   EOF
   ```
4. Crea el archivo de los tres usuarios. El heredoc sin comillas ejecuta `slappasswd`, que convierte cada contraseña en un hash `{SSHA}` antes de escribirla (está en `/usr/sbin`, fuera del PATH de un usuario normal).
   ```bash
   cat > usuarios.ldif <<EOF
   dn: uid=ana,ou=Contabilidad,dc=lab,dc=local
   objectClass: inetOrgPerson
   uid: ana
   cn: Ana Rojas
   sn: Rojas
   userPassword: $(/usr/sbin/slappasswd -s 'Ana.Lab.2026')

   dn: uid=luis,ou=Contabilidad,dc=lab,dc=local
   objectClass: inetOrgPerson
   uid: luis
   cn: Luis Mora
   sn: Mora
   userPassword: $(/usr/sbin/slappasswd -s 'Luis.Lab.2026')

   dn: uid=marta,ou=TI,dc=lab,dc=local
   objectClass: inetOrgPerson
   uid: marta
   cn: Marta Solis
   sn: Solis
   userPassword: $(/usr/sbin/slappasswd -s 'Marta.Lab.2026')
   EOF
   grep userPassword usuarios.ldif
   ```
   Cada `userPassword` debe empezar por `{SSHA}`.
5. Carga los dos archivos como administrador (`-W` pide la contraseña del paso 1).
   ```bash
   ldapadd -x -H ldap://localhost -D cn=admin,dc=lab,dc=local -W -f ous.ldif
   ldapadd -x -H ldap://localhost -D cn=admin,dc=lab,dc=local -W -f usuarios.ldif
   ```
   Cada entrada imprime `adding new entry "..."`. Si sale `Invalid credentials (49)`, la contraseña de admin no es la del paso 1.
6. Comprueba el árbol.
   ```bash
   ldapsearch -x -H ldap://localhost -b dc=lab,dc=local -LLL dn
   ```
   Deben salir siete DN: la raíz, las dos OUs y los tres usuarios (más `cn=admin` si tu versión lo crea como entrada).
7. En el atacante, abre Wireshark con permisos de captura y empieza a capturar en la interfaz del laboratorio (`eth0` en Kali) con este filtro de captura en la pantalla de inicio:
   ```
   tcp port 389
   ```
   Doble clic en `eth0` para empezar.
8. En el atacante, en una terminal, autentícate como `ana` con bind simple (`-x`) contra la víctima y busca a todos los usuarios. `-W` te pide la contraseña: escribe `Ana.Lab.2026`.
   ```bash
   ldapsearch -x -H ldap://10.10.10.101 -D uid=ana,ou=Contabilidad,dc=lab,dc=local -W \
     -b dc=lab,dc=local -LLL '(objectClass=inetOrgPerson)' cn uid
   ```
   Devuelve las tres personas con su `cn` y `uid`.
9. Repite con una contraseña equivocada (escribe cualquier otra cosa cuando la pida).
   ```bash
   ldapsearch -x -H ldap://10.10.10.101 -D uid=ana,ou=Contabilidad,dc=lab,dc=local -W -b dc=lab,dc=local -LLL dn
   ```
   Responde `ldap_bind: Invalid credentials (49)`.
10. Detén la captura en Wireshark (cuadrado rojo) y aplica el filtro de visualización de las peticiones de bind:
    ```
    ldap.protocolOp == 0
    ```
    Salen dos paquetes `bindRequest(1) "uid=ana,ou=Contabilidad,dc=lab,dc=local" simple`. En el detalle, abre Lightweight Directory Access Protocol → LDAPMessage bindRequest → authentication: simple: ahí está `simple: Ana.Lab.2026` en el primero y la contraseña errónea en el segundo, en texto claro.
11. Mira las respuestas con el filtro:
    ```
    ldap.protocolOp == 1
    ```
    Son los `bindResponse`: el primero con `resultCode: success (0)` y el segundo con `resultCode: invalidCredentials (49)`. Un IDS o un SIEM que vea muchos 49 seguidos del mismo origen está viendo fuerza bruta.
12. Guarda la captura (File → Save As → `ldap.pcapng`) y extrae lo mismo desde la terminal:
    ```bash
    tshark -r ldap.pcapng -Y 'ldap.protocolOp == 0' -T fields -e ip.src -e ldap.name -e ldap.simple
    ```
    Dos líneas con la IP del atacante, el DN y cada contraseña tal como se tecleó.

### Resultado esperado

Un directorio `dc=lab,dc=local` con `ou=Contabilidad` (ana, luis) y `ou=TI` (marta), y una captura `ldap.pcapng` donde se leen el DN y la contraseña de dos binds simples, uno con `success` y otro con `invalidCredentials (49)`.

### Comprueba que lo lograste

- ¿Qué operación LDAP es la autenticación y en qué paquete viaja la contraseña? Respuesta: el Bind; va en el campo `simple` del `bindRequest`.
- La contraseña está guardada como `{SSHA}` en el servidor. ¿Por qué igualmente se ve en claro en la red? Respuesta: el hash protege el almacenamiento, no el transporte; el bind simple manda la contraseña tal cual y el servidor la compara con el hash.
- ¿Qué tres opciones evitan que se vea? Respuesta: StartTLS en el 389 (`ldapsearch -ZZ`), LDAPS en el 636, o un bind SASL (por ejemplo Kerberos/GSSAPI) que no manda la contraseña.
- ¿Qué hallazgo de configuración apareció en el paso 2? Respuesta: el bind anónimo puede leer el árbol; en un directorio real debe desactivarse o limitarse.

### Limpieza

En la víctima, si no vas a reutilizar el directorio: `sudo apt purge slapd` y `sudo rm -rf /var/lib/ldap`. O restaura el snapshot `limpio` si lo tomaste antes de instalar slapd.

## Ejercicio 4: Exige certificado de cliente en nginx con mTLS

Nodo: [Certificates](README.md#certificates) y [mTLS](README.md#mtls).

Objetivo: crear con `openssl` una CA propia, emitir con ella un certificado de servidor y uno de cliente, configurar nginx con `ssl_verify_client on` y demostrar con `curl` tres casos: sin certificado de cliente se rechaza, con el certificado de tu CA se acepta y nginx lee tu identidad, y con un certificado de otra CA se rechaza.

Necesitas: un Linux con `openssl` y `curl` (vienen en casi todas las distribuciones) y Docker para nginx (Debian/Ubuntu: `sudo apt install docker.io`; Arch: `sudo pacman -S docker`; después `sudo systemctl start docker`). Todo en tu equipo, en `localhost`. Tiempo: 30 minutos.

### Pasos

1. Crea la carpeta de trabajo.
   ```bash
   mkdir -p ~/lab/mtls/certs && cd ~/lab/mtls/certs
   ```
2. Crea la CA: una clave RSA 3072 y un certificado autofirmado de un año marcado como CA.
   ```bash
   openssl req -x509 -newkey rsa:3072 -sha256 -days 365 -nodes \
     -keyout ca.key -out ca.crt -subj "/CN=CA Lab" \
     -addext "basicConstraints=critical,CA:TRUE" \
     -addext "keyUsage=critical,keyCertSign,cRLSign"
   ```
3. Crea la clave y la petición (CSR) del servidor, y fírmala con la CA añadiendo el SAN `localhost` (los clientes solo miran el SAN) y el uso `serverAuth`.
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout servidor.key -out servidor.csr -subj "/CN=localhost"
   cat > servidor.ext <<'EOF'
   basicConstraints=CA:FALSE
   keyUsage=critical,digitalSignature,keyEncipherment
   extendedKeyUsage=serverAuth
   subjectAltName=DNS:localhost,IP:127.0.0.1
   EOF
   openssl x509 -req -in servidor.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
     -days 30 -sha256 -extfile servidor.ext -out servidor.crt
   ```
   Imprime `Certificate request self-signature ok` y `subject=CN=localhost`.
4. Lo mismo para el cliente, con el uso `clientAuth`. Su CN será la identidad que verá nginx.
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout cliente.key -out cliente.csr -subj "/CN=cliente-ana"
   cat > cliente.ext <<'EOF'
   basicConstraints=CA:FALSE
   keyUsage=critical,digitalSignature
   extendedKeyUsage=clientAuth
   EOF
   openssl x509 -req -in cliente.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
     -days 30 -sha256 -extfile cliente.ext -out cliente.crt
   ```
5. Verifica la cadena y mira los campos del certificado de cliente.
   ```bash
   openssl verify -CAfile ca.crt servidor.crt cliente.crt
   openssl x509 -in cliente.crt -noout -subject -issuer -dates -ext extendedKeyUsage
   ```
   Salida: `servidor.crt: OK`, `cliente.crt: OK`; `subject=CN=cliente-ana`, `issuer=CN=CA Lab`, las fechas de validez y `TLS Web Client Authentication`.
6. Crea un certificado de cliente de una CA ajena, para el caso negativo: una segunda CA y un cliente firmado por ella.
   ```bash
   openssl req -x509 -newkey rsa:2048 -sha256 -days 30 -nodes -keyout otra-ca.key -out otra-ca.crt \
     -subj "/CN=Otra CA" -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign"
   openssl req -newkey rsa:2048 -nodes -keyout intruso.key -out intruso.csr -subj "/CN=intruso"
   openssl x509 -req -in intruso.csr -CA otra-ca.crt -CAkey otra-ca.key -CAcreateserial \
     -days 30 -sha256 -extfile cliente.ext -out intruso.crt
   chmod 644 *.crt *.key
   ```
   El `chmod` deja que el nginx del contenedor lea los archivos; en un servidor real las claves serían `600` y propiedad del usuario del servicio.
7. Escribe la configuración completa de nginx. `ssl_client_certificate` es la CA en la que confía para los clientes, `ssl_verify_client on` hace obligatorio el certificado y la respuesta devuelve el DN del cliente que nginx validó.
   ```bash
   cat > ~/lab/mtls/default.conf <<'EOF'
   server {
       listen 443 ssl;
       server_name localhost;

       ssl_certificate         /etc/nginx/certs/servidor.crt;
       ssl_certificate_key     /etc/nginx/certs/servidor.key;
       ssl_protocols           TLSv1.2 TLSv1.3;

       ssl_client_certificate  /etc/nginx/certs/ca.crt;
       ssl_verify_client       on;
       ssl_verify_depth        1;

       location / {
           default_type text/plain;
           return 200 "hola $ssl_client_s_dn (verificado: $ssl_client_verify)\n";
       }
   }
   EOF
   ```
8. Arranca nginx con esa configuración y los certificados, publicando el 443 del contenedor en el 8443 de tu equipo.
   ```bash
   docker run --rm -d --name nginx-mtls -p 127.0.0.1:8443:443 \
     -v ~/lab/mtls/default.conf:/etc/nginx/conf.d/default.conf:ro \
     -v ~/lab/mtls/certs:/etc/nginx/certs:ro \
     nginx:stable
   docker logs nginx-mtls 2>&1 | tail -3
   ```
   No debe haber líneas `[emerg]`. Si las hay, el mensaje dice qué directiva o archivo falla.
9. Caso 1, sin certificado de cliente. `--cacert` hace que curl confíe en tu CA para validar al servidor.
   ```bash
   curl -sS --cacert ~/lab/mtls/certs/ca.crt https://localhost:8443/
   ```
   nginx completa el TLS pero responde `400 Bad Request` con el texto `No required SSL certificate was sent`.
10. Caso 2, con el certificado y la clave de cliente.
    ```bash
    curl -sS --cacert ~/lab/mtls/certs/ca.crt \
      --cert ~/lab/mtls/certs/cliente.crt --key ~/lab/mtls/certs/cliente.key \
      https://localhost:8443/
    ```
    Respuesta: `hola CN=cliente-ana (verificado: SUCCESS)`. El servidor sabe quién eres sin contraseña.
11. Caso 3, con el certificado de la CA ajena.
    ```bash
    curl -sS --cacert ~/lab/mtls/certs/ca.crt \
      --cert ~/lab/mtls/certs/intruso.crt --key ~/lab/mtls/certs/intruso.key \
      https://localhost:8443/
    ```
    Respuesta: `400 Bad Request` con `The SSL certificate error`. En el log de nginx (`docker logs nginx-mtls 2>&1 | tail -2`) aparece el motivo, del estilo `client SSL certificate verify error: (20:unable to get local issuer certificate)`.
12. Mira el intercambio TLS del caso 2 con detalle para ver la petición del servidor y el envío del cliente.
    ```bash
    curl -v -o /dev/null --cacert ~/lab/mtls/certs/ca.crt \
      --cert ~/lab/mtls/certs/cliente.crt --key ~/lab/mtls/certs/cliente.key \
      https://localhost:8443/ 2>&1 | grep -E 'TLS|SSL connection'
    ```
    Busca tres líneas: `(IN), TLS handshake, Request CERT (13)` es el `CertificateRequest` del servidor; `(OUT), TLS handshake, Certificate (11)` es tu certificado (con OpenSSL reciente puede salir como `Unknown (25)`, que es el mismo certificado comprimido); y `(OUT), TLS handshake, CERT verify (15)` es la firma con tu clave privada. Son los mensajes de mTLS que describe el README. Los `(IN) Certificate` y `(IN) CERT verify` anteriores son los del servidor.

### Resultado esperado

Una carpeta `~/lab/mtls` con tu CA, los certificados de servidor, cliente e intruso y la configuración de nginx, y tres respuestas de curl: 400 sin certificado, 200 con `CN=cliente-ana` usando tu certificado, y 400 con el certificado de otra CA.

### Comprueba que lo lograste

- ¿Qué directiva dice a nginx en qué CA confiar para los clientes? Respuesta: `ssl_client_certificate`.
- ¿Por qué el certificado `intruso.crt` se rechaza si es técnicamente válido? Respuesta: lo firmó una CA que no está en `ssl_client_certificate`, así que la cadena no llega a una raíz de confianza.
- ¿Qué prueba el cliente además de enviar su certificado? Respuesta: que posee la clave privada, firmando el handshake (`CertificateVerify`); por eso sin `--key` no funciona.
- ¿De dónde saca nginx la identidad que podría usar para autorizar? Respuesta: del subject o el SAN del certificado validado (`$ssl_client_s_dn`).

### Limpieza

```bash
docker stop nginx-mtls
docker rmi nginx:stable
rm -r ~/lab/mtls
```
