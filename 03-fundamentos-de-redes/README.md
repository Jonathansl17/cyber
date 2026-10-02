# Fundamentos de redes

## Conceptos previos

- Red: conjunto de dispositivos conectados que intercambian datos siguiendo reglas comunes.
- Host: cualquier equipo con dirección en la red que envía o recibe datos (PC, servidor, teléfono, impresora).
- Protocolo: conjunto de reglas que fija el formato y el orden de los mensajes entre dos equipos (HTTP, TCP, IP, Ethernet).
- Cabecera (header): bloque de datos de control que un protocolo pega delante de lo que transporta; dice de dónde viene, a dónde va y cómo tratarlo.
- Payload (carga útil): lo que va dentro de la cabecera, es decir, el dato que realmente se quiere entregar.
- Dirección MAC: identificador de 48 bits grabado en la tarjeta de red (`aa:bb:cc:11:22:33`); sirve dentro de la misma red local.
- Dirección IP: identificador lógico del host para llegar a él a través de varias redes; se estudia a fondo en [`04-direccionamiento-ip`](../04-direccionamiento-ip/).
- Puerto: número de 16 bits (0-65535) que identifica a qué programa de un host va dirigido un dato; HTTP usa 80, HTTPS 443, SSH 22. Detalle en [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/).
- Ancho de banda: cuántos bits por segundo caben en un enlace (1 Gbps = mil millones de bits por segundo).
- Latencia: cuánto tarda un dato en ir de un extremo al otro, en milisegundos.
- Broadcast: mensaje enviado a todos los equipos de la red local a la vez.
- MTU (Maximum Transmission Unit): tamaño máximo de datos que cabe en una trama; en Ethernet son 1500 bytes.

## Understand the OSI Model

### El modelo OSI y sus siete capas

El **modelo OSI** (Open Systems Interconnection) es un modelo de referencia conceptual que divide la comunicación entre dos equipos en siete capas, cada una con una sola responsabilidad y que solo habla con la capa de arriba y la de abajo. Lo publicó la ISO en 1984 (norma ISO/IEC 7498-1).

Existe porque, antes de él, cada fabricante tenía su pila cerrada (IBM SNA, DECnet) y un equipo de una marca no hablaba con el de otra. Partir el problema en capas permite cambiar una pieza sin tocar el resto: puedes pasar de cable de cobre a Wi-Fi (capa 1-2) sin que el navegador (capa 7) se entere. Hoy nadie implementa OSI tal cual; se usa como idioma común para diagnosticar ("es un problema de capa 2") y para ubicar ataques y defensas.

Analogía: enviar un regalo por correo. Tú escribes la carta (aplicación), la traduces al idioma del destinatario (presentación), acuerdas con él que vas a mandar una serie de cartas (sesión), numeras los paquetes para que se puedan reordenar (transporte), escribes la dirección postal de la ciudad destino (red), el cartero del barrio lo lleva a la siguiente oficina (enlace) y el camión lo mueve físicamente por la carretera (física). Cada persona de la cadena solo mira su parte del sobre.

Las siete capas, de arriba (7) a abajo (1), con su PDU. La **PDU** (Protocol Data Unit, unidad de datos de protocolo) es el nombre que recibe el bloque de datos en cada capa:

```
N  Capa           PDU                    Qué hace                                   Ejemplos
7  Application    Datos (data)           Da servicios de red a los programas         HTTP, DNS, SMTP, SSH, FTP
6  Presentation   Datos                  Traduce formato, cifra, comprime            TLS, UTF-8, JPEG, gzip
5  Session        Datos                  Abre, mantiene y cierra el diálogo          RPC, NetBIOS, SMB sessions
4  Transport      Segmento (TCP)         Entrega de extremo a extremo entre          TCP, UDP
                  Datagrama (UDP)        programas, por puerto; fiabilidad
3  Network        Paquete                Direcciona y enruta entre redes distintas   IP, ICMP, OSPF, BGP
2  Data Link      Trama (frame)          Entrega dentro de la misma red local, por   Ethernet, 802.11, ARP, 802.1Q
                                         dirección MAC; detecta errores (FCS)
1  Physical       Bits                   Convierte bits en señales: voltaje, luz,    Cat6, fibra, radio 2,4/5 GHz
                                         ondas de radio
```

Reglas mnemotécnicas, en inglés porque así aparecen en los exámenes: de la capa 1 a la 7, "Please Do Not Throw Sausage Pizza Away"; de la 7 a la 1, "All People Seem To Need Data Processing".

Cómo leer cada capa por dentro:

