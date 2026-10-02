# Direccionamiento IP

## Conceptos previos

- Bit: un dígito binario, 0 o 1.
- Octeto: grupo de 8 bits; vale de 0 a 255 en decimal.
- Binario: sistema de numeración en base 2; cada posición de un octeto vale, de izquierda a derecha, 128, 64, 32, 16, 8, 4, 2 y 1.
- Host: cualquier equipo con dirección en la red (PC, servidor, teléfono, impresora).
- Dirección MAC: identificador de 48 bits de la tarjeta de red; solo sirve dentro de la red local. Ver [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/).
- Capas OSI: modelo de 7 capas para ubicar protocolos y equipos (capa 2 = MAC y switches, capa 3 = IP y routers). Ver [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/).
- Puerto: número de 16 bits que identifica el programa destino dentro de un host (DNS 53, DHCP 67/68, NTP 123). Ver [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/).
- Broadcast: mensaje que va a todos los equipos de la misma red local.
- Dominio de broadcast: conjunto de equipos que reciben los broadcasts de los demás; lo corta un router.
- Operación AND: compara dos bits y da 1 solo si ambos son 1 (1 AND 1 = 1; 1 AND 0 = 0; 0 AND 0 = 0).
- Enrutar: decidir por qué interfaz y hacia qué siguiente equipo sale un paquete para acercarse a su destino.
- LAN: red local de un edificio u oficina. Ver [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/).

## IP Terminology

Se estudia esta parte antes que el subnetting porque el cálculo de subredes usa la máscara, la notación CIDR y el gateway en cada paso.

### Subnet mask

La **máscara de subred** (subnet mask) es un número de 32 bits, escrito como una dirección IPv4, que indica qué parte de una dirección IP identifica a la red y qué parte identifica al host dentro de esa red: los bits a 1 marcan la red y los bits a 0 marcan el host.

Existe porque una dirección IP sola no dice dónde termina la red. `192.168.10.77` puede ser el host 77 de la red `192.168.10.0` o el host 13 de la red `192.168.10.64`; solo la máscara lo decide. Los unos siempre van seguidos y a la izquierda, y los ceros a la derecha; una máscara como `255.255.0.255` no es válida.

Analogía: un número de teléfono con prefijo de área. "506-2222-3333": el prefijo dice la región y el resto, la línea. La máscara es la regla que dice cuántas cifras son prefijo.

Cómo se aplica: se hace AND bit a bit entre la IP y la máscara, y el resultado es la dirección de red.

```
IP       192.168.10.77    = 11000000.10101000.00001010.01001101
Máscara  255.255.255.192  = 11111111.11111111.11111111.11000000
AND      192.168.10.64    = 11000000.10101000.00001010.01000000   <- red
```

Valores posibles de un octeto de máscara, que conviene saberse de memoria (cada uno añade un bit a 1 por la izquierda):

```
Bits a 1   Binario     Decimal
1          10000000    128
2          11000000    192
3          11100000    224
4          11110000    240
5          11111000    248
6          11111100    252
7          11111110    254
8          11111111    255
```

La **wildcard mask** es la máscara invertida (los 0 pasan a 1 y viceversa): `255.255.255.192` tiene wildcard `0.0.0.63`. La usan las ACL de Cisco y OSPF, y su último número + 1 da el tamaño del bloque.

Ejemplo: con máscara `255.255.255.0`, los equipos `192.168.1.10` y `192.168.1.200` están en la misma red (`192.168.1.0`); con máscara `255.255.255.128` ya no, porque el primero cae en `192.168.1.0` y el segundo en `192.168.1.128`.

```
Subnet mask → 1s = red, 0s = host; IP AND máscara = dirección de red.
Wildcard mask → máscara invertida; la usan ACL y OSPF.
```

### CIDR

**CIDR** (Classless Inter-Domain Routing, RFC 4632) es un esquema de direccionamiento y enrutamiento que escribe la máscara como una barra seguida del número de bits de red (`/24`) y permite que una red tenga cualquier tamaño, no solo los tres tamaños fijos de las antiguas clases.

Existe porque el sistema de clases (1981) desperdiciaba direcciones: una clase A daba 16,7 millones de direcciones, una clase B 65 536 y una clase C 256, sin término medio. Una empresa que necesitaba 2000 direcciones recibía una clase B y tiraba 63 000. CIDR (RFC 1519 en 1993, actualizado por el RFC 4632) eliminó las clases y además permitió resumir rutas: un proveedor anuncia un solo `/16` en vez de 256 `/24`.

Analogía: las clases eran cajas de zapatos en tres tallas fijas; CIDR es una caja que se corta a la medida.

Cómo se lee: `/n` significa n bits a 1 en la máscara. Las direcciones de un bloque son 2 elevado a (32 − n).

```
/8  = 255.0.0.0        → 16 777 216 direcciones
/16 = 255.255.0.0      → 65 536
/20 = 255.255.240.0    → 4096
/24 = 255.255.255.0    → 256
/26 = 255.255.255.192  → 64
/30 = 255.255.255.252  → 4
/32 = 255.255.255.255  → 1 (un solo host)
```

Clases históricas, solo para reconocerlas: A del 1 al 126 en el primer octeto (/8 por defecto), B del 128 al 191 (/16), C del 192 al 223 (/24), D del 224 al 239 (multicast) y E del 240 al 255 (experimental).

Ejemplo de resumen de rutas (supernetting): las redes `10.0.0.0/24`, `10.0.1.0/24`, `10.0.2.0/24` y `10.0.3.0/24` comparten los primeros 22 bits, así que un router puede anunciarlas todas como `10.0.0.0/22`. Cuatro rutas pasan a ser una.

```
CIDR → /n bits de red; tamaño libre; 2^(32−n) direcciones; resume rutas.
Clases A/B/C → tamaños fijos /8, /16, /24; obsoletas desde 1993.
```

### Default gateway

El **default gateway** (puerta de enlace predeterminada) es la dirección IP del router de la red local al que un host envía todo el tráfico cuyo destino está fuera de su propia subred.

Existe porque un host solo puede entregar directamente (por MAC, en capa 2) a los equipos de su misma red. Para todo lo demás necesita un intermediario que sepa enrutar. El gateway debe estar dentro de la misma subred que el host; un gateway `10.0.0.1` para un host `192.168.1.20/24` no funciona.

Analogía: la recepción de un edificio. Si el paquete es para otra oficina del mismo piso lo llevas tú; si es para otra ciudad lo dejas en recepción y ellos se encargan.

Ejemplo en Linux: la ruta `default` es el gateway.

```
$ ip route
default via 192.168.1.1 dev wlan0 proto dhcp metric 600
192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.20
```

La primera línea dice "todo lo que no encaje en otra ruta, mándalo a 192.168.1.1"; la segunda dice "la red 192.168.1.0/24 está conectada directamente, entrega sin intermediario". Un síntoma clásico de gateway mal configurado es poder hacer ping a la impresora de la oficina pero no a ninguna web.

### Un host decide con un AND si habla directo o por el gateway

> [!IMPORTANT]
> Antes de enviar cada paquete, el host aplica su propia máscara a su IP y a la IP destino; si las dos redes resultantes coinciden entrega directo por ARP, y si no, envía el paquete al default gateway.

Esta comparación es el algoritmo de decisión que conecta máscara, subred y gateway en una sola regla. Es la razón de que una máscara equivocada rompa la red aunque las IP sean correctas.

```
Mi IP AND mi máscara  ==  IP destino AND mi máscara ?
        sí → mismo segmento → ARP por la IP destino → trama directa
        no → otro segmento  → ARP por la IP del gateway → trama al gateway
```

```
                         Destino 192.168.1.50     Destino 8.8.8.8
Mi IP / máscara          192.168.1.20/24          192.168.1.20/24
Mi red                   192.168.1.0              192.168.1.0
Red del destino          192.168.1.0              8.8.8.0
¿Coinciden?              sí                       no
MAC destino de la trama  la de 192.168.1.50       la del gateway 192.168.1.1
```

La última fila es la que cambia: la IP destino del paquete es siempre la final, pero la MAC destino es la del vecino directo o la del gateway.

Ejemplo de fallo: un PC con IP `192.168.1.20` y máscara `/25` (por error en vez de `/24`) cree que `192.168.1.200` está en otra red (la `.128`) y manda ese tráfico al gateway, que lo devuelve a la misma LAN; funciona lento o no funciona según el router. Con `/16`, en cambio, creería que `192.168.50.5` es vecino y nunca lo enviaría al gateway.

```
Default gateway → router al que va todo lo que no es de mi subred; debe estar en mi subred.
Regla del AND → misma red: directo por ARP; distinta: al gateway.
```

