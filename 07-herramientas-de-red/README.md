# Herramientas de red

## Conceptos previos

- Interfaz de red: la "boca" por la que un equipo se conecta (`eth0`/`enp3s0` cableada, `wlan0`/`wlp2s0` Wi-Fi, `lo` loopback, la propia máquina).
- Dirección MAC: identificador de 48 bits de la tarjeta de red (`a4:5e:60:12:34:56`), usado dentro de la red local.
- Dirección IP, máscara, puerta de enlace (gateway): ver [`04-direccionamiento-ip`](../04-direccionamiento-ip/). La puerta de enlace es el router al que se manda todo lo que no es de tu red.
- Puerto, TCP, UDP, handshake, flags SYN/ACK/RST/FIN: ver [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/).
- ICMP (Internet Control Message Protocol): protocolo de mensajes de control de IP; lo usan `ping` (echo request tipo 8 / echo reply tipo 0) y `traceroute` (time exceeded tipo 11).
- TTL (time to live): contador del paquete IP que cada router resta en 1; al llegar a 0 el paquete se descarta y se avisa con ICMP tipo 11.
- DNS: sistema que traduce nombres a IP; tipos de registro A (IPv4), AAAA (IPv6), MX (correo), NS (servidores del dominio), CNAME (alias), TXT (texto libre, SPF), PTR (IP a nombre).
- Tabla de rutas: lista que el sistema consulta para decidir por qué interfaz y hacia qué router sale cada paquete.
- Modo promiscuo: modo en que la tarjeta entrega al sistema todas las tramas que ve, no solo las dirigidas a ella.
- pcap: formato de archivo estándar para guardar paquetes capturados (`.pcap`, `.pcapng`).
- Root / administrador: capturar paquetes, cambiar rutas o tocar el firewall exige privilegios (`sudo` en Linux, consola "Ejecutar como administrador" en Windows).

> [!NOTE]
> Escanear o capturar tráfico de redes ajenas sin autorización es ilegal en la mayoría de países. Todos los ejemplos usan tu red de casa, tu laboratorio o `scanme.nmap.org`, que el proyecto Nmap autoriza a escanear de forma moderada.

## Troubleshooting Tools

Las **herramientas de diagnóstico de red** son programas de línea de comandos y gráficos que permiten ver la configuración de un equipo, comprobar si otro responde, seguir el camino de los paquetes, consultar DNS, listar conexiones, descubrir puertos abiertos y mirar el contenido de los paquetes. Existen porque "no hay internet" puede ser cien cosas distintas, y cada herramienta descarta una capa.

Analogía: son el kit de un mecánico. No se desarma el motor si el problema es que no hay gasolina; se empieza por lo simple y se avanza.

El orden de diagnóstico sigue las capas, de abajo hacia arriba:

```
¿Tengo IP, gateway y DNS?           ip addr / ipconfig            capa 3 local
¿Llego al gateway?                  ping 192.168.1.1              capa 3 LAN
¿Conozco la MAC del gateway?        arp / ip neigh                capa 2
¿Llego a internet por IP?           ping 1.1.1.1                  capa 3 WAN
¿Dónde se corta el camino?          traceroute / tracert          capa 3 ruta
¿Resuelve el nombre?                dig / nslookup                capa 7 DNS
¿El servicio escucha?               ss / netstat (local)          capa 4
¿El puerto está abierto desde fuera? nmap                          capa 4
¿Qué pasa realmente en el cable?    tcpdump / Wireshark           todas
¿Lo bloquea el firewall?            iptables / nft                capa 3-4
```

Equivalencias Linux / Windows que se preguntan:

```
Linux moderno (iproute2)   Linux antiguo (net-tools)   Windows
ip addr                    ifconfig                    ipconfig
ip route                   route -n                    route print
ip neigh                   arp -n                      arp -a
ss                         netstat                     netstat
traceroute / tracepath     traceroute                  tracert
dig                        nslookup / host             nslookup / Resolve-DnsName
tcpdump                    tcpdump                     pktmon / Wireshark (dumpcap)
nft / iptables             iptables                    netsh advfirewall / Windows Defender Firewall
```

### ipconfig

**ipconfig** es un comando de Windows que muestra y gestiona la configuración IP de cada interfaz: dirección, máscara, puerta de enlace, servidores DNS, concesión DHCP y MAC. En Linux su equivalente moderno es `ip` (paquete iproute2), y el antiguo `ifconfig` (net-tools), que sigue apareciendo en exámenes pero está obsoleto.

Existe porque lo primero ante cualquier fallo es saber qué configuración tiene el equipo: sin IP válida, sin gateway o sin DNS, ninguna otra prueba tiene sentido.

Analogía: es mirar tu propia credencial antes de entrar al edificio: tu nombre (IP), tu piso (red), por qué puerta se sale (gateway) y a quién preguntar direcciones (DNS).

Banderas clave en Windows:

```
ipconfig                 resumen: IP, máscara, gateway por interfaz
ipconfig /all            todo: MAC, DHCP, servidor DHCP, concesión, DNS
ipconfig /release        suelta la IP obtenida por DHCP
ipconfig /renew          pide una IP nueva al DHCP
ipconfig /flushdns       vacía la caché DNS local
ipconfig /displaydns     muestra la caché DNS local
```

Ejemplo en Windows:

```
C:\> ipconfig /all
Ethernet adapter Ethernet:
   Physical Address. . . . . . . . . : A4-5E-60-12-34-56
   DHCP Enabled. . . . . . . . . . . : Yes
   IPv4 Address. . . . . . . . . . . : 192.168.1.10(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Lease Obtained. . . . . . . . . . : Thursday, October 1, 2026 8:00:00 AM
   Default Gateway . . . . . . . . . : 192.168.1.1
   DHCP Server . . . . . . . . . . . : 192.168.1.1
   DNS Servers . . . . . . . . . . . : 1.1.1.1
```

Lectura: la red es `192.168.1.0/24` (máscara 255.255.255.0), el router hace de gateway y de DHCP, y el DNS es Cloudflare. Si la IPv4 empezara por `169.254.x.x` (APIPA, dirección que Windows se pone sola cuando ningún DHCP responde), el problema estaría en DHCP, no en internet.

Equivalente en Linux:

```
$ ip -br addr
lo        UNKNOWN  127.0.0.1/8 ::1/128
enp3s0    UP       192.168.1.10/24 fe80::a65e:60ff:fe12:3456/64
$ ip route
default via 192.168.1.1 dev enp3s0 proto dhcp metric 100
192.168.1.0/24 dev enp3s0 proto kernel scope link src 192.168.1.10
$ resolvectl status | grep 'DNS Servers'
       DNS Servers: 1.1.1.1
```

`-br` (brief) da una línea por interfaz; `UP` indica que la interfaz está activa. Para renovar DHCP en Linux depende del gestor: `nmcli connection up <nombre>` con NetworkManager o `dhclient -r && dhclient` en sistemas clásicos.

```
ipconfig → configuración IP en Windows.
ip addr / ip -br addr → lo mismo en Linux moderno.
ifconfig → equivalente Linux obsoleto (net-tools).
169.254.x.x (APIPA) → el DHCP no respondió.
ipconfig /flushdns → vacía la caché DNS local.
```

### ping

**ping** es una utilidad que envía paquetes ICMP echo request a un destino y mide si vuelven los echo reply y cuánto tardan. Responde dos preguntas: ¿el destino es alcanzable por IP?, y ¿con qué latencia y pérdida?

Analogía: es el sonar de un submarino: mandas un "ping" y cronometras el eco.

Banderas clave:

```
Linux                          Windows              Qué hace
ping -c 4 host                 ping -n 4 host       número de paquetes (Linux es infinito por defecto)
ping -i 0.2 host               (no existe)          intervalo entre paquetes
ping -s 1472 host              ping -l 1472 host    tamaño de los datos
ping -M do -s 1472 host        ping -f -l 1472 host prohíbe fragmentar (probar MTU)
ping -t 64 host                ping -i 64 host      fija el TTL de salida
ping -4 / -6 host              ping -4 / -6 host    fuerza IPv4 / IPv6
(no aplica)                    ping -t host         Windows: sin fin, hasta Ctrl+C
```

Ojo: `-t` significa TTL en Linux y "continuo" en Windows; `-i` es intervalo en Linux y TTL en Windows.

Ejemplo:

```
$ ping -c 4 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=57 time=12.3 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=57 time=11.8 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=57 time=12.1 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=57 time=40.6 ms

--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 11.8/19.2/40.6/12.4 ms
```

Lectura: `56(84)` son 56 bytes de datos más 28 de cabeceras (20 IP + 8 ICMP). `icmp_seq` saltado indica pérdida. `ttl=57` permite estimar distancia y sistema: Linux y macOS salen con TTL 64, Windows con 128 y muchos equipos de red con 255; 64 − 57 = 7 saltos. `mdev` (desviación) alta indica jitter (variación del retardo): aquí el cuarto paquete tardó 40 ms. Reglas prácticas: menos de 30 ms en la misma ciudad es bueno; pérdida por encima del 1-2 % se nota en llamadas.

Mensajes de error: `Destination Host Unreachable` (nadie respondió al ARP en tu propia red o un router no tiene ruta), `Request timed out` / sin respuesta (el paquete o la respuesta se perdió o se filtró), `TTL expired in transit` (bucle de rutas).

```
ping → echo request/reply ICMP; mide alcance, latencia y pérdida.
TTL recibido → 64 Linux, 128 Windows, 255 equipos de red, menos los saltos.
Host Unreachable → no hay ruta o nadie contestó ARP.
Timeout → perdido o filtrado; no prueba que el host esté caído.
```

### Un ping sin respuesta no prueba que el host esté caído

> [!IMPORTANT]
> El silencio de una herramienta de red casi nunca es una prueba: un firewall que descarta ICMP o un puerto filtrado se ven igual que un equipo apagado. Para concluir "caído" hay que probar por otra vía.

La ausencia de respuesta es un resultado ambiguo que aparece en varias herramientas, porque descartar un paquete en silencio (DROP) es la política por defecto de muchos firewalls. Windows trae bloqueado el ICMP echo entrante en perfiles públicos, y casi todas las nubes lo bloquean por defecto.

```
                       ping          traceroute         nmap
Respuesta clara        echo reply    ICMP time exceeded open / closed (RST)
Silencio significa     ¿apagado o    "* * *" ¿router    filtered: ¿firewall
                       filtrado?     que no contesta?   o host caído?
Cómo confirmar         nmap -Pn,     traceroute -T      -Pn, otro puerto,
                       curl al puerto (TCP al 443)      captura con tcpdump
```

La fila "Silencio significa" tiene un signo de pregunta en las tres columnas: ahí está la idea entera.

Ejemplo: `ping servidor.empresa.com` no responde, pero `curl -I https://servidor.empresa.com` devuelve `200 OK`. El servidor está vivo y el firewall bloquea ICMP. Al revés, un ping que sí responde tampoco prueba que el servicio funcione: la máquina puede estar encendida con el servidor web caído.

Analogía: que nadie conteste el timbre no prueba que la casa esté vacía; puede que el timbre esté desconectado.

```
Silencio (DROP) → ambiguo: apagado, filtrado o perdido.
Respuesta negativa (RST, unreachable) → prueba que algo vivo contestó.
Confirmar → otra herramienta, otro protocolo u otro puerto.
```

### dig

**dig** (domain information groper) es un cliente DNS de línea de comandos, parte de BIND, que hace consultas a un servidor DNS y muestra la respuesta completa, sección por sección. Es la herramienta preferida para diagnosticar DNS porque no oculta nada: muestra flags, TTL, servidor que contestó y tiempo de respuesta. En Windows no viene instalado; allí se usa `nslookup` o `Resolve-DnsName` de PowerShell.

Analogía: es preguntar en la guía telefónica pidiendo además ver quién te atendió, de qué libro sacó el número y cuánto tiempo vale ese dato.

Banderas clave:

```
dig ejemplo.com                     registro A, usando el DNS del sistema
dig ejemplo.com MX                  tipo de registro concreto (A, AAAA, MX, NS, TXT, CNAME, SOA)
dig @8.8.8.8 ejemplo.com            pregunta a un servidor concreto
dig +short ejemplo.com              solo la respuesta
dig -x 93.184.216.34                consulta inversa (PTR): IP a nombre
dig +trace ejemplo.com              sigue la resolución desde la raíz hasta el autoritativo
dig +noall +answer ejemplo.com      solo la sección de respuesta
dig axfr @ns1.ejemplo.com ejemplo.com   pide transferencia de zona completa
```

Ejemplo:

```
$ dig ejemplo.com A

; <<>> DiG 9.20.4 <<>> ejemplo.com A
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41275
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;ejemplo.com.                   IN      A

;; ANSWER SECTION:
ejemplo.com.            3600    IN      A       93.184.216.34

;; Query time: 24 msec
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
;; WHEN: Fri Oct 02 10:00:00 CST 2026
;; MSG SIZE  rcvd: 56
```

Cómo leerla: `status: NOERROR` es éxito; `NXDOMAIN` significa que el nombre no existe; `SERVFAIL` que el servidor no pudo resolver (a menudo DNSSEC roto o autoritativo caído); `REFUSED` que el servidor no te atiende. En `flags`, `qr` es respuesta, `rd` (recursion desired) que pediste recursión, `ra` que el servidor la ofrece, y `aa` (authoritative answer) aparece solo si contestó el servidor dueño de la zona. El `3600` es el TTL en segundos: durante una hora las cachés guardarán esa respuesta, por eso un cambio de DNS "tarda en propagarse".

Uso en seguridad: `dig TXT ejemplo.com` muestra SPF, `dig TXT _dmarc.ejemplo.com` la política DMARC, y un `axfr` que funciona desde fuera es una fuga de información: entrega todos los subdominios de la empresa.

```
dig → cliente DNS detallado (Linux/macOS, BIND).
NOERROR → éxito; NXDOMAIN → no existe; SERVFAIL → fallo del resolvedor.
aa → respondió el autoritativo; ra → el servidor hace recursión.
TTL → segundos que la respuesta vive en caché.
AXFR abierto → fuga de toda la zona.
```

### nslookup

**nslookup** es un cliente DNS disponible en Windows y Linux que consulta registros de un dominio, en modo de una línea o interactivo. Es más antiguo y menos detallado que `dig`, pero es el que viene en todo Windows, por eso es el que pide el examen para ese sistema.

Analogía: el mismo encargo de la guía telefónica, pero con un empleado que te da el número sin enseñarte el expediente.

Banderas y modos:

```
nslookup ejemplo.com                 registro A/AAAA con el DNS del sistema
nslookup ejemplo.com 8.8.8.8         pregunta a un servidor concreto
nslookup -type=MX ejemplo.com        tipo concreto (Windows y Linux)
nslookup 93.184.216.34               consulta inversa PTR
nslookup                             modo interactivo: > set type=NS, > server 8.8.8.8
```

Ejemplo en Windows:

```
C:\> nslookup ejemplo.com
Server:  one.one.one.one
Address:  1.1.1.1

Non-authoritative answer:
Name:    ejemplo.com
Addresses:  2606:2800:220:1:248:1893:25c8:1946
          93.184.216.34
```

Lectura: las dos primeras líneas dicen qué servidor DNS contestó; si ahí aparece algo que no reconoces, alguien cambió tu DNS (un malware o un DHCP falso). `Non-authoritative answer` significa que la respuesta vino de una caché, no del servidor dueño del dominio: es lo normal. `*** ... can't find ejemplo.com: Non-existent domain` es el NXDOMAIN de `dig`.

```
nslookup → cliente DNS de Windows y Linux, menos detallado.
dig → cliente DNS completo de Linux; no viene en Windows.
Resolve-DnsName → equivalente moderno en PowerShell.
Non-authoritative answer → respuesta desde caché, normal.
```

### netstat

**netstat** es un comando que lista las conexiones de red del equipo, los puertos que escuchan, el proceso dueño de cada uno, las tablas de rutas y estadísticas por protocolo. En Windows sigue siendo la herramienta estándar; en Linux está obsoleto (net-tools) y su reemplazo es `ss` (socket statistics), más rápido porque lee la información directo del kernel.

Existe para responder "¿qué está hablando este equipo y con quién?", la primera pregunta cuando se sospecha de un malware que abrió una puerta trasera o se conecta a un servidor de control.

Analogía: es el registro de llamadas de un teléfono: con quién estás hablando ahora, quién está esperando que lo llamen y desde qué aparato.

Banderas clave:

```
Windows netstat                Linux ss                 Qué hace
netstat -a                     ss -a                    todas: escuchando y establecidas
netstat -n                     ss -n                    números, sin resolver nombres (rápido)
netstat -o                     ss -p                    PID / proceso dueño
netstat -b                     ss -p                    nombre del ejecutable (admin)
netstat -p tcp                 ss -t / ss -u            solo TCP / solo UDP
netstat -an | findstr LISTEN   ss -ltn                  solo los que escuchan
netstat -r                     ip route                 tabla de rutas
netstat -s                     ss -s / nstat            estadísticas por protocolo
```

La combinación más usada en Linux es `ss -tulnp`: TCP, UDP, listening, numérico, proceso.

Ejemplo en Linux:

```
$ sudo ss -tunp
Netid State  Recv-Q Send-Q   Local Address:Port      Peer Address:Port  Process
tcp   ESTAB  0      0         192.168.1.10:51234    93.184.216.34:443   users:(("firefox",pid=2210,fd=88))
tcp   ESTAB  0      0         192.168.1.10:22        192.168.1.20:60122 users:(("sshd",pid=1830,fd=4))
tcp   ESTAB  0      0         192.168.1.10:43890   203.0.113.66:4444    users:(("update",pid=3377,fd=3))
```

Lectura: la primera línea es tu navegador en una web HTTPS; la segunda, alguien de tu red conectado por SSH a tu equipo (puerto local 22). La tercera es la sospechosa: un proceso llamado `update` hablando con una IP externa al puerto 4444, el puerto por defecto de Metasploit. El siguiente paso sería ver qué es ese binario: `ls -l /proc/3377/exe`.

En Windows:

```
C:\> netstat -ano | findstr ESTABLISHED
  TCP    192.168.1.10:49822   93.184.216.34:443   ESTABLISHED   5120
C:\> tasklist /FI "PID eq 5120"
Image Name        PID  Session Name  Mem Usage
msedge.exe       5120  Console       182,340 K
```

Los estados que muestran ambos (`LISTEN`, `ESTABLISHED`, `TIME_WAIT`, `CLOSE_WAIT`, `SYN_SENT`) son los de TCP explicados en [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/). Muchos `SYN_RECV` acumulados son la firma de un SYN flood.

```
netstat → conexiones, puertos y proceso dueño; estándar en Windows.
ss → reemplazo moderno en Linux; ss -tulnp el más usado.
netstat -ano → Windows con PID; tasklist da el nombre.
LISTEN → espera conexiones; ESTABLISHED → conexión activa.
Muchos SYN_RECV → posible SYN flood.
```

### route

**route** es un comando que muestra y modifica la tabla de rutas del sistema, es decir, la lista que decide por qué interfaz y hacia qué router sale cada paquete según su destino. En Windows se usa `route print` / `route add`; en Linux el antiguo `route -n` se reemplazó por `ip route`.

Existe porque un equipo con dos interfaces (Wi-Fi y VPN, o dos tarjetas en un servidor) necesita saber por dónde mandar cada cosa, y una ruta equivocada es la causa de muchos "llego a una red pero no a otra".

Analogía: es el cartel de una rotonda: "Centro, 2a salida; Aeropuerto, 3a; cualquier otro destino, autopista". La ruta por defecto es "cualquier otro destino".

Cómo decide: gana la ruta más específica (el prefijo más largo, longest prefix match). Para ir a `10.8.0.5`, una ruta a `10.8.0.0/24` gana sobre `10.0.0.0/8`, y ambas ganan sobre la ruta por defecto `0.0.0.0/0`. Si hay empate, gana la métrica más baja.

Banderas clave:

```
Linux                                         Windows                                     Qué hace
ip route                                      route print                                 ver la tabla
ip route get 8.8.8.8                          (Find-NetRoute -RemoteIPAddress 8.8.8.8)    qué ruta usaría un destino
ip route add 10.8.0.0/24 via 192.168.1.254    route add 10.8.0.0 mask 255.255.255.0 192.168.1.254   añadir ruta
ip route del 10.8.0.0/24                      route delete 10.8.0.0                       borrar
(persistente vía NetworkManager/systemd)      route -p add ...                            ruta persistente
```

Ejemplo:

```
$ ip route
default via 192.168.1.1 dev wlp2s0 proto dhcp metric 600
10.8.0.0/24 dev tun0 proto kernel scope link src 10.8.0.2
192.168.1.0/24 dev wlp2s0 proto kernel scope link src 192.168.1.10 metric 600
$ ip route get 10.8.0.5
10.8.0.5 dev tun0 src 10.8.0.2 uid 1000
```

Lectura: todo lo que no sea de las otras redes sale por la Wi-Fi hacia el router `192.168.1.1` (`default`); la red `10.8.0.0/24` va por el túnel VPN `tun0`; la red local se alcanza directamente (`scope link`, sin router). `ip route get` confirma que `10.8.0.5` saldrá por la VPN. En seguridad, la tabla de rutas revela si una VPN es "split tunnel" (solo algunas redes por el túnel, como aquí) o "full tunnel" (la ruta `default` apunta al túnel).

```
route / ip route → muestra y cambia la tabla de rutas.
Ruta por defecto (0.0.0.0/0) → destino para todo lo demás, apunta al gateway.
Longest prefix match → gana la ruta más específica; empate → menor métrica.
Split tunnel → solo algunas redes por la VPN; full tunnel → default por la VPN.
```

### arp

**arp** es un comando que muestra y gestiona la caché ARP, la tabla que asocia direcciones IP de la red local con direcciones MAC. ARP (Address Resolution Protocol) es el protocolo que pregunta por broadcast "¿quién tiene la IP 192.168.1.1? Díselo a 192.168.1.10" y recibe "192.168.1.1 está en `a0:b1:c2:d3:e4:f5`". En Linux moderno se usa `ip neigh`.

Existe porque dentro de una red local las tramas Ethernet se entregan por MAC, no por IP; antes de mandar algo al gateway, el equipo necesita su MAC.

Analogía: sabes el nombre de alguien en una fiesta (IP) pero no su cara (MAC); preguntas en voz alta "¿quién es Ana?" y ella levanta la mano.

Banderas clave:

```
Windows / Linux antiguo    Linux moderno              Qué hace
arp -a                     ip neigh                   ver la caché
arp -d 192.168.1.1         ip neigh del 192.168.1.1 dev eth0   borrar una entrada
arp -s IP MAC              ip neigh add IP lladdr MAC dev eth0 nud permanent   entrada estática
```

Ejemplo:

```
$ ip neigh
192.168.1.1 dev wlp2s0 lladdr a0:b1:c2:d3:e4:f5 REACHABLE
192.168.1.20 dev wlp2s0 lladdr 3c:22:fb:11:22:33 STALE
192.168.1.35 dev wlp2s0 lladdr a0:b1:c2:d3:e4:f5 REACHABLE
```

Lectura: `REACHABLE` es una entrada confirmada hace poco; `STALE` una vieja que se verificará en el próximo uso; `FAILED` que nadie contestó. La señal de alarma está en la primera y la tercera línea: dos IP distintas con la misma MAC. Si una de ellas es el gateway, es la firma clásica de ARP spoofing (ARP poisoning): un equipo de la red responde ARP falsos diciendo "el gateway soy yo" para que todo el tráfico pase por él, un ataque de intermediario. Se ve en [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/); las defensas son Dynamic ARP Inspection en el switch o entradas estáticas en equipos críticos.

```
ARP → resuelve IP a MAC dentro de la red local, por broadcast.
arp -a / ip neigh → ver la caché ARP.
REACHABLE / STALE / FAILED → confirmada / vieja / sin respuesta.
Misma MAC para dos IP (una el gateway) → posible ARP spoofing.
Dynamic ARP Inspection → defensa en el switch.
```

### tracert

**tracert** (Windows) y **traceroute** (Linux) son utilidades que descubren los routers por los que pasa un paquete hasta su destino y la latencia hasta cada uno. Funcionan con un truco del TTL: mandan paquetes con TTL 1, 2, 3...; el router donde el TTL llega a 0 descarta el paquete y devuelve un ICMP time exceeded con su propia IP, revelando quién es ese salto.

```
TTL=1 ──▶ Router1 (TTL→0) ──✗  Router1 responde "time exceeded" → salto 1 = Router1
TTL=2 ──▶ Router1 ──▶ Router2 (TTL→0) ──✗  → salto 2 = Router2
TTL=3 ──▶ Router1 ──▶ Router2 ──▶ Destino  → el destino responde → fin
```

