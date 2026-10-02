# Conceptos de seguridad

## Conceptos previos

- Activo: cualquier cosa de valor para una organización que hay que proteger (un servidor, una base de datos de clientes, la reputación, un portátil).
- Control de seguridad: medida que reduce un riesgo; puede ser técnica (firewall), administrativa (política) o física (cerradura).
- Firewall: equipo o programa que deja pasar o bloquea tráfico de red según reglas (origen, destino, puerto).
- VLAN: red local virtual; separa lógicamente equipos conectados al mismo switch como si estuvieran en switches distintos.
- Autenticación: comprobar quién es alguien (contraseña, MFA). Autorización: decidir qué puede hacer una vez identificado. Detalle en [`08-autenticacion`](../08-autenticacion/).
- Hash: huella de tamaño fijo de unos datos; si cambia un bit de los datos, cambia la huella. Detalle en [`10-criptografia`](../10-criptografia/).
- SIEM: plataforma que junta los logs de toda la red y dispara alertas cuando ve patrones sospechosos. Detalle en [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).
- Exploit: código o técnica que aprovecha una vulnerabilidad concreta.
- Mínimo privilegio: principio que dice que cada usuario o proceso recibe solo los permisos que necesita para su tarea, ni uno más.

## Principios de seguridad

### Understand CIA Triad

La **tríada CIA** es un modelo de objetivos de seguridad que dice que proteger información significa preservar tres propiedades: Confidentiality (confidencialidad), Integrity (integridad) y Availability (disponibilidad).

Existe porque "seguridad" a secas no dice qué hay que proteger. La tríada convierte la pregunta en tres concretas: ¿quién puede verlo?, ¿alguien pudo cambiarlo sin que lo notemos?, ¿está ahí cuando lo necesito? Todo control, todo ataque y todo incidente se puede clasificar según cuál de las tres propiedades protege o rompe.

- Confidentiality: solo quien está autorizado puede leer los datos. Se protege con cifrado, control de acceso y clasificación de datos. La rompe una filtración, un sniffer en una red sin cifrar o un empleado que reenvía una hoja de nóminas.
- Integrity: los datos no se alteran sin autorización, y si se alteran, se detecta. Se protege con hashes, firmas digitales, control de versiones y permisos de escritura. La rompe un atacante que cambia el número de cuenta en una factura o un ransomware que cifra los archivos.
- Availability: los sistemas y datos están accesibles cuando se necesitan. Se protege con redundancia, backups, balanceo de carga y protección anti-DDoS. La rompe un ataque de denegación de servicio, un disco que muere sin réplica o un corte eléctrico.

Analogía: una caja fuerte de banco. Confidencialidad es que solo el titular la abre; integridad es que lo que hay dentro sea lo mismo que dejó; disponibilidad es que el banco abra cuando el titular lo necesita. Una caja impenetrable que nadie puede abrir nunca protege la confidencialidad y destruye la disponibilidad.

Las tres propiedades tiran en direcciones opuestas. Más cifrado y más pasos de acceso suben la confidencialidad y bajan la disponibilidad; más copias de los datos suben la disponibilidad y multiplican los lugares por donde se pueden filtrar. Por eso cada sistema decide cuál pesa más: en un hospital, la disponibilidad de la historia clínica en urgencias manda; en un banco, la integridad del saldo manda; en un servicio de mensajería privada, la confidencialidad manda.

El reverso de la tríada es DAD: Disclosure (divulgación, rompe C), Alteration (alteración, rompe I) y Destruction/Denial (destrucción o denegación, rompe A). Muchos modelos añaden dos propiedades más: autenticidad (el dato viene de quien dice venir) y no repudio (el autor no puede negar haberlo hecho).

Ejemplo: comprobar la integridad de una ISO descargada comparando su hash con el publicado por el proyecto.

```
$ sha256sum debian-12.iso
3b4f...9a1c  debian-12.iso
$ grep debian-12.iso SHA256SUMS
3b4f...9a1c  debian-12.iso      # coinciden: el archivo no fue alterado
```

Si los hashes no coinciden, el archivo cambió por el camino (descarga corrupta o un espejo manipulado): se viola la integridad aunque nadie haya leído nada (la confidencialidad sigue intacta).

> [!TIP]
> En un caso de examen, pregunta qué propiedad se perdió: si alguien leyó lo que no debía, es C; si algo cambió, es I; si no se pudo usar, es A. Un ransomware rompe I y A a la vez, y si además exfiltra los datos, también C.

```
Confidentiality → solo los autorizados leen; cifrado, control de acceso.
Integrity → nada cambia sin autorización y sin detectarse; hash, firma.
Availability → accesible cuando se necesita; redundancia, backups, anti-DDoS.
DAD → Disclosure, Alteration, Destruction: el reverso de CIA.
```

### Understand Concept of Defense in Depth

**Defense in depth** (defensa en profundidad) es una estrategia de diseño que coloca varias capas de controles independientes entre el atacante y el activo, de modo que si una capa falla, la siguiente todavía lo detiene o lo detecta.

Existe porque ningún control es perfecto: los firewalls se configuran mal, los usuarios caen en phishing, los parches llegan tarde. Si toda la seguridad depende de una sola barrera, un único fallo basta para perderlo todo. Con capas, el atacante tiene que vencer todas, y cada capa extra le da al defensor otra oportunidad de verlo.

Analogía: un castillo medieval. Foso, muralla exterior, muralla interior, torre del homenaje y guardias en cada puerta. Cruzar el foso no basta; cada obstáculo cuesta tiempo y hace ruido.

Las capas típicas, de fuera hacia dentro:

```
Atacante
   |
   v
[ Políticas y concienciación ]   formación anti-phishing, normas de uso
   |
[ Seguridad física ]             puertas con tarjeta, cámaras
   |
[ Perímetro ]                    firewall, WAF, anti-DDoS
   |
[ Red interna ]                  segmentos, IDS
   |
[ Host ]                         EDR, parches, firewall local
   |
[ Aplicación ]                   autenticación, autorización, validación
   |
[ Datos ]                        cifrado, backups
   |
   v
Activo
```

Para que funcione, las capas deben ser de tipos distintos y fallar por motivos distintos. Tres antivirus del mismo motor no son tres capas, son una repetida. Lo correcto es mezclar controles preventivos (impiden), detectivos (avisan) y correctivos (reparan), y mezclar los tipos técnico, administrativo y físico.