- Capa 1, Physical: no entiende de direcciones; solo pone y lee unos y ceros en el medio. Define conectores (RJ45), tipo de cable (Cat6 llega a 10 Gbps hasta 55 m), longitudes máximas (100 m para cobre Ethernet) y frecuencias.
- Capa 2, Data Link: agrupa bits en tramas, les pone MAC de origen y destino y un FCS (Frame Check Sequence, una suma de verificación de 4 bytes) para detectar si llegó corrupta. Solo sirve dentro de un mismo segmento local: la MAC no cruza routers.
- Capa 3, Network: pone IP de origen y destino y decide por qué camino ir (enrutamiento). Es la que permite saltar de red en red hasta el otro lado del mundo. Cada router decrementa el TTL (Time To Live, contador de saltos) en 1 y descarta el paquete si llega a 0, para evitar bucles infinitos.
- Capa 4, Transport: identifica el programa con el puerto y decide la calidad de entrega. TCP garantiza orden y reenvía lo perdido (con el handshake de tres pasos SYN, SYN-ACK, ACK); UDP no garantiza nada pero es más rápido (DNS, voz, juegos).
- Capa 5, Session: lleva el control del diálogo: quién habla, cuándo empieza, cuándo termina, cómo retomar si se corta.
- Capa 6, Presentation: asegura que ambos lados entiendan el formato: codificación de caracteres, compresión, cifrado. TLS suele ubicarse aquí, aunque en la práctica corre entre la 4 y la 7.
- Capa 7, Application: la interfaz de red que usa el programa. Ojo: el navegador no es la capa 7; el protocolo HTTP que usa el navegador sí.

Ejemplo: abres `https://ejemplo.com`. La capa 7 arma un `GET /` de HTTP, la 6 lo cifra con TLS, la 5 mantiene la sesión TLS abierta, la 4 lo mete en un segmento TCP hacia el puerto 443, la 3 le pone IP destino 93.184.215.14, la 2 le pone la MAC del router de tu casa y la 1 lo convierte en ondas de radio de Wi-Fi.

> [!WARNING]
> Las capas 5, 6 y 7 se confunden porque en TCP/IP están fundidas en una sola; en un examen, "cifrado y formato" es capa 6, "abrir y cerrar el diálogo" es capa 5 y "el protocolo que usa la aplicación" es capa 7.

```
Capa 7 Application → protocolo de la aplicación (HTTP, DNS).
Capa 6 Presentation → formato, cifrado, compresión.
Capa 5 Session → abrir, mantener y cerrar el diálogo.
Capa 4 Transport → puertos; TCP fiable / UDP rápido; PDU segmento.
Capa 3 Network → IP y enrutamiento entre redes; PDU paquete.
Capa 2 Data Link → MAC y entrega en la red local; PDU trama.
Capa 1 Physical → bits como señales en el medio.
```

### Modelo TCP/IP comparado con OSI

El **modelo TCP/IP** es el modelo de capas que de verdad implementa Internet, definido en el RFC 1122, y que agrupa la comunicación en cuatro capas: Application, Transport, Internet y Link.

Existe porque OSI se diseñó en comité y llegó tarde: cuando se publicó, TCP/IP ya corría en ARPANET y funcionaba. TCP/IP es más práctico: junta en una sola capa lo que OSI separa cuando en la realidad lo resuelve el mismo software (el navegador hace HTTP, formato y sesión a la vez).

Analogía: OSI es el plano detallado de una casa con cada habitación separada; TCP/IP es la casa construida, donde la cocina, el comedor y la sala resultaron ser un solo ambiente abierto.

```
        OSI (7)                 TCP/IP (4, RFC 1122)        Protocolos típicos
  +------------------+     +-----------------------+
  | 7 Application    |     |                       |
  | 6 Presentation   | --> |  Application          |    HTTP, DNS, TLS, SSH, SMTP
  | 5 Session        |     |                       |
  +------------------+     +-----------------------+
  | 4 Transport      | --> |  Transport            |    TCP, UDP
  +------------------+     +-----------------------+
  | 3 Network        | --> |  Internet             |    IP, ICMP
  +------------------+     +-----------------------+
  | 2 Data Link      | --> |  Link                 |    Ethernet, Wi-Fi, ARP
  | 1 Physical       |     |  (Network Access)     |
  +------------------+     +-----------------------+
```

Algunos libros (Cisco, Tanenbaum) usan un modelo híbrido de cinco capas que separa Link en Data Link y Physical. Las tres versiones dicen lo mismo; solo cambia cuánto se agrupa.

Ejemplo: cuando alguien dice "un firewall de capa 7" usa la numeración OSI aunque el equipo corra TCP/IP. La numeración de OSI ganó como vocabulario; los protocolos de TCP/IP ganaron como implementación.

```
OSI → modelo de referencia, 7 capas, vocabulario para diagnosticar.
TCP/IP → modelo implementado en Internet, 4 capas (RFC 1122).
Application TCP/IP → capas 5 + 6 + 7 de OSI.
Link TCP/IP → capas 1 + 2 de OSI.
```

### Encapsulamiento

El **encapsulamiento** es el proceso por el que cada capa, al enviar, toma lo que le entrega la capa de arriba y le añade su propia cabecera (y a veces una cola) sin mirar ni modificar lo que hay dentro. El proceso inverso en el receptor, quitar cabeceras de abajo hacia arriba, se llama **desencapsulamiento**.

Resuelve el problema de la independencia: el router solo necesita leer la cabecera IP para decidir, sin entender HTTP; el switch solo lee la cabecera Ethernet. Cada capa del emisor dialoga con la misma capa del receptor a través de su cabecera, como si el resto no existiera.

Analogía: muñecas rusas. La carta va en un sobre (TCP), el sobre en una caja con la dirección de la ciudad (IP), la caja en la saca del cartero del barrio (Ethernet). En cada oficina se abre la saca, se mira la dirección de la caja y se mete en una saca nueva para el siguiente tramo; la carta nunca se abre por el camino.

