# Certificaciones y CTFs

Datos de exámenes, versiones y precios consultados en los sitios oficiales el 2026-10-02. Los emisores cambian versiones cada 3 años aproximadamente y ajustan precios con frecuencia: antes de comprar un voucher, revisa la página oficial enlazada en cada sección.

## Conceptos previos

- Certificación: credencial que emite una organización cuando alguien demuestra, normalmente con un examen, que domina un conjunto de conocimientos definido. No es un título universitario ni una licencia legal para ejercer.
- Emisor (vendor): la organización que diseña el examen y otorga la certificación (CompTIA, Cisco, ISC2, ISACA, GIAC, OffSec, EC-Council, CREST).
- Vendor-neutral frente a vendor-specific: una certificación neutral (Security+, CISSP) no depende de la tecnología de un fabricante; una específica (CCNA) se centra en los productos de uno.
- Voucher: código prepagado que se canjea para agendar el examen; suele valer 12 meses.
- Examen supervisado (proctored): examen vigilado por una persona, en un centro de pruebas (Pearson VUE, PSI) o en remoto por cámara.
- PBQ (Performance-Based Question): pregunta práctica que pide hacer algo (configurar, arrastrar, analizar un log) en vez de elegir una opción.
- Puntaje escalado: el puntaje se convierte a una escala fija (por ejemplo 100-900) para que exámenes de distinta dificultad sean comparables; un 750/900 no es un 83 %.
- CAT (Computerized Adaptive Testing): examen adaptativo; cada pregunta se elige según cómo se respondió la anterior, y el examen termina cuando el sistema tiene suficiente confianza en el resultado.
- CPE (Continuing Professional Education): horas de formación que hay que acumular para mantener viva una certificación.
- ISO/IEC 17024 y ANAB: norma internacional para organismos que certifican personas, y el organismo de EE. UU. que acredita su cumplimiento. Una certificación acreditada tiene un proceso de examen auditado.
- DoD 8140: directiva del Departamento de Defensa de EE. UU. que exige ciertas certificaciones para puestos de ciberseguridad; por eso Security+, CISSP o CISM aparecen tanto en ofertas de empleo.
- Pentest (prueba de penetración): ataque autorizado y acotado contra sistemas de un cliente para encontrar vulnerabilidades antes que un atacante real. Detalle en [`19-programacion-y-hacking-practico`](../19-programacion-y-hacking-practico/).
- SOC (Security Operations Center): equipo que vigila alertas y responde a incidentes. Detalle en [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).
- GRC (Governance, Risk and Compliance): área que gestiona políticas, riesgos, auditorías y cumplimiento normativo. Detalle en [`17-estandares-y-cumplimiento`](../17-estandares-y-cumplimiento/).

## CTFs (Capture the Flag)

Un **CTF (Capture the Flag)** es una competición o ejercicio de seguridad en el que hay que resolver retos técnicos para encontrar una "bandera": una cadena de texto secreta con un formato conocido, como `flag{congr4tz_y0u_found_1t}` o `HTB{...}`, que demuestra que el reto se resolvió. Existe porque practicar ataques contra sistemas reales es ilegal sin autorización; un CTF ofrece objetivos vulnerables a propósito, legales y con una forma objetiva de saber si se acertó.

Analogía: un escape room. La sala está diseñada para que haya una salida, las pistas están puestas a propósito y el candado final confirma si se resolvió. Nadie entra a una casa ajena para practicar cómo abrir cerraduras.

### Jeopardy y Attack-Defense

Los formatos de CTF son las reglas de juego que definen si se resuelven retos aislados o se compite contra otros equipos en vivo. CTFtime, el sitio que lleva el calendario y la clasificación mundial de CTF, distingue tres:

- Jeopardy: un tablero de retos independientes agrupados por categorías, cada uno con un valor en puntos (más puntos cuanto más difícil). Gana el equipo con más puntos al acabar el tiempo. Categorías típicas: Web (vulnerabilidades en aplicaciones web), Crypto (romper cifrados mal usados), Pwn o Binary (explotar programas compilados, desbordamientos de memoria), Reversing (entender qué hace un ejecutable sin su código fuente), Forensics (analizar capturas, imágenes de disco, volcados de memoria), OSINT (encontrar información pública) y Misc.
- Attack-Defense: cada equipo recibe una máquina con los mismos servicios vulnerables. Hay que parchear los propios servicios (puntos de defensa, y los servicios tienen que seguir funcionando) mientras se explotan los de los rivales para robar sus banderas (puntos de ataque). Las rondas duran pocos minutos y las banderas cambian en cada ronda.
- Mixed: mezcla de los anteriores, por ejemplo un tablero Jeopardy más una fase de ataque y defensa. Una variante popular es King of the Hill: varios jugadores atacan la misma máquina y gana quien la controla más tiempo escribiendo su nombre en un archivo concreto.

```
Jeopardy                         Attack-Defense
+------+------+------+           Equipo A  <---- ataca ----  Equipo B
| Web  |Crypto| Pwn  |             | parchea                   | parchea
| 100  | 100  | 100  |             v                           v
| 200  | 200  | 300  |           servicios A               servicios B
| 500  | 400  | 500  |             ^                           |
+------+------+------+             +-------- ataca -------------+
retos aislados, sin rivales      rondas de minutos; robar y proteger banderas
directos                         a la vez
```

Además de las competiciones con fecha, existen plataformas permanentes de entrenamiento (HackTheBox, TryHackMe) con retos y máquinas siempre disponibles. No son CTF de competición en sentido estricto, pero usan el mismo mecanismo de banderas y son donde casi todo el mundo empieza.

Analogía: Jeopardy es un examen con preguntas de distinto valor; Attack-Defense es un partido de fútbol donde se defiende el propio arco y se ataca el rival al mismo tiempo.

Ejemplo: un CTF Jeopardy de 48 horas con 30 retos. Un equipo de 4 personas reparte por categorías: una persona Web, otra Crypto, otra Forensics, y la cuarta salta donde haya un reto de pocos puntos sin resolver. Resolver diez retos de 100 puntos vale lo mismo que uno de 1000, y suele ser más realista para un equipo que empieza.

> [!WARNING]
> Un CTF entrena técnica, no el trabajo completo. En un pentest real no hay bandera que confirme el éxito, el alcance está limitado por contrato y la mitad del valor está en el informe. Hay que practicar también la documentación de cada reto resuelto (writeup), no solo encontrar la bandera.

```
CTF → retos legales con una bandera que prueba la solución.
Jeopardy → tablero de retos por categorías y puntos; sin rivales directos.
Attack-Defense → defender servicios propios y atacar los ajenos a la vez, por rondas.
Mixed / King of the Hill → combinación; controlar una máquina compartida el mayor tiempo.
CTFtime → calendario y ranking mundial de CTF.
Writeup → explicación escrita de cómo se resolvió un reto.
```

### HackTheBox

**HackTheBox** (HTB) es una plataforma de entrenamiento en ciberseguridad ofensiva y defensiva que ofrece máquinas vulnerables y retos para atacar en un entorno legal, más una academia de cursos guiados. Existe para cerrar la brecha entre la teoría y la práctica: el usuario se conecta por VPN (o desde Pwnbox, una máquina de ataque en el navegador) a una red donde están las máquinas objetivo.

Qué ofrece:

- Machines: sistemas completos (Linux, Windows, Active Directory) que hay que comprometer de principio a fin. Cada una tiene dos banderas: `user.txt` (acceso inicial como usuario) y `root.txt` (escalada a administrador o root). Se publica una máquina nueva por semana gratis; el catálogo completo de máquinas retiradas requiere suscripción VIP.
- Challenges: retos sueltos de tipo Jeopardy (web, crypto, reversing, forensics, pwn, entre otras categorías), con uno nuevo gratis cada semana.
- Starting Point: serie guiada para principiantes en tres niveles. Tier 0 enseña lo básico (conectarse por VPN, enumerar puertos, interactuar con un servicio) con una herramienta por máquina. Tier 1 introduce una explotación sencilla por máquina. Tier 2 encadena enumeración, acceso inicial y escalada de privilegios con las dos banderas. Cada tier tiene máquinas gratis y otras de pago, y se puede avanzar sin hacer las de pago.
- Pro Labs: redes corporativas simuladas de muchas máquinas, para practicar movimiento lateral.
- HTB Academy: cursos por módulos con teoría y ejercicios, agrupados en rutas por rol (job-role paths). Algunas rutas terminan en una certificación práctica propia: CJCA (junior cybersecurity associate), CPTS (penetration testing specialist), CWES (web exploitation specialist), CDSA (defensive security analyst, orientada a SOC), y las de nivel experto CWEE (web), CAPE (Active Directory) y otras más recientes.
- HTB CTF: plataforma de competiciones con fecha, para equipos y empresas.

Analogía: un gimnasio de escalada con rutas marcadas por color. Starting Point son las rutas verdes con el instructor al lado; las máquinas Insane son las rutas negras que pocos completan.

Ejemplo de ruta en HTB para alguien que empieza: Starting Point Tier 0 y 1 (2 a 4 semanas), módulos de Academy de fundamentos de Linux, redes y "Getting Started", luego máquinas retiradas de dificultad Easy siguiendo después el video de IppSec de cada una, y finalmente máquinas activas sin ayuda. Una meta medible: 20 máquinas Easy y 10 Medium antes de intentar una certificación práctica.

```
HTB Machines → sistemas completos; banderas user.txt y root.txt.
HTB Challenges → retos sueltos tipo Jeopardy.
Starting Point → ruta guiada, Tier 0 / 1 / 2.
Pro Labs → redes corporativas simuladas.
HTB Academy → cursos por módulos; certificaciones CPTS, CDSA, CWES, CWEE, CAPE.
Pwnbox → máquina de ataque en el navegador.
```

### TryHackMe

**TryHackMe** (THM) es una plataforma de aprendizaje de ciberseguridad basada en salas (rooms): lecciones cortas que mezclan explicación, preguntas y una máquina para practicar en el navegador. Existe para el escalón anterior a HackTheBox: enseña desde cero, paso a paso, y cada tarea dice qué hacer y qué contestar, en lugar de soltar al usuario frente a una máquina sin pistas.

Cómo se organiza:

- Rooms: una sala trata un tema (una herramienta, una vulnerabilidad, un concepto). Tienen tareas con preguntas cuya respuesta se escribe en la web (un nombre de archivo, un puerto, una bandera).
- AttackBox: máquina de ataque en el navegador, con las herramientas instaladas; evita configurar la VPN al principio.
- Learning paths (rutas): secuencias de salas que llevan a un rol. Rutas de referencia, en orden de dificultad: Pre Security (redes, web y Linux desde cero), Cyber Security 101 (panorama de ataque y defensa), Jr Penetration Tester (metodología y explotación básica) por el lado ofensivo, y SOC Level 1 (análisis de alertas, SIEM, forense básico) por el lado defensivo.
- Retos tipo CTF y salas King of the Hill, y eventos estacionales como Advent of Cyber (una sala por día en diciembre, gratuita).
- Parte del contenido es gratis; la suscripción desbloquea todas las salas y rutas.

Analogía: TryHackMe es la escuela de manejo con instructor y pedales dobles; HackTheBox es la autopista. Se puede ir directo a la autopista, pero la mayoría aprende antes en la escuela.

Ejemplo de plan de 3 meses con THM: mes 1, ruta Pre Security completa; mes 2, Cyber Security 101; mes 3, elegir Jr Penetration Tester o SOC Level 1 según el rol buscado. Con 1 hora diaria, cada ruta de nivel inicial se completa en unas 4 a 8 semanas.

```
TryHackMe → salas guiadas con preguntas; para empezar de cero.
AttackBox → máquina de ataque en el navegador.
Rutas → Pre Security → Cyber Security 101 → Jr Penetration Tester / SOC Level 1.
THM → guiado, paso a paso; HTB → más abierto, menos pistas.
```

### VulnHub

**VulnHub** es un repositorio gratuito de máquinas virtuales vulnerables a propósito, que se descargan y se ejecutan en tu propio laboratorio para practicar pentesting sin conexión a ninguna plataforma. Existe para practicar sin suscripción ni VPN: bajas la imagen, la importas en VirtualBox o VMware y la atacas desde otra VM de tu red aislada.

Cómo se usa:

- Cada máquina (entry) trae una descripción, su dificultad y a veces una pista; el objetivo suele ser conseguir root y leer una bandera.
- Se importa en una red host-only o interna, nunca en modo bridged, para que una máquina deliberadamente vulnerable no quede expuesta a tu red de casa (ver [`06-virtualizacion`](../06-virtualizacion/)).
- Las máquinas son antiguas en su mayoría (el sitio se actualiza poco), lo que las hace buenas para aprender metodología clásica: enumerar, encontrar un servicio viejo, entrar, escalar privilegios.
- Clásicas para empezar: la serie Kioptrix y Mr-Robot.

Analogía: es comprar un kit de cerraduras de práctica para tu casa en vez de pagar la cuota de un gimnasio de cerrajería: lo tienes para siempre y entrenas a tu ritmo, pero tú montas el banco de trabajo.

Ejemplo: descargas Kioptrix Level 1 (unos 200 MB), la importas en VirtualBox en una red host-only junto a tu VM de ataque, la descubres en la red con un escaneo de tu propio rango y practicas el ciclo completo de un pentest documentando cada paso como si fuera un informe.

```
VulnHub → VMs vulnerables descargables; gratis, sin conexión; tú montas el laboratorio.
Red host-only → aísla la VM vulnerable de tu red real.
```

### picoCTF

**picoCTF** es una competencia gratuita de CTF tipo Jeopardy creada por la Universidad Carnegie Mellon para estudiantes, cuyo archivo de retos queda abierto todo el año en picoGym. Existe para enseñar desde cero: los primeros retos se resuelven con un navegador y una terminal básica, y la dificultad sube poco a poco.

Cómo se organiza:

- picoGym: cientos de retos de ediciones anteriores, siempre disponibles, por categoría: Web Exploitation, Cryptography, Reverse Engineering, Forensics, Binary Exploitation y General Skills.
- Cada reto da puntos según su dificultad y tiene pistas opcionales.
- La competencia anual (en primavera) es por equipos y está pensada para estudiantes de secundaria y universidad, aunque cualquiera puede practicar.
- Incluye una terminal web (webshell), así que no hace falta instalar nada al principio.

Analogía: es el cuaderno de ejercicios graduado de una materia: empiezas por los de una estrella y no pasas a los de cinco hasta resolver los anteriores.

Ejemplo de uso: una semana dedicada a General Skills (comandos de Linux, codificaciones como base64, permisos), la siguiente a Cryptography (cifrados clásicos, XOR) y la siguiente a Forensics (metadatos, archivos ocultos). Cada categoría refuerza un tema de esta guía.

```
picoCTF → CTF Jeopardy gratuito de Carnegie Mellon; picoGym abierto todo el año.
Categorías → Web, Crypto, Reversing, Forensics, Binary, General Skills.
```