Ejemplo: un empleado abre un adjunto malicioso. Capa 1, el filtro de correo no lo detectó. Capa 2, el EDR del portátil bloquea la macro al intentar lanzar PowerShell. Si también hubiera fallado, capa 3: el portátil está en una VLAN de usuarios que no puede hablar con el servidor de base de datos, y capa 4: los datos están cifrados y respaldados fuera de línea. El atacante tendría que vencer cuatro barreras distintas.

Sumar productos no es hacer defensa en profundidad: si todas las capas confían en lo mismo (por ejemplo, todas asumen que lo que está dentro de la red es legítimo), un solo engaño las atraviesa todas.

```
Defense in depth → varias capas independientes; si una falla, la siguiente frena o detecta.
Control preventivo → impide (firewall, MFA).
Control detectivo → avisa (IDS, logs, SIEM).
Control correctivo → repara tras el daño (backup, reimagen).
```

### Core Concepts of Zero Trust

**Zero Trust** es un modelo de seguridad que elimina la confianza implícita basada en la ubicación de red y exige verificar explícitamente identidad, dispositivo y contexto en cada solicitud de acceso a cada recurso, concediendo solo el mínimo privilegio necesario.

Existe porque el modelo clásico, "castillo y foso", confiaba en todo lo que estuviera dentro de la red corporativa. Ese modelo se rompió por tres motivos: los empleados trabajan desde casa y desde la nube, los servicios ya no viven en el centro de datos propio, y cuando un atacante entra (phishing, VPN robada) se mueve libremente porque dentro todo le confía. Zero Trust parte de la suposición contraria: asume que la red ya está comprometida.

Analogía: un hospital donde cada puerta tiene lector de tarjeta. Entrar al edificio no da acceso a quirófanos ni a farmacia; cada puerta vuelve a comprobar quién eres, si tu turno está activo y si tu rol necesita esa sala, y la tarjeta caduca al terminar el turno.

Los principios centrales, según NIST SP 800-207 y Microsoft:

- Verify explicitly (verificar siempre): cada acceso se autentica y autoriza con todas las señales disponibles: usuario, MFA, estado del dispositivo (parcheado, con EDR), ubicación, hora, nivel de riesgo.
- Least privilege (mínimo privilegio): acceso solo al recurso concreto, solo el tiempo necesario (just-in-time) y solo con las acciones necesarias (just-enough-access).
- Assume breach (asumir la brecha): diseñar como si el atacante ya estuviera dentro: segmentar al máximo, cifrar también el tráfico interno, registrar y analizar todo.
- Acceso por sesión y por recurso: la confianza no se hereda; obtener acceso a una aplicación no da acceso a otra.

Cómo funciona por dentro (arquitectura de SP 800-207):

```
 Sujeto (usuario + dispositivo)
        |
        v
 +------------------+      decide       +-------------------+
 | PEP              | <---------------- | PDP               |
 | Policy           |                   |  Policy Engine    |  <- señales: IdP, MDM/EDR,
 | Enforcement Point|  ----consulta---> |  Policy Admin.    |     threat intel, SIEM
 +------------------+                   +-------------------+
        |  (solo si permitido, sesión limitada)
        v
 Recurso (app, API, base de datos)
```

El PEP (punto de aplicación de política) es la puerta: un proxy, un gateway de acceso, un agente. Pregunta al PDP (punto de decisión, formado por el Policy Engine que decide y el Policy Administrator que abre o cierra la sesión). El motor evalúa la política con señales en tiempo real y puede revocar la sesión a mitad si el riesgo cambia.

Ejemplo: una ingeniera intenta abrir el panel de facturación a las 3:00 desde un portátil sin el último parche. El motor ve tres señales malas (hora inusual, dispositivo no conforme, recurso fuera de su rol habitual) y responde con denegación o con MFA reforzado. Una petición directa sin identidad válida al recurso recibe:

```
$ curl -i https://billing.corp.example/api/invoices
HTTP/1.1 403 Forbidden
x-reason: device_not_compliant
```

Zero Trust no es un producto, es un modelo; se implementa por fases con identidad fuerte (MFA, SSO), gestión de dispositivos, microsegmentación y acceso ZTNA en lugar de VPN plana. CISA describe la evolución en su Zero Trust Maturity Model con cinco pilares: Identity, Devices, Networks, Applications and Workloads, y Data.

> [!NOTE]
> Zero Trust no sustituye a defense in depth: es una forma de aplicarlo en la que cada capa vuelve a verificar en lugar de confiar en la anterior.

```
Perímetro clásico → confía en todo lo que está dentro de la red.
Zero Trust → no confía por ubicación; verifica cada acceso, mínimo privilegio, asume brecha.
PEP → aplica la decisión (la puerta).
PDP (Policy Engine + Policy Administrator) → decide con señales y abre o corta la sesión.
ZTNA → acceso a una app concreta tras verificar; VPN → acceso a toda la red.
```

## Riesgo y resiliencia

### Understand the Definition of Risk

El **riesgo** es una medida de la pérdida esperada que combina la probabilidad de que una amenaza explote una vulnerabilidad con el impacto que tendría sobre un activo.

Existe como concepto porque los recursos de seguridad son limitados: no se puede proteger todo igual. El riesgo permite ordenar: qué arreglar primero, cuánto gastar en cada control y qué aceptar conscientemente.

Las piezas, cada una definida:

- Amenaza (threat): algo o alguien capaz de causar daño: un grupo de ransomware, un empleado descontento, un incendio.
- Vulnerabilidad: una debilidad que la amenaza puede aprovechar: un servidor sin parchear, una contraseña débil, un centro de datos en zona inundable.
- Impacto: cuánto daño produce si ocurre: dinero, horas de caída, datos perdidos, multas, reputación.

La fórmula conceptual:

```
Riesgo = Amenaza × Vulnerabilidad × Impacto
(versión habitual: Riesgo = Probabilidad × Impacto)
```

El producto expresa una idea: si cualquiera de los tres factores es cero, no hay riesgo. Una vulnerabilidad sin amenaza que la ataque (un fallo en un equipo aislado sin red) o una amenaza sin vulnerabilidad (un ataque contra un software que no usamos) no generan riesgo real.

Analogía: dejar la bicicleta en la calle. La amenaza son los ladrones del barrio; la vulnerabilidad, no ponerle candado; el impacto, cuánto vale la bicicleta. Una bicicleta vieja de 50 dólares sin candado en un pueblo tranquilo tiene poco riesgo; una de 3000 sin candado en el centro de una ciudad, mucho.