```
Emisor (baja)                                         Receptor (sube)

[ Datos HTTP: "GET / ..." ]                      7    se entrega al programa
[ TCP 20B | Datos ]                  = segmento  4    quita TCP, mira puerto 443
[ IP 20B | TCP | Datos ]             = paquete   3    quita IP, ¿es para mí?
[ Eth 14B | IP | TCP | Datos | FCS 4B ] = trama  2    quita Ethernet, revisa FCS
 0101100101110...                    = bits      1    lee señal del cable
```

Cifras que conviene recordar: cabecera Ethernet de 14 bytes más 4 de FCS, cabecera IPv4 de 20 bytes mínimo, cabecera TCP de 20 bytes mínimo, UDP de 8 bytes. Con una MTU de 1500 bytes, a TCP le quedan 1500 − 20 − 20 = 1460 bytes de datos por segmento (el MSS, Maximum Segment Size).

Un punto que confunde: en cada salto, el router quita la trama Ethernet vieja y construye una nueva con MACs nuevas, pero deja intactas las IP de origen y destino (salvo que haga NAT). Por eso la MAC cambia en cada tramo y la IP se mantiene de extremo a extremo.

Ejemplo: capturar en tu propia máquina con `tcpdump -e` (muestra la cabecera de capa 2) el inicio de una conexión web:

```
$ sudo tcpdump -i eth0 -e -n -c 1 'tcp port 80'
12:00:01.000000 aa:bb:cc:00:00:01 > 11:22:33:44:55:66, ethertype IPv4 (0x0800),
  length 74: 192.168.1.10.51000 > 93.184.215.14.80: Flags [S], seq 1000,
  win 64240, length 0
```

Se lee de fuera hacia dentro: MAC origen y destino y `ethertype IPv4` son la capa 2; las dos IP son la capa 3; los puertos `51000` y `80` y la bandera `[S]` (SYN) son la capa 4. `length 74` = 14 de Ethernet + 20 de IP + 40 de TCP (20 fijos y 20 de opciones); `length 0` al final indica que aún no viajan datos HTTP.

```
Encapsulamiento → cada capa añade su cabecera al bajar.
Desencapsulamiento → cada capa quita su cabecera al subir.
MSS → 1500 − 20 IP − 20 TCP = 1460 bytes de datos.
MAC → cambia en cada salto; IP → se mantiene de extremo a extremo.
```

### Un dispositivo trabaja en la capa más alta que lee

> [!IMPORTANT]
> Lo que define la capa de un dispositivo o de un ataque es la cabecera más alta que necesita leer o falsificar; con esa regla se ubica cualquier equipo o ataque sin memorizar listas.

Esta regla es un criterio de clasificación que asigna a cada equipo y a cada ataque la capa de la cabecera más alta que procesa. Un hub no lee nada, solo repite señales: capa 1. Un switch lee la MAC: capa 2. Un router lee la IP: capa 3. Un firewall que filtra por puerto lee TCP/UDP: capa 4. Un WAF que bloquea `' OR 1=1` lee HTTP: capa 7.

```
Capa  Dispositivos                          Ataques típicos (detalle en 11 y 12)
7     WAF, proxy, IDS de aplicación,        SQL injection, XSS, phishing,
      firewall NGFW (de nueva generación)   HTTP flood, DNS spoofing
6     Terminador TLS                        SSL stripping, downgrade de cifrado
5     (lo maneja el software)               Session hijacking
4     Firewall stateful, balanceador L4     SYN flood, port scanning, UDP flood
3     Router, switch de capa 3              IP spoofing, ICMP flood, BGP hijacking
2     Switch, bridge, access point, NIC     ARP spoofing, MAC flooding, VLAN hopping
1     Hub, repetidor, cable, módem,         Wiretapping (pinchar el cable),
      transceptor                           jamming de radio, conectar un equipo intruso
```

La fila que prueba la idea es la de la izquierda: subiendo de capa, cada dispositivo lee una cabecera más. Un ataque de capa 2 como ARP spoofing solo funciona dentro de la red local, porque las tramas no cruzan routers; un SYN flood de capa 4 sí puede venir de cualquier parte de Internet.

Ejemplo: un atacante en la misma Wi-Fi de una cafetería envía respuestas ARP falsas diciendo "la IP del router 192.168.1.1 está en mi MAC". Es capa 2, así que la defensa también está en capa 2 (Dynamic ARP Inspection en el switch) o arriba (TLS cifra los datos aunque pasen por el intruso). Un firewall que solo mira puertos no lo ve.

Límite de la regla: varios equipos modernos trabajan en muchas capas a la vez (un router doméstico es switch, router, firewall, servidor DHCP y access point en una caja). Se clasifica cada función por separado, no la caja.

Los ataques se estudian en [`11-ataques-y-amenazas`](../11-ataques-y-amenazas/) y [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/); las defensas, en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

```
Hub → capa 1; repite bits a todos los puertos.
Switch → capa 2; reenvía por MAC.
Router → capa 3; enruta por IP.
Firewall stateful → capa 4; filtra por IP, puerto y estado de conexión.
WAF → capa 7; filtra contenido HTTP.
```

## Network Topologies

Una **topología de red** es la forma en que se conectan los nodos de una red. La topología física describe por dónde van los cables; la lógica describe cómo circulan realmente los datos, y no tienen por qué coincidir. Las cuatro clásicas son Star, Ring, Mesh y Bus; las redes reales suelen ser híbridas (una estrella de estrellas, por ejemplo).

