# Protocolos y puertos

## Conceptos previos

- Protocolo: conjunto de reglas que fija qué mensajes se mandan dos programas, en qué orden y con qué formato.
- Cliente / servidor: el cliente pide (tu navegador), el servidor atiende (la máquina que aloja la web).
- Dirección IP: número que identifica a un equipo en la red (`192.168.1.10`); se ve a fondo en [`04-direccionamiento-ip`](../04-direccionamiento-ip/).
- Puerto: número de 0 a 65535 que identifica, dentro de un equipo, a qué programa va dirigido un paquete.
- Socket: la combinación IP + puerto + protocolo de transporte (`192.168.1.10:443/tcp`).
- TCP: protocolo de transporte orientado a conexión; numera los bytes, confirma lo recibido y reenvía lo perdido.
- UDP: protocolo de transporte sin conexión; manda datagramas sueltos sin confirmar nada, más rápido y más simple.
- Modelo OSI / TCP-IP: las capas en que se divide la comunicación; se ven en [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/).
- En claro (plaintext): datos que viajan sin cifrar; cualquiera que capture el tráfico los lee.
- Cifrado simétrico / asimétrico, hash, firma digital: se ven a fondo en [`10-criptografia`](../10-criptografia/); aquí solo se usan.
- RTT (round-trip time): lo que tarda un mensaje en ir y volver; si el servidor está a 50 ms, un RTT son 100 ms.
- Daemon / servicio: programa que corre en segundo plano escuchando en un puerto (`sshd`, `httpd`).

## Common Protocols and their Uses

Un **protocolo de aplicación** es un acuerdo de comunicación de la capa 7 que define cómo dos programas intercambian un tipo concreto de información: páginas web, correo, nombres de dominio, archivos, hora o gestión de equipos. Existen porque TCP y UDP solo mueven bytes; no saben qué es un correo o una página. Cada servicio necesita su propio "idioma" encima del transporte.

Analogía: TCP/UDP son el servicio de mensajería que lleva cajas de un edificio a otro; el protocolo de aplicación es el idioma de la carta que va dentro. El cartero no lee la carta, pero el destinatario tiene que entender el idioma.

Los protocolos que se piden en Security+ y Network+ se agrupan por para qué sirven. Para cada uno importa: qué hace, si viaja en claro, y cuál es su reemplazo seguro.

Web:

- HTTP (HyperText Transfer Protocol): pide y entrega recursos web (HTML, imágenes, JSON) con métodos `GET`, `POST`, `PUT`, `DELETE`. En claro. Se desarrolla en "HTTP / HTTPS".
- HTTPS: HTTP dentro de un túnel TLS. Cifrado e integridad.

Correo:

- SMTP (Simple Mail Transfer Protocol): envía correo, de cliente a servidor y entre servidores. En claro salvo que se use STARTTLS (subir a TLS dentro de la misma conexión) o TLS implícito.
- POP3 (Post Office Protocol v3): descarga el correo al cliente y normalmente lo borra del servidor. En claro; versión segura POP3S.
- IMAP (Internet Message Access Protocol): lee el correo dejándolo en el servidor y sincroniza carpetas entre dispositivos. En claro; versión segura IMAPS.

Nombres y direcciones:

- DNS (Domain Name System): traduce nombres (`ejemplo.com`) a direcciones IP. Usa UDP para consultas normales y TCP cuando la respuesta es grande o en transferencias de zona (copiar la base completa de un servidor DNS a otro). Versiones cifradas: DoT (DNS over TLS) y DoH (DNS over HTTPS).
- DHCP (Dynamic Host Configuration Protocol): asigna automáticamente IP, máscara, puerta de enlace y DNS a los equipos que se conectan. Proceso DORA: Discover, Offer, Request, Acknowledge.
- NTP (Network Time Protocol): sincroniza relojes. Importa en seguridad porque los logs, Kerberos y los certificados dependen de la hora correcta: Kerberos rechaza tickets si el reloj difiere más de 5 minutos.

Acceso remoto:

- Telnet: terminal remota en claro, contraseña incluida. Obsoleto; se reemplaza por SSH.
- SSH (Secure Shell): terminal remota cifrada. Se desarrolla en "SSH".
- RDP (Remote Desktop Protocol): escritorio gráfico remoto de Windows. Se desarrolla en "RDP".
- VNC (Virtual Network Computing): escritorio remoto multiplataforma; muchas implementaciones sin cifrado fuerte.

Transferencia de archivos:

- FTP: transfiere archivos en claro con dos canales (control y datos).
- SFTP: transferencia de archivos dentro de SSH.
- FTPS: FTP envuelto en TLS.
- TFTP (Trivial FTP): versión mínima sobre UDP, sin autenticación; se usa para arrancar equipos por red o copiar configuraciones de routers.
- SMB (Server Message Block): compartir archivos e impresoras en redes Windows (carpetas `\\servidor\recurso`). SMBv1 está obsoleto y fue la puerta de WannaCry (EternalBlue, 2017).

Directorio y autenticación:

- LDAP (Lightweight Directory Access Protocol): consulta directorios de usuarios y grupos, como Active Directory. En claro; versión segura LDAPS (LDAP sobre TLS).
- Kerberos: autenticación por tickets en dominios Windows. Se ve en [`08-autenticacion`](../08-autenticacion/).
- RADIUS y TACACS+: autenticación centralizada para VPN, Wi-Fi empresarial y equipos de red. RADIUS cifra solo la contraseña; TACACS+ cifra el paquete completo.

Gestión y monitoreo:

- SNMP (Simple Network Management Protocol): consulta y modifica el estado de routers, switches e impresoras. v1 y v2c usan una "community string" en claro como contraseña (por defecto `public`); v3 agrega autenticación y cifrado.
- Syslog: envía mensajes de log a un servidor central. En claro por UDP; existe syslog sobre TLS.

VPN y voz:

- IPsec: cifra a nivel de capa 3 (IP); usa IKE para negociar claves. Se ve en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).
- SIP (Session Initiation Protocol): establece llamadas de voz/video (VoIP); el audio viaja aparte por RTP, y su versión cifrada es SRTP.

Ejemplo: abres el correo en el celular a las 8:00. El teléfono pide IP por DHCP, pregunta por DNS la IP de `mail.empresa.com`, sincroniza la hora por NTP, se conecta por IMAPS al puerto 993 para leer y por SMTP con STARTTLS al 587 para enviar. Seis protocolos en dos segundos, todos encima de TCP o UDP.

```
Telnet → terminal remota en claro.
SSH → terminal remota cifrada.
POP3 → descarga y borra del servidor.
IMAP → lee dejando el correo en el servidor, sincroniza carpetas.
SMTP → envía correo.
SNMP v1/v2c → gestión con community string en claro; v3 → autenticado y cifrado.
RADIUS → cifra solo la contraseña; TACACS+ → cifra todo el paquete.
```

### Todo protocolo en claro tiene su gemelo cifrado

> [!IMPORTANT]
> Casi cada protocolo clásico nació en claro y tiene una versión segura con otro puerto; en el examen, "elige el protocolo seguro" se resuelve recordando el par.

Un protocolo en claro es un protocolo que manda credenciales y datos sin cifrar, y su gemelo seguro es la misma función envuelta en TLS o SSH. Se diseñaron en los 70-90, cuando la red era de confianza; hoy cualquiera en la misma Wi-Fi con un sniffer lee lo que viaja en claro.

```
En claro (puerto)       Seguro (puerto)              Cómo se asegura
Telnet  23/tcp          SSH 22/tcp                   protocolo distinto, cifrado propio
FTP     20-21/tcp       SFTP 22/tcp | FTPS 989-990   SSH | TLS
HTTP    80/tcp          HTTPS 443/tcp                TLS
SMTP    25/tcp          SMTPS 465 | submission 587   TLS implícito | STARTTLS
POP3    110/tcp         POP3S 995/tcp                TLS
IMAP    143/tcp         IMAPS 993/tcp                TLS
LDAP    389/tcp         LDAPS 636/tcp                TLS
SNMP    161/udp v1/v2c  SNMPv3 161/udp               auth + cifrado en el propio protocolo
DNS     53/udp          DoT 853/tcp | DoH 443/tcp    TLS | HTTPS
Syslog  514/udp         syslog-TLS 6514/tcp          TLS
RTP     dinámico/udp    SRTP                         cifrado del flujo de voz
```