### pwn.college

**pwn.college** es una plataforma educativa gratuita de la Universidad Estatal de Arizona que enseña seguridad de sistemas a bajo nivel mediante "dojos": módulos con videos de clase y cientos de retos prácticos que se resuelven en un entorno Linux en el navegador. Existe para cubrir lo que las plataformas de iniciación tocan poco: cómo funcionan de verdad los programas, la memoria y el sistema operativo.

Cómo se organiza:

- Dojos por nivel: los iniciales (Linux Luminarium, Computing 101, Playing With Programs) enseñan terminal, programas y ensamblador básico; los avanzados cubren corrupción de memoria, explotación de binarios, kernel e ingeniería inversa.
- Cada módulo trae las clases en video del curso universitario real y sus retos, que se resuelven en un contenedor accesible desde el navegador o por SSH.
- Se cursa con la misma secuencia que siguen los estudiantes de la universidad, y es gratis.

Analogía: si picoCTF es el cuaderno de ejercicios, pwn.college es el curso universitario completo, con clases grabadas y laboratorio incluidos.

Ejemplo: después de leer sobre buffer overflow en [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/) y sobre C++ y ensamblador en [`19-programacion-y-hacking-practico`](../19-programacion-y-hacking-practico/), el dojo de corrupción de memoria te hace ver, reto a reto, cómo una copia sin control de longitud llega a cambiar lo que hace un programa.

```
pwn.college → curso universitario gratuito de seguridad de sistemas en dojos.
Dojo → módulo con clases en video y retos en un entorno Linux en el navegador.
VulnHub / picoCTF / pwn.college → laboratorio propio / ejercicios graduados / curso de bajo nivel.
```


## Beginner Certifications

Las **certificaciones de entrada** son credenciales pensadas para validar la base técnica (hardware, sistemas, redes y conceptos de seguridad) de quien busca su primer empleo en TI o seguridad. Existen porque los filtros de selección de personal usan certificaciones para decidir a quién entrevistar cuando el candidato no tiene experiencia; demuestran que se estudió un temario completo y estándar.

Analogía: el carné de conducir. No demuestra que alguien sea buen piloto, pero sin él no se puede ni empezar a trabajar de conductor.

### CompTIA A+

**CompTIA A+** es una certificación de entrada de CompTIA (organización sin ánimo de lucro de la industria TI, con certificaciones neutrales de proveedor) que valida los conocimientos de soporte técnico: hardware, sistemas operativos, redes básicas, seguridad básica y resolución de problemas.

- Emisor: CompTIA.
- Enfoque: soporte técnico de primer nivel (help desk): montar y reparar equipos, instalar sistemas operativos, resolver problemas de usuario.
- Nivel: entrada. Es la primera de la serie CompTIA.
- Requisitos: ninguno obligatorio. CompTIA recomienda 12 meses de experiencia práctica en un rol de soporte TI.
- Versión vigente verificada: V15, lanzada el 25 de marzo de 2025, con retiro estimado en 2028. Requiere aprobar dos exámenes de la misma versión, sin mezclar versiones: Core 1 (220-1201) y Core 2 (220-1202).
- Formato: cada examen tiene un máximo de 90 preguntas en 90 minutos, de opción múltiple (respuesta simple y múltiple), arrastrar y soltar, y PBQ. Aprobado: 675/900 en Core 1 y 700/900 en Core 2.
- Dominios Core 1: Mobile devices 13 %, Networking 23 %, Hardware 25 %, Virtualization and cloud computing 11 %, Hardware and network troubleshooting 28 %.
- Dominios Core 2: Operating systems 28 %, Security 28 %, Software troubleshooting 23 %, Operational procedures 21 %.
- Rol al que apunta: Help Desk Support Specialist, técnico de soporte, técnico de campo.

Analogía: el curso de mecánico general antes de especializarse en motores.

Ejemplo de PBQ típico: se muestra un equipo que no arranca con una lista de síntomas (pitidos, luz de disco apagada) y hay que ordenar los pasos de la metodología de resolución de problemas de CompTIA: identificar el problema, establecer una teoría, probarla, planificar la solución, verificar el funcionamiento y documentar.

> [!TIP]
> Si ya trabajas en TI o sabes armar y reparar equipos, A+ es la certificación más prescindible de la ruta: muchos saltan directo a Network+ o Security+. Tiene valor real cuando se busca un primer empleo de soporte.

```
CompTIA A+ → soporte técnico; V15, Core 1 (220-1201) + Core 2 (220-1202).
Formato A+ → 90 preguntas máx., 90 min por examen; 675 y 700 sobre 900.
```

### CompTIA Linux+

**CompTIA Linux+** es una certificación de CompTIA que valida la administración de sistemas Linux: gestión del sistema, servicios y usuarios, seguridad, automatización y resolución de problemas, en entornos locales, de nube e híbridos.

- Emisor: CompTIA.
- Enfoque: administrar servidores Linux de cualquier distribución (no se ata a una, a diferencia de las de Red Hat).
- Nivel: entrada a intermedio.
- Requisitos: ninguno obligatorio. Recomendado: 12 meses de experiencia práctica con servidores Linux y conocimientos equivalentes a A+, Network+ o Server+.
- Versión vigente verificada: V8, examen XK0-006, lanzada el 15 de julio de 2025.
- Formato: máximo 90 preguntas en 90 minutos, opción múltiple y PBQ. Aprobado: 720/900.
- Dominios: System Management 23 %, Services and User Management 20 %, Security 18 %, Automation, Orchestration, and Scripting 17 %, Troubleshooting 22 %.
- Rol al que apunta: administrador de sistemas Linux junior, ingeniero de soporte de nube, base para DevOps. En seguridad, saber Linux es obligatorio tanto en ataque (casi todo el tooling corre en Linux) como en defensa (la mayoría de servidores son Linux). Detalle en [`02-linux`](../02-linux/).

Analogía: aprender a manejar la cocina profesional donde trabajan casi todos los chefs, no solo una receta.

Ejemplo de tarea que entra en el examen: un servicio web no arranca tras un reinicio. Hay que diagnosticar con `systemctl status nginx` y `journalctl -u nginx`, encontrar que el puerto 80 ya está ocupado (`ss -tlnp`), corregir la configuración y habilitar el servicio para el arranque con `systemctl enable --now nginx`.

```
CompTIA Linux+ → administración Linux neutral de distribución; V8, XK0-006.
Formato Linux+ → 90 preguntas máx., 90 min; 720/900.
```

### CompTIA Network+

**CompTIA Network+** es una certificación de CompTIA que valida los fundamentos de redes: conceptos, implementación, operación, seguridad y resolución de problemas de redes cableadas e inalámbricas, sin atarse a un fabricante.

- Emisor: CompTIA.
- Enfoque: entender y operar redes: modelo OSI, direccionamiento IP, subredes, switching, routing, Wi-Fi, servicios (DNS, DHCP), monitorización y troubleshooting.
- Nivel: entrada.
- Requisitos: ninguno obligatorio. Recomendado: A+ y de 9 a 12 meses de experiencia como administrador de red junior o técnico de soporte de red.
- Versión vigente verificada: V9, examen N10-009, lanzada el 20 de junio de 2024, retiro estimado en 2027. CompTIA no anuncia todavía una V10.
- Formato: máximo 90 preguntas en 90 minutos, opción múltiple y PBQ. Aprobado: 720/900.
- Dominios: Networking concepts 23 %, Network implementation 20 %, Network operations 19 %, Network security 14 %, Network troubleshooting 24 %.
- Rol al que apunta: técnico de redes, NOC (centro de operaciones de red), administrador de red junior. Para seguridad es la base: no se puede defender ni atacar una red que no se entiende. Detalle en [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/) y [`04-direccionamiento-ip`](../04-direccionamiento-ip/).

