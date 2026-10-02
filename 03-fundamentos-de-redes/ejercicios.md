# Ejercicios: Fundamentos de redes

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

El Ejercicio 1 se hace en Cisco Packet Tracer (simulador gratuito); es largo y está
dividido en partes. Los Ejercicios 2 y 3 se hacen en tu propio equipo o en VMs.

## Ejercicio 1: Estrella con switch y malla de cuatro routers en Packet Tracer

Nodo: [Network Topologies](README.md#network-topologies) (Star y Mesh),
[Encapsulamiento](README.md#encapsulamiento) y
[Un dispositivo trabaja en la capa más alta que lee](README.md#un-dispositivo-trabaja-en-la-capa-más-alta-que-lee).

Objetivo: montar una topología con una estrella (un switch con varios PC) y una malla
completa de cuatro routers, hacer que un PC llegue de punta a punta a un servidor del
otro extremo, y usar el modo Simulation para ver, capa por capa, cómo se encapsula cada
PDU (trama, paquete, segmento). Además comprobarás la redundancia de la malla: si cae un
enlace, el tráfico sigue por otro camino.

Necesitas: Cisco Packet Tracer instalado (descarga gratuita de Cisco Networking
Academy); una hora. Todo es simulado, no hay equipos reales. Modelos usados: switch
`2960`, routers `2911` con dos módulos serie `HWIC-2T` cada uno, PC y servidor genéricos.

### Parte A: topología física

1. Abre Packet Tracer y arrastra al lienzo estos dispositivos: un switch `2960`, cuatro
   routers `2911` (nómbralos R1, R2, R3, R4 al colocarlos), dos PC (PC1, PC2) y un
   servidor (Srv).

2. Apaga cada router y añádele dos módulos serie para tener puertos para la malla. Haz
   clic en el router, pestaña `Physical`, apaga el interruptor, arrastra un módulo
   `HWIC-2T` a dos ranuras libres y vuelve a encenderlo. Repite en los cuatro. Cada
   router queda con cuatro puertos serie: `Serial0/0/0`, `Serial0/0/1`, `Serial0/1/0` y
   `Serial0/1/1` (usarás tres).

3. Cablea la estrella (LAN de R1). Usa cable de cobre directo (`Copper Straight-Through`):
   - PC1 `FastEthernet0` → Switch `FastEthernet0/1`
   - PC2 `FastEthernet0` → Switch `FastEthernet0/2`
   - Switch `GigabitEthernet0/1` → R1 `GigabitEthernet0/0`

4. Cablea la LAN del otro extremo (en R4) con cable directo:
   - Srv `FastEthernet0` → R4 `GigabitEthernet0/0`

5. Cablea la malla completa de los cuatro routers con cable serie (`Serial DCE`). El
   extremo que conectes primero es el DCE y lleva el reloj; abajo se indica cuál es DCE.
   Son seis enlaces (una malla completa de 4 nodos tiene 4·3/2 = 6 enlaces):
   - R1 `Serial0/0/0` (DCE) → R2 `Serial0/0/0`
   - R1 `Serial0/0/1` (DCE) → R3 `Serial0/0/0`
   - R1 `Serial0/1/0` (DCE) → R4 `Serial0/0/0`
   - R2 `Serial0/0/1` (DCE) → R3 `Serial0/0/1`
   - R2 `Serial0/1/0` (DCE) → R4 `Serial0/0/1`
   - R3 `Serial0/1/0` (DCE) → R4 `Serial0/1/0`

### Parte B: plan de direcciones

Cada enlace entre routers es punto a punto y usa un `/30` (dos hosts útiles, justo lo que
hace falta). Las dos LAN usan `/24`.

```
LAN de R1 (estrella)   192.168.1.0/24   R1 Gi0/0 = 192.168.1.1
LAN de R4 (servidor)   192.168.4.0/24   R4 Gi0/0 = 192.168.4.1

Enlace R1-R2   10.0.12.0/30   R1 = .1 (DCE)   R2 = .2
Enlace R1-R3   10.0.13.0/30   R1 = .1 (DCE)   R3 = .2
Enlace R1-R4   10.0.14.0/30   R1 = .1 (DCE)   R4 = .2
Enlace R2-R3   10.0.23.0/30   R2 = .1 (DCE)   R3 = .2
Enlace R2-R4   10.0.24.0/30   R2 = .1 (DCE)   R4 = .2
Enlace R3-R4   10.0.34.0/30   R3 = .1 (DCE)   R4 = .2
```

PC1 = `192.168.1.10`, PC2 = `192.168.1.11`, máscara `255.255.255.0`, gateway
`192.168.1.1`. Srv = `192.168.4.10`, máscara `255.255.255.0`, gateway `192.168.4.1`. Esos
datos se ponen en cada equipo en `Desktop` → `IP Configuration`.

### Parte C: configuración IOS de los routers

Abre cada router, pestaña `CLI`, y pega su bloque. En los extremos serie marcados DCE se
pone `clock rate 64000`; en el otro extremo no. Se usa OSPF para que todos los routers
aprendan todas las redes y la malla dé caminos alternativos.

R1 (tiene la estrella y es DCE de sus tres enlaces):
```
enable
configure terminal
hostname R1
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface Serial0/0/0
 ip address 10.0.12.1 255.255.255.252
 clock rate 64000
 no shutdown
interface Serial0/0/1
 ip address 10.0.13.1 255.255.255.252
 clock rate 64000
 no shutdown
interface Serial0/1/0
 ip address 10.0.14.1 255.255.255.252
 clock rate 64000
 no shutdown
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.13.0 0.0.0.3 area 0
 network 10.0.14.0 0.0.0.3 area 0
end
write memory
```

R2:
```
enable
configure terminal
hostname R2
interface Serial0/0/0
 ip address 10.0.12.2 255.255.255.252
 no shutdown
interface Serial0/0/1
 ip address 10.0.23.1 255.255.255.252
 clock rate 64000
 no shutdown
interface Serial0/1/0
 ip address 10.0.24.1 255.255.255.252
 clock rate 64000
 no shutdown
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
 network 10.0.24.0 0.0.0.3 area 0
end
write memory
```

R3:
```
enable
configure terminal
hostname R3
interface Serial0/0/0
 ip address 10.0.13.2 255.255.255.252
 no shutdown
interface Serial0/0/1
 ip address 10.0.23.2 255.255.255.252
 no shutdown
interface Serial0/1/0
 ip address 10.0.34.1 255.255.255.252
 clock rate 64000
 no shutdown
router ospf 1
 router-id 3.3.3.3
 network 10.0.13.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
 network 10.0.34.0 0.0.0.3 area 0
end
write memory
```

R4 (tiene la LAN del servidor):
```
enable
configure terminal
hostname R4
interface GigabitEthernet0/0
 ip address 192.168.4.1 255.255.255.0
 no shutdown
interface Serial0/0/0
 ip address 10.0.14.2 255.255.255.252
 no shutdown
interface Serial0/0/1
 ip address 10.0.24.2 255.255.255.252
 no shutdown
interface Serial0/1/0
 ip address 10.0.34.2 255.255.255.252
 no shutdown
router ospf 1
 router-id 4.4.4.4
 network 192.168.4.0 0.0.0.255 area 0
 network 10.0.14.0 0.0.0.3 area 0
 network 10.0.24.0 0.0.0.3 area 0
 network 10.0.34.0 0.0.0.3 area 0
end
write memory
```

El switch 2960 no necesita configuración: todos sus puertos están en la VLAN 1 por
defecto y eso basta para la estrella.

### Parte D: comprobar conectividad

1. Espera a que OSPF converja (las interfaces serie pasan de triángulos ámbar a verdes
   en unos segundos). En R1, confirma que aprendió las redes del resto:
   ```
   R1# show ip route ospf
   ```
   Debes ver rutas marcadas con `O` hacia `192.168.4.0/24` y las redes de enlace que no
   son directas.

2. Comprueba las vecindades OSPF en R1: deben aparecer R2, R3 y R4 en estado `FULL`.
   ```
   R1# show ip ospf neighbor
   ```

3. Desde PC1, abre `Desktop` → `Command Prompt` y haz ping al servidor del otro extremo:
   ```
   PC> ping 192.168.4.10
   ```
   Debe responder. Eso prueba que el paquete cruzó la estrella, entró a R1, atravesó la
   malla y llegó a la LAN de R4.

### Parte E: observar el encapsulamiento en modo Simulation

1. Pasa a modo `Simulation` (botón abajo a la derecha). En `Edit Filters`, deja solo
   `ICMP` y `ARP` para no ver ruido.

2. Desde PC1 lanza un ping a Srv (`192.168.4.10`) con el botón `Add Simple PDU` (el sobre
   cerrado) o repitiendo el ping del Command Prompt. Avanza con `Capture / Forward`.

3. Haz clic en el sobre cuando está en PC1 y mira `Outbound PDU Details`. Lee de dentro
   hacia fuera las capas: los datos ICMP (capa 4/aplicación del eco), la cabecera IP con
   origen `192.168.1.10` y destino `192.168.4.10` (capa 3) y la cabecera Ethernet con
   las MAC (capa 2). Esa es la pila de encapsulamiento.

4. Avanza salto a salto. Fíjate en lo clave: al pasar por cada router, la **cabecera IP
   no cambia** (origen y destino siguen siendo los PC/servidor), pero la **cabecera de
   capa 2 se reconstruye** en cada enlace (en los serie es PPP/HDLC en vez de Ethernet).
   Es exactamente lo que dice la teoría: la MAC/trama cambia en cada salto, la IP se
   mantiene de extremo a extremo.

5. Prueba la redundancia de la malla. Vuelve a `Realtime`, entra a R1 y apaga el enlace
   directo a R4:
   ```
   R1# configure terminal
   R1(config)# interface Serial0/1/0
   R1(config-if)# shutdown
   R1(config-if)# end
   ```
   Vuelve a hacer ping desde PC1 a `192.168.4.10`: tras unos segundos de reconvergencia
   OSPF debe volver a responder, ahora pasando por R2 o R3 en vez del enlace caído.
   Reactívalo con `no shutdown` al terminar.

### Resultado esperado

Un archivo `.pkt` con la estrella y la malla funcionando, ping correcto de PC1 al
servidor de R4, el detalle de encapsulamiento visto capa por capa en Simulation, y la
prueba de que al caer un enlace de la malla el tráfico encuentra otro camino.

### Comprueba que lo lograste

- ¿Cuántos enlaces tiene la malla completa de 4 routers y por qué? Seis, porque
  n(n−1)/2 = 4·3/2 = 6.
- Al observar un salto en Simulation, ¿qué cabecera cambia y cuál se mantiene? Cambia la
  de capa 2 (trama) en cada enlace; se mantiene la de capa 3 (IP origen y destino).
- Cuando apagaste un enlace y el ping siguió funcionando, ¿qué propiedad de la malla
  demostraste? La redundancia: varios caminos entre los mismos nodos.
- ¿Por qué cada enlace entre routers usa `/30` y no `/24`? Porque solo hay dos extremos;
  un `/30` da exactamente dos hosts útiles y no desperdicia direcciones.

### Limpieza

No aplica a tu equipo: todo vive dentro del archivo de Packet Tracer. Guárdalo con un
nombre claro (por ejemplo `estrella-y-malla.pkt`) por si quieres repetir la práctica.

## Ejercicio 2: Lee las capas 2, 3 y 4 de una conexión con tcpdump

Nodo: [Encapsulamiento](README.md#encapsulamiento).

Objetivo: capturar en tu propia máquina el inicio de una conexión web y señalar en cada
línea qué parte corresponde a la capa 2 (MAC), a la capa 3 (IP) y a la capa 4 (puertos),
y comprobar que el campo `length` cuadra con la suma de las cabeceras. Al terminar sabrás
leer una captura separando las capas.

Necesitas: tu propio equipo o VM Linux con una interfaz de red activa; `tcpdump`
(instalar con `sudo apt install tcpdump` en Debian/Ubuntu o `sudo pacman -S tcpdump` en
Arch). Necesita `sudo` para capturar. Tiempo estimado: 20 minutos. Capturas tu propio
tráfico hacia una web; nada de interceptar a terceros.

### Pasos

1. Averigua el nombre de tu interfaz de red activa (la que tiene IP y está `UP`).
   ```bash
   ip -br addr
   ```
   Anota el nombre (por ejemplo `eth0`, `enp3s0` o `wlan0`).

2. Prepara la captura: una sola línea del inicio de una conexión al puerto 443, con la
   cabecera de capa 2 visible. La `-e` muestra las MAC; `-n` evita resolver nombres; `-c
   1` corta tras el primer paquete.
   ```bash
   sudo tcpdump -i eth0 -e -n -c 1 'tcp port 443'
   ```
   Sustituye `eth0` por tu interfaz. El comando se queda esperando.

3. En otra terminal (o en el navegador), genera tráfico hacia una web para que la
   captura atrape el primer paquete, el SYN que abre la conexión.
   ```bash
   curl -s https://example.com -o /dev/null
   ```

4. Vuelve a la captura. Verás una línea parecida a:
   ```
   12:00:01.000000 aa:bb:cc:00:00:01 > 11:22:33:44:55:66, ethertype IPv4 (0x0800),
     length 74: 192.168.1.10.51000 > 93.184.215.14.443: Flags [S], seq 1000,
     win 64240, length 0
   ```

5. Señala las tres capas en esa línea, de fuera hacia dentro:
   - Capa 2: las dos direcciones MAC (`aa:bb:cc:00:00:01 > 11:22:33:44:55:66`) y
     `ethertype IPv4`.
   - Capa 3: las dos IP (`192.168.1.10 > 93.184.215.14`).
   - Capa 4: los puertos (`.51000 > .443`) y la bandera `[S]` (SYN), que marca el inicio
     del handshake TCP.

6. Comprueba que el `length` cuadra. El primer `length 74` es el tamaño total de la
   trama a nivel de enlace; se descompone en 14 (Ethernet) + 20 (IP) + 40 (TCP: 20 fijos
   más 20 de opciones del SYN) = 74. El `length 0` del final indica que todavía no viajan
   datos de aplicación, lo normal en el SYN.

### Resultado esperado

Una línea de captura anotada por ti donde quede marcado qué es capa 2, qué es capa 3 y
qué es capa 4, más la cuenta de que `14 + 20 + 40 = 74` coincide con el `length`.

### Comprueba que lo lograste

- ¿Qué parte de la línea desaparecería si quitaras la opción `-e`? Las direcciones MAC y
  el `ethertype`, es decir, toda la capa 2.
- ¿Por qué el primer paquete tiene `length 0` al final? Porque es el SYN del handshake:
  abre la conexión y aún no transporta datos de aplicación.
- Si vieras `Flags [S.]` en vez de `[S]`, ¿qué sería? El SYN-ACK, la respuesta del
  servidor; `.` representa el ACK.

### Limpieza

No aplica: `tcpdump` solo lee tráfico y `curl` descartó la respuesta a `/dev/null`. No
queda nada instalado por el ejercicio salvo el propio `tcpdump`, que puedes conservar.

## Ejercicio 3: Comparte una carpeta por NFS y un disco por iSCSI

Nodo: [Basics of NAS and SAN](README.md#basics-of-nas-and-san).

Objetivo: exportar una carpeta por NFS desde una VM y montarla en otra (modelo NAS, a
nivel de archivo), luego presentar un disco por iSCSI entre las mismas VMs (modelo SAN,
a nivel de bloque) y anotar qué lado maneja el sistema de archivos en cada caso. Al
terminar entenderás en la práctica la diferencia entre archivo y bloque.

Necesitas: dos VM Linux propias (Debian/Ubuntu) en una red aislada host-only o internal,
una hará de servidor y otra de cliente; `sudo` en ambas. Paquetes: en el servidor
`nfs-kernel-server` y `targetcli-fb`; en el cliente `nfs-common` y `open-iscsi`. Tiempo
estimado: 45 minutos. Supón servidor `192.168.56.10` y cliente `192.168.56.20`.

### Parte A: NFS (nivel archivo, el modelo NAS)

1. En el servidor, instala NFS, crea la carpeta a compartir y ponle contenido.
   ```bash
   sudo apt install nfs-kernel-server
   sudo mkdir -p /srv/compartido
   echo "archivo servido por NFS" | sudo tee /srv/compartido/hola.txt
   ```

2. Exporta la carpeta hacia la red del cliente. Añade la línea al archivo de exports y
   recarga.
   ```bash
   echo "/srv/compartido 192.168.56.0/24(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
   sudo exportfs -ra
   sudo systemctl restart nfs-kernel-server
   ```
   `rw` da lectura y escritura; `sync` escribe antes de confirmar.

3. En el cliente, instala el soporte NFS y monta la carpeta.
   ```bash
   sudo apt install nfs-common
   sudo mkdir -p /mnt/nas
   sudo mount -t nfs 192.168.56.10:/srv/compartido /mnt/nas
   ls /mnt/nas
   ```
   Debe aparecer `hola.txt`. Acabas de acceder a nivel de archivo: pediste un archivo por
   su nombre y el servidor te lo entregó.

4. Observa quién maneja el sistema de archivos. En el cliente, mira el montaje:
   ```bash
   mount | grep nas
   ```
   Verás `type nfs`. El cliente no formateó nada: el servidor es el dueño del sistema de
   archivos y tú solo pides archivos. Ese es el comportamiento NAS.

### Parte B: iSCSI (nivel bloque, el modelo SAN)

1. En el servidor, instala `targetcli` y crea un archivo que hará de disco de respaldo
   (backstore) de 500 MB.
   ```bash
   sudo apt install targetcli-fb
   sudo mkdir -p /srv/iscsi
   sudo targetcli
   ```

2. Dentro de la consola interactiva de `targetcli`, crea el backstore, el target (IQN),
   el LUN y el permiso de acceso para el cliente. Escribe cada línea y termina con
   `exit`:
   ```
   /backstores/fileio create disk01 /srv/iscsi/disk01.img 500M
   /iscsi create iqn.2026-10.lab.servidor:disco1
   /iscsi/iqn.2026-10.lab.servidor:disco1/tpg1/luns create /backstores/fileio/disk01
   /iscsi/iqn.2026-10.lab.servidor:disco1/tpg1/acls create iqn.2026-10.lab.cliente:init01
   exit
   ```
   El IQN es el nombre único del target; el ACL autoriza al iniciador del cliente.

3. En el cliente, instala el iniciador iSCSI y ponle el mismo IQN que autorizaste.
   ```bash
   sudo apt install open-iscsi
   echo "InitiatorName=iqn.2026-10.lab.cliente:init01" | sudo tee /etc/iscsi/initiatorname.iscsi
   sudo systemctl restart iscsid
   ```

4. Descubre el target del servidor e inicia sesión en él.
   ```bash
   sudo iscsiadm -m discovery -t sendtargets -p 192.168.56.10
   sudo iscsiadm -m node -T iqn.2026-10.lab.servidor:disco1 -p 192.168.56.10 --login
   ```
   El `discovery` debe listar el IQN del servidor; el `--login` conecta.

5. Comprueba que apareció un disco nuevo. Es un bloque crudo, sin formato.
   ```bash
   lsblk
   ```
   Verás un disco nuevo (por ejemplo `sdb`) de 500 MB. Nadie le ha puesto sistema de
   archivos todavía: ese trabajo te toca a ti, el cliente.

6. Formatéalo y móntalo, porque en iSCSI el sistema de archivos lo maneja el host, no el
   servidor de almacenamiento.
   ```bash
   sudo mkfs.ext4 /dev/sdb
   sudo mkdir -p /mnt/san
   sudo mount /dev/sdb /mnt/san
   mount | grep san
   ```
   Aquí está la diferencia clave: en NFS el servidor ya tenía el sistema de archivos; en
   iSCSI tú lo creaste con `mkfs`. Ese es el comportamiento SAN.

### Resultado esperado

Una carpeta NFS montada donde el cliente pide archivos por nombre, y un disco iSCSI que
el cliente tuvo que formatear él mismo, más una nota tuya que diga: "NFS = nivel archivo,
el servidor maneja el sistema de archivos; iSCSI = nivel bloque, el host cliente maneja
el sistema de archivos".

### Comprueba que lo lograste

- ¿En qué caso tuviste que ejecutar `mkfs` y por qué? En iSCSI, porque la SAN entrega
  bloques crudos y el sistema de archivos lo pone el host.
- ¿En cuál pediste un archivo por su nombre sin preocuparte de dónde estaban los bloques?
  En NFS, el modelo NAS a nivel de archivo.
- ¿Por qué iSCSI suele ir en una red separada de la de usuarios? Para que el tráfico de
  disco (bloques) no compita con el tráfico normal y mantenga el rendimiento.

### Limpieza

En el cliente: desmonta y cierra sesión iSCSI.
```bash
sudo umount /mnt/san
sudo umount /mnt/nas
sudo iscsiadm -m node -T iqn.2026-10.lab.servidor:disco1 -p 192.168.56.10 --logout
```
En el servidor: quita la exportación NFS (borra la línea de `/etc/exports` y
`sudo exportfs -ra`) y elimina el target iSCSI entrando de nuevo a `targetcli` con
`/iscsi delete iqn.2026-10.lab.servidor:disco1`. Desinstala los paquetes si no los vas a
reutilizar.