### Loopback

La **loopback** es una interfaz de red virtual que existe dentro de cada sistema operativo y que devuelve al propio equipo todo lo que se le envía, sin que el tráfico salga nunca por una tarjeta física. En IPv4 tiene reservado todo el bloque `127.0.0.0/8` (16 777 216 direcciones, aunque casi siempre se usa `127.0.0.1`); en IPv6 es la única dirección `::1/128`.

Existe para que los programas de un mismo equipo puedan comunicarse con la pila TCP/IP normal sin depender de que haya red, y para probar que la pila de red del propio sistema funciona. En Linux la interfaz se llama `lo`.

Analogía: escribirte una nota a ti mismo y dejarla en tu propio buzón; el cartero nunca la ve.

```
$ ip addr show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
```

Ejemplo: `ping 127.0.0.1` responde aunque desconectes el cable y apagues el Wi-Fi. Si no responde, el problema está en el sistema operativo, no en la red.

### localhost

**localhost** es el nombre de host reservado que siempre se resuelve a la dirección loopback del propio equipo (`127.0.0.1` en IPv4 y `::1` en IPv6).

Existe para no tener que recordar la dirección: un desarrollador escribe `http://localhost:3000` y llega a su propio servidor de pruebas. La traducción la hace el archivo local `/etc/hosts` (en Windows, `C:\Windows\System32\drivers\etc\hosts`) sin preguntar a ningún servidor DNS:

```
$ cat /etc/hosts
127.0.0.1   localhost
::1         localhost
```

Analogía: "yo" es a tu nombre lo que localhost es a 127.0.0.1: una palabra que siempre apunta a quien la dice.

Por qué importa en seguridad: un servicio que escucha solo en `127.0.0.1` no es alcanzable desde la red, mientras que uno en `0.0.0.0` escucha en todas las interfaces. `ss` lo muestra:

```
$ ss -tln
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
LISTEN  0       128         127.0.0.1:5432       0.0.0.0:*
LISTEN  0       128           0.0.0.0:22         0.0.0.0:*
```

Aquí PostgreSQL (5432) solo acepta conexiones locales y SSH (22) acepta de cualquier red. Muchos ataques SSRF (Server-Side Request Forgery, hacer que el servidor haga peticiones por ti) buscan justamente llegar a servicios de `localhost` que se creían a salvo; ver [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/).

> [!WARNING]
> localhost y loopback no son lo mismo: loopback es la interfaz y su bloque de direcciones; localhost es solo un nombre que apunta a ella, y como cualquier nombre puede manipularse editando el archivo hosts.

```
Loopback → interfaz virtual lo; 127.0.0.0/8 y ::1; el tráfico no sale del equipo.
localhost → nombre que resuelve a 127.0.0.1 / ::1 vía /etc/hosts.
0.0.0.0 al escuchar → todas las interfaces; 127.0.0.1 → solo el propio equipo.
```

## Basics of Subnetting

### Qué es el subnetting

El **subnetting** es la técnica de dividir una red IP en varias redes más pequeñas, llamadas subredes, tomando bits de la parte de host de la dirección y pasándolos a la parte de red.

Existe por tres razones. Primera, rendimiento: cada subred es un dominio de broadcast separado, así que un broadcast de contabilidad no molesta a los 500 equipos de producción. Segunda, seguridad: entre subredes el tráfico pasa por un router o firewall donde se puede filtrar (si el Wi-Fi de invitados es otra subred, no llega a los servidores). Tercera, ahorro: no se regala un `/24` de 254 hosts a un enlace que necesita 2.

Analogía: dividir un edificio de oficinas en pisos con control de acceso. El edificio es la red; cada piso, una subred; el ascensor con tarjeta, el router.

### Binario en dos minutos

Todo el subnetting es contar bits, así que hace falta convertir rápido entre decimal y binario dentro de un octeto. Se usan los valores de posición:

```
Posición   128  64  32  16   8   4   2   1
```

Decimal a binario: ve restando de izquierda a derecha. 77 = 64 + 8 + 4 + 1:

```
           128  64  32  16   8   4   2   1
77    →      0   1   0   0   1   1   0   1   = 01001101
200   →      1   1   0   0   1   0   0   0   = 11001000
45    →      0   0   1   0   1   1   0   1   = 00101101
```

Binario a decimal: suma los valores de las posiciones con 1. `11110000` = 128 + 64 + 32 + 16 = 240.

Potencias de 2 que hay que tener a mano: 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096.

### Todo subnetting es repartir 32 bits entre red y host

> [!IMPORTANT]
> Una dirección IPv4 tiene 32 bits fijos; cada bit que se pasa de host a red duplica el número de subredes y reduce a la mitad su tamaño, y de ahí salen todas las fórmulas.

Este reparto es el principio único del que se derivan las dos fórmulas del subnetting. Si una red tiene h bits de host y se toman s bits prestados:

```
Subredes creadas        = 2^s
Direcciones por subred  = 2^h
Hosts útiles por subred = 2^h − 2      (se restan la dirección de red y la de broadcast)
Tamaño de bloque        = 256 − (octeto interesante de la máscara)
```

En cada subred hay dos direcciones que no se asignan a equipos: la primera (todos los bits de host a 0) es la **dirección de red**, el nombre de la subred; la última (todos los bits de host a 1) es la **dirección de broadcast**, que llega a todos los de esa subred.

```
           /24          /25          /26          /27
Bits red   24           25           26           27
Bits host  8            7            6            5
Subredes   1            2            4            8       (partiendo de un /24)
Bloque     256          128          64           32
Hosts      254          126          62           30
```

La fila "Bits red + Bits host" suma 32 en todas las columnas: ahí está la idea entera. Lo que gana una parte lo pierde la otra.

Excepciones del −2: un `/31` (2 direcciones) se usa en enlaces punto a punto entre dos routers sin dirección de red ni broadcast (RFC 3021), y un `/32` identifica a un único host, por ejemplo en una ruta o una regla de firewall.

```
Hosts útiles → 2^h − 2; h = 32 − prefijo.
Subredes → 2^s; s = bits prestados.
Dirección de red → bits de host a 0; no asignable.
Broadcast → bits de host a 1; no asignable.
/31 → enlace punto a punto, 2 hosts (RFC 3021); /32 → un solo host.
```

### Los siete datos de un problema de subnetting

Un problema de subnetting es un ejercicio de cálculo que, dada una IP con su prefijo, pide algunos de estos siete datos; con el método de abajo salen todos a la vez:

1. Máscara en decimal.
2. Dirección de red.
3. Dirección de broadcast.
4. Primer host utilizable (red + 1).
5. Último host utilizable (broadcast − 1).
6. Número de hosts utilizables.
7. Número de subredes (respecto a la red original) o siguiente subred.

### Método paso a paso (número mágico)

El método del **número mágico** o tamaño de bloque es un procedimiento de cálculo mental que obtiene la red y el broadcast mirando un solo octeto, sin convertir toda la dirección a binario.

1. Escribe la máscara en decimal y localiza el **octeto interesante**: el primero que no es 255 ni 0 (si el prefijo es múltiplo de 8, es el octeto siguiente a los 255).
2. Tamaño de bloque = 256 − valor de ese octeto de la máscara.
3. Las subredes empiezan en múltiplos del bloque: 0, bloque, 2·bloque... La dirección de red es el múltiplo más alto que no supera el valor del octeto interesante de la IP. Los octetos a la izquierda se copian; los de la derecha pasan a 0.
4. Broadcast = siguiente múltiplo − 1 en el octeto interesante; los octetos de la derecha pasan a 255.
5. Primer host = red + 1; último host = broadcast − 1.
6. Hosts = 2^(32 − prefijo) − 2.

Analogía: buscar en qué cuadra de una calle está la casa número 77 sabiendo que cada cuadra tiene 64 casas: la cuadra empieza en la 64 y termina en la 127.

### Ejemplos resueltos

Ejemplo 1, `192.168.10.77/26` (octeto interesante: el cuarto).

```
1. /26 → 255.255.255.192; octeto interesante = 4º (192)
2. Bloque = 256 − 192 = 64 → subredes en .0, .64, .128, .192
3. 77 está entre 64 y 128 → red = 192.168.10.64
4. Broadcast = 128 − 1 → 192.168.10.127
5. Hosts: 192.168.10.65 a 192.168.10.126
6. 2^6 − 2 = 62 hosts
```

Ejemplo 2, `192.168.5.130/27`.