Analogía: es mandar a un mensajero con la orden "camina 1 cuadra y llámame desde donde estés", luego "camina 2", luego 3, hasta que llega a la casa. Con las llamadas reconstruyes el camino.

Diferencia de sondas: `tracert` de Windows manda ICMP echo; `traceroute` de Linux manda por defecto UDP a puertos altos (33434 en adelante), y el destino se reconoce porque devuelve ICMP port unreachable. Como muchos firewalls bloquean uno u otro, Linux permite elegir: `-I` usa ICMP y `-T -p 443` usa TCP SYN al 443, que casi nunca está bloqueado. `tracepath` es una variante sin root que además descubre la MTU.

Banderas clave:

```
Linux traceroute           Windows tracert      Qué hace
traceroute -n host         tracert -d host      no resolver nombres (más rápido)
traceroute -m 20 host      tracert -h 20 host   máximo de saltos (defecto 30)
traceroute -w 2 host       tracert -w 2000 host tiempo de espera por sonda (s / ms)
traceroute -I host         (siempre ICMP)       usar ICMP
traceroute -T -p 443 host  (no existe)          usar TCP SYN
mtr host                   pathping host        traceroute continuo con % de pérdida por salto
```

Ejemplo en Windows:

```
C:\> tracert -d 1.1.1.1
Tracing route to 1.1.1.1 over a maximum of 30 hops

  1     1 ms    <1 ms    <1 ms  192.168.1.1
  2     9 ms     8 ms     9 ms  10.20.0.1
  3     *        *        *     Request timed out.
  4    11 ms    12 ms    11 ms  200.50.10.1
  5    12 ms    12 ms    12 ms  1.1.1.1

Trace complete.
```

Lectura: cada línea es un salto y las tres columnas son tres sondas. El salto 1 es tu router; el 2 una IP privada del proveedor (normal, red interna del ISP); el 3 con `* * *` es un router que no responde ICMP pero sí reenvía, porque el salto 4 aparece: no es un corte. Un corte real se ve como asteriscos desde un salto en adelante hasta el final. Un salto de latencia brusco que se mantiene en todos los siguientes (8 ms → 150 ms) indica el enlace lento; si solo un salto intermedio tiene latencia alta y los siguientes no, es ese router que contesta ICMP con baja prioridad, no un problema.

```
tracert (Windows) → sondas ICMP echo.
traceroute (Linux) → sondas UDP por defecto; -I ICMP; -T TCP.
* * * aislado → router que no contesta pero reenvía; * hasta el final → corte.
mtr / pathping → traceroute continuo con estadística por salto.
```

### nmap

**nmap** (Network Mapper) es un escáner de red de código abierto que descubre qué equipos están activos, qué puertos tienen abiertos, qué servicio y versión corre en cada puerto, qué sistema operativo tienen y, con scripts, si tienen ciertas vulnerabilidades. Lo usan atacantes para reconocimiento y defensores para inventariar y auditar su propia red: es el mismo análisis desde los dos lados.

Analogía: es recorrer un edificio de noche probando cada puerta: cuáles abren (open), cuáles están cerradas con llave pero alguien dice "aquí no" (closed) y cuáles tienen un guardia que no deja ni acercarse (filtered).

Fases de un escaneo:

```
1. Resolución DNS del objetivo
2. Host discovery (¿está vivo?)  ── ping ICMP + TCP SYN 443 + TCP ACK 80 + ICMP timestamp
                                     (en la LAN, solo ARP)
3. Port scan (¿qué puertos?)     ── por defecto los 1000 más comunes de TCP
4. -sV  version detection        ── habla con el servicio y compara con firmas
5. -O   OS detection             ── huella del stack TCP/IP (TTL, ventana, opciones)
6. -sC / --script  NSE           ── scripts en Lua: enumeración, vulnerabilidades
7. Salida (-oN, -oX, -oG, -oA)
```

Estados de puerto:

```
open              un servicio acepta conexiones (respondió SYN-ACK, o datos en UDP).
closed            el host respondió RST: el puerto es alcanzable pero nada escucha.
filtered          no hubo respuesta o llegó ICMP unreachable: un firewall lo bloquea.
unfiltered        alcanzable pero no se sabe si abierto (solo en escaneo ACK).
open|filtered     no se puede distinguir (UDP, FIN, NULL, Xmas sin respuesta).
closed|filtered   no se puede distinguir (solo en escaneo idle).
```

Tipos de escaneo:

```
-sS  SYN scan ("half-open", "stealth"). Defecto con root.
     Manda SYN; SYN-ACK → open (y responde RST sin completar); RST → closed; nada → filtered.
     Rápido y no completa la conexión, así que muchas aplicaciones no lo registran.

-sT  TCP connect scan. Defecto sin root.
     Usa connect() del sistema: completa el 3-way handshake. Más lento y queda en los logs.

-sU  UDP scan. Manda datagramas UDP vacíos o específicos del protocolo.
     Respuesta UDP → open; ICMP port unreachable → closed; nada → open|filtered.
     Muy lento porque los sistemas limitan los ICMP de error (Linux: 1 por segundo).

-sA  ACK scan. No dice si está abierto: mapea reglas de firewall.
     RST → unfiltered; nada → filtered. Distingue firewall con estado de uno sin estado.

-sN / -sF / -sX  NULL (sin flags), FIN, Xmas (FIN+PSH+URG).
     Según RFC 793, un puerto cerrado responde RST y uno abierto calla.
     Pueden pasar filtros sin estado; Windows responde RST a todo, así que no sirven contra él.

-sn  Solo descubrimiento de hosts, sin escaneo de puertos (antiguo -sP).
-Pn  Salta el descubrimiento: trata todos los hosts como vivos (útil si bloquean ping).
-sV  Detección de versión del servicio.
-O   Detección de sistema operativo.
-A   Agresivo: -sV + -O + -sC + --traceroute.
```

Diagrama de lo que hace `-sS` frente a un puerto abierto y uno cerrado:

```
Puerto abierto (22)                     Puerto cerrado (23)
nmap ── SYN ───────────▶ host           nmap ── SYN ───────────▶ host
nmap ◀────────── SYN-ACK ── host        nmap ◀────────────── RST ── host
nmap ── RST ───────────▶ host           → closed
→ open (nunca hubo ESTABLISHED)
```

Selección de puertos, velocidad y salida:

```
-p 22,80,443         puertos concretos
-p 1-1024            rango
-p-                  los 65535
--top-ports 100      los 100 más comunes
-F                   rápido: 100 puertos
-T0 .. -T5           plantilla de tiempo: paranoid, sneaky, polite, normal (T3, defecto), aggressive, insane
--min-rate 1000      al menos 1000 paquetes por segundo
-oN f.txt            salida normal
-oX f.xml            XML (para importar en otras herramientas)
-oG f.gnmap          "grepeable", una línea por host
-oA base             las tres a la vez
-v / -vv             más detalle mientras corre
--reason             muestra por qué asignó cada estado
```

NSE (Nmap Scripting Engine):

NSE es el motor de scripts en Lua de nmap, con más de 600 scripts agrupados en categorías: `default`, `safe`, `discovery`, `version`, `auth`, `brute`, `vuln`, `exploit`, `intrusive`, `dos`, `malware`. `-sC` ejecuta la categoría `default`. Ejemplos: `--script http-title`, `--script ssl-enum-ciphers -p 443`, `--script vuln`, `--script smb-os-discovery`. Las categorías `brute`, `exploit`, `intrusive` y `dos` pueden tumbar servicios o bloquear cuentas: solo en sistemas propios.

Ejemplo contra el host que el proyecto Nmap ofrece para practicar:

```
$ sudo nmap -sS -sV -O -T4 scanme.nmap.org
Starting Nmap 7.95 ( https://nmap.org ) at 2026-10-02 10:00 CST
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.072s latency).
Not shown: 996 closed tcp ports (reset)
PORT      STATE    SERVICE    VERSION
22/tcp    open     ssh        OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp    open     http       Apache httpd 2.4.7 ((Ubuntu))
9929/tcp  open     nping-echo Nping echo
31337/tcp open     tcpwrapped
Device type: general purpose
Running: Linux 4.X|5.X
OS details: Linux 4.15 - 5.19

Nmap done: 1 IP address (1 host up) scanned in 9.81 seconds
```