La columna de la derecha muestra el patrón: casi todos se aseguran metiendo el mismo protocolo dentro de TLS, que por eso es la pieza central del tema.

Analogía: es la misma carta, pero en vez de postal (cualquiera la lee por el camino) va en sobre sellado.

Límite: SNMPv3 y SSH no usan TLS, traen su propia criptografía; y no todo gemelo cambia de puerto (SNMPv3 sigue en 161, STARTTLS sube a cifrado en el mismo puerto).

```
Gemelo seguro → misma función, cifrada con TLS o SSH, casi siempre en otro puerto.
STARTTLS → sube a TLS dentro de la conexión ya abierta, mismo puerto.
TLS implícito → cifrado desde el primer byte, puerto dedicado.
```

## Common Ports and their Uses

Un **puerto** es un número de 16 bits (0 a 65535) que el transporte usa para entregar cada paquete al programa correcto dentro de un equipo. Existe porque una IP identifica a la máquina, pero en ella corren a la vez un servidor web, uno de correo y uno SSH; el puerto dice a cuál va el paquete.

Analogía: la IP es la dirección del edificio; el puerto es el número de apartamento. El 443 es "el apartamento del servidor web seguro" en casi todos los edificios del mundo.

Los rangos los define IANA (la autoridad que registra números en internet):

```
0     - 1023    well-known (conocidos): servicios estándar; en Linux abrirlos exige root.
1024  - 49151   registered (registrados): aplicaciones concretas (3306 MySQL, 3389 RDP).
49152 - 65535   dynamic / ephemeral (efímeros): los toma el cliente al azar para cada conexión.
```

Una conexión siempre tiene dos puertos: el del servidor es fijo y conocido, el del cliente es efímero. Tu navegador conecta desde `192.168.1.10:51234` hacia `93.184.216.34:443`. Linux usa por defecto el rango efímero 32768-60999 (se ve en `/proc/sys/net/ipv4/ip_local_port_range`), Windows 49152-65535.

Lista completa que piden Security+ y Network+, agrupada por función. Formato: `puerto/transporte  protocolo → uso`.

Transferencia de archivos y acceso remoto:

```
20/tcp        FTP (datos, modo activo) → canal de datos de FTP.
21/tcp        FTP (control) → comandos y credenciales de FTP, en claro.
22/tcp        SSH / SCP / SFTP → terminal remota y copia de archivos cifradas.
23/tcp        Telnet → terminal remota en claro.
69/udp        TFTP → copia de archivos sin autenticación (arranque por red, configs).
445/tcp       SMB (directo sobre TCP) → archivos e impresoras compartidos de Windows.
137/udp       NetBIOS Name Service → resolución de nombres NetBIOS.
138/udp       NetBIOS Datagram → mensajes NetBIOS sin conexión.
139/tcp       NetBIOS Session → SMB antiguo sobre NetBIOS.
989-990/tcp   FTPS → FTP sobre TLS implícito (989 datos, 990 control).
3389/tcp+udp  RDP → escritorio remoto de Windows.
5900/tcp      VNC → escritorio remoto multiplataforma.
```

Web:

```
80/tcp        HTTP → web en claro.
443/tcp       HTTPS → web sobre TLS.
443/udp       HTTP/3 (QUIC) → web sobre QUIC, TLS 1.3 integrado.
8080/tcp      HTTP alternativo → proxies, paneles de administración, servidores de desarrollo.
8443/tcp      HTTPS alternativo → paneles web de administración.
```

Correo:

```
25/tcp        SMTP → envío entre servidores de correo.
110/tcp       POP3 → descarga de correo, en claro.
143/tcp       IMAP → lectura de correo, en claro.
465/tcp       SMTPS → envío con TLS implícito.
587/tcp       SMTP submission → envío del cliente al servidor, con STARTTLS y login.
993/tcp       IMAPS → IMAP sobre TLS.
995/tcp       POP3S → POP3 sobre TLS.
```

Infraestructura de red:

```
53/udp+tcp    DNS → consultas por UDP; respuestas grandes y transferencias de zona por TCP.
67/udp        DHCP servidor → el servidor escucha aquí.
68/udp        DHCP cliente → el cliente escucha aquí.
123/udp       NTP → sincronización de hora.
161/udp       SNMP → consultas del gestor al agente.
162/udp       SNMP trap → alertas que el agente manda al gestor sin que se las pidan.
514/udp       Syslog → envío de logs.
6514/tcp      Syslog sobre TLS → logs cifrados.
853/tcp       DoT → DNS sobre TLS.
```

Autenticación y directorio:

```
49/tcp        TACACS+ → AAA para equipos de red (Cisco), cifra todo el paquete.
88/tcp+udp    Kerberos → tickets de autenticación del dominio Windows.
389/tcp+udp   LDAP → consultas al directorio, en claro.
636/tcp       LDAPS → LDAP sobre TLS.
3268/tcp      LDAP Global Catalog → búsquedas en todo el bosque de Active Directory.
1812/udp      RADIUS autenticación (antiguo 1645).
1813/udp      RADIUS accounting (antiguo 1646).
```

VPN:

```
500/udp       IKE / ISAKMP → negociación de claves de IPsec.
4500/udp      IPsec NAT-T → IPsec encapsulado en UDP para atravesar NAT.
1701/udp      L2TP → túnel de capa 2, normalmente con IPsec.
1723/tcp      PPTP → VPN antigua, insegura (MS-CHAPv2 roto).
1194/udp      OpenVPN → VPN basada en TLS.
51820/udp     WireGuard → VPN moderna.
```

Bases de datos y otros:

```
135/tcp       Microsoft RPC (endpoint mapper) → llamadas remotas de Windows.
119/tcp       NNTP → grupos de noticias (Usenet).
1433/tcp      Microsoft SQL Server.
1521/tcp      Oracle Database (listener).
3306/tcp      MySQL / MariaDB.
5432/tcp      PostgreSQL.
6379/tcp      Redis.
27017/tcp     MongoDB.
5060/udp+tcp  SIP → señalización de VoIP, en claro.
5061/tcp      SIP sobre TLS.
```

Cómo se lee en la práctica: en Linux, `ss` muestra qué puertos escucha tu equipo (la herramienta se ve en [`07-herramientas-de-red`](../07-herramientas-de-red/)):

```
$ ss -tuln
Netid State   Recv-Q Send-Q  Local Address:Port   Peer Address:Port
udp   UNCONN  0      0             0.0.0.0:68          0.0.0.0:*
tcp   LISTEN  0      128           0.0.0.0:22          0.0.0.0:*
tcp   LISTEN  0      511           0.0.0.0:80          0.0.0.0:*
tcp   LISTEN  0      80          127.0.0.1:3306        0.0.0.0:*
```

Lectura: SSH (22) y HTTP (80) aceptan conexiones desde cualquier interfaz (`0.0.0.0`); MySQL (3306) solo desde la propia máquina (`127.0.0.1`), que es lo correcto para una base de datos que no debe exponerse; 68/udp es el cliente DHCP. El archivo `/etc/services` trae la tabla puerto-nombre que usan estas herramientas.

> [!TIP]
> Para el examen, memoriza en pares: 21 FTP, 22 SSH, 23 Telnet, 25 SMTP, 53 DNS, 80/443 HTTP/HTTPS, 110/995 POP3, 143/993 IMAP, 389/636 LDAP, 445 SMB, 3389 RDP. Casi todo lo cifrado vive en el 400 o el 900.