Análisis cualitativo: se puntúan probabilidad e impacto de 1 a 5 y se multiplica. Un riesgo con probabilidad 4 e impacto 5 da 20 (crítico, zona roja de la matriz); uno con 2 y 2 da 4 (bajo, zona verde). Es rápido y sirve para comparar, pero los números son opiniones.

Análisis cuantitativo: pone todo en dinero con cuatro términos.

- AV (Asset Value): valor del activo.
- EF (Exposure Factor): porcentaje del activo que se pierde en un incidente (0 a 100 %).
- SLE (Single Loss Expectancy): pérdida en un solo incidente.
- ARO (Annualized Rate of Occurrence): cuántas veces al año se espera que ocurra; 0,5 significa una vez cada dos años, 4 significa cuatro veces al año.
- ALE (Annualized Loss Expectancy): pérdida esperada por año.

```
SLE = AV × EF
ALE = SLE × ARO
Valor del control = ALE antes − ALE después − coste anual del control
```

Ejemplo con números redondos: un servidor de comercio electrónico vale 100 000 dólares (hardware, datos y ventas que genera). Un ataque de ransomware destruiría el 40 % de ese valor.

```
SLE = 100 000 × 0,40 = 40 000 $ por incidente
ARO = 0,5   (se espera un incidente cada 2 años)
ALE = 40 000 × 0,5 = 20 000 $ al año

Control propuesto: EDR + backups inmutables, 5 000 $/año, baja el ARO a 0,1
ALE nuevo = 40 000 × 0,1 = 4 000 $ al año
Valor del control = 20 000 − 4 000 − 5 000 = 11 000 $ al año  → se compra
```

Si el control costara 18 000 dólares al año, el valor sería 20 000 − 4 000 − 18 000 = −2000: protegería, pero costaría más que el daño que evita, y la decisión racional sería otro control o aceptar el riesgo.

Una vez medido el riesgo, hay cuatro respuestas:

- Mitigar (reducir): aplicar controles que bajen probabilidad o impacto (parchear, segmentar).
- Transferir (compartir): pasar el impacto económico a otro (ciberseguro, proveedor con SLA).
- Evitar: dejar de hacer la actividad que genera el riesgo (apagar el servicio expuesto).
- Aceptar: asumirlo conscientemente y por escrito, cuando está dentro del apetito de riesgo (cuánto riesgo está dispuesta a tolerar la organización).

El riesgo antes de aplicar controles es el riesgo inherente; el que queda después es el riesgo residual. Nunca llega a cero.

```
Amenaza → quién o qué puede causar daño.
Vulnerabilidad → la debilidad que aprovecha.
Impacto → cuánto daño causa.
SLE = AV × EF → pérdida por incidente.
ARO → incidentes esperados por año.
ALE = SLE × ARO → pérdida esperada por año.
Mitigar / Transferir / Evitar / Aceptar → las cuatro respuestas al riesgo.
Riesgo inherente → antes de controles; residual → lo que queda después.
```

### Sin amenaza, vulnerabilidad e impacto a la vez no hay riesgo

> [!IMPORTANT]
> El riesgo es un producto, no una suma: basta con llevar uno de los tres factores a cero para eliminarlo, y por eso la seguridad elige qué factor atacar en cada caso.

La relación multiplicativa entre amenaza, vulnerabilidad e impacto es una regla de decisión que dice cuál palanca usar: casi nunca se controla la amenaza (no se puede despedir a los ciberdelincuentes), así que el trabajo diario es bajar la vulnerabilidad (parches, configuración) o el impacto (backups, segmentación, cifrado).

```
Riesgo = Amenaza × Vulnerabilidad × Impacto
```

```
                      Servidor A        Servidor B        Servidor C
Amenaza activa        sí                sí                no (sin red)
Vulnerabilidad        sí (sin parche)   sí (sin parche)   sí (sin parche)
Impacto               alto              ~0 (sin datos,    alto
                                         se reconstruye)
Riesgo                alto              casi nulo         casi nulo
```

La última fila solo es alta en una columna: el mismo fallo sin parchear vale distinto según los otros dos factores. Por eso un escáner que reporta "vulnerabilidad crítica" no dice todavía cuánto riesgo hay; eso se trata en [`17-estandares-y-cumplimiento`](../17-estandares-y-cumplimiento/).

Límite de la idea: la amenaza casi nunca es exactamente cero, solo baja; un equipo "aislado" puede recibir un USB. El producto sirve para priorizar, no para declarar algo invulnerable.

```
Bajar vulnerabilidad → parchear, configurar, endurecer.
Bajar impacto → backups, segmentar, cifrar, minimizar datos.
Bajar amenaza → rara vez posible; disuasión, no exponerse.
```

### Understand Backups and Resiliency

Un **backup** es una copia de los datos guardada aparte del original que permite restaurarlos tras una pérdida; la **resiliencia** es la capacidad de un sistema de seguir funcionando, o volver a funcionar en un tiempo aceptable, cuando algo falla.

Existen porque los datos se pierden por motivos que ningún control preventivo elimina del todo: discos que mueren, borrados accidentales, ransomware, incendios. La disponibilidad de la tríada CIA depende de ambos: la resiliencia evita la caída, el backup permite volver si la caída ocurre.

Analogía: las fotos de la familia. Tenerlas solo en el móvil es perderlas si se cae al río; tenerlas también en el portátil ayuda, pero si hay un incendio en casa se pierden las dos; una tercera copia en casa de los abuelos sobrevive al incendio.

La regla 3-2-1:

- 3 copias de los datos (el original y dos backups).
- 2 tipos de soporte distintos (disco local y cinta, o NAS y nube), para que un mismo fallo no se lleve ambos.
- 1 copia fuera del sitio (off-site), para sobrevivir a un desastre local.

La variante moderna 3-2-1-1-0 añade 1 copia offline o inmutable (que el ransomware no puede cifrar porque no está montada o no admite cambios) y 0 errores al verificar la restauración.

Tipos de backup:

- Completo (full): copia todo cada vez. Restaurar es lo más simple (un solo conjunto) pero ocupa más y tarda más.
- Incremental: copia solo lo que cambió desde el último backup de cualquier tipo. Es el más rápido y pequeño cada día, pero restaurar exige el último completo más todos los incrementales en orden.
- Diferencial: copia lo que cambió desde el último completo. Crece cada día, pero restaurar solo exige el completo y el último diferencial.
- Snapshot: imagen del estado de un volumen o máquina virtual en un instante, casi inmediata; si vive en el mismo almacenamiento, no protege de que ese almacenamiento falle.

Ejemplo con números: completo el domingo (100 GB) y 5 GB de cambios diarios. Para restaurar el jueves:

```
Incremental: domingo + lun + mar + mié + jue → 5 conjuntos, cada uno de 5 GB
Diferencial: domingo + jueves              → 2 conjuntos; el del jueves pesa 20 GB
```

Dos métricas fijan cuánto se puede perder:

- RPO (Recovery Point Objective): cuántos datos, medidos en tiempo, se pueden perder como máximo. Un RPO de 4 horas obliga a hacer backup al menos cada 4 horas.
- RTO (Recovery Time Objective): cuánto tiempo puede estar caído el servicio hasta volver a funcionar. Un RTO de 1 hora obliga a poder restaurar en menos de 1 hora.
- MTD (Maximum Tolerable Downtime): el límite de caída tras el cual el daño al negocio es inaceptable; el RTO debe quedar por debajo.

```
  último backup          desastre                  servicio restaurado
       |---------------------|-----------------------------|
       <------- RPO -------->|<----------- RTO ------------>
       (datos que se pierden)   (tiempo sin servicio)
```

Ejemplo: una tienda en línea con backup cada noche a las 00:00 sufre un fallo a las 18:00. Pierde 18 horas de pedidos; si su RPO era 4 horas, el plan de backups no cumple y hay que pasar a replicación o backups cada hora.

Resiliencia sin backup: alta disponibilidad con redundancia (dos servidores tras un balanceador, RAID, fuentes de alimentación dobles, SAI), y sitios de recuperación: hot site (listo y sincronizado, minutos), warm site (equipo listo, datos no del todo al día, horas o días) y cold site (local con energía y red, sin equipos, semanas).

Ejemplo concreto con restic, una herramienta libre de backup cifrado:

```
$ restic -r /mnt/nas/repo backup ~/proyectos
Files:        1200 new,     0 changed,     0 unmodified
Added to the repository: 850.000 MiB
snapshot 4f2a9c1e saved

$ restic -r /mnt/nas/repo restore 4f2a9c1e --target /tmp/prueba-restore
restoring <Snapshot 4f2a9c1e> to /tmp/prueba-restore
```

> [!WARNING]
> RAID y la réplica no son backup: si borras un archivo o el ransomware lo cifra, el cambio se copia al instante a todos los discos. Y un backup que nunca se ha probado a restaurar no se sabe si sirve.

```
Backup → copia aparte para restaurar tras una pérdida.
Resiliencia → seguir funcionando o volver rápido cuando algo falla.
3-2-1 → 3 copias, 2 soportes, 1 fuera del sitio (+1 inmutable, 0 errores).
Completo → todo; restaurar fácil, ocupa más.
Incremental → cambios desde el último backup; rápido, restaurar exige la cadena entera.
Diferencial → cambios desde el último completo; restaurar = completo + último diferencial.
RPO → cuántos datos (en tiempo) se pueden perder.
RTO → cuánto tiempo puede durar la caída.
Hot / warm / cold site → minutos / horas-días / semanas para volver.
```

## Arquitectura de red segura

### Perimiter vs DMZ vs Segmentation

El **perímetro** es la frontera entre la red interna de una organización e Internet, controlada por firewalls y gateways; la **DMZ** (demilitarized zone, o screened subnet) es una subred intermedia donde se colocan los servicios que deben ser accesibles desde Internet, aislada tanto de Internet como de la red interna; la **segmentación** es la división de la red interna en zonas separadas cuyo tráfico entre sí pasa por controles.

Los tres existen para lo mismo, limitar a dónde puede llegar un atacante, a escalas distintas. El perímetro decide qué entra desde fuera. La DMZ resuelve un problema concreto: hay servidores (web, correo, DNS público) que Internet tiene que alcanzar, y si se comprometen no deben dar paso a la red interna. La segmentación resuelve lo que el perímetro no puede: una vez el atacante está dentro, evitar que salte de un portátil comprometido al servidor de base de datos (movimiento lateral).

Analogía: un edificio de oficinas. El perímetro es la puerta principal con seguridad; la DMZ es el vestíbulo de recepción, donde entran visitantes y mensajeros pero desde donde no se puede subir a las plantas; la segmentación son las puertas con tarjeta de cada planta y de cada departamento.

Topología típica con un firewall de tres patas:

```
            Internet
               |
        +--------------+
        |   Firewall   |
        +--------------+
         |            |
   +-----------+   +-------------------------------+
   |   DMZ     |   |   Red interna (segmentada)    |
   | web :443  |   |  VLAN 10 usuarios             |
   | mail :25  |   |  VLAN 20 servidores           |
   | DNS :53   |   |  VLAN 30 gestión              |
   +-----------+   |  VLAN 40 IoT / cámaras        |
                   +-------------------------------+
```

Reglas típicas del firewall:

```
Internet -> DMZ web      tcp/443          PERMITIR
Internet -> red interna  cualquiera       DENEGAR
DMZ web  -> VLAN 20 BD   tcp/5432         PERMITIR (solo el puerto de la base de datos)
DMZ      -> red interna  resto            DENEGAR
VLAN 10  -> VLAN 30      cualquiera       DENEGAR (los usuarios no tocan la gestión)
```

Tipos de segmentación, de más gruesa a más fina:

- Física: redes y switches separados (air gap en el extremo).
- Lógica con VLAN y subredes, con un firewall o ACL entre ellas. Repaso de subredes en [`03-fundamentos-de-redes`](../03-fundamentos-de-redes/).
- Microsegmentación: reglas por carga de trabajo o por aplicación, incluso entre dos máquinas de la misma subred; es la pieza de red de Zero Trust.

Ejemplo: un atacante compromete la impresora de red (VLAN 40). Sin segmentación, desde ahí escanea toda la 10.0.0.0/16 y llega al controlador de dominio. Con segmentación, la VLAN 40 solo puede hablar con el servidor de impresión en tcp/9100, y el escaneo devuelve todo filtrado.

> [!TIP]
> La diferencia de examen: el perímetro separa dentro de fuera; la DMZ es el sitio de lo que tiene que estar expuesto; la segmentación separa dentro de dentro.

```
Perímetro → frontera entre red interna e Internet; filtra lo que entra y sale.
DMZ (screened subnet) → zona para servicios públicos, aislada de Internet y de la red interna.
Segmentación → divide la red interna en zonas; frena el movimiento lateral.
Microsegmentación → reglas por carga de trabajo; base de red de Zero Trust.
```

## Equipos y adversario

### Blue / Red / Purple Teams