### Star

La **topología Star (estrella)** es una topología física en la que cada nodo se conecta con su propio cable a un dispositivo central, normalmente un switch, y todo el tráfico pasa por ese centro.

Existe porque aísla fallos: si se corta el cable de un PC, solo ese PC se cae. Es la topología de casi toda LAN Ethernet actual y del Wi-Fi (el access point es el centro).

Analogía: una rotonda con calles que salen hacia cada casa. Si una calle se bloquea, solo esa casa queda aislada; si la rotonda se bloquea, nadie se mueve.

```
        PC1   PC2
          \   /
   PC6 -- SWITCH -- PC3
          /   \
        PC5   PC4
```

Criterios: n nodos necesitan n cables. Añadir un equipo es enchufar un cable más. El centro es el punto único de fallo (SPOF, single point of failure) y también el lugar natural para monitorear: un puerto espejo (SPAN) del switch ve todo el tráfico.

Ejemplo: una oficina con 24 PCs y un switch de 24 puertos usa 24 cables. Si muere el cable del PC 7, los otros 23 siguen; si muere el switch, caen los 24. Por eso en empresas se ponen dos switches centrales redundantes.

```
Star → todos al centro; n cables; falla un cable cae un nodo; falla el centro caen todos.
```

### Ring

La **topología Ring (anillo)** es una topología en la que cada nodo se conecta exactamente a dos vecinos formando un círculo cerrado, y los datos dan la vuelta pasando de nodo en nodo.

Existió para evitar colisiones: en Token Ring (IEEE 802.5, 4 o 16 Mbps) un mensaje especial llamado token circula por el anillo y solo el nodo que lo tiene puede transmitir, como el turno de palabra. FDDI usaba un anillo doble de fibra a 100 Mbps. Hoy sobrevive en redes de operadores metropolitanos (anillos SONET/SDH de fibra) y en redes industriales.

Analogía: una mesa redonda donde solo habla quien tiene el micrófono, y el micrófono se pasa siempre al de la derecha.

```
     A ---- B
    /        \
   F          C
    \        /
     E ---- D
```

Criterios: n nodos necesitan n enlaces. En un anillo simple, un solo corte tumba todo; con anillo doble (uno en cada sentido), un corte se rodea dando la vuelta por el otro lado. Cada nodo intermedio regenera y reenvía la señal, así que un nodo comprometido ve todo lo que pasa por él.

Ejemplo: un anillo de 6 sedes con 6 tramos de fibra. Si una excavadora corta el tramo B-C en un anillo simple, se cae la red; en un anillo doble, el tráfico de B a C da la vuelta por A-F-E-D-C y llega con algo más de latencia.

```
Ring → cada nodo con dos vecinos; n enlaces; token para turnarse; un corte tumba un anillo simple.
```

### Mesh

La **topología Mesh (malla)** es una topología en la que los nodos tienen varios caminos entre sí: en malla completa (full mesh) cada nodo se conecta directamente con todos los demás; en malla parcial, solo algunos pares tienen enlace directo.

Existe para la redundancia: si un enlace cae, el tráfico sigue por otro camino. Es la forma del núcleo de Internet (los routers de los operadores se interconectan en malla parcial) y de los sistemas Wi-Fi mesh domésticos.

Analogía: una red de carreteras entre ciudades. Si cierran una autopista, llegas igual dando un rodeo por otra.

```
Full mesh, 4 nodos (6 enlaces)

   A ------- B
   | \     / |
   |   \ /   |
   |   / \   |
   | /     \ |
   C ------- D
```

Criterio numérico: una malla completa de n nodos necesita n(n−1)/2 enlaces. Con 4 nodos son 6; con 5 son 10; con 10 son 45; con 50 son 1225. Por eso la malla completa solo se usa entre pocos nodos críticos y en lo demás se usa malla parcial.

Ejemplo: una empresa con 5 sedes conecta las 5 en malla completa con 10 enlaces VPN; cualquier sede llega a otra aunque caigan hasta 3 de los 4 enlaces que salen de ella.

```
Mesh → muchos caminos; full mesh n(n−1)/2 enlaces; máxima redundancia y máximo coste.
```

### Bus

La **topología Bus** es una topología en la que todos los nodos se cuelgan de un único cable compartido, la troncal (backbone), y cada mensaje que se envía llega a todos los nodos.

Fue la primera Ethernet: 10BASE5 (cable coaxial grueso, segmentos de hasta 500 m) y 10BASE2 (coaxial fino, hasta 185 m). Cada extremo del cable lleva un terminador de 50 ohmios que absorbe la señal para que no rebote. Como el medio es compartido, dos nodos que hablan a la vez chocan; Ethernet resolvía eso con CSMA/CD (escuchar antes de hablar y, si hay colisión, esperar un tiempo aleatorio y reintentar).

Analogía: una línea de teléfono compartida en un pueblo: todos oyen todas las conversaciones y si dos hablan a la vez nadie entiende nada.

```
 [T]==+=======+=======+=======+==[T]     T = terminador 50 ohm
      |       |       |       |
     PC1     PC2     PC3     PC4
```