```
Well-known → 0-1023, servicios estándar.
Registered → 1024-49151, aplicaciones concretas.
Ephemeral → 49152-65535, puerto del cliente, uno por conexión.
Puerto del servidor → fijo y conocido; puerto del cliente → efímero y al azar.
```

### Un puerto identifica un servicio, no lo garantiza

> [!IMPORTANT]
> El número de puerto es una convención, no un candado: cualquier programa puede escuchar en cualquier puerto, así que "puerto 443 abierto" no prueba que haya HTTPS, y bloquear un puerto no bloquea un protocolo.

La asignación de puertos de IANA es un registro de convenciones que dice qué servicio se espera en cada número, pero nada en TCP/IP obliga a cumplirlo. Un administrador puede mover SSH al 2222, y un malware puede sacar datos por el 443 porque casi ningún firewall bloquea la salida web.

```
                 Lo que dice el puerto    Lo que realmente corre
Servidor A       22/tcp → SSH             OpenSSH                    coincide
Servidor B       2222/tcp → (nada)        OpenSSH                    SSH movido
Servidor C       443/tcp → HTTPS          shell inverso de malware   no coincide
```

La última columna es la que importa: por eso nmap tiene `-sV` (pregunta al servicio quién es) y los firewalls modernos inspeccionan el contenido (DPI, deep packet inspection) en vez de fiarse del número.

Analogía: el número de apartamento dice dónde vive alguien, no quién abre la puerta. Mover SSH al 2222 reduce ruido de bots, pero no es seguridad: es solo cambiar el número del timbre.

```
Puerto → convención de IANA, no garantía.
nmap -sV → identifica el servicio real que contesta.
DPI → firewall que mira el contenido, no solo el número.
```

## SSL and TLS Basics

**TLS** (Transport Layer Security) es un protocolo criptográfico que crea, encima de una conexión TCP, un canal con tres garantías: confidencialidad (nadie más lee), integridad (nadie modifica sin que se note) y autenticación del servidor (hablas con quien crees). **SSL** (Secure Sockets Layer) es su predecesor, de Netscape; el nombre sobrevive por costumbre ("certificado SSL"), pero hoy todo lo que funciona es TLS.

Existe porque TCP entrega bytes en orden pero en claro: cualquier equipo en el camino (el router del café, el proveedor de internet) puede leerlos o cambiarlos. TLS resuelve eso sin modificar la aplicación: HTTP, SMTP o LDAP se meten dentro del túnel tal cual.

Analogía: es como un furgón blindado entre dos bancos. Antes de cargar el dinero, los guardias comprueban la credencial del banco destino (certificado), acuerdan una combinación de candado que solo ellos conocen (clave de sesión) y luego cada caja viaja sellada (cifrado + integridad).

Versiones y su estado:

```
SSL 2.0  (1995)  prohibido (RFC 6176).
SSL 3.0  (1996)  prohibido (RFC 7568); ataque POODLE.
TLS 1.0  (1999)  obsoleto (RFC 8996, 2021); BEAST.
TLS 1.1  (2006)  obsoleto (RFC 8996, 2021).
TLS 1.2  (2008)  vigente; RFC 5246.
TLS 1.3  (2018)  vigente y recomendado; RFC 8446.
```

Cómo funciona por dentro, en dos fases:

1. Handshake: se negocia la versión y la suite de cifrado, el servidor prueba su identidad con su certificado, y ambos derivan las mismas claves de sesión sin mandarlas por la red (intercambio Diffie-Hellman efímero: cada lado aporta un número secreto, intercambian un valor público derivado, y ambos llegan al mismo secreto que un espía no puede calcular).
2. Record protocol: todos los datos de aplicación se cortan en registros de hasta 16 KB, cada uno cifrado con cifrado simétrico (AES-GCM o ChaCha20-Poly1305) que además lleva una etiqueta de integridad.

Se usa asimétrico solo para autenticar y acordar claves, porque es lento; el volumen de datos va con simétrico, que es miles de veces más rápido. Los detalles matemáticos (RSA, ECDHE, AES, GCM) viven en [`10-criptografia`](../10-criptografia/).

Una **suite de cifrado** (cipher suite) es el paquete de algoritmos que se usará. En TLS 1.2 el nombre lo dice todo:

```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    │     │        │           │
    │     │        │           └ hash para derivar claves (SHA-256)
    │     │        └ cifrado simétrico de los datos (AES 128 en modo GCM)
    │     └ algoritmo de la firma del certificado (RSA)
    └ intercambio de claves (Diffie-Hellman efímero de curva elíptica)
```

TLS 1.3 sacó el intercambio y la firma del nombre (se negocian aparte) y dejó solo cinco suites: `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`, `TLS_AES_128_CCM_SHA256` y `TLS_AES_128_CCM_8_SHA256`.

### Handshake TLS 1.2 vs TLS 1.3

El handshake TLS es la negociación inicial que, en uno o dos viajes de ida y vuelta, deja a cliente y servidor con claves compartidas y con el servidor autenticado. Siempre ocurre después del handshake TCP (ver "Understand Handshakes").

TLS 1.2, dos RTT antes de mandar el primer dato:

```
Cliente                                              Servidor
   │── ClientHello ─────────────────────────────────────▶│  versión máx, random_c, suites que soporta, SNI
   │                                                     │
   │◀──────────────────────────────────── ServerHello ───│  suite elegida, random_s
   │◀──────────────────────────────────── Certificate ───│  cadena X.509 del servidor
   │◀────────────────────────────── ServerKeyExchange ───│  parte pública DH, firmada con su clave privada
   │◀─────────────────────────────── ServerHelloDone ────│
   │                                                     │   ── fin del RTT 1 ──
   │── ClientKeyExchange ───────────────────────────────▶│  parte pública DH del cliente
   │── ChangeCipherSpec ────────────────────────────────▶│  "desde ahora cifro"
   │── Finished (cifrado) ──────────────────────────────▶│  hash de todo el handshake
   │                                                     │
   │◀────────────────────────────── ChangeCipherSpec ────│
   │◀────────────────────────────── Finished (cifrado) ──│
   │                                                     │   ── fin del RTT 2 ──
   │══ datos de aplicación cifrados ═════════════════════│
```

TLS 1.3, un RTT; el cliente adivina el grupo DH y manda su parte pública de entrada:

```
Cliente                                              Servidor
   │── ClientHello + key_share ─────────────────────────▶│  suites, grupos (x25519...), parte pública DH
   │                                                     │
   │◀────────────────────────── ServerHello + key_share ─│  su parte pública DH → ambos ya tienen la clave
   │◀──────────────── {EncryptedExtensions} ─────────────│  ← desde aquí todo va cifrado
   │◀──────────────── {Certificate} ─────────────────────│
   │◀──────────────── {CertificateVerify} ───────────────│  firma sobre el handshake con su clave privada
   │◀──────────────── {Finished} ────────────────────────│
   │                                                     │   ── fin del RTT 1 ──
   │── {Finished} ──────────────────────────────────────▶│
   │══ datos de aplicación cifrados ═════════════════════│
```

Las diferencias que se preguntan:

- Velocidad: 1.2 necesita 2 RTT, 1.3 solo 1. Con 50 ms de distancia al servidor, 1.3 ahorra 100 ms en cada conexión nueva.
- 0-RTT: en una reconexión, TLS 1.3 deja mandar datos en el primer mensaje usando una clave previa (PSK, pre-shared key). Costo: esos datos se pueden reenviar (ataque de replay), así que solo sirve para peticiones idempotentes como un `GET`.
- Forward secrecy obligatorio: 1.3 elimina el intercambio de claves RSA estático, donde el cliente cifraba el secreto con la clave pública del servidor. Con RSA estático, quien robe la clave privada del servidor puede descifrar todo el tráfico grabado en el pasado; con DH efímero, cada sesión tiene claves desechables y robar la clave privada no sirve para lo ya grabado.
- Certificado cifrado: en 1.2 el certificado viaja en claro (un observador ve a qué sitio vas); en 1.3 va cifrado. El nombre del sitio todavía se filtra en el SNI (Server Name Indication) del ClientHello, salvo que se use ECH (Encrypted Client Hello).
- Limpieza: 1.3 elimina RC4, 3DES, modo CBC, MD5, SHA-1, compresión y renegociación, que eran el origen de ataques como BEAST, CRIME, Lucky13 y POODLE.

