# Ejercicios: Direccionamiento IP

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Debajo de cada bloque de comandos hay una lista que explica el comando y todas sus
opciones y argumentos. Cuando un comando y una opción ya se explicaron antes en este
mismo archivo, se indica con "(ver ejercicio N)".

El Ejercicio 1 se hace en Cisco Packet Tracer y está dividido en partes. Los Ejercicios
2 a 5 se hacen en tu propio equipo.

## Ejercicio 1: Dos VLAN con DHCP y una DMZ en Packet Tracer

Nodo: [VLAN](README.md#vlan), [DHCP: el proceso DORA](README.md#dhcp-el-proceso-dora),
[DMZ](README.md#dmz) y [Default gateway](README.md#default-gateway).

Objetivo: montar un switch con dos VLAN, un router que hace de servidor DHCP por VLAN
(router-on-a-stick) y una DMZ con un servidor web, y comprobar con ping y con el navegador
qué tráfico llega y cuál bloquea el router. Al terminar verás en la práctica por qué una
VLAN de invitados no alcanza a la de ventas y por qué la DMZ no puede entrar a la LAN.

Necesitas: Cisco Packet Tracer; una hora. Modelos: router `2911` (R1), switch `2960`
(SW1), dos PC y un servidor. Todo es simulado.

### Parte A: topología física

1. Coloca en el lienzo: un router `2911` (R1), un switch `2960` (SW1), dos PC
   (PC-Ventas, PC-Invitados) y un servidor (Srv-Web).

2. Cablea con cobre directo (`Copper Straight-Through`):
   - PC-Ventas `FastEthernet0` → SW1 `FastEthernet0/1`
   - PC-Invitados `FastEthernet0` → SW1 `FastEthernet0/2`
   - SW1 `GigabitEthernet0/1` → R1 `GigabitEthernet0/0` (será el trunk)
   - Srv-Web `FastEthernet0` → R1 `GigabitEthernet0/1` (la DMZ)

### Parte B: plan de direcciones

```
VLAN 10 Ventas      10.0.10.0/24   gateway 10.0.10.1 (R1 Gi0/0.10)   por DHCP
VLAN 20 Invitados   10.0.20.0/24   gateway 10.0.20.1 (R1 Gi0/0.20)   por DHCP
DMZ (sin VLAN)      10.0.99.0/24   gateway 10.0.99.1 (R1 Gi0/1)      Srv-Web estático
```

El servidor web lleva IP fija `10.0.99.10`, máscara `255.255.255.0`, gateway `10.0.99.1`
(se pone en `Desktop` → `IP Configuration` → Static). Los dos PC se dejan en `DHCP`.

### Parte C: configuración del switch

Abre SW1, pestaña `CLI`, y pega:
```
enable
configure terminal
hostname SW1
vlan 10
 name Ventas
exit
vlan 20
 name Invitados
exit
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
interface GigabitEthernet0/1
 switchport mode trunk
end
write memory
```
- `enable` → entra al modo privilegiado (EXEC).
- `configure terminal` → entra al modo de configuración global.
- `hostname SW1` → fija el nombre del switch.
- `vlan 10` → crea la VLAN con ese ID y entra a su configuración.
- `name Ventas` → le pone un nombre descriptivo a la VLAN.
- `exit` → sube un nivel en los modos de configuración.
- `interface FastEthernet0/1` → entra a configurar ese puerto.
- `switchport mode access` → fija el puerto en modo acceso (un solo VLAN, para un equipo final).
- `switchport access vlan 10` → asigna el puerto a la VLAN 10.
- `interface GigabitEthernet0/1` → entra a configurar el puerto de subida al router.
- `switchport mode trunk` → fija el puerto en modo trunk: lleva varias VLAN etiquetadas con 802.1Q.
- `end` → vuelve al modo privilegiado.
- `write memory` → guarda la configuración en marcha a la de arranque.

Los puertos de los PC quedan cada uno en su VLAN; el puerto al router es un trunk que
lleva ambas VLAN etiquetadas.

### Parte D: configuración del router (subinterfaces, DHCP y ACL)

Abre R1, pestaña `CLI`. Primero las subinterfaces (una por VLAN) y la interfaz de la DMZ:
```
enable
configure terminal
hostname R1
interface GigabitEthernet0/0
 no ip address
 no shutdown
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.0.10.1 255.255.255.0
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.0.20.1 255.255.255.0
interface GigabitEthernet0/1
 ip address 10.0.99.1 255.255.255.0
 no shutdown
exit
```
- `enable`, `configure terminal`, `hostname R1` → ver Parte C.
- `interface GigabitEthernet0/0` → entra a la interfaz física que irá al trunk.
- `no ip address` → quita cualquier IP de la interfaz física: las IP van en las subinterfaces.
- `no shutdown` → activa la interfaz.
- `interface GigabitEthernet0/0.10` → crea/entra a la subinterfaz `.10` (una por VLAN) sobre la física.
- `encapsulation dot1Q 10` → marca que esta subinterfaz procesa las tramas etiquetadas con la VLAN 10 (802.1Q).
- `ip address 10.0.10.1 255.255.255.0` → IP (primer argumento) y máscara (segundo) de la subinterfaz; es el gateway de esa VLAN.
- `interface GigabitEthernet0/1` → entra a la interfaz física de la DMZ (sin VLAN, directa al servidor).
- `exit` → sale del modo de interfaz.

Ahora los dos ámbitos DHCP, uno por VLAN. Se excluyen las primeras direcciones para el
gateway y para equipos fijos:
```
ip dhcp excluded-address 10.0.10.1 10.0.10.9
ip dhcp excluded-address 10.0.20.1 10.0.20.9
ip dhcp pool VENTAS
 network 10.0.10.0 255.255.255.0
 default-router 10.0.10.1
 dns-server 10.0.99.10
exit
ip dhcp pool INVITADOS
 network 10.0.20.0 255.255.255.0
 default-router 10.0.20.1
 dns-server 10.0.99.10
exit
```
- `ip dhcp excluded-address 10.0.10.1 10.0.10.9` → reserva el rango de la `.1` a la `.9` para que el DHCP no lo reparta (primer y último argumento son el inicio y el fin del rango).
- `ip dhcp pool VENTAS` → crea un ámbito DHCP con ese nombre y entra a su configuración.
- `network 10.0.10.0 255.255.255.0` → la red y máscara desde las que el ámbito reparte direcciones.
- `default-router 10.0.10.1` → la puerta de enlace que el DHCP entrega a los clientes (opción 3).
- `dns-server 10.0.99.10` → el servidor DNS que el DHCP entrega (opción 6).
- `exit` → sale del ámbito.
- El segundo bloque (`INVITADOS`) es igual con las direcciones de la VLAN 20.

Por último las ACL que imponen la política de la DMZ y la separación entre VLAN. La
ACL de la DMZ permite que la DMZ responda a lo que la LAN inicia (echo-reply y conexiones
establecidas) pero le prohíbe iniciar conexiones hacia la LAN; la de invitados les
prohíbe alcanzar ventas:
```
ip access-list extended DMZ-IN
 permit icmp 10.0.99.0 0.0.0.255 any echo-reply
 permit tcp 10.0.99.0 0.0.0.255 any established
 deny ip 10.0.99.0 0.0.0.255 10.0.10.0 0.0.0.255
 deny ip 10.0.99.0 0.0.0.255 10.0.20.0 0.0.0.255
 permit ip any any
exit
ip access-list extended INVITADOS-IN
 deny ip 10.0.20.0 0.0.0.255 10.0.10.0 0.0.0.255
 permit ip any any
exit
interface GigabitEthernet0/1
 ip access-group DMZ-IN in
interface GigabitEthernet0/0.20
 ip access-group INVITADOS-IN in
end
write memory
```
- `ip access-list extended DMZ-IN` → crea una ACL extendida (filtra por protocolo, origen, destino y puertos) con ese nombre y entra a editarla.
- `permit icmp 10.0.99.0 0.0.0.255 any echo-reply` → permite respuestas de ping (`echo-reply`) que salen de la DMZ (`10.0.99.0` con wildcard `0.0.0.255`) hacia cualquier destino (`any`).
- `permit tcp 10.0.99.0 0.0.0.255 any established` → permite tráfico TCP de respuesta desde la DMZ: `established` solo casa segmentos con el bit ACK/RST puesto, o sea respuestas a conexiones que inició la LAN.
- `deny ip 10.0.99.0 0.0.0.255 10.0.10.0 0.0.0.255` → bloquea cualquier IP que la DMZ intente iniciar hacia la VLAN 10.
- `deny ip 10.0.99.0 0.0.0.255 10.0.20.0 0.0.0.255` → lo mismo hacia la VLAN 20.
- `permit ip any any` → permite todo lo demás (necesario para que la DMZ salga a Internet o responda fuera de la LAN).
- `ip access-list extended INVITADOS-IN` → segunda ACL extendida.
- `deny ip 10.0.20.0 0.0.0.255 10.0.10.0 0.0.0.255` → bloquea que invitados alcance ventas.
- `permit ip any any` → permite el resto (DHCP, salida a la DMZ, etc.).
- `interface GigabitEthernet0/1` / `interface GigabitEthernet0/0.20` → ver Parte D (bloque anterior).
- `ip access-group DMZ-IN in` → aplica la ACL `DMZ-IN` al tráfico que entra (`in`) por esa interfaz.
- `ip access-group INVITADOS-IN in` → aplica `INVITADOS-IN` al tráfico que entra desde la VLAN 20.
- `end`, `write memory` → ver Parte C.

### Parte E: comprobar DHCP y la matriz de ping

1. En cada PC, abre `Desktop` → `IP Configuration` y marca `DHCP`. PC-Ventas debe recibir
   una IP del rango `10.0.10.10`+ con gateway `10.0.10.1`; PC-Invitados, del rango
   `10.0.20.10`+ con gateway `10.0.20.1`. Si no la reciben, revisa el trunk y las VLAN.

2. Confirma en el router los préstamos DHCP entregados:
   ```
   show ip dhcp binding
   ```
   - `show ip dhcp binding` → lista las direcciones que el DHCP del router ha asignado, con su MAC y su tiempo de concesión.

   Deben aparecer las dos IP asignadas con su MAC.

3. Ejecuta la matriz de ping desde `Desktop` → `Command Prompt` de cada equipo y anota
   el resultado esperado:

   ```
   Origen          Destino              Resultado   Por qué
   PC-Ventas    →  10.0.99.10 (DMZ)     llega       la LAN puede iniciar hacia la DMZ
   PC-Invitados →  10.0.99.10 (DMZ)     llega       igual para invitados
   PC-Invitados →  PC-Ventas (10.0.10)  NO llega    INVITADOS-IN bloquea 20→10
   Srv-Web      →  PC-Ventas (10.0.10)  NO llega    DMZ-IN prohíbe que la DMZ inicie
   PC-Ventas    →  10.0.20.x (Invitados) NO llega   el deny de vuelta corta el eco
   ```

   El comando en cada caso es `ping <destino>` (envía ecos ICMP al destino indicado).

4. Prueba el servidor web desde la LAN: en PC-Ventas, `Desktop` → `Web Browser`, escribe
   `http://10.0.99.10`. Debe cargar la página por defecto del servidor. Eso demuestra que
   la LAN sí accede al servicio público de la DMZ.

5. Mira los aciertos de las ACL para confirmar que están filtrando:
   ```
   show access-lists
   ```
   - `show access-lists` → muestra todas las ACL configuradas y, por cada línea, cuántos paquetes han coincidido (`N match(es)`).

   Los contadores suben en las líneas `deny` cada vez que un ping bloqueado choca contra
   ellas.

### Resultado esperado

Un archivo `.pkt` donde los dos PC obtienen IP por DHCP en su VLAN, la LAN accede al
servidor web de la DMZ, pero invitados no alcanza a ventas y la DMZ no puede iniciar
tráfico hacia la LAN, todo verificado con la matriz de ping y con `show access-lists`.

### Comprueba que lo lograste

- ¿Por qué un solo cable físico entre el switch y el router lleva las dos VLAN? Porque es
  un trunk 802.1Q: cada trama va etiquetada con su VLAN ID y el router la enruta por la
  subinterfaz correspondiente (router-on-a-stick).
- ¿Qué pasa si quitas la línea `deny ip 10.0.20.0 ... 10.0.10.0` de INVITADOS-IN? Que un
  invitado podría hacer ping a ventas: desaparece la segmentación entre VLAN.
- ¿Por qué la DMZ puede responder a la web pero no puede iniciar un ping a la LAN? Porque
  la ACL DMZ-IN permite respuestas (`echo-reply`, `established`) pero deniega que la DMZ
  arranque conexiones nuevas hacia las redes internas.

### Limpieza

No aplica a tu equipo: todo vive en el archivo de Packet Tracer. Guárdalo como
`vlan-dhcp-dmz.pkt`.

## Ejercicio 2: Captura el proceso DORA de DHCP

Nodo: [DHCP: el proceso DORA](README.md#dhcp-el-proceso-dora).

Objetivo: capturar en tu propia máquina los cuatro mensajes DHCP (Discover, Offer,
Request, Acknowledge) al reconectar la red e identificar en la salida las opciones 1
(máscara), 3 (gateway), 6 (DNS) y 51 (duración del lease). Al terminar reconocerás el
intercambio DORA en una captura real.

Necesitas: tu propio equipo o VM Linux que obtenga IP por DHCP; `tcpdump` (instalar con
`sudo apt install tcpdump` en Debian/Ubuntu o `sudo pacman -S tcpdump` en Arch); `sudo`.
Tiempo estimado: 20 minutos. Capturas tu propio tráfico DHCP.

### Pasos

1. Identifica tu interfaz de red activa.
   ```bash
   ip -br addr
   ```
   - `ip` → herramienta estándar de red en Linux.
   - `-br` → brief: salida resumida, una línea por interfaz.
   - `addr` → subcomando que muestra las direcciones de las interfaces.

   Anota el nombre (por ejemplo `eth0` o `wlan0`).

2. Arranca la captura filtrando los puertos de DHCP (67 servidor, 68 cliente). Déjala
   corriendo.
   ```bash
   sudo tcpdump -i eth0 -n -v 'udp port 67 or udp port 68'
   ```
   - `sudo` → capturar tráfico requiere privilegios de root.
   - `tcpdump` → captura y muestra tráfico de red.
   - `-i eth0` → interface: la interfaz por la que capturar (cambia `eth0` por la tuya).
   - `-n` → no resuelve IP a nombres; muestra números.
   - `-v` → verbose: muestra más detalle, incluido el tipo de mensaje DHCP y las opciones.
   - `'udp port 67 or udp port 68'` → filtro de captura: solo tráfico UDP de los puertos 67 o 68 (los de DHCP).

3. En otra terminal, fuerza una renovación del lease para provocar el intercambio. En una
   VM con `dhclient`:
   ```bash
   sudo dhclient -r eth0 && sudo dhclient eth0
   ```
   - `sudo` → ver ejercicio 2 (paso 2).
   - `dhclient` → cliente DHCP de Linux.
   - `-r` → release: libera la concesión actual.
   - `eth0` → la interfaz sobre la que actúa.
   - `&&` → encadena: ejecuta el segundo comando solo si el primero salió bien.
   - `sudo dhclient eth0` → vuelve a pedir una concesión (sin `-r`), lo que dispara el DORA.

   Si tu sistema usa NetworkManager, puedes en su lugar desconectar y reconectar la
   interfaz con `nmcli device disconnect eth0 && nmcli device connect eth0` (`nmcli`
   controla NetworkManager; `device disconnect`/`device connect` bajan y suben la
   interfaz indicada).

4. Vuelve a la captura. Busca los cuatro mensajes en orden. Con `-v`, `tcpdump` etiqueta
   cada uno (`Discover`, `Offer`, `Request`, `ACK`):
   ```
   IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request ... Option 53 ... Discover
   IP 192.168.1.1.67 > 192.168.1.20.68: BOOTP/DHCP, Reply ... Option 53 ... Offer
   IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request ... Option 53 ... Request
   IP 192.168.1.1.67 > 192.168.1.20.68: BOOTP/DHCP, Reply ... Option 53 ... ACK
   ```
   Fíjate en que el Discover y el Request salen de `0.0.0.0` hacia el broadcast
   `255.255.255.255`, porque el cliente aún no tiene IP.

5. En el Offer y el ACK, localiza las opciones que entregan la configuración:
   - `Subnet-Mask Option 1`: la máscara.
   - `Default-Gateway Option 3` (o `Router`): la puerta de enlace.
   - `Domain-Name-Server Option 6`: los servidores DNS.
   - `Lease-Time Option 51`: la duración del lease en segundos.

### Resultado esperado

Una captura con las cuatro líneas D-O-R-A identificadas y las opciones 1, 3, 6 y 51
señaladas en el Offer o el ACK, más la observación de que el cliente arranca sin IP
(origen `0.0.0.0`, destino broadcast).

### Comprueba que lo lograste

- ¿Por qué el Discover va a `255.255.255.255` y no a la IP del servidor? Porque el
  cliente todavía no tiene IP ni sabe dónde está el servidor: pregunta por broadcast.
- ¿Qué opción dice cuánto dura la concesión y en qué unidad? La opción 51, en segundos.
- ¿Qué par de puertos UDP usa siempre este intercambio? 67 en el servidor y 68 en el
  cliente.

### Limpieza

No aplica: solo capturaste y renovaste tu propio lease, algo que el sistema hace
normalmente. Puedes conservar `tcpdump`.

## Ejercicio 3: Sigue la jerarquía del DNS con dig

Nodo: [DNS: resolución y tipos de registro](README.md#dns-resolución-y-tipos-de-registro).

Objetivo: usar `dig +trace` para ver cómo una consulta recorre la raíz, el TLD y el
servidor autoritativo, y luego consultar los registros MX, TXT y PTR de un dominio. Al
terminar sabrás leer la jerarquía del DNS y distinguir los tipos de registro.

Necesitas: tu propio equipo o VM con salida a Internet; `dig` (instalar con `sudo apt
install dnsutils` en Debian/Ubuntu o `sudo pacman -S bind` en Arch). Tiempo estimado: 20
minutos. Usa un dominio tuyo o uno muy conocido como `wikipedia.org`.

### Pasos

1. Haz una consulta normal del registro A para tener el resultado final.
   ```bash
   dig wikipedia.org A +noall +answer
   ```
   - `dig` → herramienta de consulta de DNS.
   - `wikipedia.org` → el nombre de dominio a consultar.
   - `A` → el tipo de registro pedido (A = nombre a IPv4).
   - `+noall` → apaga todas las secciones de la salida (deja una salida limpia).
   - `+answer` → vuelve a encender solo la sección de respuesta.

   La línea de respuesta muestra el nombre, el TTL, la clase `IN`, el tipo `A` y la IP.

2. Ahora sigue la jerarquía completa con `+trace`. En vez de pedirle la respuesta a tu
   resolver, `dig` pregunta desde la raíz hacia abajo.
   ```bash
   dig wikipedia.org +trace
   ```
   - `dig wikipedia.org` → ver ejercicio 3 (paso 1).
   - `+trace` → hace la resolución iterativa paso a paso: pregunta a la raíz, al TLD y al autoritativo, mostrando cada delegación.

   Lee la salida de arriba hacia abajo: primero los servidores raíz que te mandan al
   TLD, luego los de `.org` que te mandan al autoritativo, y por último el autoritativo
   que da la IP. Son los pasos 2, 3 y 4 de la resolución.

3. Consulta el servidor de correo (registro MX), que incluye una prioridad.
   ```bash
   dig wikipedia.org MX +noall +answer
   ```
   - `dig wikipedia.org ... +noall +answer` → ver ejercicio 3 (paso 1).
   - `MX` → el tipo de registro pedido (servidor de correo, con un número de prioridad).

4. Consulta los registros de texto (TXT), donde suelen vivir SPF, DKIM y DMARC.
   ```bash
   dig wikipedia.org TXT +noall +answer
   ```
   - `TXT` → el tipo de registro pedido (texto libre); el resto, ver ejercicio 3 (paso 1).

   Busca una línea que empiece con `v=spf1`: es la política SPF del dominio.

5. Haz una consulta inversa (PTR): de una IP a su nombre. Toma una de las IP que te dio
   el paso 1.
   ```bash
   dig -x 198.35.26.96 +noall +answer
   ```
   - `dig` → ver ejercicio 3 (paso 1).
   - `-x 198.35.26.96` → consulta inversa: busca el registro PTR de esa IP (su nombre).
   - `+noall +answer` → ver ejercicio 3 (paso 1).

   Devuelve el nombre asociado a esa IP (o nada, si el dueño no publicó PTR).

### Resultado esperado

La traza de una consulta pasando por raíz, TLD y autoritativo, más las respuestas MX, TXT
y PTR del dominio, y la capacidad de decir qué información da cada tipo de registro.

### Comprueba que lo lograste

- En `+trace`, ¿quién te dice la IP final, la raíz o el autoritativo? El autoritativo del
  dominio; la raíz y el TLD solo te derivan al siguiente servidor.
- ¿Qué número extra trae un registro MX y para qué sirve? La prioridad: a menor número,
  servidor de correo preferido.
- ¿Qué hace `dig -x` distinto de una consulta normal? Resuelve al revés, de IP a nombre
  (registro PTR), usando la zona `in-addr.arpa`.

### Limpieza

No aplica: `dig` solo hace consultas de lectura.

## Ejercicio 4: Lee tu gateway, tu caché ARP y tus servicios locales

Nodo: [Default gateway](README.md#default-gateway), [ARP](README.md#arp) y
[localhost](README.md#localhost).

Objetivo: en tu propia máquina, identificar tu puerta de enlace con `ip route`, la MAC
del gateway en la caché ARP con `ip neigh`, y qué servicios escuchan solo en `127.0.0.1`
con `ss -tln`. Al terminar sabrás leer el estado de red de un equipo de un vistazo.

Necesitas: tu propio equipo o VM Linux. `ip` y `ss` vienen en el paquete `iproute2`, ya
instalado. Tiempo estimado: 15 minutos.

### Pasos

1. Encuentra tu puerta de enlace: es la ruta `default`.
   ```bash
   ip route
   ```
   - `ip` → herramienta estándar de red (ver ejercicio 2, paso 1).
   - `route` → subcomando que muestra la tabla de rutas.

   La línea `default via X.X.X.X dev ...` te da la IP del gateway. La otra línea
   (`.../24 dev ... scope link`) es tu red conectada directamente. Anota la IP del
   gateway.

2. Mira la caché ARP (vecinos de capa 2 que tu equipo conoce) y localiza la MAC de ese
   gateway.
   ```bash
   ip neigh
   ```
   - `ip` → ver ejercicio 2 (paso 1).
   - `neigh` → subcomando (neighbour) que muestra la tabla de vecinos: la caché ARP en IPv4 y NDP en IPv6.

   Busca la línea con la IP del gateway: muestra `lladdr` seguido de su dirección MAC y
   un estado (`REACHABLE`, `STALE`).

3. Si el gateway no aparece o está `STALE`, provoca tráfico hacia él para refrescar la
   entrada y vuelve a mirar.
   ```bash
   ping -c 2 $(ip route | awk '/default/{print $3; exit}')
   ip neigh
   ```
   - `ping` → envía ecos ICMP al destino.
   - `-c 2` → count: envía 2 paquetes y termina.
   - `$(...)` → sustitución de comando: ejecuta lo de dentro y usa su salida como argumento del `ping`.
   - `ip route` → ver ejercicio 4 (paso 1).
   - `|` → tubería: pasa la salida a `awk`.
   - `awk '/default/{print $3; exit}'` → programa de awk: en la línea que contiene `default`, imprime el tercer campo (la IP del gateway) y para.
   - `ip neigh` → ver ejercicio 4 (paso 2).

   Ahora debe salir `REACHABLE`.

4. Lista los puertos TCP en escucha y fíjate en la dirección local de cada uno.
   ```bash
   ss -tln
   ```
   - `ss` → muestra sockets (conexiones y puertos).
   - `-t` → sockets TCP.
   - `-l` → solo los que están en escucha (listening).
   - `-n` → numeric: muestra puertos como número, sin traducir a nombre de servicio.

   Distingue dos casos en la columna `Local Address:Port`:
   - `127.0.0.1:PUERTO`: el servicio solo acepta conexiones del propio equipo; no es
     alcanzable desde la red.
   - `0.0.0.0:PUERTO` (o `*:PUERTO`): escucha en todas las interfaces, también desde la
     red.

5. Haz la lista de qué servicios están atados a `localhost`.
   ```bash
   ss -tln | grep '127.0.0.1'
   ```
   - `ss -tln` → ver ejercicio 4 (paso 4).
   - `|` → tubería (ver ejercicio 4, paso 3).
   - `grep` → filtra líneas por patrón.
   - `'127.0.0.1'` → patrón: deja solo los sockets atados a loopback.

   Esos son los que un atacante de la red no puede tocar directamente (aunque sí podría
   intentarlo con técnicas como SSRF desde dentro del propio equipo).

### Resultado esperado

Tres datos anotados de tu máquina: la IP de tu gateway, su MAC según la caché ARP, y la
lista de servicios que escuchan solo en `127.0.0.1` frente a los que escuchan en todas
las interfaces.

### Comprueba que lo lograste

- ¿Por qué la caché ARP solo tiene direcciones de tu propia red y no la de un servidor de
  Internet? Porque ARP funciona dentro del dominio de broadcast local; para salir usas
  la MAC del gateway, no la del destino final.
- ¿Qué diferencia de seguridad hay entre un servicio en `127.0.0.1:5432` y uno en
  `0.0.0.0:5432`? El primero solo acepta conexiones locales; el segundo acepta de
  cualquier red, lo que amplía su exposición.
- Si `ip neigh` muestra el gateway como `STALE`, ¿significa que está caído? No: solo que
  la entrada no se ha usado hace un rato; un ping la vuelve a poner `REACHABLE`.

### Limpieza

No aplica: todos los comandos son de lectura (salvo el ping, que no cambia nada).

## Ejercicio 5: Subnetea diez direcciones a mano y verifícalo con Python

Nodo: [Basics of Subnetting](README.md#basics-of-subnetting), en concreto el método del
número mágico.

Objetivo: inventar diez direcciones con prefijo, calcular a mano la red, el broadcast y
el número de hosts de cada una con el método del número mágico, y comprobar cada
resultado con el módulo `ipaddress` de Python. Al terminar habrás automatizado la
corrección de tus propios ejercicios de subnetting.

Necesitas: papel y lápiz (o un archivo de texto) y `python3`, que ya viene en cualquier
Linux. Tiempo estimado: 30 minutos.

### Pasos

1. Inventa diez direcciones con prefijos variados, mezclando el octeto interesante en el
   segundo, tercero y cuarto octeto para practicar los tres casos. Por ejemplo:
   ```
   192.168.100.37/29
   172.31.200.9/22
   10.10.10.10/30
   192.168.1.200/25
   10.20.30.40/13
   172.16.45.200/20
   192.168.5.130/27
   10.0.5.17/28
   172.20.100.100/18
   192.168.200.50/26
   ```

2. Para cada una, aplica el método del número mágico a mano: localiza el octeto
   interesante (el primero de la máscara que no es 255 ni 0), calcula el bloque como
   `256 − valor de ese octeto`, encuentra el múltiplo del bloque que no supera el octeto
   de la IP (esa es la red), el siguiente múltiplo menos 1 (el broadcast) y
   `2^(32−prefijo) − 2` hosts. Anota red, broadcast y hosts de cada una.

3. Comprueba cada resultado con Python. Este comando imprime red, máscara, broadcast y
   hosts útiles de una dirección con prefijo:
   ```bash
   python3 -c 'import ipaddress as i; n=i.ip_interface("192.168.100.37/29").network; print(n, n.netmask, n.broadcast_address, n.num_addresses-2)'
   ```
   - `python3` → el intérprete de Python 3.
   - `-c '...'` → ejecuta el programa Python que va entre comillas, en lugar de un archivo.
   - El programa: `import ipaddress as i` carga el módulo y lo apoda `i`; `i.ip_interface("192.168.100.37/29")` crea el objeto dirección+prefijo; `.network` saca su red; `print(...)` muestra la red, `.netmask` (máscara), `.broadcast_address` (broadcast) y `.num_addresses-2` (hosts útiles, restando red y broadcast).

   Salida esperada para ese ejemplo:
   ```
   192.168.100.32/29 255.255.255.248 192.168.100.39 6
   ```
   Cambia la dirección y repite para las diez.

4. Para no reescribir el comando diez veces, pásalas todas de una pasada con un bucle:
   ```bash
   for ip in 192.168.100.37/29 172.31.200.9/22 10.10.10.10/30 192.168.1.200/25 \
             10.20.30.40/13 172.16.45.200/20 192.168.5.130/27 10.0.5.17/28 \
             172.20.100.100/18 192.168.200.50/26; do
     python3 -c "import ipaddress as i,sys; n=i.ip_interface(sys.argv[1]).network; print(sys.argv[1], '->', n, n.broadcast_address, n.num_addresses-2)" "$ip"
   done
   ```
   - `for ip in ... ; do ... done` → bucle de shell: recorre la lista de direcciones asignando cada una a la variable `ip`.
   - `\` al final de línea → continúa el comando en la línea siguiente.
   - `python3 -c "..."` → ver ejercicio 5 (paso 3); aquí el programa lee la dirección con `sys.argv[1]` (el primer argumento que recibe) en lugar de tenerla fija.
   - `"$ip"` → pasa el valor actual del bucle como argumento al `python3`.

5. Compara cada línea con tus cálculos a mano. Donde no cuadren, vuelve al número mágico
   de esa dirección: casi siempre el error está en elegir mal el octeto interesante o el
   múltiplo del bloque.

### Resultado esperado

Diez direcciones resueltas a mano (red, broadcast y hosts) y la salida de Python para las
mismas diez, coincidiendo. Las que no coincidan al principio te muestran exactamente qué
paso del método se te escapó.

### Comprueba que lo lograste

- Para `10.20.30.40/13`, ¿cuál es el octeto interesante y el bloque? El segundo (máscara
  `255.248.0.0`), bloque `256 − 248 = 8`; la red es `10.16.0.0`.
- ¿Por qué se restan 2 al número de direcciones para dar los hosts útiles? Porque la
  primera (todos los bits de host a 0) es la dirección de red y la última (todos a 1) es
  el broadcast, y ninguna se asigna a un equipo.
- ¿Qué devuelve `num_addresses` sin el `-2`? El total de direcciones del bloque
  (`2^(32−prefijo)`), incluyendo red y broadcast.

### Limpieza

No aplica: solo hiciste cálculos y consultas con Python.
