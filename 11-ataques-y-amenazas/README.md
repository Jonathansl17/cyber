# Ataques y amenazas

## Conceptos previos

- Amenaza (threat): cualquier cosa que puede causar daño a un sistema; un atacante, un malware, un empleado descuidado.
- Vulnerabilidad: debilidad que una amenaza puede aprovechar (un bug, una contraseña débil, una persona confiada).
- Exploit: código o técnica que aprovecha una vulnerabilidad concreta para hacer algo que el sistema no debería permitir.
- Payload: la parte del ataque que hace el daño una vez dentro (cifrar archivos, abrir una puerta trasera, robar datos).
- Vector de ataque: el camino por el que llega el ataque (correo, USB, sitio web, llamada telefónica).
- Persistencia: mecanismo con el que un atacante sobrevive a reinicios y cierres de sesión (una tarea programada, una clave de registro).
- C2 (command and control): servidor desde el que el atacante da órdenes a las máquinas infectadas y recibe lo robado.
- IOC (indicator of compromise): rastro técnico que delata un ataque (un hash, una IP, un dominio); se desarrolla en [`13-frameworks-de-amenazas`](../13-frameworks-de-amenazas/).
- EDR (Endpoint Detection and Response): agente instalado en cada equipo que vigila procesos, archivos y red y puede aislar la máquina.
- Firma (signature): patrón fijo (un hash, una secuencia de bytes) con el que un antivirus reconoce algo ya conocido.
- Hash: huella digital de un archivo; cambia por completo si cambia un solo byte (ver [`10-criptografia`](../10-criptografia/)).
- MFA (multi-factor authentication): pedir dos o más pruebas de identidad distintas (algo que sabes, tienes o eres); ver [`08-autenticacion`](../08-autenticacion/).
- Tríada CIA: confidencialidad, integridad y disponibilidad, lo que cada ataque intenta romper (ver [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/)).

## Learn how Malware works and Types

### Cómo funciona el malware

El **malware** (malicious software) es software escrito para actuar contra los intereses de quien lo ejecuta: robar, espiar, destruir, secuestrar o usar el equipo para otros fines. Existe porque un atacante casi nunca puede estar sentado frente a la máquina de la víctima: necesita un programa que haga el trabajo por él, a distancia y a escala.

Analogía: el malware es un empleado infiltrado. Primero tiene que entrar al edificio (entrega), luego conseguir que alguien le abra una puerta (ejecución), esconder una copia de la llave para volver mañana (persistencia), llamar a su jefe para recibir órdenes (C2) y finalmente hacer el trabajo sucio (acción).

Casi todo malware moderno recorre las mismas cinco etapas, aunque cada familia pone el énfasis en una distinta:

```
 1. Entrega      correo con adjunto, enlace, USB, descarga, exploit remoto
        |
 2. Ejecución    el usuario abre el archivo, una macro corre, el exploit salta
        |
 3. Persistencia clave Run del registro, tarea programada, servicio, cron
        |
 4. C2           el equipo "llama a casa" por HTTPS/DNS cada N segundos
        |
 5. Acción       cifrar, robar credenciales, propagarse, minar, espiar
```

Para no ser detectado, el malware usa evasión: ofusca su código (lo reescribe para que no coincida con firmas), se empaqueta (comprime y cifra el ejecutable, que se descifra solo en memoria), detecta si corre en una máquina virtual o sandbox y en ese caso no hace nada, y se inyecta dentro de procesos legítimos (`explorer.exe`, `svchost.exe`) para que su actividad parezca normal.

Ejemplo: un empleado recibe `Factura_0912.docm`. Al habilitar macros, la macro lanza PowerShell, que descarga un segundo archivo (dropper → payload), crea la clave `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\Updater` y empieza a conectarse cada 60 segundos a `https://cdn-update[.]example/check`. Cada etapa deja un rastro distinto, y por eso las defensas se reparten por etapas: filtro de correo (1), bloqueo de macros (2), monitoreo de claves Run (3), proxy que bloquea dominios recién registrados (4), copias de seguridad (5).

```
Dropper → programa pequeño cuyo único trabajo es bajar o soltar el malware real.
Payload → la parte que hace el daño.
Packer → envoltorio que comprime/cifra el ejecutable para esquivar firmas.
Sandbox → entorno aislado donde se ejecuta algo sospechoso para observarlo.
```

### El malware se clasifica por cómo se propaga y por qué hace

> [!IMPORTANT]
> Los nombres de malware no son categorías excluyentes: cada muestra se describe en dos ejes, cómo llega y se propaga, y qué hace una vez dentro. WannaCry es a la vez gusano y ransomware.

La clasificación del malware es un sistema de dos ejes que separa el mecanismo de propagación (virus, gusano, troyano) del objetivo o carga (ransomware, spyware, keylogger, rootkit, botnet). Estudiantes y exámenes caen en tratar "troyano" y "ransomware" como opciones excluyentes, cuando en la práctica una muestra real combina varias etiquetas.

```
                  Virus            Gusano            Troyano
Necesita host     sí (archivo)     no                no (es el host)
Necesita usuario  sí (ejecutar)    no                sí (instalar)
Se replica solo   sí, en archivos  sí, por la red    no
Qué carga lleva   cualquiera       cualquiera        cualquiera
```

La última fila es idéntica en las tres columnas: el eje de propagación no dice nada de lo que hace el malware, y por eso hacen falta los dos ejes.

Analogía: una enfermedad se describe por cómo se contagia (aire, agua, contacto) y por qué órgano ataca; "gripe pulmonar por aire" son dos datos, no uno.

Límite de la idea: algunas etiquetas (fileless, rootkit) describen una técnica de ocultación más que una propagación o una carga; se suman como un tercer adjetivo.

```
Eje propagación → virus, gusano, troyano: cómo entra y se reproduce.
Eje carga → ransomware, spyware, keylogger, botnet: qué hace dentro.
Técnica → rootkit, fileless: cómo se esconde.
```

### Virus

Un **virus** es un malware que se adjunta a un archivo o programa legítimo (su host) y se replica infectando otros archivos cuando ese host se ejecuta. No se mueve solo: necesita que alguien ejecute el archivo infectado y que ese archivo viaje (copiado a un USB, enviado por correo).

Analogía: como un virus biológico, no vive fuera de una célula; necesita un archivo huésped y que alguien lo "toque".

Cómo funciona: el virus inserta su código dentro del ejecutable (o en el sector de arranque, o como macro en un documento) y modifica el punto de entrada para que su código corra primero; luego devuelve el control al programa original para que el usuario no note nada. Variantes: virus de archivo, de sector de arranque (boot sector), de macro (en documentos Office), polimórficos (cambian su forma en cada copia para esquivar firmas) y metamórficos (reescriben su propio código).

Ejemplo: un `.exe` de un juego pirata pesa 2 MB; tras la infección pesa 2,1 MB y su hash cambió. Cada vez que se ejecuta, infecta otros 5 ejecutables de la misma carpeta.

Señales: ejecutables cuyo tamaño o hash cambia sin actualización; antivirus que alerta sobre muchos archivos a la vez; macros en documentos que no deberían tenerlas.

Defensas: antivirus con firmas y heurística, bloquear macros de Internet (Office lo hace por defecto desde 2022), no ejecutar software de origen dudoso, listas de aplicaciones permitidas (application allowlisting).

### Gusano (worm)

Un **gusano** es un malware que se replica a sí mismo y se propaga por la red de máquina en máquina sin necesitar un archivo huésped ni acción del usuario. Explota servicios de red vulnerables o credenciales débiles para saltar solo.

Analogía: un virus necesita que le des la mano a alguien; un gusano entra por la ventilación de todo el edificio.

Cómo funciona: escanea rangos de IP buscando un puerto con un servicio vulnerable, envía el exploit, copia su cuerpo al nuevo equipo y repite desde ahí. El crecimiento es exponencial: si cada equipo infecta 2 por minuto, en 10 minutos se pasa de 1 a más de 1000.

