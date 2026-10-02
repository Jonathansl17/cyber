# Defensa y hardening

## Conceptos previos

- Superficie de ataque: el conjunto de puntos por donde un atacante podría entrar (puertos abiertos, servicios, cuentas, programas instalados).
- Endpoint: cualquier equipo final que usa una persona o que presta un servicio: laptop, servidor, teléfono, máquina virtual.
- Servicio: programa que corre en segundo plano y normalmente escucha en un puerto de red (SSH en el 22, RDP en el 3389). Ver [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/).
- Malware: software escrito para dañar, espiar o tomar control de un equipo (virus, gusano, troyano, ransomware). Los tipos se detallan en [`11-ataques-y-amenazas`](../11-ataques-y-amenazas/).
- Firma (signature): patrón fijo que identifica algo conocido, como el hash de un archivo malicioso o una secuencia de bytes.
- Hash: resumen de longitud fija de un archivo; si cambia un solo byte, cambia el hash. Ver [`10-criptografia`](../10-criptografia/).
- Dirección MAC: identificador de 48 bits de una tarjeta de red (`3c:52:82:aa:10:5f`), usado en la capa 2.
- VLAN: red lógica separada dentro de un mismo switch.
- Mínimo privilegio y defensa en profundidad: principios de diseño explicados en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/).
- Falso positivo / falso negativo: alerta sobre algo inocente / algo malicioso que pasa sin alerta.
- RADIUS: protocolo que centraliza la autenticación de red (puertos UDP 1812 y 1813). Ver [`08-autenticacion`](../08-autenticacion/).
- Domain Controller y Active Directory: el servidor de Windows que guarda usuarios, equipos y políticas de una organización.

## Operating System Hardening

El **hardening del sistema operativo** es el proceso de configurar un sistema para que ofrezca la menor superficie de ataque posible: quitar lo que sobra, cerrar lo que no se usa y endurecer lo que queda.

Un sistema recién instalado viene pensado para funcionar en cualquier sitio, no para ser seguro en el tuyo: trae servicios activos por si acaso, cuentas por defecto, protocolos viejos habilitados por compatibilidad y permisos amplios. Cada una de esas cosas es una puerta. El hardening existe porque la mayoría de las intrusiones no usan una vulnerabilidad genial, sino una configuración descuidada: un servicio que nadie recordaba, una contraseña de fábrica, un parche que no se aplicó.

Analogía: es como mudarse a una casa y, antes de dormir, cambiar las cerraduras que dejó el dueño anterior, tapiar la puerta trasera que nadie usa y poner rejas en la ventana del sótano.

Cómo se hace, en orden:

