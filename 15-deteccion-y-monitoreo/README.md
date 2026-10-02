# Detección y monitoreo

## Conceptos previos

- SOC (Security Operations Center): el equipo que vigila la seguridad de una organización las 24 horas, recibe alertas, las investiga y escala las que son reales.
- Analista SOC: la persona que hace el triaje de alertas. Nivel 1 (L1) clasifica, nivel 2 (L2) investiga a fondo, nivel 3 (L3) caza amenazas y afina detecciones.
- Evento: cualquier cosa que un sistema registra (un login, un paquete, un proceso que arranca). Alerta: un evento o conjunto de eventos que una regla consideró sospechoso.
- Log (registro): línea de texto o registro estructurado que un sistema escribe cuando pasa algo, normalmente con fecha, origen y descripción.
- IOC (Indicator of Compromise): dato observable que delata un ataque conocido: un hash de archivo, una IP, un dominio, una URL.
- Hash: resumen de longitud fija de un archivo (MD5 de 128 bits, SHA-256 de 256 bits); si cambia un byte, cambia el hash. Ver [`10-criptografia`](../10-criptografia/).
- Falso positivo / falso negativo: alerta sobre algo inocente / ataque que pasa sin alerta. Explicados con métricas en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/).
- Firma (signature): patrón fijo que identifica algo conocido, como una secuencia de bytes o un hash.
- Línea base (baseline): lo que es normal en una red o un equipo concreto (cuánto tráfico, a qué horas, qué procesos).
- Inline: un equipo está inline cuando el tráfico tiene que atravesarlo para llegar a su destino, como un peaje en una carretera.
- SPAN / TAP: un puerto SPAN (o mirror) es un puerto del switch que recibe una copia del tráfico de otros puertos; un TAP es un dispositivo físico que copia el tráfico de un cable. Ninguno de los dos puede bloquear nada.
- Tupla de 5 (5-tuple): IP origen, IP destino, puerto origen, puerto destino y protocolo; identifica una conexión. Puertos en [`05-protocolos-y-puertos`](../05-protocolos-y-puertos/).
- Payload: el contenido útil de un paquete, lo que va después de las cabeceras.
- Sandbox: entorno aislado donde se ejecuta algo sospechoso para ver qué hace sin poner en riesgo el equipo real. Ver [`14-defensa-y-hardening`](../14-defensa-y-hardening/).
- Defang: escribir un IOC de forma que no sea clicable (`hxxp://malo[.]com`, `203.0.113[.]7`) para compartirlo sin riesgo.
- Direcciones de ejemplo: `192.0.2.0/24`, `198.51.100.0/24` y `203.0.113.0/24` están reservadas para documentación; en esta nota hacen de "IP externa". Ver [`04-direccionamiento-ip`](../04-direccionamiento-ip/).

## Detección de intrusiones

### Basics of IDS and IPS

Un **IDS (Intrusion Detection System)** es un sistema de vigilancia que inspecciona tráfico o actividad de un equipo, la compara con reglas o con un modelo de lo normal y genera una alerta cuando algo coincide; un **IPS (Intrusion Prevention System)** es lo mismo, pero además puede bloquear lo que detecta.

Existen porque un firewall decide por direcciones y puertos: deja pasar todo lo que va al puerto 443 de un servidor web, sea una visita normal o un ataque de inyección SQL dentro de esa misma conexión. El IDS/IPS mira dentro del paquete y del flujo, donde el firewall clásico no mira.

Analogía: el firewall es el portero que revisa si tu nombre está en la lista; el IDS es la cámara de seguridad con alguien mirando la pantalla, que avisa pero no detiene a nadie; el IPS es el guardia de la puerta que, además de mirar, puede impedir que entres.

Cómo detecta. Hay dos métodos, y casi todos los productos combinan ambos:

- Detección por firmas: compara lo que ve con una base de patrones conocidos ("una petición HTTP cuyo URI contiene `../../etc/passwd`"). Es precisa y barata, y explica por qué disparó (la regla tal), pero no ve nada que no esté en la base: un ataque nuevo o una variante con un byte cambiado pasa sin alerta.
- Detección por anomalías: aprende una línea base y alerta cuando algo se aleja de ella ("este servidor nunca habla por DNS con más de 50 consultas por minuto y ahora hace 5.000"). Puede ver ataques nunca vistos, pero produce más falsos positivos (un backup nocturno nuevo también es "anómalo") y es difícil explicar por qué disparó.

Variante intermedia: la detección por inspección de protocolo con estado (stateful protocol analysis), que conoce cómo debe comportarse un protocolo (un comando FTP no debería medir 4.000 bytes) y alerta ante desvíos de la especificación.

Cómo se coloca. Esta es la diferencia práctica entre IDS e IPS:

```
Pasivo (IDS): ve una copia, no puede bloquear

  Internet ── Firewall ── Switch ──── Servidores
                            │ SPAN (copia)
                            └──── IDS ──> alerta al SIEM

Inline (IPS): el tráfico lo atraviesa, puede descartar paquetes

  Internet ── Firewall ── IPS ── Switch ── Servidores
                           │
                           └──> alerta + paquete descartado
```

El IDS pasivo no añade latencia ni puede tumbar la red si falla, pero cuando avisa el paquete ya llegó. El IPS inline puede descartar el paquete, cortar la conexión con un TCP RST o bloquear la IP durante un tiempo, pero cada milisegundo que tarda en inspeccionar se suma a todo el tráfico, y si se cae hay que decidir de antemano qué pasa: fail-open (deja pasar todo, la red sigue viva pero sin protección) o fail-closed (corta todo, seguro pero sin servicio).

Falsos positivos. Una regla dispara sobre tráfico inocente: un IDS lo convierte en ruido para el analista; un IPS lo convierte en una caída de servicio, porque bloquea a usuarios legítimos. Por eso las reglas nuevas se despliegan primero en modo solo alerta, se miden unos días y solo después se pasan a bloqueo. Las métricas (precisión, recall, falacia de la tasa base) están en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/).

> [!WARNING]
> Un falso positivo en un IPS no es una alerta molesta: es un usuario legítimo bloqueado. Toda regla nueva va primero en modo alerta y pasa a bloqueo cuando se midió su tasa de falsos positivos.

Snort y Suricata son los dos motores libres de referencia. Snort (de Cisco Talos, versión actual Snort 3) y Suricata (de la OISF, multihilo de origen) usan casi la misma sintaxis de reglas y pueden funcionar como IDS o como IPS según cómo se instalen. Una regla tiene una cabecera (acción, protocolo, origen, flecha de dirección, destino) y unas opciones entre paréntesis:

```
acción  proto  origen     puerto  dir  destino     puerto   (opciones)
alert   icmp   any        any     ->   $HOME_NET   any      (msg:"..."; itype:8; sid:1000001; rev:1;)
```

Ejemplo de laboratorio, inofensivo: detectar un ping hacia tu red y una cadena concreta en una URL HTTP. Archivo `local.rules`:

```
alert icmp any any -> $HOME_NET any (msg:"LAB ICMP Echo Request hacia la red interna"; itype:8; sid:1000001; rev:1;)
alert http any any -> $HOME_NET any (msg:"LAB cadena de prueba en el URI"; http_uri; content:"/hola-ids"; nocase; sid:1000002; rev:1;)
```

- `alert`: la acción. En modo IPS existen `drop` (descarta y alerta) y `reject` (descarta y envía un RST o un ICMP de rechazo).
- `itype:8`: tipo ICMP 8, Echo Request, el ping.
- `http_uri; content:"/hola-ids"; nocase;`: busca esa cadena dentro del URI ya normalizado, sin distinguir mayúsculas.
- `sid`: identificador único de la regla. Los sid de 1.000.000 en adelante están reservados para reglas locales, para no chocar con las oficiales.
- `rev`: versión de la regla; se incrementa al editarla.

Se prueba con Snort 3 leyendo de la interfaz y mostrando alertas en formato corto, y desde otra máquina del laboratorio se lanza `ping -c 1 192.168.56.20` y `curl http://192.168.56.20/hola-ids`:

```
$ sudo snort -c /etc/snort/snort.lua -R local.rules -i eth1 -A alert_fast
10/02-14:03:11.120344 [**] [1:1000001:1] "LAB ICMP Echo Request hacia la red interna" [**] [Priority: 0] {ICMP} 192.168.56.10 -> 192.168.56.20
10/02-14:03:15.402981 [**] [1:1000002:1] "LAB cadena de prueba en el URI" [**] [Priority: 0] {TCP} 192.168.56.10:51544 -> 192.168.56.20:80
```

`[1:1000001:1]` se lee generador:sid:revisión. Suricata escribe lo mismo en `fast.log` y, con todo el detalle en JSON, en `eve.json`, que es lo que normalmente se envía al SIEM. Para Suricata la segunda regla se escribe con el buffer `http.uri;` en lugar de `http_uri;`.