```
1. /27 → 255.255.255.224
2. Bloque = 256 − 224 = 32 → .0, .32, .64, .96, .128, .160 ...
3. 130 está entre 128 y 160 → red = 192.168.5.128
4. Broadcast = 192.168.5.159
5. Hosts: .129 a .158
6. 2^5 − 2 = 30 hosts
```

Ejemplo 3, `172.16.45.200/20` (el octeto interesante es el tercero; el cuarto da igual).

```
1. /20 → 255.255.240.0; octeto interesante = 3º (240)
2. Bloque = 256 − 240 = 16 → 0, 16, 32, 48 ...
3. 45 está entre 32 y 48 → red = 172.16.32.0   (4º octeto a 0)
4. Broadcast = 47 en el 3º y 255 en el 4º → 172.16.47.255
5. Hosts: 172.16.32.1 a 172.16.47.254
6. 2^12 − 2 = 4094 hosts
```

Ejemplo 4, `10.20.30.40/13` (octeto interesante: el segundo).

```
1. /13 → 255.248.0.0; octeto interesante = 2º (248)
2. Bloque = 256 − 248 = 8 → 0, 8, 16, 24 ...
3. 20 está entre 16 y 24 → red = 10.16.0.0
4. Broadcast = 10.23.255.255
5. Hosts: 10.16.0.1 a 10.23.255.254
6. 2^19 − 2 = 524 286 hosts
```

Ejemplo 5, dividir `10.0.0.0/24` en 4 subredes iguales.

```
Necesito 4 subredes → 2^s ≥ 4 → s = 2 bits prestados → /24 + 2 = /26
Bloque 64:
  10.0.0.0/26    hosts .1 – .62     broadcast .63
  10.0.0.64/26   hosts .65 – .126   broadcast .127
  10.0.0.128/26  hosts .129 – .190  broadcast .191
  10.0.0.192/26  hosts .193 – .254  broadcast .255
```

Ejemplo 6, elegir la máscara para un número de hosts. Busca el menor h con 2^h − 2 ≥ hosts pedidos; el prefijo es 32 − h.

```
50 hosts  → 2^6 − 2 = 62 ≥ 50   → h = 6 → /26
500 hosts → 2^9 − 2 = 510 ≥ 500 → h = 9 → /23 (255.255.254.0)
2 hosts (enlace entre routers) → 2^2 − 2 = 2 → /30
```

Ejemplo 7, VLSM. **VLSM** (Variable Length Subnet Mask, máscara de longitud variable) es la técnica de partir una red en subredes de tamaños distintos, cada una a la medida de lo que necesita. Regla: ordena de mayor a menor y asigna en ese orden, para que los bloques queden alineados. Con `192.168.1.0/24` para departamentos de 100, 50 y 20 hosts y un enlace de 2:

```
100 hosts → /25 (126) → 192.168.1.0/25     hosts .1 – .126    bc .127
 50 hosts → /26 (62)  → 192.168.1.128/26   hosts .129 – .190  bc .191
 20 hosts → /27 (30)  → 192.168.1.192/27   hosts .193 – .222  bc .223
  2 hosts → /30 (2)   → 192.168.1.224/30   hosts .225 – .226  bc .227
Libre: 192.168.1.228 – 192.168.1.255 para crecer
```

Ejemplo 8, ¿están en la misma subred `192.168.1.100` y `192.168.1.200` con `/25`? Bloque 128: la primera cae en `192.168.1.0/25` y la segunda en `192.168.1.128/25`. No, necesitan un router entre ellas.

Para comprobar resultados sin herramientas extra, Python trae el módulo `ipaddress`:

```
$ python3 -c 'import ipaddress as i; n=i.ip_interface("172.16.45.200/20").network; print(n, n.netmask, n.broadcast_address, n.num_addresses-2)'
172.16.32.0/20 255.255.240.0 172.16.47.255 4094
```

> [!TIP]
> En el examen, primero localiza el octeto interesante y calcula 256 menos la máscara: con ese bloque salen la red y el broadcast en segundos, y solo hace falta binario cuando dudas.

### Ejercicios

Resuélvelos a mano y luego compara con las soluciones.

1. `192.168.100.37/29`: red, broadcast, rango de hosts y número de hosts.
2. `172.31.200.9/22`: red, broadcast, rango de hosts y número de hosts.
3. `10.10.10.10/30`: red, broadcast y hosts.
4. `192.168.1.200/25`: red, broadcast y número de hosts.
5. ¿Cuántos hosts útiles tiene un `/21`?
6. ¿Qué prefijo mínimo necesitas para 1000 hosts?
7. Divide `192.168.50.0/24` en 8 subredes iguales: prefijo, las tres primeras subredes y el broadcast de la tercera.
8. ¿Están `10.1.1.130` y `10.1.1.190` en la misma subred `/26`?
9. VLSM: reparte `172.16.0.0/24` entre redes de 60, 28 y 12 hosts.

Soluciones:

```
1. /29 bloque 8 → red 192.168.100.32, bc .39, hosts .33 – .38, 6 hosts
2. /22 bloque 4 en el 3º → red 172.31.200.0, bc 172.31.203.255,
   hosts 172.31.200.1 – 172.31.203.254, 1022 hosts
3. /30 bloque 4 → red 10.10.10.8, bc .11, hosts .9 – .10 (2)
4. /25 bloque 128 → red 192.168.1.128, bc .255, 126 hosts
5. 2^11 − 2 = 2046
6. 2^10 − 2 = 1022 ≥ 1000 → h = 10 → /22
7. 8 = 2^3 → /27, bloque 32: .0/27, .32/27, .64/27; bc de la tercera = 192.168.50.95
8. Bloque 64: ambas caen en 10.1.1.128 – 10.1.1.191 → sí
9. 60 → /26 172.16.0.0 – .63; 28 → /27 172.16.0.64 – .95; 12 → /28 172.16.0.96 – .111
```

```
Subnetting → partir una red tomando bits de host para la red.
Octeto interesante → el primero de la máscara que no es 255 ni 0.
Bloque (número mágico) → 256 − octeto interesante de la máscara.
Red → múltiplo del bloque ≤ valor de la IP; broadcast → siguiente múltiplo − 1.
VLSM → subredes de tamaños distintos; asignar de mayor a menor.
```

## Public vs Private IP Addresses

### Direcciones privadas (RFC 1918)

Una **dirección IP privada** es una dirección IPv4 reservada por el RFC 1918 para uso interno de cualquier organización, que no es única en el mundo y que los routers de Internet no enrutan. Una **dirección IP pública** es una dirección única a escala mundial, asignada a través de IANA y los registros regionales (LACNIC en Latinoamérica), alcanzable desde cualquier punto de Internet.

Existe porque IPv4 solo tiene 2^32 ≈ 4300 millones de direcciones, menos que dispositivos conectados; IANA entregó sus últimos bloques libres en 2011. El RFC 1918 (1996) reservó tres rangos que todo el mundo puede reutilizar dentro de su red, y NAT traduce a una pública solo en la salida a Internet.

Los tres rangos exactos, que hay que saber de memoria:

```
Bloque CIDR       Primera dirección   Última dirección    Direcciones    Clase histórica
10.0.0.0/8        10.0.0.0            10.255.255.255      16 777 216     1 red clase A
172.16.0.0/12     172.16.0.0          172.31.255.255       1 048 576     16 redes clase B
192.168.0.0/16    192.168.0.0         192.168.255.255         65 536     256 redes clase C
```

Analogía: las extensiones telefónicas de una empresa. La extensión 105 existe en miles de empresas a la vez y no sirve para llamar desde fuera; para eso está el número público de la centralita.

Otros bloques especiales que se confunden con los privados (registro de IANA, RFC 6890):

```
127.0.0.0/8         Loopback
169.254.0.0/16      Link-local / APIPA (RFC 3927): el equipo se la pone solo si DHCP no responde
100.64.0.0/10       Shared address space / CGNAT (RFC 6598): entre el operador y tu router
192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24   Documentación (RFC 5737)
224.0.0.0/4         Multicast
0.0.0.0/8           "Esta red"; 0.0.0.0 como origen al arrancar DHCP
255.255.255.255/32  Broadcast limitado
```

Ejemplo: tu portátil tiene `192.168.1.20` (privada) y el router de casa tiene en su lado de Internet `181.x.x.x` (pública). Si ves `100.64.x.x` en la interfaz WAN del router, tu operador usa CGNAT y compartes la pública con otros clientes; si ves `169.254.x.x` en tu portátil, no encontró servidor DHCP.

> [!WARNING]
> El bloque 172.16.0.0/12 termina en 172.31.255.255: 172.15.0.1 y 172.32.0.1 son direcciones públicas, y es la trampa más repetida de los exámenes.

