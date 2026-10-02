# Frameworks de amenazas

## Conceptos previos

- Adversario (threat actor): la persona o grupo que ataca; puede ser un criminal que busca dinero, un Estado que espía o un empleado descontento.
- Intrusión: un ataque que logró entrar, o intentó entrar, a un sistema que no le pertenece.
- TTPs (Tactics, Techniques and Procedures): el "cómo" de un atacante; la táctica es el objetivo de un paso, la técnica es la forma de lograrlo y el procedimiento es la receta concreta que usa ese grupo.
- IOC (Indicator of Compromise): un dato observable que delata una intrusión ya ocurrida (un hash, una IP, un dominio).
- SOC (Security Operations Center): el equipo que vigila los sistemas de la organización y responde a las alertas. Detalle en [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).
- SIEM: el sistema que junta los logs de toda la organización en un solo lugar y permite buscarlos y generar alertas. Detalle en [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).
- EDR (Endpoint Detection and Response): el agente instalado en cada equipo que registra procesos, archivos y conexiones y puede aislar la máquina.
- C2 (Command and Control): el canal por el que el malware ya instalado recibe órdenes del atacante y le devuelve datos.
- Phishing: correo o mensaje falso que engaña a la víctima para que abra un adjunto, haga clic o entregue credenciales. Detalle en [`11-ataques-y-amenazas`](../11-ataques-y-amenazas/).
- Exploit: código que aprovecha una vulnerabilidad (un fallo de software) para hacer algo que el sistema no debería permitir.
- Telemetría: los datos que los sistemas generan sobre sí mismos (logs, eventos de proceso, flujos de red) y que el defensor puede consultar.
- Blue team: el equipo defensor; red team: el equipo que simula al atacante con permiso.

## Cyber Kill Chain

### Cyber Kill Chain

La **Cyber Kill Chain** es un modelo de fases que describe, en 7 pasos ordenados, lo que un atacante tiene que completar para lograr su objetivo dentro de una red. Lockheed Martin la publicó en 2011 (Hutchins, Cloppert y Amin, "Intelligence-Driven Computer Network Defense Informed by Analysis of Adversary Campaigns and Intrusion Kill Chains"), tomando el concepto militar de "kill chain": la secuencia encontrar, fijar, rastrear, apuntar, atacar y evaluar.

Por qué existe: antes, la defensa se pensaba como "detectar el malware y limpiarlo". El problema es que cuando el antivirus salta, el atacante ya lleva días dentro. La Kill Chain obliga a ver la intrusión como un proceso con etapas previas, y cada etapa es una oportunidad de detectarlo antes de que llegue al daño. El enfoque se llama defensa guiada por inteligencia (intelligence-driven defense): aprender de cada intrusión, aunque haya fallado, para cortar la siguiente más temprano.

Analogía: un robo a un banco. Los ladrones vigilan el edificio, consiguen herramientas, entran por una puerta, abren la bóveda, se comunican por radio con el chofer y se llevan el dinero. La policía no necesita esperar a la bóveda: un guardia que nota a alguien fotografiando la entrada ya rompió el plan en el paso 1.

Las 7 fases, en orden:

```
 1. Reconnaissance          investigar a la víctima
 2. Weaponization           armar el paquete de ataque (exploit + payload)
 3. Delivery                hacer llegar el paquete a la víctima
 4. Exploitation            ejecutar el código aprovechando una vulnerabilidad
 5. Installation            dejar una puerta trasera persistente
 6. Command and Control     abrir el canal remoto de órdenes (C2)
 7. Actions on Objectives   hacer aquello a lo que vino: robar, cifrar, sabotear
```

Qué hace el atacante en cada fase y cómo corta el defensor:

1. **Reconnaissance (reconocimiento)**: el atacante recolecta correos de empleados en LinkedIn, subdominios en registros DNS públicos, puertos abiertos con escaneos. Corte: reducir lo que se expone (quitar banners de versión, no publicar organigramas con correos), y vigilar los logs del servidor web y del firewall buscando escaneos y rastreos inusuales. Es la fase con menos visibilidad para el defensor, porque gran parte ocurre fuera de su red.
2. **Weaponization (armado)**: el atacante junta un exploit con un payload (el código que quiere ejecutar), por ejemplo un documento de Word con una macro que descarga un troyano. Ocurre en la máquina del atacante, así que el defensor no lo ve. Corte indirecto: analizar los artefactos de ataques pasados (qué constructor de macros usan, qué metadatos dejan) para escribir firmas que reconozcan el arma cuando llegue.
3. **Delivery (entrega)**: el paquete llega por correo, por una web comprometida (watering hole), por un USB o por un servicio expuesto. Corte: filtro de correo con sandbox, SPF/DKIM/DMARC, bloqueo de adjuntos con macros, proxy web, deshabilitar autorun de USB, formar al usuario para que reporte. Es la primera fase donde el defensor tiene evidencia propia.
4. **Exploitation (explotación)**: el código se ejecuta aprovechando una vulnerabilidad o la acción del usuario (habilitar macros). Corte: parches al día, deshabilitar macros de Internet por política, protección contra exploits del EDR (DEP, ASLR), mínimo privilegio para que lo ejecutado no tenga permisos de administrador.
5. **Installation (instalación)**: el malware se queda: crea un servicio, una tarea programada, una clave `Run` del registro o un web shell. Corte: lista blanca de aplicaciones (application allowlisting), EDR que alerta ante nuevos servicios o claves de autoarranque, monitoreo de integridad de archivos en servidores web.
6. **Command and Control (C2)**: el equipo infectado contacta al servidor del atacante, normalmente por HTTPS o DNS para mezclarse con el tráfico normal, y queda a la espera de órdenes. Corte: proxy con categorías y bloqueo de dominios recién registrados, filtrado de salida en el firewall (solo los puertos y destinos necesarios), detección de beaconing (conexiones a intervalos regulares, por ejemplo cada 60 segundos), sinkhole DNS de dominios maliciosos conocidos.
7. **Actions on Objectives (acciones sobre el objetivo)**: robo de datos, movimiento a otros equipos, cifrado con ransomware, destrucción. Corte: segmentación de red, DLP (prevención de fuga de datos), alertas por volumen de salida anómalo, copias de seguridad fuera de línea, respuesta a incidentes rápida (ver [`16-respuesta-a-incidentes-y-forense`](../16-respuesta-a-incidentes-y-forense/)).

Ejemplo: un SOC recibe 200 correos de phishing en una semana. 190 los bloquea el filtro de correo (corte en Delivery). De los 10 que llegan, 8 usuarios los reportan sin abrirlos. Un usuario abre el adjunto, pero la política que deshabilita macros de Internet impide la ejecución (corte en Exploitation). El último usuario usa una máquina sin esa política; el troyano se instala y a los 5 minutos empieza a contactar `update-cdn-sync[.]com` cada 60 segundos. El proxy bloquea el dominio porque tiene 3 días de registrado (corte en C2). El atacante tuvo que ganar 7 veces seguidas; el defensor ganó con cualquiera de esas barreras.

```
Hora      Fuente   Evento
09:14:02  mail     adjunto factura_0423.docm entregado a usuario B
09:16:40  EDR      WINWORD.EXE -> powershell.exe -enc JABjAD0A...
09:16:45  EDR      nueva clave HKCU\...\Run\Updater -> C:\Users\B\AppData\upd.exe
09:21:45  proxy    DENY upd.exe -> update-cdn-sync.com (categoria: newly registered)
09:22:45  proxy    DENY upd.exe -> update-cdn-sync.com
```

Las líneas 2, 3 y 4 son las fases 4, 5 y 6 vistas desde la telemetría: el defensor que sabe leer el log en clave de Kill Chain sabe exactamente en qué paso va el atacante y qué le falta.

La matriz de cursos de acción del paper original propone seis verbos para cada fase, conocidos como las "6 D": Detect (detectar: un IDS ve el escaneo), Deny (negar: el firewall bloquea el puerto), Disrupt (interrumpir: el antivirus mata el proceso a mitad de ejecución), Degrade (degradar: limitar el ancho de banda de salida para que la exfiltración tarde días), Deceive (engañar: un honeypot o credenciales trampa), Destroy (destruir: acción ofensiva contra la infraestructura del atacante, reservada a fuerzas del orden o militares, nunca a una empresa).

Límites conocidos: el modelo nació pensando en malware que entra desde fuera, y se queda corto con amenazas internas, ataques a aplicaciones web o cloud sin malware, y con lo que pasa después de la primera máquina (moverse por la red queda aplastado dentro de la fase 7). Además, críticos señalan que hace mucho énfasis en el perímetro, justo donde el atacante tiene más opciones.

> [!WARNING]
> Weaponization ocurre en la máquina del atacante: el defensor no puede ver esa fase, solo deducirla analizando el arma cuando llega. En el examen, "qué fase no tiene visibilidad directa" es Weaponization (y casi toda Reconnaissance).