```
IDS → detecta y alerta; pasivo, ve una copia; no bloquea.
IPS → detecta y bloquea; inline, el tráfico lo atraviesa.
Firma → patrón conocido; preciso, ciego ante lo nuevo.
Anomalía → desvío de la línea base; ve lo nuevo, más falsos positivos.
Fail-open / fail-closed → si el IPS cae, deja pasar todo / corta todo.
Firewall → decide por IP, puerto y estado; IPS → mira dentro del flujo.
```

### NIDS

Un **NIDS (Network-based IDS)** es un IDS que se coloca en un punto de la red y analiza el tráfico de todos los equipos que pasan por ese punto, en lugar de vigilar un solo equipo.

Existe porque instalar un agente en cada máquina no siempre es posible (impresoras, cámaras IP, equipos industriales, invitados) y porque un sensor bien colocado ve de una sola vez la conversación de cientos de equipos. Su límite es que solo ve lo que pasa por su segmento y, si el tráfico va cifrado con TLS, solo ve metadatos (IPs, puertos, el nombre del servidor en el SNI, el certificado), no el contenido.

Analogía: es la cámara del pasillo de un edificio de oficinas. Ve quién entra y sale de cada despacho, pero no lo que pasa dentro de un despacho con la puerta cerrada, ni a quien va por otro pasillo.

Cómo se despliega:

- Ubicación: en la frontera con Internet (detrás del firewall, para no alertar de lo que el firewall ya bloquea), entre la DMZ y la red interna y delante de los servidores críticos. Cada segmento que importa necesita su sensor.
- Captura: por puerto SPAN o TAP. El SPAN puede perder paquetes si el switch está saturado; el TAP copia todo pero cuesta dinero y hay que instalarlo en el cable.
- Capacidad: un enlace de 1 Gbit/s al 70 % de uso son unos 87 MB por segundo que inspeccionar; si el sensor no da abasto, descarta paquetes sin avisar y sin alertar. Se vigila el contador de paquetes perdidos (`kernel_drops` en Suricata).
- Lo que no ve: tráfico este-oeste entre dos equipos del mismo switch si no se replica, tráfico cifrado (salvo descifrado TLS en un proxy) y lo que pasa dentro de un equipo.

Ejemplo: una red con 300 equipos y un sensor Suricata en el SPAN del switch de núcleo. A las 03:00 dispara una regla de Emerging Threats por un User-Agent asociado a un malware conocido desde `10.0.5.44` hacia `203.0.113.80:8080`. El analista no necesitó ningún agente en `10.0.5.44` para saber que ese equipo es el primero que hay que revisar.

```
NIDS → sensor en la red; ve muchos equipos; no ve dentro del host ni el contenido cifrado.
HIDS → agente en el equipo; ve procesos, archivos y logs de ese equipo.
SPAN → copia del switch; barato, puede perder paquetes.
TAP → copia física del cable; completo, cuesta instalarlo.
```

### NIPS

Un **NIPS (Network-based IPS)** es un IPS de red que se instala inline en un enlace y puede descartar paquetes o cortar conexiones que coinciden con una regla antes de que lleguen a su destino.

Existe porque en ciertos ataques avisar no basta: un exploit que funciona en un solo paquete ya hizo el daño cuando el analista lee la alerta. El NIPS gana tiempo bloqueando en el acto. Hoy rara vez es una caja separada: suele ser una función del next-generation firewall (NGFW), que combina firewall, IPS, filtro de URLs y control de aplicaciones.

Analogía: es el control de seguridad del aeropuerto. Todo pasajero pasa por el arco; si pita, no embarca. Pero si el arco se estropea o es demasiado lento, se forma una cola que afecta a todos, y si es demasiado sensible, deja en tierra a gente inocente.

Cómo actúa cuando una regla coincide:

1. Descarta el paquete (`drop`) y, en reglas con `reject`, avisa al origen con un TCP RST o un ICMP de inalcanzable.
2. Puede bloquear la IP de origen durante un tiempo (por ejemplo 10 minutos).
3. Registra la alerta igual que un IDS, con la diferencia de que el campo de acción dice `blocked`.

Decisiones de diseño con números: un NIPS que añade 2 ms por paquete es aceptable en navegación web, pero no en una red de bolsa o de control industrial. Se dimensiona con margen (un sensor para 1 Gbit/s no se pone en un enlace de 10 Gbit/s) y se decide fail-open o fail-closed según qué duele más, quedarse sin protección o sin servicio. Un hospital suele elegir fail-open; un sistema de pagos con datos de tarjetas puede elegir fail-closed.

Ejemplo: Suricata en modo IPS con una regla `drop` contra un exploit conocido de un servidor web. Llegan 40 peticiones del exploit en un minuto desde `198.51.100.23`; las 40 se descartan, el servidor nunca las recibe y en `eve.json` aparecen 40 eventos con `"action":"blocked"`.

El HIPS, el equivalente en el propio equipo, se explica en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

```
NIDS → detecta en la red; pasivo.
NIPS → bloquea en la red; inline.
HIPS → bloquea en el equipo (ver 14-defensa-y-hardening).
NGFW → firewall + IPS + control de aplicaciones en una caja.
```

### Un IPS solo puede bloquear lo que lo atraviesa

> [!IMPORTANT]
> La capacidad de bloquear no la da el software sino la posición: el mismo Suricata es IDS en un puerto SPAN e IPS inline. Lo que va por otro camino, o cifrado, no lo ve ninguno de los dos.

La posición de un sensor es una decisión de arquitectura que fija qué tráfico puede ver y qué puede hacer con él, y por eso pesa más que la marca del producto.

```
                  IDS en SPAN    IPS inline        Agente en el host (HIPS/EDR)
Ve el tráfico     copia          original          solo el de ese equipo
Puede bloquear    no             sí                sí, en ese equipo
Si falla          nada se corta  fail-open/closed  solo ese equipo
Ve contenido TLS  no             no (sin proxy)    sí, antes de cifrar
```

La fila "Puede bloquear" depende solo de la columna, es decir, de la posición. Por eso una red madura combina sensores de red con agentes en los equipos.

Ejemplo: un atacante que ya está dentro se mueve de un PC a otro dentro de la misma VLAN. Ese tráfico nunca pasa por el NIPS del perímetro; solo lo ven un agente en los equipos o un sensor en ese switch.

```
Posición → decide si el sensor ve y si puede bloquear.
Tráfico este-oeste → entre equipos internos; el sensor del perímetro no lo ve.
```

## Understand the following

### SIEM

Un **SIEM (Security Information and Event Management)** es una plataforma que recoge los logs de toda la organización en un solo lugar, los pone en un formato común, los correlaciona con reglas y genera alertas que un analista puede investigar con búsquedas.

Existe porque cada sistema guarda sus propios logs en su propio formato y en su propio disco: el controlador de dominio, el firewall, el proxy, el servidor de correo, el EDR. Un ataque real deja una huella pequeña en cada uno, y ninguna de esas huellas sola parece grave. Además, un atacante que toma un equipo puede borrar sus logs locales; si ya se enviaron al SIEM, la copia queda a salvo.

Analogía: es la sala de control de una ciudad que recibe las cámaras de todas las calles, las alarmas de todos los bancos y las llamadas a emergencias en una sola pared de pantallas, con alguien que sabe que "alarma en el banco + coche a toda velocidad en la calle de al lado" es un atraco y no dos incidentes sueltos.

Cómo funciona por dentro, en cuatro etapas:

```
Fuentes                 Recolección          Normalización        Correlación        Analista
DC Windows ──agente──┐
Firewall ──syslog────┼──> colector ──> parseo a campos ──> reglas + ──> alerta ──> búsqueda
Proxy ──API──────────┤                  comunes (src_ip,     ventanas de        e investigación
EDR ──API────────────┘                  user, action, time)  tiempo
                                              │
                                              └──> almacenamiento (retención: p. ej. 90 días caliente, 1 año frío)
```

1. Recolección: agentes en los equipos (Splunk Universal Forwarder, Elastic Agent, Wazuh agent), syslog desde equipos de red y APIs para servicios en la nube. Lo crítico es la marca de tiempo: si los relojes no están sincronizados por NTP, la secuencia de eventos sale desordenada.
2. Normalización (parseo): convierte cada formato en campos comunes. El firewall dice `SRC=203.0.113.7`, Windows dice `IpAddress` y el proxy dice `c-ip`; tras normalizar, los tres son `src_ip` y se pueden buscar juntos. Esquemas comunes: Splunk CIM, Elastic ECS.
3. Correlación: reglas que unen eventos de distintas fuentes en una ventana de tiempo. Ejemplo: "10 o más logins fallidos (4625) para el mismo usuario en 5 minutos, seguidos de un login correcto (4624) desde la misma IP" es un ataque de fuerza bruta que tuvo éxito; ninguno de esos eventos solo lo dice.
4. Alerta, búsqueda y retención: la alerta va a una cola de triaje; el analista pivota con búsquedas; los datos se conservan el tiempo que pidan la investigación y la normativa. NIST SP 800-92 trata la gestión de logs en detalle.