Criterios: barata (un solo cable), pero un corte o un terminador suelto tumba todo el segmento, y el rendimiento cae al crecer el número de nodos porque hay más colisiones. Desde el punto de vista de seguridad, cualquier nodo puede escuchar todo el tráfico.

Ejemplo: 10 PCs en un coaxial de 150 m. Si se afloja el conector T del PC 5, los 10 pierden red, y encontrar el punto exacto obliga a revisar el cable entero.

> [!NOTE]
> Una red con hub es estrella física pero bus lógico: el hub repite cada trama por todos los puertos, así que todos ven todo y hay colisiones; el switch rompió ese bus lógico al enviar cada trama solo al puerto destino.

```
Star → centro único; fallo aislado salvo el centro; LAN y Wi-Fi actuales.
Ring → círculo; token; anillo doble para sobrevivir a un corte.
Mesh → varios caminos; n(n−1)/2 enlaces en full mesh; núcleo de Internet.
Bus → un cable compartido; terminadores; colisiones; histórica.
Topología física → por dónde van los cables.
Topología lógica → cómo circulan realmente los datos.
```

## Understand these

Las redes se clasifican por el área geográfica que cubren y por el medio que usan. De menor a mayor alcance: PAN (personal, unos pocos metros, Bluetooth), LAN, MAN y WAN; la WLAN es una LAN sin cables. Se explica primero la LAN porque las otras se definen respecto a ella.

### LAN

Una **LAN** (Local Area Network, red de área local) es una red que conecta dispositivos dentro de un área pequeña, como una casa, una oficina o un edificio, bajo un mismo dueño y administración.

Existe para compartir recursos cercanos (impresoras, servidores de archivos, la salida a Internet) con mucha velocidad y poca latencia. Usa Ethernet (IEEE 802.3) por cable: hoy 1 Gbps al escritorio y 10 Gbps o más entre switches, con latencias por debajo de 1 ms. Todo lo que hay en ella pertenece a la organización, así que también la controla y la asegura ella.

Analogía: los pasillos de un edificio de oficinas: dentro te mueves rápido y sin pedir permiso a nadie de fuera.

Por dentro, una LAN es normalmente un dominio de broadcast: un mensaje broadcast (como una pregunta ARP) llega a todos sus equipos. Para partirla en dominios más pequeños se usan VLAN (ver [`04-direccionamiento-ip`](../04-direccionamiento-ip/)).

Ejemplo: una oficina de 40 empleados con un switch de 48 puertos, un servidor de archivos y un router hacia Internet; la copia de un archivo de 1 GB entre dos PCs tarda unos 10 segundos a 1 Gbps.

```
LAN → área pequeña, un dueño, Ethernet, 1-10 Gbps, < 1 ms.
```

### MAN

Una **MAN** (Metropolitan Area Network, red de área metropolitana) es una red que interconecta varias LAN dentro de una ciudad o área metropolitana, típicamente de 5 a 50 km.

Existe porque una organización con varias sedes en la misma ciudad (un ayuntamiento, una universidad con varios campus, un hospital con clínicas) necesita unirlas con más velocidad de la que da Internet y sin depender de ella. Suele construirse con anillos de fibra óptica y Metro Ethernet que ofrece un operador local o la propia municipalidad.

Analogía: el sistema de metro de una ciudad, que une barrios distintos sin salir de la ciudad.

Ejemplo: una universidad con 3 campus a 8 km entre sí los une con un anillo de fibra de 10 Gbps alquilado al operador local; para los estudiantes, los 3 campus funcionan como una sola red.

```
MAN → varias LAN en una ciudad, 5-50 km, fibra/Metro Ethernet, operador local.
```

### WAN

Una **WAN** (Wide Area Network, red de área amplia) es una red que conecta LAN y MAN separadas por grandes distancias, entre ciudades, países o continentes, usando enlaces que normalmente pertenecen a operadores de telecomunicaciones.

Existe porque ninguna empresa puede tender su propio cable entre San José y Madrid: alquila capacidad a quien sí lo tiene. Tecnologías típicas: líneas dedicadas, MPLS (red privada del operador), VPN sobre Internet y SD-WAN, que elige automáticamente el mejor enlace disponible. La latencia sube a decenas o cientos de ms (un viaje transatlántico ronda los 80-150 ms ida y vuelta) y el ancho de banda es más caro. Internet es la WAN más grande que existe.

Analogía: las autopistas y rutas aéreas entre países: no son tuyas, pagas peaje para usarlas.

Ejemplo: una empresa con sede en Costa Rica y sucursal en España une ambas LAN con una VPN de sitio a sitio sobre Internet; un ping entre las dos sedes muestra unos 120 ms frente a menos de 1 ms dentro de cada oficina.

```
$ ping -c 3 10.20.0.1
64 bytes from 10.20.0.1: icmp_seq=1 ttl=62 time=121 ms
64 bytes from 10.20.0.1: icmp_seq=2 ttl=62 time=119 ms
64 bytes from 10.20.0.1: icmp_seq=3 ttl=62 time=120 ms
```

```
WAN → entre ciudades/países, enlaces de operadores, MPLS/VPN/SD-WAN, latencia alta.
```

### WLAN

Una **WLAN** (Wireless LAN, red de área local inalámbrica) es una LAN que conecta los dispositivos por ondas de radio en lugar de cables, siguiendo el estándar IEEE 802.11, conocido comercialmente como Wi-Fi.