Ejemplo real: WannaCry (mayo de 2017) combinaba gusano y ransomware. Se propagaba por SMBv1 (puerto TCP 445) usando el exploit EternalBlue (boletín MS17-010) e infectó más de 200 000 equipos en unos 150 países en pocos días; Microsoft había publicado el parche dos meses antes.

Señales: picos de tráfico hacia un mismo puerto en muchas IP internas (escaneo lateral), CPU alta en muchos equipos a la vez, alertas del IDS por el mismo exploit desde varias fuentes internas.

Defensas: parchear rápido servicios expuestos, cerrar puertos que no se usan (TCP 445 nunca expuesto a Internet), segmentar la red (ver [`14-defensa-y-hardening`](../14-defensa-y-hardening/)), firewall en cada host.

```
Virus → se pega a un archivo y necesita que alguien lo ejecute.
Gusano → se copia solo por la red, sin archivo huésped ni usuario.
```

### Troyano (trojan)

Un **troyano** es un malware que se disfraza de programa legítimo o útil para que la víctima lo instale voluntariamente, y una vez instalado ejecuta funciones ocultas. No se replica: depende del engaño.

Analogía: el caballo de Troya; los troyanos abrieron la puerta ellos mismos porque parecía un regalo.

Cómo funciona: el instalador hace lo que promete (o lo aparenta) mientras instala en segundo plano el payload. La variante más común es el RAT (Remote Access Trojan): da al atacante control remoto completo (pantalla, archivos, cámara, shell). Un backdoor es cualquier acceso oculto que salta la autenticación normal; un RAT es una forma de backdoor.

Ejemplo: alguien busca "descargar editor PDF gratis", baja `PDFPro_Setup.exe` de un sitio de anuncios; el editor funciona, pero también instala un RAT que se conecta a un C2 por el puerto 443.

Señales: programas que abren conexiones salientes persistentes sin motivo, procesos con nombre parecido a uno del sistema en rutas raras (`C:\Users\ana\AppData\Roaming\svchost.exe`), firmas digitales ausentes o inválidas.

Defensas: instalar solo desde fuentes oficiales, verificar firma digital y hash, cuentas de usuario sin privilegios de administrador, EDR.

```
Troyano → engaña para ser instalado; no se replica.
RAT → troyano que da control remoto total.
Backdoor → cualquier acceso oculto que se salta el login.
```

### Ransomware

El **ransomware** es un malware que cifra los archivos de la víctima (o bloquea el sistema) y exige un rescate, normalmente en criptomonedas, a cambio de la clave para descifrarlos. Ataca la disponibilidad, y en su forma moderna también la confidencialidad.

Analogía: alguien entra a tu casa, pone candado a todos los cajones y te deja una nota con el precio de la llave.

Cómo funciona: tras entrar (phishing, RDP expuesto, VPN sin parchear), el operador suele pasar días moviéndose por la red, roba datos, borra las copias de sombra de Windows y desactiva copias de seguridad; luego cifra todo de golpe con criptografía híbrida: cada archivo con una clave simétrica (AES) y esas claves con la clave pública del atacante (RSA), así que sin su clave privada no se recupera. La doble extorsión añade la amenaza de publicar los datos robados aunque restaures tus copias. Hoy funciona como negocio (RaaS, Ransomware as a Service): un grupo desarrolla el malware y "afiliados" lo despliegan a cambio de un porcentaje.

Ejemplo: un viernes a las 23:00 un servidor de archivos de 2 TB empieza a renombrar `informe.xlsx` a `informe.xlsx.locked` a razón de 500 archivos por minuto, y en cada carpeta aparece `LEEME_RECUPERAR.txt` pidiendo 5 BTC.

Señales: renombrado masivo de archivos con una extensión nueva, uso de `vssadmin` o `wbadmin` para borrar copias, picos de escritura en disco, notas de rescate. En ATT&CK la técnica es T1486 (Data Encrypted for Impact).

Defensas: la regla 3-2-1 de copias de seguridad (3 copias, en 2 medios distintos, 1 fuera de línea o inmutable) y probar la restauración; MFA en VPN y RDP; no exponer RDP (TCP 3389) a Internet; EDR con protección contra cifrado masivo; segmentación. Antes de pagar, buscar descifradores gratuitos en No More Ransom.

### Rootkit

Un **rootkit** es un conjunto de herramientas que se instala con privilegios altos y modifica el propio sistema operativo para ocultar su presencia y la de otro malware. No es tanto un tipo de ataque como una técnica de ocultación.

Analogía: un ladrón que no solo entra a la casa, sino que reemplaza las cámaras de seguridad por unas que muestran la sala vacía.

Cómo funciona: intercepta las funciones que el sistema usa para responder "qué procesos hay" o "qué archivos hay en esta carpeta" y filtra la respuesta. Niveles, de menos a más peligroso:

```
Usuario (user-mode)  reemplaza o engancha librerías/programas (ls, ps)
Kernel               driver malicioso que altera tablas del núcleo
Bootkit              infecta el arranque (MBR/VBR) y carga antes que el SO
Firmware/UEFI        vive en el chip de la placa; sobrevive a formatear el disco
```

Ejemplo: en Linux, `ps aux` no muestra el proceso `kworkerd` que consume el 90 % de CPU, pero el monitor del hipervisor sí ve el consumo. El rootkit filtró la salida de `ps`.

Señales: discrepancias entre vistas (lo que dice el sistema vs. lo que se ve desde fuera, técnica llamada cross-view), drivers sin firmar, Secure Boot desactivado sin motivo, herramientas como `rkhunter` o `chkrootkit` con hallazgos.

Defensas: Secure Boot y arranque medido (TPM), solo drivers firmados, escanear desde un medio externo arrancable. Ante un rootkit de kernel, lo seguro es reinstalar desde cero, porque no puedes confiar en nada de lo que el sistema te diga.

### Spyware

El **spyware** es un malware que recopila información sobre el usuario o su actividad sin su consentimiento y la envía a un tercero. Ataca la confidencialidad.

Analogía: un micrófono escondido en la lámpara del salón.

Cómo funciona: registra navegación, capturas de pantalla, ubicación, contactos, micrófono o cámara, y lo exfiltra poco a poco para no llamar la atención. Variantes: adware agresivo que rastrea para mostrar publicidad, stalkerware instalado por una persona cercana en el teléfono de la víctima, y spyware comercial de vigilancia (como Pegasus) que usa exploits zero-click, es decir, que infectan sin que la víctima toque nada.

Ejemplo: una extensión de navegador "convertidor de PDF" con 50 000 usuarios envía cada URL visitada a un servidor externo.

Señales: batería o datos móviles que se consumen sin explicación, permisos excesivos (una linterna que pide micrófono y ubicación), extensiones que no recuerdas haber instalado.

Defensas: revisar permisos de apps y extensiones, actualizar el SO (los exploits de spyware comercial suelen parchearse rápido), modo de bloqueo (Lockdown Mode en iOS) para personas de alto riesgo, EDR o antimalware móvil.

### Keylogger

Un **keylogger** es un tipo de spyware (o un dispositivo físico) que registra cada tecla pulsada para capturar contraseñas, mensajes y números de tarjeta.

Analogía: alguien que mira tus manos mientras escribes, pero sin cansarse nunca.

Cómo funciona: en software, se engancha a los eventos de teclado del sistema (en Windows, por ejemplo, con hooks de teclado) o lee el buffer de entrada, y guarda las teclas con la ventana activa ("Chrome – Banco – login"). En hardware, es un adaptador USB que se conecta entre el teclado y el PC y guarda todo en su memoria interna. ATT&CK lo cataloga como T1056.001 (Keylogging).

Ejemplo: en un cibercafé, un adaptador USB de 3 cm entre el teclado y la torre guarda 16 MB de pulsaciones; el atacante vuelve una semana después y se lo lleva.

