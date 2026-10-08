# Ejercicios: Herramientas de red

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Los ejercicios 1 y 2 usan el laboratorio del [ejercicio 1 de Virtualización](../05-virtualizacion/ejercicios.md#ejercicio-1-rompe-y-restaura-una-vm-con-un-snapshot): una VM `atacante` (Kali) y una VM `victima` (Debian o Ubuntu) en la red interna `labnet`, sin salida a internet. Como esa red no tiene internet, instala en la víctima lo que pida cada ejercicio antes de pasarla a `labnet` (o con una segunda tarjeta en modo NAT que luego quitas). En los comandos, sustituye `10.10.10.101` por la IP real de tu víctima (`ip -4 -br addr` dentro de ella).

## Ejercicio 1: Observa un escaneo SYN y uno connect con tcpdump

Nodo: [nmap](README.md#nmap) y [tcpdump](README.md#tcpdump).

Objetivo: capturar con tcpdump los paquetes que genera `nmap -sS` y `nmap -sT` contra tu víctima, y relacionar cada estado que informa nmap (open, closed) con la secuencia exacta de flags que aparece en la captura, incluido el ACK extra que distingue al escaneo connect.

Necesitas: el laboratorio descrito arriba. En el atacante, `nmap` y `tcpdump` (vienen en Kali; en Debian/Ubuntu: `sudo apt install nmap tcpdump`). En la víctima, SSH escuchando en el 22 (`sudo apt install openssh-server`, antes de pasarla a `labnet`) y `python3`. Todo el tráfico queda dentro de la red interna. Tiempo: 30 minutos.

### Pasos

1. En la víctima, abre un segundo puerto en el rango que vas a escanear: un servidor web mínimo en el 80. Déjalo corriendo en su terminal.
   ```bash
   mkdir -p /tmp/web-vacio
   sudo python3 -m http.server 80 --directory /tmp/web-vacio
   ```
   - `mkdir -p /tmp/web-vacio` → crea un directorio vacío que será lo único que publique el servidor; `-p` no da error si ya existe.
   - `sudo` → ejecuta como root; los puertos por debajo de 1024 solo los puede abrir root. Por eso no se sirve el directorio actual: un servidor como root sobre tu home publicaría tus archivos (claves SSH incluidas) a toda la red.
   - `python3 -m http.server` → ejecuta el módulo `http.server` de Python, un servidor web que sirve los archivos de un directorio.
   - `80` → puerto en el que escucha (por defecto escucha en todas las interfaces).
   - `--directory /tmp/web-vacio` → directorio que se publica, en vez del directorio actual.

   Muestra `Serving HTTP on 0.0.0.0 port 80`. Con el SSH, la víctima tiene abiertos el 22 y el 80; el resto del 1 al 100 debe estar cerrado.
2. En la víctima, en otra terminal, confirma los puertos abiertos.
   ```bash
   ss -tln
   ```
   - `ss` → muestra los sockets del sistema.
   - `-t` → solo TCP.
   - `-l` → solo los que escuchan.
   - `-n` → puertos en número, sin traducirlos a nombre de servicio.

   Deben aparecer `0.0.0.0:22` y `0.0.0.0:80` en estado `LISTEN`.
3. En el atacante, define la IP de la víctima y la interfaz del laboratorio.
   ```bash
   V=10.10.10.101
   IF=$(ip route get "$V" | awk '{for(i=1;i<NF;i++) if($i=="dev") print $(i+1)}')
   echo "$V $IF"
   ```
   - `V=10.10.10.101` → guarda la IP de la víctima en la variable `V`.
   - `IF=$(...)` → guarda en `IF` la salida del comando de dentro.
   - `ip route get "$V"` → pregunta al kernel por qué ruta saldría un paquete hacia la víctima; la respuesta incluye `dev <interfaz>`.
   - `awk '{for(i=1;i<NF;i++) if($i=="dev") print $(i+1)}'` → recorre los campos de la línea (`NF` es cuántos hay) y, cuando uno vale `dev`, imprime el siguiente: el nombre de la interfaz.
   - `echo "$V $IF"` → muestra las dos variables para comprobarlas.

4. En el atacante, terminal 1: captura todo lo que vaya o venga de la víctima y guárdalo en un archivo.
   ```bash
   sudo tcpdump -nn -i "$IF" -Z "$USER" -w scan-sS.pcap "host $V"
   ```
   - `sudo` → ejecuta como root, necesario para capturar.
   - `tcpdump` → capturador de paquetes.
   - `-nn` → no traduce IPs a nombres ni puertos a nombres de servicio.
   - `-i "$IF"` → interfaz donde se captura (la del paso 3).
   - `-Z "$USER"` → en cuanto abre la interfaz, tcpdump deja de ser root y pasa a ser tu usuario (`$USER` lo expande tu shell antes de `sudo`). Muchas distribuciones ya hacen que tcpdump suelte privilegios hacia un usuario `tcpdump` sin permiso de escritura en tu home, y entonces `-w` falla con `Permission denied`; con `-Z "$USER"` el archivo se crea en tu directorio y es tuyo, así que luego lo lees sin `sudo`.
   - `-w scan-sS.pcap` → no imprime los paquetes: los graba en ese archivo.
   - `"host $V"` → filtro de captura: solo paquetes cuyo origen o destino sea la IP de la víctima.
5. En el atacante, terminal 2: lanza el escaneo SYN de los puertos 1 a 100.
   ```bash
   sudo nmap -sS -p 1-100 --reason 10.10.10.101
   ```
   - `sudo` → `-sS` necesita root porque fabrica los paquetes a mano.
   - `nmap` → escáner de puertos.
   - `-sS` → escaneo SYN: manda un SYN y decide por la respuesta, sin completar el handshake.
   - `-p 1-100` → puertos a escanear, del 1 al 100.
   - `--reason` → añade la columna REASON: qué respuesta le hizo decidir cada estado.
   - `10.10.10.101` → objetivo, la IP de la víctima.

   Salida esperada:
   ```
   Not shown: 98 closed tcp ports (reset)
   PORT   STATE SERVICE REASON
   22/tcp open  ssh     syn-ack ttl 64
   80/tcp open  http    syn-ack ttl 64
   ```
   Detén tcpdump con `Ctrl+C` en la terminal 1.
6. Lee la captura solo para el puerto abierto 22.
   ```bash
   tcpdump -nn -r scan-sS.pcap 'tcp port 22'
   ```
   - `-nn` → (ver paso 4).
   - `-r scan-sS.pcap` → lee los paquetes de ese archivo en vez de capturar; por eso no hace falta `sudo`.
   - `'tcp port 22'` → filtro: solo TCP con puerto origen o destino 22.

   Verás tres líneas:
   ```
   10.10.10.100.45678 > 10.10.10.101.22: Flags [S], seq 123456789, win 1024, options [mss 1460], length 0
   10.10.10.101.22 > 10.10.10.100.45678: Flags [S.], seq 987654321, ack 123456790, win 64240, ...
   10.10.10.100.45678 > 10.10.10.101.22: Flags [R], seq 123456790, win 0, length 0
   ```
   SYN del atacante, SYN-ACK de la víctima (puerto abierto) y un RST del atacante que corta antes de completar el handshake. Nunca hay un `[.]` del atacante: la conexión no llega a ESTABLISHED.
7. Lee la captura para un puerto cerrado, por ejemplo el 23.
   ```bash
   tcpdump -nn -r scan-sS.pcap 'tcp port 23'
   ```
   - `-nn`, `-r scan-sS.pcap` → (ver paso 6).
   - `'tcp port 23'` → filtro: solo TCP con puerto origen o destino 23.

   Dos líneas: el `[S]` del atacante y un `[R.]` (RST-ACK) de la víctima. Ese RST es lo que nmap llama `closed` con razón `reset`.
8. Cuenta los RST que mandó la víctima: deben ser tantos como puertos cerrados.
   ```bash
   tcpdump -nn -r scan-sS.pcap "src host $V and tcp[tcpflags] & tcp-rst != 0" | wc -l
   ```
   - `-nn`, `-r scan-sS.pcap` → (ver paso 6).
   - `src host $V` → solo paquetes cuyo origen es la víctima.
   - `and` → exige también la condición siguiente.
   - `tcp[tcpflags] & tcp-rst != 0` → toma el byte de flags TCP, aplica un AND con el bit RST y se queda con los paquetes en que ese bit está activo.
   - Comillas dobles → dejan que la shell sustituya `$V` por la IP.
   - `wc -l` → cuenta las líneas, una por paquete.

   Resultado esperado: `98` (los 100 puertos menos el 22 y el 80). Si sale algo más, nmap reenvió alguna sonda.
9. Repite la captura y el escaneo con connect (`-sT`), que usa la llamada `connect()` del sistema y completa el handshake. Terminal 1:
   ```bash
   sudo tcpdump -nn -i "$IF" -Z "$USER" -w scan-sT.pcap "host $V"
   ```
   - `sudo`, `-nn`, `-i "$IF"`, `"host $V"` → (ver paso 4).
   - `-w scan-sT.pcap` → graba esta segunda captura en otro archivo.

   Terminal 2:
   ```bash
   nmap -sT -p 1-100 --reason 10.10.10.101
   ```
   - `-sT` → escaneo connect: usa la llamada `connect()` del sistema operativo, que completa el handshake; no necesita root.
   - `-p 1-100`, `--reason`, `10.10.10.101` → (ver paso 5).

   El resultado de nmap es el mismo (22 y 80 open, razón `syn-ack`). Detén tcpdump.
10. Lee el puerto abierto en la nueva captura.
    ```bash
    tcpdump -nn -r scan-sT.pcap 'tcp port 22'
    ```
    - `-nn`, `'tcp port 22'` → (ver paso 6).
    - `-r scan-sT.pcap` → lee la captura del escaneo connect.

    Ahora hay al menos cuatro líneas: `[S]`, `[S.]`, `[.]` (el ACK del atacante que completa el handshake: la conexión llegó a ESTABLISHED) y después un `[R.]` del atacante para cerrarla. En el 22 puede aparecer antes del cierre el banner de SSH de la víctima (`[P.]` con `length` mayor que 0).
11. Comprueba en la víctima la consecuencia práctica: el escaneo connect queda registrado por el servicio. Mira el log de SSH de los últimos minutos.
    ```bash
    sudo journalctl -u ssh --since "-10 min"
    ```
    - `sudo` → ejecuta como root, para poder leer los logs del sistema.
    - `journalctl` → lee el registro de systemd (journal).
    - `-u ssh` → solo las entradas de la unidad `ssh` (el servicio SSH).
    - `--since "-10 min"` → solo las entradas de los últimos 10 minutos (hora relativa a ahora).

    Verás líneas del estilo `error: kex_exchange_identification: Connection closed by remote host` y `Connection closed by 10.10.10.100 port 45680` a la hora del `-sT`; el `-sS` no deja ninguna, porque la conexión nunca se abrió. (En Ubuntu la unidad también es `ssh`; en otras distribuciones puede ser `sshd`.)

### Resultado esperado

Dos capturas (`scan-sS.pcap` y `scan-sT.pcap`) y una tabla en tus notas con la secuencia de flags por caso: `-sS` abierto `[S] [S.] [R]`; `-sS` cerrado `[S] [R.]`; `-sT` abierto `[S] [S.] [.] [R.]`; `-sT` cerrado `[S] [R.]`. Además, la línea del log de SSH que solo aparece con `-sT`.

### Comprueba que lo lograste

- ¿Qué paquete hace que nmap declare un puerto `closed`? Respuesta: un RST (`[R.]`) de la víctima en respuesta al SYN.
- ¿Qué paquete aparece en `-sT` y nunca en `-sS`? Respuesta: el `[.]` (ACK) del atacante que completa el 3-way handshake.
- ¿Por qué `-sT` no necesita `sudo` y `-sS` sí? Respuesta: `-sT` usa `connect()` como cualquier programa; `-sS` fabrica paquetes a mano con sockets raw, que exigen root.
- Si un puerto no respondiera nada, ¿qué estado daría nmap? Respuesta: `filtered`, lo que se ve en el ejercicio 2.

### Limpieza

Detén el servidor web de la víctima con `Ctrl+C` y borra su directorio (`rmdir /tmp/web-vacio`); en el atacante, borra las capturas: `rm scan-sS.pcap scan-sT.pcap`.

## Ejercicio 2: Audita los puertos expuestos y filtra uno con iptables

Nodo: [netstat](README.md#netstat), [iptables](README.md#iptables) y [Port Scanners](README.md#port-scanners).

Objetivo: inventariar los puertos que escucha la víctima con el proceso dueño de cada uno, decidir cuáles deberían estar expuestos, y bloquear uno con iptables comprobando desde el atacante que pasa de `open` a `filtered` con DROP y a `closed` con REJECT.

Necesitas: el laboratorio descrito arriba. En la víctima, `iptables` (Debian/Ubuntu: `sudo apt install iptables`, antes de pasarla a `labnet`; en Debian 12 y Ubuntu actuales el comando escribe en nftables por debajo), `python3` y SSH del ejercicio 1. En el atacante, `nc` y `nmap` (vienen en Kali). Tiempo: 30 minutos.

### Pasos

1. En la víctima, levanta un servicio de prueba en el 8080 y déjalo corriendo en su terminal. Será el puerto que bloquees.
   ```bash
   python3 -m http.server 8080
   ```
   - `python3 -m http.server` → (ver ejercicio 1).
   - `8080` → puerto de escucha; al ser mayor que 1024 no necesita `sudo`.
2. En la víctima, en otra terminal, lista todo lo que escucha con el proceso dueño.
   ```bash
   sudo ss -tulnp
   ```
   - `sudo` → root, para ver el proceso dueño de los sockets de otros usuarios.
   - `-t`, `-l`, `-n` → (ver ejercicio 1).
   - `-u` → incluye también sockets UDP.
   - `-p` → muestra el proceso (nombre, PID y descriptor) que tiene abierto cada socket.

   Ejemplo:
   ```
   Netid State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
   udp   UNCONN 0      0            0.0.0.0:68        0.0.0.0:*    users:(("dhclient",pid=512,fd=7))
   tcp   LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=688,fd=3))
   tcp   LISTEN 0      5            0.0.0.0:8080      0.0.0.0:*    users:(("python3",pid=1420,fd=3))
   ```
3. Para cada línea, comprueba el ejecutable real del PID (el nombre del proceso lo puede elegir el propio programa; el enlace `exe` no).
   ```bash
   sudo ls -l /proc/688/exe /proc/1420/exe
   ```
   - `sudo` → root, porque el enlace `exe` de procesos de otros usuarios no se puede leer sin privilegios.
   - `ls -l` → formato largo; en un enlace simbólico muestra a dónde apunta (`exe -> /usr/sbin/sshd`).
   - `/proc/688/exe`, `/proc/1420/exe` → enlace que el kernel mantiene para cada PID hacia el ejecutable que se está ejecutando; cambia los PID por los de tu salida del paso 2.

   Respuesta esperada: `/usr/sbin/sshd` y `/usr/bin/python3.x`.
4. Escribe en tus notas una decisión por puerto: `22/tcp sshd → necesario, pero solo desde la red del lab`; `68/udp cliente DHCP → necesario, no es un servicio que se ofrezca`; `8080/tcp python3 → no debería estar expuesto`.
5. En el atacante, confirma que el 8080 está abierto.
   ```bash
   nc -zv -w 3 10.10.10.101 8080
   ```
   - `nc` → netcat, abre conexiones TCP o UDP desde la terminal.
   - `-z` → modo escaneo: solo prueba si la conexión se establece, sin mandar datos.
   - `-v` → modo detallado: dice si conectó o por qué falló.
   - `-w 3` → límite de 3 segundos para conectar antes de rendirse.
   - `10.10.10.101` → IP de la víctima.
   - `8080` → puerto al que se conecta.

   Con el `nc` de Kali (netcat-traditional): `10.10.10.101: inverse host lookup failed... (UNKNOWN) [10.10.10.101] 8080 (http-alt) open`. Con el de OpenBSD: `Connection to 10.10.10.101 8080 port [tcp/http-alt] succeeded!`.
6. En la víctima, mira las reglas actuales y bloquea el 8080 con DROP.
   ```bash
   sudo iptables -L INPUT -n -v --line-numbers
   sudo iptables -I INPUT 1 -p tcp --dport 8080 -j DROP
   sudo iptables -L INPUT -n -v --line-numbers
   ```
   - `sudo` → modificar o leer el firewall exige root.
   - `iptables` → herramienta para ver y cambiar las reglas del firewall del kernel.
   - `-L INPUT` → lista las reglas de la cadena `INPUT`, la que filtra el tráfico que entra al propio equipo.
   - `-n` → muestra IPs y puertos en número, sin resolver nombres.
   - `-v` → modo detallado: añade los contadores de paquetes (`pkts`) y bytes y las interfaces de cada regla.
   - `--line-numbers` → numera las reglas, para poder referirte a ellas por posición.
   - `-I INPUT 1` → inserta una regla en la cadena `INPUT` en la posición 1, para que se evalúe antes que cualquier otra.
   - `-p tcp` → la regla solo se aplica a paquetes TCP.
   - `--dport 8080` → solo a los que van al puerto de destino 8080.
   - `-j DROP` → acción (target): descartar el paquete en silencio, sin responder.

   La nueva regla aparece como `1  0  0  DROP  tcp  --  *  *  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080`, con los contadores a cero.
7. En el atacante, repite la prueba con nc y con nmap.
   ```bash
   nc -zv -w 3 10.10.10.101 8080
   sudo nmap -sS -p 8080 --reason 10.10.10.101
   ```
   - `nc -zv -w 3 10.10.10.101 8080` → (ver paso 5).
   - `sudo nmap -sS --reason 10.10.10.101` → (ver ejercicio 1).
   - `-p 8080` → escanea solo el puerto 8080.

   nc tarda los 3 segundos y termina con `Connection timed out` (o `timed out: Operation now in progress`); nmap muestra `8080/tcp filtered http-proxy no-response`. El SYN llega y se descarta en silencio.
8. En la víctima, vuelve a listar las reglas: los contadores `pkts` de la regla 1 han subido. Es la prueba de que el paquete llegó al equipo y lo mató el firewall, no la red.
   ```bash
   sudo iptables -L INPUT -n -v --line-numbers
   ```
   - `-L INPUT`, `-n`, `-v`, `--line-numbers` → (ver paso 6).
9. Cambia DROP por REJECT con reset TCP para ver la otra variante.
   ```bash
   sudo iptables -R INPUT 1 -p tcp --dport 8080 -j REJECT --reject-with tcp-reset
   ```
   - `-R INPUT 1` → reemplaza la regla número 1 de la cadena `INPUT` por la que se escribe a continuación.
   - `-p tcp`, `--dport 8080` → (ver paso 6).
   - `-j REJECT` → acción: rechazar el paquete respondiendo con un error en vez de descartarlo en silencio.
   - `--reject-with tcp-reset` → el error que se responde es un RST de TCP (sin esta opción, REJECT manda un ICMP port unreachable).

10. En el atacante, repite la prueba.
    ```bash
    nc -zv -w 3 10.10.10.101 8080
    sudo nmap -sS -p 8080 --reason 10.10.10.101
    ```
    - `nc -zv -w 3 10.10.10.101 8080` y `sudo nmap -sS -p 8080 --reason 10.10.10.101` → (ver paso 7).

    Ahora la respuesta es inmediata: `Connection refused` y `8080/tcp closed http-proxy reset`. Desde fuera es indistinguible de un puerto donde no escucha nada, aunque `python3` sigue escuchando.
11. En la víctima, comprueba que el proceso sigue escuchando:
    ```bash
    ss -tln | grep 8080
    ```
    - `ss -tln` → (ver ejercicio 1): sockets TCP que escuchan, en formato numérico.
    - `grep 8080` → se queda con las líneas que contienen `8080`.

    Sigue en `LISTEN`: el firewall no cierra el puerto, solo impide llegar a él.
12. Si prefieres ufw en lugar de iptables, el equivalente es: `sudo ufw allow 22/tcp`, `sudo ufw enable` (con ufw activo, todo lo no permitido ya entra en DROP), `sudo ufw status numbered`, y `sudo ufw reject 8080/tcp` para la variante REJECT.

### Resultado esperado

Una tabla de la víctima con puerto, proceso, ejecutable real y decisión; y la evidencia de tres estados del mismo puerto vistos desde el atacante: `open` sin regla, `filtered` con DROP (timeout, contadores que suben) y `closed` con REJECT tcp-reset (refused inmediato), con el proceso escuchando en los tres casos.

### Comprueba que lo lograste

- ¿Por qué con DROP nc tarda 3 segundos y con REJECT responde al instante? Respuesta: DROP descarta sin avisar, así que el cliente espera hasta su límite; REJECT contesta con un RST.
- Si un cliente no conecta y el contador de la regla DROP no sube, ¿dónde está el problema? Respuesta: antes del servidor, en la red; el paquete no llegó a la cadena INPUT.
- ¿Qué estado de nmap indica que hay un firewall de por medio? Respuesta: `filtered`; `closed` significa que el host contestó con RST.
- ¿Cerrar el puerto con el firewall equivale a parar el servicio? Respuesta: no; el proceso sigue escuchando y un cambio en las reglas lo vuelve a exponer. Si un servicio no hace falta, se para y se deshabilita.

### Limpieza

En la víctima, borra la regla y detén el servicio de prueba:
```bash
sudo iptables -D INPUT 1
sudo iptables -L INPUT -n --line-numbers
```
- `-D INPUT 1` → borra la regla número 1 de la cadena `INPUT`.
- `-L INPUT`, `-n`, `--line-numbers` → (ver paso 6); sin `-v` no se muestran los contadores.

Y `Ctrl+C` en la terminal del `python3 -m http.server 8080`. Si usaste ufw: `sudo ufw delete reject 8080/tcp` y, si no lo quieres activo, `sudo ufw disable`.

## Ejercicio 3: Compara traceroute con UDP, ICMP y TCP

Nodo: [traceroute](README.md#traceroute).

Objetivo: trazar la ruta hasta `1.1.1.1` con los tres tipos de sonda de `traceroute` de Linux (UDP por defecto, ICMP con `-I` y TCP SYN al 443 con `-T`), ver en una captura qué paquete manda cada uno, y explicar con tus propios datos por qué los asteriscos aparecen en unos saltos con un método y no con otro.

Necesitas: un Linux con internet (tu equipo o una VM en modo NAT o bridged); `traceroute` y `tcpdump` (Debian/Ubuntu: `sudo apt install traceroute tcpdump`; Arch: `sudo pacman -S traceroute tcpdump`). El destino `1.1.1.1` es el DNS público de Cloudflare: solo lo sondeas con unas decenas de paquetes. Tiempo: 25 minutos.

### Pasos

1. Averigua tu interfaz de salida.
   ```bash
   IF=$(ip route get 1.1.1.1 | awk '{for(i=1;i<NF;i++) if($i=="dev") print $(i+1)}')
   echo "$IF"
   ```
   - Todo el bloque → (ver ejercicio 1, paso 3), con `1.1.1.1` como destino de la consulta de ruta en lugar de la víctima.
2. En una terminal, deja una captura que muestre las sondas y las respuestas de los tres métodos.
   ```bash
   sudo tcpdump -nn -i "$IF" 'icmp or (udp and dst portrange 33434-33534) or (tcp and host 1.1.1.1 and port 443)'
   ```
   - `sudo tcpdump -nn -i "$IF"` → (ver ejercicio 1, paso 4). Sin `-w`, imprime cada paquete en pantalla.
   - `icmp` → cualquier paquete ICMP (sondas echo y respuestas time exceeded, port unreachable, echo reply).
   - `or` → basta que se cumpla una de las condiciones.
   - `udp and dst portrange 33434-33534` → UDP con puerto de destino entre 33434 y 33534, el rango que usa traceroute para sus sondas UDP.
   - `tcp and host 1.1.1.1 and port 443` → TCP desde o hacia `1.1.1.1` con puerto 443 en origen o destino.
   - Paréntesis y comillas simples → agrupan las condiciones y evitan que la shell los interprete.
3. En otra terminal, método UDP (el de defecto, no necesita root).
   ```bash
   traceroute -n -q 3 1.1.1.1 | tee tr-udp.txt
   ```
   - `traceroute` → traza los routers del camino mandando sondas con TTL creciente (1, 2, 3...); por defecto usa UDP.
   - `-n` → no resuelve las IP de los saltos a nombres, para que vaya más rápido.
   - `-q 3` → tres sondas por salto (el valor por defecto, escrito para que se vea).
   - `1.1.1.1` → destino.
   - `tee tr-udp.txt` → muestra la salida en pantalla y a la vez la guarda en `tr-udp.txt`.

   En tcpdump verás líneas `UDP, length 32` hacia `1.1.1.1.33434`, `.33435`... y respuestas `ICMP time exceeded in-transit` de cada router. Es habitual que las últimas líneas de la traza sean `* * *` hasta el salto 30: el destino (o un firewall delante) descarta UDP a puertos altos y no devuelve el `port unreachable` que haría terminar la traza.
4. Método ICMP echo. En kernels recientes funciona sin root si tu grupo está en `net.ipv4.ping_group_range`; si da `Operation not permitted`, ponle `sudo`.
   ```bash
   traceroute -n -I 1.1.1.1 | tee tr-icmp.txt
   ```
   - `-n`, `1.1.1.1` → (ver paso 3).
   - `-I` → usa ICMP echo request como sonda en vez de UDP.
   - `tee tr-icmp.txt` → muestra la salida y la guarda en `tr-icmp.txt`.

   En tcpdump las sondas son `ICMP echo request` y el final de la ruta llega como `ICMP echo reply` desde `1.1.1.1`. La traza termina en un salto concreto con `1.1.1.1` y sus tiempos.
5. Método TCP SYN al 443. Necesita root porque fabrica los SYN a mano.
   ```bash
   sudo traceroute -n -T -p 443 1.1.1.1 | tee tr-tcp.txt
   ```
   - `sudo` → root, necesario para fabricar los SYN con sockets raw.
   - `-n`, `1.1.1.1` → (ver paso 3).
   - `-T` → usa TCP SYN como sonda.
   - `-p 443` → con TCP, puerto de destino fijo al que van todas las sondas (con UDP sería el puerto base que se incrementa en cada sonda).
   - `tee tr-tcp.txt` → muestra la salida y la guarda en `tr-tcp.txt`.

   En tcpdump las sondas son `Flags [S]` hacia `1.1.1.1.443` y el destino contesta `Flags [S.]` (puerto abierto) o `[R.]`. Los routers intermedios siguen respondiendo con `ICMP time exceeded`, igual que antes.
6. Detén tcpdump y pon las tres trazas lado a lado para comparar salto por salto.
   ```bash
   paste -d'|' <(cut -c1-45 tr-udp.txt) <(cut -c1-45 tr-icmp.txt) <(cut -c1-45 tr-tcp.txt)
   ```
   - `paste` → une línea a línea varios archivos, poniendo la línea 1 de cada uno en la misma fila, y así sucesivamente.
   - `-d'|'` → usa `|` como separador entre columnas en lugar del tabulador.
   - `<( ... )` → sustitución de procesos de bash: la salida del comando de dentro se pasa a `paste` como si fuera un archivo.
   - `cut -c1-45` → se queda con los caracteres 1 a 45 de cada línea, para que las tres columnas quepan en pantalla.
   - `tr-udp.txt`, `tr-icmp.txt`, `tr-tcp.txt` → las tres trazas guardadas.

7. Clasifica cada `*` que veas en uno de estos tres casos y anótalo:
   - Un salto intermedio con `* * *` en los tres métodos, seguido de saltos que sí responden: ese router reenvía pero no genera (o limita) los ICMP time exceeded. No depende de la sonda, porque la respuesta siempre es ICMP.
   - Saltos finales con `*` en UDP y respuesta en ICMP y TCP: el destino o un firewall cercano descarta UDP a puertos altos, pero deja pasar ping y el 443.
   - Un `*` suelto entre respuestas válidas en un mismo salto: pérdida o limitación de tasa puntual; repite y suele cambiar.
8. Cuenta los saltos hasta el destino en cada método.
   ```bash
   for f in tr-udp.txt tr-icmp.txt tr-tcp.txt; do printf '%s: ' "$f"; grep -cE '^ *[0-9]+ .*1\.1\.1\.1 ' "$f"; done
   ```
   - `for f in tr-udp.txt tr-icmp.txt tr-tcp.txt; do ...; done` → repite el cuerpo una vez por archivo, con su nombre en la variable `f`.
   - `printf '%s: ' "$f"` → imprime el nombre del archivo seguido de `: `, sin salto de línea, para que el número salga al lado.
   - `grep -c` → en vez de mostrar las líneas que coinciden, imprime cuántas son.
   - `-E` → expresiones regulares extendidas (permite `+` sin escapar).
   - `'^ *[0-9]+ .*1\.1\.1\.1 '` → líneas que empiezan (`^`) con espacios opcionales (` *`), un número de salto (`[0-9]+`), un espacio y, más adelante (`.*`), la IP `1.1.1.1` seguida de espacio; los `\.` son puntos literales. Así solo cuenta líneas de salto (empiezan por su número), no la cabecera `traceroute to 1.1.1.1`.
   - `"$f"` → archivo en el que busca.

   Un `1` indica que ese método llegó al destino; `0`, que la traza nunca vio a `1.1.1.1` contestar.

### Resultado esperado

Tres archivos `tr-udp.txt`, `tr-icmp.txt` y `tr-tcp.txt`, y una explicación escrita por ti: qué sonda usa cada método (vista en tcpdump), en qué saltos cambian los asteriscos entre métodos y a cuál de los tres casos del paso 7 corresponde cada uno.

### Comprueba que lo lograste

- ¿Qué mensaje ICMP revela la IP de cada router intermedio, sea cual sea el método? Respuesta: `time exceeded` (tipo 11), generado cuando el TTL llega a 0.
- ¿Cómo sabe traceroute que llegó al destino con UDP, con ICMP y con TCP? Respuesta: con UDP recibe `port unreachable`; con ICMP, `echo reply`; con TCP, un SYN-ACK o un RST del destino.
- Un salto con `* * *` en medio y los siguientes respondiendo, ¿es un corte? Respuesta: no; ese router reenvía los paquetes pero no contesta, por eso los saltos posteriores aparecen.
- ¿Por qué `-T -p 443` suele llegar más lejos que el UDP por defecto? Respuesta: casi ningún firewall bloquea la entrada al 443 de un servidor web, mientras que UDP a puertos altos se filtra a menudo.

### Limpieza

`rm tr-udp.txt tr-icmp.txt tr-tcp.txt`. Si instalaste `traceroute` solo para esto: `sudo apt remove traceroute` (Debian/Ubuntu) o `sudo pacman -Rs traceroute` (Arch).

## Ejercicio 4: Analiza tu propio tráfico con Wireshark y tshark

Nodo: [Protocol Analyzers](README.md#protocol-analyzers) y [Packet Sniffers](README.md#packet-sniffers).

Objetivo: capturar unos segundos de tu propio tráfico, encontrar en él lo que te interesa con filtros de visualización (DNS, HTTP, SYN iniciales, una IP concreta), reconstruir una petición HTTP completa con Follow TCP Stream y resumir la captura con las estadísticas de conversaciones y de jerarquía de protocolos. Cada paso en Wireshark tiene su equivalente en `tshark`, para que puedas hacerlo también por SSH o en un script.

Necesitas: tu equipo Linux con escritorio, internet, `curl` y `python3`. Wireshark: en Arch `sudo pacman -S wireshark-qt` (trae `tshark`); en Debian/Ubuntu `sudo apt install wireshark tshark` y responde "Sí" cuando pregunte si los usuarios sin privilegios pueden capturar. Solo capturas tráfico de tu propio equipo. Tiempo: 40 minutos.

### Pasos

1. Date permiso de captura sin ser root. La captura la hace el programa `dumpcap`, que solo pueden ejecutar root y los miembros del grupo `wireshark`.
   ```bash
   sudo usermod -aG wireshark "$USER"
   ```
   - `sudo` → ejecuta como root; cambiar grupos de un usuario lo exige.
   - `usermod` → modifica una cuenta de usuario.
   - `-aG wireshark` → añade (`-a`, append) el usuario al grupo suplementario (`-G`) `wireshark` sin quitarlo de los demás; sin `-a` perdería el resto de grupos.
   - `"$USER"` → tu nombre de usuario.

   Cierra la sesión gráfica y vuelve a entrar (o reinicia): los grupos se leen al iniciar sesión. La alternativa sin tocar grupos es capturar con `sudo tshark ...`, pero no abras la interfaz gráfica de Wireshark como root: son millones de líneas de disectores procesando datos de la red con todos los privilegios.
2. Comprueba que el permiso funciona.
   ```bash
   groups
   tshark -D
   ```
   - `groups` → lista los grupos de tu sesión actual; debe aparecer `wireshark`.
   - `tshark` → versión de terminal de Wireshark.
   - `-D` → lista las interfaces en las que puede capturar.

   Salida de ejemplo de `tshark -D`:
   ```
   1. enp3s0
   2. any
   3. lo (Loopback)
   ...
   ```
   Si sale `Couldn't run dumpcap in child process: Permission denied` o la lista está vacía, aún no volviste a iniciar sesión.
3. Prepara un servidor web local en un directorio dedicado, con una página propia. Será el tráfico HTTP en claro que vas a reconstruir.
   ```bash
   mkdir -p ~/lab-wireshark/www
   echo "hola wireshark" > ~/lab-wireshark/www/index.html
   python3 -m http.server 8000 --bind 127.0.0.1 --directory ~/lab-wireshark/www
   ```
   - `mkdir -p ~/lab-wireshark/www` → crea el directorio del laboratorio y el de la web; `-p` crea los intermedios y no falla si ya existen.
   - `echo "hola wireshark" > .../index.html` → escribe ese texto en `index.html`; `>` crea o sobrescribe el archivo.
   - `python3 -m http.server 8000` → servidor web mínimo de Python en el puerto 8000 (alto, así que no necesita `sudo`).
   - `--bind 127.0.0.1` → escucha solo en loopback: nadie de tu red puede conectarse.
   - `--directory ~/lab-wireshark/www` → publica solo ese directorio, no el directorio en el que estás.

   Muestra `Serving HTTP on 127.0.0.1 port 8000` y se queda corriendo. Déjalo en esta terminal.
4. En una segunda terminal, captura 30 segundos en todas las interfaces y guarda el resultado.
   ```bash
   cd ~/lab-wireshark
   tshark -i any -a duration:30 -w propio.pcapng
   ```
   - `cd ~/lab-wireshark` → entra al directorio del laboratorio, donde se guardará la captura.
   - `-i any` → pseudo-interfaz de Linux que captura en todas a la vez, incluida `lo`, por donde pasa el tráfico a `127.0.0.1`.
   - `-a duration:30` → condición de parada automática (`-a`, autostop): termina a los 30 segundos.
   - `-w propio.pcapng` → guarda los paquetes en ese archivo (formato pcapng) en vez de mostrarlos.

   Muestra un contador de paquetes mientras captura.
5. Mientras corre la captura, en una tercera terminal genera tráfico: una resolución DNS, una petición HTTP en claro a tu servidor y una HTTPS a un sitio público.
   ```bash
   getent ahostsv4 example.com
   curl -s http://127.0.0.1:8000/index.html
   curl -s -o /dev/null https://example.com
   ```
   - `getent ahostsv4 example.com` → resuelve el nombre a IPv4 como lo haría cualquier programa; si no estaba en caché, sale una consulta DNS.
   - `curl -s http://127.0.0.1:8000/index.html` → pide la página a tu servidor local; `-s` quita la barra de progreso. Debe imprimir `hola wireshark`.
   - `curl -s -o /dev/null https://example.com` → hace una conexión HTTPS completa y descarta el cuerpo (`-o /dev/null`).

   Apunta la IP que dio `getent` (la primera columna). Espera a que tshark termine solo.
6. Abre la captura en Wireshark.
   ```bash
   wireshark propio.pcapng &
   ```
   - `wireshark` → analizador gráfico.
   - `propio.pcapng` → archivo que abre al arrancar.
   - `&` → lo lanza en segundo plano para dejar libre la terminal.

   Verás la lista de paquetes arriba, el detalle por capas en medio y los bytes abajo. Además de tus peticiones aparecerá tráfico de fondo (navegador, actualizaciones, sincronización): es normal y es justo para lo que sirven los filtros.
7. Aplica estos filtros de visualización, uno a uno, en la barra superior (Enter para aplicar; la barra se pone verde si la sintaxis es válida y roja si no):
   ```
   dns
   http
   tcp.flags.syn == 1 && tcp.flags.ack == 0
   ip.addr == <IP de example.com del paso 5>
   ```
   - `dns` → solo paquetes que Wireshark decodificó como DNS. Busca la consulta `Standard query A example.com` y su `Standard query response` con la IP. Si no aparece, la respuesta estaba en caché o tu sistema usa DNS cifrado (DoT/DoH); prueba con otro nombre que no hayas visitado.
   - `http` → solo HTTP en claro: tu `GET /index.html HTTP/1.1` y la respuesta `HTTP/1.0 200 OK`. El HTTPS a `example.com` no sale aquí, sale como `tls`.
   - `tcp.flags.syn == 1 && tcp.flags.ack == 0` → solo el primer paquete de cada conexión TCP (SYN sin ACK): una línea por conexión que se abrió durante la captura. `&&` es el Y lógico.
   - `ip.addr == ...` → todo lo que tenga esa IP como origen o destino: el handshake TCP, el TLS y el cierre de tu conexión HTTPS.

   Anota cuántos paquetes muestra cada filtro (barra de estado inferior, `Displayed:`).
8. Reconstruye la petición HTTP. Con el filtro `http`, haz clic derecho en el paquete `GET /index.html` → Follow → TCP Stream (atajo `Ctrl+Alt+Shift+T`).

   Se abre una ventana con la conversación completa como texto: en rojo lo que mandó el cliente (`GET /index.html HTTP/1.1`, `Host: 127.0.0.1:8000`, `User-Agent: curl/...`) y en azul la respuesta del servidor (`HTTP/1.0 200 OK`, `Server: SimpleHTTP/... Python/...`, y al final `hola wireshark`). Al cerrar la ventana, Wireshark deja aplicado el filtro `tcp.stream eq N` con el número de esa conexión. Repite Follow sobre un paquete de la conexión HTTPS: solo verás bytes ilegibles, porque va cifrada.
9. Resume la captura. Menú Statistics → Conversations, pestaña IPv4: una fila por par de IPs con paquetes, bytes y duración; ordena por Bytes para ver con quién habló más tu equipo. Luego Statistics → Protocol Hierarchy: un árbol con el porcentaje de paquetes y bytes de cada protocolo (`tcp` → `tls`, `http`; `udp` → `dns`).
10. Repite lo mismo desde la terminal con tshark, sobre el mismo archivo.
    ```bash
    tshark -r propio.pcapng -Y 'dns'
    tshark -r propio.pcapng -Y 'tcp.flags.syn == 1 && tcp.flags.ack == 0'
    tshark -r propio.pcapng -Y 'http' -T fields -e tcp.stream -e http.request.method -e http.request.uri -e http.response.code
    ```
    - `-r propio.pcapng` → lee de ese archivo en vez de capturar.
    - `-Y '...'` → filtro de visualización, la misma sintaxis que la barra de Wireshark.
    - `-T fields` → imprime solo los campos pedidos con `-e`, separados por tabuladores.
    - `-e tcp.stream` → número de conexión TCP (el `N` de `tcp.stream eq N`).
    - `-e http.request.method`, `-e http.request.uri` → método y ruta de la petición.
    - `-e http.response.code` → código de estado de la respuesta.

    La última debe dar dos líneas con el mismo número de stream: una `GET  /index.html` y otra con `200`. Anota ese número.
11. Follow TCP Stream y estadísticas desde la terminal.
    ```bash
    tshark -r propio.pcapng -q -z follow,tcp,ascii,N
    tshark -r propio.pcapng -q -z conv,ip
    tshark -r propio.pcapng -q -z io,phs
    ```
    - `-q` → no imprime la lista de paquetes, solo el resultado de las estadísticas.
    - `-z follow,tcp,ascii,N` → reconstruye la conexión TCP número `N` (cámbialo por el del paso 10) como texto ASCII; equivale a Follow TCP Stream.
    - `-z conv,ip` → tabla de conversaciones por par de IPs; equivale a Statistics → Conversations (con `conv,tcp` la verías por conexión).
    - `-z io,phs` → protocol hierarchy statistics; equivale a Statistics → Protocol Hierarchy.

    El `follow` muestra la petición y la respuesta completas con `hola wireshark` al final. La primera línea de `io,phs` es `frame` y debajo verás `sll` (la cabecera que usa Linux en la interfaz `any` en lugar de Ethernet), luego `ip`, `tcp`, `udp`...

### Resultado esperado

El archivo `propio.pcapng` y en tus notas: cuántos paquetes daba cada uno de los cuatro filtros, la conversación HTTP reconstruida (petición y respuesta), la IP con la que tu equipo intercambió más bytes y el porcentaje de `tls` frente a `http` en la jerarquía de protocolos.

### Comprueba que lo lograste

- ¿Por qué Follow TCP Stream muestra la página en el HTTP local y no en el HTTPS? Respuesta: HTTP viaja en claro; en HTTPS el contenido va cifrado con TLS y Wireshark solo ve bytes.
- ¿Qué diferencia hay entre el filtro de captura de tcpdump (`host 1.2.3.4`) y el de visualización de Wireshark (`ip.addr == 1.2.3.4`)? Respuesta: el de captura decide qué se guarda y no se puede deshacer; el de visualización solo oculta paquetes de una captura ya guardada.
- ¿Por qué `tcp.flags.syn == 1 && tcp.flags.ack == 0` da exactamente una línea por conexión? Respuesta: solo el primer paquete del 3-way handshake lleva SYN sin ACK; el SYN-ACK tiene el ACK activo.
- ¿Por qué hace falta estar en el grupo `wireshark` y no basta con instalar el programa? Respuesta: capturar exige capacidades de root (`CAP_NET_RAW`); `dumpcap` las tiene, pero solo lo puede ejecutar root o el grupo `wireshark`.

### Limpieza

`Ctrl+C` en la terminal del servidor web y `rm -r ~/lab-wireshark`. Si no quieres conservar el permiso de captura: `sudo gpasswd -d "$USER" wireshark` (`gpasswd -d` quita al usuario de ese grupo) y vuelve a iniciar sesión.

## Ejercicio 5: Detecta servicios y versiones en scanme.nmap.org

Nodo: [nmap](README.md#nmap) y [Port Scanners](README.md#port-scanners).

Objetivo: hacer tu primer escaneo contra un equipo de internet que no es tuyo, pero con permiso explícito: `scanme.nmap.org`, que el proyecto Nmap mantiene precisamente para esto. Detectarás qué servicio y qué versión corre en unos pocos puertos y entenderás de dónde sale cada dato.

Alcance legal, antes de nada: escanear sistemas sin autorización puede ser delito según la legislación de tu país (en muchos lo es el acceso o el intento de acceso no autorizado) y casi siempre viola los términos de tu proveedor de internet. Solo escanea tus equipos, tu laboratorio o lo que tengas autorizado por escrito. La autorización de `scanme.nmap.org` está en su propia página (http://scanme.nmap.org): dice "You are authorized to scan this machine with Nmap or other port scanners", pide no saturarlo ("unas pocas veces al día está bien, no 100") y prohíbe usarlo para probar herramientas de fuerza bruta contra SSH. Escanear no autoriza nada más: ni explotar servicios ni atacar el equipo. Por eso este ejercicio escanea solo cinco puertos y lo haces una vez.

Necesitas: un Linux con internet y `nmap` (`sudo apt install nmap` o `sudo pacman -S nmap`). Tiempo: 15 minutos.

### Pasos

1. Lee la autorización del objetivo, para ver por ti mismo qué permite.
   ```bash
   curl -s http://scanme.nmap.org | grep -i "authorized"
   ```
   - `curl -s http://scanme.nmap.org` → descarga la página del servidor sin barra de progreso.
   - `| grep -i "authorized"` → muestra solo las líneas que contienen `authorized`, sin distinguir mayúsculas (`-i`).

   Debe salir el párrafo con `You are authorized to scan this machine with Nmap or other port scanners. Try not to hammer on the server too hard. A few scans in a day is fine...`.
2. Escanea cinco puertos detectando versiones.
   ```bash
   nmap -sV -p 22,80,443,9929,31337 --reason scanme.nmap.org
   ```
   - `nmap` → escáner de puertos.
   - `-sV` → detección de versión: tras ver que un puerto está abierto, se conecta, lee lo que el servicio responde (el banner) y le manda sondas para compararlas con su base de firmas (`nmap-service-probes`).
   - `-p 22,80,443,9929,31337` → solo esos puertos, separados por comas.
   - `--reason` → muestra en qué respuesta se basa cada estado.
   - `scanme.nmap.org` → objetivo autorizado.

   Sin `sudo`, nmap usa el escaneo connect (`-sT`) porque no puede fabricar paquetes a mano. Salida esperada (las versiones pueden cambiar):
   ```
   PORT      STATE  SERVICE    REASON       VERSION
   22/tcp    open   ssh        syn-ack      OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
   80/tcp    open   http       syn-ack      Apache httpd 2.4.7 ((Ubuntu))
   443/tcp   closed https      conn-refused
   9929/tcp  open   nping-echo syn-ack      Nping echo
   31337/tcp closed Elite      conn-refused
   Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
   ```
3. Interpreta cada columna. `STATE`: `open` si el servicio aceptó la conexión, `closed` si el host respondió RST (`conn-refused` en un escaneo connect). `SERVICE` en un puerto cerrado es solo el nombre que el archivo `nmap-services` asocia a ese número; en uno abierto con `-sV` es lo que nmap identificó. `VERSION` sale del banner: el de SSH anuncia el software y hasta el paquete de la distribución (`6.6.1p1 Ubuntu 2ubuntu2.13`), y `Service Info` deduce de ahí el sistema operativo.
4. Comprueba tú el banner de SSH sin nmap, para ver de dónde sale la versión.
   ```bash
   timeout 5 nc scanme.nmap.org 22 </dev/null
   ```
   - `timeout 5` → mata el comando a los 5 segundos si no termina solo.
   - `nc scanme.nmap.org 22` → netcat: abre una conexión TCP a ese host y puerto y muestra lo que llegue.
   - `</dev/null` → no envía nada; solo escucha lo que el servidor dice primero.

   Debe imprimir una línea como `SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2.13` y, a los 5 segundos, terminar (algunas versiones de `nc` no cierran solas al recibir el fin de la entrada, de ahí el `timeout`): el servidor SSH anuncia su versión antes de cualquier autenticación, y eso es lo que leyó `-sV`.
5. Valora lo encontrado como lo haría un auditor: busca en tus notas qué antigüedad tienen OpenSSH 6.6 y Apache 2.4.7 (ambos de 2014) y anota que, en un sistema real, un banner así sería un hallazgo (software sin parchear y fuga de información de versión). En `scanme` es deliberado: es un equipo de prueba.

### Resultado esperado

Una tabla con los cinco puertos, su estado, el servicio y la versión detectados, el banner de SSH obtenido con `nc` y una frase que explique la diferencia entre lo que dice `SERVICE` en un puerto cerrado y en uno abierto con `-sV`.

### Comprueba que lo lograste

- ¿Por qué puedes escanear `scanme.nmap.org` y no la web de tu universidad? Respuesta: su dueño lo autoriza públicamente y con condiciones; sin autorización expresa del dueño, escanear puede ser ilegal.
- ¿De dónde saca nmap la versión `OpenSSH 6.6.1p1`? Respuesta: del banner que el servidor manda al conectar, comparado con las firmas de `nmap-service-probes`.
- ¿Qué significa `conn-refused`? Respuesta: el host respondió con RST al intento de conexión: el puerto está cerrado pero el host está vivo.
- ¿Por qué el nombre `Elite` del 31337 no dice nada del equipo? Respuesta: el puerto está cerrado; nmap solo pone el nombre que asocia por convención a ese número.

### Limpieza

No se crea ningún archivo. No repitas el escaneo en bucle ni amplíes el rango de puertos: respeta el límite de "unas pocas veces al día".