Analogía: conocer el sistema de carreteras de un país (tipos de vía, señales, rotondas) antes de trabajar en control de tráfico.

Ejemplo de pregunta típica: un host con `192.168.10.70/26` no llega a `192.168.10.130`. Hay que ver que un /26 tiene bloques de 64 direcciones, así que el host está en la subred `.64-.127` y el destino en `.128-.191`: necesita pasar por el router, y el fallo está en la puerta de enlace.

```
CompTIA Network+ → fundamentos de redes neutrales; V9, N10-009.
Formato Network+ → 90 preguntas máx., 90 min; 720/900.
```

### CCNA

**CCNA** (Cisco Certified Network Associate) es una certificación de Cisco que valida la capacidad de instalar, configurar, operar y diagnosticar redes empresariales, con práctica real sobre equipos e IOS de Cisco (el sistema operativo de sus routers y switches).

- Emisor: Cisco.
- Enfoque: redes con más profundidad práctica que Network+: configurar VLAN, trunks, STP, OSPF, ACL, NAT, DHCP y Wi-Fi en la línea de comandos de Cisco, más fundamentos de automatización.
- Nivel: entrada a intermedio (associate). Es más difícil que Network+.
- Requisitos: ninguno formal. Cisco sugiere al menos un año de experiencia implementando soluciones Cisco.
- Versión vigente verificada: examen 200-301 CCNA v1.1. Cisco anunció en mayo de 2026 la v2.0, con el mismo código 200-301, más peso en troubleshooting, nube, enfoque de seguridad y un apartado sobre IA en la operación de redes; según los anuncios difundidos, entra en vigor el 3 de febrero de 2027. Quien empiece a estudiar ahora y vaya a examinarse después de esa fecha debe usar el temario v2.0.
- Formato: 120 minutos, preguntas de opción múltiple, arrastrar y soltar y simulaciones de configuración. Precio: 300 USD. Validez: 3 años; se renueva aprobando el examen otra vez o con 30 créditos de educación continua de Cisco.
- Dominios (v1.1): Network Fundamentals 20 %, Network Access 20 %, IP Connectivity 25 %, IP Services 10 %, Security Fundamentals 15 %, Automation and Programmability 10 %.
- Rol al que apunta: ingeniero de redes junior, administrador de redes. En seguridad, es la base para roles de seguridad de redes y para CCNP Security.

Analogía: Network+ es saber las reglas de tráfico; CCNA es además saber manejar un modelo concreto de camión, porque es el que hay en la mayoría de las empresas.

Ejemplo de tarea en simulación: configurar una ACL estándar que impida a la red `10.1.1.0/24` llegar al servidor `10.2.2.10` y aplicarla en la interfaz correcta.

```
R1(config)# access-list 10 deny host 10.1.1.0 0.0.0.255
R1(config)# access-list 10 permit any
R1(config)# interface g0/1
R1(config-if)# ip access-group 10 out
```

Hay una trampa a propósito en la primera línea: `host` solo se usa con una única dirección; para la red completa la forma correcta es `access-list 10 deny 10.1.1.0 0.0.0.255`. Esos detalles de sintaxis son los que el CCNA evalúa y Network+ no.

```
CCNA → redes Cisco, práctica en IOS; 200-301 v1.1 (v2.0 desde el 3 feb 2027).
Formato CCNA → 120 min, simulaciones incluidas; 300 USD; validez 3 años.
Network+ → teoría neutral; CCNA → más profundidad y configuración real.
```

### CompTIA Security+

**CompTIA Security+** es una certificación de CompTIA que valida los conocimientos base de ciberseguridad: conceptos, amenazas y vulnerabilidades, arquitectura segura, operaciones de seguridad y gestión del programa de seguridad. Es la certificación de entrada en seguridad más pedida en ofertas de empleo, en parte porque cumple la directiva DoD 8140 de EE. UU.

- Emisor: CompTIA.
- Enfoque: amplitud en seguridad: criptografía, identidad y acceso, ataques comunes, controles, respuesta a incidentes, riesgo y cumplimiento.
- Nivel: entrada en seguridad (intermedio en la serie CompTIA).
- Requisitos: ninguno obligatorio. Recomendado: Network+ y dos años de experiencia como administrador de seguridad o de sistemas.
- Versión vigente verificada: V7, examen SY0-701, lanzada el 7 de noviembre de 2023. Su retiro está fijado para el 11 de junio de 2027 en inglés (13 de agosto de 2027 en japonés, portugués, español y tailandés).
- Versión siguiente: V8, examen SY0-801, con lanzamiento el 17 de noviembre de 2026, según la página oficial. Dominios de la V8: General Security Concepts 16 %, Threats, Vulnerabilities, and Attacks 24 %, Security Architecture 19 %, Security Operations 27 %, Security Program Management and Oversight 14 %. Añade más cobertura de riesgos relacionados con IA.
- Formato (V7 y V8): máximo 90 preguntas en 90 minutos, opción múltiple y PBQ. Aprobado: 750/900.
- Dominios V7: General security concepts 12 %, Threats, vulnerabilities, and mitigations 22 %, Security architecture 18 %, Security operations 28 %, Security program management and oversight 20 %.
- Rol al que apunta: analista SOC junior, administrador de seguridad, administrador de sistemas con responsabilidades de seguridad, soporte de seguridad.

Analogía: el examen teórico general de primeros auxilios: cubre de todo un poco y es lo mínimo que se pide para trabajar en un puesto de socorro.

Ejemplo de PBQ: se muestran los logs de un firewall y un servidor web con cinco eventos, y hay que clasificar cada uno (escaneo de puertos, inyección SQL, fuerza bruta, tráfico legítimo, exfiltración) y arrastrar el control que lo mitiga (WAF, bloqueo de cuenta, regla de firewall, DLP).

> [!NOTE]
> Entre noviembre de 2026 y junio de 2027 conviven SY0-701 y SY0-801. Quien ya estudió con material de la 701 puede presentarla hasta su retiro; quien empieza de cero después de noviembre de 2026 debería ir directo a la 801.

```
CompTIA Security+ → base de ciberseguridad; DoD 8140.
V7 SY0-701 → vigente; retiro 11 jun 2027 (inglés).
V8 SY0-801 → desde 17 nov 2026; más peso en amenazas e IA.
Formato Security+ → 90 preguntas máx., 90 min; 750/900.
```

## Advanced Certifications

Las **certificaciones avanzadas** son credenciales que validan especialización (pentesting, auditoría, gestión, defensa) o experiencia profesional acumulada, y que normalmente exigen años de trabajo o un examen práctico exigente. Existen porque, una vez superado el filtro de entrada, lo que distingue a un profesional es demostrar que sabe hacer algo concreto o que tiene la trayectoria para dirigir.

Se dividen en dos familias muy distintas:

- Técnicas y prácticas: demuestran habilidad con las manos, con exámenes de laboratorio (OSCP, CREST CRT, GIAC con CyberLive) o con preguntas técnicas (CEH, GSEC).
- De gestión y auditoría: demuestran criterio y experiencia para dirigir programas de seguridad o auditarlos (CISSP, CISM, CISA). Exigen 5 años de experiencia verificada.

### CEH