Cómo leerla: `Host is up (0.072s latency)` viene del descubrimiento. `Not shown: 996 closed tcp ports (reset)` resume los 996 puertos del top 1000 que respondieron RST. En la tabla, `SERVICE` es lo que dice `/etc/services` para ese número y `VERSION` es lo que el servicio realmente respondió; la versión es lo que se cruza con bases de CVE (OpenSSH 6.6 y Apache 2.4.7 son de 2014, vulnerables a fallos conocidos). `tcpwrapped` significa que el puerto completó el handshake pero cerró sin decir nada (lo protege un TCP wrapper o un filtro de aplicación). La detección de SO es una estimación por huella, no una certeza.

Descubrir equipos en tu LAN:

```
$ sudo nmap -sn 192.168.1.0/24
Nmap scan report for 192.168.1.1
Host is up (0.0021s latency).
MAC Address: A0:B1:C2:D3:E4:F5 (TP-Link Technologies)
Nmap scan report for 192.168.1.20
Host is up (0.0090s latency).
MAC Address: 3C:22:FB:11:22:33 (Apple)
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.10 seconds
```

En la red local nmap usa ARP para descubrir, que ningún firewall de host puede bloquear sin aislarse de la red, y muestra el fabricante deducido de la MAC.

> [!TIP]
> Receta de examen: `-sS` si eres root (sigiloso, half-open), `-sT` si no; `-sU` para UDP (lento, open|filtered); `-sn` solo hosts; `-Pn` si bloquean ping; `-sV` versiones; `-O` SO; `-A` todo junto; `-p-` los 65535 puertos.

```
nmap → descubre hosts, puertos, servicios, versiones y SO.
-sS → SYN half-open, defecto con root; -sT → connect completo, defecto sin root.
-sU → UDP, lento, open|filtered si no hay respuesta.
-sA → mapea firewall (filtered / unfiltered), no apertura.
-sN/-sF/-sX → flags raros; inútiles contra Windows.
-sn → solo hosts; -Pn → no hacer descubrimiento.
-sV / -O / -A → versión / SO / todo.
open → acepta; closed → RST; filtered → sin respuesta o ICMP unreachable.
NSE → scripts Lua; -sC = categoría default.
-T0..T5 → velocidad; T3 por defecto.
```

### tcpdump

**tcpdump** es un capturador de paquetes de línea de comandos para Linux, macOS y BSD que lee el tráfico de una interfaz (o de un archivo pcap), lo filtra con expresiones BPF y lo muestra decodificado en texto o lo guarda en pcap para analizarlo después. Usa la biblioteca libpcap, la misma base de Wireshark y de muchos IDS. En Windows el equivalente nativo es `pktmon` y, en la práctica, `dumpcap`/Wireshark.

Existe porque las demás herramientas te dicen lo que el sistema cree que pasa; tcpdump te enseña lo que de verdad pasó por el cable. Es la prueba final: si el paquete no aparece en la captura, no salió.

Analogía: es una cámara de seguridad en la puerta del edificio que graba a todo el que entra y sale; el filtro BPF es decirle "graba solo a los que llevan uniforme de mensajero".

Banderas clave:

```
-i eth0       interfaz donde capturar (-i any: todas; -D lista las disponibles)
-n            no resolver IP a nombres; -nn tampoco puertos a nombres (más rápido y claro)
-c 100        parar tras 100 paquetes
-w f.pcap     guardar en archivo (binario, para Wireshark); no muestra en pantalla
-r f.pcap     leer de un archivo en vez de la red
-v / -vv      más detalle de cabeceras (TTL, longitud, checksum)
-A            contenido en ASCII (ver HTTP en claro)
-X            contenido en hexadecimal + ASCII
-e            mostrar cabecera de capa 2 (MAC origen/destino)
-s 0          capturar paquetes completos (0 = 262144 bytes, el defecto actual)
-tttt         marcas de tiempo con fecha legible
-C 100 -W 5   rotar archivos de 100 MB, máximo 5 (captura larga sin llenar el disco)
```

Filtros BPF:

Un **filtro BPF** (Berkeley Packet Filter) es una expresión que el kernel compila y aplica a cada paquete antes de copiarlo a tcpdump, de modo que lo que no coincide ni siquiera se captura. Va al final del comando, entre comillas simples. Se construye con primitivas:

```
Tipo        host 10.0.0.5     net 10.0.0.0/24     port 53     portrange 8000-8100
Dirección   src host 10.0.0.5     dst port 443     (sin src/dst: cualquiera de los dos)
Protocolo   tcp   udp   icmp   arp   ip6   ether host aa:bb:cc:dd:ee:ff
Lógica      and (&&)   or (||)   not (!)   y paréntesis para agrupar
Tamaño      greater 1000   less 64
Bytes       tcp[tcpflags] & tcp-syn != 0     ip[8] < 5   (byte 8 de IP = TTL)
```

Recetas:

```
tcpdump -nn -i eth0 host 192.168.1.20                       todo de o hacia un equipo
tcpdump -nn -i eth0 'port 53'                               DNS
tcpdump -nn -i eth0 'tcp port 80 or tcp port 443'           web
tcpdump -nn -i eth0 'net 192.168.1.0/24 and not port 22'    la LAN, sin tu propio SSH
tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'   solo SYN iniciales (escaneos, SYN flood)
tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-rst != 0'          RST (puertos cerrados, cortes)
tcpdump -nn -i eth0 icmp                                     ping y traceroute
tcpdump -nn -i eth0 arp                                      ARP (detectar spoofing)
tcpdump -nn -A -i eth0 'tcp port 21'                         credenciales FTP en claro (laboratorio)
```

Cuando se captura por SSH, `not port 22` es obligatorio: si no, tcpdump captura su propia salida, que genera más tráfico, que se captura... en bucle.

Cómo leer una línea:

```
10:00:01.150923 IP 192.168.1.10.51234 > 93.184.216.34.443: Flags [P.], seq 1:518, ack 1, win 502, length 517
│               │  │                    │                    │            │          │      │        │
hora            │  origen.puerto        destino.puerto       flags        bytes      ACK    ventana  datos
                protocolo de red                             [P.] PSH+ACK 1 a 517
```

Flags en tcpdump: `[S]` SYN, `[S.]` SYN-ACK, `[.]` ACK, `[P.]` PSH-ACK (datos), `[F.]` FIN-ACK, `[R]` RST, `[R.]` RST-ACK. Los números de secuencia son relativos al inicio (empiezan en 1) salvo que se pida `-S`.

Ejemplo: alguien hace un escaneo SYN contra tu máquina.

```
$ sudo tcpdump -nn -i eth0 'tcp[tcpflags] == tcp-syn' -c 6
10:05:00.000101 IP 192.168.1.66.41000 > 192.168.1.10.22: Flags [S], seq 2944, win 1024, options [mss 1460], length 0
10:05:00.000140 IP 192.168.1.66.41000 > 192.168.1.10.80: Flags [S], seq 2944, win 1024, options [mss 1460], length 0
10:05:00.000172 IP 192.168.1.66.41000 > 192.168.1.10.443: Flags [S], seq 2944, win 1024, options [mss 1460], length 0
10:05:00.000201 IP 192.168.1.66.41000 > 192.168.1.10.3389: Flags [S], seq 2944, win 1024, options [mss 1460], length 0
10:05:00.000233 IP 192.168.1.66.41000 > 192.168.1.10.8080: Flags [S], seq 2944, win 1024, options [mss 1460], length 0
10:05:00.000260 IP 192.168.1.66.41000 > 192.168.1.10.21: Flags [S], seq 2944, win 1024, options [mss 1460], length 0
```

Lectura: un solo origen, el mismo puerto origen y el mismo seq, apuntando a seis puertos distintos en 160 microsegundos, con ventana 1024: es la huella de `nmap -sS`. Un navegador real usaría un puerto origen distinto por conexión y una ventana de decenas de miles.

> [!WARNING]
> El filtro de captura de tcpdump (BPF: `tcp port 80`, `host 10.0.0.5`) y el filtro de visualización de Wireshark (`tcp.port == 80`, `ip.addr == 10.0.0.5`) tienen sintaxis distinta y momentos distintos: el primero decide qué se graba y lo descartado se pierde; el segundo solo oculta lo ya grabado.