> [!WARNING]
> "Certificado SSL" es solo un nombre comercial: el certificado es X.509 y sirve igual para TLS 1.2 o 1.3. Si un servidor todavía acepta SSLv3 o TLS 1.0 es una vulnerabilidad aunque tenga un certificado perfecto.

### Certificados

Un **certificado X.509** es un documento firmado digitalmente por una autoridad de certificación (CA, certificate authority) que une una clave pública con una identidad, normalmente un nombre de dominio. Resuelve el problema del intermediario: sin él, un atacante en el camino podría mandarte su propia clave pública diciendo "soy tu banco".

Analogía: es el pasaporte del servidor. La CA es el gobierno que lo emite; tu navegador confía en el pasaporte porque confía en el gobierno, cuyo sello ya conoce.

Campos que importan:

- Subject / Subject Alternative Name (SAN): los nombres para los que vale (`www.ejemplo.com`, `ejemplo.com`). Los navegadores solo miran el SAN.
- Issuer: quién lo firmó.
- Validity (Not Before / Not After): fechas de validez. Desde marzo de 2026 el máximo para certificados públicos es 200 días; baja a 100 en 2027 y a 47 en 2029 (CA/Browser Forum). Let's Encrypt emite de 90 días.
- Public Key: la clave pública del servidor (RSA 2048 o más, o ECDSA P-256).
- Signature: la firma de la CA sobre todo lo anterior.

La cadena de confianza:

```
Root CA (autofirmada, preinstalada en el sistema/navegador, vive offline)
   │ firma
   ▼
Intermediate CA (la que firma en el día a día)
   │ firma
   ▼
Certificado hoja (leaf) de www.ejemplo.com  ← lo manda el servidor junto con la intermedia
```

Cómo valida el cliente: comprueba cada firma subiendo hasta una raíz de su almacén de confianza, que la fecha actual esté dentro de la validez, que el nombre pedido esté en el SAN y que no esté revocado (vía OCSP, consulta en línea, o CRL, lista de revocados). Si algo falla, el navegador muestra la pantalla roja.

Tipos según cuánto se verificó y cuánto cubre:

- DV (Domain Validated): la CA solo comprobó que controlas el dominio. Es lo que da Let's Encrypt, gratis y automático con ACME.
- OV (Organization Validated): además verificó que la organización existe.
- EV (Extended Validation): verificación legal exhaustiva. Los navegadores ya no lo muestran distinto.
- Wildcard (`*.ejemplo.com`): cubre un nivel de subdominios (`a.ejemplo.com`, no `x.a.ejemplo.com`).
- SAN / multidominio: varios nombres en un solo certificado.
- Autofirmado (self-signed): firmado por su propia clave, sin CA. Útil en un laboratorio; en producción el cliente no tiene cómo confiar en él.

Ejemplo: inspeccionar el handshake real de un sitio con `openssl`.

```
$ openssl s_client -connect ejemplo.com:443 -servername ejemplo.com -brief
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: CN = ejemplo.com
Hash used: SHA256
Signature type: ECDSA
Verification: OK
Server Temp Key: X25519, 253 bits
```

Lectura: se negoció TLS 1.3 con AES-256-GCM, el certificado es del dominio pedido y está firmado con ECDSA, la cadena se validó (`Verification: OK`) y el intercambio fue DH efímero sobre la curva X25519, así que hay forward secrecy. Para ver las fechas: `openssl s_client -connect ejemplo.com:443 </dev/null 2>/dev/null | openssl x509 -noout -dates -subject -issuer`.

```
SSL → predecesor, todas sus versiones prohibidas.
TLS 1.2 → vigente, 2 RTT, admite suites débiles si se configuran.
TLS 1.3 → 1 RTT, solo DH efímero, cinco suites AEAD, certificado cifrado.
Certificado → une clave pública con un nombre, firmado por una CA.
Root CA → raíz preinstalada; Intermediate → firma en el día a día; Leaf → el del servidor.
OCSP / CRL → consulta en línea / lista de certificados revocados.
Forward secrecy → robar la clave privada no descifra sesiones pasadas.
```

## Network Protocols

Los seis protocolos de esta parte son los que el roadmap destaca porque un analista de seguridad los ve todos los días: tres para administrar equipos o mover archivos (SSH, RDP, FTP/SFTP) y dos para la web (HTTP/HTTPS, SSL/TLS).

### SSH

**SSH** (Secure Shell) es un protocolo cliente-servidor que da una terminal remota cifrada y autenticada sobre TCP 22, y que además sirve para copiar archivos (SCP, SFTP) y crear túneles. Nació en 1995 para reemplazar a Telnet, rlogin y rsh, que mandaban la contraseña en claro.

Analogía: Telnet es gritarle órdenes a un empleado desde la calle; SSH es hablarle por un teléfono cifrado después de comprobar que el que contesta es él.

Cómo funciona por dentro (RFC 4251-4254), en tres capas:

1. Transport layer: intercambio de versiones (`SSH-2.0-OpenSSH_9.8`), negociación de algoritmos, intercambio de claves Diffie-Hellman y verificación de la host key (la clave que identifica al servidor). Desde aquí todo va cifrado.
2. User authentication: el usuario se autentica con contraseña o, mejor, con clave pública: el cliente firma un reto con su clave privada y el servidor lo comprueba contra `~/.ssh/authorized_keys`. La clave privada nunca sale del cliente.
3. Connection layer: sobre el mismo túnel se multiplexan canales: la shell, una copia SFTP, un port forwarding.

TOFU (trust on first use): la primera vez que conectas, el cliente muestra la huella de la host key y pregunta si confías. Si la aceptas, queda en `~/.ssh/known_hosts`; si un día cambia, SSH grita "REMOTE HOST IDENTIFICATION HAS CHANGED", que puede ser una reinstalación o un ataque de intermediario.

Ejemplo: generar una clave y conectar.

```
$ ssh-keygen -t ed25519 -C "jony@laptop"
Your identification has been saved in /home/jony/.ssh/id_ed25519
Your public key has been saved in /home/jony/.ssh/id_ed25519.pub
$ ssh-copy-id admin@192.168.1.50
$ ssh admin@192.168.1.50
The authenticity of host '192.168.1.50 (192.168.1.50)' can't be established.
ED25519 key fingerprint is SHA256:Vq3n0bJ2y1mX8kQ9r7tW4pL6sZ2cA5dE8fG1hJ3kM0o.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
admin@srv:~$
```

Túneles (port forwarding), un uso que también abusan los atacantes para saltarse firewalls:

```
ssh -L 8080:127.0.0.1:80 admin@srv     local: tu localhost:8080 llega al puerto 80 de srv
ssh -R 9000:127.0.0.1:22 admin@vps     remoto: el puerto 9000 del vps llega a tu 22
ssh -D 1080 admin@srv                  dinámico: proxy SOCKS que sale por srv
```

Hardening básico en `/etc/ssh/sshd_config`: `PasswordAuthentication no`, `PermitRootLogin no`, solo claves ed25519 o RSA de 3072 bits o más, y `fail2ban` contra fuerza bruta. Se amplía en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

```
Telnet → terminal remota en claro, 23/tcp.
SSH → terminal remota cifrada, 22/tcp.
Host key → identifica al servidor; known_hosts la recuerda.
Clave de usuario → identifica al cliente; authorized_keys la autoriza.
ssh -L / -R / -D → túnel local / remoto / proxy SOCKS.
```

### RDP