**CEH** (Certified Ethical Hacker) es una certificación de EC-Council que valida el conocimiento de las fases, técnicas y herramientas de hacking ético, desde el reconocimiento hasta la explotación, en un examen de opción múltiple.

- Emisor: EC-Council (International Council of E-Commerce Consultants).
- Enfoque: amplitud en técnicas ofensivas. La versión actual se comercializa como CEH AI (v13) e incorpora uso de IA en el temario. Los 20 módulos: Introduction to Ethical Hacking; Footprinting and Reconnaissance; Scanning Networks; Enumeration; Vulnerability Analysis; System Hacking; Malware Threats; Sniffing; Social Engineering; Denial-of-Service; Session Hijacking; Evading IDS, Firewalls, and Honeypots; Hacking Web Servers; Hacking Web Applications; SQL Injection; Hacking Wireless Networks; Hacking Mobile Platforms; IoT and OT Hacking; Cloud Computing; Cryptography.
- Nivel: intermedio (aunque su contenido es más amplio que profundo).
- Requisitos: hacer la formación oficial de EC-Council (la tasa de solicitud de 100 USD va incluida), o demostrar 2 años de experiencia en seguridad de la información y pagar una tasa de solicitud no reembolsable de 100 USD. La solicitud se resuelve en 5 a 10 días hábiles y vale 3 meses.
- Formato: examen 312-50, 125 preguntas de opción múltiple en 4 horas, no es a libro abierto. El punto de corte varía entre 60 % y 85 % según la dificultad de la versión del examen que toque. Existe además CEH Practical: 6 horas, 20 retos en un laboratorio (iLabs); quien aprueba ambos obtiene CEH Master. Validez: 3 años.
- Rol al que apunta: pentester junior, analista de seguridad, puestos gubernamentales y de consultoría que la piden por contrato (está en la lista DoD 8140).

Analogía: un examen teórico muy completo sobre todas las herramientas de un taller; demuestra que se conocen, no que se sabe reparar un coche bajo presión.

Ejemplo de pregunta típica: "Un atacante envía paquetes con los flags FIN, PSH y URG activados. ¿Qué tipo de escaneo es?" Respuesta: Xmas scan (`nmap -sX`). Es conocimiento de reconocimiento, no habilidad de explotación.

> [!WARNING]
> En la comunidad técnica, el CEH de opción múltiple tiene fama de medir memoria de herramientas más que habilidad. Sigue abriendo puertas en recursos humanos, gobierno y consultoras, pero para un puesto de pentester el examen práctico (CEH Practical, OSCP, CPTS o CRT) pesa mucho más.

```
CEH → hacking ético en opción múltiple; EC-Council; v13 (CEH AI).
Formato CEH → 312-50, 125 preguntas, 4 h; corte 60-85 %; validez 3 años.
Elegibilidad CEH → formación oficial o 2 años de experiencia + 100 USD.
CEH Practical → 6 h, 20 retos; CEH + Practical = CEH Master.
```

### CISA

**CISA** (Certified Information Systems Auditor) es una certificación de ISACA que valida la capacidad de auditar, controlar y asegurar sistemas de información: planificar auditorías de TI, evaluar controles y gobierno de TI, y reportar hallazgos.

- Emisor: ISACA (Information Systems Audit and Control Association).
- Enfoque: auditoría de TI, no ataque ni defensa técnica. Mira si los controles existen, están bien diseñados y funcionan.
- Nivel: avanzado (profesional con experiencia).
- Requisitos: 5 años de experiencia profesional en auditoría, control o seguridad de sistemas de información, obtenida en los 10 años anteriores a la solicitud. Se pueden sustituir hasta 3 años con exenciones (por ejemplo, ciertos títulos universitarios). Se puede aprobar el examen antes y hay 5 años desde el aprobado para solicitar la certificación.
- Formato: 150 preguntas de opción múltiple en 4 horas, en centros PSI o en remoto. El puntaje es escalado de 200 a 800 y se aprueba con 450.
- Dominios: Information System Auditing Process 18 %, Governance and Management of IT 18 %, Information Systems Acquisition, Development and Implementation 12 %, Information Systems Operations and Business Resilience 26 %, Protection of Information Assets 26 %.
- Precio verificado: 575 USD para miembros de ISACA y 760 USD para no miembros, más 50 USD de tasa de solicitud de certificación.
- Mantenimiento: CPE anuales y trienales (120 horas cada 3 años, mínimo 20 por año).
- Rol al que apunta: auditor de TI, auditor interno, consultor de riesgo y cumplimiento, Big Four.

Analogía: el inspector que revisa los libros contables de una empresa, pero para sistemas: no construye ni opera, comprueba que lo que dicen las políticas se cumple de verdad.

Ejemplo de pregunta típica: "Un auditor descubre que los desarrolladores tienen acceso de escritura a producción. ¿Qué debe hacer primero?" La respuesta correcta en mentalidad de auditor no es "quitar el acceso" (eso lo decide la dirección), sino documentar el hallazgo, evaluar el riesgo y comprobar si hay controles compensatorios.

```
CISA → auditoría de sistemas de información; ISACA.
Formato CISA → 150 preguntas, 4 h; aprueba 450 en escala 200-800.
Experiencia CISA → 5 años (hasta 3 con exenciones).
Precio CISA → 575 USD miembros / 760 USD no miembros + 50 USD solicitud.
```

### CISM

**CISM** (Certified Information Security Manager) es una certificación de ISACA que valida la capacidad de dirigir un programa de seguridad de la información: gobierno, gestión de riesgos, desarrollo del programa y gestión de incidentes, con mirada de negocio.

- Emisor: ISACA.
- Enfoque: gestión de la seguridad, no técnica. Cómo alinear la seguridad con los objetivos del negocio, cómo justificar presupuesto, cómo medir riesgo.
- Nivel: avanzado (gestión).
- Requisitos: 5 años de experiencia en seguridad de la información, de los cuales al menos 3 en gestión de seguridad, en 3 o más de las cuatro áreas del temario. La experiencia debe ser de los 10 años anteriores a la solicitud o de los 5 posteriores al aprobado. ISACA admite sustituir parte de la experiencia general con ciertas certificaciones o títulos, pero no los años de gestión.
- Formato: 150 preguntas de opción múltiple en 4 horas, puntaje escalado 200-800, aprobado 450.
- Cambio de temario verificado: el 3 de noviembre de 2026 entra en vigor un nuevo Exam Content Outline. Mantiene los cuatro dominios con nuevos pesos: Information Security Governance 18 %, Information Security Risk Management 20 %, Information Security Program 33 %, Incident Management 29 %, y añade contenido de arquitectura empresarial y arquitectura de seguridad.
- Precio verificado: 575 USD miembros, 760 USD no miembros, más 50 USD de tasa de solicitud.
- Rol al que apunta: gerente de seguridad, responsable de seguridad, CISO, consultor de gobierno de seguridad.

Analogía: el director de un hospital frente al cirujano. No opera, pero decide cuántos quirófanos hay, qué riesgos se aceptan y qué hacer cuando algo sale mal.

Ejemplo de pregunta típica: "La dirección quiere lanzar un producto que incumple la política de cifrado. ¿Qué hace el gerente de seguridad?" La respuesta CISM no es bloquearlo: es presentar el riesgo a la dirección en términos de negocio para que el dueño del riesgo decida aceptarlo, mitigarlo o transferirlo, y documentar la decisión.