```
tcpdump → captura en línea de comandos con libpcap.
-i / -nn / -c / -w / -r → interfaz / sin nombres / cantidad / guardar / leer.
-A / -X → contenido ASCII / hex.
BPF → filtro compilado en el kernel: host, net, port, src/dst, tcp/udp/icmp, and/or/not.
tcp[tcpflags] & tcp-syn != 0 → paquetes con SYN.
[S] SYN, [S.] SYN-ACK, [.] ACK, [P.] datos, [F.] FIN, [R] RST.
not port 22 → evita capturar tu propia sesión SSH.
```

### iptables

**iptables** es la herramienta de Linux para definir las reglas del firewall del kernel (Netfilter): qué paquetes se aceptan, se descartan, se rechazan o se traducen (NAT). Hoy la mayoría de distribuciones usa por debajo nftables, su sucesor, y el comando `iptables` suele ser una capa de compatibilidad (`iptables-nft`); frontends como `ufw` o `firewalld` generan las reglas por ti. En Windows el equivalente es Windows Defender Firewall (`netsh advfirewall`, `New-NetFirewallRule`).

Aparece entre las herramientas de diagnóstico porque una regla de firewall es la causa silenciosa de muchos "no conecta": el servicio escucha (`ss` lo muestra), pero el paquete muere antes de llegar.

Analogía: es el guardia de la entrada con una lista de instrucciones que lee de arriba abajo; aplica la primera que coincide y, si ninguna coincide, aplica la política por defecto.

Estructura: tablas que contienen cadenas que contienen reglas.

```
Tabla filter (decidir si pasa)    cadenas INPUT (hacia este equipo), OUTPUT (desde él),
                                  FORWARD (que lo atraviesa, si hace de router)
Tabla nat (traducir direcciones)  PREROUTING (DNAT, redirigir puertos), POSTROUTING (SNAT/MASQUERADE)
Tabla mangle                      modificar campos (TTL, marcas)

Paquete entrante ─▶ PREROUTING ─▶ ¿es para mí? ─sí─▶ INPUT ─▶ proceso local
                                       │no                          │
                                       ▼                            ▼
                                    FORWARD ─▶ POSTROUTING ◀── OUTPUT
```

Objetivos (targets): `ACCEPT` (pasa), `DROP` (se descarta en silencio, el otro lado ve timeout), `REJECT` (se descarta avisando con RST o ICMP unreachable, el otro lado ve "connection refused"), `LOG` (registra y sigue evaluando).

Banderas clave:

```
iptables -L -n -v --line-numbers     listar reglas con contadores y números
iptables -S                          listar en formato de comandos
iptables -A INPUT ...                añadir al final
iptables -I INPUT 1 ...              insertar en la posición 1
iptables -D INPUT 3                  borrar la regla 3
iptables -P INPUT DROP               política por defecto de la cadena
-p tcp --dport 22                    protocolo y puerto destino
-s 192.168.1.0/24                    origen
-i eth0                              interfaz de entrada
-m conntrack --ctstate ESTABLISHED,RELATED   respuestas a conexiones ya permitidas
iptables-save > reglas / iptables-restore < reglas    guardar y restaurar
```

Ejemplo: firewall mínimo de un servidor que solo ofrece SSH desde la LAN y HTTPS a todos.

```
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -P INPUT DROP
```

```
$ sudo iptables -L INPUT -n -v --line-numbers
Chain INPUT (policy DROP 1520 packets, 91200 bytes)
num   pkts bytes target  prot opt in  out  source           destination
1      812  60K  ACCEPT  all  --  lo  *    0.0.0.0/0        0.0.0.0/0
2      45K  38M  ACCEPT  all  --  *   *    0.0.0.0/0        0.0.0.0/0   ctstate RELATED,ESTABLISHED
3       30  1800 ACCEPT  tcp  --  *   *    192.168.1.0/24   0.0.0.0/0   tcp dpt:22
4      900  54K  ACCEPT  tcp  --  *   *    0.0.0.0/0        0.0.0.0/0   tcp dpt:443
5       12   1008 ACCEPT icmp --  *   *    0.0.0.0/0        0.0.0.0/0   icmptype 8
```

Lectura: el orden importa, se evalúa de arriba abajo. La regla 2 deja pasar las respuestas a conexiones ya aceptadas (firewall con estado). Los contadores `pkts` dicen qué regla está trabajando: si un cliente no conecta al 443 y el contador de la regla 4 no sube, el paquete no está llegando al servidor (problema antes, en la red). Los 1520 paquetes de la política DROP son todo lo demás que se descartó, típicamente escaneos de internet. El diseño de firewalls se amplía en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

Equivalente nftables del mismo listado: `nft list ruleset`.

```
iptables → reglas del firewall Netfilter de Linux.
nftables → sucesor; iptables-nft es la capa de compatibilidad.
INPUT / OUTPUT / FORWARD → hacia mí / desde mí / a través de mí.
DROP → silencio (timeout); REJECT → aviso (connection refused).
Primera regla que coincide gana; si ninguna, política por defecto.
ESTABLISHED,RELATED → firewall con estado.
ufw / firewalld → frontends que generan las reglas.
```

### Packet Sniffers

Un **packet sniffer** (capturador de paquetes) es una herramienta, de software o hardware, que copia los paquetes que pasan por una interfaz de red para guardarlos o mostrarlos. tcpdump, `dumpcap`, `tshark` y el propio Wireshark (en su parte de captura) son sniffers. Sirve para diagnosticar fallos y detectar ataques, y en manos de un atacante, para robar lo que viaja en claro.

Analogía: es un micrófono puesto en la línea telefónica: graba todo lo que pasa, entienda o no el idioma.

Cómo funciona y qué puede ver: la tarjeta en modo promiscuo entrega todas las tramas que llegan a ella, no solo las suyas. Pero lo que llega depende de la red:

```
Hub (antiguo)            reenvía todo a todos → el sniffer ve el tráfico de toda la red.
Switch                   reenvía cada trama solo al puerto del destino → el sniffer ve
                         solo su tráfico, más broadcast y multicast.
Port mirroring (SPAN)    el switch copia el tráfico de unos puertos a otro → así se monta un IDS.
Network TAP              dispositivo físico en el cable que copia todo, sin depender del switch.
Wi-Fi modo monitor       la tarjeta capta tramas 802.11 de todas las redes del canal.
```

Por eso en una red con switch un atacante necesita primero desviar el tráfico hacia sí (ARP spoofing, ver "arp") para poder espiarlo. Y por eso los protocolos cifrados (ver [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/)) dejan al sniffer viendo solo metadatos: quién habla con quién, cuándo y cuánto.

Ejemplo: en un laboratorio propio, capturar el login de un servidor FTP de pruebas.

```
$ sudo tcpdump -nn -A -i eth0 'tcp port 21' | grep -E 'USER|PASS'
USER admin
PASS Verano2026!
```

Detección: un sniffer pasivo no emite nada, así que es difícil de ver; se detectan modos promiscuos con `ip link` (bandera `PROMISC`) en tus propios equipos, y en la red con anomalías ARP.

```
Packet sniffer → copia paquetes de una interfaz (captura).
Modo promiscuo → la tarjeta acepta tramas no dirigidas a ella.
Switch → solo ves tu tráfico; SPAN o TAP → copia del tráfico ajeno.
Modo monitor → captura Wi-Fi de todas las redes del canal.
```

### Port Scanners

Un **port scanner** (escáner de puertos) es una herramienta que envía sondas a un rango de puertos de uno o varios equipos y clasifica cada puerto según la respuesta (abierto, cerrado, filtrado). nmap es el estándar; otros son `masscan` (escanea internet entero en minutos, a millones de paquetes por segundo), `rustscan` (descubre puertos rápido y pasa el resultado a nmap), `netcat` (`nc -zv host 20-25`, comprobación rápida) y Angry IP Scanner (gráfico).

A diferencia del sniffer, que escucha, el escáner pregunta activamente: genera tráfico y por eso deja rastro en los logs y en un IDS.

Analogía: el sniffer es quien escucha conversaciones en el pasillo; el escáner es quien va tocando todas las puertas.