**RDP** (Remote Desktop Protocol) es un protocolo de Microsoft que transmite el escritorio gráfico de un equipo Windows a otro y devuelve teclado, ratón, portapapeles, audio e impresoras, sobre TCP 3389 (y UDP 3389 para mejorar el rendimiento). Existe porque administrar un servidor Windows suele requerir interfaz gráfica, y porque permite a los empleados usar su PC de oficina desde casa.

Analogía: es un monitor y un teclado con un cable de cientos de kilómetros; la pantalla no se copia como video entero, se mandan solo las partes que cambiaron, como quien describe por teléfono solo lo que se movió.

Cómo funciona: el cliente (`mstsc.exe` en Windows, `xfreerdp` o Remmina en Linux) conecta al 3389, negocia seguridad y, con NLA (Network Level Authentication), el usuario se autentica vía CredSSP antes de que el servidor cree la sesión gráfica. Después el canal va cifrado con TLS. Sin NLA, el servidor dibuja la pantalla de login a cualquiera que conecte, gastando recursos y exponiendo más código antes de la autenticación.

Por qué importa en seguridad: RDP expuesto a internet es una de las puertas de entrada favoritas del ransomware, por fuerza bruta de contraseñas o por fallos como BlueKeep (CVE-2019-0708), que permitía ejecutar código sin autenticarse en Windows 7 y Server 2008. Buscadores como Shodan listan millones de equipos con 3389 abierto.

Ejemplo: conectar desde Linux a un laboratorio propio.

```
$ xfreerdp /v:192.168.1.60 /u:admin /cert:ignore
[INFO][com.freerdp.core] - Connected to 192.168.1.60:3389 with security: NLA
```

> [!WARNING]
> Nunca se publica 3389 directo a internet: RDP va detrás de una VPN o de un RD Gateway (que lo envuelve en HTTPS 443), con NLA, MFA y bloqueo de cuenta tras varios intentos fallidos.

```
RDP → escritorio gráfico de Windows, 3389/tcp+udp.
NLA → autenticar antes de crear la sesión gráfica.
RD Gateway → publica RDP dentro de HTTPS 443.
VNC → escritorio remoto multiplataforma, 5900/tcp.
```

### FTP

**FTP** (File Transfer Protocol) es un protocolo de transferencia de archivos de 1971 (RFC 959 en su forma actual) que usa dos conexiones TCP separadas: una de control en el puerto 21 para comandos y otra de datos para el contenido. Todo, usuario y contraseña incluidos, viaja en claro.

Analogía: es pedir por un mostrador (control) y que te entreguen el paquete por una ventanilla aparte (datos). El problema es quién abre la ventanilla.

Modo activo vs pasivo, la pregunta clásica:

```
Activo:  cliente ──(21)──▶ servidor     "PORT 192,168,1,10,195,80"  → abre tu 50000
         cliente ◀──(20)── servidor     el SERVIDOR conecta hacia el cliente
         Problema: el firewall/NAT del cliente bloquea esa conexión entrante.

Pasivo:  cliente ──(21)──▶ servidor     "PASV" → "227 Entering Passive Mode (...,197,20)"
         cliente ──(50452)──▶ servidor  el CLIENTE conecta al puerto alto que le dieron
         Es el modo por defecto hoy porque atraviesa NAT.
```

En `PORT 192,168,1,10,195,80` el puerto se calcula 195 × 256 + 80 = 50000.

Ejemplo: lo que ve un sniffer en la misma red cuando alguien entra por FTP.

```
220 (vsFTPd 3.0.5)
USER admin
331 Please specify the password.
PASS Verano2026!
230 Login successful.
```

La contraseña aparece literal. Además, el FTP anónimo (`USER anonymous`) mal configurado expone archivos a cualquiera; es un hallazgo típico de las salas de TryHackMe.

```
FTP → archivos en claro, 21 control + 20 datos (activo).
Modo activo → el servidor conecta al cliente; choca con NAT.
Modo pasivo → el cliente conecta a un puerto alto del servidor; el habitual.
TFTP → versión mínima, UDP 69, sin autenticación.
```

### SFTP

**SFTP** (SSH File Transfer Protocol) es un subsistema de SSH que transfiere y administra archivos (listar, renombrar, borrar, cambiar permisos) dentro del túnel cifrado de SSH, por una única conexión en el puerto 22. No tiene nada que ver con FTP salvo el propósito: es otro protocolo, binario, diseñado desde cero.

Existe porque FTP manda credenciales en claro y su doble canal complica los firewalls. SFTP hereda la autenticación por clave pública de SSH y usa un solo puerto.

Analogía: FTP es mandar documentos en sobres abiertos por dos ventanillas; SFTP es meterlos en el mismo furgón blindado de SSH que ya usas para administrar el servidor.

Ejemplo:

```
$ sftp admin@192.168.1.50
Connected to 192.168.1.50.
sftp> ls
backup.tar.gz   informe.pdf
sftp> get informe.pdf
Fetching /home/admin/informe.pdf to informe.pdf
informe.pdf                       100%  240KB   8.1MB/s   00:00
sftp> bye
```

> [!WARNING]
> SFTP no es "FTP seguro con TLS": eso es FTPS (puertos 989/990, o 21 con `AUTH TLS`). SFTP es SSH en el puerto 22. Es la confusión de puertos más preguntada del examen.

```
SFTP → archivos sobre SSH, 22/tcp, un solo canal.
FTPS → FTP sobre TLS, 990/989 (implícito) o 21 con AUTH TLS (explícito).
SCP → copia simple sobre SSH, 22/tcp; sin listar ni renombrar.
FTP → archivos en claro, 20/21.
```

### HTTP / HTTPS

**HTTP** (HyperText Transfer Protocol) es un protocolo de petición-respuesta sin estado, sobre TCP 80, con el que un cliente pide recursos a un servidor web; **HTTPS** es exactamente el mismo HTTP viajando dentro de un túnel TLS, en el puerto 443. "Sin estado" significa que cada petición es independiente: el servidor no recuerda la anterior, y por eso existen las cookies, que el navegador devuelve en cada petición para que el servidor lo reconozca.

Analogía: HTTP es pedir en una ventanilla con un formulario: método (qué quieres hacer), ruta (sobre qué), encabezados (datos extra) y opcionalmente un cuerpo. La ventanilla responde con un código y el contenido, y olvida quién eras.

Una petición y su respuesta reales:

```
GET /login HTTP/1.1
Host: www.ejemplo.com
User-Agent: Mozilla/5.0
Cookie: session=4f9a2c

HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 1532
Strict-Transport-Security: max-age=31536000
Set-Cookie: session=4f9a2c; Secure; HttpOnly

<html>...
```

Métodos: `GET` (leer), `POST` (enviar datos/crear), `PUT` (reemplazar), `PATCH` (modificar parte), `DELETE` (borrar), `HEAD` (solo encabezados), `OPTIONS` (qué métodos admite).

Códigos de estado por centena:

```
1xx informativo    101 Switching Protocols (paso a WebSocket).
2xx éxito          200 OK, 201 Created, 204 No Content.
3xx redirección    301 Moved Permanently, 302 Found, 304 Not Modified.
4xx error cliente  400 Bad Request, 401 Unauthorized (falta autenticarse),
                   403 Forbidden (autenticado pero sin permiso), 404 Not Found,
                   429 Too Many Requests.
5xx error servidor 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable.
```

Versiones: HTTP/1.1 (texto, una petición a la vez por conexión), HTTP/2 (binario, muchas peticiones multiplexadas en una conexión TCP), HTTP/3 (sobre QUIC, que va por UDP 443 con TLS 1.3 integrado y evita que un paquete perdido frene a todos los demás).

Encabezados de seguridad que se ven en el examen: `Strict-Transport-Security` (HSTS: obliga al navegador a usar solo HTTPS durante `max-age` segundos, 31536000 = un año), `Content-Security-Policy`, y las banderas de cookie `Secure` (solo por HTTPS) y `HttpOnly` (JavaScript no puede leerla). Los ataques web (XSS, inyección, CSRF) se ven en [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/).