Ejemplo concreto: búsqueda en SPL (Search Processing Language, el lenguaje de Splunk) que encuentra IPs con muchos logins fallidos en los últimos 15 minutos:

```
index=wineventlog EventCode=4625 earliest=-15m
| stats count AS fallos, dc(TargetUserName) AS usuarios_distintos BY IpAddress
| where fallos > 10
| sort - fallos
```

```
IpAddress        fallos  usuarios_distintos
203.0.113.45     312     87
192.168.10.23    14      1
```

Se lee así: `203.0.113.45` probó 87 usuarios distintos, 312 intentos; eso es password spraying desde fuera. `192.168.10.23` falló 14 veces con un solo usuario: probablemente alguien que olvidó su contraseña o un servicio con la clave vieja guardada. El siguiente paso del analista es pivotar: `index=wineventlog EventCode=4624 IpAddress=203.0.113.45` para saber si alguno de esos intentos acabó en un login correcto.

Productos: Splunk, Microsoft Sentinel, Elastic Security, IBM QRadar; Wazuh y Security Onion como opciones libres para laboratorio.

> [!TIP]
> Un SIEM no detecta nada que no reciba. Ante una alerta que "no debería faltar", lo primero es comprobar si la fuente envía logs y si la normalización llena los campos que usa la regla.

```
Recolección → traer los logs de cada fuente al SIEM.
Normalización → poner nombres de campo comunes a formatos distintos.
Correlación → unir eventos de varias fuentes en una ventana de tiempo.
SIEM → recoge, normaliza, correlaciona, alerta, busca y retiene.
Log management → solo recoge y guarda; sin correlación.
```

### SOAR

Un **SOAR (Security Orchestration, Automation and Response)** es una plataforma que ejecuta automáticamente los pasos repetitivos de la respuesta a una alerta (enriquecer, consultar, bloquear, abrir un ticket) siguiendo un playbook, y deja al analista solo las decisiones que requieren criterio.

Existe porque un SOC recibe cientos o miles de alertas al día y la mayoría exige los mismos diez minutos de trabajo mecánico: copiar la IP, mirarla en VirusTotal, buscar el usuario en el directorio, mirar si el hash ya se vio, abrir un ticket. Hecho a mano, eso agota a las personas y deja alertas sin mirar. El SOAR hace esos diez minutos en segundos.

Analogía: es el piloto automático de un avión. Mantiene rumbo y altitud y ejecuta listas de comprobación, pero el despegue, el aterrizaje y cualquier emergencia rara siguen en manos del piloto.

Cómo funciona:

- Orquestación: conectores (integraciones por API) con el SIEM, el EDR, el firewall, el correo, el directorio, VirusTotal y el sistema de tickets. El SOAR habla con todos ellos.
- Automatización: cada paso que una máquina puede hacer sin criterio humano.
- Playbook: el flujo de pasos para un tipo de alerta, con condiciones ("si el veredicto es malicioso, entonces...") y puntos de aprobación humana. Playbooks y runbooks se definen en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/).
- Caso: el SOAR agrupa la alerta, todo lo que encontró y lo que se hizo, con fecha y hora; sirve de registro para la investigación.

Ejemplo concreto, playbook de "correo de phishing reportado por un usuario":

```
1. Usuario reporta correo  ──>  SOAR crea caso
2. Extrae IOCs: remitente, URLs, adjuntos (hash SHA-256)
3. Enriquece: URL a urlscan, hash a VirusTotal, dominio a WHOIS
4. ¿Algún veredicto malicioso?
      no ──> cierra el caso como benigno, responde al usuario      (automático)
      sí ──> 5. busca el mismo correo en todos los buzones (encontró 23)
             6. PIDE APROBACIÓN al analista para borrarlos           (humano)
             7. borra los 23, bloquea el dominio en el proxy
             8. si alguien hizo clic (log del proxy), abre incidente y resetea su contraseña
```

Con 200 reportes al día a 10 minutos cada uno son más de 33 horas de trabajo manual; si el playbook cierra solo el 80 % benigno, quedan 40 casos para personas.

> [!NOTE]
> Lo que se automatiza sin aprobación son las acciones reversibles y de bajo impacto (enriquecer, etiquetar). Borrar correos, aislar equipos o bloquear cuentas suelen llevar un paso de aprobación humana.

```
SIEM → detecta: recoge logs y genera alertas.
SOAR → responde: ejecuta playbooks sobre esas alertas.
Orquestación → conectar herramientas entre sí.
Automatización → ejecutar pasos sin intervención humana.
Playbook → flujo de decisiones para un tipo de alerta.
```

## Learn how to find and use these logs

### Event Logs

Los **Event Logs de Windows** son el sistema de registro de Windows que guarda, en archivos `.evtx` con formato estructurado, cada evento del sistema identificado por un número (Event ID), un canal, una fecha y unos campos propios de ese ID.

Existen porque Windows necesita dejar constancia de quién entró, qué se ejecutó y qué cambió, y porque la mayoría de las redes corporativas giran alrededor de Active Directory: las huellas de un ataque a una red Windows están, casi siempre, en estos registros.

Analogía: es el libro de registro de la recepción de un edificio, donde cada tipo de suceso tiene su propio código: "E-1 entrada", "E-2 entrada rechazada", "L-1 llave maestra entregada". Para saber qué pasó un domingo por la noche no se lee el libro entero, se buscan los códigos.

Dónde están y cómo se leen:

- Archivos: `C:\Windows\System32\winevt\Logs\` (`Security.evtx`, `System.evtx`, `Application.evtx` y uno por cada canal adicional).
- Canales que más usa un analista: Security (logins, cuentas, privilegios), System (servicios, drivers, arranques), `Microsoft-Windows-PowerShell/Operational` (evento 4104, el código de los scripts ejecutados) y `Microsoft-Windows-Sysmon/Operational` si se instaló Sysmon.
- Herramientas: el Visor de eventos (`eventvwr.msc`), `Get-WinEvent` en PowerShell, `wevtutil` en la consola; en producción se envían al SIEM.
- Lo que se registra depende de la política de auditoría (Advanced Audit Policy). Por ejemplo, 4688 no aparece si no se activa "Audit Process Creation", y su línea de comandos tampoco si no se activa "Include command line in process creation events".

Los Event IDs que un analista debe reconocer de memoria:

- 4624: inicio de sesión correcto. Campos clave: Logon Type, Account Name, Source Network Address, Logon ID.
- 4625: inicio de sesión fallido. Campo Status/Sub Status: `0xC000006A` contraseña incorrecta, `0xC0000064` el usuario no existe, `0xC0000234` cuenta bloqueada.
- 4634: cierre de sesión. Se une con su 4624 por el Logon ID para saber cuánto duró la sesión.
- 4672: privilegios especiales asignados al nuevo inicio de sesión; aparece justo después del 4624 de una cuenta administradora.
- 4688: se creó un proceso. Campos: proceso nuevo, proceso padre, línea de comandos.
- 4720: se creó una cuenta de usuario.
- 4732: se añadió un miembro a un grupo local con seguridad habilitada (por ejemplo, Administradores).
- 1102: se borró el log de auditoría (Security). Rara vez es legítimo.
- 7045: se instaló un servicio nuevo. Está en el canal System, no en Security. Es una forma común de persistencia y de ejecución remota (PsExec instala un servicio).

Los tipos de logon del 4624 y el 4625 dicen cómo entró la persona:

```
Tipo 2  Interactive          teclado y pantalla del propio equipo
Tipo 3  Network              acceso por red: carpeta compartida SMB, net use, muchas autenticaciones remotas
Tipo 10 RemoteInteractive    Escritorio remoto (RDP)
(otros comunes: 4 batch/tarea programada, 5 servicio, 7 desbloqueo, 11 credenciales en caché)
```

Con Network Level Authentication activado, un RDP con contraseña incorrecta se registra como 4625 de tipo 3, no de tipo 10, porque la autenticación ocurre antes de abrir la sesión gráfica.

Ejemplo concreto: buscar los logins fallidos de la última hora en un equipo de laboratorio.

```
PS> Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddHours(-1)} |
      Select-Object TimeCreated, @{n='Usuario';e={$_.Properties[5].Value}}, @{n='IP';e={$_.Properties[19].Value}}