El **Blue Team** es el equipo defensivo que protege, monitoriza, detecta y responde a ataques; el **Red Team** es un equipo ofensivo autorizado que emula a un adversario real para comprobar si la organización lo detiene y lo detecta; el **Purple Team** es una forma de trabajo colaborativa en la que rojos y azules ejercitan juntos cada técnica para mejorar la detección en el momento.

Existen porque una defensa que nunca se pone a prueba solo se sabe buena en teoría. El red team contesta "¿nos detectarían?", el blue team convierte la respuesta en mejoras y el purple team acorta el camino entre ambas cosas.

Analogía: un equipo de fútbol. El blue team es la defensa del partido real; el red team es un rival de entrenamiento que copia el estilo del próximo contrincante sin avisar de sus jugadas; el purple team es el entrenamiento en el que el rival ejecuta una jugada, se para el juego, la defensa la analiza y se repite hasta que la defiende bien.

- Blue team: analistas de SOC, ingenieros de detección, respuesta a incidentes, hardening. Herramientas: SIEM, EDR, IDS, gestión de vulnerabilidades.
- Red team: objetivos concretos ("llegar a la base de datos de nóminas"), duración larga (semanas), sigilo, emulación de TTPs (tácticas, técnicas y procedimientos) de un grupo real según MITRE ATT&CK ([`13-frameworks-de-amenazas`](../13-frameworks-de-amenazas/)). Normalmente el blue team no sabe cuándo ocurre.
- Purple team: el red team ejecuta una técnica, el blue team comprueba si saltó alerta, se ajusta la regla y se repite. No siempre es un equipo fijo, a menudo es un ejercicio.
- White team: los árbitros; fijan reglas de enfrentamiento (alcance, horarios, qué está prohibido) y supervisan el ejercicio.

Red team y pentest no son lo mismo. Un pentest (prueba de penetración) busca el mayor número de vulnerabilidades en un alcance acotado, en días, sin necesidad de sigilo. Un red team busca un objetivo con sigilo para medir detección y respuesta, no para listar fallos.

Ejemplo: en un ejercicio purple de una tarde, el red team ejecuta en un laboratorio la técnica T1003 (volcado de credenciales de memoria). El SIEM no alerta. El blue team añade una regla de detección por acceso al proceso LSASS, el red team repite y esta vez salta la alerta en 30 segundos. Resultado: una técnica más detectada, documentada en la matriz ATT&CK del equipo.

```
Blue team → defiende: monitoriza, detecta, responde.
Red team → emula a un adversario real con sigilo para medir detección.
Purple team → rojos y azules juntos; técnica, comprobar alerta, ajustar, repetir.
White team → árbitros; reglas de enfrentamiento.
Pentest → máximas vulnerabilidades en alcance acotado; red team → un objetivo, sigilo.
```

### Privilege Escalation

La **escalada de privilegios** es una técnica de ataque en la que alguien que ya tiene un acceso limitado a un sistema consigue permisos que no le corresponden, aprovechando un fallo de configuración, una vulnerabilidad o credenciales expuestas.

Existe como fase de casi todo ataque porque el acceso inicial (un phishing, una aplicación web vulnerable) suele dar una cuenta con pocos permisos: un usuario normal o la cuenta de servicio `www-data`. Para robar datos, desactivar defensas o moverse por la red, el atacante necesita más. MITRE ATT&CK la recoge como táctica TA0004.

Analogía: un hotel. Entrar como huésped te da la tarjeta de tu habitación. Escalada vertical es conseguir la llave maestra del gerente; escalada horizontal es conseguir que tu tarjeta abra la habitación de otro huésped.

- Vertical: de menos a más privilegio. Usuario normal a root o administrador; cuenta de cliente a cuenta de administración de la aplicación.
- Horizontal: mismo nivel de privilegio, pero sobre los datos o la identidad de otro usuario. Cambiar `?id=1001` por `?id=1002` en una web y ver la factura de otra persona (un IDOR, referencia directa insegura a objetos, en [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/)).

Vectores habituales, a nivel conceptual:

- Software sin parchear: un fallo del kernel o de un servicio que corre como root.
- Configuraciones permisivas: reglas de `sudo` demasiado amplias, binarios con el bit SUID que no deberían tenerlo, servicios de Windows con rutas sin comillas o permisos de escritura, tareas programadas que ejecutan un script editable por cualquiera.
- Credenciales expuestas: contraseñas en archivos de configuración, historial de shell, variables de entorno.
- Fallos de autorización en aplicaciones: el servidor no comprueba en cada petición si el usuario puede hacer esa acción.

Ejemplo defensivo: auditar tu propio servidor Linux de laboratorio para encontrar lo que un atacante buscaría.

```
$ id
uid=1001(ana) gid=1001(ana) groups=1001(ana)
$ sudo -l
User ana may run the following commands on lab:
    (root) NOPASSWD: /usr/bin/vim        # problema: vim puede abrir una shell como root
$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/passwd
/usr/bin/sudo
/usr/local/bin/backup-tool              # SUID no estándar: revisar
```

La regla de sudo que permite `vim` como root equivale a dar root: el editor puede lanzar comandos. La corrección es dar permiso solo al comando exacto que la tarea necesita.

Defensas:

- Mínimo privilegio en usuarios, servicios y cuentas de aplicación; nada corre como root si no lo necesita.
- Parcheo regular de sistema operativo y servicios.
- Revisar `sudoers`, binarios SUID y permisos de servicios con herramientas de auditoría (Lynis, benchmarks CIS, ver [`14-defensa-y-hardening`](../14-defensa-y-hardening/)).
- Gestión de cuentas privilegiadas (PAM): contraseñas de administrador en una bóveda, acceso just-in-time, sesiones grabadas.
- En aplicaciones, comprobar la autorización en el servidor en cada petición, con denegación por defecto.
- Detectar: alertas por nuevos miembros en grupos de administradores, uso de `sudo` fuera de lo habitual, procesos hijos raros de servicios web.

> [!NOTE]
> La escalada de privilegios no es el acceso inicial: siempre presupone que el atacante ya está dentro con algún permiso. Por eso el mínimo privilegio limita el daño incluso cuando la primera barrera ya falló.

```
Escalada vertical → de menos a más privilegio (usuario a root/admin).
Escalada horizontal → mismo nivel, datos o identidad de otro usuario (IDOR).
SUID → binario que se ejecuta con los permisos de su dueño; si es root y abusable, escalada.
PAM → gestión de cuentas privilegiadas: bóveda, just-in-time, sesiones grabadas.
```

## Detección y operación

### False Negative / False Positive