> [!NOTE]
> HTTPS protege el contenido y la ruta (`/login?user=...`), pero no oculta a qué IP te conectas ni, sin ECH, el nombre del sitio en el SNI; tampoco hace segura una web vulnerable, solo el camino hasta ella.

```
HTTP → petición-respuesta sin estado, 80/tcp, en claro.
HTTPS → HTTP dentro de TLS, 443/tcp.
HTTP/2 → binario y multiplexado sobre TCP.
HTTP/3 → sobre QUIC (UDP 443), TLS 1.3 integrado.
401 → no autenticado; 403 → autenticado sin permiso.
HSTS → obliga al navegador a usar solo HTTPS.
```

### SSL / TLS

SSL / TLS, como nodo de protocolos de red, es la capa de cifrado que se intercala entre el transporte (TCP) y la aplicación, y que convierte un protocolo en claro en su gemelo seguro sin cambiarlo. El funcionamiento completo (versiones, handshake 1.2 vs 1.3, suites, certificados) está en "SSL and TLS Basics", arriba; aquí solo lo que añade verlo como protocolo de red.

Dónde se sitúa:

```
Aplicación     HTTP     SMTP     IMAP     LDAP
               ───────────────────────────────
Seguridad              TLS (record protocol)
               ───────────────────────────────
Transporte                    TCP
Red                           IP
```

Analogía: es una funda universal: cualquier protocolo cabe dentro, y por fuera todos se ven iguales, bytes cifrados.

Dos formas de activarlo:

- TLS implícito: el puerto ya es TLS desde el primer byte (443, 465, 636, 993, 995).
- TLS explícito o STARTTLS: la conexión empieza en claro en el puerto normal (25, 587, 143, 389) y el cliente pide subir a TLS con un comando. Riesgo: un atacante en medio puede borrar la oferta de STARTTLS (ataque de downgrade/STRIPTLS) y el cliente sigue en claro si no exige cifrado.

Herramientas para auditar la configuración TLS de un servidor propio: `nmap --script ssl-enum-ciphers -p 443 host`, `testssl.sh` y SSL Labs (en "Recursos").

```
TLS implícito → cifrado desde el primer byte, puerto propio.
STARTTLS → empieza en claro y sube a TLS; vulnerable a downgrade si no se exige.
DTLS → TLS adaptado a UDP (VPN, WebRTC).
QUIC → transporte sobre UDP con TLS 1.3 integrado.
```

## Understand Handshakes

Un **handshake** es un intercambio inicial de mensajes con el que dos partes se ponen de acuerdo antes de comunicarse: confirman que el otro está ahí, sincronizan contadores, negocian parámetros o derivan claves. Existe porque mandar datos sin ese acuerdo previo sería como hablarle a alguien sin saber si escucha ni en qué idioma.

Analogía: es el "¿aló? / sí, dígame / habla Juan" del teléfono, antes de decir a qué llamas.

### TCP 3-way handshake

El **3-way handshake** es el intercambio de tres segmentos (SYN, SYN-ACK, ACK) con el que TCP abre una conexión y sincroniza los números de secuencia de ambos lados. El número de secuencia (seq) es el contador que TCP usa para numerar cada byte enviado; el ACK es "ya recibí hasta el byte N-1, mándame el N". Cada lado elige su seq inicial (ISN, initial sequence number) al azar, para que nadie pueda adivinarlo e inyectar paquetes falsos.

Banderas (flags) de TCP que aparecen: SYN (sincronizar, abrir), ACK (confirmo), FIN (terminé de mandar), RST (corta en seco), PSH (entrega ya a la aplicación), URG (urgente).

```
Cliente 192.168.1.10:51234                     Servidor 93.184.216.34:443
  CLOSED                                          LISTEN
    │── SYN      seq=1000 ─────────────────────────▶│
  SYN_SENT                                        SYN_RECEIVED
    │◀───────────────── SYN,ACK  seq=5000 ack=1001 ─│
    │── ACK      seq=1001 ack=5001 ────────────────▶│
  ESTABLISHED                                     ESTABLISHED
    │══ datos (aquí empieza el handshake TLS) ══════│
```

Lectura: el cliente propone su ISN 1000; el servidor confirma "espero el 1001" y propone el suyo, 5000; el cliente confirma "espero el 5001". El SYN consume un número de secuencia aunque no lleve datos, por eso el ACK es ISN+1.

Ejemplo: capturar tu propio handshake con tcpdump (la herramienta se ve en [`07-herramientas-de-red`](../07-herramientas-de-red/)).

```
$ sudo tcpdump -i eth0 -nn 'host 93.184.216.34 and tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'
10:00:01.100 IP 192.168.1.10.51234 > 93.184.216.34.443: Flags [S], seq 1000, win 64240, length 0
10:00:01.150 IP 93.184.216.34.443 > 192.168.1.10.51234: Flags [S.], seq 5000, ack 1001, win 65535, length 0
10:00:01.150 IP 192.168.1.10.51234 > 93.184.216.34.443: Flags [.], ack 5001, win 502, length 0
```

En la salida de tcpdump `[S]` es SYN, `[S.]` es SYN-ACK (el punto significa ACK) y `[.]` es un ACK solo. Los 50 ms entre el primer y el segundo paquete son un RTT.

Relevancia en seguridad: el SYN flood manda miles de SYN sin completar el tercer paso, llenando la cola de conexiones a medio abrir del servidor (ver [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/)); la defensa son las SYN cookies. Y el escaneo SYN de nmap (`-sS`) manda el SYN, mira si vuelve SYN-ACK (abierto) o RST (cerrado) y nunca completa el handshake.

UDP no tiene handshake: el primer datagrama ya lleva datos. Por eso es más rápido y por eso es más fácil falsificar la IP de origen en UDP.

```
SYN → pide abrir y propone ISN.
SYN-ACK → acepta, confirma ISN+1 y propone el suyo.
ACK → confirma; conexión ESTABLISHED.
RST → aborta la conexión de inmediato; también la respuesta de un puerto cerrado.
SYN cookies → defensa contra SYN flood, no guarda estado hasta el ACK.
```

### Cierre con FIN

El **cierre ordenado de TCP** es un intercambio de cuatro segmentos (FIN, ACK, FIN, ACK) en el que cada lado cierra por separado su sentido de la conexión. Son cuatro y no tres porque TCP es full-duplex (dos carriles independientes): que yo termine de hablar no significa que tú hayas terminado.

```
Cliente                                         Servidor
ESTABLISHED                                     ESTABLISHED
  │── FIN  seq=1500 ─────────────────────────────▶│
FIN_WAIT_1                                      CLOSE_WAIT   (aún puede mandar datos)
  │◀──────────────────────────────── ACK 1501 ────│
FIN_WAIT_2
  │◀──────────────────────────────── FIN seq=8000─│
                                                LAST_ACK
  │── ACK 8001 ──────────────────────────────────▶│
TIME_WAIT (espera 2×MSL)                        CLOSED
  ▼
CLOSED
```

TIME_WAIT: quien cierra primero espera dos veces el MSL (maximum segment lifetime, la vida máxima estimada de un paquete en la red; Linux usa un TIME_WAIT fijo de 60 segundos) para que un paquete rezagado de esta conexión no se confunda con una nueva que reutilice los mismos puertos. Muchas conexiones en TIME_WAIT en `ss` son normales en un servidor web con mucho tráfico; muchas en CLOSE_WAIT indican una aplicación que no está cerrando sus sockets.

Analogía: "ya no tengo más que decir" / "entendido" ... "yo tampoco" / "entendido, adiós". Un RST, en cambio, es colgar el teléfono a mitad de frase: no hay cuatro pasos ni TIME_WAIT. Los firewalls y los IPS usan RST para matar conexiones.