Existe para dar movilidad: portátiles y teléfonos se conectan sin cable dentro del área de cobertura de un access point (AP), que hace de puente entre la radio y la LAN cableada. Opera en las bandas de 2,4 GHz (más alcance, más interferencia), 5 GHz y 6 GHz (Wi-Fi 6E/7, más velocidad, menos alcance); la cobertura de un AP en interiores ronda los 30-50 m. La red se identifica por su SSID (el nombre que ves al buscar redes). Como el aire es un medio compartido, usa CSMA/CA: evita colisiones en lugar de detectarlas.

Analogía: una conversación en voz alta dentro de una sala: todos los que estén en la sala pueden oírla, por eso hace falta hablar en clave.

Por esa razón la WLAN tiene un problema de seguridad que la LAN cableada no tiene: cualquiera dentro del alcance de la señal recibe las tramas, aunque no esté dentro del edificio. Se protege con cifrado WPA2 o, mejor, WPA3 (ver [`10-criptografia`](../10-criptografia/)); una red abierta sin cifrado expone todo lo que no vaya cifrado en capas superiores.

Ejemplo: una casa con un router Wi-Fi en 5 GHz da unos 500 Mbps a 5 m del AP, pero baja a 50 Mbps dos paredes más allá; el vecino, a 20 m, aún ve el SSID.

> [!TIP]
> Para clasificar un tipo de red pregunta dos cosas: cuánta distancia cubre (edificio, ciudad, país) y si va por cable o por radio; la WLAN no es un tamaño distinto, es una LAN por radio.

```
LAN → edificio, cable Ethernet, un dueño.
WLAN → LAN por radio (802.11), AP, SSID, WPA2/WPA3.
MAN → ciudad, une varias LAN.
WAN → países y continentes, enlaces alquilados; Internet es la mayor.
PAN → pocos metros alrededor de una persona (Bluetooth).
```

## Basics of NAS and SAN

### NAS

Un **NAS** (Network Attached Storage, almacenamiento conectado a la red) es un dispositivo de almacenamiento con su propio sistema operativo y sistema de archivos que se conecta a la LAN y comparte carpetas con los demás equipos a nivel de archivo.

Existe para centralizar archivos: en vez de que cada PC guarde sus documentos en su disco, todos los guardan en una caja común con discos redundantes (RAID) y copias de seguridad. Los clientes acceden con protocolos de compartición de archivos sobre la red normal: SMB/CIFS (TCP 445, típico de Windows) o NFS (TCP 2049, típico de Linux). El NAS decide dónde va cada bloque en sus discos; el cliente solo pide "dame el archivo `informe.pdf`".

Analogía: una biblioteca: pides un libro por su título y el bibliotecario sabe en qué estante está; tú nunca tocas los estantes.

Ejemplo: una oficina de 15 personas compra un NAS de 4 discos de 4 TB en RAID 5 (12 TB útiles) y monta una carpeta compartida en cada PC:

```
$ sudo mount -t nfs 192.168.1.50:/compartido /mnt/nas
$ ls /mnt/nas
contabilidad  marketing  plantillas
```

Riesgo típico: como el NAS está en la misma LAN y habla SMB, un ransomware en un solo PC puede cifrar todas las carpetas compartidas a las que ese usuario tiene escritura. Por eso se combinan permisos mínimos con instantáneas (snapshots) de solo lectura.

```
NAS → caja en la LAN que comparte archivos; SMB 445 / NFS 2049; nivel archivo.
```

### SAN

Una **SAN** (Storage Area Network, red de área de almacenamiento) es una red dedicada y de alta velocidad que conecta servidores con cabinas de discos y les entrega almacenamiento a nivel de bloque, de modo que cada servidor ve ese espacio como si fuera un disco local suyo.

Existe para servidores que necesitan rendimiento y disponibilidad que un NAS no da: bases de datos, clústeres de virtualización (ver [`06-virtualizacion`](../06-virtualizacion/)). Va separada de la LAN para que el tráfico de disco no compita con el de usuarios. Tecnologías: Fibre Channel (8, 16, 32 o 64 Gbps, con sus propios switches), iSCSI (SCSI encapsulado en TCP, puerto 3260, sobre Ethernet) y FCoE. La cabina divide su espacio en LUN (Logical Unit Number, cada "disco virtual" que se asigna a un servidor); el servidor lo formatea con su propio sistema de archivos (NTFS, ext4, VMFS).

Analogía: alquilar un trastero: te dan un espacio vacío con llave, y tú decides cómo poner las estanterías y qué guardar en cada una.

Seguridad: se controla quién ve cada LUN con zoning (en los switches Fibre Channel) y LUN masking (en la cabina). Un error de zoning que presenta la misma LUN a dos servidores que no se coordinan corrompe el sistema de archivos.

Ejemplo: un servidor Linux descubre una cabina iSCSI y recibe una LUN de 500 GB que aparece como un disco más:

```
$ sudo iscsiadm -m discovery -t sendtargets -p 10.0.50.10
10.0.50.10:3260,1 iqn.2026-01.lab.cabina:lun-db
$ lsblk | grep sdb
sdb      8:16   0  500G  0 disk
```

### NAS entrega archivos, SAN entrega discos