1. Partir de una línea base (baseline) publicada en vez de improvisar. Las más usadas son los CIS Benchmarks (nivel 1: cambios seguros para casi cualquier equipo; nivel 2: más estrictos, pueden romper funciones) y las DISA STIG del gobierno de EE. UU.
2. Reducir: desinstalar paquetes y deshabilitar servicios que el equipo no necesita.
3. Cuentas: deshabilitar o renombrar cuentas por defecto, prohibir el inicio de sesión remoto de root/Administrator, aplicar contraseñas largas y MFA.
4. Red: cerrar puertos (ver [Port Blocking](#port-blocking)) y activar el firewall del host (ver [Host Based Firewall](#host-based-firewall)).
5. Actualizar: aplicar parches del sistema y de las aplicaciones (ver [Patching](#patching)).
6. Control de acceso obligatorio: SELinux o AppArmor en Linux encierran a cada servicio en un perfil, de modo que un servidor web comprometido no pueda leer `/etc/shadow` aunque corra como root. Esto es Mandatory Access Control (MAC), distinto del filtrado por dirección MAC de la sección siguiente.
7. Kernel y arranque: Secure Boot, cifrado de disco completo (LUKS, BitLocker), parámetros `sysctl` restrictivos.
8. Registro: dejar los logs encendidos y enviados a un servidor central (ver [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/)).

Ejemplo en un servidor Linux de laboratorio: primero se mira qué escucha, luego se apaga lo que sobra y se endurece SSH.

```bash
$ ss -tlnp
State   Recv-Q  Send-Q  Local Address:Port   Process
LISTEN  0       128     0.0.0.0:22           users:(("sshd",pid=612))
LISTEN  0       64      0.0.0.0:21           users:(("vsftpd",pid=640))
LISTEN  0       128     0.0.0.0:111          users:(("rpcbind",pid=401))

$ sudo systemctl disable --now vsftpd rpcbind
Removed "/etc/systemd/system/multi-user.target.wants/vsftpd.service".
Removed "/etc/systemd/system/multi-user.target.wants/rpcbind.service".
```

```
# /etc/ssh/sshd_config.d/10-hardening.conf
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
```

```
# /etc/sysctl.d/90-hardening.conf
kernel.kptr_restrict = 2          # oculta direcciones del kernel a usuarios
net.ipv4.tcp_syncookies = 1       # resiste inundaciones SYN
net.ipv4.conf.all.accept_redirects = 0
```

Para medir el resultado se usa un auditor como Lynis, que puntúa de 0 a 100:

```bash
$ sudo lynis audit system --quick
  Hardening index : 58 [###########         ]
  Suggestions     : 31
```

Tras aplicar los cambios el índice sube, por ejemplo de 58 a 77. El número en sí no es una meta oficial; sirve para comparar el mismo equipo antes y después.

En Windows el equivalente es aplicar una línea base de seguridad de Microsoft o del CIS mediante [Group Policy](#group-policy): deshabilitar SMBv1, exigir firma SMB, bloquear macros de Office descargadas de internet, activar BitLocker.

> [!TIP]
> Antes de deshabilitar nada, inventaria: `ss -tlnp` en Linux o `Get-NetTCPConnection -State Listen` en PowerShell. No se puede endurecer lo que no se sabe que existe.

```
Baseline (CIS/STIG) → la lista de configuraciones seguras de referencia.
Hardening           → aplicar esa lista y reducir la superficie de ataque.
MAC (SELinux)       → control de acceso obligatorio: encierra procesos aunque sean root.
Lynis               → auditor que mide cuánto se ha endurecido un Linux.
```

## Understand Hardening Concepts

### MAC-based

El **control basado en MAC** es un mecanismo de acceso a la red que permite o bloquea un dispositivo según la dirección MAC de su tarjeta de red.

Existe porque es lo primero que un switch o un punto de acceso ve de un equipo: antes de que tenga IP, ya tiene MAC. Se usa en dos sitios. En switches se llama port security: el puerto aprende qué MAC están autorizadas y reacciona si aparece otra. En Wi-Fi se llama filtrado MAC: el punto de acceso solo deja asociarse a una lista blanca.

Analogía: es como un portero que deja pasar a quien lleva el uniforme correcto. Funciona contra el despistado, pero cualquiera puede comprar el mismo uniforme.

Cómo funciona port security en un switch Cisco: se fija un máximo de direcciones por puerto y qué hacer si se supera. Las tres acciones son `protect` (descarta en silencio), `restrict` (descarta y registra) y `shutdown` (apaga el puerto, estado err-disabled; es la acción por defecto).

```
Switch(config)# interface fa0/5
Switch(config-if)# switchport mode access
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 2
Switch(config-if)# switchport port-security mac-address sticky
Switch(config-if)# switchport port-security violation shutdown

Switch# show port-security interface fa0/5
Port Security              : Enabled
Port Status                : Secure-shutdown
Violation Mode             : Shutdown
Maximum MAC Addresses      : 2
Total MAC Addresses        : 2
Security Violation Count   : 1
Last Source Address:Vlan   : 3c52.8211.0042:10
```

Ejemplo: en una oficina, el puerto 5 tiene un teléfono IP y una PC (2 MAC). Alguien conecta un switch casero con 3 equipos más; aparece una tercera MAC, el puerto se apaga y el contador de violaciones sube a 1.

> [!WARNING]
> La MAC viaja en claro en cada trama y se cambia con un comando (`ip link set dev wlan0 address 3c:52:82:aa:10:5f`). El filtrado MAC frena errores y curiosos, no a un atacante: nunca es la única barrera.

```
Port security  → límite de MAC por puerto de switch; protect / restrict / shutdown.
Filtrado MAC   → lista blanca de MAC en un punto de acceso Wi-Fi.
MAC spoofing   → cambiar la MAC propia para suplantar una autorizada.
```

### NAC-based

El **control de acceso a la red (NAC, Network Access Control)** es un sistema que decide si un dispositivo puede entrar a la red, y a qué parte, según quién es y en qué estado está.

Resuelve el problema que deja abierto el filtrado MAC: saber de verdad quién se conecta y si el equipo es sano. Un NAC pregunta dos cosas: identidad (usuario y equipo, normalmente con 802.1X y certificados o credenciales) y postura (posture assessment: ¿tiene antivirus activo?, ¿está parcheado?, ¿tiene el disco cifrado?). Según las respuestas asigna una VLAN: la corporativa, una de invitados o una de cuarentena donde solo alcanza el servidor de parches.

Analogía: es el control de un aeropuerto. No basta con llegar a la puerta; muestras pasaporte (identidad), pasas por el escáner (postura) y te mandan a la sala que te corresponde.

Cómo funciona por dentro, con 802.1X:

```
 Equipo (suplicante)      Switch / AP (autenticador)      Servidor NAC/RADIUS
        |                           |                              |
        |-- EAPOL-Start ----------->|                              |
        |<- pide identidad ---------|                              |
        |-- identidad + credencial->|-- Access-Request ----------->|
        |                           |        (verifica usuario y postura)
        |                           |<- Access-Accept + VLAN 30 ---|
        |<====== puerto abierto en VLAN 30 (o 99 = cuarentena) ===|
```

Hasta que llega el Access-Accept, el puerto solo deja pasar tramas EAPOL; no hay IP ni tráfico normal. Los productos conocidos son Cisco ISE, Aruba ClearPass y PacketFence (libre). Puede ser con agente (un programa en el equipo informa la postura) o sin agente (el NAC escanea el equipo desde fuera, útil para impresoras e IoT, que a veces caen en MAC Authentication Bypass, es decir, se autorizan por MAC porque no saben hablar 802.1X).

Ejemplo: una laptop con el antivirus apagado hace 802.1X con credenciales válidas. El NAC acepta la identidad pero la postura falla, así que la manda a la VLAN 99, donde solo ve el servidor de actualizaciones. Al reactivar el antivirus, el NAC la reautentica y la pasa a la VLAN 30.

```
MAC-based → decide por la dirección MAC, falsificable.
NAC-based → decide por identidad verificada + postura del equipo, y asigna VLAN.
802.1X    → el estándar que transporta esa autenticación en el puerto (ver EAP vs PEAP).
```

### Port Blocking

El **bloqueo de puertos** es una práctica de filtrado que impide el tráfico hacia o desde puertos TCP/UDP que no tienen un uso legítimo.

Cada puerto abierto es un servicio que alguien puede atacar. Bloquear lo innecesario reduce la superficie sin tener que confiar en que el servicio detrás sea seguro. Se aplica en dos direcciones. Entrada (ingress): que nadie de internet alcance servicios internos, como SMB (445), RDP (3389) o Telnet (23). Salida (egress): que un equipo infectado no pueda hablar hacia afuera por puertos raros, o que las PCs no envíen correo directo por el puerto 25 (solo el servidor de correo).

Analogía: un edificio con 65.535 puertas; dejas abiertas solo la principal y la de carga, y el resto con llave.

El criterio práctico es "denegar todo por defecto y permitir lo necesario" (default deny). Algunos bloqueos habituales en el perímetro:

- 23/TCP (Telnet), 21/TCP (FTP): protocolos en texto plano.
- 135-139/TCP-UDP y 445/TCP (RPC, NetBIOS, SMB): nunca expuestos a internet; WannaCry se propagó por 445 en 2017.
- 3389/TCP (RDP): solo detrás de VPN o de un [Jump Server](#jump-server).
- 25/TCP de salida: solo desde el servidor de correo.

Ejemplo con nftables en un servidor web de laboratorio que solo debe ofrecer 22, 80 y 443:

```bash
$ sudo nft add table inet filtro
$ sudo nft add chain inet filtro entrada '{ type filter hook input priority 0; policy drop; }'
$ sudo nft add rule inet filtro entrada ct state established,related accept
$ sudo nft add rule inet filtro entrada iif lo accept
$ sudo nft add rule inet filtro entrada tcp dport '{ 22, 80, 443 }' accept

$ nmap -p 21,22,80,443,445 10.0.0.10      # desde otra máquina del laboratorio
PORT    STATE    SERVICE
21/tcp  filtered ftp
22/tcp  open     ssh
80/tcp  open     http
443/tcp open     https
445/tcp filtered microsoft-ds
```

`filtered` significa que el paquete se descartó sin respuesta: el escáner no sabe si hay algo detrás.

```
Puerto cerrado  → nadie escucha; el sistema responde RST.
Puerto filtrado → un firewall descarta el paquete; no hay respuesta.
Ingress         → filtrar lo que entra a la red.
Egress          → filtrar lo que sale; frena exfiltración y propagación.
```

### Group Policy

**Group Policy** es el mecanismo de Windows y Active Directory que aplica configuraciones de forma centralizada a miles de usuarios y equipos mediante objetos llamados GPO (Group Policy Objects).

Sin él, endurecer 500 PCs significaría tocar 500 PCs a mano y que cada una acabe distinta. Con una GPO se define la regla una vez en el controlador de dominio y todos los equipos del alcance la aplican y la vuelven a aplicar si alguien la cambia localmente.

Analogía: es el reglamento interno de una empresa enviado a todas las sucursales; si un gerente local cambia una regla, en la siguiente revisión vuelve a quedar como dice el reglamento.

Cómo funciona por dentro:

- Una GPO tiene dos mitades: Configuración del equipo (se aplica al arrancar) y Configuración del usuario (al iniciar sesión).
- Se vincula a un sitio, a un dominio o a una unidad organizativa (OU). El orden de aplicación es LSDOU: Local, Site, Domain, OU. La última que se aplica gana, así que la GPO de la OU más específica manda sobre la del dominio, salvo que una GPO superior esté marcada como Enforced.
- Los equipos refrescan las políticas cada 90 minutos con un desfase aleatorio de hasta 30; los controladores de dominio, cada 5 minutos.

```
 Dominio corp.local  ── GPO "Base de seguridad" (longitud mínima 14, bloqueo tras 10 intentos)
   └── OU Servidores  ── GPO "Servidores" (deshabilita SMBv1, firewall estricto)
         └── OU Web   ── GPO "Web" (solo permite RDP desde el jump server)
 Un servidor en OU Web recibe las tres, en ese orden; si chocan, gana "Web".
```

Ejemplos de hardening típicos por GPO: longitud mínima de contraseña 14, bloqueo de cuenta tras 10 intentos fallidos, deshabilitar SMBv1, AppLocker (lista de programas permitidos), auditoría de creación de procesos con línea de comandos (genera el evento 4688, ver [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/)).

Para comprobar qué se aplicó en un equipo:

```
C:\> gpupdate /force
Updating policy...
Computer Policy update has completed successfully.
User Policy update has completed successfully.

C:\> gpresult /r /scope computer
    Applied Group Policy Objects
    -----------------------------
        Web
        Servidores
        Base de seguridad
```

> [!NOTE]
> Una GPO solo alcanza a equipos unidos al dominio. Las laptops personales, los equipos Linux o los que no ven al controlador de dominio necesitan otra herramienta (Intune, Ansible, la política local).

```
GPO      → objeto con un conjunto de configuraciones.
LSDOU    → orden de aplicación: Local, Site, Domain, OU; la última gana.
Enforced → impide que una GPO inferior la sobrescriba.
gpresult → muestra qué GPO se aplicaron a un equipo o usuario.
```

### Sinkholes

Un **sinkhole** es un mecanismo de redirección que desvía el tráfico destinado a un destino malicioso hacia un servidor controlado por los defensores, para cortarlo y para ver quién lo intentó.

Existe porque el malware casi siempre necesita llamar a casa: a su servidor de mando y control (C2) por nombre de dominio o IP. Si ese nombre se resuelve a una IP inofensiva que controlas, el malware se queda hablando con una pared, y cada equipo que llama a la pared es un equipo infectado que acabas de identificar.

Analogía: es cambiar el número de teléfono del jefe de una banda por el de la comisaría. Los miembros siguen llamando, nadie recibe órdenes y la policía anota quién llamó.

Hay dos variantes:

- DNS sinkhole: el servidor DNS interno responde a dominios de una lista negra con una IP propia (por ejemplo, `10.0.0.250`) o con `0.0.0.0`. Pi-hole hace exactamente esto contra anuncios; los firewalls de Palo Alto o el RPZ de BIND lo hacen contra dominios de C2.
- Sinkhole de enrutamiento (blackhole): un router anuncia la ruta de una IP o red maliciosa hacia una interfaz nula o hacia un servidor de análisis. Se usa también contra ataques DDoS.

```
 PC infectada ── "¿IP de c2-malo.example?" ──> DNS interno (lista de sinkhole)
      |                                             |
      |<──────────── "10.0.0.250" ──────────────────┘
      |
      └── conexión a 10.0.0.250 ──> servidor sinkhole (registra IP origen, hora)
                                    alerta al SOC: "10.0.5.23 está infectada"
```

Ejemplo de laboratorio con dnsmasq:

```bash
# /etc/dnsmasq.d/sinkhole.conf
address=/c2-malo.example/10.0.0.250

$ dig +short c2-malo.example @10.0.0.53
10.0.0.250
```

Caso real: en mayo de 2017 el ransomware WannaCry consultaba un dominio no registrado antes de cifrar; si el dominio respondía, se detenía. Un investigador lo registró y lo apuntó a un servidor propio, lo que frenó la propagación mundial y además dio la lista de IPs infectadas que llamaban.

```
DNS sinkhole     → responde dominios maliciosos con una IP controlada.
Blackhole route  → envía el tráfico de una IP a ninguna parte o a un analizador.
Valor defensivo  → corta el C2 y delata a los equipos infectados.
```

### ACLs

Una **ACL (Access Control List, lista de control de acceso)** es una lista ordenada de reglas que dice qué está permitido y qué está denegado sobre un recurso: paquetes que cruzan una interfaz o usuarios que tocan un archivo.

Existe porque el control de acceso necesita un sitio donde escribirse de forma explícita y revisable. El término se usa en dos mundos, y conviene distinguirlos:

- ACL de red: en routers, switches de capa 3 y listas de red en la nube. Cada regla compara campos del paquete (IP origen, IP destino, protocolo, puerto) y decide `permit` o `deny`.
- ACL de sistema de archivos: en NTFS (la DACL de cada archivo) o en Linux con POSIX ACL (`setfacl`). Cada entrada dice qué usuario o grupo puede leer, escribir o ejecutar.

Analogía: es la lista de invitados de una fiesta leída de arriba abajo por el portero. En cuanto encuentra tu nombre con "pasa" o "no pasa", deja de leer. Si no estás en la lista, no pasas.

Las reglas que hay que saber de las ACL de red:

1. Se evalúan de arriba abajo y se detienen en la primera coincidencia (first match). El orden importa.
2. Al final hay un deny implícito: lo que no coincide con nada, se descarta.
3. Son sin estado (stateless) en su forma clásica: no recuerdan conexiones, así que hay que permitir también la respuesta o usar la palabra `established`.
4. En Cisco, las estándar (números 1-99) solo miran la IP origen y se colocan cerca del destino; las extendidas (100-199) miran origen, destino, protocolo y puerto, y se colocan cerca del origen.

Ejemplo: la red de invitados `192.168.50.0/24` solo debe navegar por web y nunca tocar los servidores `10.0.10.0/24`.

```
R1(config)# ip access-list extended INVITADOS
R1(config-ext-nacl)# 10 deny   ip  192.168.50.0 0.0.0.255 10.0.10.0 0.0.0.255
R1(config-ext-nacl)# 20 permit udp 192.168.50.0 0.0.0.255 any eq 53
R1(config-ext-nacl)# 30 permit tcp 192.168.50.0 0.0.0.255 any eq 80
R1(config-ext-nacl)# 40 permit tcp 192.168.50.0 0.0.0.255 any eq 443
R1(config-ext-nacl)# exit
R1(config)# interface g0/1
R1(config-if)# ip access-group INVITADOS in

R1# show access-lists INVITADOS
Extended IP access list INVITADOS
    10 deny ip 192.168.50.0 0.0.0.255 10.0.10.0 0.0.0.255 (12 matches)
    20 permit udp 192.168.50.0 0.0.0.255 any eq domain (340 matches)
    30 permit tcp 192.168.50.0 0.0.0.255 any eq www (95 matches)
    40 permit tcp 192.168.50.0 0.0.0.255 any eq 443 (1210 matches)
```

Si la regla 10 estuviera después de una `permit ip any any`, nunca se evaluaría. Ejemplo de ACL de archivo en Linux: dar lectura a un auditor sin cambiar el grupo del archivo.

```bash
$ setfacl -m u:auditor:r /srv/informes/q3.pdf
$ getfacl /srv/informes/q3.pdf
# owner: ana
# group: finanzas
user::rw-
user:auditor:r--
group::r--
other::---
```

> [!WARNING]
> El error clásico es el orden: una regla amplia arriba tapa a las específicas de abajo. Las reglas específicas van primero y lo que no se nombra cae en el deny implícito.

```
ACL de red       → reglas permit/deny sobre paquetes; first match; deny implícito.
ACL estándar     → solo IP origen (1-99); cerca del destino.
ACL extendida    → origen, destino, protocolo, puerto (100-199); cerca del origen.
ACL de archivos  → quién puede leer/escribir/ejecutar un archivo (NTFS, setfacl).
Firewall stateful→ a diferencia de la ACL clásica, recuerda conexiones.
```

### Patching

El **patching (gestión de parches)** es el proceso continuo de encontrar, probar e instalar las actualizaciones que corrigen fallos de seguridad y errores del software.

Existe porque cada mes se publican miles de vulnerabilidades (CVE) y, una vez publicado un parche, los atacantes comparan la versión vieja con la nueva para entender el fallo y explotarlo en días. Un sistema sin parchear es una puerta cuya cerradura defectuosa ya está descrita en internet. La mayoría de las brechas grandes usan vulnerabilidades con parche disponible desde hace meses.

Analogía: es el retiro de un coche por defecto de fábrica. El fabricante ya avisó y ofrece el arreglo gratis; el riesgo está en no llevarlo al taller.

El ciclo, tal como lo describe NIST SP 800-40:

1. Inventario: saber qué equipos y qué versiones hay. Sin inventario no hay parcheo.
2. Priorizar: por gravedad (CVSS de 9,0 a 10 es crítico, de 7,0 a 8,9 alto), por exposición (¿da a internet?) y sobre todo por explotación real: el catálogo KEV de CISA lista las vulnerabilidades que ya se explotan.
3. Probar en un grupo pequeño (anillo piloto) para no romper producción.
4. Desplegar por anillos: piloto → 10 % → resto.
5. Verificar que se instaló y reiniciar si hace falta; un parche del kernel no protege hasta el reinicio.
6. Si no se puede parchear: mitigación temporal (deshabilitar la función, aislar el equipo, regla en el firewall) y documentar la excepción.

Microsoft publica sus parches el segundo martes de cada mes (Patch Tuesday). En Linux se revisa así:

```bash
$ apt list --upgradable 2>/dev/null | head -4
openssl/stable-security 3.0.15-1~deb12u1 amd64 [upgradable from: 3.0.13-1~deb12u1]
openssh-server/stable-security 1:9.2p1-2+deb12u4 amd64 [upgradable from: ...]
linux-image-amd64/stable-security 6.1.115-1 amd64 [upgradable from: 6.1.106-3]

$ sudo apt full-upgrade -y && sudo needrestart -r l
Running kernel version: 6.1.0-25 ; expected: 6.1.0-27  → reboot required
```

Ejemplo de priorización: llegan 40 parches. 3 son críticos y están en KEV en servidores expuestos a internet: esta semana. 12 altos en equipos internos: dentro del mes. 25 bajos: en la ventana trimestral.

```
Parche      → la corrección publicada.
CVSS        → puntuación de gravedad 0-10; 9,0+ crítico.
KEV (CISA)  → lista de vulnerabilidades explotadas de verdad; máxima prioridad.
Mitigación  → medida temporal cuando no se puede parchear todavía.
```

### Jump Server

Un **jump server** (o bastion host) es un equipo endurecido que funciona como único punto de entrada administrativo hacia una red protegida: los administradores entran primero en él y desde ahí saltan a los servidores internos.

Resuelve el problema de tener SSH o RDP abiertos en cada servidor. En lugar de 50 puertas, hay una: muy vigilada, con MFA, con cada sesión registrada y con los servidores internos configurados para aceptar administración solo desde la IP del bastión.

Analogía: es la recepción de un edificio de oficinas. Nadie sube directo a los pisos; todos pasan por el mostrador, enseñan identificación y quedan en el registro de visitas.

```
 Internet / red de usuarios
          |
          | SSH 22 o RDP 3389, con MFA
          v
   +--------------+     solo desde 10.0.1.5
   | Jump server  |-------------------------> 10.0.20.5 (BD)
   | 10.0.1.5     |-------------------------> 10.0.20.6 (app)
   | logs, MFA    |-------------------------> 10.0.20.7 (web)
   +--------------+
   Red de servidores 10.0.20.0/24: el firewall descarta SSH/RDP de cualquier otro origen.
```

Ejemplo con SSH, donde la opción `-J` (ProxyJump) atraviesa el bastión sin dejar claves privadas en él:

```bash
$ ssh -J ana@bastion.lab.example ana@10.0.20.5
ana@10.0.20.5's password:
Last login: Tue Sep 30 09:12:44 2026 from 10.0.1.5
```

El servidor interno registra que la conexión vino de `10.0.1.5`, y el bastión registra que fue `ana` quien la abrió. Reglas para que no se convierta en el punto débil: hardening máximo, sin navegación web ni correo, MFA obligatorio, parches al día, registro de sesiones enviado fuera del propio bastión.

> [!WARNING]
> Un bastión comprometido da acceso a todo lo que hay detrás. Concentrar el acceso solo mejora la seguridad si ese punto único está más protegido y más vigilado que todo lo demás.

```
Jump server / bastion → único punto de entrada administrativa, endurecido y auditado.
ProxyJump (ssh -J)    → atraviesa el bastión sin copiar claves a él.
PAW                   → estación de trabajo dedicada solo a administrar.
VPN                   → une al usuario a la red; no limita por sí sola a qué servidores llega.
```

### Endpoint Security

La **seguridad de endpoints** es el conjunto de controles instalados en cada equipo final para prevenir, detectar y responder a amenazas directamente en él.

Existe porque el perímetro ya no encierra a los equipos: las laptops viajan, trabajan desde casa y se conectan a Wi-Fi públicas. El equipo tiene que defenderse solo. Además, casi todo ataque termina ejecutando algo en un endpoint, así que es donde mejor se ve.

Analogía: si el firewall del perímetro es la muralla de la ciudad, la seguridad de endpoints es la cerradura, la alarma y la cámara de cada casa.

Las piezas que la componen, cada una explicada en su sección:

- [Antivirus](#antivirus) y [Antimalware](#antimalware): bloquean lo conocido.
- [EDR](#edr): registra el comportamiento y responde.
- [Host Based Firewall](#host-based-firewall) y [HIPS](#hips): filtran red y acciones en el propio equipo.
- [DLP](#dlp): evita que salgan datos sensibles.
- Cifrado de disco (BitLocker, LUKS): si roban la laptop, los datos no se leen.
- Lista de aplicaciones permitidas (AppLocker, WDAC): solo se ejecuta lo aprobado.
- [Patching](#patching) y configuración segura ([Operating System Hardening](#operating-system-hardening)).

La familia de productos se nombra así: EPP (Endpoint Protection Platform) agrupa la prevención (antivirus, firewall, control de dispositivos); EDR añade detección y respuesta; XDR extiende la misma idea a correo, red, nube e identidades en una sola consola.

Ejemplo: una laptop pierde el cargador en un hotel y alguien la roba. El cifrado de disco protege los datos; el EDR la marca como desconectada; el administrador revoca su certificado de VPN. Ningún control solo cubre todo el incidente.

```
EPP → prevención en el equipo (AV, firewall, control de USB).
EDR → detección + investigación + respuesta en el equipo.
XDR → EDR extendido a correo, red, nube e identidades.
```

### Ninguna capa basta sola

> [!IMPORTANT]
> Cada control de este tema falla en algún caso; la protección real sale de apilarlos para que el fallo de uno lo cubra el siguiente.

La defensa en profundidad es el principio que dice que los controles se ponen en capas independientes, de modo que un atacante tenga que vencerlas todas y que cada capa tenga una ocasión de detectarlo. Está explicada como principio en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/); aquí se aplica a los controles concretos de este tema.

```
                    Phishing con adjunto   Robo de laptop        Escaneo desde internet
Port Blocking       no aplica              no aplica             detiene el 445 y el 3389
NAC                 no aplica              no aplica             no aplica
Patching            cierra el exploit      no aplica             cierra el exploit
Antivirus/EDR       bloquea o detecta      no aplica             detecta el payload
Cifrado de disco    no aplica              protege los datos     no aplica
Fallo de 1 capa     lo cubre otra          lo cubre otra         lo cubre otra
```

La última fila es igual en las tres columnas: ahí está la idea entera. Ningún ataque es frenado por todas las capas, pero todos son frenados por alguna.

Ejemplo: un correo trae un documento con macro. La GPO bloquea macros de internet; si un usuario logra saltarlo, el antivirus reconoce el dropper; si es nuevo, el EDR ve a Word lanzando PowerShell; si llega a llamar a su C2, el sinkhole lo corta y delata al equipo. Cuatro oportunidades en lugar de una.

El límite: apilar controles sin vigilarlos solo suma ruido y costos. Cada capa tiene que generar alertas que alguien mire (ver [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/)).

```
Defensa en profundidad → capas independientes; el fallo de una lo cubre otra.
Control único          → un solo punto de fallo.
```

## Understand the following Terms

### Antivirus

Un **antivirus** es un programa de protección de endpoints que busca y bloquea software malicioso conocido comparando archivos y procesos con una base de firmas.

Nació en los años 80 contra virus que se copiaban en disquetes y sigue siendo la primera capa porque es barato: comparar un hash o un patrón de bytes contra una base de datos lleva microsegundos y frena la enorme masa de malware ya catalogado.

Analogía: es el cartel de "Se busca" en la comisaría. Reconoce al instante a quien ya está fichado, pero no a alguien con la cara nueva.

Cómo detecta:

- Firmas: hash exacto del archivo o secuencias de bytes características. Rápido, casi sin falsos positivos, pero cambiar un byte del malware cambia el hash.
- Heurística: reglas sobre rasgos sospechosos (empaquetado raro, importa funciones de inyección de código) para atrapar variantes.
- Análisis en tiempo real (on-access): revisa cada archivo al abrirlo o escribirlo. Análisis bajo demanda (on-demand): escaneo programado del disco.

Ejemplo de laboratorio con ClamAV y el archivo de prueba EICAR, una cadena inofensiva que todos los antivirus detectan por convenio:

```bash
$ printf 'X5O!P%%@AP[4\\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > eicar.txt
$ clamscan eicar.txt
eicar.txt: Eicar-Test-Signature FOUND
----------- SCAN SUMMARY -----------
Infected files: 1
```

```
Firma      → patrón de algo ya conocido; preciso, ciego ante lo nuevo.
Heurística → reglas sobre rasgos sospechosos; atrapa variantes, más falsos positivos.
On-access  → revisa al abrir/escribir. On-demand → escaneo programado.
```

### Antimalware

El **antimalware** es un software de protección que detecta y elimina cualquier tipo de programa malicioso, no solo virus: gusanos, troyanos, spyware, adware, rootkits, ransomware y programas potencialmente no deseados (PUP).

La distinción es histórica. "Antivirus" nació cuando la amenaza eran virus que infectaban archivos; cuando aparecieron el spyware y el adware de los 2000, surgieron herramientas separadas "antimalware" (Malwarebytes, Spybot) que cubrían lo que el antivirus ignoraba. Hoy los productos comerciales cubren ambas cosas y el nombre es casi marketing: Microsoft Defender Antivirus es, técnicamente, un motor antimalware.

Analogía: el antivirus es el médico que trata la gripe; el antimalware es el médico general que trata cualquier infección.

Lo que sí sigue siendo útil distinguir: el antimalware suele añadir detección de PUP (barras de navegador, mineros incluidos en instaladores), limpieza de persistencia (claves de registro, tareas programadas) y detección de rootkits, que se esconden del propio sistema operativo.

Ejemplo en Windows de laboratorio:

```
PS C:\> Start-MpScan -ScanType QuickScan
PS C:\> Get-MpThreatDetection | Select ThreatID, Resources, ActionSuccess
ThreatID   Resources                                    ActionSuccess
--------   ---------                                    -------------
2147735503 {file:_C:\Users\ana\Downloads\setup_pro.exe} True
```

```
Antivirus   → término histórico: virus que infectan archivos.
Antimalware → todo tipo de malware + PUP + rootkits; hoy suelen ser el mismo producto.
```

### EDR

Un **EDR (Endpoint Detection and Response)** es una plataforma de seguridad que registra de forma continua lo que pasa en cada endpoint, detecta comportamientos maliciosos en esa telemetría y permite responder desde una consola central.

Existe porque los atacantes aprendieron a evitar las firmas: malware nuevo cada vez, o directamente sin malware, usando herramientas legítimas del sistema (PowerShell, `certutil`, `wmic`), lo que se llama living off the land. El antivirus pregunta "¿este archivo es malo?"; el EDR pregunta "¿esto que está pasando es normal?".

Analogía: el antivirus es el guardia que compara caras con fotos de fichados; el EDR es la cámara que graba todo el edificio y avisa cuando alguien fuerza una puerta, sea quien sea, y además guarda la grabación para la investigación.

Cómo funciona por dentro:

1. Un agente en el equipo captura eventos: procesos creados (con padre e hijo y línea de comandos), conexiones de red, archivos escritos, cambios de registro, carga de módulos.
2. Envía la telemetría a la consola central (normalmente en la nube).
3. Reglas de comportamiento y modelos detectan cadenas sospechosas, a menudo mapeadas a técnicas de MITRE ATT&CK (ver [`13-frameworks-de-amenazas`](../13-frameworks-de-amenazas/)).
4. Respuesta: matar el proceso, poner el archivo en cuarentena, aislar el equipo de la red (solo puede hablar con la consola), abrir una consola remota, recoger evidencias.

Ejemplo de lo que vería un analista:

```
ALERTA  Severidad: Alta   Técnica: T1059.001 (PowerShell)
Host: PC-CONTA-07   Usuario: CORP\ana
WINWORD.EXE (pid 4120)
  └─ cmd.exe /c powershell -nop -w hidden -enc SQBFAFgAIAAoAE4A...
       └─ powershell.exe (pid 5288) → conexión TCP a 203.0.113.50:443
Acción tomada: proceso terminado, equipo aislado de la red.
```

Word no tiene por qué lanzar PowerShell oculto con un comando codificado en Base64. Ningún archivo de esa cadena tiene por qué estar en una base de firmas; lo que delata es el comportamiento. Productos: Microsoft Defender for Endpoint, CrowdStrike Falcon, SentinelOne; Wazuh y Velociraptor como opciones libres para laboratorio.

```
Antivirus → ¿este archivo coincide con algo conocido? Bloquea.
EDR       → ¿este comportamiento es normal? Registra, detecta, responde.
XDR       → EDR + correo + red + nube + identidad.
```

### Las firmas solo atrapan lo que ya se conoce

> [!IMPORTANT]
> Toda detección por firma llega tarde por diseño: alguien tuvo que ver el ataque antes para escribirla. Lo nuevo solo se atrapa mirando comportamiento o anomalías.

La detección por firmas es un método que compara lo observado contra patrones de amenazas ya catalogadas, y por eso su punto ciego es exactamente lo que nadie ha catalogado aún. Esta misma tensión aparece en el antivirus, en el HIPS y en los IDS de red de [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).

```
                         Malware conocido   Variante recompilada   Ataque con PowerShell legítimo
Antivirus por firma      lo bloquea         no lo ve               no lo ve
Heurística               lo bloquea         probablemente lo ve    a veces
EDR por comportamiento   lo ve              lo ve                  lo ve
Coste en falsos positivos bajo              medio                  alto
```

Las columnas segunda y tercera muestran el hueco: a medida que el ataque se aleja de lo catalogado, la firma deja de servir y solo queda el comportamiento, que a cambio genera más falsas alarmas.

Ejemplo: un atacante toma un ransomware conocido y cambia una cadena de texto. El hash cambia, las 60 firmas del hash dejan de coincidir, pero el comportamiento (renombrar 10.000 archivos en 2 minutos y borrar las copias de sombra con `vssadmin delete shadows`) es idéntico y el EDR lo detiene.

El límite: el comportamiento no reemplaza a las firmas. Las firmas son baratas y precisas contra el 90 % de lo que circula; se usan ambas.

```
Firma          → precisa y barata; ciega ante lo nuevo.
Comportamiento → ve lo nuevo; más falsos positivos y más caro de operar.
```

### DLP

La **prevención de pérdida de datos (DLP, Data Loss Prevention)** es un conjunto de herramientas y políticas que identifica información sensible y evita que salga de la organización por canales no autorizados.

Los controles anteriores protegen contra lo que entra; la DLP vigila lo que sale. Existe porque muchas fugas no son hackeos sino errores (un Excel de clientes enviado al correo personal) o empleados que se llevan datos al irse, y porque leyes como el RGPD o PCI DSS obligan a proteger ciertos datos (ver [`17-estandares-y-cumplimiento`](../17-estandares-y-cumplimiento/)).

Analogía: es el detector de un museo en la salida, no en la entrada. Le da igual qué traes; le importa que no te lleves un cuadro.

Cómo funciona:

- Dónde mira, según el estado del dato: en reposo (discos, carpetas compartidas, nube), en movimiento (correo, web, transferencias) y en uso (copiar al portapapeles, imprimir, pegar en un USB).
- Cómo reconoce lo sensible: expresiones regulares con validación (16 dígitos que pasan el algoritmo de Luhn = probable tarjeta), huellas (fingerprint) de documentos concretos, etiquetas de clasificación ("Confidencial") y diccionarios.
- Qué hace: registrar, avisar al usuario, cifrar automáticamente, pedir justificación o bloquear.

Ejemplo: una política dice "más de 5 números de tarjeta válidos en un mensaje hacia un dominio externo → bloquear y avisar". Un empleado adjunta un CSV con 200 tarjetas a un correo a Gmail; el mensaje no sale, el empleado ve un aviso y el equipo de seguridad recibe una alerta. Un correo interno con una sola tarjeta solo se registra.

> [!NOTE]
> La DLP de red no ve dentro del tráfico cifrado con TLS salvo que la organización inspeccione TLS; por eso hoy pesa más la DLP en el endpoint y en la nube.

```
DLP en endpoint → vigila USB, impresión, portapapeles en el equipo.
DLP de red      → vigila correo y web al salir.
DLP en la nube  → vigila archivos compartidos en SaaS.
Luhn            → prueba que distingue un número de tarjeta real de 16 dígitos al azar.
```

### Firewall & Nextgen Firewall

Un **firewall** es un dispositivo o programa de red que filtra el tráfico entre zonas aplicando reglas sobre direcciones, puertos y estado de las conexiones; un **next-generation firewall (NGFW)** es un firewall que además identifica aplicaciones, usuarios y contenido dentro del tráfico.

El firewall existe para separar zonas de confianza distinta (internet, DMZ, red interna) y que solo cruce lo explícitamente permitido. El NGFW aparece porque hoy casi todo viaja por los puertos 80 y 443: permitir "TCP 443" ya no dice nada, porque por ahí pasan la intranet, Dropbox, un túnel de un atacante y un juego en línea.

Analogía: el firewall clásico es un guardia que mira el remitente y la dirección del sobre; el NGFW abre el sobre, lee de qué se trata y sabe quién lo envía.

Cómo evolucionaron:

```
1. Filtro de paquetes (stateless)  → mira cada paquete solo: IP, puerto. Como una ACL.
2. Stateful                        → recuerda conexiones en una tabla; permite la respuesta
                                     de lo que salió, rechaza lo que llega sin haber pedido.
3. Proxy / application gateway     → termina la conexión y la rehace; entiende el protocolo.
4. NGFW                            → stateful + identificación de aplicación (App-ID)
                                     + usuario (integración con AD) + IPS integrado
                                     + filtrado de URL + inspección TLS.
```

La tabla de estado es la diferencia clave entre 1 y 2: si la PC `10.0.0.5:51000` abrió una conexión a `93.184.216.34:443`, el firewall anota esa pareja y deja pasar las respuestas que coinciden, sin necesitar una regla de entrada.

Ejemplo de regla NGFW: "El grupo Ventas puede usar `salesforce-base`, pero no `salesforce-upload` fuera del horario laboral; nadie puede usar `bittorrent` en ningún puerto". Un firewall clásico no puede expresar esa regla porque solo ve IP y puertos. Un firewall de aplicaciones web (WAF) es otra cosa: protege una aplicación web concreta de inyecciones y XSS (ver [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/)).

```
Stateless → decide paquete a paquete; no recuerda conexiones.
Stateful  → mantiene tabla de conexiones; permite respuestas automáticamente.
NGFW      → stateful + aplicación + usuario + IPS + URL + TLS.
WAF       → protege una aplicación web (capa 7 HTTP), no una red.
```

### HIPS

Un **HIPS (Host-based Intrusion Prevention System)** es un sistema de prevención que vive en un solo equipo y bloquea acciones sospechosas en él: llamadas al sistema, cambios de archivos críticos, modificaciones del registro o patrones de ataque en el tráfico que llega a ese host.

Existe porque un IDS/IPS de red solo ve paquetes, y muchas veces cifrados; el HIPS ve lo que pasa dentro del equipo después de descifrar. Su pariente pasivo es el HIDS (Host-based Intrusion Detection System), que solo alerta. Las versiones de red, NIDS y NIPS, se explican en [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).

Analogía: un sistema de alarma dentro de la caja fuerte, no en la puerta del banco; ve si alguien la manipula aunque haya entrado por un túnel.

Cómo trabaja: monitoriza la integridad de archivos (FIM, File Integrity Monitoring: compara hashes de `/etc/passwd` o de binarios del sistema con una línea base), revisa logs locales, intercepta llamadas al sistema y aplica reglas de comportamiento. Ejemplos: Wazuh/OSSEC (HIDS con respuesta activa), los módulos HIPS de las suites de endpoint, y fail2ban como HIPS sencillo basado en logs.

Ejemplo con fail2ban en un servidor de laboratorio: 5 fallos de SSH en 10 minutos desde la misma IP → bloqueo de 1 hora.

```
# /etc/fail2ban/jail.local
[sshd]
enabled  = true
maxretry = 5
findtime = 10m
bantime  = 1h

$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  `- Total failed:     47
`- Actions
   |- Currently banned: 1
   `- Banned IP list:   198.51.100.23
```

```
HIDS → en el host, detecta y alerta.
HIPS → en el host, detecta y bloquea.
NIDS / NIPS → lo mismo pero en la red; ver 15-deteccion-y-monitoreo.
FIM  → vigila cambios en archivos críticos comparando hashes.
```

### Host Based Firewall

Un **firewall basado en host** es un firewall que corre en el propio equipo y filtra el tráfico que entra y sale de ese equipo concreto.

El firewall de red protege la frontera, pero no ve el tráfico entre dos equipos de la misma red. Si una PC de la oficina se infecta, intentará saltar a las vecinas (movimiento lateral) por SMB o RDP sin pasar nunca por el firewall perimetral. El firewall del host cierra esa vía. Además viaja con la laptop: en la Wi-Fi del aeropuerto, es la única barrera.

Analogía: el firewall perimetral es la valla del barrio; el del host es la cerradura de cada casa. La valla no sirve de nada si el ladrón ya vive en el barrio.

En Windows es Windows Defender Firewall, con tres perfiles que se eligen según la red detectada: Domain (red corporativa), Private (casa) y Public (cafetería, el más estricto). En Linux, nftables/iptables directamente o con interfaces como ufw y firewalld. Una ventaja sobre el de red: puede filtrar por programa ("solo `firefox.exe` puede salir al 443").

Ejemplo con ufw en una laptop Linux:

```bash
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
$ sudo ufw enable
$ sudo ufw status verbose
Status: active
Default: deny (incoming), allow (outgoing), disabled (routed)
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    192.168.1.0/24
```

Y en Windows, bloquear SMB entrante entre estaciones de trabajo:

```
PS C:\> New-NetFirewallRule -DisplayName "Bloquear SMB entrante" -Direction Inbound `
        -Protocol TCP -LocalPort 445 -Action Block -Profile Domain,Private,Public
```

```
Firewall de red  → protege la frontera entre zonas.
Firewall de host → protege un equipo; frena movimiento lateral; viaja con la laptop.
Perfiles Windows → Domain, Private, Public (el más estricto).
```

### Sandboxing

El **sandboxing** es una técnica de aislamiento que ejecuta código no confiable en un entorno cerrado, con acceso restringido al resto del sistema, para que lo que haga no tenga efecto fuera.

Resuelve dos problemas distintos. En prevención, limita el daño: el navegador ejecuta cada pestaña en un proceso sandbox, así que un exploit en una web no llega al sistema de archivos. En análisis, permite observar: se ejecuta un adjunto sospechoso en una máquina desechable y se registra qué archivos crea, qué registros toca y a qué dominios llama.

Analogía: es el arenero de un parque. El niño puede hacer lo que quiera dentro; la arena no sale del cajón y al final se rastrilla y queda como nueva.

Mecanismos, de menos a más aislamiento:

- Del sistema operativo: seccomp (limita las llamadas al sistema de un proceso en Linux), AppArmor/SELinux, AppContainer en Windows, firejail.
- De aplicación: la sandbox de Chrome o de Firefox, el modo de vista protegida de Office.
- Máquinas desechables: Windows Sandbox (una VM ligera que se borra al cerrar), contenedores y VMs (ver [`06-virtualizacion`](../06-virtualizacion/)).
- Sandboxes de análisis de malware: ANY.RUN, Joe Sandbox, Cuckoo/CAPE. Su uso como herramienta de SOC está en [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).

Ejemplo: abrir un PDF dudoso con firejail, sin red y con un directorio personal vacío.

```bash
$ firejail --net=none --private zathura factura.pdf
Parent pid 8812, child pid 8813
Child process initialized in 41.22 ms
```

> [!WARNING]
> El malware moderno detecta sandboxes: comprueba si hay drivers de VirtualBox, si la máquina tiene menos de 2 núcleos o si nadie mueve el ratón, y en ese caso no hace nada. Un "no pasó nada en la sandbox" no prueba que el archivo sea inofensivo.

```
Sandbox de prevención → limita lo que puede hacer el código (navegador, seccomp).
Sandbox de análisis   → ejecuta malware para observarlo (ANY.RUN, Cuckoo).
Evasión de sandbox    → el malware se queda quieto si detecta un entorno de análisis.
```

### ACL

En la lista de términos del roadmap, ACL es el mismo concepto que [ACLs](#acls) de la parte anterior: una lista ordenada de reglas permit/deny sobre paquetes o sobre archivos, con evaluación de arriba abajo, primera coincidencia y deny implícito. La sección completa, con el ejemplo de router y de `setfacl`, está ahí.

Analogía de repaso: la lista de invitados que el portero lee de arriba abajo hasta encontrar tu nombre.

```
ACL → lista ordenada permit/deny; first match; deny implícito (detalle en ACLs).
```

### EAP vs PEAP

**EAP (Extensible Authentication Protocol)** es un marco de autenticación que transporta distintos métodos de verificación entre un dispositivo y un servidor; **PEAP (Protected EAP)** es uno de esos métodos, que primero crea un túnel TLS y dentro de él realiza una segunda autenticación más débil, normalmente usuario y contraseña.

EAP existe porque 802.1X (el control de acceso por puerto en redes cableadas y en Wi-Fi Enterprise) necesitaba un formato común sin atarse a un solo tipo de credencial. EAP no autentica por sí mismo: es el sobre, y el método (EAP-TLS, PEAP, EAP-TTLS...) es la carta. Está definido en el RFC 3748. En 802.1X hay tres roles: el suplicante (el equipo), el autenticador (switch o punto de acceso) y el servidor de autenticación (RADIUS). Entre equipo y switch el EAP viaja en tramas EAPOL; entre switch y RADIUS, dentro de RADIUS.

Analogía: EAP es la ventanilla estándar de una embajada; cada país (método) pide documentos distintos. EAP-TLS pide pasaporte biométrico a ambas partes; PEAP pone una mampara blindada (el túnel) y detrás de ella acepta un carné con contraseña.

Los métodos que hay que reconocer:

- EAP-TLS: certificados en el servidor y en cada cliente (RFC 5216). El más fuerte; exige una PKI para emitir certificados a todos los equipos.
- PEAP (normalmente PEAPv0/EAP-MSCHAPv2): certificado solo en el servidor; el túnel TLS protege el intercambio MSCHAPv2 de usuario y contraseña. Muy común porque reutiliza las cuentas de Active Directory.
- EAP-TTLS: similar a PEAP, admite más métodos internos.
- EAP-FAST: de Cisco, usa credenciales precompartidas (PAC) en lugar de certificados.
- LEAP: antiguo, de Cisco, roto; no debe usarse.

```
 Cliente                       AP                        RADIUS
   |-- EAPOL: identidad ------->|-- RADIUS: identidad ----->|
   |<========== túnel TLS (el cliente valida el certificado del servidor) ==========>|
   |      dentro del túnel: MSCHAPv2 con usuario y contraseña de AD                   |
   |<- EAP-Success -------------|<- Access-Accept + clave PMK --|
   |<====== 4-way handshake con el AP usando esa PMK (ver WPA) ======>|
```

Ejemplo: una universidad usa WPA2-Enterprise con PEAP. Cada alumno entra con su usuario y contraseña institucional; el servidor RADIUS presenta un certificado `radius.uni.example`. Si un atacante monta un punto de acceso falso con el mismo nombre de red (evil twin) y los teléfonos no validan el certificado, los alumnos le envían el intercambio MSCHAPv2 y el atacante puede intentar romperlo sin conexión. Con EAP-TLS ese ataque no obtiene nada reutilizable.

> [!WARNING]
> PEAP solo es seguro si el cliente valida el certificado del servidor. Si se configura "no validar certificado", el túnel TLS protege la conversación con el atacante.

```
EAP      → marco que transporta métodos de autenticación (no es un método).
802.1X   → control de acceso por puerto que usa EAP: suplicante, autenticador, RADIUS.
EAP-TLS  → certificados en ambos lados; el más fuerte.
PEAP     → túnel TLS con certificado del servidor + contraseña dentro (MSCHAPv2).
LEAP     → obsoleto y roto.
```

### WPS

**WPS (Wi-Fi Protected Setup)** es un mecanismo de configuración rápida de redes Wi-Fi que permite conectar un dispositivo sin escribir la contraseña, pulsando un botón en el router o introduciendo un PIN de 8 dígitos.

Nació en 2006 para que usuarios domésticos pudieran conectar impresoras y consolas sin teclear contraseñas WPA2 de 20 caracteres. Resolvió la comodidad a costa de la seguridad: el PIN acabó siendo un atajo para obtener la contraseña real de la red.

Analogía: es una caja de llaves junto a la puerta con un candado de combinación. Da igual lo buena que sea la cerradura de la casa si la caja se abre probando combinaciones.

Por qué el PIN es débil, con números:

- 8 dígitos parecerían 10^8 = 100.000.000 combinaciones.
- El último dígito es una suma de verificación de los otros 7: quedan 10^7.
- El router valida el PIN en dos mitades y responde por separado si la primera mitad (4 dígitos) es correcta: 10^4 = 10.000 intentos para la primera y 10^3 = 1.000 para la segunda.
- Total: 11.000 intentos como máximo. A unos pocos segundos por intento, horas.
- Además, en 2014 se publicó el ataque Pixie Dust: en muchos chipsets los números aleatorios del protocolo eran predecibles y el PIN se calcula sin conexión en segundos.

Una vez que el atacante tiene el PIN, el router le entrega la contraseña WPA2 de la red. El método de botón (push button) es menos grave porque exige presencia física y abre una ventana de 2 minutos.

Ejemplo: al revisar el panel de administración de tu router de casa encuentras "WPS: activado (PIN y botón)". Aunque la contraseña Wi-Fi tenga 20 caracteres, ese PIN deja la red a 11.000 intentos de distancia. La medida correcta es desactivar WPS por completo en el router, no solo el botón, y comprobar después que la opción sigue apagada tras cada actualización de firmware.

```
WPS PIN     → 8 dígitos validados en mitades: 11.000 intentos máximo.
Pixie Dust  → cálculo offline del PIN por aleatoriedad débil.
Push button → ventana de 2 minutos con presencia física; menos grave.
Remedio     → desactivar WPS por completo.
```

### WPA vs WPA2 vs WPA3 vs WEP

**WEP, WPA, WPA2 y WPA3** son las cuatro generaciones de protocolos de seguridad para redes Wi-Fi, que definen cómo se autentica un dispositivo y cómo se cifra el tráfico por el aire.

Existen porque en Wi-Fi cualquiera dentro del alcance recibe todas las tramas; sin cifrado, el vecino lee tu tráfico como si estuviera conectado al mismo cable. Cada generación nació para tapar los fallos de la anterior.

Analogía: son cuatro generaciones de candados para la misma puerta. El primero se abría con un clip; el segundo era el primero reforzado deprisa; el tercero es un buen candado que se puede forzar si la combinación es corta; el cuarto obliga al ladrón a probar cada combinación delante de la puerta, una por una.

Las cuatro generaciones, en orden:

- WEP (1999): cifrado RC4 con un vector de inicialización de solo 24 bits que se repite enseguida en una red con tráfico. Con suficientes paquetes capturados la clave se deduce en minutos. Está roto sin remedio y no debe usarse nunca.
- WPA (2003): solución de emergencia para el hardware de WEP. Sigue usando RC4, pero con TKIP, que cambia la clave por paquete. Mejoró mucho a WEP, pero TKIP también tiene debilidades conocidas y está obsoleto.
- WPA2 (2004): cifrado AES con CCMP, el estándar durante casi veinte años. Es sólido si la contraseña es larga. Su punto débil en modo Personal es que la contraseña se puede probar sin conexión (offline) a partir de un handshake capturado, así que una contraseña corta o de diccionario cae. En 2017 KRACK mostró un fallo de implementación del handshake, corregido con parches.
- WPA3 (2018): sustituye la clave compartida por SAE (Simultaneous Authentication of Equals, el intercambio "Dragonfly"). Con SAE capturar el intercambio no sirve para probar contraseñas offline: cada intento exige hablar con el punto de acceso. Añade secreto hacia adelante (si mañana se filtra la contraseña, el tráfico grabado hoy sigue protegido), cifrado de 192 bits en modo Enterprise y obliga a proteger las tramas de gestión (PMF), lo que corta los ataques de desautenticación.

Cada generación tiene dos modos:

- Personal (PSK o SAE): una contraseña compartida por todos. Para casas y oficinas pequeñas.
- Enterprise (802.1X): cada usuario se autentica con sus propias credenciales contra un servidor RADIUS, usando EAP (ver [EAP vs PEAP](#eap-vs-peap)). Para empresas: dar de baja a un empleado no obliga a cambiar la contraseña de todos.

El 4-way handshake es el intercambio de cuatro mensajes con el que WPA2 (y WPA3 tras SAE) confirman que cliente y punto de acceso conocen la misma clave maestra sin enviarla, y derivan de ella las claves de esa sesión:

```
Cliente                                      Punto de acceso
   |                                                |
   |  <-- 1. ANonce (número aleatorio del AP) ---   |
   |                                                |
   |  Cliente calcula la PTK = f(PMK, ANonce,       |
   |  SNonce, MAC del AP, MAC del cliente)          |
   |                                                |
   |  --- 2. SNonce + MIC (prueba de que sabe) -->  |
   |                                                |
   |                 AP calcula la misma PTK y comprueba el MIC
   |                                                |
   |  <-- 3. GTK cifrada + MIC (clave de grupo) --  |
   |                                                |
   |  --- 4. ACK (confirmación) ----------------->  |
   |                                                |
   Desde aquí todo el tráfico va cifrado con la PTK (unicast) y la GTK (broadcast)
```

Términos del diagrama: la PMK (Pairwise Master Key) es la clave maestra, que en modo Personal sale de la contraseña y del nombre de la red; los nonces son números aleatorios de un solo uso; la PTK (Pairwise Transient Key) es la clave de esa sesión para ese cliente; la GTK (Group Temporal Key) es la compartida para tráfico de difusión; el MIC es un código que prueba que el mensaje lo generó alguien que conoce la clave. La contraseña nunca viaja por el aire, pero en WPA2-Personal el MIC del mensaje 2 permite comprobar contraseñas candidatas sin conexión, y por eso la longitud de la contraseña lo es todo.

Ejemplo: una red WPA2-Personal con contraseña `casa2024` (8 caracteres, palabra más año) cae ante una prueba offline de diccionario en minutos; la misma red con una frase de 20 caracteres aleatorios necesitaría más tiempo del que existe el universo. En WPA3 ni siquiera la contraseña corta se puede atacar offline: cada intento obliga a hablar con el punto de acceso, que puede limitarlos.

> [!TIP]
> Configuración recomendada hoy: WPA3 (o modo de transición WPA2/WPA3 si hay dispositivos antiguos), WPS desactivado, contraseña de 15 caracteres o más, y WPA2/WPA3-Enterprise con 802.1X en cualquier empresa.

```
WEP   → RC4 con IV de 24 bits; roto en minutos. No usar.
WPA   → RC4 + TKIP; parche de emergencia, obsoleto.
WPA2  → AES-CCMP; sólido con contraseña larga; el handshake permite pruebas offline.
WPA3  → SAE; sin pruebas offline, secreto hacia adelante, PMF obligatorio.
Personal   → una contraseña para todos (PSK/SAE).
Enterprise → credenciales por usuario con 802.1X, EAP y RADIUS.
4-way handshake → deriva las claves de sesión sin enviar la contraseña.
```

## Honeypots

Un **honeypot** es un sistema señuelo, sin ningún uso legítimo, que se expone a propósito para atraer a los atacantes, detectarlos y estudiar lo que hacen sin arriesgar sistemas reales.

Analogía: el billete marcado que la policía deja en la caja: nadie honrado tiene motivos para tocarlo, así que quien lo toca se delata.

Por qué funciona: como nadie legítimo debería usar el honeypot, cualquier interacción con él es sospechosa por definición. Eso lo convierte en una de las alertas con menos falsos positivos que existen, al contrario que un IDS que tiene que separar lo malo de mucho tráfico normal.

Tipos, según cuánto se deja hacer al atacante:

- Baja interacción: simula servicios de forma superficial (un falso SSH que acepta cualquier contraseña y registra lo que se teclea). Fácil y seguro de montar; el atacante experto lo reconoce pronto.
- Alta interacción: un sistema real o casi real, vigilado al detalle. Enseña mucho más sobre las técnicas del atacante, pero exige aislarlo muy bien para que no sirva de trampolín hacia la red real.

La misma idea a otras escalas, todas parte de la "deception" o engaño defensivo:

- Honeynet: una red entera de honeypots que simula un entorno.
- Honeyfile: un archivo cebo ("contraseñas_admin.xlsx") que dispara una alerta si alguien lo abre.
- Honeytoken: un dato falso (una cuenta de usuario que nadie usa, una clave de API cebo) cuyo uso delata un acceso indebido.

Ejemplo: en la red interna se crea un servidor llamado `FS-BACKUP-02` que no usa nadie. Un martes a las 3:00 recibe un intento de inicio de sesión desde el portátil de contabilidad. Nadie de contabilidad tiene motivo para buscar ese servidor: o es un error, o alguien está explorando la red desde ese portátil, y hay que investigar ya.

> [!WARNING]
> Un honeypot mal aislado es un regalo para el atacante: si lo compromete, puede usarlo para saltar a la red real. Va en un segmento separado y con el tráfico saliente bloqueado.

```
Honeypot   → sistema señuelo; cualquier contacto es sospechoso.
Honeynet   → red de honeypots.
Honeyfile  → archivo cebo que alerta al abrirse.
Honeytoken → dato o credencial falsa cuyo uso delata un acceso.
```

## Recursos para aprender y practicar

### Videos

- [Hardening Techniques](https://www.youtube.com/watch?v=wXoC46Qr_9Q) — Professor Messer; hardening, port blocking, HIPS, EDR (Operating System Hardening, Hardening Concepts).
- [Operating System Security](https://www.youtube.com/watch?v=4dpTyRM6BU8) — Professor Messer; Group Policy, SELinux y hardening de sistemas.
- [Endpoint Security](https://www.youtube.com/watch?v=83pCkSSj1IQ) — Professor Messer; NAC, posture assessment, EDR y XDR.
- [Firewall Types](https://www.youtube.com/watch?v=mq1HRM-zGtQ) — Professor Messer; firewalls de red, NGFW y UTM.
- [Wireless Security Settings](https://www.youtube.com/watch?v=KaqKoKNEKnE) — Professor Messer; WPA2, WPA3, SAE, 802.1X y EAP.
- [Deception and Disruption](https://www.youtube.com/watch?v=X_qfMVty4ts) — Professor Messer; honeypots, honeynets, honeyfiles y honeytokens.
- [The 4-Way Handshake](https://www.youtube.com/watch?v=9M8kVYFhMDw) — CWNPTV; el handshake de WPA2 mensaje por mensaje.
- [Antivirus ≠ EDR. Stop Mixing Them Up.](https://www.youtube.com/watch?v=g-S9chO_hTQ) — Security Weekly; diferencia entre antivirus y EDR.

### Lectura y documentación

- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks) — guías de hardening paso a paso para cada sistema operativo y servicio.
- [DISA STIGs](https://public.cyber.mil/stigs/) y [STIG Viewer](https://www.stigviewer.com/) — las guías de hardening del Departamento de Defensa de EE. UU.
- [NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final) — seguridad general de servidores.
- [NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final) — gestión empresarial de parches (Patching).
- [NIST SP 800-41 Rev. 1](https://csrc.nist.gov/pubs/sp/800/41/r1/final) — guía de firewalls y sus políticas.
- [NIST SP 800-83 Rev. 1](https://csrc.nist.gov/pubs/sp/800/83/r1/final) — prevención y gestión de malware en endpoints.
- [NIST SP 800-153](https://csrc.nist.gov/pubs/sp/800/153/final) — seguridad de redes inalámbricas (WLAN).
- [Wi-Fi Alliance: Security](https://www.wi-fi.org/discover-wi-fi/security) — WPA3 explicado por quien lo certifica.
- [KRACK Attacks](https://www.krackattacks.com/) — el fallo del handshake de WPA2 explicado por su descubridor.
- [Microsoft: Group Policy overview](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/group-policy-overview) — cómo se aplican las GPO.
- [Windows Sandbox](https://learn.microsoft.com/en-us/windows/security/application-security/application-isolation/windows-sandbox/) — sandbox integrado en Windows para abrir archivos dudosos.
- [MITRE D3FEND](https://d3fend.mitre.org/) — catálogo de técnicas defensivas, el contrapunto de ATT&CK.
- [MITRE ATT&CK Mitigations](https://attack.mitre.org/mitigations/enterprise/) — qué control mitiga qué técnica.
- [CISA Known Exploited Vulnerabilities](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) — para priorizar qué parchear primero.

### Práctica

- [TryHackMe: Windows Fundamentals 1](https://tryhackme.com/room/windowsfundamentals1xbx) y [Windows Fundamentals 2](https://tryhackme.com/room/windowsfundamentals2x0x) — gratis; cuentas, UAC, configuración y herramientas del sistema que se endurecen.
- [TryHackMe: Linux Strength Training](https://tryhackme.com/room/linuxstrengthtraining) y [Linux Privilege Escalation](https://tryhackme.com/room/linprivesc) — gratis; permisos, SUID y configuraciones que el hardening debe cerrar.
- [TryHackMe: Network Security Essentials](https://tryhackme.com/room/networksecurityessentials) e [Intro to Endpoint Security](https://tryhackme.com/room/introtoendpointsecurity) — gratis; firewalls, segmentación y protección de endpoints.
- [TryHackMe: Intro to Antivirus](https://tryhackme.com/room/introtoav) — cómo detecta un antivirus.
- [TryHackMe: Introduction to Honeypots](https://tryhackme.com/room/introductiontohoneypots) — montar y leer un honeypot.
- [Ejercicio 1: Aplica un subconjunto del CIS nivel 1 a una VM](ejercicios.md#ejercicio-1-aplica-un-subconjunto-del-cis-nivel-1-a-una-vm) — endurecer una VM Ubuntu con siete grupos de controles CIS y medir con `ss`, `nmap` y Lynis la superficie antes y después.
- [Ejercicio 2: Audita el cifrado y el WPS de tu router](ejercicios.md#ejercicio-2-audita-el-cifrado-y-el-wps-de-tu-router) — demostrar con `nmcli` e `iw` que tu red solo anuncia AES-CCMP con WPA2 o WPA3 y que WPS está apagado.

## Cuadro resumen

Todo lo visto, en una línea por término.

Operating System Hardening

```
OS Hardening → reducir la superficie de ataque: quitar lo que sobra y endurecer lo que queda.
```

Understand Hardening Concepts

```
MAC-based        → permitir o bloquear dispositivos por dirección MAC; fácil de falsificar, es solo una capa.
NAC-based        → comprobar identidad y estado del equipo antes de dejarlo entrar a la red.
Port Blocking    → cerrar puertos y servicios que no se usan.
Group Policy     → aplicar configuración de seguridad centralizada a equipos Windows del dominio.
Sinkholes        → redirigir dominios o tráfico malicioso a un destino controlado.
ACLs             → listas de reglas que permiten o deniegan acceso a recursos o tráfico.
Patching         → aplicar actualizaciones que corrigen vulnerabilidades, priorizando las explotadas.
Jump Server      → único punto de entrada vigilado para administrar una zona protegida.
Endpoint Security→ conjunto de protecciones en cada equipo final.
Ninguna capa basta sola → defensa en profundidad aplicada al hardening.
```

Understand the following Terms

```
Antivirus     → detecta malware conocido, sobre todo por firmas.
Antimalware   → amplía el antivirus a más tipos de amenazas y técnicas de detección.
EDR           → registra y analiza el comportamiento de cada equipo y permite responder.
Firmas        → solo atrapan lo que ya se conoce; lo nuevo exige comportamiento.
DLP           → impide que datos sensibles salgan de donde deben estar.
Firewall      → filtra tráfico por IP, puerto y protocolo.
NGFW          → firewall que además entiende aplicaciones, usuarios y contenido.
HIPS          → detecta y bloquea ataques dentro de un equipo concreto.
Host FW       → firewall que corre en el propio equipo.
Sandboxing    → ejecutar algo dudoso en un entorno aislado y desechable.
ACL           → misma idea que ACLs aplicada a red y archivos.
EAP vs PEAP   → EAP es el marco de autenticación; PEAP lo envuelve en un túnel TLS.
WPS           → PIN de 8 dígitos validado en mitades: 11.000 intentos máximo; desactivarlo.
WEP           → RC4 con IV de 24 bits; roto en minutos. No usar.
WPA           → RC4 + TKIP; parche de emergencia, obsoleto.
WPA2          → AES-CCMP; sólido con contraseña larga; el handshake permite pruebas offline.
WPA3          → SAE; sin pruebas offline, secreto hacia adelante, PMF obligatorio.
4-way handshake → deriva las claves de sesión sin enviar la contraseña.
```

Honeypots

```
Honeypot   → sistema señuelo; cualquier contacto es sospechoso.
Honeynet   → red de honeypots.
Honeyfile  → archivo cebo que alerta al abrirse.
Honeytoken → dato o credencial falsa cuyo uso delata un acceso.
```