Uso en seguridad: un paquete que llega desde Internet con IP de origen privada es falso con seguridad (spoofing), así que los firewalls de borde descartan esos orígenes (filtrado de bogons, direcciones que nunca deberían aparecer en Internet). Al revés, las direcciones privadas en logs o cabeceras HTTP filtradas revelan la estructura interna de una red, información útil en reconocimiento.

### Privada no significa protegida

> [!IMPORTANT]
> Una IP privada no es una medida de seguridad: NAT oculta qué equipo hay detrás, pero cualquier conexión que salga desde dentro (un malware, un enlace en un correo) abre el camino de vuelta, y dentro de la LAN todos se ven entre sí.

Esta idea es el error de concepto más común sobre el direccionamiento privado. NAT impide que alguien de fuera inicie una conexión hacia un equipo interno solo como efecto secundario: el router no sabe a quién entregar un paquete entrante sin una entrada en su tabla. No inspecciona contenido ni distingue tráfico malicioso.

```
                         IP privada + NAT          Firewall con política
Oculta la IP interna     sí                        no necesariamente
Bloquea entrada no pedida sí, por efecto lateral   sí, por regla explícita
Filtra lo que sale       no                        sí
Separa equipos internos  no                        sí, entre zonas
```

Las tres últimas filas son las que importan: NAT solo cubre una, y además por accidente.