Señales: dispositivos extraños en los puertos USB, procesos que se enganchan al teclado, envíos periódicos de archivos de texto pequeños hacia fuera.

Defensas: MFA (la contraseña capturada no basta), gestores de contraseñas que autocompletan sin teclear, llaves de seguridad FIDO2, revisar físicamente los equipos públicos, EDR.

```
Spyware → espía cualquier actividad (navegación, cámara, ubicación).
Keylogger → espía solo el teclado; puede ser software o un aparato físico.
```

### Botnet

Una **botnet** es una red de equipos infectados (bots o zombis) controlados de forma remota por un mismo atacante (el botmaster) a través de infraestructura de C2. Su valor está en la escala: miles de equipos pequeños suman mucho ancho de banda y muchas IP distintas.

Analogía: un titiritero con miles de marionetas; cada una es débil, pero juntas llenan el escenario.

Cómo funciona: cada bot "llama a casa" periódicamente (beaconing) para pedir órdenes. El C2 puede ser centralizado (un servidor HTTP o IRC, fácil de tumbar), peer-to-peer (los bots se pasan órdenes entre ellos, sin un punto único) o usar DGA (Domain Generation Algorithm: cada día el malware calcula cientos de dominios posibles y el atacante registra solo uno, así que bloquear dominios uno a uno no sirve). Usos: DDoS (ver [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/)), spam, relleno de credenciales, minería, alquiler a otros criminales.

Ejemplo real: Mirai (2016) entró en cámaras IP y routers probando unas 60 combinaciones de usuario y contraseña por defecto por Telnet, y usó cientos de miles de ellos para tumbar el proveedor DNS Dyn, dejando sin servicio a sitios grandes durante horas.

Señales: conexiones salientes a intervalos muy regulares (cada 300 s exactos), consultas DNS a dominios aleatorios tipo `xk3j9qz.top`, tráfico saliente inusual desde dispositivos IoT.

Defensas: cambiar credenciales por defecto, no exponer Telnet/SSH de IoT a Internet, segmentar IoT en su propia VLAN, filtrar egreso (lo que sale de la red), bloquear dominios recién registrados.

### Fileless malware

El **fileless malware** es un malware que opera sin escribir un ejecutable propio en disco: vive en memoria y usa herramientas legítimas del sistema (PowerShell, WMI, `rundll32`, `mshta`) para hacer su trabajo. A esta táctica se le llama "living off the land" (vivir de lo que hay en el terreno), y a esas herramientas, LOLBins.

Analogía: un ladrón que no trae herramientas, usa las del propio taller; nadie sospecha de un martillo que siempre estuvo ahí.

Cómo funciona: un documento o enlace lanza PowerShell con un comando codificado en Base64 que descarga código y lo ejecuta directamente en memoria (`IEX`). La persistencia se esconde en el registro (un script guardado como valor de una clave) o en una suscripción de eventos WMI. Como no hay un `.exe` nuevo, el antivirus basado en archivos no tiene nada que escanear.


Ejemplo: Sysmon registra a `winword.exe` como proceso padre de `powershell.exe` con `-enc`; un procesador de texto no tiene ningún motivo legítimo para lanzar una consola.

Señales: relaciones padre-hijo anómalas (Office → PowerShell), comandos codificados, eventos 4104 de PowerShell (Script Block Logging) con `IEX` o `DownloadString`, suscripciones WMI nuevas.

Defensas: activar PowerShell Script Block Logging y AMSI, Constrained Language Mode, EDR basado en comportamiento, reglas ASR de Microsoft (bloquean que Office cree procesos hijos), limitar quién puede usar PowerShell.

### Otros tipos de malware

Hay tipos menos frecuentes en los exámenes pero que aparecen en incidentes reales:

- Bomba lógica (logic bomb): código que espera una condición (una fecha, el despido de un empleado) para activarse. Ejemplo: un administrador deja un script que borra la base de datos si su usuario desaparece del directorio.
- Adware y bloatware: software que muestra publicidad o que viene preinstalado sin pedirlo; molesta y amplía la superficie de ataque.
- Cryptojacker: usa la CPU o GPU de la víctima para minar criptomonedas; señal típica, CPU al 100 % sostenido.
- Dropper y downloader: piezas pequeñas cuyo trabajo es instalar o descargar el malware principal.
- Infostealer: roba de golpe contraseñas guardadas en el navegador, cookies de sesión y carteras de criptomonedas.

```
Bomba lógica → espera una condición para activarse.
Cryptojacker → roba capacidad de cómputo para minar.
Infostealer → roba credenciales y cookies guardadas de una vez.
```

## Attack Types and Differences

### Social Engineering

La **ingeniería social** (social engineering) es un conjunto de técnicas de manipulación psicológica que llevan a una persona a revelar información, dar acceso o ejecutar una acción que la perjudica. Existe porque engañar a una persona suele ser más barato que romper una máquina: no hace falta un exploit si alguien te da la contraseña.

Analogía: el timador del "tío del pueblo" que llama diciendo que es un sobrino en apuros; no rompe ninguna cerradura, convence al dueño de abrirla.

Cómo funciona: explota principios psicológicos de persuasión. Los que repiten los exámenes:

- Autoridad: "soy del departamento de TI / soy el director".
- Urgencia: "si no lo haces en 30 minutos se bloquea la cuenta".
- Escasez: "solo quedan 3 cupos".
- Prueba social: "todos tus compañeros ya lo completaron".
- Familiaridad o simpatía: "hablamos la semana pasada en la conferencia".
- Intimidación: "si no pagas habrá consecuencias legales".
- Confianza: hacerse pasar por un proveedor conocido.

Todos los ataques que siguen (phishing, vishing, tailgating, impersonation...) son variantes de ingeniería social por distintos canales.

Ejemplo: un correo "de RR. HH." con asunto "Ajuste salarial 2026 – confirmar antes de las 17:00" combina autoridad, urgencia e interés propio en una sola línea.

Señales: presión de tiempo, petición que se sale del procedimiento normal, pedir secreto ("no se lo comentes a nadie"), canal inusual para ese tipo de solicitud.

Defensas: formación y simulacros de phishing, procedimientos que no dependan de la buena fe (verificar por un segundo canal conocido), cultura donde reportar no se castiga, política clara de qué nunca se pide por teléfono o correo.

### Phishing, vishing, smishing y whaling son el mismo ataque por distinto canal

> [!IMPORTANT]
> Phishing, vishing (whishing), smishing y whaling no son técnicas distintas: son la misma ingeniería social que cambia el canal (correo, voz, SMS) o el objetivo (cualquiera, una persona concreta, un alto directivo).

La familia phishing es un grupo de ataques que comparten mecanismo (suplantar a alguien de confianza y empujar a una acción) y se distinguen solo por dos variables: el medio y a quién apuntan.

```
               Phishing      Vishing        Smishing       Whaling
Canal          correo        voz/teléfono   SMS/mensajería correo o voz
Objetivo       masivo        cualquiera     masivo         alto directivo
Mecanismo      suplantar + urgencia + acción (clic, dato, pago)
```

La última fila es idéntica en todas las columnas: ahí está la idea entera. Spear phishing es la versión dirigida a una persona concreta con datos personalizados; whaling es spear phishing contra un "pez gordo".

Analogía: es el mismo timador; a veces escribe una carta, a veces llama, a veces manda un mensaje.

Límite: cambiar de canal sí cambia las señales (en voz no hay enlace que inspeccionar) y algunas defensas (filtro de correo vs. verificación por callback).

```
Phishing → por correo, normalmente masivo.
Spear phishing → por correo, a una persona concreta con datos suyos.
Vishing → por voz.
Smishing → por SMS o mensajería.
Whaling → contra altos directivos.
```

### Phishing

El **phishing** es un ataque de ingeniería social por correo electrónico que suplanta a una entidad de confianza para que la víctima haga clic en un enlace, abra un adjunto o entregue credenciales o dinero. Es el vector de entrada más común en los informes de incidentes; en ATT&CK es T1566, con subtécnicas como T1566.001 (adjunto) y T1566.002 (enlace).