TimeCreated            Usuario        IP
-----------            -------        --
02/10/2026 03:12:44    administrador  203.0.113.45
02/10/2026 03:12:43    admin          203.0.113.45
02/10/2026 03:12:43    soporte        203.0.113.45
```

La historia completa de un ataque típico se lee como una secuencia de IDs:

```
03:12  4625 ×300  tipo 3   desde 203.0.113.45   fuerza bruta
03:20  4624       tipo 10  soporte desde 203.0.113.45   entró por RDP
03:20  4672                privilegios especiales: soporte es admin
03:22  4720                se crea la cuenta "svc_backup"
03:22  4732                svc_backup añadida a Administradores
03:25  7045                servicio nuevo "UpdaterSvc" → C:\Users\Public\u.exe
03:40  1102                se borra el log de Security
```

Sysmon (System Monitor, de Sysinternals) es un servicio gratuito de Microsoft que amplía estos registros: evento 1 creación de proceso con hashes, 3 conexión de red, 11 archivo creado, 22 consulta DNS. Es la mejora más barata que se puede hacer a la visibilidad de un equipo Windows.

```
4624 → login correcto; 4625 → login fallido; 4634 → logoff.
4672 → login con privilegios de admin.
4688 → proceso creado.
4720 → cuenta creada; 4732 → añadida a grupo local.
1102 → log de seguridad borrado; 7045 → servicio instalado (canal System).
Tipo 2 → local; tipo 3 → red; tipo 10 → RDP.
```

### syslogs

**Syslog** es un estándar de mensajes de registro que usan Linux, Unix y casi todos los equipos de red (routers, switches, firewalls) para escribir eventos con una prioridad y enviarlos, si se quiere, a un servidor central.

Existe porque cada programa necesitaba una forma común de decir "esto pasó y es así de grave" sin inventar su propio formato, y porque los equipos de red no tienen disco donde guardar meses de logs: los mandan por la red a un colector.

Analogía: es el correo interno de una empresa con sobres de colores. El departamento que envía (facility) va escrito en el remite y el color del sobre indica la urgencia (severity); el que reparte sabe qué abrir primero sin leer el contenido.

Cómo funciona por dentro. Cada mensaje lleva una prioridad PRI entre `<` y `>` que combina dos números:

```
PRI = facility × 8 + severity
<34>  →  34 = 4 × 8 + 2  →  facility 4 (auth), severity 2 (Critical)
```

Las 8 severidades de RFC 5424, en orden (menor número, más grave):

```
0 Emergency      el sistema no se puede usar
1 Alert          hay que actuar de inmediato
2 Critical       condición crítica
3 Error          error
4 Warning        advertencia
5 Notice         normal pero significativo
6 Informational  informativo
7 Debug          depuración
```

Las facilities (24, del 0 al 23) dicen qué parte del sistema habla. Las que más ve un analista: 0 kern (núcleo), 1 user, 2 mail, 3 daemon (servicios), 4 auth y 10 authpriv (autenticación: aquí aparecen los logins por SSH y los `sudo`), 9 cron, y 16-23 local0-local7, libres para que cada equipo o aplicación las use (los firewalls y switches suelen enviar por una de ellas).

Formato RFC 5424 (el actual; el antiguo es RFC 3164, el "BSD syslog", todavía muy común):

```
<34>1 2026-10-02T03:12:44.003Z srv-web01 sshd 2211 - - Failed password for invalid user admin from 203.0.113.45 port 51122 ssh2
 │  │ │                        │         │    │    │ │ └ mensaje
 │  │ │                        │         │    │    │ └ datos estructurados (- = ninguno)
 │  │ │                        │         │    │    └ MSGID
 │  │ │                        │         │    └ PID
 │  │ │                        │         └ aplicación
 │  │ │                        └ equipo
 │  │ └ marca de tiempo con zona horaria
 │  └ versión del formato
 └ PRI
```

Transporte: UDP 514 tradicional (rápido, pero sin confirmación: si se pierde, se perdió), TCP 514 o 601, y TLS en el 6514 (RFC 5425) cuando los logs cruzan redes no confiables.

Dónde mirar en Linux: `/var/log/auth.log` (Debian/Ubuntu) o `/var/log/secure` (RHEL) para autenticación; en distribuciones solo con systemd, el journal:

```
$ journalctl -u sshd -p warning --since "1 hour ago"
oct 02 03:12:44 srv-web01 sshd[2211]: Failed password for invalid user admin from 203.0.113.45 port 51122 ssh2
oct 02 03:12:46 srv-web01 sshd[2213]: Failed password for root from 203.0.113.45 port 51130 ssh2
```

`-p warning` muestra severidad 4 y todo lo más grave (0 a 4).

```
Facility → quién habla (auth, kern, cron, local0-7).
Severity → qué tan grave (0 Emergency ... 7 Debug).
PRI = facility × 8 + severity.
RFC 3164 → syslog BSD antiguo; RFC 5424 → formato actual con versión y zona horaria.
UDP 514 → sin confirmación; TLS 6514 → cifrado y fiable.
```

### netflow

**NetFlow** es un protocolo de telemetría de red, creado por Cisco, con el que un router, switch o firewall resume cada conversación que pasa por él en un registro de flujo (quién habló con quién, por qué puerto, cuánto y cuándo) y lo exporta a un colector, sin guardar el contenido.

Existe porque guardar todos los paquetes de una red es carísimo (un enlace de 1 Gbit/s lleno produce más de 10 TB al día), mientras que los flujos de ese mismo enlace ocupan del orden de cientos de veces menos. Con flujos se pueden conservar meses de historia y responder preguntas como "¿este equipo habló alguna vez con esa IP?" o "¿quién sacó 40 GB anoche?".

Analogía: es la factura detallada del teléfono. Dice a qué número llamaste, a qué hora, cuántos minutos duró la llamada; no dice nada de lo que se habló.

Cómo funciona:

- Un flujo es la secuencia de paquetes que comparten la tupla de 5 (más la interfaz y el tipo de servicio) en una ventana de tiempo. El equipo lo mantiene en una caché y lo exporta cuando la conexión termina (FIN o RST), cuando lleva inactivo un tiempo (típico 15 segundos) o cuando lleva activo demasiado (típico 30 minutos, para no esperar a que termine una descarga eterna).
- Versiones: NetFlow v5 (campos fijos, solo IPv4), NetFlow v9 (RFC 3954, basado en plantillas, admite IPv6) e IPFIX (RFC 7011, el estándar de la IETF derivado de v9, puerto 4739). sFlow es un primo que muestrea paquetes en lugar de agregar flujos. El puerto que suelen usar los colectores NetFlow es UDP 2055, aunque es configurable.
- Muestreo: en enlaces muy cargados se exporta 1 de cada N paquetes (por ejemplo 1:1000), lo que basta para estadísticas pero puede perder conexiones cortas.

Qué guarda un registro de flujo: IP y puerto de origen y destino, protocolo, número de paquetes y de bytes, hora de inicio y fin, flags TCP acumulados (SYN, ACK, FIN, RST vistos en el flujo), tipo de servicio, interfaces de entrada y salida y, en routers de borde, sistemas autónomos y siguiente salto.

Qué no guarda: el payload. Ni la URL, ni el nombre de usuario, ni el archivo, ni el contenido del correo. Tampoco el nombre del dominio (para eso están los logs DNS o del proxy).

Ejemplo concreto con `nfdump` sobre los flujos de una noche, ordenados por bytes:

```
$ nfdump -R /var/cache/nfdump -s ip/bytes -n 3 'src net 10.0.0.0/8'
Top 3 Src IP Addr ordered by bytes:
Date first seen          Duration  Proto  Src IP Addr    Flows  Packets   Bytes
2026-10-02 01:00:12.410  10780.2   any    10.0.5.44         3    31.2 M   41.8 G
2026-10-02 00:00:03.002  14399.9   any    10.0.2.10      4120     2.1 M    1.9 G
2026-10-02 00:00:07.551  14390.1   any    10.0.3.18       988   620.4 k  480.2 M
```

`10.0.5.44` envió casi 42 GB en 3 flujos durante tres horas de madrugada. Los flujos no dicen qué envió, pero sí a quién: el siguiente paso es filtrar por esa IP origen, ver el destino y contrastarlo con los logs del proxy y del EDR. Otros patrones que salen solo con flujos: un equipo que contacta 1.000 IPs internas en el puerto 445 en un minuto (escaneo o gusano), o conexiones pequeñas a la misma IP externa cada 60 segundos exactos (beaconing de un malware que llama a su servidor de control).

```
NetFlow v5 → campos fijos, IPv4.
NetFlow v9 → plantillas, IPv6 (RFC 3954).
IPFIX → estándar IETF basado en v9 (RFC 7011, puerto 4739).
sFlow → muestreo de paquetes, no flujos agregados.
Flujo → quién, con quién, cuánto, cuándo; nunca qué se dijo.
```

### Packet Captures

Una **captura de paquetes (packet capture, pcap)** es un archivo que guarda copia completa de cada paquete que pasó por una interfaz de red, cabeceras y contenido incluidos, con la hora exacta en que se vio.

Existe porque es la única fuente que muestra exactamente qué se dijo en la red: el flujo dice que hubo una conversación de 2 MB, la captura muestra la petición HTTP, el archivo descargado y la respuesta del servidor. Es la prueba de más detalle y la más cara de guardar, así que se usa para investigar algo concreto, no para archivar todo para siempre.

Analogía: si NetFlow es la factura del teléfono, el pcap es la grabación de la llamada.

Cómo funciona:

- Formatos: `.pcap` (el clásico de libpcap) y `.pcapng` (el actual de Wireshark, admite varias interfaces y comentarios en el archivo).
- Captura: `tcpdump` en servidores y sensores, Wireshark o `dumpcap` en escritorio. El filtro de captura usa la sintaxis BPF (`host 10.0.5.44 and port 80`) y decide qué se guarda; lo que no pasa el filtro se pierde.
- Análisis: Wireshark disecciona cada protocolo en campos y permite filtrar con filtros de visualización, que no borran nada, solo esconden lo que no coincide. Sintaxis distinta a la de BPF.
- El tráfico cifrado con TLS se ve como cifrado; sin las claves de sesión solo quedan metadatos (handshake, SNI, certificado).

Filtros de visualización de Wireshark que más se usan:

```
ip.addr == 10.0.5.44                         todo lo que va o viene de esa IP
tcp.port == 443                              origen o destino 443
http.request.method == "POST"                envíos de formularios y subidas
dns.qry.name contains "update"               consultas DNS con esa cadena
tcp.flags.syn == 1 && tcp.flags.ack == 0     intentos de conexión (detecta escaneos)
tls.handshake.extensions_server_name         muestra el SNI de cada conexión TLS
!(arp || dns || icmp)                        esconder ruido para ver el resto
```

Herramientas de Wireshark para un analista: Statistics → Conversations (quién habló con quién y cuánto), Statistics → Protocol Hierarchy (qué protocolos hay), Follow → TCP Stream (reconstruye la conversación entera como texto) y File → Export Objects → HTTP (extrae los archivos descargados por HTTP para calcular su hash).

Ejemplo concreto en un laboratorio propio: capturar mientras se descarga una página y luego filtrar las peticiones.

```
$ sudo tcpdump -i eth0 -w lab.pcap host 192.168.56.20 and port 80
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C 48 packets captured