```
Reconnaissance → el atacante investiga; corte: reducir exposición, vigilar escaneos.
Weaponization → arma exploit + payload; corte: firmas a partir de armas pasadas.
Delivery → el arma llega (correo, web, USB); corte: filtro de correo, proxy, formación.
Exploitation → el código se ejecuta; corte: parches, macros bloqueadas, mínimo privilegio.
Installation → persistencia; corte: allowlisting, EDR sobre autoarranques.
Command and Control → canal remoto; corte: filtrado de salida, detección de beaconing.
Actions on Objectives → robo, cifrado, sabotaje; corte: segmentación, DLP, backups.
```

### Al defensor le basta cortar un eslabón

> [!IMPORTANT]
> El atacante necesita completar las 7 fases en orden; el defensor gana si rompe cualquiera de ellas, y gana más barato cuanto antes la rompe.

La asimetría de la Kill Chain es una idea que invierte el dicho "el atacante solo necesita acertar una vez": dentro de una intrusión concreta, es el atacante quien necesita acertar en cada paso, y el defensor quien solo necesita un acierto.

```
                1     2     3     4     5     6     7
Atacante:      gana  gana  gana  gana  gana  gana  gana   → logra el objetivo
Defensor A:    -     -     CORTA                         → intrusión fallida
Defensor B:    -     -     -     -     -     CORTA       → intrusión fallida, pero con limpieza
```

Las dos filas de defensores terminan igual, pero cortar en Delivery cuesta un correo en cuarentena, y cortar en C2 cuesta reimaginar una máquina e investigar qué más hizo.

Analogía: un dominó de 7 fichas. Basta sacar una para que la última no caiga.

Límite: esto vale para una intrusión; en el agregado, el atacante intenta muchas veces y solo necesita que una cadena completa salga bien. Por eso la defensa se construye en capas (defense in depth): varias barreras por fase, no una sola. Detalle en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

```
Cadena completa → requisito del atacante para ganar.
Un eslabón roto → suficiente para el defensor; más temprano, más barato.
Defense in depth → varias barreras por fase porque el atacante reintenta.
```

## Understand Frameworks

### Diamond Model

El **Diamond Model of Intrusion Analysis** es un modelo de análisis que describe cada evento de una intrusión con cuatro elementos conectados entre sí: quién ataca, con qué, desde dónde y a quién. Lo publicaron Sergio Caltagirone, Andrew Pendergast y Christopher Betz en 2013.

Por qué existe: la Kill Chain dice en qué paso va un ataque, pero no ayuda a relacionar ataques entre sí ni a responder "¿esto que vimos hoy es el mismo grupo que el mes pasado?". El Diamond Model da una forma fija de anotar cada evento para que los eventos se puedan comparar, agrupar y usar como punto de partida para buscar más.

Los 4 vértices:

```
                 Adversary
                 (quién)
                /        \
               /          \
      Infrastructure ---- Capability
      (desde dónde)        (con qué)
               \          /
                \        /
                  Victim
                 (a quién)
```

- **Adversary (adversario)**: quién está detrás. Se distingue el operador (quien ejecuta el ataque con el teclado) del cliente (quien se beneficia y lo encarga); pueden ser personas distintas. Suele ser el vértice que menos se conoce al principio.
- **Capability (capacidad)**: las herramientas y técnicas que usó: un malware, un exploit, un script de PowerShell, una técnica de robo de credenciales. El conjunto completo de capacidades de un adversario se llama su arsenal.
- **Infrastructure (infraestructura)**: los medios físicos o lógicos por los que la capacidad llega a la víctima: direcciones IP, dominios, servidores de correo, servidores C2, USB. Tipo 1 es la infraestructura que el adversario controla o posee; tipo 2 es la de intermediarios (máquinas comprometidas de terceros, servicios en la nube gratuitos) que sirve para ocultar su origen. También existen los proveedores de servicio (ISP, registradores de dominios) que la hacen posible.
- **Victim (víctima)**: el objetivo. Se distingue la persona víctima (la organización, el sector, las personas) de los activos víctima (la máquina, la cuenta, la red atacada).

Las líneas que unen los vértices son relaciones: el adversario desarrolla una capacidad, la capacidad usa una infraestructura, la infraestructura conecta con la víctima. Un evento solo está completo cuando se sabe algo de los cuatro.

Meta-características (meta-features): datos que acompañan a cada evento sin ser vértices.

- Timestamp: cuándo empezó y terminó el evento; permite ordenar eventos y ver horarios del atacante.
- Phase: en qué fase de una cadena de ataque cae el evento (aquí se conecta con la Kill Chain).
- Result: si tuvo éxito, falló o se desconoce, y qué afectó (confidencialidad, integridad, disponibilidad).
- Direction: hacia dónde fue: de la infraestructura a la víctima, de la víctima a la infraestructura, en ambos sentidos, etc. Un correo entrante y una exfiltración saliente tienen direcciones opuestas.
- Methodology: la clase general de actividad (phishing, escaneo de puertos, denegación de servicio).
- Resources: lo que el adversario necesitó: software, conocimiento, dinero, acceso, hardware.

El modelo extendido añade dos ejes: el socio-político (la relación de necesidad entre adversario y víctima: por qué este atacante quiere a esta víctima; dinero, espionaje, ideología) y el tecnológico (la relación entre capacidad e infraestructura: qué tecnología permite que esa herramienta use esa infraestructura).

El uso clave es el pivoting (pivoteo): partir de un vértice conocido para descubrir los demás. Varios eventos encadenados en orden forman un activity thread (hilo de actividad), y los hilos que se parecen se agrupan en activity groups, que es como un analista llega a decir "esto es la misma campaña".

Analogía: la ficha de un detective sobre un robo. Sospechoso (adversario), arma (capacidad), auto de huida (infraestructura) y víctima. Si aparece el mismo auto en otro robo, el detective pivotea desde ese vértice y conecta los dos casos.

Ejemplo: un SOC recibe un correo de phishing. Anota: víctima = 3 contadores del área de finanzas; capacidad = documento Excel con macro que descarga un troyano; infraestructura = dominio `factura-pagos[.]net` con IP `203.0.113.50`; adversario = desconocido. Pivotea desde la IP en un servicio de DNS pasivo y encuentra 12 dominios más que resolvieron a la misma IP en los últimos 90 días, con temas de facturas. Bloquea los 12 de antemano y busca en el proxy si algún empleado visitó alguno. Así un solo correo se convierte en el mapa de una campaña.

> [!TIP]
> Si la pregunta es "qué relaciona a estos dos incidentes", la respuesta va por el Diamond Model; si es "en qué paso estaba el atacante", va por la Kill Chain; si es "qué técnica exacta usó", va por ATT&CK.

```
Adversary → quién ataca (operador y cliente).
Capability → con qué ataca (malware, exploit, técnica).
Infrastructure → desde dónde llega (IP, dominio, C2); tipo 1 propia, tipo 2 de intermediarios.
Victim → a quién ataca (persona víctima y activos víctima).
Meta-features → timestamp, phase, result, direction, methodology, resources.
Pivoting → partir de un vértice conocido para descubrir los otros.
```

### Kill Chain