> [!IMPORTANT]
> La diferencia entre NAS y SAN no es el tamaño ni el precio, sino quién maneja el sistema de archivos: en el NAS lo maneja el propio NAS y el cliente pide archivos; en la SAN lo maneja el servidor y la red solo le entrega bloques crudos.

Esta distinción es el criterio que separa almacenamiento a nivel de archivo de almacenamiento a nivel de bloque. Un bloque es un trozo de tamaño fijo del disco (por ejemplo 4 KB) sin nombre; un archivo es un conjunto de bloques al que un sistema de archivos le pone nombre, carpeta y permisos.

```
                     DAS (disco local)   NAS                  SAN
Dónde está           Dentro del servidor En la LAN            Red dedicada
Red                  Ninguna (SATA/SAS)  Ethernet compartida  Fibre Channel / iSCSI
Protocolo            -                   SMB 445, NFS 2049    FC, iSCSI 3260
El cliente ve        Un disco            Una carpeta          Un disco
Sistema de archivos  Lo maneja el host   Lo maneja el NAS     Lo maneja el host
Uso típico           Un PC               Archivos de oficina  Bases de datos, VMs
```

La fila "Sistema de archivos" es la que lo decide todo: DAS y SAN coinciden (el host formatea), y el NAS es el único que lo hace por su cuenta. Por eso una SAN se siente como un disco local que está lejos, y un NAS como una carpeta compartida.

Límite: hay equipos unificados que ofrecen NAS y SAN a la vez desde la misma cabina; se clasifica cada servicio por separado según qué entrega.

Ejemplo: 10 usuarios editando hojas de cálculo compartidas, NAS. Un clúster de 3 hipervisores que necesita mover máquinas virtuales entre ellos sin copiar discos, SAN.

```
NAS → nivel archivo; el NAS gestiona el sistema de archivos; carpeta compartida.
SAN → nivel bloque; el servidor gestiona el sistema de archivos; disco remoto.
DAS → disco conectado directamente al servidor, sin red.
LUN → disco lógico que la SAN presenta a un servidor.
```

## Recursos para aprender y practicar

### Videos