Analogía: una carta con el membrete del banco copiado a la perfección, pero con el número de teléfono del estafador.

Cómo funciona: el atacante registra un dominio parecido, clona la página de login del servicio real y envía correos que llevan ahí. Las credenciales que introduces van a su servidor; los kits modernos (adversary-in-the-middle) incluso reenvían en tiempo real tu código MFA al sitio real y roban la cookie de sesión.

Ejemplo realista:

Señales que delatan un correo de phishing, en un caso inventado de "tu cuenta será suspendida":

- Urgencia y amenaza: "si no verificas en 24 horas, la cuenta se bloquea".
- Remitente que no cuadra: el nombre visible dice el banco, pero la dirección real es de un dominio parecido o gratuito.
- Enlace engañoso: al pasar el ratón, el dominio real es algo como `banco.com.verify-account.xyz`; lo que manda es lo que va justo antes del último punto, así que el dueño es `verify-account.xyz` y `banco.com` es solo un subdominio que el atacante controla.
- Saludo genérico, errores de idioma y adjuntos inesperados (`.zip`, `.html`, documentos con macros).
- Fallos de SPF, DKIM o DMARC en las cabeceras, que el filtro de correo suele marcar.

Señales: remitente que no coincide con el dominio legítimo, `Reply-To` distinto del `From`, enlaces cuyo destino real (pasando el ratón) no es el que dice el texto, urgencia, saludos genéricos, cabeceras con SPF, DKIM o DMARC fallidos.

Defensas: filtro de correo con SPF, DKIM y DMARC configurados (DMARC en `p=reject`), aislamiento de enlaces y adjuntos, MFA resistente a phishing (FIDO2/passkeys, que no funcionan en un dominio falso), botón de "reportar phishing", simulacros.

### Whishing

El **whishing**, más conocido como **vishing** (voice phishing), es phishing por llamada de voz: el atacante telefonea haciéndose pasar por banco, soporte técnico, autoridad o compañero para obtener datos, accesos o pagos. ATT&CK lo recoge como T1566.004 (Spearphishing Voice).

Analogía: el mismo estafador del correo, pero ahora con voz amable y prisa, y sin un enlace que puedas revisar con calma.

Cómo funciona: falsifica el identificador de llamada (caller ID spoofing) para que aparezca el número del banco, usa datos previos de la víctima (nombre, últimos dígitos de la tarjeta) para ganar credibilidad y presiona para actuar durante la llamada. Variante moderna: clonación de voz con IA a partir de unos segundos de audio público de un directivo.

Ejemplo realista: "Le llamo de la mesa de ayuda; vemos un acceso sospechoso a su cuenta desde otro país. Para bloquearlo, dígame el código de 6 dígitos que le acaba de llegar por SMS". Ese código es el MFA que el atacante está intentando usar en ese momento. En 2023, un grupo entró en una gran cadena de casinos de Las Vegas llamando a la mesa de ayuda y haciéndose pasar por un empleado para que le restablecieran el acceso.

Señales: piden códigos MFA, contraseñas o instalar software de acceso remoto; insisten en no colgar; el número "parece" correcto pero la petición se sale del procedimiento.

Defensas: regla de colgar y devolver la llamada a un número oficial conocido (callback), la mesa de ayuda verifica identidad con algo que el atacante no tenga (videollamada, aprobación del jefe), nunca dictar códigos MFA, palabra clave interna para pagos urgentes.

### Whaling

El **whaling** es un phishing dirigido a altos directivos (CEO, CFO, directores) o que suplanta a ellos, diseñado a medida con información pública para conseguir pagos grandes o datos estratégicos. Es spear phishing con objetivo de alto valor.

Analogía: un pescador que no tira la red al mar, sino que espera semanas con un arpón a una sola ballena.

Cómo funciona: el atacante estudia al directivo (LinkedIn, entrevistas, informes anuales, firma de correo), elige un momento creíble (cierre de trimestre, adquisición en curso, viaje del CEO) y redacta un mensaje con el tono y vocabulario de la empresa. Su pariente cercano es el BEC (Business Email Compromise): usar una cuenta real comprometida, o una muy parecida, para pedir transferencias.

Ejemplo realista: con el CEO de viaje (anunciado en redes), el CFO recibe un correo "del CEO": "Estamos cerrando una adquisición confidencial. Transfiere 250 000 USD a esta cuenta del bufete antes de las 15:00 y no lo comentes con nadie hasta el anuncio".

Señales: petición de transferencia fuera del flujo de aprobación, cambio de cuenta bancaria de un proveedor, secretismo, dominio con una letra cambiada, el remitente "no puede hablar por teléfono ahora".

Defensas: doble aprobación y verificación por teléfono a un número conocido para toda transferencia o cambio de datos bancarios, formación específica para directivos y sus asistentes, etiqueta "EXTERNO" en correos de fuera, reducir la información pública sobre viajes y organigrama.

### Smishing

El **smishing** (SMS phishing) es phishing por mensaje de texto o apps de mensajería: un SMS corto con un enlace o un número para llamar, suplantando a un banco, paquetería u organismo público.

Analogía: el mismo folleto engañoso, pero metido en tu bolsillo, donde lo lees con menos atención.

Cómo funciona: los SMS no muestran cabeceras ni dominio de remitente verificable y en el móvil la URL completa casi no se ve, así que el engaño es más fácil. Los enlaces suelen ir acortados o a dominios recién registrados con apariencia de servicio real.

Ejemplo realista:

```
Correos: Su paquete está retenido por tasas pendientes (1,99 EUR).
Pague aquí antes de 24 h: https://correos-entrega-es.top/p/8812
```

El pago pequeño sirve para capturar la tarjeta completa y el código 3-D Secure.

Señales: mensajes de paquetes que no esperas, importes pequeños, dominios `.top`, `.xyz` o con guiones, números remitentes de otro país, errores de redacción.

Defensas: no tocar enlaces de SMS, entrar a la app o web oficial escribiendo la dirección, reportar el mensaje (en muchos países se reenvía al 7726), filtros antispam del operador.

### Spam vs Spim

El **spam** es correo electrónico no solicitado enviado de forma masiva, y el **spim** (spam over instant messaging) es lo mismo en mensajería instantánea y chats (WhatsApp, Telegram, Teams, Discord). Ninguno es por sí mismo un ataque dirigido, pero ambos son el medio de transporte de mucho phishing y malware.

Analogía: el spam es el buzoneo de folletos en tu buzón; el spim es alguien que te deja folletos en la mesa mientras tomas un café.

Cómo funciona: se envía desde botnets o cuentas comprometidas o creadas en masa. El spam se combate con filtros maduros (reputación de IP, SPF/DKIM/DMARC, análisis de contenido) que bloquean la gran mayoría antes de la bandeja; el spim es más peligroso proporcionalmente porque los chats tienen menos filtrado y el mensaje suele llegar desde la cuenta comprometida de un contacto real.

Ejemplo: un compañero de Discord, con la cuenta robada, te escribe "mira, sales en este video jaja" con un enlace a una página que roba tu token de sesión.

Señales: mensajes que no encajan con el estilo del contacto, enlaces sin contexto, ofertas demasiado buenas.

Defensas: filtros antispam y reputación de remitentes, en chats no confiar en enlaces aunque vengan de un contacto conocido, verificar por otro canal, MFA para evitar que tu cuenta acabe siendo la que envía spim.

```
Spam → mensajes masivos no solicitados por correo.
Spim → mensajes masivos no solicitados por mensajería instantánea.
Phishing → mensaje engañoso con intención de robar; puede viajar dentro de spam o spim.
```

### Shoulder Surfing

El **shoulder surfing** es una técnica de obtención de información que consiste en mirar por encima del hombro (o a distancia, con cámara o prismáticos) la pantalla o el teclado de alguien para ver contraseñas, PIN o datos.

