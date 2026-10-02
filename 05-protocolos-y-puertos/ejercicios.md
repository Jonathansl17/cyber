# Ejercicios: Protocolos y puertos

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

## Ejercicio 1: Captura un handshake TCP y TLS

Nodo: Understand Handshakes ([TCP 3-way handshake](README.md#tcp-3-way-handshake), [Cierre con FIN](README.md#cierre-con-fin), [Handshake TLS 1.2 vs TLS 1.3](README.md#handshake-tls-12-vs-tls-13)).

Objetivo: grabar una conexión HTTPS completa de tu propio equipo y señalar en la captura los tres segmentos del 3-way handshake, el ClientHello con su SNI, la versión de TLS que eligió el servidor y los segmentos FIN del cierre. Al final sabrás distinguir con datos el handshake TCP del TLS y ver qué cambia entre TLS 1.2 y 1.3.

Necesitas: un Linux con internet (tu equipo o una VM en modo NAT); `tcpdump`, `curl` y Wireshark (que trae `tshark`). Instalación en Debian/Ubuntu: `sudo apt install tcpdump curl wireshark`; en Arch: `sudo pacman -S tcpdump curl wireshark-qt`. Solo capturas tu propio tráfico hacia un sitio público de ejemplo (`example.com`). Tiempo: 30-40 minutos.

### Pasos

1. Averigua por qué interfaz sale tu tráfico a internet y guárdala en una variable.
   ```bash
   IF=$(ip route get 1.1.1.1 | awk '{for(i=1;i<NF;i++) if($i=="dev") print $(i+1)}')
   echo "$IF"
   ```
   - `IF=$(...)` → ejecuta lo que hay dentro de `$( )` y guarda su salida en la variable de shell `IF`.
   - `ip route get 1.1.1.1` → pregunta al kernel qué ruta usaría para llegar a `1.1.1.1` (un destino cualquiera de internet); la respuesta incluye `dev <interfaz>`.
   - `|` → pasa la salida de `ip` como entrada de `awk`.
   - `awk '{...}'` → procesa cada línea, dividida en campos por espacios (`$1`, `$2`...; `NF` es el número de campos).
   - `for(i=1;i<NF;i++) if($i=="dev") print $(i+1)` → recorre los campos y, cuando uno vale `dev`, imprime el siguiente, que es el nombre de la interfaz.
   - `echo "$IF"` → muestra el valor de la variable; las comillas evitan que la shell la parta si tuviera espacios.

   Verás el nombre de tu interfaz cableada o Wi-Fi (`eth0`, `enp3s0`, `wlan0`, `wlp2s0`...).
2. Resuelve una sola IP de `example.com` y guárdala. Así la captura solo contendrá esta conexión y no el resto de tu tráfico HTTPS.
   ```bash
   IP=$(getent ahostsv4 example.com | awk 'NR==1{print $1}')
   echo "$IP"
   ```
   - `getent` → consulta las bases de datos del sistema (hosts, usuarios, servicios...) con la misma configuración que usan los programas (`/etc/nsswitch.conf`).
   - `ahostsv4` → base de datos a consultar: resolución de nombres solo a direcciones IPv4, como lo haría `getaddrinfo()`.
   - `example.com` → nombre que se resuelve.
   - `awk 'NR==1{print $1}'` → `NR==1` se queda solo con la primera línea y `print $1` imprime su primer campo, la IP.
   - `IP=$(...)` y `echo "$IP"` → (ver paso 1).
3. En una terminal, arranca la captura filtrando ese host y el puerto 443. Déjala corriendo.
   ```bash
   sudo tcpdump -i "$IF" -nn -w hs.pcap "tcp port 443 and host $IP"
   ```
   - `sudo` → ejecuta el comando como root; capturar en una interfaz exige privilegios.
   - `tcpdump` → capturador de paquetes de línea de comandos.
   - `-i "$IF"` → interfaz en la que se escucha (la del paso 1).
   - `-nn` → no traduce direcciones IP a nombres ni números de puerto a nombres de servicio (las versiones antiguas necesitaban la segunda `n` para los puertos).
   - `-w hs.pcap` → no imprime los paquetes: los guarda en bruto en el archivo `hs.pcap` para analizarlos después.
   - `"tcp port 443 and host $IP"` → filtro de captura (sintaxis pcap-filter): solo TCP con puerto origen o destino 443 y con la IP del paso 2 como origen o destino; `and` exige ambas condiciones.

   Debe mostrar `tcpdump: listening on <interfaz>...` y quedarse esperando.
4. En otra terminal (define otra vez `IP` si es una shell nueva), haz dos peticiones a esa misma IP: una con la versión de TLS que negocie curl (será 1.3) y otra limitada a TLS 1.2.
   ```bash
   curl -s -o /dev/null --resolve example.com:443:$IP https://example.com
   sleep 2
   curl -s -o /dev/null --tls-max 1.2 --resolve example.com:443:$IP https://example.com
   ```
   - `curl` → cliente HTTP(S) de línea de comandos.
   - `-s` → modo silencioso: no muestra barra de progreso ni mensajes de error.
   - `-o /dev/null` → escribe el cuerpo de la respuesta en `/dev/null`, es decir, lo descarta.
   - `--resolve example.com:443:$IP` → para el par nombre `example.com` y puerto `443` usa la IP indicada en vez de consultar el DNS; así curl conecta a la IP que filtraste sin dejar de mandar el nombre en el SNI.
   - `https://example.com` → URL que se pide.
   - `sleep 2` → espera 2 segundos para separar las dos conexiones en la captura.
   - `--tls-max 1.2` → versión máxima de TLS que curl ofrecerá; obliga a negociar TLS 1.2.
5. Vuelve a la terminal de tcpdump y detenla con `Ctrl+C`. Mostrará cuántos paquetes capturó, algo como `24 packets captured`. Si `hs.pcap` quedó con dueño root y Wireshark no lo abre, cámbialo:
   ```bash
   sudo chown "$USER" hs.pcap
   ```
   - `sudo` → (ver paso 3); hace falta porque el archivo es de root.
   - `chown` → cambia el dueño de un archivo.
   - `"$USER"` → variable de entorno con tu nombre de usuario: el nuevo dueño.
   - `hs.pcap` → archivo al que se le cambia el dueño.
6. Mira el 3-way handshake desde la terminal, leyendo el archivo con tcpdump y quedándote solo con los paquetes que llevan SYN, FIN o RST.
   ```bash
   tcpdump -nn -r hs.pcap 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'
   ```
   - `-nn` → (ver paso 3).
   - `-r hs.pcap` → lee los paquetes de ese archivo en vez de una interfaz; por eso no necesita `sudo`.
   - `tcp[tcpflags]` → el byte de flags de la cabecera TCP.
   - `& (tcp-syn|tcp-fin|tcp-rst)` → AND bit a bit con una máscara que tiene encendidos los bits SYN, FIN y RST (`|` los combina).
   - `!= 0` → se queda con los paquetes que tengan al menos uno de esos tres bits activos.
   - Las comillas simples evitan que la shell interprete `&`, `|` y los paréntesis.

   Para cada una de las dos conexiones debes ver un `[S]` (tu equipo, puerto efímero alto, hacia `.443`), un `[S.]` (el servidor responde SYN-ACK) y, más abajo, los `[F.]` del cierre. El tercer paso del handshake es un `[.]` sin SYN, por eso no sale con este filtro; lo verás en Wireshark.
7. Abre la captura en Wireshark.
   ```bash
   wireshark hs.pcap &
   ```
   - `wireshark` → analizador gráfico de paquetes.
   - `hs.pcap` → archivo de captura que abre al arrancar.
   - `&` → lanza el programa en segundo plano para que la terminal quede libre.

   En la barra de filtro escribe `tcp.flags.syn == 1 || tcp.flags.fin == 1 || tcp.flags.reset == 1` y pulsa Enter. Selecciona el primer paquete `[SYN]`; el siguiente es `[SYN, ACK]`. Quita el filtro (Clear) y comprueba que justo después del SYN-ACK hay un `[ACK]` de tu equipo: ese es el tercer paso. En el panel de detalle, abre "Transmission Control Protocol" y anota `Sequence Number (raw)` del SYN y `Acknowledgment Number (raw)` del SYN-ACK: el segundo debe ser el primero más uno.
8. Localiza el ClientHello y su SNI. Filtro:
   ```
   tls.handshake.type == 1
   ```
   - `tls.handshake.type` → campo de Wireshark con el tipo de mensaje del handshake TLS.
   - `== 1` → tipo 1, ClientHello.

   Saldrán dos paquetes (uno por conexión). En el detalle, abre Transport Layer Security → Handshake Protocol: Client Hello → Extension: server_name → Server Name Indication extension → Server Name. Debe decir `example.com`, en claro, aunque la conexión vaya a estar cifrada.
9. Localiza los ServerHello y la versión negociada. Filtro:
   ```
   tls.handshake.type == 2
   ```
   - `tls.handshake.type == 2` → mensajes de handshake de tipo 2, ServerHello (ver paso 8).

   En el primero (conexión TLS 1.3), el campo `Version` dice `TLS 1.2 (0x0303)` por compatibilidad con equipos viejos; la versión real está en Extension: supported_versions → `Supported Version: TLS 1.3 (0x0304)`. En el segundo (con `--tls-max 1.2`) no hay extensión supported_versions y `Version` es `TLS 1.2 (0x0303)`. Lo mismo desde la terminal:
   ```bash
   tshark -r hs.pcap -Y 'tls.handshake.type == 2' -T fields \
     -e tcp.stream -e tls.handshake.version -e tls.handshake.extensions.supported_version
   ```
   - `tshark` → versión de terminal de Wireshark.
   - `-r hs.pcap` → lee los paquetes de ese archivo.
   - `-Y 'tls.handshake.type == 2'` → filtro de visualización (la misma sintaxis que la barra de Wireshark): solo los ServerHello.
   - `-T fields` → en vez del resumen normal, imprime solo los campos que pidas con `-e`, separados por tabuladores.
   - `-e tcp.stream` → número de conexión TCP que asigna Wireshark (0 la primera, 1 la segunda).
   - `-e tls.handshake.version` → el campo `Version` del ServerHello.
   - `-e tls.handshake.extensions.supported_version` → la versión de la extensión supported_versions, si existe.
   - `\` → al final de la línea, continúa el comando en la línea siguiente.

   Salida esperada: una línea `0  0x0303  0x0304` y otra `1  0x0303` (sin tercer campo).
10. Comprueba que en TLS 1.2 el certificado viaja en claro y en TLS 1.3 no. Filtro:
    ```
    tls.handshake.type == 11
    ```
    - `tls.handshake.type == 11` → mensajes de handshake de tipo 11, Certificate (ver paso 8).

    Solo debe aparecer un paquete, el de la conexión 1 (TLS 1.2): ábrelo y verás la cadena de certificados con el nombre del sitio. En la conexión 0 (TLS 1.3) el mensaje Certificate existe pero va dentro de registros cifrados (`Application Data`), así que Wireshark no puede mostrarlo.
11. Mide el RTT del handshake TCP.
    ```bash
    tshark -r hs.pcap -2 -Y 'tcp.analysis.initial_rtt' -T fields -e tcp.stream -e tcp.analysis.initial_rtt | sort -u
    ```
    - `tshark -r hs.pcap` → (ver paso 9).
    - `-2` → análisis en dos pasadas: primero recorre toda la captura y en la segunda aplica el filtro con los cálculos ya hechos.
    - `-Y 'tcp.analysis.initial_rtt'` → solo paquetes que tengan calculado el campo iRTT.
    - `-T fields`, `-e tcp.stream` → (ver paso 9).
    - `-e tcp.analysis.initial_rtt` → imprime el RTT inicial de la conexión, en segundos.
    - `sort -u` → ordena las líneas y elimina las repetidas, porque cada paquete de la conexión repite el mismo valor.

    Es el tiempo entre tu SYN y el ACK final, en segundos (por ejemplo `0.098`). Multiplícalo por los RTT que cuesta cada versión (1 de TCP + 2 de TLS 1.2, o 1 + 1 de TLS 1.3) para estimar cuánto esperas antes del primer byte útil.
12. Revisa el cierre. Filtro en Wireshark:
    ```
    tcp.flags.fin == 1 || tcp.flags.reset == 1
    ```
    - `tcp.flags.fin == 1` → segmentos TCP con el bit FIN activo.
    - `||` → OR lógico: basta que se cumpla una de las dos condiciones.
    - `tcp.flags.reset == 1` → segmentos con el bit RST activo.

    Lo normal es ver un `[FIN, ACK]` de cada lado, cada uno confirmado por un `[ACK]` del otro: esos son los cuatro segmentos. Antes del FIN suele aparecer un registro TLS `Alert (close_notify)`, que es el cierre de TLS dentro del de TCP. Si en vez del segundo FIN ves un `[RST]`, uno de los dos lados cortó en seco sin esperar (es común en servidores con mucho tráfico); anótalo como diferencia frente al diagrama del README.

### Resultado esperado

Un archivo `hs.pcap` con dos conexiones HTTPS a la misma IP y una lista tuya con, para cada una: los tres paquetes del handshake TCP con sus números de secuencia, el SNI del ClientHello, la versión real negociada (1.3 y 1.2), si el certificado se ve o no, el iRTT y cómo se cerró (FIN de ambos lados o RST).

### Comprueba que lo lograste

- ¿Qué número de ACK manda el servidor en el SYN-ACK si tu SYN tenía `seq` raw 1000? Respuesta: 1001, porque el SYN consume un número de secuencia.
- ¿Por qué el ServerHello de la conexión TLS 1.3 dice `0x0303` en `Version`? Respuesta: es un campo heredado que TLS 1.3 deja congelado en 1.2; la versión real va en la extensión supported_versions.
- ¿Qué ve un observador de la red en la conexión TLS 1.3 sin poder descifrar nada? Respuesta: las IP y puertos, el SNI `example.com` del ClientHello y los tamaños y tiempos de los registros; no ve el certificado ni la URL.
- ¿Cuál handshake ocurre primero y en qué paquete empieza el otro? Respuesta: primero el TCP (SYN, SYN-ACK, ACK); el TLS empieza con el ClientHello, que es el primer paquete con datos después del ACK.

### Limpieza

Borra la captura si no la vas a guardar: `rm hs.pcap`.

## Ejercicio 2: Identifica los puertos que escucha tu equipo

Nodo: [Common Ports and their Uses](README.md#common-ports-and-their-uses) y [Cierre con FIN](README.md#cierre-con-fin) (estado TIME_WAIT).

Objetivo: hacer el inventario de todos los puertos TCP y UDP que tu equipo tiene abiertos, ponerle nombre de servicio a cada uno con `/etc/services`, saber qué proceso lo abrió y en qué dirección escucha, y explicar las conexiones en TIME_WAIT que veas.

Necesitas: tu equipo Linux (cualquier distribución trae `ss` en el paquete `iproute2` y el archivo `/etc/services`). No hace falta instalar nada. Tiempo: 20 minutos.

### Pasos

1. Lista los sockets que escuchan, TCP y UDP, en formato numérico.
   ```bash
   ss -tuln
   ```
   - `ss` → muestra los sockets del sistema (sustituto moderno de `netstat`).
   - `-t` → incluye sockets TCP.
   - `-u` → incluye sockets UDP.
   - `-l` → solo los que están escuchando (en TCP, estado LISTEN).
   - `-n` → muestra números de puerto en vez de nombres de servicio.

   Columnas que importan: `Netid` (tcp/udp), `State` (`LISTEN` en TCP, `UNCONN` en UDP) y `Local Address:Port`. Ejemplo de lectura:
   ```
   udp   UNCONN 0  0   0.0.0.0:5353     0.0.0.0:*
   tcp   LISTEN 0  128 127.0.0.1:631    0.0.0.0:*
   tcp   LISTEN 0  128 0.0.0.0:22       0.0.0.0:*
   ```
2. Clasifica cada línea por la dirección de escucha. Escribe en una nota, para cada puerto, una de estas tres etiquetas:
   - `127.0.0.1` o `[::1]`: solo accesible desde tu propio equipo.
   - `0.0.0.0`, `[::]` o `*`: accesible desde cualquier red a la que estés conectado.
   - Una IP concreta de tu LAN (`192.168.1.10`): accesible solo por esa interfaz.
3. Ponle nombre a cada puerto con `/etc/services`. Este bucle saca la lista sin repetidos y consulta cada uno:
   ```bash
   ss -Htuln | awk '{n=split($5,a,":"); print a[n]"/"$1}' | sort -u -n | while read -r p; do
     printf '%-12s %s\n' "$p" "$(getent services "$p" | awk '{print $1}')"
   done
   ```
   - `ss -tuln` → (ver paso 1).
   - `-H` → quita la línea de cabecera para que `awk` solo reciba datos.
   - `awk '{n=split($5,a,":"); print a[n]"/"$1}'` → parte el quinto campo (`Local Address:Port`) por `:` en el arreglo `a`; `split` devuelve cuántos trozos hay (`n`), así que `a[n]` es el último, el puerto, incluso en IPv6 con varios `:`. Imprime `puerto/protocolo`, porque `$1` es el Netid (tcp o udp).
   - `sort -u -n` → ordena por valor numérico (`-n`) y elimina repetidos (`-u`).
   - `while read -r p; do ... done` → lee la entrada línea a línea en la variable `p`; `-r` evita que las barras invertidas se interpreten como escapes.
   - `printf '%-12s %s\n'` → imprime el primer argumento alineado a la izquierda en 12 caracteres (`%-12s`), un espacio, el segundo (`%s`) y un salto de línea.
   - `"$p"` → el `puerto/protocolo` leído.
   - `getent services "$p"` → busca ese puerto y protocolo en la base de datos de servicios (`/etc/services`); no devuelve nada si no está.
   - `awk '{print $1}'` → se queda con la primera columna, el nombre del servicio.


   Salida de ejemplo:
   ```
   22/tcp       ssh
   546/udp      dhcpv6-client
   631/tcp      ipp
   5353/udp     mdns
   33763/tcp
   ```
   Los que salen sin nombre no están en el registro de IANA o son puertos efímeros abiertos por aplicaciones (servidores de desarrollo, el propio navegador, editores).
4. Para los puertos sin nombre (y para comprobar los que sí lo tienen), pregunta qué proceso los abrió. Hace falta `sudo` para ver procesos de otros usuarios.
   ```bash
   sudo ss -tulnp
   ```
   - `sudo` → ejecuta como root, necesario para ver el proceso dueño de sockets de otros usuarios.
   - `-t`, `-u`, `-l`, `-n` → (ver paso 1).
   - `-p` → muestra el proceso (nombre, PID y descriptor de archivo) que tiene abierto cada socket.

   La última columna dice, por ejemplo, `users:(("cupsd",pid=812,fd=7))`. Anota el programa de cada puerto.
5. Compara nombre y proceso. Si `/etc/services` dice `ipp` y el proceso es `cupsd` (el servidor de impresión), coinciden. Si un puerto con nombre conocido lo abre un programa que no esperas, márcalo: recuerda que el número de puerto es una convención, no una garantía.
6. Mira el rango de puertos efímeros de tu sistema, para saber qué números son "del lado cliente".
   ```bash
   cat /proc/sys/net/ipv4/ip_local_port_range
   ```
   - `cat` → muestra el contenido de un archivo.
   - `/proc/sys/net/ipv4/ip_local_port_range` → archivo virtual del kernel con el primer y el último puerto que se usan como puertos efímeros (equivale al parámetro sysctl `net.ipv4.ip_local_port_range`).

   En Linux suele ser `32768	60999`.
7. Lista las conexiones en TIME_WAIT. Si no aparece ninguna, abre un par de webs en el navegador y repite en menos de un minuto.
   ```bash
   ss -tan state time-wait
   ```
   - `-t`, `-n` → (ver paso 1).
   - `-a` → muestra todos los sockets, tanto los que escuchan como las conexiones.
   - `state time-wait` → filtro de estado: solo sockets TCP en TIME_WAIT. Con este filtro desaparece la columna State.

   Cada línea es `Local Address:Port` y `Peer Address:Port`. El puerto local cae en el rango efímero del paso 6 y el remoto suele ser 443: tu equipo cerró primero la conexión y espera 60 s antes de liberar ese par de puertos.
8. Cuenta cuántas hay por destino, y repite al cabo de un minuto para ver que desaparecen solas.
   ```bash
   ss -Htan state time-wait | awk '{print $4}' | sort | uniq -c | sort -rn | head
   sleep 60; ss -Htan state time-wait | wc -l
   ```
   - `ss -Htan state time-wait` → (ver pasos 3 y 7): conexiones en TIME_WAIT sin cabecera.
   - `awk '{print $4}'` → imprime el cuarto campo, `Peer Address:Port` (sin la columna State, los campos son Recv-Q, Send-Q, local y remoto).
   - `sort` → ordena las líneas para que las iguales queden juntas.
   - `uniq -c` → junta las líneas consecutivas iguales y antepone cuántas veces aparece cada una.
   - `sort -rn` → ordena por ese número (`-n`) de mayor a menor (`-r`).
   - `head` → muestra solo las 10 primeras líneas.
   - `sleep 60` → espera 60 segundos; el `;` ejecuta después el siguiente comando.
   - `wc -l` → cuenta las líneas, es decir, cuántas conexiones siguen en TIME_WAIT.

### Resultado esperado

Una tabla en tus notas (texto plano) con una fila por puerto: puerto/protocolo, nombre de `/etc/services`, proceso dueño, dirección de escucha y tu decisión (debe estar así, debería escuchar solo en localhost, o sobra). Además, una frase que explique de dónde salen las conexiones en TIME_WAIT de tu equipo.

### Comprueba que lo lograste

- ¿Qué diferencia hay entre `0.0.0.0:22` y `127.0.0.1:22`? Respuesta: el primero acepta conexiones desde cualquier interfaz, incluida la red; el segundo solo desde el propio equipo.
- ¿Por qué el puerto local de las conexiones TIME_WAIT es alto y cambia en cada línea? Respuesta: es el puerto efímero que el sistema eligió para cada conexión cliente.
- ¿Por qué TIME_WAIT aparece en tu equipo y no en el servidor? Respuesta: TIME_WAIT lo sufre quien cierra primero; en estas conexiones tu equipo mandó el primer FIN.
- ¿Basta que `/etc/services` diga `ssh` para el 22 para saber que es OpenSSH? Respuesta: no; el nombre solo es la convención de IANA, el proceso real lo dice `ss -p` y la versión del servicio la confirmaría `nmap -sV`.