Un **falso positivo** es una alerta que se dispara ante una actividad legítima (el sistema dice "ataque" y no lo es); un **falso negativo** es un ataque real que el sistema no detecta (el sistema calla y sí había ataque).

Existen como conceptos porque toda herramienta de detección (antivirus, IDS, SIEM, filtro de spam, escáner de vulnerabilidades) decide con información incompleta y se equivoca de las dos maneras. Los dos errores tienen costes muy distintos, y casi siempre bajar uno sube el otro.

Analogía: una alarma de casa. Falso positivo: suena porque pasó el gato. Falso negativo: entra un ladrón y no suena. Si el gato la dispara diez veces por semana, la familia acaba desconectándola, y entonces todos los robos son falsos negativos.

- Coste del falso positivo: tiempo de analista perdido, servicios bloqueados sin motivo (un antivirus que pone en cuarentena un programa legítimo de contabilidad) y, sobre todo, fatiga de alertas: los analistas dejan de mirar con atención.
- Coste del falso negativo: el ataque sigue su curso sin que nadie actúe; es el error más peligroso porque no deja rastro visible en el momento.

El umbral de la regla mueve el equilibrio. Ejemplo: una regla de fuerza bruta que alerta con "más de N intentos fallidos de login en 5 minutos".

```
N = 3    → salta con usuarios que olvidan la contraseña: muchos falsos positivos
N = 50   → un atacante lento que prueba 20 contraseñas cada 5 minutos pasa sin alerta:
           falsos negativos
N = 10 + condición "desde IP no vista antes" → menos ruido sin perder los ataques típicos
```

> [!WARNING]
> Un sistema "sin alertas" no es un sistema seguro: puede ser uno lleno de falsos negativos. Ajustar reglas solo para bajar el ruido sin medir lo que se deja de ver es la trampa clásica.

```
Falso positivo → alerta sin ataque; cuesta tiempo y produce fatiga de alertas.
Falso negativo → ataque sin alerta; el error más peligroso.
Umbral → subirlo baja falsos positivos y sube falsos negativos, y al revés.
```

### True Negative / True Positive

Un **verdadero positivo** es una alerta que corresponde a un ataque real (el sistema dice "ataque" y acierta); un **verdadero negativo** es la ausencia de alerta ante actividad legítima (el sistema calla y acierta).

Junto con los dos errores anteriores forman la matriz de confusión, una tabla de cuatro casillas que cruza lo que dijo el sistema con lo que pasó en realidad:

```
                         Realidad: ataque        Realidad: legítimo
Sistema: alerta          TP (verdadero pos.)     FP (falso positivo)
Sistema: sin alerta      FN (falso negativo)     TN (verdadero neg.)
```

Truco para leer los nombres: la segunda palabra (positivo/negativo) es lo que dijo el sistema; la primera (verdadero/falso) es si acertó.

De la matriz salen las métricas de un detector:

```
Recall (tasa de detección)   = TP / (TP + FN)  → de los ataques reales, cuántos vi
Precisión                    = TP / (TP + FP)  → de mis alertas, cuántas eran reales
Tasa de falsos positivos     = FP / (FP + TN)  → de lo legítimo, cuánto marqué mal
```

Analogía: un detector de metales en un aeropuerto. Pita con un cuchillo (TP), no pita con un pasajero limpio (TN), pita con un cinturón (FP), no pita con un arma de cerámica (FN).

Ejemplo de triaje en un SOC: en un turno llegan 200 alertas de "ejecución de PowerShell codificado". El analista revisa y concluye que 6 eran intrusiones reales (TP) y 194 eran el script de inventario de IT (FP). Después, el equipo de forense descubre 2 intrusiones que no generaron alerta (FN). Precisión = 6 / 200 = 3 %; recall = 6 / 8 = 75 %. La acción correcta es excluir el script de inventario por su hash y firma (no por su nombre, que el atacante podría copiar), lo que sube la precisión sin tocar el recall, y buscar por qué se escaparon los 2 FN.

```
TP → alerta y era ataque: acierto.
TN → sin alerta y era legítimo: acierto.
FP → alerta y era legítimo: error.
FN → sin alerta y era ataque: error.
Recall = TP/(TP+FN) → cuántos ataques veo.
Precisión = TP/(TP+FP) → cuántas alertas valen la pena.
```

### Con eventos raros, casi todas las alertas son falsas

> [!IMPORTANT]
> Cuando los ataques son muy pocos frente a la actividad normal, hasta un detector con una tasa de error minúscula produce muchas más alertas falsas que verdaderas; por eso la precisión, y no la tasa de detección, decide si un SOC se ahoga.

Esta idea, la falacia de la tasa base, es una consecuencia directa de la matriz de confusión: la tasa de falsos positivos se aplica sobre el enorme volumen de actividad legítima, mientras que la tasa de detección se aplica sobre el puñado de ataques reales.

```
Eventos legítimos × tasa FP   frente a   ataques reales × recall
```

```
                          Detector "bueno"            Mismo detector
                          (10 000 eventos/día,        (10 000 eventos/día,
                           1 000 ataques)              10 ataques)
Recall                    90 %                        90 %
Tasa de falsos positivos  1 %                         1 %
TP al día                 900                         9
FP al día                 90                          ~100
Precisión                 91 %                        8 %
```

Las dos primeras filas son idénticas y la última cambia de 91 % a 8 %: el detector es el mismo, lo que cambió es lo raro que es el ataque. En la realidad los ataques son la segunda columna: de cada 12 alertas, 11 son falsas.

Ejemplo cotidiano: una prueba médica con 99 % de acierto para una enfermedad que tiene 1 de cada 10 000 personas da, al hacérsela a toda una ciudad, cien positivos falsos por cada enfermo real.

Límite: no significa que la detección no sirva, sino que hay que medir y ajustar la precisión (correlación de varias señales, listas de exclusión bien hechas, contexto del activo) y no solo comprobar que "salta con el ataque de prueba".

```
Falacia de la tasa base → con ataques raros, la mayoría de alertas son falsas positivas.
Precisión → la métrica que decide la carga del SOC.
```

### Understand Concept of Runbooks

Un **runbook** es un documento operativo con los pasos concretos, ordenados y verificables para ejecutar una tarea o responder a una situación específica, escrito para que cualquier operador cualificado obtenga el mismo resultado; un **playbook** es un documento de nivel superior que describe la estrategia completa para un tipo de incidente (quién decide, qué fases, a quién se avisa) y que remite a varios runbooks para la ejecución.