Analogía: el que mira tu PIN en la cola del cajero.

Cómo funciona: aprovecha espacios públicos o compartidos (aviones, cafeterías, oficinas abiertas, transporte). Puede ser directo o grabado con el móvil a varios metros, y con zoom un PIN de 4 dígitos se lee desde lejos.

Ejemplo: en un tren, un pasajero graba a 2 metros a un consultor que inicia sesión en la VPN de su cliente; con el video a cámara lenta saca usuario y contraseña.

Señales: alguien que se coloca detrás o al lado sin motivo, móviles apuntando a tu pantalla, cámaras cerca de cajeros o terminales.

Defensas: filtros de privacidad en la pantalla, tapar el teclado al teclear el PIN, bloquear la sesión al levantarse, MFA (la contraseña vista no basta), contraseñas largas que no se memorizan de un vistazo, posicionar puestos de trabajo para que las pantallas no den a pasillos o ventanas.

### Tailgating

El **tailgating** es un ataque de acceso físico en el que una persona no autorizada entra a una zona restringida siguiendo de cerca a alguien que sí tiene acceso, aprovechando que la puerta queda abierta. Cuando el empleado lo deja pasar a sabiendas (por cortesía o engaño), se llama piggybacking.

Analogía: colarse en el metro pegado detrás de otro pasajero cuando pasa el torniquete.

Cómo funciona: el atacante se presenta cargado de cajas, con uniforme de repartidor o hablando por teléfono; la cortesía hace el resto. Una vez dentro, puede conectar un dispositivo a un puerto de red libre, dejar USB "olvidados" o fotografiar pizarras.

Ejemplo: en un edificio con tarjeta, alguien con chaleco reflectante y dos cajas de pizza espera a que salga un empleado; este le sujeta la puerta. Dentro, conecta un Raspberry Pi a una toma de red de una sala de reuniones vacía.

Señales: personas sin tarjeta visible, puertas que se quedan abiertas, grupos que pasan con una sola lectura de tarjeta (el sistema registra 1 acceso y entran 3).

Defensas: torniquetes o mantraps (esclusas de dos puertas que solo dejan pasar a una persona), vigilancia, tarjetas visibles obligatorias, política de "una tarjeta, una persona" y formación para que pedir la tarjeta no sea descortés, puertos de red sin uso desactivados (802.1X).

```
Tailgating → se cuela sin que la víctima se dé cuenta o lo consienta.
Piggybacking → la víctima lo deja pasar a sabiendas.
```

### Dumpster Diving

El **dumpster diving** es una técnica de recolección de información que consiste en buscar en la basura (papeleras, contenedores, equipos desechados) documentos o dispositivos con datos útiles para un ataque.

Analogía: reconstruir la vida de alguien leyendo los recibos que tiró al contenedor.

Cómo funciona: organigramas, listas de teléfonos internos, facturas de proveedores, post-its con contraseñas o discos duros mal borrados dan material para ingeniería social posterior (saber a quién llamar y con qué nombres).

Ejemplo: un atacante saca de un contenedor de oficina un listado de extensiones y una factura del proveedor de impresoras; al día siguiente llama a recepción: "Soy de [proveedor real], vengo a revisar la impresora de la planta 3, me pidió la extensión 4120".

Señales: contenedores revueltos, personas ajenas en zonas de reciclaje; normalmente se descubre después, cuando un ataque usa datos que solo estaban en papel.

Defensas: trituradoras de corte cruzado, contenedores de papel confidencial cerrados con llave, borrado seguro o destrucción física de discos (ver NIST SP 800-88 en [`14-defensa-y-hardening`](../14-defensa-y-hardening/)), política de mesa limpia.

### Zero day

Un **zero day** (día cero) es una vulnerabilidad desconocida para el fabricante, o conocida pero sin parche disponible, y el exploit que la aprovecha. El nombre viene de que el defensor ha tenido "cero días" para preparar una corrección.

Analogía: una cerradura con un defecto que solo conoce el ladrón; el cerrajero ni sabe que existe, así que no puede arreglarla.

Cómo funciona: el ciclo de vida de una vulnerabilidad tiene una ventana peligrosa entre su descubrimiento por un atacante y la aplicación del parche por la víctima.

```
descubierta     usada en     fabricante    parche       víctima
por atacante -> ataques  ->  se entera -> publicado -> lo instala
   |<------- zero day ------------>|<-- n-day: conocida, sin parchear -->|
```

Después del parche deja de ser zero day y pasa a ser n-day (conocida), pero sigue siendo explotable en cada equipo que no se actualice; muchos incidentes graves usan n-days, no zero days. Los zero days valen mucho en el mercado (de decenas de miles a millones de dólares según el objetivo), así que suelen reservarse para objetivos valiosos.

Ejemplo real: Log4Shell (CVE-2021-44228), en la librería Java Log4j, permitía ejecutar código remoto con solo lograr que la aplicación escribiera en sus logs un texto especial controlado por el atacante; se explotó masivamente en diciembre de 2021, en los días alrededor de su divulgación pública.

Señales: por definición no hay firma, así que se detecta por comportamiento: un proceso de servidor web que lanza una shell, conexiones salientes nuevas desde un servidor que nunca las hacía, caídas inexplicables de un servicio.

Defensas: defensa en profundidad (si una capa cae, otra frena), mínimo privilegio, segmentación, EDR y detección por comportamiento, WAF con reglas virtuales mientras llega el parche, inventario de software (SBOM) para saber rápido si estás afectado, y parchear deprisa en cuanto deja de ser zero day (seguir el catálogo KEV de CISA).

> [!WARNING]
> Un zero day no es "cualquier ataque nuevo" ni "un ataque muy grave": es una vulnerabilidad sin parche disponible. Cuando el parche sale, el mismo fallo es un n-day, y la mayoría de las intrusiones explotan n-days sin parchear.

### Reconnaissance

El **reconocimiento** (reconnaissance) es la fase de recolección de información sobre un objetivo antes de atacarlo: personas, dominios, IP, tecnologías y puntos de entrada. Es la primera fase de la [Cyber Kill Chain](../13-frameworks-de-amenazas/) y en ATT&CK es la táctica TA0043.

Analogía: el ladrón que pasa una semana observando la casa: a qué hora sale la familia, si hay perro, qué ventana queda abierta.

Se divide según si el atacante toca o no los sistemas del objetivo:

- Pasivo: no envía nada al objetivo; consulta fuentes públicas y de terceros (OSINT). Whois, DNS públicos, registros de certificados (crt.sh), LinkedIn, ofertas de empleo ("buscamos experto en Fortinet"), Shodan, buscadores. El objetivo no puede detectarlo en sus propios logs.
- Activo: interactúa directamente con los sistemas del objetivo: escaneo de puertos, enumeración de servicios, banner grabbing, llamar a recepción. Da información más precisa, pero deja rastro en firewalls e IDS.

Ejemplo pasivo y activo contra un dominio de laboratorio propio:

Qué expone cada fuente, visto desde el defensor:

- Pasivo: el registro whois y los DNS públicos muestran dominios, servidores de correo y a veces contactos; los registros públicos de certificados (Certificate Transparency) revelan subdominios internos que alguien publicó con HTTPS; las ofertas de empleo delatan qué tecnologías usa la empresa. Nada de esto toca tus sistemas.
- Activo: un escaneo de puertos o una enumeración de servicios sí llega a tus equipos, y queda registrado en el firewall y el IDS como ráfagas de conexiones desde una misma IP.

Ejercicio defensivo: busca tu propio dominio en un buscador de Certificate Transparency y revisa qué subdominios quedaron expuestos sin que nadie lo supiera.

Señales (solo del activo): muchas conexiones a puertos distintos desde una misma IP en segundos, peticiones a rutas inexistentes (404 en masa), consultas DNS de transferencia de zona (AXFR) rechazadas.