```
CISM → gestión de programas de seguridad; ISACA.
Formato CISM → 150 preguntas, 4 h; 450 en escala 200-800.
Experiencia CISM → 5 años, 3 de ellos en gestión de seguridad.
Temario CISM desde 3 nov 2026 → 18 / 20 / 33 / 29 %.
CISA → audita controles; CISM → dirige el programa de seguridad.
```

### GSEC

**GSEC** (GIAC Security Essentials) es una certificación de GIAC que valida conocimientos prácticos de seguridad más allá de la terminología: defensa de redes, endpoints Windows y Linux, criptografía, nube, respuesta a incidentes y gestión de vulnerabilidades.

- Emisor: GIAC (ver la sección GIAC). Curso asociado de SANS: SEC401 Security Essentials: Network, Endpoint, and Cloud.
- Enfoque: amplitud técnica defensiva, con más profundidad práctica que Security+.
- Nivel: entrada a intermedio en la escala GIAC; pensada para profesionales nuevos en seguridad con base en TI y redes.
- Requisitos: ninguno formal. No es obligatorio hacer el curso de SANS.
- Formato: 106 preguntas en 4 horas, aprobado 72 %. Incluye preguntas CyberLive (prácticas, en máquinas virtuales con herramientas reales). A libro abierto con material impreso (ver GIAC).
- Áreas que evalúa (entre los objetivos publicados): control de acceso, criptografía, seguridad en Linux y Windows, respuesta a incidentes, seguridad en AWS y Azure, arquitectura de red, escaneo de vulnerabilidades y pruebas de penetración básicas.
- Precio: 999 USD por intento sin el curso (precio estándar GIAC verificado); el curso SANS se paga aparte y cuesta varias veces más.
- Rol al que apunta: analista de seguridad, administrador de seguridad, ingeniero de seguridad junior.

Analogía: si Security+ es el examen teórico de primeros auxilios, GSEC es el mismo temario con maniquí y desfibrilador en la sala.

Ejemplo de pregunta CyberLive: en una máquina virtual con un pcap, encontrar qué host interno hizo más consultas DNS a un dominio concreto y escribir su IP; se resuelve con Wireshark o `tcpdump` y un filtro, no recordando una definición.

```
GSEC → seguridad esencial práctica; GIAC; curso SANS SEC401.
Formato GSEC → 106 preguntas, 4 h, 72 %; CyberLive; libro abierto impreso.
Security+ → amplitud teórica; GSEC → amplitud con práctica.
```

### GPEN

**GPEN** (GIAC Penetration Tester) es una certificación de GIAC que valida la capacidad de planificar y ejecutar pruebas de penetración en redes empresariales: reconocimiento, escaneo, explotación, ataques a contraseñas, ataques a Active Directory y Kerberos, y elaboración de informes.

- Emisor: GIAC. Curso asociado: SANS SEC560 Enterprise Penetration Testing.
- Enfoque: pentesting de infraestructura con metodología profesional, incluidos ataques a Azure y frameworks de mando y control.
- Nivel: intermedio a avanzado.
- Requisitos: ninguno formal.
- Formato: 82 preguntas en 3 horas, aprobado 73 %. Incluye CyberLive. 120 días desde la activación para presentar el examen. A libro abierto con material impreso.
- Precio: 999 USD por intento (estándar GIAC).
- Rol al que apunta: pentester de redes, red teamer junior, consultor de seguridad ofensiva, sobre todo en empresas que valoran SANS.

Analogía: un examen de conducción en circuito cerrado, con preguntas teóricas y maniobras prácticas, frente a OSCP, que es una carrera de 24 horas.

Ejemplo de tarea práctica: en una VM, con un hash NTLM capturado, usar `hashcat -m 1000` con un diccionario para obtener la contraseña y luego autenticarse en otro host del laboratorio con ella.

```
GPEN → pentesting de red empresarial; GIAC; curso SANS SEC560.
Formato GPEN → 82 preguntas, 3 h, 73 %; CyberLive.
```

### GWAPT

**GWAPT** (GIAC Web Application Penetration Tester) es una certificación de GIAC que valida la capacidad de evaluar la seguridad de aplicaciones web con técnicas de pentesting: reconocimiento, autenticación, gestión de sesiones, inyección SQL, ataques cross-site y uso de herramientas de prueba.

- Emisor: GIAC. Curso asociado: SANS SEC542 Web App Penetration Testing and Ethical Hacking.
- Enfoque: pentesting web. Ocho áreas evaluadas: cross-site attacks, reconnaissance, authentication attacks, configuration testing, web application overview, session management, SQL injection attacks y testing tools. Detalle técnico de esos ataques en [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/).
- Nivel: intermedio.
- Requisitos: ninguno formal.
- Formato: 82 preguntas en 3 horas, aprobado 71 %. Incluye CyberLive. 120 días desde la activación. Libro abierto impreso.
- Precio: 999 USD por intento (estándar GIAC).
- Rol al que apunta: pentester web, analista de seguridad de aplicaciones (AppSec), bug bounty.

Analogía: un inspector especializado solo en puertas y ventanas: no revisa los cimientos del edificio, pero conoce cada tipo de cerradura.

Ejemplo de tarea práctica: con Burp Suite interceptar el login de una app de laboratorio, ver que la cookie de sesión no cambia tras autenticarse (session fixation) y demostrarlo fijando la cookie en un segundo navegador.

```
GWAPT → pentesting web; GIAC; curso SANS SEC542.
Formato GWAPT → 82 preguntas, 3 h, 71 %; CyberLive.
GPEN → infraestructura; GWAPT → aplicaciones web.
```

### GIAC

**GIAC** (Global Information Assurance Certification) es el organismo certificador asociado a SANS Institute que emite más de cuarenta certificaciones técnicas de ciberseguridad, cada una ligada normalmente a un curso de SANS. Existe para certificar habilidades técnicas concretas con exámenes rigurosos: GSEC, GPEN y GWAPT son tres de su catálogo, que cubre también forense (GCFA, GCFE), respuesta a incidentes (GCIH), análisis de malware (GREM) y nube, entre otras.

Rasgos comunes a sus certificaciones:

- Libro abierto con material impreso: se pueden llevar libros del curso y notas impresas o manuscritas, incluido un índice propio; no se permite nada digital ni material con aspecto de preguntas de examen. Hacer un buen índice de los libros del curso es la estrategia de estudio clásica.
- CyberLive: preguntas prácticas en máquinas virtuales con herramientas reales dentro del examen.
- Supervisión remota (ProctorU) o en centro (Pearson VUE).
- Precio verificado: la mayoría cuesta 999 USD por intento sin curso; las de nivel inicial menos (GFACT 399 USD). Reintento: 899 USD en la mayoría. Renovación: 499 USD.
- Validez: 4 años; se renueva con créditos de formación (CPE) y la tasa de renovación.
- Los cursos de SANS que preparan cada certificación son la parte cara (varios miles de dólares), pero no son obligatorios.

Analogía: una escuela de oficios con exámenes muy específicos: no da un título de "técnico general", sino uno por especialidad (electricista, fontanero, soldador).

Ejemplo de decisión: alguien que trabaja en un SOC y quiere especializarse en respuesta a incidentes elegiría GCIH; alguien que quiere hacer pentest web, GWAPT. Ninguna exige la otra.

```
GIAC → certificador de SANS; 40+ certificaciones técnicas especializadas.
Libro abierto impreso → se permite material del curso, notas e índice; nada digital.
CyberLive → preguntas prácticas en VM dentro del examen.
Precio GIAC → 999 USD la mayoría; reintento 899; renovación 499; validez 4 años.
```

### OSCP