Existen porque en un incidente, a las 3:00 y con presión, la gente olvida pasos, improvisa y cada analista lo hace distinto. El runbook convierte el conocimiento de los expertos en un procedimiento repetible, reduce el tiempo de respuesta, sirve para formar a los nuevos y es la base para automatizar (las plataformas SOAR, de orquestación y respuesta automatizada, ejecutan runbooks como código).

Analogía: en una cocina, el playbook es el plan para un banquete de 200 personas (qué se sirve, en qué orden, quién lleva cada estación); el runbook es la receta exacta de un plato, con cantidades y tiempos.

Estructura típica de un runbook:

```
Título:        Contener un endpoint con malware confirmado
Disparador:    alerta EDR de severidad alta confirmada como TP
Requisitos:    acceso a la consola EDR, rol "SOC L2"
Pasos:
  1. Aislar el equipo desde la consola EDR (mantiene la conexión con la consola).
  2. Verificar: el equipo no responde a ping desde la VLAN de usuarios.
  3. Recoger triage: lista de procesos, conexiones, hash del binario.
  4. Deshabilitar la cuenta del usuario en el directorio y revocar sesiones.
  5. Abrir ticket de incidente con los artefactos adjuntos.
Escalado:      si hay más de 3 equipos afectados → activar playbook de ransomware.
Rollback:      quitar aislamiento solo tras reimagen aprobada.
```

Un buen runbook tiene un disparador claro, pasos atómicos (una acción por paso), un criterio de verificación tras cada paso, puntos de escalado y se revisa después de cada uso. Las fases de respuesta a incidentes en las que se encajan están en [`16-respuesta-a-incidentes-y-forense`](../16-respuesta-a-incidentes-y-forense/).

Ejemplo de jerarquía: el playbook "Phishing" define las fases (triage, contención, erradicación, comunicación) y llama a los runbooks "Bloquear remitente en la pasarela de correo", "Buscar y borrar el correo en todos los buzones" y "Restablecer credenciales del usuario".

```
Playbook → estrategia para un tipo de incidente: fases, roles, decisiones.
Runbook → pasos concretos y verificables de una tarea.
SOAR → plataforma que automatiza runbooks.
```

## Recursos para aprender y practicar

### Videos