Usos defensivos: inventario de activos (¿qué hay realmente en mi red?), comprobar que el firewall aplica lo que se cree, detectar servicios no autorizados (un empleado que levantó un servidor en el 8080), y verificar el resultado de un parche. Usos ofensivos: la fase de reconocimiento de cualquier ataque, en la táctica Reconnaissance/Discovery de MITRE ATT&CK (ver [`13-frameworks-de-amenazas`](../13-frameworks-de-amenazas/)).

Ejemplo con netcat, sin nmap disponible:

```
$ nc -zv -w 1 192.168.1.50 20-25
nc: connect to 192.168.1.50 port 20 (tcp) failed: Connection refused
nc: connect to 192.168.1.50 port 21 (tcp) failed: Connection refused
Connection to 192.168.1.50 22 port [tcp/ssh] succeeded!
nc: connect to 192.168.1.50 port 23 (tcp) failed: Connection refused
nc: connect to 192.168.1.50 port 24 (tcp) failed: Connection refused
nc: connect to 192.168.1.50 port 25 (tcp) failed: Connection refused
```

`-z` solo comprueba (no manda datos), `-v` detallado, `-w 1` espera un segundo. "Connection refused" es un RST (closed); un timeout sería filtered.

```
Port scanner → pregunta activamente a muchos puertos y clasifica la respuesta.
nmap → estándar, detallado; masscan → velocidad a escala internet.
nc -zv → comprobación rápida de unos pocos puertos.
Sniffer → escucha (pasivo); scanner → pregunta (activo, deja rastro).
```

### Protocol Analyzers

Un **protocol analyzer** (analizador de protocolos) es una herramienta que toma paquetes capturados y los decodifica campo por campo según cada protocolo, reconstruye conversaciones y permite filtrarlos y buscar en ellos. Wireshark es el analizador de referencia; `tshark` es su versión de línea de comandos y Zeek convierte el tráfico en logs estructurados para un SOC. La diferencia con un sniffer es el entendimiento: el sniffer guarda bytes, el analizador te dice que esos bytes son un ClientHello de TLS 1.3 con SNI `ejemplo.com`.

Analogía: el sniffer es la grabadora; el analizador es el traductor que transcribe la grabación, separa a quién habla y marca las frases raras.

Wireshark como protocol analyzer:

Wireshark tiene más de 3000 disectores (decodificadores de protocolo). Su ventana tiene tres paneles: la lista de paquetes (uno por línea), el detalle del paquete seleccionado como árbol por capas (Frame, Ethernet, IP, TCP, HTTP...) y los bytes en hexadecimal.

Filtros de visualización (display filters), con sintaxis propia `protocolo.campo operador valor`:

```
ip.addr == 192.168.1.20                  origen o destino
ip.src == 192.168.1.20 && tcp.port == 443
tcp.flags.syn == 1 && tcp.flags.ack == 0 SYN iniciales
tcp.flags.reset == 1                     RST
tcp.analysis.retransmission              retransmisiones (pérdida de paquetes)
dns.flags.rcode == 3                     respuestas NXDOMAIN
http.request.method == "POST"            formularios enviados
tls.handshake.type == 1                  ClientHello
tls.handshake.extensions_server_name contains "ejemplo"   SNI
arp.duplicate-address-detected           posible ARP spoofing
frame contains "password"                texto en cualquier parte
!(arp || dns || icmp)                    quitar ruido
```

Funciones que convierten capturas en respuestas:

- Follow > TCP Stream: reconstruye la conversación completa como texto (ver una sesión HTTP o FTP entera).
- Statistics > Conversations y Endpoints: quién habla más con quién; un equipo interno que manda gigas a una IP desconocida salta aquí.
- Statistics > Protocol Hierarchy: qué porcentaje es cada protocolo; un 30 % de DNS es anómalo (posible túnel DNS).
- Analyze > Expert Information: avisos automáticos (retransmisiones, resets, checksums malos).
- File > Export Objects > HTTP/SMB: extrae archivos transferidos en claro (útil en forense de malware, ver [`16-respuesta-a-incidentes-y-forense`](../16-respuesta-a-incidentes-y-forense/)).
- Columna "Time" con delta: detectar beaconing, conexiones periódicas exactas de un malware a su servidor de control (cada 60 s, por ejemplo).

Ejemplo: con `tshark`, extraer de una captura los sitios HTTPS que visitó un equipo, leyendo el SNI.

```
$ tshark -r captura.pcap -Y 'tls.handshake.type == 1' -T fields -e ip.src -e tls.handshake.extensions_server_name
192.168.1.10    www.ejemplo.com
192.168.1.10    cdn.ejemplo.com
192.168.1.20    update.proveedor.net
```

Lectura: aunque el tráfico va cifrado, el SNI del ClientHello viaja en claro (salvo ECH) y revela los nombres de los sitios. El flujo habitual es capturar con tcpdump en el servidor (`-w`) y analizar después en Wireshark en tu equipo.

```
Protocol analyzer → decodifica, reconstruye y filtra lo capturado.
Wireshark → analizador gráfico; tshark → su versión de consola.
Display filter → ip.addr == x, tcp.port == 443, tls.handshake.type == 1.
Follow TCP Stream → conversación completa.
Conversations / Protocol Hierarchy → quién habla con quién / qué protocolos.
Export Objects → extraer archivos transferidos.
Zeek → convierte tráfico en logs para un SOC.
```

### Captura, sondeo e interpretación son tres trabajos distintos

> [!IMPORTANT]
> Packet sniffer, port scanner y protocol analyzer no son sinónimos: el sniffer escucha y guarda, el scanner pregunta y clasifica, el analyzer interpreta lo capturado. Una misma herramienta puede hacer dos (Wireshark captura y analiza), pero el examen pregunta por la función.

Las tres categorías son roles distintos dentro del análisis de red, definidos por qué hacen con el tráfico, no por el nombre del programa.

```
                     Packet sniffer       Port scanner         Protocol analyzer
Genera tráfico       no (pasivo)          sí (activo)          no
Qué produce          paquetes crudos/pcap lista de puertos     decodificación por campos
Pregunta que         ¿qué pasó por        ¿qué escucha         ¿qué significa
responde             el cable?            ese equipo?          ese tráfico?
Ejemplos             tcpdump, dumpcap     nmap, masscan, nc    Wireshark, tshark, Zeek
Lo detecta un IDS    difícil              sí                   no aplica (offline)
```

La fila "Genera tráfico" separa a los tres: solo el scanner toca al objetivo, y por eso solo él necesita autorización explícita para usarse contra sistemas ajenos y solo él aparece en los logs del otro lado.

Ejemplo: pregunta de examen "un analista necesita ver qué contraseña se envió en un login FTP sospechoso dentro de un pcap" → protocol analyzer (Wireshark, Follow TCP Stream). "Necesita saber qué servicios expone un servidor nuevo" → port scanner. "Necesita grabar el tráfico de un servidor durante una hora" → packet sniffer.

Límite: las fronteras se cruzan en la práctica (tcpdump decodifica cabeceras básicas; nmap con `-sV` interpreta respuestas), así que la respuesta es la función principal.

```
Sniffer → escucha y guarda (pasivo).
Scanner → pregunta y clasifica (activo).
Analyzer → interpreta lo guardado.
```

## Recursos para aprender y practicar

### Videos