**OSCP** (OffSec Certified Professional) es una certificación de OffSec (antes Offensive Security) que valida la capacidad de comprometer máquinas y un entorno de Active Directory en un examen práctico de 24 horas, y de documentarlo en un informe profesional. Es la certificación práctica de pentesting más reconocida para entrar en la profesión.

- Emisor: OffSec. Curso asociado: PEN-200 (Penetration Testing with Kali Linux).
- Enfoque: pentesting práctico de principio a fin: enumeración, explotación, escalada de privilegios en Linux y Windows, y ataque a Active Directory. Su lema histórico es "Try Harder".
- Nivel: intermedio; no tiene requisitos formales, pero se espera soltura en redes, Linux, Windows y scripting básico.
- Formato: 23 horas y 45 minutos de examen práctico supervisado por cámara, más 24 horas para entregar el informe. Se aprueba con 70 de 100 puntos: tres máquinas independientes de 20 puntos cada una y un conjunto de Active Directory de 40 puntos. Desde el 1 de noviembre de 2024 ya no hay puntos extra por ejercicios del curso.
- OSCP+: quien aprueba desde esa fecha recibe OSCP+, que caduca a los 3 años; el OSCP clásico no caduca.
- Precio verificado: 1.749 USD el paquete con curso, laboratorios y un intento.
- Rol: pentester junior, red team, consultor de seguridad ofensiva.

Analogía: es el examen práctico de conducir en ciudad real; no basta con saberse las señales, tienes que llegar al destino en un tiempo dado y luego explicar por escrito la ruta.

Ejemplo de preparación: 2 o 3 meses del curso PEN-200 y sus laboratorios, más unas 40 o 50 máquinas de práctica de dificultad fácil y media (TryHackMe, VulnHub), escribiendo un informe de cada una como si fuera el del examen.

```
OSCP → examen práctico de 24 h: 3 máquinas (60 pts) + Active Directory (40 pts); aprobar con 70.
OSCP+ → versión desde nov 2024; caduca a los 3 años.
Informe → 24 h extra; sin informe no hay certificación.
```

### CREST

**CREST** es un organismo acreditador sin ánimo de lucro, nacido en el Reino Unido, que certifica tanto a empresas de servicios de seguridad como a profesionales individuales, con exámenes muy valorados para trabajar en pentesting en Reino Unido, Europa, Australia y Asia. Existe para dar a los clientes una garantía independiente de que quien hace la prueba de penetración tiene un nivel mínimo comprobado.

Sus certificaciones de pentesting más comunes forman una escalera:

- CPSA (CREST Practitioner Security Analyst): nivel de entrada; examen teórico de 120 preguntas en 2 horas, sin material, aprobado con el 60 %. Cubre redes, protocolos, Windows, Linux, aplicaciones web y metodología.
- CRT (CREST Registered Penetration Tester): examen práctico de 2,5 horas más 15 minutos de lectura; exige tener el CPSA y tiene validez de 3 años. Es el requisito habitual para trabajar como pentester en empresas acreditadas por CREST.
- CCT (CREST Certified Tester), en variantes de infraestructura y de aplicaciones: el nivel senior, mucho más exigente.
- Rol: pentester en consultoras acreditadas, sobre todo fuera de América.

Analogía: es la licencia de un gremio profesional: el cliente no contrata a cualquiera, contrata a alguien con el sello del colegio.

Ejemplo: una consultora de Londres que quiere hacer pentests a bancos necesita estar acreditada por CREST, y para eso su personal tiene que tener CRT o CCT; por eso esas ofertas de empleo piden CREST antes que otras certificaciones.

```
CREST → acreditador de empresas y profesionales de pentest; fuerte en Reino Unido y Commonwealth.
CPSA → teórico de entrada: 120 preguntas, 2 h, 60 %.
CRT  → práctico de 2,5 h; exige CPSA; válido 3 años.
CCT  → nivel senior, infraestructura o aplicaciones.
```

### CISSP

**CISSP** (Certified Information Systems Security Professional) es una certificación de ISC2 que acredita conocimiento amplio de gestión y arquitectura de seguridad en los 8 dominios de su cuerpo de conocimiento, y es la certificación más pedida para puestos senior y de liderazgo en ciberseguridad.

- Emisor: ISC2.
- Enfoque: amplitud y gestión, no técnica profunda: "piensa como un gerente". Se dice que mide un kilómetro de ancho y un centímetro de profundidad.
- Requisitos: 5 años de experiencia remunerada en al menos 2 de los 8 dominios (1 año se puede convalidar con un título universitario o con ciertas certificaciones). Quien aprueba sin la experiencia queda como Associate of ISC2 hasta reunirla.
- Formato: examen adaptativo (CAT) de 100 a 150 preguntas en 3 horas; la dificultad se ajusta según las respuestas.
- Precio verificado: 749 USD.
- Los 8 dominios, con su peso en el examen:
  1. Security and Risk Management, 16 %.
  2. Asset Security, 10 %.
  3. Security Architecture and Engineering, 13 %.
  4. Communication and Network Security, 13 %.
  5. Identity and Access Management, 13 %.
  6. Security Assessment and Testing, 12 %.
  7. Security Operations, 13 %.
  8. Software Development Security, 10 %.
- Rol: responsable de seguridad, arquitecto, consultor senior, camino a CISO.

Analogía: es la especialidad de medicina general para jefes de servicio: no te hace cirujano, pero te permite coordinar a todos los especialistas.

Ejemplo: una pregunta típica no pide el comando para aislar un equipo, sino qué debe hacer primero un responsable de seguridad ante un incidente; la respuesta correcta suele ser la que protege a las personas y sigue la política, no la más técnica.

```
CISSP → ISC2; gestión y arquitectura en 8 dominios; puestos senior.
Requisito → 5 años en 2 de 8 dominios; sin ellos, Associate of ISC2.
Examen → adaptativo, 100-150 preguntas, 3 h.
```

## Ruta sugerida

Una **ruta de certificación** es el orden en que conviene sacar las certificaciones para que cada una se apoye en la anterior y apunte al rol que se busca. No hace falta sacarlas todas: lo que más pesa al buscar trabajo es la combinación de una o dos certificaciones con práctica demostrable (CTFs, laboratorios, un blog con informes).

```
                 Base (0-12 meses)
     Network+ ──► Security+         (A+ y Linux+ si vienes de cero en soporte o Linux;
        │                             CCNA en lugar de Network+ si apuntas a redes)
        │
        ├──────────── Ofensivo ─────────────┐
        │                                   │
        ▼                                   ▼
   Defensivo (SOC, IR)               Pentest / red team
   práctica: TryHackMe SOC L1,       práctica: TryHackMe, VulnHub,     
   CyberDefenders, BTLO              picoCTF, pwn.college
        │                                   │
   GSEC o CySA+                      CEH (si lo pide RR. HH.) ──► OSCP ──► GPEN / CRT / GWAPT
        │                                   │
        └──────────► Gestión y senior ◄─────┘
                     CISA (auditoría) · CISM (gestión) · CISSP (liderazgo)
```

Ejemplo de primer año: meses 1 a 4, Network+ con práctica de redes; meses 5 a 8, Security+ en paralelo con las rutas gratuitas de TryHackMe; meses 9 a 12, elegir lado y acumular 30 a 50 retos o máquinas resueltas y documentadas.

> [!TIP]
> Para el primer empleo, Security+ más práctica demostrable abre más puertas que una certificación avanzada sin experiencia; las certificaciones de gestión (CISSP, CISM, CISA) exigen años de experiencia y llegan después.

## Recursos para aprender y practicar

### Videos