La **Kill Chain**, como nodo de "Understand Frameworks", es la familia de modelos de cadena de ataque que ordenan una intrusión en fases secuenciales; la versión de Lockheed Martin, explicada completa en [Cyber Kill Chain](#cyber-kill-chain), es la original, y la **Unified Kill Chain** es su revisión más usada hoy.

Lo que añade este nodo es cómo encaja la Kill Chain entre los otros dos frameworks y por qué hizo falta una versión unificada. La de Lockheed Martin tiene dos huecos: aplasta todo lo que pasa después de la primera máquina en una sola fase (Actions on Objectives) y asume que el ataque entra con malware desde fuera. En una intrusión real, la mayor parte del tiempo el atacante está dentro, moviéndose de equipo en equipo.

La **Unified Kill Chain** (UKC) es un modelo de 18 fases que publicó Paul Pols en 2017 combinando la Kill Chain de Lockheed Martin con las tácticas de MITRE ATT&CK, y que reconoce que un atacante repite ciclos dentro de la red. Agrupa las 18 fases en tres ciclos:

```
IN (Initial Foothold: conseguir el primer pie dentro)
  Reconnaissance, Resource Development, Delivery, Social Engineering,
  Exploitation, Persistence, Defense Evasion, Command & Control
        |
        v
THROUGH (Network Propagation: moverse por la red)
  Pivoting, Discovery, Privilege Escalation, Execution,
  Credential Access, Lateral Movement      <-- se repite por cada máquina nueva
        |
        v
OUT (Action on Objectives: cumplir el objetivo)
  Collection, Exfiltration, Impact, Objectives
```

8 + 6 + 4 = 18 fases. La diferencia práctica: en la UKC el ciclo "Through" puede darse muchas veces, y el atacante puede volver a "In" desde una máquina interna hacia otra red. Nótese que la UKC usa nombres de táctica de ATT&CK de 2017 (por ejemplo "Defense Evasion"), no los de la versión actual de ATT&CK (ver el nodo siguiente).

Analogía: la Kill Chain original es el plano de cómo un ladrón entra a una casa; la UKC también describe cómo, ya dentro, recorre cuarto por cuarto buscando la caja fuerte, y que puede salir por el patio a la casa vecina.

Ejemplo: un ransomware moderno entra por phishing (In), pasa 10 días robando contraseñas y saltando de 1 a 40 servidores (Through, repetido 40 veces) y al final cifra los 40 de golpe (Out). En la Kill Chain original esos 10 días caben enteros en la fase 7; en la UKC cada salto tiene su fase y su oportunidad de detección.

```
Cyber Kill Chain (Lockheed Martin, 2011) → 7 fases lineales, foco en entrar.
Unified Kill Chain (Pols, 2017) → 18 fases en 3 ciclos In/Through/Out, foco también en moverse dentro.
```

### ATT&CK

**MITRE ATT&CK** (Adversarial Tactics, Techniques, and Common Knowledge) es una base de conocimiento pública y gratuita que cataloga los comportamientos reales de los atacantes, organizados en tácticas y técnicas, a partir de intrusiones observadas y documentadas. La mantiene MITRE, una organización sin fines de lucro de Estados Unidos, desde 2013, y se consulta en attack.mitre.org.

Por qué existe: la Kill Chain dice "fase 5, Installation", pero no dice de cuántas formas se puede instalar algo ni cómo detectar cada una. ATT&CK baja al detalle: lista cada técnica concreta, qué grupos la usaron, cómo se detecta y cómo se mitiga. Le da a defensores, red teams y proveedores un vocabulario común: "T1566" significa lo mismo para todos.

Analogía: la Kill Chain es el índice de un libro de recetas (entradas, platos fuertes, postres); ATT&CK es el libro entero, con cada receta, sus ingredientes y quién la cocina.

Hay tres matrices: Enterprise (Windows, macOS, Linux, redes, nube, contenedores, identidad), Mobile (Android, iOS) e ICS (sistemas de control industrial). La que más se estudia es Enterprise.

Estructura:

- **Tácticas (tactics)**: el "por qué" de un paso, el objetivo táctico del atacante. Son las columnas de la matriz. ID con formato `TA####`.
- **Técnicas (techniques)**: el "cómo" se logra esa táctica. ID `T####`. Una técnica puede servir para varias tácticas.
- **Sub-técnicas (sub-techniques)**: una variante más específica de una técnica. ID `T####.###`.
- **Procedimientos (procedures)**: cómo lo hizo un grupo concreto en un caso real, documentado en la página de la técnica.
- **Grupos (groups)**: actores de amenaza conocidos con sus técnicas observadas. ID `G####`; por ejemplo G0007 es APT28.
- **Software**: malware y herramientas, ID `S####`. **Mitigations**: contramedidas, ID `M####`. **Campaigns**: campañas concretas, ID `C####`.

La matriz Enterprise vigente (ATT&CK v19.2, publicada el 28 de abril de 2026) tiene 15 tácticas, en este orden:

```
 1. TA0043  Reconnaissance          investigar a la víctima
 2. TA0042  Resource Development    conseguir infraestructura, cuentas, herramientas
 3. TA0001  Initial Access          entrar a la red
 4. TA0002  Execution               ejecutar código
 5. TA0003  Persistence             quedarse aunque reinicien
 6. TA0004  Privilege Escalation    conseguir más permisos
 7. TA0005  Stealth                 esconderse y parecer actividad normal
 8. TA0112  Defense Impairment      romper o apagar las defensas
 9. TA0006  Credential Access       robar contraseñas, hashes, tokens
10. TA0007  Discovery               conocer la red por dentro
11. TA0008  Lateral Movement        saltar a otros equipos
12. TA0009  Collection              juntar los datos de interés
13. TA0011  Command and Control     comunicarse con el equipo infectado
14. TA0010  Exfiltration            sacar los datos
15. TA0040  Impact                  cifrar, borrar, interrumpir
```

Hasta la v18 eran 14 tácticas: la v19 dividió la antigua Defense Evasion (TA0005) en dos. Stealth conserva el ID TA0005 y agrupa lo que busca pasar desapercibido sin tocar las defensas (ofuscar archivos T1027, enmascarar nombres T1036, inyectar en procesos T1055, borrar indicadores T1070). Defense Impairment es nueva (TA0112) y agrupa lo que ataca a las propias defensas: desactivar o modificar herramientas de seguridad como EDR, antivirus o agentes de logs (T1685), desactivar el firewall (T1686), subvertir controles de confianza como la firma de código (T1553).

> [!WARNING]
> Muchos cursos, preguntas de certificación y salas de práctica aún dicen "14 tácticas" e incluyen Defense Evasion: eso era cierto hasta ATT&CK v18. La versión vigente (v19.2) tiene 15, con Stealth y Defense Impairment en su lugar.

Ejemplo de técnica con sub-técnicas, T1566 Phishing (táctica Initial Access):

```
T1566        Phishing
T1566.001    Spearphishing Attachment    (adjunto malicioso)
T1566.002    Spearphishing Link          (enlace malicioso)
T1566.003    Spearphishing via Service   (por redes sociales, mensajería)
T1566.004    Spearphishing Voice         (por llamada telefónica)
```

Su página lista mitigaciones como M1054 Software Configuration (SPF, DKIM, DMARC), M1017 User Training y M1049 Antivirus/Antimalware, y estrategias de detección como correlacionar un correo con adjunto seguido de un proceso hijo sospechoso de Office.

Cómo lo usa el defensor:

- Mapear alertas: cada regla del SIEM se etiqueta con su técnica (`T1059.001` PowerShell), y así se sabe qué cubre la detección.
- Medir cobertura: cruzar todas las técnicas contra las reglas propias muestra los huecos.
- Priorizar por amenaza: si el sector de la organización lo atacan 3 grupos concretos, se mira qué técnicas usan esos grupos y se cubren esas primero.
- Emular adversarios: el red team ejecuta las técnicas de un grupo real y el blue team verifica si las ve.

El **ATT&CK Navigator** es una aplicación web de MITRE que muestra la matriz como un mapa y permite colorear técnicas y guardar capas (layers) en JSON. Un uso típico: una capa con las técnicas de APT28 en rojo, otra con las técnicas que el SOC detecta en verde; superponerlas muestra en un vistazo qué técnicas de ese grupo nadie vigila.

Ejemplo: un SOC tiene 40 reglas de detección. Al etiquetarlas, descubre que 25 caen en Execution y Initial Access, y ninguna en Exfiltration ni en Lateral Movement. Con el Navigator lo ve como dos columnas vacías y prioriza escribir reglas para T1021 (Remote Services) y T1041 (Exfiltration Over C2 Channel).

Ojo al leer un mapa de cobertura: muchas técnicas tienen procedimientos tan distintos que una sola regla nunca cubre la técnica entera. Una casilla verde significa "tengo al menos algo", no "estoy protegido".

```
Táctica (TA####) → el objetivo del paso; columna de la matriz; 15 en Enterprise v19.
Técnica (T####) → cómo se logra la táctica.
Sub-técnica (T####.###) → variante específica de una técnica.
Procedimiento → cómo lo hizo un grupo concreto en un caso real.
Grupo (G####) → actor de amenaza con sus técnicas observadas.
Navigator → herramienta web para colorear la matriz en capas y ver cobertura.
Stealth (TA0005) → esconderse y parecer normal.
Defense Impairment (TA0112) → romper o apagar las defensas.
```

### Los tres frameworks responden preguntas distintas

> [!IMPORTANT]
> Kill Chain, Diamond Model y ATT&CK no compiten: cada uno responde una pregunta distinta sobre la misma intrusión, y un análisis completo usa los tres a la vez.

La relación entre los frameworks es una división del trabajo: la Kill Chain ordena en el tiempo, el Diamond Model relaciona entidades, y ATT&CK nombra comportamientos con precisión.

```
                 Kill Chain           Diamond Model            ATT&CK
Unidad           fase                 evento                   técnica
Granularidad     gruesa (7 o 18)      media (4 vértices)       fina (cientos de técnicas)
Responde         ¿en qué paso va?     ¿quién, con qué, desde   ¿cómo exactamente lo hizo?
                                      dónde, a quién?
Sirve para       elegir dónde cortar  agrupar y pivotear       mapear detecciones y huecos
```

La fila "Responde" es la que importa: tres preguntas distintas, ninguna sustituye a otra.

Ejemplo: un mismo correo de phishing se anota como fase 3 Delivery (Kill Chain), como evento con víctima = finanzas, capacidad = macro, infraestructura = dominio X (Diamond, con meta-característica Phase = Delivery) y como T1566.001 Spearphishing Attachment (ATT&CK).

Límite: ninguno de los tres detecta nada por sí solo; son formas de organizar lo que la telemetría ya muestra.

```
Kill Chain → cuándo: fase del ataque.
Diamond Model → quién/qué/dónde/a quién: relaciones entre eventos.
ATT&CK → cómo: técnica exacta con su ID.
```

## Basics of Threat Intel, OSINT

### Threat Intelligence

La **threat intelligence** (inteligencia de amenazas, o CTI, Cyber Threat Intelligence) es información sobre amenazas que fue recolectada, analizada y puesta en contexto para que alguien pueda tomar una decisión con ella. La diferencia con un simple dato es el análisis: "la IP 203.0.113.50" es un dato; "la IP 203.0.113.50 es un servidor C2 del grupo que ataca a bancos de la región desde hace 2 semanas, bloquéala" es inteligencia.

Por qué existe: un SOC recibe miles de alertas y no puede defender todo por igual. La inteligencia le dice qué amenazas son relevantes para su organización, sector y país, y así decide qué parchear primero, qué reglas escribir y qué alertas mirar con lupa.

Analogía: el pronóstico del tiempo. El dato es "presión atmosférica de 990 hPa"; la inteligencia es "mañana llueve en tu ciudad a las 3 p. m., lleva paraguas". Lo útil es lo que te dice qué hacer.

Se clasifica en cuatro tipos según quién la usa y a qué plazo:

- **Estratégica (strategic)**: visión de alto nivel para directivos y junta. Tendencias, riesgos por sector, motivaciones geopolíticas, impacto en el negocio. Formato: informes en lenguaje no técnico. Ejemplo: "los ataques de ransomware contra hospitales de la región crecieron al doble este año; conviene invertir en backups fuera de línea". Plazo: meses a años.
- **Táctica (tactical)**: las TTPs de los atacantes, para arquitectos de seguridad y equipos de detección. Ejemplo: "los grupos de ransomware actuales entran por VPN sin MFA y usan PsExec para moverse"; se traduce en reglas de detección y cambios de configuración. Plazo: semanas a meses.
- **Operacional (operational)**: detalles de una campaña o ataque concreto en curso o inminente: quién, cuándo, contra quién, con qué objetivo. Para el SOC y el equipo de respuesta. Ejemplo: "un grupo anunció en un foro que atacará a empresas de logística del país este fin de semana". Plazo: días.
- **Técnica (technical)**: los indicadores concretos y legibles por máquina: hashes, IPs, dominios, URLs, firmas. Para el SOC y las herramientas (SIEM, firewall, EDR). Ejemplo: una lista de 50 hashes SHA-256 y 10 dominios para bloquear hoy. Plazo: horas a días, porque caducan rápido.

> [!NOTE]
> Las fuentes no se ponen de acuerdo en el borde entre táctica, operacional y técnica: algunas llaman "táctica" a la lista de IOCs y reservan TTPs para la operacional. Lo estable es el eje: estratégica para decidir inversiones, técnica para alimentar máquinas, y las otras dos en medio.

Ejemplo de la misma amenaza en los cuatro tipos: a la junta se le dice "el phishing con facturas falsas es nuestro riesgo número 1 este año" (estratégica); al equipo de detección, "usan Excel con macros que lanzan PowerShell codificado" (táctica); al SOC, "la campaña empezó el lunes y apunta a finanzas" (operacional); al firewall, "bloquear `factura-pagos[.]net` y estos 4 hashes" (técnica).

```
Estratégica → tendencias y riesgo de negocio, para directivos; meses a años.
Táctica → TTPs de los atacantes, para detección y arquitectura; semanas a meses.
Operacional → campaña concreta en curso (quién, cuándo, contra quién), para SOC/IR; días.
Técnica → IOCs legibles por máquina (hash, IP, dominio), para herramientas; horas a días.
```

### Ciclo de inteligencia

El **ciclo de inteligencia** (intelligence lifecycle) es un proceso de 6 fases que convierte una necesidad de información en inteligencia útil y vuelve a empezar con lo aprendido. Viene de la inteligencia militar y gubernamental y se aplica igual en ciberseguridad.

```
     1. Direction ----------> 2. Collection
     (qué necesito saber)     (juntar datos)
           ^                        |
           |                        v
     6. Feedback               3. Processing
     (¿sirvió?)                (limpiar, normalizar)
           ^                        |
           |                        v
     5. Dissemination <------- 4. Analysis
     (entregar a quien         (sacar conclusiones
      decide)                   y recomendaciones)
```

1. **Direction / Planning (dirección)**: definir los requisitos de inteligencia: qué activos proteger, qué preguntas hay que responder. Ejemplo: "¿qué grupos atacan a bancos centroamericanos y por dónde entran?".
2. **Collection (recolección)**: juntar datos de fuentes internas (logs, alertas, incidentes pasados) y externas (feeds, OSINT, informes de proveedores, comunidades de intercambio).
3. **Processing (procesamiento)**: convertir los datos crudos a un formato usable: quitar duplicados, normalizar campos, traducir, cargarlos en una plataforma.
4. **Analysis (análisis)**: un analista los interpreta, los relaciona (aquí entran el Diamond Model y ATT&CK) y produce conclusiones con acciones recomendadas.
5. **Dissemination (difusión)**: entregar el producto a cada consumidor en su formato: informe ejecutivo a la junta, reglas al equipo de detección, IOCs al firewall.
6. **Feedback (retroalimentación)**: los consumidores dicen si les sirvió y qué les falta, y eso ajusta la dirección del siguiente ciclo.

Analogía: un periodista que investiga una nota. Decide el tema, entrevista y junta documentos, los ordena, escribe la historia, la publica y lee las reacciones para la siguiente nota.

Ejemplo: un SOC tiene como requisito "detectar ransomware antes del cifrado". Recolecta 20 informes públicos de incidentes de ransomware, los procesa extrayendo las técnicas ATT&CK de cada uno, analiza que 15 de 20 usaron RDP o VPN expuesta y 12 de 20 borraron las shadow copies, entrega 2 reglas nuevas al equipo de detección, y el feedback es que la regla de shadow copies dio 30 falsos positivos por semana por un software de backup legítimo. El siguiente ciclo afina eso.

Contraejemplo: una lista de 100.000 IPs maliciosas cargada directamente en el firewall no es inteligencia, porque se saltó dirección, análisis y feedback. El resultado típico son miles de alertas sin contexto y muchos falsos positivos.

```
Direction → definir qué hay que saber.
Collection → juntar datos internos y externos.
Processing → limpiar y normalizar.
Analysis → interpretar y recomendar.
Dissemination → entregar a cada consumidor en su formato.
Feedback → ajustar el siguiente ciclo.
```

### IOCs vs IOAs

Un **IOC** (Indicator of Compromise) es una evidencia observable que indica que un sistema ya fue comprometido; un **IOA** (Indicator of Attack) es un patrón de comportamiento que indica que un ataque está ocurriendo, sin importar qué herramienta concreta lo ejecute.

Por qué importa la diferencia: el IOC es reactivo y frágil. Mira el pasado ("este hash ya se vio en un ataque") y el atacante lo cambia en segundos. El IOA mira la intención ("Word lanzó PowerShell codificado que se conecta a Internet") y sirve aunque el atacante cambie de archivo, IP o dominio.

Analogía: un IOC es la huella digital de un ladrón conocido: sirve si vuelve el mismo ladrón con los mismos dedos. Un IOA es ver a alguien probando picaportes de autos de madrugada: no sabes quién es, pero sabes lo que está haciendo.

Ejemplos de IOC: hash SHA-256 de un archivo, IP de un servidor C2, dominio malicioso, URL de descarga, nombre de un archivo o de un mutex, clave de registro concreta, asunto de un correo de phishing.

Ejemplos de IOA:

- `WINWORD.EXE` crea un proceso `powershell.exe` con el parámetro `-enc` (código codificado en Base64).
- Un proceso que no es de sistema lee la memoria de `lsass.exe`, donde Windows guarda credenciales.
- Una cuenta inicia sesión en 30 servidores en 5 minutos.
- `vssadmin delete shadows /all` seguido de miles de archivos renombrados en un minuto (patrón de ransomware).

```
IOC  (lista de bloqueo)                 IOA  (regla de comportamiento)
sha256 = 9f86d081884c7d65...            parent = WINWORD.EXE
ip     = 203.0.113.50                   AND child = powershell.exe
domain = factura-pagos.net              AND cmdline contiene "-enc"
→ falla si el atacante recompila       → sigue funcionando con otro hash,
  o cambia de servidor                    otra IP y otro dominio
```

Criterio: los IOCs sirven para bloqueo rápido y para buscar hacia atrás ("¿alguien contactó esta IP en los últimos 30 días?"); los IOAs sirven para detectar lo nuevo. Un SOC maduro usa ambos.

```
IOC → evidencia de que ya hubo compromiso; concreto, reactivo, fácil de evadir.
IOA → comportamiento de un ataque en curso; independiente de la herramienta, más difícil de evadir.
```

### Pirámide del dolor

La **Pyramid of Pain** (pirámide del dolor) es un modelo que ordena los tipos de indicadores según cuánto le cuesta al atacante cambiarlos cuando el defensor los detecta y bloquea. La propuso David Bianco en 2013.

```
                  /\
                 /  \        TTPs                 Tough!
                /----\
               /      \      Tools                Challenging
              /--------\
             /          \    Network/Host         Annoying
            /  Artifacts \
           /--------------\
          /                \ Domain Names         Simple
         /------------------\
        /                    \ IP Addresses       Easy
       /----------------------\
      /                        \ Hash Values      Trivial
     /__________________________\
```

De abajo hacia arriba:

1. **Hash values (trivial)**: cambiar un solo byte del archivo da otro hash. El atacante lo hace en segundos, o su malware lo hace solo en cada infección.
2. **IP addresses (easy)**: un proxy, una VPN o un VPS nuevo de unos pocos dólares dan otra IP en minutos.
3. **Domain names (simple)**: registrar otro dominio cuesta poco dinero y algo de tiempo (registro, propagación DNS).
4. **Network/Host artifacts (annoying)**: rastros que deja la herramienta en la red o el equipo: un User-Agent raro, un patrón de URL del C2, una clave de registro, un nombre de servicio. Cambiarlos obliga al atacante a modificar y reconfigurar su herramienta.
5. **Tools (challenging)**: detectar la herramienta en sí (por su comportamiento, con reglas YARA o firmas genéricas) obliga al atacante a conseguir o escribir otra, y a aprender a usarla.
6. **TTPs (tough)**: detectar el comportamiento (por ejemplo "volcar la memoria de LSASS para robar credenciales", sea cual sea la herramienta) obliga al atacante a cambiar su forma de operar. Es lo más caro para él; en muchos casos, se rinde o va por otro objetivo.

Analogía: echar a un vendedor ambulante molesto. Si lo reconoces por la camisa (hash), se cambia de camisa. Si lo reconoces por la esquina donde se pone (IP), cambia de esquina. Si lo reconoces por cómo trabaja (TTP: se acerca a los que salen del banco), tendría que cambiar de oficio.

Ejemplo: tras un incidente, el SOC bloquea 1 hash, 1 IP y 1 dominio. Al día siguiente, el mismo grupo vuelve con otro hash, otra IP y otro dominio: los tres bloqueos sirvieron 24 horas. Entonces el SOC escribe una regla de comportamiento: "alertar cuando cualquier proceso que no sea un agente de backup conocido ejecute `vssadmin delete shadows`". Esa regla atrapa el siguiente intento aunque todo lo de abajo haya cambiado.

```
Hash values → trivial de cambiar.
IP addresses → fácil de cambiar.
Domain names → simple de cambiar.
Network/Host artifacts → molesto de cambiar; obliga a reconfigurar la herramienta.
Tools → difícil; obliga a cambiar de herramienta.
TTPs → lo más duro; obliga a cambiar de forma de operar.
```

### Cuanto más arriba en la pirámide, más le cuesta al atacante

> [!IMPORTANT]
> El valor de una detección se mide por lo que le cuesta al atacante evadirla: bloquear un hash le cuesta segundos; detectar su TTP le cuesta rehacer su operación.

El principio de la pirámide es una regla de priorización que conecta casi todo el tema: los IOAs viven arriba y los IOCs abajo, ATT&CK cataloga justamente la cima (las TTPs), y el threat hunting busca sobre todo comportamientos, no hashes.

```
                        Hash/IP/Dominio        Artifacts/Tools        TTPs
Tipo de indicador       IOC                    IOC/IOA                IOA
Dónde se cataloga       feeds de IOCs          YARA, firmas           ATT&CK
Costo de evasión        segundos a horas       días                   semanas a meses
Duración del bloqueo    horas a días           semanas                meses a años
```

La última fila es la que importa: la duración de la detección crece con el costo para el atacante.

Límite: la base no es inútil. Los IOCs son baratos de aplicar, se automatizan y sirven para buscar hacia atrás si ya hubo contacto. Lo que no se debe es quedarse solo en la base. Además, las detecciones de arriba son más caras de escribir y dan más falsos positivos, porque un administrador legítimo también usa PowerShell.

```
Base de la pirámide → barato para ambos; útil para bloqueo rápido y búsqueda hacia atrás.
Cima de la pirámide → caro para ambos; es lo que de verdad frena al atacante.
```

### STIX/TAXII

**STIX** (Structured Threat Information Expression) es un lenguaje estándar en JSON para describir inteligencia de amenazas, y **TAXII** (Trusted Automated Exchange of Intelligence Information) es un protocolo sobre HTTPS para transportar esa inteligencia entre organizaciones y herramientas. Ambos los mantiene OASIS; la versión vigente de los dos es la 2.1.

Por qué existen: si cada proveedor describe un indicador a su manera (un CSV, un PDF, un correo), compartir inteligencia exige copiar y pegar a mano. Con STIX todos hablan el mismo formato y con TAXII las máquinas se lo envían solas.

Analogía: STIX es el idioma y el formato de una carta (con remitente, destinatario y asunto en lugares fijos); TAXII es el servicio de correo que la entrega.

STIX 2.1 define objetos:

- SDOs (STIX Domain Objects), 18 tipos, entre ellos: Indicator, Malware, Threat Actor, Intrusion Set, Campaign, Attack Pattern (una técnica, se enlaza con su ID de ATT&CK), Infrastructure, Vulnerability, Tool, Course of Action, Identity, Report y Observed Data.
- SROs (STIX Relationship Objects): Relationship (por ejemplo "el malware X usa la infraestructura Y") y Sighting ("vimos este indicador en nuestra red").

Ejemplo de un Indicator en STIX 2.1:

```json
{
  "type": "indicator",
  "spec_version": "2.1",
  "id": "indicator--8e2e2d2b-17d4-4cbf-938f-98ee46b3cd3f",
  "created": "2026-09-01T10:00:00.000Z",
  "modified": "2026-09-01T10:00:00.000Z",
  "name": "Dominio C2 de campaña de facturas falsas",
  "indicator_types": ["malicious-activity"],
  "pattern": "[domain-name:value = 'factura-pagos.net']",
  "pattern_type": "stix",
  "valid_from": "2026-09-01T10:00:00Z"
}
```

TAXII 2.1 define dos servicios: Collections (un servidor guarda conjuntos de objetos STIX y los clientes los piden o publican, modelo petición-respuesta) y Channels (publicación-suscripción, definido en el estándar pero todavía sin especificar en detalle). Funciona sobre HTTPS, normalmente en el puerto 443, con autenticación.

```
cliente TAXII (SIEM/TIP)                       servidor TAXII (proveedor de CTI)
   GET /taxii2/                     ------->   discovery: qué API roots hay
   GET /api1/collections/           ------->   lista de collections
   GET /api1/collections/<id>/objects/?added_after=2026-09-01T00:00:00Z
                                    <-------   bundle STIX con indicadores nuevos
```

Ejemplo: una plataforma de inteligencia (TIP) como MISP u OpenCTI consulta cada hora un servidor TAXII de un CERT nacional, recibe 30 indicadores nuevos en STIX, y los pasa automáticamente al SIEM para buscar coincidencias en los logs de los últimos 30 días.

```
STIX → formato (JSON) para describir inteligencia: objetos y relaciones.
TAXII → protocolo (HTTPS) para transportar STIX entre sistemas.
SDO → objeto de dominio (Indicator, Malware, Threat Actor...).
SRO → objeto de relación (Relationship, Sighting).
```

### OSINT para defensores

**OSINT** (Open Source Intelligence) es inteligencia obtenida a partir de fuentes públicas y legales de acceso abierto: Internet, registros públicos, redes sociales, medios, bases de datos abiertas. "Open source" aquí significa "fuente abierta", no software de código abierto.

Por qué le importa al defensor: el atacante hace OSINT en la fase de Reconnaissance; el defensor hace lo mismo sobre su propia organización para ver lo que el atacante verá, y además usa fuentes abiertas para enriquecer alertas (¿esta IP tiene mala reputación? ¿este dominio se registró ayer?).

Analogía: revisar tu casa desde la calle como lo haría un ladrón: qué ventanas se ven abiertas, qué cajas de televisores nuevos dejaste en la basura, qué publicaste en redes de que te vas de vacaciones.

Dos usos para el defensor:

1. Reducción de superficie expuesta: buscar qué se ve de la propia organización: subdominios olvidados, servicios expuestos, correos de empleados filtrados en brechas, credenciales publicadas en repositorios de código, documentos con metadatos internos.
2. Enriquecimiento y triaje de alertas: dada una IP, dominio, URL o hash de una alerta, consultar su reputación y contexto.

Fuentes OSINT para defensores (todas con uso gratuito básico):

- VirusTotal: análisis de archivos, URLs, dominios e IPs con decenas de motores antivirus, y relaciones entre ellos.
- AlienVault OTX (Open Threat Exchange): comunidad que publica "pulses" con IOCs de campañas.
- abuse.ch: proyectos como MalwareBazaar (muestras de malware por hash), ThreatFox (IOCs) y URLhaus (URLs que distribuyen malware).
- Shodan: buscador de dispositivos y servicios expuestos en Internet; útil para ver qué puertos de la propia organización se ven desde fuera.
- urlscan.io: visita una URL en un entorno aislado y muestra capturas, peticiones y dominios contactados.
- Registros WHOIS y DNS pasivo: fecha de registro de un dominio y qué IPs resolvió en el pasado.
- Certificate Transparency (por ejemplo crt.sh): todos los certificados TLS emitidos para un dominio, lo que revela subdominios.
- OSINT Framework: un directorio en forma de árbol de herramientas OSINT por categoría.

Ejemplo de triaje: una alerta muestra a un equipo contactando `cdn-update-check[.]com`. El analista consulta: WHOIS dice que el dominio se registró hace 2 días; VirusTotal da 8 de 90 motores lo marcan como malicioso; urlscan muestra una página de login falsa de Microsoft 365; el DNS pasivo muestra que la IP aloja otros 15 dominios parecidos. En 10 minutos la alerta pasa de "dudosa" a "phishing confirmado", con 15 dominios extra para bloquear.

```
$ whois cdn-update-check.com | grep -i creation
Creation Date: 2026-09-29T14:02:11Z
```

Cuidado con VirusTotal: subir un archivo interno (un contrato, un documento con datos de clientes) lo deja disponible para los suscriptores de pago de la plataforma. En triaje se busca primero por hash, y el archivo se sube solo si no es confidencial.

Ética y legalidad: OSINT defensivo se limita a fuentes públicas y a la propia organización o a lo autorizado; usar credenciales filtradas para iniciar sesión, o hacer ingeniería social para obtener datos, ya no es OSINT pasivo y puede ser delito.

```
OSINT → inteligencia de fuentes públicas y legales.
Uso defensivo 1 → ver la propia superficie expuesta como la vería el atacante.
Uso defensivo 2 → enriquecer alertas: reputación, fecha de registro, relaciones.
VirusTotal → reputación de archivos, URLs, IPs y dominios.
Shodan → servicios expuestos en Internet.
abuse.ch → MalwareBazaar, ThreatFox, URLhaus: IOCs y muestras.
OSINT Framework → directorio de herramientas OSINT.
```

## Basics and Concepts of Threat Hunting

### Threat Hunting

El **threat hunting** (caza de amenazas) es una actividad proactiva en la que un analista busca, a mano y guiado por una idea, atacantes que ya están dentro de la red y que ninguna alerta automática detectó. Parte de un supuesto incómodo: asumir que ya hubo compromiso (assume breach).

Por qué existe: las detecciones automáticas solo ven lo que alguien ya sabía describir. Un atacante cuidadoso usa herramientas legítimas del sistema (PowerShell, WMI, RDP, lo que se llama living off the land) y no dispara ninguna regla. El tiempo que pasa dentro sin ser detectado se llama dwell time (tiempo de permanencia), y en muchas intrusiones se mide en semanas. El hunting existe para acortarlo.

Analogía: un guardia de museo no se limita a esperar que suene la alarma; recorre las salas pensando "si yo quisiera robar ese cuadro, ¿por dónde entraría?", y va a revisar justo ese lugar.

Qué necesita: telemetría con suficiente detalle y retención (logs de procesos de EDR, Sysmon, DNS, proxy, autenticación, de al menos 30 a 90 días), una plataforma para consultarla (el SIEM o el EDR), y conocimiento del atacante (ATT&CK, inteligencia) y de lo que es normal en la propia red.

Resultado de una cacería: se encuentra una intrusión (se pasa a respuesta a incidentes), o no se encuentra nada y se demuestra que esa técnica no está ocurriendo; en ambos casos suele quedar una regla de detección nueva y un mejor conocimiento de la red. Una cacería sin hallazgos no es una cacería fallida.

El nivel de madurez se mide a menudo con el Hunting Maturity Model (HMM), de 0 a 4: HMM0 solo depende de alertas automáticas, HMM1 busca IOCs de inteligencia, HMM2 sigue procedimientos de cacería de otros, HMM3 crea sus propios procedimientos, HMM4 automatiza las cacerías exitosas como detecciones.

Ejemplo de consulta de cacería sobre logs de Sysmon (evento 1, creación de proceso), en lenguaje de un SIEM como Splunk:

```
index=sysmon EventCode=1 ParentImage="*\\WINWORD.EXE"
  (Image="*\\powershell.exe" OR Image="*\\cmd.exe" OR Image="*\\wscript.exe")
| stats count by host, User, CommandLine

host       User       CommandLine                                     count
FIN-PC07   ana.r      powershell.exe -nop -w hidden -enc JABzAD0A...   1
```

Una sola máquina, un solo evento, y nadie había recibido alerta: ese es el tipo de hallazgo que justifica el hunting.

```
Threat hunting → búsqueda proactiva y humana de atacantes que no dispararon alertas.
Assume breach → supuesto de partida: el atacante ya está dentro.
Dwell time → tiempo que el atacante pasa dentro sin ser detectado; el hunting lo acorta.
HMM → modelo de madurez de hunting, de 0 (solo alertas) a 4 (automatiza lo cazado).
```

### Hunting por hipótesis

El **hunting por hipótesis** (hypothesis-driven hunting) es un método de cacería que parte de una afirmación concreta y verificable sobre lo que un atacante podría estar haciendo en la red, y la contrasta con los datos. Es la forma más estructurada de hacer hunting.

Por qué una hipótesis: "buscar cosas raras" en millones de eventos no termina nunca. Una hipótesis acota qué datos mirar, qué consulta escribir y cuándo parar.

Una buena hipótesis es específica, se puede comprobar con los datos que existen, y se basa en algo: inteligencia (un grupo que ataca al sector usa cierta técnica), una técnica de ATT&CK sin cobertura en el SOC, un incidente pasado o un cambio en el entorno.

```
mala:  "Hay atacantes en la red."
buena: "Un atacante podría estar usando tareas programadas (T1053.005) para
        persistir en servidores Windows; si es así, veremos tareas creadas con
        schtasks.exe por cuentas que no son de administración, apuntando a
        rutas en AppData o Temp, en los últimos 30 días."
```

Otros disparadores de cacería, además de la hipótesis: los IOCs de un informe de inteligencia reciente (búsqueda guiada por inteligencia, "¿alguien contactó estos 20 dominios?") y el análisis de anomalías sin hipótesis previa (comparar con la línea base, ver el método Baseline de PEAK).

Analogía: un médico no pide todos los análisis posibles; sospecha algo ("puede ser anemia") y pide el análisis que lo confirma o lo descarta.

Ejemplo: la hipótesis de las tareas programadas da 340 tareas creadas en 30 días. 330 son de software conocido (actualizadores, backup). Quedan 10: 9 son de un script de inventario de TI, y 1 crea `C:\Users\Public\svc.exe` cada 15 minutos en un servidor de archivos. Esa es la pista.

```
Hipótesis → afirmación específica y verificable sobre una técnica del atacante.
Búsqueda por IOCs → comprobar indicadores de un informe en los datos propios.
Búsqueda por anomalías → comparar con lo normal sin hipótesis previa.
```

### Ciclo de threat hunting

El **ciclo de threat hunting** es un proceso iterativo en el que cada cacería parte de una hipótesis, investiga los datos, encuentra patrones y deja algo automatizado para que la siguiente empiece desde más arriba. La versión más citada es el Threat Hunting Loop de Sqrrl (empresa de hunting luego comprada por Amazon), con 4 pasos:

```
   1. Crear hipótesis  ------------->  2. Investigar con herramientas
          ^                                  y técnicas (consultas,
          |                                  visualización, estadística)
          |                                         |
   4. Informar y enriquecer  <-------  3. Descubrir nuevos patrones
      la analítica (detecciones,             y TTPs
      inteligencia, documentación)
```

El paso 4 es el que hace que el hunting escale: lo que se encontró a mano una vez se convierte en una detección automática, y el cazador queda libre para la siguiente hipótesis. Si ese paso falta, el equipo encuentra lo mismo una y otra vez.

Analogía: un explorador que dibuja el mapa de cada camino que recorre; el siguiente explorador ya no necesita recorrer ese camino, empieza donde el mapa termina.

Ejemplo: la cacería de tareas programadas descubre un patrón (tarea que ejecuta un binario desde `C:\Users\Public` cada 15 minutos). Se convierte en una regla del SIEM, se documenta la consulta, se reporta el hallazgo a respuesta a incidentes y se etiqueta con T1053.005 en el ATT&CK Navigator como técnica ahora cubierta.

```
Crear hipótesis → qué podría estar haciendo el atacante.
Investigar → consultar y visualizar la telemetría.
Descubrir patrones → hallazgos, TTPs y lo que es normal.
Informar y enriquecer → convertir el hallazgo en detección y documentación.
```

### PEAK

**PEAK** (Prepare, Execute, Act with Knowledge) es un framework de threat hunting publicado por el equipo SURGe de Splunk en 2023 (David Bianco, autor también de la pirámide del dolor, es uno de sus creadores) que organiza cada cacería en tres fases y reconoce tres tipos de cacería.

Por qué existe: el ciclo de Sqrrl es útil pero muy general. PEAK lo vuelve un proceso con pasos concretos y añade que no toda cacería parte de una hipótesis.

Las tres fases, con el conocimiento (Knowledge) atravesándolas todas:

```
   Prepare            Execute              Act
   elegir tema        reunir datos         documentar y preservar
   investigar         analizar             convertir en detecciones
   definir alcance    escalar hallazgos    comunicar resultados
   (hipótesis, datos) críticos a IR
   ^------------------- Knowledge -------------------^
   (inteligencia, experiencia, conocimiento del negocio, hallazgos previos)
```

1. **Prepare (preparar)**: elegir el tema, investigarlo (cómo funciona la técnica, qué datos la dejan ver), formular la hipótesis o la pregunta y definir el alcance: qué sistemas, qué fuentes, qué ventana de tiempo.
2. **Execute (ejecutar)**: reunir los datos, analizarlos, refinar las consultas, y si aparece algo crítico, escalarlo de inmediato a respuesta a incidentes en vez de esperar al final.
3. **Act (actuar)**: documentar lo hecho, preservar las consultas, crear detecciones a partir de lo encontrado y comunicar los resultados.

Los tres tipos de cacería:

- **Hypothesis-driven (por hipótesis)**: el método explicado arriba.
- **Baseline (línea base)**: análisis exploratorio para entender qué es normal (qué procesos corren, qué cuentas inician sesión dónde y a qué hora) y encontrar lo que se aparta. Ejemplo: de 2.000 equipos, 1.998 ejecutan entre 40 y 60 binarios distintos al día; 2 ejecutan 300.
- **Model-Assisted (M-ATH)**: usar modelos de machine learning o estadísticos (clustering, detección de anomalías, clasificación) para señalar candidatos que el analista revisa. Ejemplo: un modelo que puntúa dominios por cuán aleatorios parecen sus nombres para encontrar dominios generados por algoritmo (DGA).

Analogía: una expedición a la montaña: preparar la ruta y el equipo, subir, y al bajar escribir el informe y marcar el camino para los siguientes; la experiencia de expediciones anteriores pesa en las tres etapas.

```
PEAK → framework de hunting de Splunk: Prepare, Execute, Act, con Knowledge en todas.
Prepare → tema, investigación, hipótesis y alcance.
Execute → datos, análisis, escalado de lo crítico.
Act → documentar, crear detecciones, comunicar.
Hypothesis-driven → cacería que parte de una hipótesis.
Baseline → cacería que parte de entender lo normal.
Model-Assisted (M-ATH) → cacería asistida por modelos de ML o estadística.
```

### Threat hunting vs monitoreo

El **monitoreo de seguridad** es una actividad reactiva en la que el SOC espera y atiende las alertas que generan reglas automáticas; el threat hunting es la actividad proactiva que busca lo que esas reglas no vieron. Se complementan: el hunting alimenta al monitoreo con reglas nuevas, y el monitoreo libera al cazador de lo ya conocido.

> [!WARNING]
> Revisar las alertas del SIEM, por muchas que sean, no es threat hunting. Si el punto de partida fue una alerta, es triaje o investigación; el hunting empieza sin alerta, desde una hipótesis.

```
                     Monitoreo (SOC)               Threat hunting
Disparador           una alerta automática         una hipótesis humana
Postura              reactiva: espera              proactiva: sale a buscar
Qué detecta          lo conocido (reglas, IOCs)    lo desconocido (TTPs, anomalías)
Quién                analista L1/L2, 24x7          analista con experiencia, por campañas
Resultado            alerta cerrada o escalada     hallazgo, o detección nueva, o ambos
```

La primera fila es la diferencia de fondo: quién empuja la investigación, una máquina o una persona.

Analogía: el monitoreo es la alarma de la casa, que suena cuando alguien abre una ventana con sensor; el hunting es recorrer la casa revisando las ventanas que no tienen sensor.

Ejemplo: en un mes, el SOC atiende 3.000 alertas y escala 12 incidentes (monitoreo). En ese mismo mes, un cazador dedica 2 semanas a una hipótesis sobre robo de credenciales en LSASS, encuentra 1 servidor comprometido que no generó ninguna alerta, y deja 2 reglas nuevas en el SIEM; desde el mes siguiente, eso ya es monitoreo. La detección automática en sí se estudia en [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).

```
Monitoreo → reactivo; parte de una alerta; detecta lo conocido.
Threat hunting → proactivo; parte de una hipótesis; busca lo desconocido.
Triaje → revisar una alerta para decidir si es real; no es hunting.
```

## Recursos para aprender y practicar

### Videos

- [Breaking The Kill-Chain: A Defensive Approach](https://www.youtube.com/watch?v=II91fiUax2g) — The CISO Perspective; recorre las fases y qué control corta cada una. Nodo: Cyber Kill Chain.
- [An Introduction to the Diamond Model of Intrusion Analysis by its Co-Author Sergio Caltagirone](https://www.youtube.com/watch?v=Yb4rg2NbgNw) — Threat Intelligence Academy; el modelo explicado por su coautor, con pivoting y activity threads. Nodo: Diamond Model.
- [MITRE ATT&CK Framework](https://www.youtube.com/watch?v=Yxv1suJYMI8) — MITRE; presentación oficial de la base de conocimiento en 4 minutos. Nodo: ATT&CK.
- [Introduction To The MITRE ATT&CK Framework](https://www.youtube.com/watch?v=LCec9K0aAkM) — HackerSploit; recorrido por la matriz, técnicas, grupos y Navigator (anterior a v19: muestra 14 tácticas con Defense Evasion). Nodo: ATT&CK.
- [Threat Intelligence - CompTIA Security+ SY0-701 - 4.3](https://www.youtube.com/watch?v=86fruE9jkKk) — Professor Messer; fuentes de inteligencia, OSINT y STIX/TAXII en formato de certificación. Nodos: Threat Intel, OSINT, STIX/TAXII.
- [The Cycle of Cyber Threat Intelligence](https://www.youtube.com/watch?v=J7e74QLVxCk) — SANS DFIR; charla de una hora sobre el ciclo de inteligencia y sus errores comunes. Nodo: ciclo de inteligencia.
- [Pyramid of Pain: Intel-Driven Detection/Response to Increase Adversary's Cost](https://www.youtube.com/watch?v=zlAWbdSlhaQ) — RVAsec, charla de David Bianco; la pirámide contada por su autor. Nodo: pirámide del dolor.
- [Cybersecurity Threat Hunting Explained](https://www.youtube.com/watch?v=VNp35Uw_bSM) — IBM Technology; qué es el hunting y en qué se diferencia del monitoreo. Nodo: Threat Hunting.

### Lectura y documentación

- [Cyber Kill Chain](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html) — Lockheed Martin; página oficial con las 7 fases y el paper original de 2011.
- [The Unified Kill Chain](https://www.unifiedkillchain.com/) — Paul Pols; las 18 fases y el paper completo.
- [The Diamond Model of Intrusion Analysis (PDF)](https://www.activeresponse.org/wp-content/uploads/2013/07/diamond.pdf) — Caltagirone, Pendergast y Betz, 2013; el paper original con vértices, meta-características y pivoting.
- [MITRE ATT&CK](https://attack.mitre.org/) — sitio oficial; [Enterprise Tactics](https://attack.mitre.org/tactics/enterprise/) muestra las 15 tácticas vigentes, [T1566 Phishing](https://attack.mitre.org/techniques/T1566/) es un buen modelo de página de técnica y [APT28 (G0007)](https://attack.mitre.org/groups/G0007/) de página de grupo.
- [ATT&CK Versions](https://attack.mitre.org/resources/versions/) — historial de versiones, para saber con qué versión se escribió un curso o una pregunta.
- [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/) — la herramienta web para crear capas de cobertura.
- [The Pyramid of Pain](https://detect-respond.blogspot.com/2013/03/the-pyramid-of-pain.html) — David Bianco; el artículo original.
- [Indicator of Compromise (glosario NIST)](https://csrc.nist.gov/glossary/term/indicator_of_compromise) — definición formal de IOC.
- [STIX/TAXII documentation](https://oasis-open.github.io/cti-documentation/) — OASIS; especificaciones, objetos STIX 2.1 y ejemplos.
- [MISP](https://www.misp-project.org/) — plataforma abierta de intercambio de inteligencia que habla STIX/TAXII.
- [OSINT Framework](https://osintframework.com/) — directorio de herramientas OSINT por categoría.
- Fuentes OSINT para triaje: [VirusTotal](https://www.virustotal.com/), [AlienVault OTX](https://otx.alienvault.com/), [abuse.ch](https://abuse.ch/) ([MalwareBazaar](https://bazaar.abuse.ch/), [ThreatFox](https://threatfox.abuse.ch/)), [urlscan.io](https://urlscan.io/), [Shodan](https://www.shodan.io/).
- [The PEAK Threat Hunting Framework](https://www.splunk.com/en_us/blog/security/peak-threat-hunting-framework.html) — Splunk SURGe; las fases Prepare, Execute, Act y los tres tipos de cacería.
- [Generating Hypotheses for Successful Threat Hunting](https://www.sans.org/white-papers/37172/) — SANS white paper; cómo escribir hipótesis de cacería buenas.

### Práctica

- [TryHackMe: Cyber Kill Chain](https://tryhackme.com/room/cyberkillchainzmt) — clasificar acciones de un atacante en cada fase. Nodo: Cyber Kill Chain.
- [TryHackMe: Pyramid Of Pain](https://tryhackme.com/room/pyramidofpainax) — gratis; clasificar indicadores según cuánto le cuestan al atacante. El Diamond Model no tiene sala gratis: practícalo rellenando los cuatro vértices con un informe público de [The DFIR Report](https://thedfirreport.com/).
- [TryHackMe: Unified Kill Chain](https://tryhackme.com/room/unifiedkillchain) y [Intro to Threat Emulation](https://tryhackme.com/room/threatemulationintro) — gratis; fases del ataque y uso de ATT&CK para emular adversarios. Para navegar ATT&CK, usa directamente el [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/).
- [TryHackMe: Intro to Cyber Threat Intel](https://tryhackme.com/room/cyberthreatintel) — tipos de inteligencia, ciclo de vida y estándares. Nodos: Threat Intel, ciclo, STIX/TAXII.
- [TryHackMe: Threat Intelligence Tools](https://tryhackme.com/room/threatinteltools) — usar urlscan.io, abuse.ch, Talos y otras fuentes para investigar IOCs. Nodos: OSINT para defensores, IOCs.
- [TryHackMe: Intro to Threat Hunting](https://tryhackme.com/room/threathuntingintroduction) — mentalidad del cazador, hipótesis y diferencia con respuesta a incidentes. Nodos: Threat Hunting, hipótesis, hunting vs monitoreo.
- [TryHackMe: OhSINT](https://tryhackme.com/room/ohsint), [Sakura Room](https://tryhackme.com/room/sakura) y [Searchlight - IMINT](https://tryhackme.com/room/searchlightosint) — gratis; encadenar fuentes públicas a partir de un solo dato. Nodo: OSINT.
- [CyberDefenders: Blue Team CTF Challenges](https://cyberdefenders.org/blueteam-ctf-challenges/) — retos de análisis con categoría de Threat Intel y de Threat Hunting sobre logs reales.
- [Ejercicio 1: Por qué bloquear un hash no basta](ejercicios.md#ejercicio-1-por-qué-bloquear-un-hash-no-basta) — contar en MalwareBazaar los hashes distintos de una familia y justificar con ese número por qué el hash está en la base de la pirámide.
- [Ejercicio 2: Huecos de cobertura de Sysmon frente a APT28](ejercicios.md#ejercicio-2-huecos-de-cobertura-de-sysmon-frente-a-apt28) — superponer en el ATT&CK Navigator las capas de APT28 y de Sysmon y explicar 3 técnicas sin cobertura.
- [Ejercicio 3: Superficie expuesta de tu dominio con OSINT](ejercicios.md#ejercicio-3-superficie-expuesta-de-tu-dominio-con-osint) — listar subdominios propios en Certificate Transparency, ver sus puertos en Shodan y señalar lo que no debería ser público.
- [Ejercicio 4: Caza por hipótesis de una tarea programada sospechosa](ejercicios.md#ejercicio-4-caza-por-hipótesis-de-una-tarea-programada-sospechosa) — escribir una hipótesis sobre T1053.005 y confirmarla cruzando el evento 1 de Sysmon con el 4698 de Security.

## Cuadro resumen

Todo lo visto, en una línea por término.

Cyber Kill Chain

```
Cyber Kill Chain → modelo de 7 fases de Lockheed Martin (2011) para cortar intrusiones lo antes posible.
Reconnaissance → el atacante investiga; corte: reducir exposición, vigilar escaneos.
Weaponization → arma exploit + payload; corte: firmas a partir de armas pasadas; sin visibilidad directa.
Delivery → el arma llega (correo, web, USB); corte: filtro de correo, proxy, formación.
Exploitation → el código se ejecuta; corte: parches, macros bloqueadas, mínimo privilegio.
Installation → persistencia; corte: allowlisting, EDR sobre autoarranques.
Command and Control → canal remoto; corte: filtrado de salida, detección de beaconing.
Actions on Objectives → robo, cifrado, sabotaje; corte: segmentación, DLP, backups.
6 D → Detect, Deny, Disrupt, Degrade, Deceive, Destroy: cursos de acción por fase.
Cadena completa → requisito del atacante para ganar.
Un eslabón roto → suficiente para el defensor; más temprano, más barato.
Defense in depth → varias barreras por fase porque el atacante reintenta.
```

Understand Frameworks

```
Adversary → quién ataca (operador y cliente).
Capability → con qué ataca (malware, exploit, técnica).
Infrastructure → desde dónde llega (IP, dominio, C2); tipo 1 propia, tipo 2 de intermediarios.
Victim → a quién ataca (persona víctima y activos víctima).
Meta-features → timestamp, phase, result, direction, methodology, resources.
Pivoting → partir de un vértice conocido para descubrir los otros.
Cyber Kill Chain (Lockheed Martin, 2011) → 7 fases lineales, foco en entrar.
Unified Kill Chain (Pols, 2017) → 18 fases en 3 ciclos In/Through/Out, foco también en moverse dentro.
ATT&CK → base de conocimiento de MITRE con comportamientos reales de atacantes; v19.2 vigente.
Táctica (TA####) → el objetivo del paso; columna de la matriz; 15 en Enterprise v19.
Técnica (T####) → cómo se logra la táctica.
Sub-técnica (T####.###) → variante específica de una técnica; T1566.001 Spearphishing Attachment.
Procedimiento → cómo lo hizo un grupo concreto en un caso real.
Grupo (G####) → actor de amenaza con sus técnicas observadas; G0007 = APT28.
Navigator → herramienta web para colorear la matriz en capas y ver cobertura.
Stealth (TA0005) → esconderse y parecer normal.
Defense Impairment (TA0112) → romper o apagar las defensas.
Kill Chain → cuándo: fase del ataque.
Diamond Model → quién/qué/dónde/a quién: relaciones entre eventos.
ATT&CK → cómo: técnica exacta con su ID.
```

Basics of Threat Intel, OSINT

```
Threat intelligence → información sobre amenazas analizada y con contexto para decidir.
Estratégica → tendencias y riesgo de negocio, para directivos; meses a años.
Táctica → TTPs de los atacantes, para detección y arquitectura; semanas a meses.
Operacional → campaña concreta en curso (quién, cuándo, contra quién), para SOC/IR; días.
Técnica → IOCs legibles por máquina (hash, IP, dominio), para herramientas; horas a días.
Direction → definir qué hay que saber.
Collection → juntar datos internos y externos.
Processing → limpiar y normalizar.
Analysis → interpretar y recomendar.
Dissemination → entregar a cada consumidor en su formato.
Feedback → ajustar el siguiente ciclo.
IOC → evidencia de que ya hubo compromiso; concreto, reactivo, fácil de evadir.
IOA → comportamiento de un ataque en curso; independiente de la herramienta, más difícil de evadir.
Hash values → trivial de cambiar.
IP addresses → fácil de cambiar.
Domain names → simple de cambiar.
Network/Host artifacts → molesto de cambiar; obliga a reconfigurar la herramienta.
Tools → difícil; obliga a cambiar de herramienta.
TTPs → lo más duro; obliga a cambiar de forma de operar.
Base de la pirámide → barato para ambos; útil para bloqueo rápido y búsqueda hacia atrás.
Cima de la pirámide → caro para ambos; es lo que de verdad frena al atacante.
STIX → formato (JSON) para describir inteligencia: objetos y relaciones; v2.1.
TAXII → protocolo (HTTPS, 443) para transportar STIX entre sistemas; Collections y Channels.
SDO → objeto de dominio (Indicator, Malware, Threat Actor...).
SRO → objeto de relación (Relationship, Sighting).
OSINT → inteligencia de fuentes públicas y legales.
Uso defensivo 1 → ver la propia superficie expuesta como la vería el atacante.
Uso defensivo 2 → enriquecer alertas: reputación, fecha de registro, relaciones.
VirusTotal → reputación de archivos, URLs, IPs y dominios; buscar por hash antes de subir.
Shodan → servicios expuestos en Internet.
abuse.ch → MalwareBazaar, ThreatFox, URLhaus: IOCs y muestras.
OSINT Framework → directorio de herramientas OSINT.
```

Basics and Concepts of Threat Hunting

```
Threat hunting → búsqueda proactiva y humana de atacantes que no dispararon alertas.
Assume breach → supuesto de partida: el atacante ya está dentro.
Dwell time → tiempo que el atacante pasa dentro sin ser detectado; el hunting lo acorta.
HMM → modelo de madurez de hunting, de 0 (solo alertas) a 4 (automatiza lo cazado).
Hipótesis → afirmación específica y verificable sobre una técnica del atacante.
Búsqueda por IOCs → comprobar indicadores de un informe en los datos propios.
Búsqueda por anomalías → comparar con lo normal sin hipótesis previa.
Crear hipótesis → qué podría estar haciendo el atacante.
Investigar → consultar y visualizar la telemetría.
Descubrir patrones → hallazgos, TTPs y lo que es normal.
Informar y enriquecer → convertir el hallazgo en detección y documentación.
PEAK → framework de hunting de Splunk: Prepare, Execute, Act, con Knowledge en todas.
Prepare → tema, investigación, hipótesis y alcance.
Execute → datos, análisis, escalado de lo crítico.
Act → documentar, crear detecciones, comunicar.
Hypothesis-driven → cacería que parte de una hipótesis.
Baseline → cacería que parte de entender lo normal.
Model-Assisted (M-ATH) → cacería asistida por modelos de ML o estadística.
Monitoreo → reactivo; parte de una alerta; detecta lo conocido.
Threat hunting → proactivo; parte de una hipótesis; busca lo desconocido.
Triaje → revisar una alerta para decidir si es real; no es hunting.
```
