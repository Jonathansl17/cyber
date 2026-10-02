# Ejercicios: Herramientas de red

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Los ejercicios 1 y 2 usan el laboratorio del [ejercicio 1 de Virtualización](../06-virtualizacion/ejercicios.md#ejercicio-1-rompe-y-restaura-una-vm-con-un-snapshot): una VM `atacante` (Kali) y una VM `victima` (Debian o Ubuntu) en la red interna `labnet`, sin salida a internet. Como esa red no tiene internet, instala en la víctima lo que pida cada ejercicio antes de pasarla a `labnet` (o con una segunda tarjeta en modo NAT que luego quitas). En los comandos, sustituye `10.10.10.101` por la IP real de tu víctima (`ip -4 -br addr` dentro de ella).

## Ejercicio 1: Observa un escaneo SYN y uno connect con tcpdump

Nodo: [nmap](README.md#nmap) y [tcpdump](README.md#tcpdump).

Objetivo: capturar con tcpdump los paquetes que genera `nmap -sS` y `nmap -sT` contra tu víctima, y relacionar cada estado que informa nmap (open, closed) con la secuencia exacta de flags que aparece en la captura, incluido el ACK extra que distingue al escaneo connect.

Necesitas: el laboratorio descrito arriba. En el atacante, `nmap` y `tcpdump` (vienen en Kali; en Debian/Ubuntu: `sudo apt install nmap tcpdump`). En la víctima, SSH escuchando en el 22 (`sudo apt install openssh-server`, antes de pasarla a `labnet`) y `python3`. Todo el tráfico queda dentro de la red interna. Tiempo: 30 minutos.

### Pasos

1. En la víctima, abre un segundo puerto en el rango que vas a escanear: un servidor web mínimo en el 80. Déjalo corriendo en su terminal.
   ```bash
   sudo python3 -m http.server 80
   ```
   Muestra `Serving HTTP on 0.0.0.0 port 80`. Con el SSH, la víctima tiene abiertos el 22 y el 80; el resto del 1 al 100 debe estar cerrado.
2. En la víctima, en otra terminal, confirma los puertos abiertos.
   ```bash
   ss -tln
   ```
   Deben aparecer `0.0.0.0:22` y `0.0.0.0:80` en estado `LISTEN`.
3. En el atacante, define la IP de la víctima y la interfaz del laboratorio.
   ```bash
   V=10.10.10.101
   IF=$(ip route get "$V" | awk '{for(i=1;i<NF;i++) if($i=="dev") print $(i+1)}')
   echo "$V $IF"
   ```
4. En el atacante, terminal 1: captura todo lo que vaya o venga de la víctima y guárdalo en un archivo (con `-w` no imprime los paquetes, solo los graba).
   ```bash
   sudo tcpdump -nn -i "$IF" -w scan-sS.pcap "host $V"
   ```
5. En el atacante, terminal 2: lanza el escaneo SYN de los puertos 1 a 100. `--reason` hace que nmap diga qué respuesta le hizo decidir cada estado.
   ```bash
   sudo nmap -sS -p 1-100 --reason 10.10.10.101
   ```
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
   Dos líneas: el `[S]` del atacante y un `[R.]` (RST-ACK) de la víctima. Ese RST es lo que nmap llama `closed` con razón `reset`.
8. Cuenta los RST que mandó la víctima: deben ser tantos como puertos cerrados.
   ```bash
   tcpdump -nn -r scan-sS.pcap "src host $V and tcp[tcpflags] & tcp-rst != 0" | wc -l
   ```
   Resultado esperado: `98` (los 100 puertos menos el 22 y el 80). Si sale algo más, nmap reenvió alguna sonda.
9. Repite la captura y el escaneo con connect (`-sT`), que usa la llamada `connect()` del sistema y completa el handshake. Terminal 1:
   ```bash
   sudo tcpdump -nn -i "$IF" -w scan-sT.pcap "host $V"
   ```
   Terminal 2:
   ```bash
   nmap -sT -p 1-100 --reason 10.10.10.101
   ```
   El resultado de nmap es el mismo (22 y 80 open, razón `syn-ack`). Detén tcpdump.
10. Lee el puerto abierto en la nueva captura.
    ```bash
    tcpdump -nn -r scan-sT.pcap 'tcp port 22'
    ```
    Ahora hay al menos cuatro líneas: `[S]`, `[S.]`, `[.]` (el ACK del atacante que completa el handshake: la conexión llegó a ESTABLISHED) y después un `[R.]` del atacante para cerrarla. En el 22 puede aparecer antes del cierre el banner de SSH de la víctima (`[P.]` con `length` mayor que 0).
11. Comprueba en la víctima la consecuencia práctica: el escaneo connect queda registrado por el servicio. Mira el log de SSH de los últimos minutos.
    ```bash
    sudo journalctl -u ssh --since "-10 min"
    ```
    Verás líneas del estilo `error: kex_exchange_identification: Connection closed by remote host` y `Connection closed by 10.10.10.100 port 45680` a la hora del `-sT`; el `-sS` no deja ninguna, porque la conexión nunca se abrió. (En Ubuntu la unidad también es `ssh`; en otras distribuciones puede ser `sshd`.)

### Resultado esperado

Dos capturas (`scan-sS.pcap` y `scan-sT.pcap`) y una tabla en tus notas con la secuencia de flags por caso: `-sS` abierto `[S] [S.] [R]`; `-sS` cerrado `[S] [R.]`; `-sT` abierto `[S] [S.] [.] [R.]`; `-sT` cerrado `[S] [R.]`. Además, la línea del log de SSH que solo aparece con `-sT`.

### Comprueba que lo lograste

- ¿Qué paquete hace que nmap declare un puerto `closed`? Respuesta: un RST (`[R.]`) de la víctima en respuesta al SYN.
- ¿Qué paquete aparece en `-sT` y nunca en `-sS`? Respuesta: el `[.]` (ACK) del atacante que completa el 3-way handshake.
- ¿Por qué `-sT` no necesita `sudo` y `-sS` sí? Respuesta: `-sT` usa `connect()` como cualquier programa; `-sS` fabrica paquetes a mano con sockets raw, que exigen root.
- Si un puerto no respondiera nada, ¿qué estado daría nmap? Respuesta: `filtered`, lo que se ve en el ejercicio 2.

### Limpieza

Detén el servidor web de la víctima con `Ctrl+C` y borra las capturas: `rm scan-sS.pcap scan-sT.pcap`.

## Ejercicio 2: Audita los puertos expuestos y filtra uno con iptables

Nodo: [netstat](README.md#netstat), [iptables](README.md#iptables) y [Port Scanners](README.md#port-scanners).

Objetivo: inventariar los puertos que escucha la víctima con el proceso dueño de cada uno, decidir cuáles deberían estar expuestos, y bloquear uno con iptables comprobando desde el atacante que pasa de `open` a `filtered` con DROP y a `closed` con REJECT.

Necesitas: el laboratorio descrito arriba. En la víctima, `iptables` (Debian/Ubuntu: `sudo apt install iptables`, antes de pasarla a `labnet`; en Debian 12 y Ubuntu actuales el comando escribe en nftables por debajo), `python3` y SSH del ejercicio 1. En el atacante, `nc` y `nmap` (vienen en Kali). Tiempo: 30 minutos.

### Pasos

1. En la víctima, levanta un servicio de prueba en el 8080 y déjalo corriendo en su terminal. Será el puerto que bloquees.
   ```bash
   python3 -m http.server 8080
   ```
2. En la víctima, en otra terminal, lista todo lo que escucha con el proceso dueño.
   ```bash
   sudo ss -tulnp
   ```
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
   Respuesta esperada: `/usr/sbin/sshd` y `/usr/bin/python3.x`.
4. Escribe en tus notas una decisión por puerto: `22/tcp sshd → necesario, pero solo desde la red del lab`; `68/udp cliente DHCP → necesario, no es un servicio que se ofrezca`; `8080/tcp python3 → no debería estar expuesto`.
5. En el atacante, confirma que el 8080 está abierto. `-z` solo prueba la conexión sin mandar datos, `-v` lo explica y `-w 3` pone 3 segundos de límite.
   ```bash
   nc -zv -w 3 10.10.10.101 8080
   ```
   Con el `nc` de Kali (netcat-traditional): `10.10.10.101: inverse host lookup failed... (UNKNOWN) [10.10.10.101] 8080 (http-alt) open`. Con el de OpenBSD: `Connection to 10.10.10.101 8080 port [tcp/http-alt] succeeded!`.
6. En la víctima, mira las reglas actuales y bloquea el 8080 con DROP, insertando la regla en la primera posición para que se evalúe antes que cualquier otra.
   ```bash
   sudo iptables -L INPUT -n -v --line-numbers
   sudo iptables -I INPUT 1 -p tcp --dport 8080 -j DROP
   sudo iptables -L INPUT -n -v --line-numbers
   ```
   La nueva regla aparece como `1  0  0  DROP  tcp  --  *  *  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080`, con los contadores a cero.
7. En el atacante, repite la prueba con nc y con nmap.
   ```bash
   nc -zv -w 3 10.10.10.101 8080
   sudo nmap -sS -p 8080 --reason 10.10.10.101
   ```
   nc tarda los 3 segundos y termina con `Connection timed out` (o `timed out: Operation now in progress`); nmap muestra `8080/tcp filtered http-proxy no-response`. El SYN llega y se descarta en silencio.
8. En la víctima, vuelve a listar las reglas: los contadores `pkts` de la regla 1 han subido. Es la prueba de que el paquete llegó al equipo y lo mató el firewall, no la red.
   ```bash
   sudo iptables -L INPUT -n -v --line-numbers
   ```
9. Cambia DROP por REJECT con reset TCP para ver la otra variante.
   ```bash
   sudo iptables -R INPUT 1 -p tcp --dport 8080 -j REJECT --reject-with tcp-reset
   ```
10. En el atacante, repite la prueba.
    ```bash
    nc -zv -w 3 10.10.10.101 8080
    sudo nmap -sS -p 8080 --reason 10.10.10.101
    ```
    Ahora la respuesta es inmediata: `Connection refused` y `8080/tcp closed http-proxy reset`. Desde fuera es indistinguible de un puerto donde no escucha nada, aunque `python3` sigue escuchando.
11. En la víctima, comprueba que el proceso sigue escuchando:
    ```bash
    ss -tln | grep 8080
    ```
    Sigue en `LISTEN`: el firewall no cierra el puerto, solo impide llegar a él.
12. Si prefieres ufw en lugar de iptables, el equivalente es: `sudo ufw allow 22/tcp`, `sudo ufw enable` (con ufw activo, todo lo no permitido ya entra en DROP), `sudo ufw status numbered`, y `sudo ufw reject 8080/tcp` para la variante REJECT. En Windows, el inventario es `netstat -ano` más `tasklist /FI "PID eq <pid>"`, y el bloqueo, en PowerShell como administrador: `New-NetFirewallRule -DisplayName "Lab bloquea 8080" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Block`.

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
Y `Ctrl+C` en la terminal del `python3 -m http.server 8080`. Si usaste ufw: `sudo ufw delete reject 8080/tcp` y, si no lo quieres activo, `sudo ufw disable`.

## Ejercicio 3: Compara traceroute con UDP, ICMP y TCP

Nodo: [tracert](README.md#tracert).

Objetivo: trazar la ruta hasta `1.1.1.1` con los tres tipos de sonda de `traceroute` de Linux (UDP por defecto, ICMP con `-I` y TCP SYN al 443 con `-T`), ver en una captura qué paquete manda cada uno, y explicar con tus propios datos por qué los asteriscos aparecen en unos saltos con un método y no con otro.

Necesitas: un Linux con internet (tu equipo o una VM en modo NAT o bridged); `traceroute` y `tcpdump` (Debian/Ubuntu: `sudo apt install traceroute tcpdump`; Arch: `sudo pacman -S traceroute tcpdump`). En Windows, `tracert` solo hace ICMP; sirve para comparar con el método `-I`. El destino `1.1.1.1` es el DNS público de Cloudflare: solo lo sondeas con unas decenas de paquetes. Tiempo: 25 minutos.

### Pasos

1. Averigua tu interfaz de salida.
   ```bash
   IF=$(ip route get 1.1.1.1 | awk '{for(i=1;i<NF;i++) if($i=="dev") print $(i+1)}')
   echo "$IF"
   ```
2. En una terminal, deja una captura que muestre las sondas y las respuestas de los tres métodos: UDP a los puertos altos de traceroute, cualquier ICMP, y TCP al 443 de `1.1.1.1`.
   ```bash
   sudo tcpdump -nn -i "$IF" 'icmp or (udp and dst portrange 33434-33534) or (tcp and host 1.1.1.1 and port 443)'
   ```
3. En otra terminal, método UDP (el de defecto, no necesita root). `-n` evita resolver nombres para que vaya más rápido; `-q 3` son tres sondas por salto (el valor por defecto, escrito para que se vea).
   ```bash
   traceroute -n -q 3 1.1.1.1 | tee tr-udp.txt
   ```
   En tcpdump verás líneas `UDP, length 32` hacia `1.1.1.1.33434`, `.33435`... y respuestas `ICMP time exceeded in-transit` de cada router. Es habitual que las últimas líneas de la traza sean `* * *` hasta el salto 30: el destino (o un firewall delante) descarta UDP a puertos altos y no devuelve el `port unreachable` que haría terminar la traza.
4. Método ICMP echo, como `tracert` de Windows. En kernels recientes funciona sin root si tu grupo está en `net.ipv4.ping_group_range`; si da `Operation not permitted`, ponle `sudo`.
   ```bash
   traceroute -n -I 1.1.1.1 | tee tr-icmp.txt
   ```
   En tcpdump las sondas son `ICMP echo request` y el final de la ruta llega como `ICMP echo reply` desde `1.1.1.1`. La traza termina en un salto concreto con `1.1.1.1` y sus tiempos.
5. Método TCP SYN al 443. Necesita root porque fabrica los SYN a mano.
   ```bash
   sudo traceroute -n -T -p 443 1.1.1.1 | tee tr-tcp.txt
   ```
   En tcpdump las sondas son `Flags [S]` hacia `1.1.1.1.443` y el destino contesta `Flags [S.]` (puerto abierto) o `[R.]`. Los routers intermedios siguen respondiendo con `ICMP time exceeded`, igual que antes.
6. Detén tcpdump y pon las tres trazas lado a lado para comparar salto por salto.
   ```bash
   paste -d'|' <(cut -c1-45 tr-udp.txt) <(cut -c1-45 tr-icmp.txt) <(cut -c1-45 tr-tcp.txt)
   ```
7. Clasifica cada `*` que veas en uno de estos tres casos y anótalo:
   - Un salto intermedio con `* * *` en los tres métodos, seguido de saltos que sí responden: ese router reenvía pero no genera (o limita) los ICMP time exceeded. No depende de la sonda, porque la respuesta siempre es ICMP.
   - Saltos finales con `*` en UDP y respuesta en ICMP y TCP: el destino o un firewall cercano descarta UDP a puertos altos, pero deja pasar ping y el 443.
   - Un `*` suelto entre respuestas válidas en un mismo salto: pérdida o limitación de tasa puntual; repite y suele cambiar.
8. Cuenta los saltos hasta el destino en cada método.
   ```bash
   for f in tr-udp.txt tr-icmp.txt tr-tcp.txt; do printf '%s: ' "$f"; grep -cE '^ *[0-9]+ .*1\.1\.1\.1 ' "$f"; done
   ```
   El patrón solo cuenta líneas de salto (empiezan por su número), no la cabecera `traceroute to 1.1.1.1`. Un `1` indica que ese método llegó al destino; `0`, que la traza nunca vio a `1.1.1.1` contestar.

### Resultado esperado

Tres archivos `tr-udp.txt`, `tr-icmp.txt` y `tr-tcp.txt`, y una explicación escrita por ti: qué sonda usa cada método (vista en tcpdump), en qué saltos cambian los asteriscos entre métodos y a cuál de los tres casos del paso 7 corresponde cada uno.

### Comprueba que lo lograste

- ¿Qué mensaje ICMP revela la IP de cada router intermedio, sea cual sea el método? Respuesta: `time exceeded` (tipo 11), generado cuando el TTL llega a 0.
- ¿Cómo sabe traceroute que llegó al destino con UDP, con ICMP y con TCP? Respuesta: con UDP recibe `port unreachable`; con ICMP, `echo reply`; con TCP, un SYN-ACK o un RST del destino.
- Un salto con `* * *` en medio y los siguientes respondiendo, ¿es un corte? Respuesta: no; ese router reenvía los paquetes pero no contesta, por eso los saltos posteriores aparecen.
- ¿Por qué `-T -p 443` suele llegar más lejos que el UDP por defecto? Respuesta: casi ningún firewall bloquea la entrada al 443 de un servidor web, mientras que UDP a puertos altos se filtra a menudo.

### Limpieza

`rm tr-udp.txt tr-icmp.txt tr-tcp.txt`. Si instalaste `traceroute` solo para esto: `sudo apt remove traceroute` (Debian/Ubuntu) o `sudo pacman -Rs traceroute` (Arch).