$ tshark -r lab.pcap -Y 'http.request' -T fields -e ip.src -e http.host -e http.request.uri
192.168.56.10   192.168.56.20   /hola-ids
192.168.56.10   192.168.56.20   /favicon.ico
```

> [!WARNING]
> Un pcap contiene todo lo que pasó en claro: contraseñas, cookies de sesión, datos personales. Se trata como dato sensible, se guarda cifrado y no se sube a ningún servicio público.

```
pcap → copia completa de paquetes: cabeceras y contenido.
pcapng → formato actual de Wireshark, varias interfaces y comentarios.
Filtro de captura (BPF) → decide qué se guarda; lo demás se pierde.
Filtro de visualización → decide qué se muestra; no borra nada.
Follow TCP Stream → reconstruye la conversación completa.
```

### Firewall Logs

Los **logs de firewall** son los registros que escribe un firewall por cada conexión que permite (allow/accept) o rechaza (deny/drop/reject), con la tupla de 5, la interfaz, la regla que decidió y la hora.

Existen porque el firewall está en el punto de paso entre zonas de la red y es el primer lugar donde queda constancia de un escaneo, de una conexión saliente a un destino raro o de un servicio expuesto por error. Los dos tipos de línea cuentan cosas distintas: un deny dice qué se intentó y no pasó; un allow dice qué pasó realmente, y es el que se busca cuando se investiga qué hizo un equipo comprometido.

Analogía: es el registro de la caseta de vigilancia de un condominio: placa, hora, a qué casa iba y si se le abrió la barrera o no.

Cómo se lee una línea. Ejemplo de un firewall de Linux (UFW, que escribe a través de netfilter en el log del kernel):

```
Oct  2 03:11:02 gw kernel: [UFW BLOCK] IN=eth0 OUT= MAC=52:54:00:12:34:56:52:54:00:ab:cd:ef:08:00 SRC=203.0.113.45 DST=192.0.2.10 LEN=60 TOS=0x00 PREC=0x00 TTL=49 ID=54321 DF PROTO=TCP SPT=51122 DPT=3389 WINDOW=64240 RES=0x00 SYN URGP=0
```

- `[UFW BLOCK]`: la acción (se rechazó). `[UFW ALLOW]` sería una conexión permitida y registrada.
- `IN=eth0 OUT=`: entró por `eth0` y no salió por ninguna: iba al propio firewall o se descartó antes de reenviarse.
- `SRC` / `DST`: IP de origen y de destino.
- `PROTO=TCP SPT=51122 DPT=3389`: TCP desde el puerto 51122 hacia el 3389 (RDP).
- `SYN`: era un primer paquete de conexión. Un SYN rechazado hacia un puerto es un intento de conexión, típico de un escaneo.
- `TTL=49`: el paquete hizo unos 15 saltos (empezó probablemente en 64), coherente con un origen lejano en Internet.

Los firewalls comerciales escriben los mismos datos con otro formato (por ejemplo `action=deny srcip=203.0.113.45 dstip=192.0.2.10 dstport=3389 policyid=12`), y el SIEM los normaliza a los mismos campos.

Ejemplo concreto de lectura en conjunto: en 60 segundos aparecen 1.000 líneas BLOCK desde `203.0.113.45` hacia `192.0.2.10`, cada una con un `DPT` distinto del 1 al 1000 y todas con `SYN`: es un escaneo de puertos. Si luego aparece una línea ALLOW de esa IP hacia el puerto 443, ese es el único servicio que encontró abierto, y el siguiente lugar a mirar es el log del servidor web.

```
Allow/accept → la conexión pasó; muestra lo que realmente ocurrió.
Deny/drop → se descartó en silencio; el origen no recibe respuesta.
Reject → se rechazó avisando (TCP RST o ICMP); el origen sabe que está cerrado.
SYN rechazado hacia muchos puertos → escaneo.
```

### Lo que no se registró no se puede investigar

> [!IMPORTANT]
> Los logs hay que activarlos, centralizarlos y guardarlos antes del incidente. Durante el incidente ya solo se puede leer lo que se guardó, y un atacante con privilegios borra lo que quedó solo en el equipo.

La visibilidad es una propiedad que se construye por adelantado: cada fuente aporta una pieza distinta del mismo suceso y ninguna sustituye a las otras.

```
                     Event Logs      syslog         NetFlow        pcap           Firewall
Responde             quién entró     qué servicio   quién habló    qué se dijo    qué pasó o no
                     y qué ejecutó   hizo qué       con quién      exactamente    la frontera