```
FIN → terminé de mandar en mi sentido.
CLOSE_WAIT → el otro cerró, yo aún no; muchos = fuga de sockets.
TIME_WAIT → espera de 2×MSL antes de liberar el par de puertos.
RST → cierre abrupto, sin negociación.
```

### TLS handshake

El TLS handshake es el segundo handshake de una conexión HTTPS: ocurre dentro de la conexión TCP ya establecida y negocia cifrado y claves. Su detalle completo, con diagramas de 1.2 y 1.3, está en "Handshake TLS 1.2 vs TLS 1.3", arriba.

### El handshake TCP ocurre antes que el TLS: son dos negociaciones distintas

> [!IMPORTANT]
> Una conexión HTTPS nueva hace primero el 3-way handshake de TCP (abre el canal) y después, encima, el handshake TLS (lo asegura); cada uno cuesta sus propios RTT y protege cosas diferentes.

La secuencia de apertura de HTTPS es una pila de handshakes que se suman: TCP no sabe nada de cifrado y TLS no sabe abrir conexiones.

```
                 TCP 3-way            TLS 1.2              TLS 1.3
Capa             transporte (4)       sesión/presentación  sesión/presentación
Mensajes         SYN, SYN-ACK, ACK    Hello, Cert, KeyEx,  Hello+key_share,
                                      CCS, Finished        {Cert, Verify, Finished}
RTT que cuesta   1                    2                    1
Qué logra        canal fiable         canal cifrado        canal cifrado
Total HTTPS      —                    1 + 2 = 3 RTT        1 + 1 = 2 RTT
```

La fila "Total HTTPS" es la idea: con un servidor a 50 ms (RTT de 100 ms), HTTPS sobre TLS 1.2 tarda 300 ms antes del primer byte útil y con TLS 1.3 tarda 200 ms. HTTP/3 sobre QUIC funde ambos en uno solo: 1 RTT total (0 si se reconecta).

Analogía: primero se marca y alguien contesta el teléfono (TCP); después los dos acuerdan hablar en clave (TLS). Ninguno de los dos pasos sustituye al otro.

Límite: un firewall que solo mira TCP ve un handshake normal hacia el 443 aunque dentro no haya TLS (ver "Un puerto identifica un servicio, no lo garantiza").

```
TCP handshake → abre el canal fiable, 1 RTT.
TLS 1.2 handshake → 2 RTT más; TLS 1.3 → 1 RTT más.
QUIC → transporte y TLS en un solo handshake.
```

### WPA 4-way handshake

El **4-way handshake de WPA2/WPA3** es el intercambio de cuatro mensajes EAPOL entre un punto de acceso Wi-Fi y un cliente que, partiendo de un secreto que ambos ya conocen, deriva claves de cifrado frescas para esa sesión y prueba que el otro conoce el secreto sin mandarlo por el aire. EAPOL (EAP over LAN) es el formato de trama que lo transporta.

Piezas: la PMK (pairwise master key) sale de la contraseña Wi-Fi en WPA2-Personal (o de 802.1X en Enterprise); cada lado aporta un número aleatorio (ANonce del AP, SNonce del cliente); de PMK + nonces + MACs sale la PTK (clave de sesión para el tráfico unicast), y el AP entrega la GTK (clave de grupo para broadcast).

```
AP (autenticador)                              Cliente (suplicante)
   │── 1. ANonce ──────────────────────────────────▶│  cliente ya puede calcular la PTK
   │◀────────────────────────── 2. SNonce + MIC ────│  AP calcula la PTK; el MIC prueba que el cliente sabe la PMK
   │── 3. GTK cifrada + MIC ───────────────────────▶│  AP prueba que también la sabe
   │◀────────────────────────────────── 4. ACK ─────│  ambos instalan las claves
```

Analogía: dos personas que comparten una contraseña se prueban mutuamente que la saben diciendo cada una un número al azar y mostrando el resultado de combinarlo con la contraseña, sin decir nunca la contraseña.

Relevancia: en WPA2-Personal quien capture los mensajes 1 y 2 puede probar contraseñas offline contra el MIC (diccionario), por eso una clave Wi-Fi corta es débil aunque el cifrado sea AES. WPA3 reemplaza la derivación por SAE (Dragonfly), que resiste ese ataque. KRACK (2017) abusaba de reinstalar la clave en el mensaje 3. WPA2/WPA3, SAE y el hardening Wi-Fi se ven en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

```
PMK → secreto maestro (de la contraseña Wi-Fi o de 802.1X).
ANonce / SNonce → números aleatorios del AP / cliente.
PTK → clave de sesión unicast derivada en el handshake.
GTK → clave de grupo para broadcast, la entrega el AP.
MIC → prueba de que se conoce la clave sin revelarla.
SAE (WPA3) → reemplaza la PSK de WPA2, resiste diccionario offline.
```

## Recursos para aprender y practicar

### Videos