Ejemplo: un PC con `192.168.1.30` abre un adjunto con malware que conecta hacia afuera a un servidor del atacante; NAT deja pasar la respuesta porque la conexión la inició el PC. Desde ahí el atacante alcanza cualquier `192.168.1.x`. La protección real es firewall con reglas, segmentación y monitoreo; ver [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

```
IP privada → RFC 1918; reutilizable; no enrutable en Internet; necesita NAT para salir.
IP pública → única en el mundo; asignada vía IANA/RIR; alcanzable desde Internet.
10.0.0.0/8 → 10.0.0.0 – 10.255.255.255.
172.16.0.0/12 → 172.16.0.0 – 172.31.255.255.
192.168.0.0/16 → 192.168.0.0 – 192.168.255.255.
169.254.0.0/16 → APIPA, sin DHCP.
100.64.0.0/10 → CGNAT del operador.
NAT ≠ seguridad → oculta, pero no filtra lo que sale ni separa la LAN.
```

## Understand the Terminology

Se presentan en orden de dependencia: primero IP, switch y router, porque los demás términos se apoyan en ellos.

### IP

**IP** (Internet Protocol) es el protocolo de capa 3 que da a cada interfaz una dirección lógica y lleva paquetes de una red a otra, salto a salto, hasta el destino, sin garantizar que lleguen ni en qué orden (eso lo resuelve TCP arriba).

Existe para unir redes distintas en una sola Internet: una MAC solo sirve dentro de la LAN, mientras que una IP tiene estructura jerárquica (red + host) que permite a los routers decidir el camino sin conocer cada equipo del mundo.

Analogía: la dirección postal. La MAC es el nombre de la persona; la IP es calle, ciudad y país, lo que el correo necesita para llevar la carta de un país a otro.

Dos versiones en uso:

- IPv4 (RFC 791): 32 bits, cuatro octetos en decimal (`192.168.1.20`), unos 4300 millones de direcciones, agotadas.
- IPv6 (RFC 8200): 128 bits, ocho grupos hexadecimales (`2001:db8::1`), unos 3,4 × 10^38 direcciones; no necesita NAT. Los ceros seguidos se abrevian con `::` una sola vez por dirección.

Campos de la cabecera IPv4 que más importan en seguridad: IP origen y destino (se pueden falsificar: IP spoofing), TTL (cada router le resta 1; el valor inicial delata el sistema operativo, 64 en Linux y 128 en Windows) y Protocol (6 = TCP, 17 = UDP, 1 = ICMP).

Ejemplo: `ip -4 addr show wlan0` muestra `inet 192.168.1.20/24`, que es la IP y el prefijo de la interfaz en una sola línea.

```
IP → capa 3; dirección lógica y entrega entre redes; sin garantías.
IPv4 → 32 bits, decimal punteado, agotado.
IPv6 → 128 bits, hexadecimal, sin necesidad de NAT.
```

### Switch

Un **switch** es un dispositivo de capa 2 que conecta equipos dentro de una misma LAN y reenvía cada trama solo por el puerto donde está la MAC destino, gracias a una tabla de direcciones MAC (CAM table) que aprende sola.

Existe para reemplazar al hub, que repetía todo por todos los puertos. Aprende así: cuando entra una trama por el puerto 3 con MAC origen `aa:aa`, anota "aa:aa está en el puerto 3". Si la MAC destino no está en la tabla, la envía por todos los puertos (flooding) una vez, y aprende de la respuesta. Los broadcasts sí van a todos.

Analogía: un conserje que va memorizando en qué oficina se sienta cada persona y, desde entonces, lleva cada sobre solo a la puerta correcta.

```
$ show mac address-table        (switch Cisco)
Vlan    Mac Address       Type      Ports
   1    aaaa.bbbb.0001    DYNAMIC   Gi0/1
   1    aaaa.bbbb.0002    DYNAMIC   Gi0/2
  20    aaaa.bbbb.0003    DYNAMIC   Gi0/3
```

Ejemplo de ataque: MAC flooding llena la tabla (unas miles de entradas en switches baratos) con MAC falsas; cuando se llena, el switch inunda todo como un hub y el atacante ve el tráfico ajeno. La defensa es port security (máximo N MAC por puerto).

### Router

Un **router** es un dispositivo de capa 3 que conecta redes IP distintas y decide, para cada paquete, por qué interfaz enviarlo consultando su tabla de rutas, eligiendo la ruta más específica (el prefijo más largo) que coincida con el destino.

Existe porque un switch no sabe salir de su red. El router separa dominios de broadcast, decrementa el TTL en cada salto y, en los equipos domésticos, además hace NAT, DHCP y firewall. Las rutas pueden ser estáticas (escritas a mano) o aprendidas con protocolos dinámicos (OSPF dentro de una organización, BGP entre proveedores de Internet).

Analogía: una oficina de correos que mira solo la ciudad del destino y decide a qué otra oficina mandar la saca.

Ejemplo de prefijo más largo: con rutas `10.0.0.0/8 → A` y `10.1.0.0/16 → B`, un paquete hacia `10.1.5.5` sale por B, porque `/16` es más específico que `/8`.

```
$ traceroute -n 8.8.8.8
 1  192.168.1.1    1.2 ms      <- router de casa (gateway)
 2  100.64.0.1     8.5 ms      <- router del operador (CGNAT)
 3  200.x.x.x     12.0 ms
 4  8.8.8.8       25.3 ms
```

```
Switch → capa 2; reenvía tramas por MAC dentro de la LAN; tabla CAM.
Router → capa 3; une redes; enruta por IP; prefijo más largo gana.
```

### VLAN

Una **VLAN** (Virtual LAN, IEEE 802.1Q) es una LAN lógica creada dentro de un switch que separa los puertos en grupos que no se ven entre sí en capa 2, como si fueran switches físicos distintos, aunque compartan el mismo hardware.

Existe para segmentar sin comprar más switches: contabilidad, invitados y cámaras pueden colgar del mismo equipo y estar aislados. Cada VLAN es un dominio de broadcast propio y suele corresponder a una subred IP; para pasar de una VLAN a otra hace falta un router o un switch de capa 3, donde se aplican reglas. Entre switches, un enlace **trunk** transporta varias VLAN a la vez añadiendo a cada trama una etiqueta de 4 bytes con el VLAN ID (12 bits: valores 1 a 4094).

Analogía: un edificio de oficinas compartido por varias empresas: mismos pasillos y ascensores, pero cada empresa solo abre sus propias puertas.

Ejemplo: switch de 24 puertos; puertos 1-10 en VLAN 10 (Ventas, `10.0.10.0/24`), 11-20 en VLAN 20 (Invitados, `10.0.20.0/24`), 24 como trunk al router. Un invitado no puede hacer ARP a un PC de ventas: tiene que pasar por el router, que lo bloquea.

Ataque típico: VLAN hopping, saltar a otra VLAN abusando del trunk (switch spoofing o doble etiquetado); se previene desactivando la negociación automática de trunk y no usando la VLAN nativa para usuarios. Ver [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/).

```
VLAN → LAN lógica en un switch; dominio de broadcast propio; 802.1Q; IDs 1-4094.
Trunk → enlace que lleva varias VLAN etiquetadas.
```

### ARP

**ARP** (Address Resolution Protocol, RFC 826) es el protocolo que, dentro de una LAN, averigua qué dirección MAC corresponde a una dirección IP, mediante una pregunta en broadcast y una respuesta en unicast.

Existe porque IP decide a qué IP enviar, pero la trama Ethernet necesita una MAC destino. El host pregunta a todos "¿quién tiene 192.168.1.1? que me lo diga 192.168.1.20", solo el dueño responde "192.168.1.1 está en 11:22:33:44:55:66", y la respuesta se guarda en la caché ARP unos minutos.

Analogía: gritar en una sala "¿quién es Ana?" y que Ana levante la mano; desde entonces sabes dónde se sienta.

```
$ ip neigh
192.168.1.1 dev wlan0 lladdr 11:22:33:44:55:66 REACHABLE
192.168.1.50 dev wlan0 lladdr aa:bb:cc:dd:ee:ff STALE
```

Ejemplo de ataque: ARP no tiene autenticación, así que un atacante en la misma LAN puede enviar respuestas no pedidas diciendo que la IP del gateway es su MAC (ARP spoofing o poisoning) y ponerse en medio del tráfico (MITM). Defensa: Dynamic ARP Inspection en el switch y cifrado de extremo a extremo (TLS). Solo funciona dentro del dominio de broadcast; en IPv6 lo sustituye NDP.

### NAT

**NAT** (Network Address Translation, RFC 3022) es una función del router que reescribe las direcciones IP (y en su variante más común, los puertos) de los paquetes al pasar entre dos redes, típicamente para que muchos equipos con IP privada salgan a Internet con una sola IP pública.

Existe como parche al agotamiento de IPv4. La variante doméstica es PAT (Port Address Translation, también llamada NAT overload): el router anota en una tabla qué puerto público asignó a cada conexión interna y deshace la traducción en la respuesta.

Analogía: la centralita de una empresa: todas las llamadas salen con el mismo número público y la centralita recuerda qué extensión hizo cada llamada para pasarle la respuesta.

```
Tabla PAT del router (IP pública 203.0.113.5)
Interna              Pública               Destino
192.168.1.20:51000   203.0.113.5:40001     93.184.215.14:443
192.168.1.21:51000   203.0.113.5:40002     93.184.215.14:443
```

Variantes: NAT estático (una privada fija a una pública fija, para publicar un servidor), NAT dinámico (un grupo de públicas) y port forwarding (abrir un puerto público hacia un equipo interno; publicar el 3389 de escritorio remoto así es una causa clásica de intrusiones).

```
NAT → reescribe IP al cruzar el router; privada ↔ pública.
PAT → muchas privadas, una pública, distinguidas por puerto.
Port forwarding → puerto público fijo hacia un equipo interno.
```

### DHCP

**DHCP** (Dynamic Host Configuration Protocol) es el protocolo cliente-servidor que asigna automáticamente a cada equipo que se conecta su configuración de red: IP, máscara, default gateway y servidores DNS, por un tiempo limitado llamado lease (concesión).

Existe para no configurar a mano cientos de equipos y para no repetir direcciones. Sin DHCP, cada portátil que entra a una oficina necesitaría que alguien le escribiera una IP libre.

Analogía: el mostrador de un hotel que te da una habitación libre al llegar, la llave con su número y el horario del desayuno, y que recupera la habitación cuando te vas.

Ejemplo: al conectarte al Wi-Fi de una cafetería recibes `10.0.0.57/24`, gateway `10.0.0.1` y DNS `10.0.0.1` durante 2 horas.

### DNS

**DNS** (Domain Name System) es el sistema distribuido y jerárquico que traduce nombres de dominio legibles, como `ejemplo.com`, en direcciones IP y otros datos asociados al nombre.

Existe porque las personas recuerdan nombres y las máquinas necesitan IP, y porque una sola lista central (el antiguo archivo `HOSTS.TXT` de ARPANET) no escalaba. DNS reparte la responsabilidad: cada dueño de dominio publica sus propios registros.

Analogía: la agenda de contactos del teléfono: buscas "Mamá" y el teléfono marca el número.

Ejemplo: escribes `ejemplo.com` en el navegador; antes de cualquier conexión, el sistema pregunta al DNS y recibe `93.184.215.14`.

```
DHCP → entrega IP, máscara, gateway y DNS automáticamente, con lease.
DNS → traduce nombres a IP y otros datos; jerárquico y distribuido.
```

### DMZ

Una **DMZ** (Demilitarized Zone, zona desmilitarizada) es un segmento de red aislado, situado entre Internet y la red interna, donde se colocan los servidores que deben ser accesibles desde fuera (web, correo, DNS público), de modo que si uno es comprometido el atacante no queda dentro de la LAN.

Existe porque publicar un servidor web en la misma red que los PCs de la empresa significa que una vulnerabilidad en la web es una puerta a todo. Con DMZ, el firewall permite Internet → DMZ solo a puertos concretos (443), DMZ → LAN casi nada, y LAN → DMZ lo necesario para administrar (NIST SP 800-41 describe estas arquitecturas).

Analogía: la recepción de un banco: los clientes entran al vestíbulo, pero la bóveda está detrás de otra puerta con otra cerradura.

```
            Internet
               |
         [ Firewall ]------------ DMZ  (10.0.99.0/24)
               |                  web 443, correo 25
               |
          LAN interna (10.0.10.0/24)

Reglas:
  Internet → DMZ : solo 443 y 25
  DMZ → LAN      : denegado (salvo excepciones puntuales)
  LAN → DMZ      : SSH de administración desde 10.0.10.5
```

Ejemplo: un atacante explota el servidor web en `10.0.99.10`; intenta conectar a `10.0.10.20` (contabilidad) y el firewall lo bloquea porque DMZ → LAN está denegado. Ojo: lo que los routers domésticos llaman "DMZ host" es otra cosa, reenviar todos los puertos a un equipo interno, y expone ese equipo por completo.

### VPN

Una **VPN** (Virtual Private Network) es un túnel cifrado que transporta tráfico de una red privada a través de una red pública como Internet, de forma que los extremos se comunican como si estuvieran conectados a la misma red local.

Existe para dar confidencialidad e integridad sobre una red en la que no se confía, y para extender una red privada sin tender cables. Dos usos: site-to-site (une dos oficinas de forma permanente, router con router) y remote access (un empleado se conecta a la red de la empresa desde casa). Protocolos comunes: IPsec, OpenVPN y WireGuard (UDP 51820 por defecto). El cifrado se estudia en [`10-criptografia`](../10-criptografia/).

Analogía: un tubo opaco por el que pasan tus cartas a través de una plaza pública: la gente ve que hay un tubo, pero no qué viaja dentro.

Ejemplo: desde el Wi-Fi de un aeropuerto, el portátil levanta una VPN con la oficina y recibe `10.8.0.6`; ahora llega al servidor interno `10.0.10.50` y el Wi-Fi del aeropuerto solo ve tráfico cifrado hacia la IP pública de la empresa.

```
DMZ → segmento aislado para servidores públicos entre Internet y la LAN.
VPN → túnel cifrado sobre red pública; site-to-site o remote access.
```

### VM

Una **VM** (máquina virtual) es un equipo completo simulado por software sobre un equipo físico, con su propia tarjeta de red virtual e IP; se estudia en [`06-virtualizacion`](../06-virtualizacion/).

## Functions of each

### DHCP: el proceso DORA

La asignación DHCP (RFC 2131) es un intercambio de cuatro mensajes, conocido como **DORA** por sus iniciales (Discover, Offer, Request, Acknowledge), en el que un cliente sin IP obtiene su configuración de un servidor. El servidor escucha en UDP 67 y el cliente en UDP 68.

Funciona con broadcast porque al principio el cliente no tiene IP ni sabe dónde está el servidor:

```
Cliente (sin IP)                                      Servidor DHCP 192.168.1.1
     |                                                       |
     |-- DISCOVER  0.0.0.0:68 → 255.255.255.255:67 --------->|  "¿hay algún servidor?"
     |<-------- OFFER  ofrece 192.168.1.20, lease 24 h -------|  "te ofrezco esta IP"
     |-- REQUEST  broadcast: "acepto 192.168.1.20" --------->|  (broadcast para avisar
     |                                                       |   a otros servidores)
     |<-------- ACK  confirmado + máscara, gateway, DNS -------|  "es tuya"
     |                                                       |
```

Detalles del funcionamiento:

- El REQUEST va en broadcast para que, si respondieron varios servidores, los demás sepan que su oferta no fue elegida.
- Lease: al llegar al 50 % del tiempo el cliente intenta renovar directamente con el servidor (T1); al 87,5 %, con cualquiera (T2). Si caduca sin renovar, deja de usar la IP.
- Opciones más usadas: 1 (máscara), 3 (router/gateway), 6 (servidores DNS), 51 (duración del lease).
- Si el servidor está en otra subred, el router hace de DHCP relay (`ip helper-address` en Cisco), porque los broadcasts no cruzan routers.
- Si nadie responde, el equipo se asigna una dirección APIPA `169.254.x.x`.
- Una reserva DHCP ata una IP fija a una MAC concreta (para impresoras o servidores) sin configurarla a mano en el equipo.

Analogía: llegar a un hotel sin reserva: preguntas en voz alta en el vestíbulo si hay habitación (Discover), un recepcionista te ofrece la 20 (Offer), dices delante de todos "me quedo la 20" (Request) y te dan la llave (Ack).

Ejemplo: capturar el proceso en tu propia máquina al reconectar la interfaz:

```
$ sudo tcpdump -i wlan0 -n 'udp port 67 or udp port 68'
10:00:00.10 IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from aa:bb:cc:dd:ee:ff, length 300
10:00:00.12 IP 192.168.1.1.67 > 192.168.1.20.68: BOOTP/DHCP, Reply, length 300
10:00:00.13 IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from aa:bb:cc:dd:ee:ff, length 300
10:00:00.15 IP 192.168.1.1.67 > 192.168.1.20.68: BOOTP/DHCP, Reply, length 300
```

Las cuatro líneas son D, O, R y A (con `-v` tcpdump muestra el tipo de mensaje de cada una).

Ataques: rogue DHCP (un servidor falso que entrega como gateway la IP del atacante y lo pone en medio del tráfico) y DHCP starvation (pedir miles de IP con MAC falsas hasta agotar el rango). Defensa: DHCP snooping en el switch, que solo acepta respuestas DHCP por los puertos marcados como de confianza.

```
Discover → cliente busca servidor (broadcast).
Offer → servidor ofrece IP y lease.
Request → cliente acepta en broadcast.
Acknowledge → servidor confirma y entrega máscara, gateway, DNS.
DHCP puertos → servidor UDP 67, cliente UDP 68.
Lease → renueva al 50 % (T1) y al 87,5 % (T2).
DHCP relay → reenvía DHCP entre subredes.
DHCP snooping → solo puertos de confianza pueden responder DHCP.
```

### DNS: resolución y tipos de registro

La **resolución DNS** es el proceso de consultas (RFC 1034 y 1035) por el que un nombre se traduce en un registro, recorriendo una jerarquía de servidores de la raíz hacia abajo. Usa el puerto 53, UDP para consultas normales y TCP para respuestas grandes y transferencias de zona.

La jerarquía, leyendo el nombre de derecha a izquierda:

```
                       . (raíz, 13 identidades a.root-servers.net ... m.root-servers.net)
                 /     |      \
              com     org      cr          <- TLD (Top-Level Domain)
              /
         ejemplo                            <- servidor autoritativo de ejemplo.com
           /
        www                                  <- www.ejemplo.com → registro A
```

Pasos de una resolución completa de `www.ejemplo.com`:

```
PC ──(1) ¿www.ejemplo.com?──> Resolver recursivo (ISP, 1.1.1.1, 8.8.8.8)
                                 │ (2) ¿.com? ──────────> Raíz      → "pregunta a los de .com"
                                 │ (3) ¿ejemplo.com? ───> TLD .com  → "pregunta a ns1.ejemplo.com"
                                 │ (4) ¿www? ───────────> Autoritativo → "93.184.215.14, TTL 3600"
PC <─(5) 93.184.215.14 ──────────┘  (guarda en caché 3600 s)
```

- Consulta recursiva: el PC pide "dame la respuesta final" y el resolver hace todo el trabajo.
- Consulta iterativa: cada servidor responde "no lo sé, pregunta a este otro"; así trabaja el resolver con la raíz, el TLD y el autoritativo.
- Caché y TTL: cada respuesta trae un TTL en segundos; mientras no caduque, el resolver y el propio sistema responden sin volver a preguntar. Por eso un cambio de DNS tarda en "propagarse".
- Orden en el equipo: primero el archivo hosts, luego la caché local, luego el resolver configurado (por DHCP).

Tipos de registro que hay que conocer:

```
A      → nombre a IPv4                       www.ejemplo.com.  A     93.184.215.14
AAAA   → nombre a IPv6                       www.ejemplo.com.  AAAA  2001:db8::10
CNAME  → alias de otro nombre                blog.ejemplo.com. CNAME www.ejemplo.com.
MX     → servidor de correo, con prioridad   ejemplo.com.      MX    10 mail.ejemplo.com.
NS     → servidores autoritativos de la zona ejemplo.com.      NS    ns1.ejemplo.com.
TXT    → texto libre; SPF, DKIM, DMARC        ejemplo.com.      TXT   "v=spf1 mx -all"
PTR    → IP a nombre (resolución inversa)    14.215.184.93.in-addr.arpa. PTR www.ejemplo.com.
SOA    → datos de la zona: serial, tiempos   ejemplo.com.      SOA   ns1... admin... 2026100201
SRV    → servicio, puerto y host              _sip._tcp.ejemplo.com. SRV 10 5 5060 sip.ejemplo.com.
```

Analogía: preguntar una dirección en una ciudad desconocida. El recepcionista del hotel (resolver) no la sabe, pero sabe a quién preguntar: la oficina de turismo del país (raíz) te manda a la de la ciudad (TLD), que te manda al vecino del barrio (autoritativo), que sí sabe la casa exacta. El recepcionista apunta la respuesta para el siguiente huésped (caché).

Ejemplo con `dig`, consultando el registro A:

```
$ dig www.ejemplo.com A +noall +answer
www.ejemplo.com.   3600   IN   A   93.184.215.14
```

`dig +trace www.ejemplo.com` muestra los pasos 2, 3 y 4 uno por uno, y `dig -x 93.184.215.14` hace la consulta inversa (PTR).

Seguridad: DNS clásico va sin cifrar ni firmar, así que permite cache poisoning (envenenar la caché de un resolver con una respuesta falsa), spoofing en la LAN y DNS tunneling (sacar datos escondidos en consultas). DNSSEC firma las respuestas para garantizar que son auténticas; DoH (DNS over HTTPS, 443) y DoT (DNS over TLS, 853) las cifran. Desde el lado del atacante, los registros públicos (MX, TXT, subdominios) son la primera fuente de reconocimiento.

> [!NOTE]
> DNSSEC y DoH/DoT resuelven problemas distintos: DNSSEC prueba que la respuesta es auténtica pero no la oculta; DoH y DoT la ocultan en el camino pero no prueban que el servidor diga la verdad.

```
Resolver recursivo → hace todo el trabajo por el cliente y cachea.
Raíz → indica el servidor del TLD.
TLD → indica el autoritativo del dominio.
Autoritativo → tiene la respuesta final de su zona.
TTL → segundos que una respuesta puede quedarse en caché.
A / AAAA → nombre a IPv4 / IPv6.
CNAME → alias; MX → correo; NS → autoritativos; TXT → SPF/DKIM/DMARC.
PTR → IP a nombre; SOA → datos de la zona; SRV → servicio y puerto.
DNS puerto → 53 UDP y TCP; DoT 853; DoH 443.
DNSSEC → autenticidad; DoH/DoT → confidencialidad.
```

### NTP

**NTP** (Network Time Protocol, RFC 5905) es el protocolo que sincroniza los relojes de los equipos de una red con una fuente de hora precisa, con errores típicos de pocos milisegundos en Internet y por debajo de 1 ms en una LAN. Usa UDP 123.

Existe porque cada reloj de computador deriva (se adelanta o atrasa varios segundos al día) y muchas cosas dependen de que todos marquen la misma hora:

- Logs: para reconstruir un incidente hay que ordenar eventos de varios equipos; con relojes desfasados la línea de tiempo miente (ver [`16-respuesta-a-incidentes-y-forense`](../16-respuesta-a-incidentes-y-forense/)).
- Kerberos rechaza la autenticación si la diferencia supera 5 minutos por defecto.
- Certificados TLS: con la fecha mal, un certificado válido parece caducado o uno caducado parece válido.
- Códigos TOTP de doble factor cambian cada 30 segundos.

Cómo funciona: se organiza en estratos (stratum). Stratum 0 son los relojes de referencia (atómicos, GPS); stratum 1 son los servidores conectados directamente a ellos; cada nivel que sincroniza con el anterior suma 1, hasta 15. Stratum 16 significa "no sincronizado". El cliente envía una petición con su hora, el servidor responde con las suyas, y con los cuatro tiempos (T1 envío, T2 recepción en servidor, T3 respuesta del servidor, T4 recepción) calcula cuánto tarda el viaje y cuánto se desvía su reloj:

```
Retardo (delay)       = (T4 − T1) − (T3 − T2)
Desfase (offset)      = ((T2 − T1) + (T3 − T4)) / 2
```

T1 y T4 los mide el reloj del cliente y T2 y T3 el del servidor; el cálculo asume que ida y vuelta tardan lo mismo. El cliente corrige poco a poco (acelera o frena el reloj) en lugar de saltar, para no romper programas que miden tiempos.

Analogía: ajustar tu reloj de pulsera con el reloj de la plaza, descontando el tiempo que tardas en mirar y volver la vista.

Ejemplo con números: T1 = 10,000 s, T2 = 10,060 s, T3 = 10,061 s, T4 = 10,021 s. Retardo = 0,021 − 0,001 = 0,020 s (20 ms). Desfase = (0,060 + 0,040) / 2 = 0,050 s: el reloj del cliente va 50 ms atrasado.

```
$ timedatectl
               Local time: Fri 2026-10-02 10:00:00 CST
System clock synchronized: yes
              NTP service: active
```

Fuentes públicas: el proyecto NTP Pool (`pool.ntp.org`) agrupa miles de servidores voluntarios. Riesgos: un NTP falso puede retrasar el reloj para revivir certificados caducados, y los servidores NTP antiguos con el comando `monlist` se abusaron para ataques DDoS de amplificación. NTS (Network Time Security) añade autenticación.

```
NTP → sincroniza relojes; UDP 123.
Stratum → 0 referencia; 1 servidor conectado a ella; 16 = no sincronizado.
Offset → cuánto se desvía el reloj del cliente.
Delay → cuánto tarda el viaje de ida y vuelta.
Por qué importa → logs, Kerberos (5 min), TLS, TOTP (30 s).
```

### IPAM

**IPAM** (IP Address Management) es la práctica, y el software que la soporta, de planificar, registrar y vigilar todo el espacio de direcciones IP de una organización: qué subredes existen, qué IP está asignada a qué equipo, y cómo encajan con DHCP y DNS.

Existe porque en una red de miles de equipos la hoja de cálculo deja de servir: aparecen IP duplicadas, subredes solapadas y nadie sabe a quién pertenece `10.4.7.23`. Un IPAM integra el plan de subredes, los ámbitos de DHCP y las zonas DNS, de modo que asignar una IP actualiza también el registro DNS, y descubre por escaneo qué direcciones están realmente en uso.

Analogía: el catastro de una ciudad: dice qué parcela es de quién, cuáles están libres y quién construyó dónde, en vez de que cada vecino lleve su propia libreta.

Herramientas: NetBox (código abierto, inventario de red e IPAM), phpIPAM (código abierto), Infoblox y BlueCat (comerciales, IPAM + DHCP + DNS en uno, lo que se llama DDI) y el rol IPAM de Windows Server.

Ejemplo: una empresa con 30 sedes reserva `10.0.0.0/8` y asigna un `/16` a cada sede (`10.1.0.0/16` San José, `10.2.0.0/16` Heredia...) y dentro, un `/24` por VLAN. En el IPAM, buscar `10.2.20.15` devuelve "VLAN 20 Heredia, portátil de Juan, visto hace 3 minutos".

Importancia en seguridad: un inventario de direcciones fiable es el punto de partida para responder a un incidente (¿de quién es esta IP que habla con un servidor malicioso?) y para detectar equipos no autorizados: una IP activa que el IPAM no conoce es un equipo que nadie dio de alta.

```
DHCP → asigna IP automáticamente (DORA).
DNS → traduce nombres a IP (jerarquía, caché, registros).
NTP → sincroniza la hora (estratos, offset).
IPAM → inventario y planificación de todas las IP; integra DHCP y DNS.
```

## Recursos para aprender y practicar

### Videos

- [What is Subnetting? - Subnetting Mastery - Part 1 of 7](https://www.youtube.com/watch?v=BWZ-MHIhqjM) — Practical Networking; los siete datos de todo problema de subnetting, para Basics of Subnetting.
- [How to solve ANY Subnetting Problems in 60 seconds or less - Subnetting Mastery - Part 3 of 7](https://www.youtube.com/watch?v=5-wlfAdcmFQ) — Practical Networking; el método rápido con hoja de referencia; el resto de la serie (partes 2 y 4-7) sigue en el mismo canal, para Basics of Subnetting.
- [Calculating IPv4 Subnets and Hosts - CompTIA Network+ N10-009 - 1.7](https://www.youtube.com/watch?v=cYQOMifDlKI) — Professor Messer; cálculo de subredes y hosts, para Basics of Subnetting, CIDR y subnet mask.
- [Public vs Private IP Address](https://www.youtube.com/watch?v=po8ZFG0Xc4Q) — PowerCert Animated Videos; rangos privados y papel de NAT, para Public vs Private IP Addresses.
- [Address Resolution Protocol (ARP) in less than 5 minutes](https://www.youtube.com/watch?v=QPi5Nvxaosw) — Practical Networking; pregunta y respuesta ARP y la caché, para ARP y default gateway.
- [DHCP Explained - Dynamic Host Configuration Protocol](https://www.youtube.com/watch?v=e6-TaH5bkjo) — PowerCert Animated Videos; DORA y leases animados, para DHCP.
- [How DNS Works - Computerphile](https://www.youtube.com/watch?v=uOfonONtIuk) — Computerphile; resolución recursiva e iterativa, para DNS.
- [Network Time Protocol (NTP) - Computerphile](https://www.youtube.com/watch?v=BAo5C2qbLq8) — Computerphile; por qué y cómo se sincronizan los relojes, para NTP.

### Lectura y documentación

- [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918) — la fuente de los tres rangos privados.
- [IANA IPv4 Special-Purpose Address Registry](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml) — lista oficial de bloques especiales (loopback, APIPA, CGNAT, documentación); su marco es el [RFC 6890](https://www.rfc-editor.org/rfc/rfc6890).
- [RFC 4632: Classless Inter-domain Routing (CIDR)](https://www.rfc-editor.org/rfc/rfc4632) — notación de prefijo y agregación de rutas.
- [RFC 3021: Using 31-Bit Prefixes on IPv4 Point-to-Point Links](https://www.rfc-editor.org/rfc/rfc3021) — la excepción del /31.
- [RFC 6598](https://www.rfc-editor.org/rfc/rfc6598) y [RFC 3927](https://www.rfc-editor.org/rfc/rfc3927) — el bloque 100.64.0.0/10 de CGNAT y las direcciones link-local 169.254.0.0/16.
- [RFC 826: An Ethernet Address Resolution Protocol](https://www.rfc-editor.org/rfc/rfc826) y [RFC 3022: Traditional NAT](https://www.rfc-editor.org/rfc/rfc3022) — ARP y NAT en su definición original.
- [RFC 2131: Dynamic Host Configuration Protocol](https://www.rfc-editor.org/rfc/rfc2131) — DORA, leases y temporizadores T1/T2.
- [RFC 1034](https://www.rfc-editor.org/rfc/rfc1034) y [RFC 1035](https://www.rfc-editor.org/rfc/rfc1035) — conceptos y formato de DNS.
- [RFC 5905: Network Time Protocol Version 4](https://www.rfc-editor.org/rfc/rfc5905) — estratos, offset y delay.
- [What is DNS?](https://www.cloudflare.com/learning/dns/what-is-dns/) y [DNS records](https://www.cloudflare.com/learning/dns/dns-records/) — Cloudflare Learning; resolución y tipos de registro explicados con diagramas.
- [What is a subnet?](https://www.cloudflare.com/learning/network-layer/what-is-a-subnet/) — Cloudflare Learning; subredes y máscaras.
- [What is a VPN?](https://www.cloudflare.com/learning/access-management/what-is-a-vpn/) — Cloudflare Learning; túneles y cifrado.
- [NIST SP 800-41 Rev. 1: Guidelines on Firewalls and Firewall Policy](https://csrc.nist.gov/pubs/sp/800/41/r1/final) — arquitecturas con DMZ y políticas de firewall.
- [NTP Pool Project](https://www.ntppool.org/) — servidores NTP públicos y cómo usarlos.
- [NetBox](https://github.com/netbox-community/netbox) — IPAM e inventario de red de código abierto, para IPAM.
- [ip-address(8)](https://man.archlinux.org/man/ip-address.8), [dig(1)](https://man.archlinux.org/man/dig.1) e [ipcalc(1)](https://man.archlinux.org/man/ipcalc.1) — páginas de manual de las herramientas de los ejemplos.
- [Subnetting Mastery](https://www.practicalnetworking.net/stand-alone/subnetting-mastery/) — Practical Networking; la serie completa de 7 videos con la hoja de referencia para descargar.
- [IP Calculator (ipcalc)](https://jodies.de/ipcalc) — calculadora web para comprobar tus respuestas, no para sustituir el cálculo a mano.

### Práctica

- [subnettingpractice.com](https://subnettingpractice.com/) — preguntas de subnetting generadas sin fin, con solución; practica Basics of Subnetting.
- [subnetting.net](https://www.subnetting.net/) — preguntas de práctica, tutoriales y un juego contrarreloj; practica Basics of Subnetting y CIDR.
- [subnetipv4.com](https://subnetipv4.com/) — problemas aleatorios con autocorrección, del autor de Subnetting Mastery; practica los siete datos de un problema.
- [TryHackMe: Intro to LAN](https://tryhackme.com/room/introtolan) — subnetting, ARP y DHCP con preguntas; practica Basics of Subnetting, ARP y DHCP.
- [TryHackMe: Secure Network Architecture](https://tryhackme.com/room/introtosecurityarchitecture) — gratis; VLAN, DMZ, segmentación, firewalls y routing; practica VLAN, DMZ, Router y Switch. Para DNS a fondo: [DNS in Detail](https://tryhackme.com/room/dnsindetail) (gratis).
- [TryHackMe: DNS in Detail](https://tryhackme.com/room/dnsindetail) — jerarquía, tipos de registro y consultas; practica DNS.
- [TryHackMe: Introductory Networking](https://tryhackme.com/room/introtonetworking) — capas, IP y herramientas básicas (`ping`, `traceroute`, `dig`); practica IP y DNS.
- [HTB Academy: Introduction to Networking](https://academy.hackthebox.com/course/preview/introduction-to-networking) — direccionamiento IPv4 e IPv6, subnetting, MAC y terminología de protocolos desde la óptica de seguridad; practica Basics of Subnetting e IP Terminology.
- [Cisco Packet Tracer](https://www.netacad.com/cisco-packet-tracer) — arma un switch con dos VLAN, un router con un servidor DHCP por VLAN y una DMZ con un servidor web; comprueba con ping qué llega y qué no.
- Ejercicio en casa: ejecuta `sudo tcpdump -i <interfaz> -n -v 'udp port 67 or udp port 68'` en tu máquina, desconecta y reconecta la red, e identifica los cuatro mensajes DORA y las opciones 1, 3, 6 y 51 en la salida.
- Ejercicio en casa: `dig +trace` sobre un dominio tuyo o conocido para ver raíz, TLD y autoritativo; luego `dig MX`, `dig TXT` y `dig -x` sobre el mismo dominio.
- Ejercicio en casa: `ip route`, `ip neigh` y `ss -tln` en tu máquina; identifica tu gateway, la MAC del gateway en la caché ARP y qué servicios escuchan solo en 127.0.0.1.
- Ejercicio en casa: inventa 10 IP con prefijo, resuélvelas a mano y comprueba con `python3 -c 'import ipaddress as i; n=i.ip_interface("IP/prefijo").network; print(n, n.broadcast_address, n.num_addresses-2)'`.

## Cuadro resumen

Todo lo visto, en una línea por término.

IP Terminology

```
Subnet mask → 1s = red, 0s = host; IP AND máscara = dirección de red.
Octetos de máscara → 128, 192, 224, 240, 248, 252, 254, 255.
Wildcard mask → máscara invertida; la usan ACL y OSPF.
CIDR → /n bits de red; tamaño libre; 2^(32−n) direcciones; resume rutas (RFC 4632).
Clases A/B/C → tamaños fijos /8, /16, /24; obsoletas desde 1993.
Default gateway → router al que va todo lo que no es de mi subred; debe estar en mi subred.
Regla del AND → misma red: directo por ARP; distinta: al gateway.
Loopback → interfaz virtual lo; 127.0.0.0/8 y ::1; el tráfico no sale del equipo.
localhost → nombre que resuelve a 127.0.0.1 / ::1 vía /etc/hosts.
0.0.0.0 al escuchar → todas las interfaces; 127.0.0.1 → solo el propio equipo.
```

Basics of Subnetting

```
Subnetting → partir una red tomando bits de host para la red.
Hosts útiles → 2^h − 2; h = 32 − prefijo.
Subredes → 2^s; s = bits prestados.
Dirección de red → bits de host a 0; no asignable.
Broadcast → bits de host a 1; no asignable.
/31 → enlace punto a punto, 2 hosts (RFC 3021); /32 → un solo host.
Siete datos → máscara, red, broadcast, primer host, último host, nº hosts, subredes.
Octeto interesante → el primero de la máscara que no es 255 ni 0.
Bloque (número mágico) → 256 − octeto interesante de la máscara.
Red → múltiplo del bloque ≤ valor de la IP; broadcast → siguiente múltiplo − 1.
/24 → 254 hosts; /25 → 126; /26 → 62; /27 → 30; /28 → 14; /29 → 6; /30 → 2.
VLSM → subredes de tamaños distintos; asignar de mayor a menor.
```

Public vs Private IP Addresses

```
IP privada → RFC 1918; reutilizable; no enrutable en Internet; necesita NAT para salir.
IP pública → única en el mundo; asignada vía IANA/RIR; alcanzable desde Internet.
10.0.0.0/8 → 10.0.0.0 – 10.255.255.255.
172.16.0.0/12 → 172.16.0.0 – 172.31.255.255.
192.168.0.0/16 → 192.168.0.0 – 192.168.255.255.
169.254.0.0/16 → APIPA, sin DHCP.
100.64.0.0/10 → CGNAT del operador.
Bogon → dirección que nunca debería venir de Internet; se filtra en el borde.
NAT ≠ seguridad → oculta, pero no filtra lo que sale ni separa la LAN.
```

Understand the Terminology

```
IP → capa 3; dirección lógica y entrega entre redes; sin garantías.
IPv4 → 32 bits, decimal punteado, agotado; IPv6 → 128 bits, hexadecimal.
Switch → capa 2; reenvía tramas por MAC dentro de la LAN; tabla CAM.
Router → capa 3; une redes; enruta por IP; prefijo más largo gana.
VLAN → LAN lógica en un switch; dominio de broadcast propio; 802.1Q; IDs 1-4094.
Trunk → enlace que lleva varias VLAN etiquetadas.
ARP → IP a MAC dentro de la LAN; broadcast; sin autenticación (ARP spoofing).
NAT → reescribe IP al cruzar el router; privada ↔ pública.
PAT → muchas privadas, una pública, distinguidas por puerto.
Port forwarding → puerto público fijo hacia un equipo interno.
DHCP → entrega IP, máscara, gateway y DNS automáticamente, con lease.
DNS → traduce nombres a IP y otros datos; jerárquico y distribuido.
DMZ → segmento aislado para servidores públicos entre Internet y la LAN.
VPN → túnel cifrado sobre red pública; site-to-site o remote access.
VM → equipo simulado por software con red virtual (ver 06-virtualizacion).
```

Functions of each

```
Discover → cliente busca servidor (broadcast).
Offer → servidor ofrece IP y lease.
Request → cliente acepta en broadcast.
Acknowledge → servidor confirma y entrega máscara, gateway, DNS.
DHCP puertos → servidor UDP 67, cliente UDP 68.
Lease → renueva al 50 % (T1) y al 87,5 % (T2).
DHCP relay → reenvía DHCP entre subredes.
DHCP snooping → solo puertos de confianza pueden responder DHCP.
Resolver recursivo → hace todo el trabajo por el cliente y cachea.
Raíz → indica el servidor del TLD; TLD → indica el autoritativo.
Autoritativo → tiene la respuesta final de su zona.
TTL → segundos que una respuesta puede quedarse en caché.
A / AAAA → nombre a IPv4 / IPv6.
CNAME → alias; MX → correo; NS → autoritativos; TXT → SPF/DKIM/DMARC.
PTR → IP a nombre; SOA → datos de la zona; SRV → servicio y puerto.
DNS puerto → 53 UDP y TCP; DoT 853; DoH 443.
DNSSEC → autenticidad; DoH/DoT → confidencialidad.
NTP → sincroniza relojes; UDP 123.
Stratum → 0 referencia; 1 servidor conectado a ella; 16 = no sincronizado.
Offset → cuánto se desvía el reloj del cliente; Delay → ida y vuelta.
Por qué importa la hora → logs, Kerberos (5 min), TLS, TOTP (30 s).
IPAM → inventario y planificación de todas las IP; integra DHCP y DNS.
```