Contenido            no              parcial        no             sí             no
Coste de guardar     bajo            bajo           bajo           muy alto       bajo
Hay que activarlo    sí (auditoría)  sí (envío)     sí (exportar)  sí (sensor)    sí (log por regla)
```

La última fila es igual en todas las columnas: ninguna fuente existe por defecto con el detalle y la retención necesarios.

Ejemplo: durante un incidente se descubre que el atacante entró hace 45 días. Si el SIEM retiene 30, la entrada original ya no existe en ningún lado; si además el atacante lanzó un 1102 en el equipo, la copia local tampoco. Por eso las guías (NIST SP 800-92) piden enviar los logs fuera del equipo en tiempo casi real, sincronizar relojes por NTP y definir la retención antes de necesitarla.

```
Centralizar → la copia en el SIEM sobrevive al borrado local.
Retención → si es menor que el tiempo de permanencia del atacante, se pierde la entrada.
NTP → sin relojes sincronizados, la línea de tiempo no cuadra.
```

## Understand Common Tools

Todas estas herramientas sirven para enriquecer un IOC: tomar un hash, una URL, un dominio o una IP de una alerta y averiguar qué sabe el mundo de él. Antes de cada una conviene saber una regla general: casi todos los servicios gratuitos comparten lo que se les envía con otros usuarios o con clientes de pago.

### VirusTotal

**VirusTotal** es un servicio en línea de Google que analiza archivos, URLs, dominios e IPs con unos 70 motores antivirus y de reputación a la vez y guarda el historial de todo lo que se le ha enviado.

Existe porque ningún antivirus solo ve todo, y porque al analista le interesa tanto el veredicto como el contexto: cuándo se vio por primera vez ese archivo, con qué nombres circula, a qué dominios se conecta cuando se ejecuta.

Analogía: es pedir la opinión de 70 médicos a la vez y además consultar el historial clínico del paciente.

Qué subir y qué no:

- Buscar primero por hash (SHA-256), no subir el archivo. Si el hash ya existe, se obtiene el informe completo sin entregar nada.
- No subir documentos internos (contratos, nóminas, un Excel con datos de clientes), aunque parezcan sospechosos: los archivos subidos quedan disponibles para descarga por los clientes de pago de VirusTotal, entre ellos investigadores de todo el mundo y también atacantes. Lo mismo con URLs que llevan tokens o datos personales (enlaces de restablecer contraseña, enlaces de documentos compartidos).
- Subir un malware propio de un ataque dirigido avisa al atacante de que se le detectó: él también puede vigilar si su muestra aparece.

Cómo leer el resultado:

```
SHA-256  9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
Detection: 48/72 security vendors flagged this file as malicious
Names: factura_octubre.pdf.exe, invoice.exe
First submission: 2026-09-28 06:14:02 UTC
Behavior: contacts hxxp://update-check[.]example/gate.php
```

- El ratio `48/72`: muchos motores de acuerdo es fuerte indicio. `1/72` o `2/72` suele ser un falso positivo de un motor heurístico, sobre todo en herramientas de administración legítimas. `0/72` no prueba que sea seguro: un malware nuevo o dirigido puede tener cero detecciones.
- First submission: un archivo "conocido" desde hace 3 años es distinto a uno visto por primera vez hace 2 horas.
- Pestañas Relations y Behavior: dominios, IPs y archivos relacionados, que son nuevos IOCs para buscar en el SIEM.

```
Buscar por hash → consulta sin entregar el archivo.
Subir archivo → lo comparte con clientes de pago; nunca documentos internos.
0/72 → no prueba que sea seguro; 1-2/72 → posible falso positivo.
```

### urlscan

**urlscan.io** es un servicio en línea que visita una URL con un navegador automático, toma una captura de pantalla y registra todo lo que la página carga: dominios, IPs, certificados, scripts y redirecciones.

Existe porque abrir una URL sospechosa en el propio navegador es justo lo que el atacante quiere. urlscan la abre por ti, desde su infraestructura, y te enseña lo que habrías visto.

Analogía: es mandar a un mensajero a mirar por la ventana de una casa sospechosa y que vuelva con una foto y la lista de todo lo que había dentro, sin que tú te acerques.

Qué subir y qué no:

- Cada escaneo tiene una visibilidad: Public (cualquiera lo ve y aparece en las búsquedas), Unlisted (no aparece en búsquedas públicas, pero sí lo ven investigadores de seguridad con acceso) y Private (solo tú; requiere cuenta y depende del plan).
- No escanear en público URLs con datos de la organización o personales: enlaces con tokens, restablecimiento de contraseña, documentos compartidos o URLs con el correo de un empleado como parámetro (`?email=ana@empresa.com`). Escanearla en público publica ese dato y, a veces, el atacante sabe así a quién se le escapó la campaña.
- Antes de escanear, buscar si ya existe un escaneo de ese dominio.

Cómo leer el resultado: la captura de pantalla (¿imita el login de un banco o de Microsoft 365?), el dominio final tras las redirecciones, la IP y el país del servidor, la edad del certificado TLS, los dominios contactados y el veredicto (malicious/no classification). Una página idéntica al login de Microsoft servida desde un dominio registrado hace 2 días es phishing aunque el veredicto diga "No classification".

```
urlscan → visita la URL por ti; captura, recursos, redirecciones.
Public / Unlisted / Private → todos / investigadores / solo tú.
```

### any.run

**ANY.RUN** es una sandbox interactiva en línea que ejecuta un archivo o abre una URL en una máquina virtual (normalmente Windows) y muestra en tiempo real los procesos, la actividad de red y los cambios en el sistema, dejando que el analista interactúe con la máquina.

Existe porque mucho malware no hace nada hasta que alguien pulsa "Habilitar contenido" en un documento de Office, resuelve un captcha o espera unos minutos; una sandbox automática sin interacción lo ve inofensivo. En ANY.RUN el analista puede hacer esos clics.

Analogía: es el laboratorio de pruebas de una bomba con un brazo robótico que tú manejas: puedes abrir el paquete, cortar el cable rojo y ver qué pasa desde detrás del cristal.

Qué subir y qué no:

- En el plan gratuito (Community) todos los análisis son públicos: cualquiera puede ver el informe y descargar la muestra. Lo mismo que en VirusTotal: nada interno, nada con datos personales, y cuidado con muestras de un ataque dirigido.
- Si el adjunto viene en un correo real de un empleado, el documento puede contener su nombre o datos de clientes.

Cómo leer el resultado: el árbol de procesos (por ejemplo `WINWORD.EXE → cmd.exe → powershell.exe -enc ...` es una cadena clásica de documento malicioso), las conexiones HTTP y DNS, los archivos creados, las técnicas mapeadas a MITRE ATT&CK y el veredicto (malicious, suspicious, no threats detected). Los IOCs de red que salen aquí se buscan después en el SIEM y en los logs del proxy.

```
any.run → sandbox interactiva: el analista hace clics dentro.
Plan Community → análisis públicos; nada interno.
```

### Joe Sandbox

**Joe Sandbox** es una plataforma de análisis de malware de la empresa Joe Security que ejecuta archivos y URLs en sandboxes automáticas para Windows, Linux, macOS y Android y genera un informe muy detallado con una puntuación de riesgo.

Existe para el análisis en profundidad y en lote: frente a ANY.RUN, que brilla en la interacción manual, Joe Sandbox se centra en un informe automático exhaustivo (comportamiento, red, firmas, análisis estático, técnicas ATT&CK) y en ofrecer versiones on-premise para organizaciones que no pueden enviar nada a la nube.

Analogía: si ANY.RUN es manejar tú el brazo robótico, Joe Sandbox es dejar el paquete en un laboratorio que hace 200 pruebas automáticas y te devuelve un informe forense largo.

Qué subir y qué no: la versión gratuita en la nube (Joe Sandbox Cloud Basic) publica los informes. Para muestras con datos internos se usa una versión de pago con análisis privado o la instalación on-premise. Igual que con las demás: buscar primero si ya existe un informe del mismo hash.

Cómo leer el resultado: la clasificación final (malicious, suspicious, clean) con su puntuación, las firmas de comportamiento que dispararon agrupadas por categoría (persistencia, evasión, robo de credenciales), el árbol de procesos, los dominios y las IPs contactados y los archivos que soltó. Las firmas son lo que justifica el veredicto: un "malicious" apoyado en "crea un servicio, desactiva Defender y contacta IP sin dominio" pesa más que uno apoyado en una sola heurística.

```
Joe Sandbox → informe automático exhaustivo, varios sistemas operativos, opción on-premise.
any.run → interacción manual en tiempo real.
Cloud Basic → gratis, informes públicos.
```

### urlvoid

**URLVoid** es un servicio en línea que consulta un dominio contra varias decenas de listas de bloqueo y motores de reputación y resume cuántos lo marcan como peligroso, junto con datos del dominio y de su servidor.

Existe para dar una respuesta rápida a "¿este dominio tiene mala fama?" sin visitarlo, consultando de una vez listas que habría que revisar una a una.

Analogía: es preguntar en el barrio por un vecino nuevo: cuántos de los porteros de la zona lo tienen en su lista de gente problemática.

Qué subir y qué no: se le da un dominio, no una URL completa. Así no se envía la ruta ni los parámetros (que pueden contener tokens o correos) y se obtiene lo único que evalúa, la reputación del dominio.

Cómo leer el resultado: el ratio de detecciones (`3/40` por ejemplo), la fecha de registro del dominio, la IP y el país del servidor. Igual que en VirusTotal, `0` detecciones no significa seguro: un dominio de phishing creado esta mañana todavía no está en ninguna lista, y lo que lo delata es su edad.

```
urlvoid → reputación de un dominio en listas de bloqueo; se le da el dominio, no la URL.
0 detecciones + dominio de 2 días → sospechoso igual.
```

### WHOIS

**WHOIS** es un protocolo y un servicio de consulta que devuelve los datos de registro de un dominio o de un bloque de direcciones IP: quién lo registró, con qué registrador, cuándo se creó y cuándo vence, y qué servidores DNS usa.

Existe porque los dominios y las IPs se reparten a través de registros públicos (ICANN, los registros de cada TLD, los RIR como ARIN, RIPE o LACNIC), y esos datos son la forma de saber quién es responsable de un recurso en Internet. Para dominios genéricos, ICANN sustituyó oficialmente WHOIS por RDAP (Registration Data Access Protocol, respuestas en JSON) a principios de 2025; la consulta se sigue llamando "whois" en el día a día, y lookup.icann.org ya usa RDAP.

Analogía: es el registro de la propiedad de una ciudad: dice cuándo se compró la casa y por medio de qué notaría, aunque desde las leyes de privacidad muchas veces no dice el nombre del dueño.

Qué subir y qué no: WHOIS solo recibe un dominio o una IP, datos que ya son públicos, así que el riesgo de privacidad es bajo. Lo que sí hay que saber es que los datos del titular suelen estar ocultos (REDACTED FOR PRIVACY) por el RGPD europeo o por servicios de privacidad, así que el nombre del titular rara vez sirve.

Cómo leer el resultado:

```
$ whois ejemplo-sospechoso.com
Domain Name: EJEMPLO-SOSPECHOSO.COM
Registrar: NameCheap, Inc.
Creation Date: 2026-09-30T11:02:17Z
Registry Expiry Date: 2027-09-30T11:02:17Z
Name Server: NS1.EXAMPLE-DNS.NET
Registrant Organization: REDACTED FOR PRIVACY
```

- Creation Date: lo más útil. Un dominio creado hace 2 días que manda correos de "su banco" es casi seguro phishing; los bancos no estrenan dominio para escribir a sus clientes.
- Expiry Date a un año exacto: típico de dominios desechables comprados al mínimo.
- Name Server y Registrar: permiten agrupar dominios de una misma campaña.
- Para una IP (`whois 203.0.113.45`), el resultado dice a qué organización y país está asignado el bloque y el contacto de abuso (`abuse@...`) para reportar.

```
WHOIS → datos de registro de un dominio o IP; antiguo, texto libre.
RDAP → su reemplazo estructurado en JSON (lookup.icann.org).
Creation Date reciente → señal fuerte de phishing.
REDACTED FOR PRIVACY → titular oculto por RGPD; normal, no sospechoso por sí solo.
```

### Subir una muestra a un servicio público es publicarla

> [!IMPORTANT]
> Enviar un archivo o una URL a un servicio gratuito equivale a compartirlo con desconocidos, incluido el atacante. Primero se busca por hash o por dominio; solo se sube si no contiene datos propios y si avisar al atacante no importa.

La privacidad de un análisis es una propiedad del servicio y del plan, no de la intención del analista, y se decide antes de pulsar "enviar".

```
                    VirusTotal        urlscan            any.run (free)  Joe Sandbox Basic  urlvoid / WHOIS