- [What is the CIA Triad](https://www.youtube.com/watch?v=kPPFNrlN3zo) — IBM Technology; las tres propiedades con ejemplos. Nodo: Understand CIA Triad.
- [The CIA Triad - CompTIA Security+ SY0-701 - 1.2](https://www.youtube.com/watch?v=SBcDGb9l6yo) — Professor Messer; versión de examen con hashing y firmas para integridad. Nodo: Understand CIA Triad.
- [Network Security | Defense in Depth](https://www.youtube.com/watch?v=IiWFMIgKaqQ) — Network Direction; capas de defensa en una red real. Nodo: Defense in Depth.
- [Zero Trust Explained in 4 mins](https://www.youtube.com/watch?v=yn6CPQ9RioA) — IBM Technology; principios de verificar siempre y asumir brecha. Nodo: Core Concepts of Zero Trust.
- [Risk Analysis - CompTIA Security+ SY0-701 - 5.2](https://www.youtube.com/watch?v=Ykx7t54y-oo) — Professor Messer; análisis cualitativo y cuantitativo, SLE, ARO, ALE. Nodo: Definition of Risk.
- [Backups - CompTIA Security+ SY0-701 - 3.4](https://www.youtube.com/watch?v=8mGSwRScqIM) — Professor Messer; tipos de backup, frecuencia, cifrado y snapshots. Nodo: Backups and Resiliency.
- [RPO and RTO Explained](https://www.youtube.com/watch?v=rD3nBaS3OG4) — Amazon Web Services; las dos métricas de recuperación. Nodo: Backups and Resiliency.
- [What is a DMZ? (Demilitarized Zone)](https://www.youtube.com/watch?v=dqlzQXo1wqo) — PowerCert Animated Videos; DMZ con uno y dos firewalls. Nodo: Perimiter vs DMZ vs Segmentation.
- [Segmentation and Access Control - CompTIA Security+ SY0-701 - 2.5](https://www.youtube.com/watch?v=yDeDGCh_PDs) — Professor Messer; segmentación, ACL y listas de permitidos. Nodo: Perimiter vs DMZ vs Segmentation.
- [Penetration Tests - CompTIA Security+ SY0-701 - 5.5](https://www.youtube.com/watch?v=wEMzVfwBiWY) — Professor Messer; pruebas ofensivas, defensivas e integradas (rojo, azul, púrpura). Nodo: Blue / Red / Purple Teams.
- [Privilege Escalation - SY0-601 CompTIA Security+ : 1.3](https://www.youtube.com/watch?v=ksjU3Iu195Q) — Professor Messer; escalada vertical y horizontal y sus mitigaciones. Nodo: Privilege Escalation.
- [False Positives and False Negatives - CompTIA Security+ SY0-401: 2.1](https://www.youtube.com/watch?v=bUNBzMnfHLw) — Professor Messer; los dos tipos de error de un detector. Nodos: False Negative / False Positive y True Negative / True Positive.
- [Same Alert. Real Attack or False Positive? (How SOC Analysts Decide)](https://www.youtube.com/watch?v=inIjEbWE8FM) — MyDFIR; triaje real de una alerta para decidir TP o FP. Nodo: True Negative / True Positive.
- [What is a playbook/runbook in SOC?](https://www.youtube.com/watch?v=ap24Ka-T5v8) — Rajneesh Gupta; diferencia entre playbook y runbook en un SOC. Nodo: Runbooks.

### Lectura y documentación

- [NIST SP 800-207, Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final) — la definición de referencia: principios, PEP/PDP y modelos de despliegue.
- [CISA Zero Trust Maturity Model](https://www.cisa.gov/zero-trust-maturity-model) — los cinco pilares y las etapas de madurez.
- [Microsoft: Zero Trust overview](https://learn.microsoft.com/en-us/security/zero-trust/zero-trust-overview) — verify explicitly, least privilege y assume breach aplicados.
- [NIST SP 800-30 Rev. 1, Guide for Conducting Risk Assessments](https://csrc.nist.gov/pubs/sp/800/30/r1/final) — amenaza, vulnerabilidad, probabilidad e impacto en el método de NIST.
- [NIST SP 800-34 Rev. 1, Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final) — BIA, RTO, RPO, MTD y sitios de recuperación.
- [CISA #StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide) — backups offline y cifrados, segmentación y respuesta ante ransomware.
- [MITRE ATT&CK TA0004 Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) — catálogo de técnicas de escalada con detecciones y mitigaciones.
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — mínimo privilegio, denegar por defecto y escalada horizontal y vertical en aplicaciones.
- [CISA Federal Government Cybersecurity Incident and Vulnerability Response Playbooks](https://www.cisa.gov/resources-tools/resources/federal-government-cybersecurity-incident-and-vulnerability-response-playbooks) — ejemplo oficial de playbooks con fases y listas de comprobación.

### Práctica

- [TryHackMe: Security Principles](https://tryhackme.com/room/securityprinciples) — gratis; CIA, DAD, defensa en profundidad, Zero Trust y modelos de seguridad. Nodos: CIA Triad, Defense in Depth, Zero Trust.
- [TryHackMe: Red Team Fundamentals](https://tryhackme.com/room/redteamfundamentals) — qué es un compromiso de red team, roles y diferencias con un pentest. Nodo: Blue / Red / Purple Teams.
- [TryHackMe: SOC L1 Alert Triage](https://tryhackme.com/room/socl1alerttriage) — gratis; clasificar alertas como verdaderos o falsos positivos. Nodos: FN/FP y TN/TP.
- [TryHackMe: Linux Privilege Escalation](https://tryhackme.com/room/linprivesc) — vectores de escalada en Linux en una máquina de laboratorio. Nodo: Privilege Escalation.
- [TryHackMe: Windows PrivEsc](https://tryhackme.com/room/windows10privesc) — gratis; servicios, tareas y permisos mal configurados en Windows. Nodo: Privilege Escalation.
- [PortSwigger Web Security Academy: Access control](https://portswigger.net/web-security/access-control) — laboratorios gratuitos de escalada vertical y horizontal en aplicaciones web. Nodo: Privilege Escalation.
- [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) — los niveles con binarios SUID enseñan cómo un permiso mal dado entrega privilegios. Nodo: Privilege Escalation.
- [Ejercicio 1: Aplica la regla 3-2-1 y mide tu RPO y RTO](ejercicios.md#ejercicio-1-aplica-la-regla-3-2-1-y-mide-tu-rpo-y-rto) — copia una carpeta con restic en 3 copias, 2 soportes y 1 fuera del sitio, bórrala y mide el tiempo de restauración.
- [Ejercicio 2: Calcula SLE ALE y el valor de un control](ejercicios.md#ejercicio-2-calcula-sle-ale-y-el-valor-de-un-control) — pon en dinero el riesgo de tres activos tuyos y decide con el número qué respuesta al riesgo aplicas.
- [Ejercicio 3: Segmenta una red de laboratorio y compruébala con nmap](ejercicios.md#ejercicio-3-segmenta-una-red-de-laboratorio-y-compruébala-con-nmap) — monta DMZ, usuarios y servidores tras un firewall y demuestra con nmap que solo ves los puertos permitidos.
- [Ejercicio 4: Escribe un runbook de restauración y pruébalo](ejercicios.md#ejercicio-4-escribe-un-runbook-de-restauración-y-pruébalo) — vuelve "restaurar proyectos desde backup" un runbook de una página y valídalo con otra persona.

## Cuadro resumen

Todo lo visto, en una línea por término.

Principios de seguridad

```
Confidentiality → solo los autorizados leen; cifrado, control de acceso.
Integrity → nada cambia sin autorización y sin detectarse; hash, firma.
Availability → accesible cuando se necesita; redundancia, backups, anti-DDoS.
DAD → Disclosure, Alteration, Destruction: el reverso de CIA.
Defense in depth → varias capas independientes; si una falla, la siguiente frena o detecta.
Control preventivo / detectivo / correctivo → impide / avisa / repara.
Zero Trust → no confía por ubicación; verifica cada acceso, mínimo privilegio, asume brecha.
PEP → aplica la decisión (la puerta); PDP → decide con señales y abre o corta la sesión.
ZTNA → acceso a una app concreta tras verificar; VPN → acceso a toda la red.
```

Riesgo y resiliencia

```
Riesgo = Amenaza × Vulnerabilidad × Impacto → si un factor es cero, no hay riesgo.
SLE = AV × EF → pérdida por incidente.
ALE = SLE × ARO → pérdida esperada por año.
Valor del control = ALE antes − ALE después − coste anual.
Mitigar / Transferir / Evitar / Aceptar → las cuatro respuestas al riesgo.
Riesgo inherente → antes de controles; residual → lo que queda después.
Bajar vulnerabilidad o impacto → las palancas reales; la amenaza rara vez se controla.
3-2-1 → 3 copias, 2 soportes, 1 fuera del sitio (+1 inmutable, 0 errores).
Completo / Incremental / Diferencial → todo / desde el último backup / desde el último completo.
RPO → cuántos datos (en tiempo) se pueden perder; RTO → cuánto puede durar la caída.
Hot / warm / cold site → minutos / horas-días / semanas para volver.
RAID y réplica → no son backup: copian también el borrado y el cifrado.
```

Arquitectura de red segura

```
Perímetro → frontera entre red interna e Internet.
DMZ (screened subnet) → servicios públicos aislados de Internet y de la red interna.
Segmentación → divide la red interna en zonas; frena el movimiento lateral.
Microsegmentación → reglas por carga de trabajo; base de red de Zero Trust.
```

Equipos y adversario

```
Blue team → defiende: monitoriza, detecta, responde.
Red team → emula a un adversario real con sigilo para medir detección.
Purple team → rojos y azules juntos; técnica, comprobar alerta, ajustar, repetir.
White team → árbitros; reglas de enfrentamiento.
Pentest → máximas vulnerabilidades en alcance acotado; red team → un objetivo, sigilo.
Escalada vertical → de menos a más privilegio (usuario a root/admin).
Escalada horizontal → mismo nivel, datos o identidad de otro usuario (IDOR).
Defensas de escalada → mínimo privilegio, parches, revisar sudo/SUID, PAM, authz en servidor.
```

Detección y operación

```
Falso positivo → alerta sin ataque; cuesta tiempo y produce fatiga de alertas.
Falso negativo → ataque sin alerta; el error más peligroso.
TP → alerta y era ataque; TN → sin alerta y era legítimo.
Recall = TP/(TP+FN) → cuántos ataques veo.
Precisión = TP/(TP+FP) → cuántas alertas valen la pena.
Tasa FP = FP/(FP+TN) → cuánto de lo legítimo marco mal.
Falacia de la tasa base → con ataques raros, la mayoría de alertas son falsas positivas.
Playbook → estrategia para un tipo de incidente: fases, roles, decisiones.
Runbook → pasos concretos y verificables de una tarea.
SOAR → plataforma que automatiza runbooks.
```