- [Command Line Tools - CompTIA Network+ N10-009 - 5.5](https://www.youtube.com/watch?v=iYojIK1173k) — Professor Messer; ping, traceroute, nslookup, dig, tcpdump, netstat, ipconfig, arp y route con la mirada del examen. Sirve para casi todos los nodos.
- [Software Tools - CompTIA Network+ N10-009 - 5.5](https://www.youtube.com/watch?v=b81_GeoN12I) — Professor Messer; protocol analyzers, port scanners y otras herramientas de software. Sirve para "Packet Sniffers", "Port Scanners" y "Protocol Analyzers".
- [Nmap Tutorial to find Network Vulnerabilities](https://www.youtube.com/watch?v=4t4kBkMsDbQ) — NetworkChuck; tipos de escaneo, modo sigiloso, detección de SO, scripts NSE, visto también en Wireshark. Sirve para "nmap".
- [tcpdump - Traffic Capture & Analysis](https://www.youtube.com/watch?v=1lDfCRM6dWk) — HackerSploit; captura, filtros y guardado en pcap. Sirve para "tcpdump".
- [Learn Wireshark! Tutorial for BEGINNERS](https://www.youtube.com/watch?v=OU-A2EmVrKQ) — Chris Greer; interfaz, filtros y primeros análisis. Sirve para "Protocol Analyzers".
- [Traceroute (tracert) Explained - Network Troubleshooting](https://www.youtube.com/watch?v=up3bcBLZS74) — PowerCert Animated Videos; el truco del TTL animado. Sirve para "tracert".
- [How TCP Works - The Handshake](https://www.youtube.com/watch?v=HCHFX5O1IaQ) — Chris Greer; leer un handshake en Wireshark, base para interpretar escaneos. Sirve para "tcpdump" y "Protocol Analyzers".

### Lectura y documentación

- [Nmap Reference Guide](https://nmap.org/book/man.html), [Port Scanning Basics](https://nmap.org/book/man-port-scanning-basics.html) y [Port Scanning Techniques](https://nmap.org/book/man-port-scanning-techniques.html) — documentación oficial: estados de puerto y cada tipo de escaneo.
- [tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html) y [pcap-filter(7)](https://www.tcpdump.org/manpages/pcap-filter.7.html) — banderas de tcpdump y sintaxis BPF completa.
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/) y [Display Filter Reference](https://www.wireshark.org/docs/dfref/) — manual y todos los campos filtrables.
- [ip(8)](https://man7.org/linux/man-pages/man8/ip.8.html), [ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html), [ping(8)](https://man7.org/linux/man-pages/man8/ping.8.html), [tracepath(8)](https://man7.org/linux/man-pages/man8/tracepath.8.html) e [iptables(8)](https://man7.org/linux/man-pages/man8/iptables.8.html) — man pages de Linux.
- [BIND 9 manual pages (dig, nslookup)](https://bind9.readthedocs.io/en/latest/manpages.html) — referencia oficial de `dig`.
- Microsoft Learn, comandos de Windows: [ipconfig](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ipconfig), [ping](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ping), [tracert](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert), [netstat](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netstat), [nslookup](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/nslookup), [arp](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/arp), [route](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/route_ws2008).
- [iptables — ArchWiki](https://wiki.archlinux.org/title/Iptables) — tablas, cadenas y ejemplos prácticos.
- [RFC 792 — ICMP](https://www.rfc-editor.org/rfc/rfc792) — los mensajes que usan ping y traceroute.

### Práctica

- [Nmap: The Basics](https://tryhackme.com/room/nmap01) y [Further Nmap](https://tryhackme.com/room/furthernmap) — TryHackMe; descubrimiento, tipos de escaneo, NSE y evasión. Practica "nmap" y "Port Scanners".
- [TShark](https://tryhackme.com/room/tshark) — TryHackMe, gratis; capturar y filtrar desde la terminal con la versión CLI de Wireshark. Practica "tcpdump" y "Packet Sniffers".
- [Carnage](https://tryhackme.com/room/c2carnage) — TryHackMe, gratis; investigar un pcap real de una infección con Wireshark. Practica "Protocol Analyzers".
- [Network Enumeration with Nmap](https://academy.hackthebox.com/course/preview/network-enumeration-with-nmap) — HTB Academy; host discovery, escaneo, NSE, evasión de firewall, con laboratorios. Practica "nmap".
- [Intro to Network Traffic Analysis](https://academy.hackthebox.com/course/preview/intro-to-network-traffic-analysis) — HTB Academy; tcpdump y Wireshark aplicados. Practica "tcpdump" y "Protocol Analyzers".
- [Malware-Traffic-Analysis.net training exercises](https://www.malware-traffic-analysis.net/training-exercises.html) — pcaps de infecciones reales con preguntas y respuestas. Practica "Protocol Analyzers" con mirada de SOC.
- Ejercicio en casa: en una terminal `sudo tcpdump -nn -i any 'host 192.168.1.X'` y en otra `nmap -sS -p 1-100 192.168.1.X` contra otra máquina tuya; identifica en la captura los `[S]`, `[S.]` y `[R]` y relaciónalos con los estados open y closed. Repite con `-sT` y nota el `[.]` extra que completa el handshake. Practica "nmap" y "tcpdump".
- Ejercicio en casa: corre `ss -tulnp` (Linux) o `netstat -ano` (Windows), identifica el proceso de cada puerto que escucha y decide si debería estar expuesto; luego bloquea uno con `iptables`/`ufw` y comprueba desde otra máquina con `nc -zv` que pasa de open a filtered. Practica "netstat", "iptables" y "Port Scanners".
- Ejercicio en casa: compara `traceroute 1.1.1.1`, `traceroute -I 1.1.1.1` y `sudo traceroute -T -p 443 1.1.1.1`, y explica por qué los asteriscos cambian entre uno y otro. Practica "tracert".

## Cuadro resumen

Todo lo visto, en una línea por término.

Troubleshooting Tools: configuración y alcance

```
Orden de diagnóstico → configuración, gateway, ARP, internet por IP, ruta, DNS, servicio, puerto, captura, firewall.
ipconfig → configuración IP en Windows; /all, /release, /renew, /flushdns.
ip addr / ifconfig → equivalente Linux moderno / obsoleto.
169.254.x.x (APIPA) → el DHCP no respondió.
ping → ICMP echo request/reply; latencia, pérdida, TTL.
TTL recibido → 64 Linux, 128 Windows, 255 equipos de red, menos los saltos.
Silencio (DROP) → ambiguo: apagado, filtrado o perdido; confirmar por otra vía.
arp / ip neigh → caché IP-MAC; misma MAC para dos IP (una el gateway) → ARP spoofing.
route / ip route → tabla de rutas; gana el prefijo más largo, luego la menor métrica.
tracert → sondas ICMP (Windows); traceroute → UDP por defecto, -I ICMP, -T TCP (Linux).
* * * aislado → router que no contesta; * hasta el final → corte.
```

Troubleshooting Tools: DNS y conexiones

```
dig → cliente DNS detallado; @servidor, tipo, +short, +trace, -x.
nslookup → cliente DNS de Windows y Linux, menos detallado.
NOERROR / NXDOMAIN / SERVFAIL → éxito / no existe / fallo del resolvedor.
aa → respuesta autoritativa; Non-authoritative → desde caché.
AXFR abierto → fuga de toda la zona.
netstat → conexiones, puertos y PID; netstat -ano en Windows.
ss → reemplazo Linux; ss -tulnp el más usado.
Muchos SYN_RECV → posible SYN flood.
```

Troubleshooting Tools: escaneo, captura y firewall

```
nmap → hosts, puertos, servicios, versiones, SO y scripts.
-sS → SYN half-open, defecto con root; -sT → connect completo, sin root.
-sU → UDP, lento; -sA → mapea firewall; -sN/-sF/-sX → flags raros, inútiles contra Windows.
-sn → solo hosts; -Pn → sin descubrimiento; -sV / -O / -A → versión / SO / todo.
open → acepta; closed → RST; filtered → sin respuesta o ICMP unreachable.
Top 1000 → puertos por defecto; -p- → los 65535; -T3 → velocidad por defecto.
NSE → scripts Lua; -sC = categoría default.
tcpdump → captura con libpcap; -i, -nn, -c, -w, -r, -A, -X.
BPF → host, net, port, src/dst, tcp/udp/icmp, and/or/not, tcp[tcpflags].
[S] SYN, [S.] SYN-ACK, [.] ACK, [P.] datos, [F.] FIN, [R] RST.
Filtro de captura (BPF) → decide qué se graba; display filter de Wireshark → oculta lo grabado.
iptables → reglas Netfilter; INPUT / OUTPUT / FORWARD.
DROP → timeout; REJECT → connection refused.
Primera regla que coincide gana; si ninguna, política por defecto.
nftables → sucesor de iptables; ufw / firewalld → frontends.
```

Troubleshooting Tools: categorías de herramientas

```
Packet sniffer → copia paquetes de una interfaz (pasivo).
Modo promiscuo → acepta tramas ajenas; switch → solo ves lo tuyo; SPAN/TAP → copia ajena.
Port scanner → pregunta a muchos puertos y clasifica (activo, deja rastro).
nmap / masscan / nc -zv → detallado / escala internet / comprobación rápida.
Protocol analyzer → decodifica, reconstruye y filtra lo capturado.
Wireshark → analizador gráfico; tshark → consola; Zeek → logs para SOC.
Follow TCP Stream / Conversations / Export Objects → conversación / quién con quién / archivos.
Sniffer escucha, scanner pregunta, analyzer interpreta.
```