Qué se envía        archivo o URL     URL completa       archivo o URL   archivo o URL      dominio / IP
Quién lo ve         clientes de pago  según visibilidad  todos           todos              dato ya público
Alternativa segura  buscar por hash   Unlisted/Private   plan privado    on-premise         no hace falta
```

La fila "Quién lo ve" es la que decide: solo urlvoid y WHOIS reciben algo que ya era público.

Ejemplo: llega un adjunto `Nomina_septiembre.xlsm` a 30 empleados. Lo correcto es calcular su SHA-256 y buscarlo; si no existe, analizarlo en una sandbox privada o en una máquina aislada propia. Subirlo a VirusTotal expondría la nómina de la empresa si el archivo es real, y avisaría al atacante si es malicioso y dirigido.

```
Buscar antes que subir → hash o dominio primero.
Dato interno → sandbox privada u on-premise, nunca un servicio gratuito.
Defang → compartir IOCs como hxxp://malo[.]com.
```

## Recursos para aprender y practicar

### Videos

- [Intrusion Prevention - CompTIA Security+ SY0-701 - 3.2](https://www.youtube.com/watch?v=7QuYupuic3Q) — Professor Messer; IDS frente a IPS, inline frente a pasivo, firmas y anomalías. Nodos: Basics of IDS and IPS, NIDS, NIPS.
- [Snort 3 - Rule Writing (with labs)](https://www.youtube.com/watch?v=CystKHV2gnI) — Cisco Talos Intelligence Group; anatomía de una regla de Snort 3 y cómo probarla. Nodo: Basics of IDS and IPS.
- [What Is SIEM?](https://www.youtube.com/watch?v=9RfsRn7m7OE) — IBM Technology; recolección, normalización y correlación en un SIEM. Nodo: SIEM.
- [What is SOAR (Security, Orchestration, Automation & Response)](https://www.youtube.com/watch?v=k7ju95jDxFA) — IBM Technology; playbooks y automatización de la respuesta. Nodo: SOAR.
- [Log Data - CompTIA Security+ SY0-701 - 4.9](https://www.youtube.com/watch?v=EDru1LTYDJw) — Professor Messer; logs de firewall, de aplicación, de endpoint, NetFlow y capturas como fuentes de investigación. Nodos: Firewall Logs, netflow, Packet Captures, syslogs.
- [RDP Event Log Forensics](https://www.youtube.com/watch?v=myzG11BP3Sk) — 13Cubed; qué eventos de Windows deja un acceso por RDP y cómo se encadenan (4624 tipo 10, 4625, 4634 y otros). Nodo: Event Logs.
- [MALWARE Analysis with Wireshark // TRICKBOT Infection](https://www.youtube.com/watch?v=Brx4cygfmg8) — Chris Greer; filtros de visualización y análisis de un pcap real de infección. Nodo: Packet Captures.
- [How to use Interactive Malware Sandbox – ANY.RUN Tutorial](https://www.youtube.com/watch?v=lyBmNALZO1w) — ANY.RUN; enviar una muestra, interactuar y leer el informe. Nodo: any.run.

### Lectura y documentación

- [NIST SP 800-94, Guide to Intrusion Detection and Prevention Systems](https://csrc.nist.gov/pubs/sp/800/94/final) — tipos de IDPS, métodos de detección y despliegue. Nodos: Basics of IDS and IPS, NIDS, NIPS.
- [NIST SP 800-92, Guide to Computer Security Log Management](https://csrc.nist.gov/pubs/sp/800/92/final) — recolección, centralización y retención de logs. Nodos: SIEM, todos los de logs.
- [NIST SP 800-61 Rev. 3, Incident Response Recommendations](https://csrc.nist.gov/pubs/sp/800/61/r3/final) — dónde encajan la detección y el análisis dentro de la respuesta a incidentes. Nodos: SIEM, SOAR.
- [Snort 3 documentation](https://docs.snort.org/) — sintaxis de reglas, opciones y modos de ejecución. Nodo: Basics of IDS and IPS.
- [Suricata: Rules Format](https://docs.suricata.io/en/latest/rules/intro.html) — cabecera, acciones y opciones de las reglas de Suricata. Nodos: Basics of IDS and IPS, NIPS.
- [Microsoft Learn: 4624(S) An account was successfully logged on](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624) — campos del evento y tabla completa de tipos de logon. Nodo: Event Logs.
- [Microsoft Learn: Appendix L, Events to Monitor](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/appendix-l--events-to-monitor) — lista de Event IDs con su criticidad. Nodo: Event Logs.
- [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) — instalación, configuración y lista de eventos de Sysmon. Nodo: Event Logs.
- [RFC 5424, The Syslog Protocol](https://www.rfc-editor.org/rfc/rfc5424) — formato actual, facilities y las 8 severidades. Nodo: syslogs.
- [RFC 3164, The BSD syslog Protocol](https://www.rfc-editor.org/rfc/rfc3164) — el formato antiguo que muchos equipos todavía envían. Nodo: syslogs.
- [RFC 3954, Cisco Systems NetFlow Services Export Version 9](https://www.rfc-editor.org/rfc/rfc3954) — plantillas y campos de NetFlow v9. Nodo: netflow.
- [RFC 7011, IPFIX Protocol Specification](https://www.rfc-editor.org/rfc/rfc7011) — el estándar IETF de exportación de flujos. Nodo: netflow.
- [Wireshark User's Guide: Building Display Filter Expressions](https://www.wireshark.org/docs/wsug_html_chunked/ChWorkBuildDisplayFilterSection.html) — sintaxis completa de filtros de visualización. Nodo: Packet Captures.
- [Wireshark Wiki: DisplayFilters](https://wiki.wireshark.org/DisplayFilters) — ejemplos de filtros listos para usar. Nodo: Packet Captures.
- [VirusTotal documentation](https://docs.virustotal.com/) — cómo leer informes y qué pasa con lo que se envía. Nodo: VirusTotal.
- [urlscan.io: About](https://urlscan.io/about/) — qué hace el escáner y las opciones de visibilidad. Nodo: urlscan.
- [ANY.RUN](https://any.run/) — planes y privacidad de los análisis. Nodo: any.run.
- [Joe Security](https://www.joesecurity.org/) — productos Joe Sandbox, versiones en la nube y on-premise. Nodo: Joe Sandbox.
- [URLVoid](https://www.urlvoid.com/) — consulta de reputación de dominios. Nodo: urlvoid.
- [ICANN Lookup](https://lookup.icann.org/) — consulta oficial de registro de dominios vía RDAP. Nodo: WHOIS.

### Práctica

- [TryHackMe: Snort](https://tryhackme.com/room/snort) — modos de Snort y escritura de reglas en una máquina de laboratorio. Nodos: Basics of IDS and IPS, NIDS, NIPS.
- [TryHackMe: Intrusion Detection](https://tryhackme.com/room/idsevasion) — gratis; cómo detecta un IDS y por qué a veces falla. Nodo: Basics of IDS and IPS.
- [TryHackMe: Intro to SIEM](https://tryhackme.com/room/introtosiem) — fuentes de logs, normalización y reglas de correlación. Nodo: SIEM.
- [TryHackMe: Splunk Basics - Did you SIEM?](https://tryhackme.com/room/splunkforloganalysis-aoc2025-x8fj2k4rqp) y [Splunk: Exploring SPL](https://tryhackme.com/room/splunkexploringspl) — gratis; primeras búsquedas en SPL. Nodo: SIEM.
- [TryHackMe: Log Analysis with SIEM](https://tryhackme.com/room/loganalysiswithsiem) — gratis; investigar alertas correlacionando logs. Nodos: SIEM, Event Logs.
- [TryHackMe: Windows Logging for SOC](https://tryhackme.com/room/windowsloggingforsoc) e [Intro to Logs](https://tryhackme.com/room/introtologs) — gratis; eventos de Windows, Sysmon y tipos de logs. Nodo: Event Logs.
- [TryHackMe: Carnage](https://tryhackme.com/room/c2carnage) y [Traffic Analysis Essentials](https://tryhackme.com/room/trafficanalysisessentials) — gratis; investigar un pcap real en Wireshark. Nodo: Packet Captures. Más pcaps para practicar: [Malware-Traffic-Analysis.net](https://www.malware-traffic-analysis.net/training-exercises.html).
- [Splunk BOTS v1](https://github.com/splunk/botsv1) y [BOTS v3](https://github.com/splunk/botsv3) — conjuntos de datos reales de ataque (Windows, firewall, flujos, Sysmon) para cargar en un Splunk propio y cazar. Nodos: SIEM, Event Logs, Firewall Logs.
- [CyberDefenders](https://cyberdefenders.org/) — retos azules con pcaps, logs y memoria. Nodos: Packet Captures, Event Logs, SIEM.
- [Blue Team Labs Online](https://blueteamlabs.online/) — investigaciones de phishing, logs y tráfico. Nodos: herramientas comunes, Packet Captures.
- [LetsDefend](https://letsdefend.io/) — simulador de SOC con cola de alertas, SIEM y casos. Nodos: SIEM, SOAR, herramientas comunes.
- Ejercicio en casa: en dos máquinas virtuales de VirtualBox en red solo-anfitrión, instala Suricata en una, carga las dos reglas de esta nota y genera el tráfico con `ping` y `curl` desde la otra; luego cambia `alert` por `drop` y ejecuta Suricata en modo IPS con NFQUEUE para ver el bloqueo. Nodos: Basics of IDS and IPS, NIPS.
- Ejercicio en casa: en una VM Windows de evaluación activa la auditoría de creación de procesos con línea de comandos, instala Sysmon, falla 5 veces el login, crea un usuario y añádelo a Administradores; después encuentra 4625, 4720, 4732 y 4688 con `Get-WinEvent`. Nodo: Event Logs.
- Ejercicio en casa: con `logger -p auth.warning "prueba de syslog"` escribe un mensaje y encuéntralo con `journalctl -p warning`; calcula a mano su PRI. Nodo: syslogs.
- Ejercicio en casa: captura con `tcpdump` tu propia navegación durante 5 minutos, ábrela en Wireshark y responde con filtros: cuántos dominios distintos consultaste por DNS y a qué SNI fue cada conexión TLS. Nodo: Packet Captures.
- Ejercicio en casa: busca en VirusTotal (por hash, sin subir nada) el SHA-256 de un archivo de prueba EICAR y lee su informe; luego consulta en lookup.icann.org la fecha de creación de tres dominios que conozcas. Nodos: VirusTotal, WHOIS.

## Cuadro resumen

Todo lo visto, en una línea por término.

Detección de intrusiones

```
IDS → detecta y alerta; pasivo, ve una copia; no bloquea.
IPS → detecta y bloquea; inline, el tráfico lo atraviesa.
Firma → patrón conocido; preciso, ciego ante lo nuevo.
Anomalía → desvío de la línea base; ve lo nuevo, más falsos positivos.
Fail-open / fail-closed → si el IPS cae, deja pasar todo / corta todo.
Falso positivo en IPS → bloquea a un usuario legítimo; reglas nuevas primero en modo alerta.
Snort / Suricata → motores libres; regla = acción proto origen -> destino (opciones; sid; rev).
sid ≥ 1.000.000 → reservado para reglas locales.
NIDS → sensor en la red; ve muchos equipos; no ve dentro del host ni el contenido cifrado.
SPAN → copia del switch; barato, puede perder paquetes. TAP → copia física, completa.
NIPS → bloquea en la red; inline; drop / reject.
HIPS → bloquea en el equipo (ver 14-defensa-y-hardening).
NGFW → firewall + IPS + control de aplicaciones en una caja.
Posición → decide si el sensor ve y si puede bloquear.
Tráfico este-oeste → entre equipos internos; el sensor del perímetro no lo ve.
```

Understand the following

```
SIEM → recoge, normaliza, correlaciona, alerta, busca y retiene.
Recolección → traer los logs de cada fuente al SIEM.
Normalización → poner nombres de campo comunes a formatos distintos.
Correlación → unir eventos de varias fuentes en una ventana de tiempo.
SPL → lenguaje de búsqueda de Splunk: index=... | stats count by campo | where ...
SOAR → responde: ejecuta playbooks sobre las alertas del SIEM.
Orquestación → conectar herramientas entre sí.
Automatización → ejecutar pasos sin intervención humana; acciones destructivas con aprobación.
Playbook → flujo de decisiones para un tipo de alerta.
```

Learn how to find and use these logs

```
Event Logs → .evtx en C:\Windows\System32\winevt\Logs; Security, System, Sysmon.
4624 → login correcto; 4625 → login fallido; 4634 → logoff.
4672 → login con privilegios de admin.
4688 → proceso creado (requiere activar auditoría y línea de comandos).
4720 → cuenta creada; 4732 → añadida a grupo local.
1102 → log de seguridad borrado; 7045 → servicio instalado (canal System).
Tipo 2 → local; tipo 3 → red; tipo 10 → RDP.
Facility → quién habla (auth, kern, cron, local0-7).
Severity → 0 Emergency, 1 Alert, 2 Critical, 3 Error, 4 Warning, 5 Notice, 6 Informational, 7 Debug.
PRI = facility × 8 + severity.
RFC 3164 → syslog BSD antiguo; RFC 5424 → formato actual.
UDP 514 → sin confirmación; TLS 6514 → cifrado y fiable.
NetFlow → quién, con quién, cuánto, cuándo; nunca el payload.
NetFlow v5 / v9 / IPFIX → campos fijos / plantillas (RFC 3954) / estándar IETF (RFC 7011).
pcap → copia completa de paquetes: cabeceras y contenido; dato sensible.
Filtro de captura (BPF) → qué se guarda. Filtro de visualización → qué se muestra.
Follow TCP Stream → reconstruye la conversación completa.
Allow → la conexión pasó; deny/drop → descartada en silencio; reject → rechazada avisando.
SYN rechazado hacia muchos puertos → escaneo.
Centralizar → la copia en el SIEM sobrevive al borrado local.
Retención → si es menor que la permanencia del atacante, se pierde la entrada.
NTP → sin relojes sincronizados, la línea de tiempo no cuadra.
```

Understand Common Tools

```
VirusTotal → unos 70 motores; buscar por hash; subir comparte con clientes de pago.
0/72 → no prueba que sea seguro; 1-2/72 → posible falso positivo.
urlscan → visita la URL por ti; Public / Unlisted / Private.
any.run → sandbox interactiva; plan Community con análisis públicos.
Joe Sandbox → informe automático exhaustivo; Cloud Basic público; on-premise para lo privado.
urlvoid → reputación de un dominio en listas de bloqueo; se le da el dominio, no la URL.
WHOIS → datos de registro de un dominio o IP; Creation Date reciente → señal de phishing.
RDAP → reemplazo estructurado de WHOIS (lookup.icann.org).
Buscar antes que subir → hash o dominio primero; dato interno nunca a un servicio gratuito.
Defang → compartir IOCs como hxxp://malo[.]com.
```