Defensas: reducir lo que se publica (ofertas de empleo sin versiones exactas, metadatos de documentos limpios), no filtrar subdominios internos, IDS y alertas por escaneos, honeypots, cerrar servicios innecesarios. Las herramientas se ven en [`07-herramientas-de-red`](../07-herramientas-de-red/) y el OSINT en [`13-frameworks-de-amenazas`](../13-frameworks-de-amenazas/).

> [!NOTE]
> El reconocimiento activo contra sistemas que no son tuyos y sin autorización por escrito puede ser delito; practícalo solo en tus laboratorios o en plataformas como TryHackMe y HTB.

```
Reconocimiento pasivo → sin tocar al objetivo (OSINT); no deja rastro en sus logs.
Reconocimiento activo → interactúa con sus sistemas (escaneos); más preciso, detectable.
```

### Impersonation

La **suplantación** (impersonation) es una técnica de ingeniería social en la que el atacante finge ser otra persona o rol (técnico, auditor, proveedor, directivo, policía) para obtener confianza y acceso. Suele apoyarse en un pretexto (pretexting): una historia inventada pero coherente que justifica la petición.

Analogía: el falso inspector del gas que llama a tu puerta con un carnet impreso en casa.

Cómo funciona: cuanta más información previa (del reconocimiento o del dumpster diving) tenga el atacante, más creíble es el papel: nombres reales de compañeros, jerga interna, número de ticket plausible. Puede ser en persona, por teléfono (vishing) o por correo (phishing). La suplantación de marca (brand impersonation) es la versión a gran escala: copiar logotipos y estilo de una empresa conocida.

Ejemplo realista: "Hola, soy Carlos, de la auditoría externa de [empresa auditora real]. Laura, de Finanzas, me dijo que tú me podías pasar el listado de proveedores con sus cuentas antes del viernes".

Señales: alguien que conoce algunos nombres pero no los procedimientos, que pide saltarse un paso "porque Laura ya lo aprobó", credenciales o uniformes que no se pueden verificar.

Defensas: verificar identidad por un canal independiente (llamar a Laura a su extensión real), visitas registradas y acompañadas, política explícita de qué datos se entregan a terceros y por qué vía, formación con casos reales.

### Watering Hole Attack

Un **watering hole attack** es un ataque dirigido en el que el atacante compromete un sitio web legítimo que un grupo concreto de víctimas visita a menudo, para infectarlas cuando entran. En vez de ir a buscar a la víctima, la espera donde ya va.

Analogía: el depredador que no persigue a la gacela por la sabana, sino que espera junto al abrevadero al que todas tienen que ir a beber. De ahí el nombre.

Por qué existe: las organizaciones bien defendidas filtran el correo y entrenan a su gente contra el phishing, así que el atacante busca un camino que la víctima considere seguro. Un sitio que el personal visita a diario (el portal de un proveedor, un foro del sector, la web de la asociación profesional) no despierta sospechas, y a menudo está menos protegido que el objetivo real.