- [What is CTF? An introduction to security Capture The Flag competitions](https://www.youtube.com/watch?v=8ev9ZX9J45A) — LiveOverflow; qué es un CTF y cómo se juega (CTFs).
- [BEGINNER Capture The Flag - PicoCTF 2021 001 "Obedient Cat"](https://www.youtube.com/watch?v=P07NH5F-t3s) — John Hammond; inicio de su serie resolviendo picoCTF (picoCTF).
- [Introduction: What is pwn.college?](https://www.youtube.com/watch?v=hh4XAU6XYP0) — pwn.college; cómo funciona la plataforma (pwn.college).
- [Free CCNA | Network Devices | Day 1](https://www.youtube.com/watch?v=H8W9oMNSuwo) — Jeremy's IT Lab; inicio del curso completo y gratuito de CCNA (CCNA).
- [Everyone Is Confused About the OSCP+](https://www.youtube.com/watch?v=Vun7pLuihhI) — Tib3rius; qué cambió con OSCP+ (OSCP).
- [OSCP vs. GPEN vs. CEH: The Ultimate Guide!](https://www.youtube.com/watch?v=aBqUK9YkjQk) — Luke Gough; comparación de las tres certificaciones ofensivas (CEH, GPEN, OSCP).

El curso completo y gratuito de Security+ y Network+ está en [professormesser.com](https://www.professormesser.com/).

### Lectura y documentación

- [CompTIA A+](https://www.comptia.org/en-us/certifications/a/), [Linux+](https://www.comptia.org/en-us/certifications/linux/), [Network+](https://www.comptia.org/en-us/certifications/network/) y [Security+](https://www.comptia.org/en-us/certifications/security/) — páginas oficiales con objetivos de examen y precios.
- [Cisco CCNA](https://www.cisco.com/site/us/en/learn/training-certifications/certifications/enterprise/ccna/index.html) — página oficial del CCNA 200-301.
- [EC-Council CEH](https://www.eccouncil.org/train-certify/certified-ethical-hacker-ceh/) — página oficial del CEH.
- [ISACA CISA](https://www.isaca.org/credentialing/cisa) y [CISM](https://www.isaca.org/credentialing/cism) — requisitos y dominios.
- [GIAC](https://www.giac.org/): [GSEC](https://www.giac.org/certifications/security-essentials-gsec/), [GPEN](https://www.giac.org/certifications/penetration-tester-gpen/) y [GWAPT](https://www.giac.org/certifications/web-application-penetration-tester-gwapt/) — objetivos y formato de cada examen.
- [OffSec PEN-200 (OSCP)](https://www.offsec.com/courses/pen-200/) — curso y examen.
- [CREST CPSA](https://www.crest-approved.org/skills-certifications-careers/crest-practitioner-security-analyst/) y [CREST CRT](https://www.crest-approved.org/skills-certifications-careers/crest-registered-penetration-tester/) — formato y requisitos.
- [ISC2 CISSP](https://www.isc2.org/certifications/cissp) y [requisitos de experiencia](https://www.isc2.org/certifications/cissp/cissp-experience-requirements) — dominios y experiencia exigida.
- [CTFtime](https://ctftime.org/) — calendario de CTFs públicos y clasificación de equipos.

### Práctica

- [TryHackMe: Pentesting Fundamentals](https://tryhackme.com/room/pentestingfundamentals), [Vulnversity](https://tryhackme.com/room/vulnversity), [Basic Pentesting](https://tryhackme.com/room/basicpentestingjt), [Blue](https://tryhackme.com/room/blue) y [Simple CTF](https://tryhackme.com/room/easyctf) — gratis; las primeras máquinas completas para practicar la metodología del OSCP.
- [TryHackMe: SOC Fundamentals](https://tryhackme.com/room/socfundamentals) y [SOC L1 Alert Triage](https://tryhackme.com/room/socl1alerttriage) — gratis; el lado defensivo de la ruta.
- [VulnHub: Kioptrix Level 1](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/) y [Mr-Robot](https://www.vulnhub.com/entry/mr-robot-1,151/) — máquinas clásicas para tu laboratorio local.
- [picoCTF](https://picoctf.org/) — entra a picoGym y resuelve retos por categoría, empezando por General Skills.
- [pwn.college](https://pwn.college/) — empieza por los dojos iniciales (Linux Luminarium) antes de pasar a explotación de binarios.
- [OverTheWire](https://overthewire.org/wargames/) — wargames por niveles: Bandit (Linux), Natas (web), Narnia (binarios).
- [CyberDefenders](https://cyberdefenders.org/blueteam-ctf-challenges/) — investigaciones defensivas gratuitas con pcaps, logs e imágenes de disco.
- [Ejercicio 1: Monta el laboratorio local y descubre la máquina objetivo](ejercicios.md#ejercicio-1-monta-el-laboratorio-local-y-descubre-la-máquina-objetivo) — deja Kioptrix Level 1 y una VM de ataque en una red host-only aislada y enumera sus servicios.
- [Ejercicio 2: Compromete Kioptrix Level 1 y redacta el informe](ejercicios.md#ejercicio-2-compromete-kioptrix-level-1-y-redacta-el-informe) — haz el ciclo completo hasta root y documéntalo con la plantilla de informe del OSCP.

## Cuadro resumen

Todo lo visto, en una línea por término.

CTFs (Capture the Flag)

```
CTF         → competencia de retos de seguridad; cada reto esconde una bandera.
Jeopardy    → retos independientes por categoría y puntos.
Attack-Defense → cada equipo defiende sus servicios y ataca los del resto.
HackTheBox  → máquinas realistas con pocas pistas; más abierto que TryHackMe.
TryHackMe   → salas guiadas con preguntas; para empezar de cero.
VulnHub     → VMs vulnerables descargables; gratis y sin conexión.
picoCTF     → CTF Jeopardy gratuito de Carnegie Mellon; picoGym abierto todo el año.
pwn.college → curso gratuito de seguridad de sistemas en dojos.
```

Beginner Certifications

```
CompTIA A+       → soporte técnico: hardware, sistemas operativos y resolución de problemas.
CompTIA Linux+   → administración de Linux.
CompTIA Network+ → fundamentos de redes, neutral respecto al fabricante.
CCNA             → redes con equipos Cisco; configuración práctica.
CompTIA Security+→ fundamentos de seguridad; la certificación base más pedida.
```

Advanced Certifications

```
CEH    → EC-Council; hacking ético, mayormente teórico; muy pedido por RR. HH.
CISA   → ISACA; auditoría de sistemas de información.
CISM   → ISACA; gestión de un programa de seguridad.
GSEC   → GIAC; fundamentos técnicos amplios, con libro abierto impreso.
GPEN   → GIAC; pentesting de red.
GWAPT  → GIAC; pentesting de aplicaciones web.
GIAC   → certificador de SANS; 40+ certificaciones técnicas especializadas.
OSCP   → examen práctico de 24 h: 3 máquinas (60 pts) + Active Directory (40 pts); aprobar con 70.
CREST  → acreditador de pentest; CPSA teórico de entrada, CRT práctico.
CISSP  → ISC2; gestión y arquitectura en 8 dominios; 5 años de experiencia.
```

Ruta sugerida

```
Base      → Network+ y Security+ (A+, Linux+ o CCNA según el punto de partida).
Defensivo → práctica SOC y GSEC o CySA+.
Ofensivo  → práctica en CTFs y máquinas, OSCP, luego GPEN, CRT o GWAPT.
Senior    → CISA, CISM o CISSP, con años de experiencia.
```