- [Understanding the OSI Model - CompTIA Network+ N10-009 - 1.1](https://www.youtube.com/watch?v=AYgXr1dynKU) — Professor Messer; las siete capas con ejemplos reales, para Understand the OSI Model.
- [OSI Model: A Practical Perspective - Networking Fundamentals - Lesson 2a](https://www.youtube.com/watch?v=LkolbURrtTs) — Practical Networking; qué hace cada capa en la práctica y cómo se encapsula, para OSI y encapsulamiento.
- [what is TCP/IP and OSI? // FREE CCNA // EP 3](https://www.youtube.com/watch?v=CRdL1PcherM) — NetworkChuck; origen de ambos modelos y por qué se sigue hablando de OSI, para la comparación OSI vs TCP/IP.
- [Packet Traveling - How Packets Move Through a Network](https://www.youtube.com/watch?v=rYodcvhh7b8) — Practical Networking; cómo cambian las cabeceras de capa 2 y se mantienen las de capa 3 en cada salto, para encapsulamiento y dispositivos por capa.
- [Network Topologies - CompTIA Network+ N10-009 - 1.6](https://www.youtube.com/watch?v=3ARTjvpZCoQ) — Professor Messer; estrella, malla e híbridas, para Network Topologies.
- [Network Topologies (Star, Bus, Ring, Mesh, Ad hoc, Infrastructure, & Wireless Mesh Topology)](https://www.youtube.com/watch?v=zbqrNg4C98U) — PowerCert Animated Videos; las cuatro topologías del roadmap animadas, para Star, Ring, Mesh y Bus.
- [Network Types: LAN, WAN, PAN, CAN, MAN, SAN, WLAN](https://www.youtube.com/watch?v=4_zSIXb7tLQ) — PowerCert Animated Videos; todos los tipos de red por alcance, para MAN, LAN, WAN y WLAN.
- [NAS vs SAN - Network Attached Storage vs Storage Area Network](https://www.youtube.com/watch?v=3yZDDr0JKVc) — PowerCert Animated Videos; diferencia archivo frente a bloque, para Basics of NAS and SAN.

### Lectura y documentación

- [RFC 1122: Requirements for Internet Hosts](https://www.rfc-editor.org/rfc/rfc1122) — define las capas Link, Internet, Transport y Application del modelo TCP/IP.
- [What is the OSI Model?](https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/) — Cloudflare Learning; las siete capas y qué ataques DDoS afectan a cada una.
- [OSI Model](https://www.practicalnetworking.net/series/packet-traveling/osi-model/) — Practical Networking; artículo sobre la función de cada capa, complemento del video.
- [What is a WAN?](https://www.cloudflare.com/learning/network-layer/what-is-a-wan/) — Cloudflare Learning; WAN, MPLS y SD-WAN frente a LAN.
- [Storage Area Network (SAN) vs. Network Attached Storage (NAS)](https://www.ibm.com/think/topics/san-vs-nas) — IBM; comparación detallada de casos de uso.
- [Storage Networking Primer](https://www.snia.org/education/storage_networking_primer) — SNIA, la asociación de la industria de almacenamiento; NAS, SAN, Fibre Channel e iSCSI.
- [tcpdump(1)](https://man.archlinux.org/man/tcpdump.1) — página de manual; la opción `-e` muestra la cabecera de capa 2.

### Práctica

- [TryHackMe: Networking Concepts](https://tryhackme.com/room/networkingconcepts) — OSI, TCP/IP y encapsulamiento con preguntas guiadas; practica Understand the OSI Model.
- [TryHackMe: Intro to LAN](https://tryhackme.com/room/introtolan) — topologías, switches y routers; practica Network Topologies y LAN.
- [TryHackMe: What is Networking?](https://tryhackme.com/room/whatisnetworking) — introducción para quien empieza de cero; practica conceptos previos y LAN.
- [HTB Academy: Introduction to Networking](https://academy.hackthebox.com/course/preview/introduction-to-networking) — tipos de red, topologías y modelos de capas desde la óptica de seguridad; practica toda la nota.
- [Cisco Packet Tracer](https://www.netacad.com/cisco-packet-tracer) — simulador gratuito; arma una estrella con un switch y una malla de 4 routers y observa en modo Simulation cómo se encapsula cada PDU capa por capa.
- Ejercicio en casa: en tu propia máquina, `sudo tcpdump -i <interfaz> -e -n -c 5 'tcp port 443'` mientras abres una web; identifica en cada línea qué parte es capa 2, 3 y 4, y comprueba que `length` = 14 + cabecera IP + cabecera TCP + datos.
- Ejercicio en casa: con dos máquinas virtuales propias, exporta una carpeta por NFS desde una y móntala en la otra; luego compárala con un disco iSCSI (paquete `targetcli` en el servidor) y anota qué lado formatea el sistema de archivos en cada caso.

## Cuadro resumen

Todo lo visto, en una línea por término.

Understand the OSI Model

```
OSI → modelo de referencia de 7 capas (ISO/IEC 7498-1), vocabulario para diagnosticar.
Capa 7 Application → protocolo de la aplicación (HTTP, DNS); PDU datos.
Capa 6 Presentation → formato, cifrado, compresión (TLS, UTF-8); PDU datos.
Capa 5 Session → abrir, mantener y cerrar el diálogo; PDU datos.
Capa 4 Transport → puertos; TCP fiable / UDP rápido; PDU segmento / datagrama.
Capa 3 Network → IP y enrutamiento entre redes; TTL; PDU paquete.
Capa 2 Data Link → MAC, entrega en la red local, FCS; PDU trama.
Capa 1 Physical → bits como señales en el medio; PDU bits.
Mnemotecnia 1→7 → Please Do Not Throw Sausage Pizza Away.
TCP/IP → modelo implementado en Internet, 4 capas (RFC 1122).
Application TCP/IP → capas 5 + 6 + 7 de OSI.
Link TCP/IP → capas 1 + 2 de OSI.
Encapsulamiento → cada capa añade su cabecera al bajar.
Desencapsulamiento → cada capa quita su cabecera al subir.
Cabeceras → Ethernet 14 + FCS 4; IPv4 20; TCP 20; UDP 8 bytes.
MSS → 1500 − 20 IP − 20 TCP = 1460 bytes de datos.
MAC → cambia en cada salto; IP → se mantiene de extremo a extremo.
Regla de capa → un equipo o ataque está en la capa de la cabecera más alta que lee.
Hub → capa 1; repite bits a todos los puertos.
Switch → capa 2; reenvía por MAC.
Router → capa 3; enruta por IP.
Firewall stateful → capa 4; filtra por IP, puerto y estado de conexión.
WAF → capa 7; filtra contenido HTTP.
Ataques capa 2 → ARP spoofing, MAC flooding, VLAN hopping; solo en la LAN.
Ataques capa 3-4 → IP spoofing, ICMP flood, SYN flood, port scanning.
Ataques capa 7 → SQLi, XSS, phishing, HTTP flood.
```

Network Topologies

```
Topología física → por dónde van los cables.
Topología lógica → cómo circulan realmente los datos.
Star → todos al centro; n cables; falla un cable cae un nodo; falla el centro caen todos.
Ring → cada nodo con dos vecinos; n enlaces; token; un corte tumba un anillo simple.
Mesh → muchos caminos; full mesh n(n−1)/2 enlaces; máxima redundancia y coste.
Bus → un cable compartido con terminadores de 50 ohm; colisiones; histórica.
Hub → estrella física, bus lógico.
```

Understand these

```
LAN → edificio, cable Ethernet, un dueño, 1-10 Gbps, < 1 ms.
WLAN → LAN por radio (802.11), AP, SSID, 2,4/5/6 GHz, WPA2/WPA3.
MAN → ciudad, une varias LAN, 5-50 km, fibra/Metro Ethernet.
WAN → países y continentes, enlaces alquilados, MPLS/VPN/SD-WAN; Internet es la mayor.
PAN → pocos metros alrededor de una persona (Bluetooth).
```

Basics of NAS and SAN

```
NAS → caja en la LAN que comparte archivos; SMB 445 / NFS 2049; nivel archivo.
SAN → red dedicada que entrega bloques; Fibre Channel / iSCSI 3260; nivel bloque.
DAS → disco conectado directamente al servidor, sin red.
LUN → disco lógico que la SAN presenta a un servidor.
Zoning / LUN masking → controlan qué servidor ve cada LUN.
Criterio NAS vs SAN → quién maneja el sistema de archivos: el NAS o el servidor.
```