Cómo se ve desde el defensor: el sitio comprometido sigue funcionando normal, pero sirve un script añadido que solo actúa con ciertos visitantes (por ejemplo, los que llegan desde el rango de IP de la empresa objetivo) y que intenta aprovechar una vulnerabilidad del navegador o empujar una descarga. Por eso se combina con frecuencia con el [Drive by Attack](#drive-by-attack) de la sección siguiente.

Ejemplo: 40 ingenieros de una empresa de energía consultan cada semana el foro de un fabricante de turbinas. El foro tiene un CMS sin parchear. Un día, solo los visitantes que llegan desde la red de esa empresa reciben un script extra; tres equipos con el navegador desactualizado terminan con malware, y el resto de visitantes del mundo no nota nada.

Señales: varios equipos de la misma organización infectados el mismo día sin correo sospechoso en común, y en los logs del proxy todos visitaron el mismo sitio externo poco antes; el EDR muestra al navegador lanzando procesos que no debería.

Defensas: navegadores y plugins siempre parcheados, aislamiento del navegador o filtrado web por categorías y reputación, EDR que vigile los procesos hijos del navegador, y compartir indicadores con el sector cuando se detecta un sitio comprometido.

```
Phishing      → el atacante lleva el anzuelo a la víctima.
Watering hole → el atacante espera a la víctima en un sitio que ya visita.
```

### Drive by Attack

Un **drive-by attack** (drive-by download o drive-by compromise) es un ataque en el que basta con visitar una página web para que el equipo quede comprometido, sin que la víctima haga clic en nada ni acepte una descarga a conciencia.

Analogía: pisar un chicle por pasar por la acera; no hiciste nada más que caminar por ahí.

Cómo funciona a nivel conceptual: la página (comprometida o creada por el atacante, o un anuncio malicioso dentro de una página legítima, lo que se llama malvertising) carga código que comprueba la versión del navegador y sus componentes, y si encuentra una vulnerabilidad conocida la aprovecha para ejecutar código en el equipo. En la variante más simple no hay vulnerabilidad: la página fuerza una descarga con un nombre engañoso ("actualizacion_navegador.exe") y confía en que el usuario la abra.

Ejemplo: una persona busca "descargar plantilla de factura gratis", el primer resultado es un anuncio, la página tarda dos segundos en cargar y no pasa nada visible. Esa noche el antivirus alerta de un proceso desconocido que se conecta cada 60 segundos a un dominio registrado hace una semana.

Señales: descargas que el usuario no recuerda haber pedido, procesos hijos del navegador, conexiones salientes periódicas (beaconing) a dominios recién creados.

Defensas: parchear navegador y sistema operativo (la mayoría de drive-by aprovechan fallos ya corregidos), bloquear anuncios y categorías web de riesgo, quitar plugins que no se usan, ejecutar el navegador sin privilegios de administrador y con aislamiento. En ATT&CK es la técnica T1189, Drive-by Compromise.

```
Watering hole → la estrategia: elegir qué sitio comprometer según a quién quieres alcanzar.
Drive-by      → el mecanismo: la visita sola basta para infectar.
```

### Typo Squatting

El **typosquatting** es la práctica de registrar dominios que se parecen a uno legítimo, contando con que la gente se equivoque al escribir o no se fije al leer, para llevarla a un sitio falso. También se le llama URL hijacking.

Analogía: abrir una tienda llamada "Walmrat" justo al lado de la real; algunos entran sin darse cuenta.

Las variantes más comunes, con un banco inventado `mibanco.com`:

- Error de tecleo: `mibnaco.com`, `mibancoo.com`.
- Otro dominio de nivel superior: `mibanco.co`, `mibanco.net`.
- Caracteres parecidos: `rnibanco.com` (r y n juntas parecen una m), o letras de otro alfabeto que se ven iguales (ataque homógrafo con dominios internacionalizados).
- Palabras añadidas: `mibanco-seguridad.com`, `mibanco-login.com`.

Para qué se usa: páginas de phishing, publicidad, distribución de malware, y también en la cadena de suministro de software: paquetes con nombres casi idénticos a librerías populares en npm o PyPI (`reqeusts` en vez de `requests`) que se instalan por un error al escribir.

Ejemplo: un empleado escribe a mano la dirección del portal de nóminas, se equivoca en una letra y llega a una copia perfecta que le pide usuario y contraseña. Lo único distinto es una letra en la barra de direcciones.

Señales: el dominio no coincide letra por letra, certificado emitido hace pocos días a nombre de otra entidad, dominio registrado recientemente.

Defensas: que la empresa registre preventivamente las variantes obvias de su propio dominio, vigilar el registro de dominios parecidos (herramientas como dnstwist generan y comprueban las variantes de tu dominio), usar marcadores en vez de escribir direcciones, filtros DNS que bloqueen dominios recién registrados, y en desarrollo fijar las dependencias y revisar el nombre exacto de cada paquete.

### Brute Force vs Password Spray

Un ataque de **fuerza bruta** (brute force) es un ataque de contraseñas que prueba muchas contraseñas contra una misma cuenta hasta acertar. Un **password spray** es un ataque de contraseñas que prueba unas pocas contraseñas muy comunes contra muchas cuentas distintas, para no disparar el bloqueo de ninguna.

Analogía: la fuerza bruta es probar mil llaves en una sola puerta; el spray es probar la misma llave maestra barata en mil puertas del edificio, una vez en cada una.

Por qué existe el spray: casi todos los sistemas bloquean una cuenta tras unos pocos fallos seguidos (por ejemplo, 5). La fuerza bruta contra una cuenta choca con ese límite enseguida. El spray lo esquiva: un solo intento por cuenta cada cierto tiempo nunca llega al umbral, y en una organización de 2.000 personas basta con que unas pocas usen "Verano2026!" para entrar.

La diferencia se ve en los logs, y es la forma de reconocer cada uno:

```
Fuerza bruta (una cuenta, muchos intentos):
  09:00:01  fallo  usuario=ana    origen=203.0.113.7
  09:00:02  fallo  usuario=ana    origen=203.0.113.7
  09:00:03  fallo  usuario=ana    origen=203.0.113.7
  ... 500 fallos en 10 minutos contra "ana" → cuenta bloqueada

Password spray (muchas cuentas, un intento cada una):
  09:00:01  fallo  usuario=ana    origen=203.0.113.7
  09:00:01  fallo  usuario=bruno  origen=203.0.113.7
  09:00:02  fallo  usuario=carla  origen=203.0.113.7
  ... 1 fallo por cuenta en 800 cuentas, misma IP y misma hora
```

En el primero la señal es el número de fallos por cuenta; en el segundo, el número de cuentas distintas que fallan desde un mismo origen en poco tiempo, aunque cada cuenta tenga un solo fallo. Una alerta que solo cuenta fallos por cuenta no ve nunca un spray.

Relacionados, para no confundirlos: el ataque de diccionario es una fuerza bruta que prueba palabras de una lista en vez de todas las combinaciones; el credential stuffing prueba pares usuario-contraseña reales filtrados de otro sitio, y funciona porque la gente reutiliza contraseñas.

Defensas: MFA, que deja inútil la contraseña adivinada; bloqueo o retraso progresivo por cuenta contra la fuerza bruta; detección por origen y por número de cuentas distintas contra el spray; prohibir contraseñas comunes y filtradas (lo que recomienda NIST SP 800-63B en lugar de forzar símbolos raros); comprobar contraseñas contra listas de filtraciones.

> [!TIP]
> Si en el examen o en los logs ves muchos fallos en una sola cuenta, es fuerza bruta; si ves un fallo en cada una de muchas cuentas desde el mismo origen, es password spray.

```
Fuerza bruta      → muchas contraseñas contra una cuenta; la frena el bloqueo de cuenta.
Password spray    → pocas contraseñas comunes contra muchas cuentas; esquiva el bloqueo.
Diccionario       → fuerza bruta con una lista de palabras probables.
Credential stuff. → pares usuario-contraseña filtrados de otro sitio; explota la reutilización.
```

## Understand Threat Classification

### Zero Day

En la clasificación de amenazas, una **amenaza zero day** es una amenaza que aprovecha una vulnerabilidad que el fabricante todavía no conoce o para la que aún no existe parche, así que los defensores tienen "cero días" de ventaja. El concepto completo, con su ciclo de vida y el paso a n-day, está en la sección [Zero day](#zero-day) de la parte anterior.

Lo que añade la clasificación es dónde cae: un zero day es por definición una amenaza desconocida para las defensas basadas en firmas, porque no hay firma de algo que nadie ha visto. Por eso se defiende con capas que no dependen de conocerlo: detección por comportamiento, mínimo privilegio, segmentación y respuesta rápida.

Ejemplo: el lunes aparece un fallo en un servidor de correo muy usado que se aprovecha activamente; el parche sale el jueves. Durante esos tres días, quien tenía ese servidor expuesto solo contaba con su monitoreo y su segmentación.

### Known vs Unknown

Las **amenazas conocidas** (known) son amenazas que ya han sido identificadas, analizadas y documentadas, de modo que existen firmas, indicadores o parches para ellas. Las **amenazas desconocidas** (unknown) son amenazas nuevas o modificadas que todavía no figuran en ninguna base de firmas, y por eso hay que detectarlas por lo que hacen y no por cómo son.

Analogía: el guardia con el cartel de "se busca" reconoce al ladrón que sale en la foto (conocido); al que nunca ha sido fichado solo lo detecta si se comporta raro, por ejemplo si prueba todas las puertas (desconocido).

Cómo se detecta cada una:

- Conocida: antivirus por firmas, listas de indicadores (hashes, IPs, dominios), escáneres de vulnerabilidades que buscan CVE publicados. Barato y con pocos falsos positivos.
- Desconocida: análisis de comportamiento y anomalías, EDR, sandboxing, threat hunting. Atrapa lo nuevo, pero genera más falsos positivos y exige más trabajo humano.

Ejemplo: un malware conocido llega con el mismo hash que ya figura en VirusTotal, y el antivirus lo borra al instante. Ese mismo malware recompilado con un cambio mínimo tiene otro hash, la firma ya no coincide, y solo lo delata su comportamiento, como cifrar 500 archivos por minuto.

```
Amenaza conocida    → ya documentada; se detecta por firma o indicador.
Amenaza desconocida → nueva o modificada; se detecta por comportamiento o anomalía.
```

### APT

Una **APT** (Advanced Persistent Threat, amenaza persistente avanzada) es un actor de amenazas, normalmente un grupo patrocinado por un Estado o muy bien financiado, que ataca un objetivo concreto con técnicas sofisticadas y se mantiene oculto dentro de la red durante meses o años para espiar o sabotear.

Cada palabra significa algo:

- Advanced: tiene recursos para desarrollar herramientas propias, usar zero days y combinar varias técnicas.
- Persistent: no busca entrar y salir rápido, sino quedarse; si lo expulsan, vuelve a intentarlo.
- Threat: tiene intención y capacidad; es un adversario humano con objetivos, no un malware automático.

Analogía: no es el carterista que roba y huye, es el espía que consigue trabajo en la empresa y pasa dos años copiando documentos sin que nadie lo note.

Contexto con otros actores de amenazas: un script kiddie usa herramientas ajenas sin entenderlas; un grupo criminal busca dinero rápido (ransomware); un hacktivista busca notoriedad para una causa; un insider ya está dentro. La APT se distingue por la combinación de objetivo concreto, recursos y paciencia.

Ejemplo: un informe público de 2013 documentó cómo un grupo vinculado a un Estado mantuvo acceso a la red de más de 140 organizaciones durante un promedio de casi un año cada una, sacando terabytes de datos. MITRE ATT&CK mantiene fichas de más de cien de estos grupos, con las técnicas que se les han observado.

Señales: no hay una sola alerta ruidosa, sino pequeños indicios sostenidos, como cuentas que inician sesión a horas raras, tráfico saliente constante y pequeño hacia el mismo destino, o herramientas legítimas del sistema usadas de forma inusual.

Defensas: monitoreo continuo y threat hunting, segmentación, mínimo privilegio, MFA, inteligencia de amenazas sobre los grupos que atacan a tu sector, y planes de respuesta preparados. Los marcos para seguirlas están en [`13-frameworks-de-amenazas`](../13-frameworks-de-amenazas/).

> [!NOTE]
> APT describe a quién te enfrentas (un actor), no qué herramienta usa; una APT puede usar phishing, zero days o malware común según le convenga.

```
Zero day   → vulnerabilidad sin parche; nadie tiene firma todavía.
Conocida   → documentada; firmas e indicadores la atrapan.
Desconocida→ nueva o modificada; solo el comportamiento la delata.
APT        → actor con recursos, objetivo concreto y permanencia larga.
```

## Recursos para aprender y practicar

### Videos

- [An Overview of Malware](https://www.youtube.com/watch?v=-eZs8wjjGGE) — Professor Messer; panorama de los tipos de malware (Learn how Malware works and Types).
- [Other Malware Types](https://www.youtube.com/watch?v=nu27ovJ5rqw) — Professor Messer; keyloggers, rootkits, logic bombs y demás.
- [Phishing](https://www.youtube.com/watch?v=9SD6DRCKZFU) — Professor Messer; phishing, vishing, smishing, whaling y cómo reconocerlos.
- [Impersonation](https://www.youtube.com/watch?v=X3yoNAVuKwA) — Professor Messer; pretexting y suplantación.
- [Watering Hole Attacks](https://www.youtube.com/watch?v=z413PV6l_Ys) — Professor Messer; el ataque del abrevadero con casos reales.
- [Password Attacks](https://www.youtube.com/watch?v=-ZfbifHwEVE) — Professor Messer; fuerza bruta, diccionario y password spraying.
- [Zero-day Vulnerabilities](https://www.youtube.com/watch?v=FDFxGLnZtoY) — Professor Messer; qué es un zero day y por qué es tan valioso.
- [Threat Actors](https://www.youtube.com/watch?v=6xUH0t6ugIM) — Professor Messer; APT, crimen organizado, hacktivistas e insiders.
- [Finding WEIRD Typosquatting Websites](https://www.youtube.com/watch?v=h0_L4BApOdA) — John Hammond; cómo se ven los dominios de typosquatting en la práctica.

El curso completo de Security+ SY0-701 de Professor Messer, ordenado por objetivo del examen, está en [professormesser.com](https://www.professormesser.com/security-plus/sy0-701/sy0-701-video/sy0-701-comptia-security-plus-course/).

### Lectura y documentación

- [MITRE ATT&CK T1566 Phishing](https://attack.mitre.org/techniques/T1566/) — sub-técnicas, ejemplos de grupos y mitigaciones.
- [MITRE ATT&CK T1189 Drive-by Compromise](https://attack.mitre.org/techniques/T1189/) — incluye los watering holes.
- [MITRE ATT&CK T1110 Brute Force](https://attack.mitre.org/techniques/T1110/) y [T1110.003 Password Spraying](https://attack.mitre.org/techniques/T1110/003/) — detección y mitigación de cada variante.
- [MITRE ATT&CK T1583.001 Domains](https://attack.mitre.org/techniques/T1583/001/) — cómo registran dominios parecidos los atacantes.
- [MITRE ATT&CK T1195 Supply Chain Compromise](https://attack.mitre.org/techniques/T1195/) — typosquatting de paquetes y otras vías.
- [MITRE ATT&CK Groups](https://attack.mitre.org/groups/) — fichas de APT y otros grupos con sus técnicas.
- [Mandiant APT1 report](https://www.mandiant.com/resources/reports/apt1-exposing-one-chinas-cyber-espionage-units) — el informe clásico sobre una APT, del que sale el ejemplo de la sección APT.
- [CISA: Advanced Persistent Threats](https://www.cisa.gov/topics/cyber-threats-and-advisories/advanced-persistent-threats) — avisos sobre actores estatales.
- [CISA Known Exploited Vulnerabilities](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) — vulnerabilidades explotadas activamente; muestra cuántas fueron zero days.
- [Google Project Zero: 0day "In the Wild"](https://googleprojectzero.blogspot.com/p/0day.html) — registro de zero days detectados en uso real.
- [CISA: Recognize and report phishing](https://www.cisa.gov/secure-our-world/recognize-and-report-phishing) — señales de phishing para usuarios.
- [NIST SP 800-63B](https://pages.nist.gov/800-63-4/sp800-63b.html) — recomendaciones oficiales sobre contraseñas, bloqueo y listas de contraseñas filtradas.
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) y [Blocking Brute Force Attacks](https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks) — defensas contra ataques de contraseñas.

### Práctica

- [Google Phishing Quiz](https://phishingquiz.withgoogle.com/) — distinguir correos legítimos de phishing; practica las señales de Phishing.
- [TryHackMe: Phishing Emails 1](https://tryhackme.com/room/phishingemails1tryoe) — analizar cabeceras y contenido de correos reales de phishing.
- [TryHackMe: MAL: Malware Introductory](https://tryhackme.com/room/malmalintroductory) y [Malware Classification](https://tryhackme.com/room/malwareclassification) — gratis; tipos de malware y primeros pasos de análisis en entorno seguro.
- [TryHackMe: Brute It](https://tryhackme.com/room/bruteit) y [Hydra](https://tryhackme.com/room/hydra) — gratis; fuerza bruta y diccionario contra máquinas de laboratorio, para ver qué rastro deja en los logs.
- [dnstwist](https://dnstwist.it/) ([código](https://github.com/elceef/dnstwist)) — genera las variantes de typosquatting de un dominio; pruébalo con el dominio de tu universidad o empresa y mira cuántas están registradas.
- [Have I Been Pwned](https://haveibeenpwned.com/) — comprueba si tu correo aparece en filtraciones; enseña por qué funciona el credential stuffing.
- [CVE.org](https://www.cve.org/) — busca una vulnerabilidad famosa y sigue su historia desde zero day hasta parche.
- Ejercicio en casa: con los logs de autenticación de tu propio equipo o servidor de laboratorio (`journalctl -u ssh` o el visor de eventos con ID 4625), cuenta fallos por cuenta y cuentas distintas por IP, y decide si el patrón se parece más a fuerza bruta o a spray.

## Cuadro resumen

Todo lo visto, en una línea por término.

Learn how Malware works and Types

```
Malware      → software creado para dañar, robar o tomar control sin permiso.
Virus        → se pega a un archivo y necesita que lo ejecuten para propagarse.
Gusano       → se propaga solo por la red sin intervención humana.
Troyano      → se disfraza de programa útil para que la víctima lo instale.
Ransomware   → cifra los datos y pide rescate; borra copias para impedir la recuperación.
Rootkit      → se esconde en lo profundo del sistema para ocultarse a sí mismo y a otros.
Spyware      → espía la actividad del usuario y la envía al atacante.
Keylogger    → registra lo que se teclea para robar contraseñas.
Botnet       → red de equipos infectados controlados a distancia por el atacante.
Fileless     → vive en memoria y usa herramientas legítimas del sistema; no deja ejecutable.
```

Attack Types and Differences

```
Social Engineering → manipular personas en vez de romper sistemas.
Phishing           → engaño por correo suplantando a alguien de confianza.
Whishing (vishing) → phishing por llamada de voz.
Whaling            → phishing dirigido a altos directivos.
Smishing           → phishing por SMS o mensajería.
Spam vs Spim       → correo basura vs mensajería instantánea basura.
Shoulder Surfing   → mirar por encima del hombro para ver datos o contraseñas.
Tailgating         → colarse detrás de alguien autorizado por una puerta controlada.
Dumpster Diving    → buscar información útil en la basura de la víctima.
Zero day           → vulnerabilidad sin parche disponible; sin firma posible.
Reconnaissance     → recolectar información del objetivo; pasivo no toca, activo sí deja rastro.
Impersonation      → fingir ser otra persona o rol con un pretexto creíble.
Watering hole      → comprometer un sitio que el grupo objetivo ya visita.
Drive-by           → infección con solo visitar la página.
Typosquatting      → dominios o paquetes casi iguales a los legítimos.
Fuerza bruta       → muchas contraseñas contra una cuenta; la frena el bloqueo.
Password spray     → pocas contraseñas comunes contra muchas cuentas; esquiva el bloqueo.
```

Understand Threat Classification

```
Zero Day    → amenaza desconocida por definición; se defiende con capas y comportamiento.
Conocida    → documentada; firmas e indicadores la atrapan.
Desconocida → nueva o modificada; solo el comportamiento la delata.
APT         → actor con recursos, objetivo concreto y permanencia larga.
```