- [Common Ports - CompTIA Network+ N10-009 - 1.4](https://www.youtube.com/watch?v=jX1pobYmZdE) — Professor Messer; la lista de puertos del examen con el protocolo de cada uno. Sirve para "Common Ports and their Uses".
- [Secure Protocols - CompTIA Security+ SY0-701 - 4.5](https://www.youtube.com/watch?v=9NAKCyOtFH0) — Professor Messer; pares en claro/seguro y selección de protocolo. Sirve para "Common Protocols and their Uses".
- [Network Ports Explained](https://www.youtube.com/watch?v=g2fT-g9PX9o) — PowerCert Animated Videos; introducción animada a qué es un puerto y los más comunes. Sirve para "Common Ports and their Uses".
- [TLS Handshake Explained - Computerphile](https://www.youtube.com/watch?v=86cQJ0MMses) — Computerphile (Dr Mike Pound); el handshake TLS paso a paso. Sirve para "SSL and TLS Basics".
- [SSL, TLS, HTTPS Explained](https://www.youtube.com/watch?v=j9QmMEWmcfo) — ByteByteGo; HTTPS, certificados y claves de sesión en pocos minutos. Sirve para "HTTP / HTTPS" y "SSL / TLS".
- [TCP connection walkthrough | Networking tutorial (13 of 13)](https://www.youtube.com/watch?v=F27PLin3TV0) — Ben Eater; una conexión TCP real leída en Wireshark, apertura y cierre. Sirve para "Understand Handshakes".
- [How TCP Works - The Handshake](https://www.youtube.com/watch?v=HCHFX5O1IaQ) — Chris Greer; el 3-way handshake y sus opciones en capturas reales. Sirve para "TCP 3-way handshake".
- [How TCP really works // Three-way handshake // TCP/IP Deep Dive](https://www.youtube.com/watch?v=rmFX1V49K8U) — David Bombal con Chris Greer; versión larga, números de secuencia, ventana, FIN y RST. Sirve para "Understand Handshakes".

### Lectura y documentación

- [IANA Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) — la tabla oficial de puertos.
- [RFC 9293 — Transmission Control Protocol](https://www.rfc-editor.org/rfc/rfc9293) — TCP actual: handshake, estados, FIN, RST, TIME_WAIT.
- [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446) y [RFC 5246 — TLS 1.2](https://www.rfc-editor.org/rfc/rfc5246) — las especificaciones de ambos handshakes.
- [RFC 8996 — Deprecating TLS 1.0 and TLS 1.1](https://www.rfc-editor.org/rfc/rfc8996) — por qué se retiraron.
- [The Illustrated TLS 1.3 Connection](https://tls13.xargs.org/) y [The Illustrated TLS 1.2 Connection](https://tls12.xargs.org/) — cada byte del handshake explicado.
- [What happens in a TLS handshake?](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/) — Cloudflare Learning; resumen claro de los pasos.
- [RFC 4253 — SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253) y [SSH protocol (ssh.com)](https://www.ssh.com/academy/ssh/protocol) — capas de SSH.
- [Understanding Remote Desktop Protocol](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/understanding-remote-desktop-protocol) — Microsoft Learn; arquitectura de RDP.
- [RFC 959 — File Transfer Protocol](https://www.rfc-editor.org/rfc/rfc959) — FTP, modos activo y pasivo.
- [An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) — MDN; peticiones, respuestas, versiones. Y [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) para métodos y códigos.
- [How It Works — Let's Encrypt](https://letsencrypt.org/how-it-works/) — emisión automática de certificados DV con ACME.
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) — configuraciones TLS recomendadas por servidor.

### Práctica

- [Network Services](https://tryhackme.com/room/networkservices) y [Network Services 2](https://tryhackme.com/room/networkservices2) — TryHackMe, gratis; SMB, Telnet, FTP, NFS, SMTP y MySQL en máquinas de laboratorio. Practica "Common Protocols and their Uses".
- [Networking Secure Protocols](https://tryhackme.com/room/networksecurityprotocols) — TryHackMe; TLS, SSH y VPN. Practica "SSL and TLS Basics" y "SSH".
- [Protocols and Servers](https://tryhackme.com/room/protocolsandservers) — TryHackMe; Telnet, HTTP, FTP, SMTP, POP3, IMAP conectándose a mano. Practica "Network Protocols".
- [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) — wargame que se juega entero por SSH; los primeros niveles enseñan `ssh`, puertos y `openssl s_client`. Practica "SSH" y "SSL / TLS".
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/) — analiza versión, suites y cadena de certificados de un sitio; prueba con tu propio dominio o uno público. Practica "SSL and TLS Basics".
- [badssl.com](https://badssl.com/) — subdominios con certificados rotos a propósito (expirado, autofirmado, nombre equivocado); ábrelos y lee el error del navegador. Practica "Certificados".
- Ejercicio en casa: captura tu propio handshake con `sudo tcpdump -i <interfaz> -nn -w hs.pcap 'tcp port 443'` mientras abres una web, y ábrelo en Wireshark: identifica SYN/SYN-ACK/ACK, el ClientHello (filtro `tls.handshake.type == 1`) con su SNI, la versión negociada en el ServerHello y los cuatro segmentos FIN/ACK del cierre. Practica "Understand Handshakes".
- Ejercicio en casa: ejecuta `ss -tuln` y `ss -tan state time-wait` en tu equipo y nombra cada puerto que escucha con ayuda de `/etc/services`. Practica "Common Ports and their Uses".

## Cuadro resumen

Todo lo visto, en una línea por término.

Common Protocols and their Uses

```
Protocolo de aplicación → idioma de capa 7 encima de TCP/UDP para un servicio concreto.
HTTP / HTTPS → web en claro / dentro de TLS.
SMTP → envía correo; POP3 → descarga y borra; IMAP → lee dejando en el servidor.
DNS → nombre a IP; UDP normal, TCP para respuestas grandes y transferencias de zona.
DHCP → asigna IP automáticamente (DORA).
NTP → sincroniza la hora; Kerberos falla con más de 5 min de diferencia.
Telnet → terminal en claro; SSH → terminal cifrada.
SMB → archivos de Windows; SMBv1 obsoleto (EternalBlue).
LDAP / LDAPS → directorio en claro / sobre TLS.
RADIUS → cifra solo la contraseña; TACACS+ → cifra todo.
SNMP v1/v2c → community string en claro; v3 → autenticado y cifrado.
Gemelo seguro → misma función cifrada con TLS o SSH, casi siempre en otro puerto.
STARTTLS → sube a TLS en el mismo puerto; TLS implícito → puerto dedicado.
```

Common Ports and their Uses

```
Rangos → 0-1023 well-known; 1024-49151 registered; 49152-65535 ephemeral.
20/21 FTP; 22 SSH/SFTP/SCP; 23 Telnet; 25 SMTP; 49 TACACS+; 53 DNS; 67/68 DHCP; 69 TFTP.
80 HTTP; 88 Kerberos; 110 POP3; 123 NTP; 135 RPC; 137-139 NetBIOS; 143 IMAP; 161/162 SNMP.
389 LDAP; 443 HTTPS; 445 SMB; 465 SMTPS; 500 IKE; 514 Syslog; 587 submission; 636 LDAPS.
853 DoT; 989/990 FTPS; 993 IMAPS; 995 POP3S; 1433 MSSQL; 1521 Oracle; 1701 L2TP; 1723 PPTP.
1812/1813 RADIUS; 3306 MySQL; 3389 RDP; 4500 IPsec NAT-T; 5060/5061 SIP; 5432 PostgreSQL; 5900 VNC.
Puerto → convención de IANA, no garantía; nmap -sV identifica el servicio real.
DPI → firewall que mira el contenido, no solo el número.
```

SSL and TLS Basics

```
TLS → canal con confidencialidad, integridad y autenticación del servidor sobre TCP.
SSL 2.0/3.0 → prohibidos; TLS 1.0/1.1 → obsoletos (RFC 8996); 1.2 y 1.3 vigentes.
Cipher suite → intercambio + firma + cifrado simétrico + hash; TLS 1.3 tiene solo cinco.
TLS 1.2 handshake → 2 RTT; admite RSA estático (sin forward secrecy).
TLS 1.3 handshake → 1 RTT, solo DH efímero, certificado cifrado, 0-RTT con riesgo de replay.
Forward secrecy → robar la clave privada no descifra sesiones pasadas.
SNI → nombre del sitio en claro en el ClientHello; ECH lo cifra.
Certificado X.509 → une clave pública con un nombre (SAN), firmado por una CA.
Cadena → Root (preinstalada) → Intermediate → Leaf.
Validez máxima pública → 200 días desde 2026, 100 en 2027, 47 en 2029.
OCSP / CRL → revocación en línea / por lista.
DV / OV / EV → dominio / organización / validación extendida; wildcard cubre un nivel.
```

Network Protocols

```
SSH → terminal cifrada, 22/tcp; host key en known_hosts, clave de usuario en authorized_keys.
ssh -L / -R / -D → túnel local / remoto / proxy SOCKS.
RDP → escritorio de Windows, 3389/tcp+udp; NLA, VPN o RD Gateway, nunca expuesto.
FTP → archivos en claro, 21 control + 20 datos; activo choca con NAT, pasivo es el habitual.
SFTP → archivos sobre SSH, 22/tcp.
FTPS → FTP sobre TLS, 989/990.
HTTP → petición-respuesta sin estado, 80/tcp; métodos GET, POST, PUT, PATCH, DELETE.
Códigos → 2xx éxito, 3xx redirección, 4xx error cliente (401 sin autenticar, 403 sin permiso), 5xx error servidor.
HTTP/2 → binario multiplexado; HTTP/3 → QUIC sobre UDP 443.
HSTS → obliga a usar solo HTTPS; cookies Secure y HttpOnly.
SSL / TLS como protocolo → capa entre transporte y aplicación; implícito o STARTTLS.
STARTTLS → vulnerable a downgrade si el cliente no exige cifrado.
```

Understand Handshakes

```
Handshake → intercambio inicial para sincronizar y negociar antes de comunicarse.
TCP 3-way → SYN, SYN-ACK, ACK; sincroniza ISN aleatorios; ACK = seq + 1.
Flags TCP → SYN abrir, ACK confirmar, FIN terminar, RST abortar, PSH entregar ya.
SYN flood → SYN sin completar; defensa SYN cookies.
Cierre FIN → cuatro segmentos porque cada sentido cierra aparte.
TIME_WAIT → 2×MSL (60 s en Linux); CLOSE_WAIT acumulado → aplicación que no cierra.
TLS handshake → segundo handshake, dentro de la conexión TCP ya abierta.
HTTPS nueva → TCP 1 RTT + TLS 1.2 2 RTT = 3; con TLS 1.3 = 2; QUIC = 1.
WPA 4-way → ANonce, SNonce+MIC, GTK+MIC, ACK; deriva PTK desde la PMK.
Captura del 4-way en WPA2-PSK → permite diccionario offline; WPA3 SAE lo resiste.
```
