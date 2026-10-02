# Respuesta a incidentes y forense

## Conceptos previos

- Evento: cualquier cosa observable que pasa en un sistema (un login, una conexión, un archivo creado). La gran mayoría son normales.
- Incidente de seguridad: evento o conjunto de eventos que viola una política de seguridad o amenaza la confidencialidad, integridad o disponibilidad de un activo. Todo incidente es un evento; casi ningún evento es un incidente.
- IoC (Indicator of Compromise): rastro técnico que delata un ataque: un hash de malware, una IP o dominio de mando y control, una clave de registro rara, un nombre de servicio inventado.
- C2 (Command and Control): servidor desde el que el atacante da órdenes al malware ya instalado en la víctima.
- Movimiento lateral: cuando el atacante, ya dentro de un equipo, salta a otros equipos de la misma red.
- SIEM y EDR: el SIEM junta logs de toda la red y genera alertas; el EDR es un agente en cada equipo que registra procesos, conexiones y archivos y puede aislar el equipo. Detalle en [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/).
- Hash: huella de tamaño fijo de unos datos; si cambia un solo bit de los datos, cambia la huella entera. Detalle en [`10-criptografia`](../10-criptografia/).
- Playbook y runbook: el playbook es la estrategia para un tipo de incidente (ransomware, phishing); el runbook son los pasos concretos de una tarea. Detalle en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/).
- RAM (memoria volátil): memoria de trabajo del equipo; su contenido desaparece al apagarlo.
- Imagen de disco: copia bit a bit de un disco entero, incluido el espacio vacío y lo borrado.
- Write blocker: dispositivo o software que deja leer un disco pero bloquea cualquier escritura, para no alterar la evidencia.
- DFIR (Digital Forensics and Incident Response): nombre con el que la industria junta forense digital y respuesta a incidentes, porque en la práctica las hace el mismo equipo.
- CSIRT (Computer Security Incident Response Team): el equipo de la organización encargado de responder a incidentes.

## Understand the Incident Response Process

El **proceso de respuesta a incidentes** es un ciclo ordenado de fases que una organización sigue para prepararse, detectar, frenar, limpiar y recuperarse de un incidente de seguridad, y para aprender de él. Existe porque, sin un proceso escrito, cada incidente se maneja improvisando: alguien apaga el servidor y destruye la evidencia, nadie avisa a legal a tiempo, se limpia un equipo y el atacante sigue en otros diez.

El modelo de SANS tiene seis fases y se recuerda con la sigla PICERL: Preparation, Identification, Containment, Eradication, Recovery, Lessons Learned. Es un ciclo, no una línea: lo aprendido al final alimenta la preparación del siguiente incidente, y durante un incidente real se vuelve atrás (al erradicar aparece otro equipo infectado y hay que volver a identificar y contener).

```
          +--------------+
   +----> | Preparation  |  (antes del incidente)
   |      +------+-------+
   |             v
   |      +--------------+
   |      | Identification| <-----------+
   |      +------+-------+              |
   |             v                      | aparece otro
   |      +--------------+              | equipo afectado
   |      | Containment  |              |
   |      +------+-------+              |
   |             v                      |
   |      +--------------+              |
   |      | Eradication  | -------------+
   |      +------+-------+
   |             v
   |      +--------------+
   |      |  Recovery    |
   |      +------+-------+
   |             v
   |      +--------------+
   +----- |Lessons Learned|
          +--------------+
```

Analogía: un incendio en un edificio. Preparación es tener extintores, alarmas y simulacros; identificación es que suene la alarma y alguien confirme que hay fuego; contención es cerrar las puertas cortafuego para que no se extienda; erradicación es apagar el fuego y retirar lo que sigue humeando; recuperación es reparar y volver a abrir las oficinas; lecciones aprendidas es el informe de los bomberos que dice que el cableado del tercer piso era viejo.

### Preparation

**Preparation** es la fase previa al incidente en la que la organización arma todo lo que necesitará para responder: plan, equipo, herramientas, visibilidad y contactos. Es la única fase que ocurre con calma; todas las demás ocurren con el reloj corriendo.

Qué se prepara, en concreto:

- Plan de respuesta a incidentes (IRP): documento aprobado por la dirección que define qué es un incidente, niveles de severidad, quién decide qué y a quién se avisa.
- Equipo (CSIRT) con roles claros: líder del incidente (incident commander), analistas técnicos, enlace con legal, comunicaciones y dirección. Con suplentes, porque los incidentes llegan un sábado a las 3 a. m.
- Playbooks por tipo de incidente: ransomware, phishing con robo de credenciales, cuenta de nube comprometida, fuga de datos, DDoS.
- Visibilidad: logs centralizados en un SIEM con retención suficiente (90 días en caliente y 1 año archivado es un punto de partida habitual), EDR en los equipos, sincronización horaria NTP en todo (sin relojes alineados no se puede reconstruir una línea de tiempo).
- Jump bag o kit de respuesta: discos externos limpios y grandes, write blocker, cables, una distribución forense en USB, herramientas portables (FTK Imager, WinPmem, scripts de recolección), formularios de cadena de custodia en papel.
- Canales fuera de banda: un grupo de mensajería o un puente telefónico que no dependa de la red comprometida. Si el atacante tiene el correo, lee los correos del equipo de respuesta.
- Inventario de activos y diagramas de red actualizados: no se puede contener lo que no se sabe que existe.
- Ejercicios: simulacros de mesa (tabletop) al menos una vez al año, donde se recorre un escenario hablando, sin tocar sistemas.

Analogía: un equipo de bomberos que entrena, revisa las mangueras y conoce el plano de cada edificio antes de que haya fuego.

Ejemplo: una empresa de 200 empleados hace un tabletop de ransomware. Al llegar a "aislamos los equipos", descubren que nadie sabe quién tiene permiso para desconectar el servidor de facturación y que el contacto del proveedor de backup está en un correo que estaría cifrado. Dos hallazgos que cuestan cero arreglar hoy y horas durante un incidente real.

Si en un examen preguntan en qué fase se hace un simulacro, se escribe el plan o se compra el write blocker, la respuesta es Preparation, aunque la idea haya salido de un incidente anterior.

```
IRP → plan aprobado: definiciones, severidades, roles, escalado.
Playbook → guía por tipo de incidente.
Jump bag → kit físico y de software listo antes del incidente.
Fuera de banda → canal de comunicación que no pasa por la red comprometida.
Tabletop → simulacro hablado de un escenario.
```

### Identification

**Identification** (también Detection) es la fase en la que se decide si un evento es realmente un incidente, qué tan grave es y qué alcance tiene. El problema que resuelve: un SOC recibe miles de alertas al día y casi todas son falsos positivos; hay que separar la señal del ruido y, cuando es real, entender qué se tocó.

Cómo funciona por dentro:

1. Llega una señal: alerta del SIEM o EDR, aviso de un usuario ("me salió una ventana rara"), aviso de un tercero (el banco, un investigador, la policía) o el propio atacante pidiendo rescate.
2. Triaje: se valida la alerta contra otras fuentes. ¿El proceso sospechoso existe? ¿El dominio contactado es malicioso según inteligencia de amenazas? ¿El usuario estaba trabajando a esa hora?
3. Declaración: si se confirma, se declara el incidente, se abre un ticket o registro con hora exacta, y se asigna una severidad (por ejemplo, de 1 a 4 según datos afectados, número de equipos y si el servicio está caído).
4. Alcance (scoping): se buscan los IoC encontrados en todo el entorno. Un hash de malware visto en un equipo se busca en los 500; la IP del C2 se busca en los logs del firewall de los últimos 30 días.
5. Documentación desde el minuto uno: quién vio qué, a qué hora, con qué herramienta. Ese registro alimenta la línea de tiempo y, si el caso llega a juicio, la evidencia.

Analogía: un médico de urgencias. Primero decide si el paciente que llega con dolor de pecho tiene un infarto o una indigestión (triaje); si es infarto, mide qué tan extendido está antes de operar.

Ejemplo: el EDR alerta de `powershell.exe -enc JABjAD0A...` (comando codificado en Base64) en el portátil de contabilidad a las 02:14. El analista decodifica el comando: descarga un archivo de `hxxp://update-check[.]example/a.ps1`. Busca ese dominio en el proxy: lo contactaron 3 equipos más en las últimas 48 horas. Ya no es un incidente de un equipo, son cuatro, y la severidad sube.

> [!WARNING]
> Identificar no es buscar al culpable ni limpiar. Si el analista borra el archivo malicioso en cuanto lo ve, el atacante se entera, cambia de táctica y se pierde la evidencia de cómo entró.

```
Evento → algo que pasa; casi siempre normal.
Alerta → evento que una regla marcó como sospechoso.
Incidente → alerta confirmada que viola una política o amenaza CIA.
Triaje → validar y priorizar alertas.
Scoping → medir alcance buscando los IoC en todo el entorno.
```

### Containment

**Containment** es la fase en la que se limita el daño del incidente impidiendo que el atacante avance, sin destruir todavía su presencia ni la evidencia. Existe porque erradicar lleva tiempo (hay que entender qué hizo el atacante) y mientras tanto el daño no puede seguir creciendo.

SANS la divide en tres momentos:

- Contención a corto plazo: frenar ya. Aislar el equipo de la red con el EDR (sigue encendido y con la RAM intacta, pero solo habla con la consola del EDR), mover el puerto del switch a una VLAN de cuarentena, bloquear la IP o dominio del C2 en el firewall y el DNS, deshabilitar la cuenta comprometida.
- Copia del sistema: antes de tocar nada más, capturar la memoria y la imagen del disco (ver la parte de forense). Es la última oportunidad de tener la evidencia tal como estaba.
- Contención a largo plazo: arreglos temporales que permiten seguir operando mientras se prepara la limpieza: parchear el servidor expuesto, rotar contraseñas de cuentas de servicio, poner reglas extra en el firewall, mover un servicio crítico a un servidor limpio.

Decisión clave: no siempre se contiene de inmediato. Si el atacante tiene acceso a 50 equipos y solo se conocen 4, bloquear esos 4 le avisa de que fue descubierto y puede activar el ransomware en los otros 46. A veces se observa en silencio unas horas para medir el alcance completo y luego se corta todo a la vez. Esa decisión la toma el líder del incidente con dirección y legal, nunca un analista solo.

Analogía: un médico que pone un torniquete. No cura la herida, pero evita que el paciente se desangre mientras llega el quirófano.

Ejemplo: un servidor web comprometido. Contención corta: regla en el firewall que bloquea su salida a Internet salvo al puerto 443 del balanceador; se captura su RAM con LiME y su disco con `dd`. Contención larga: se levanta una copia del sitio en un servidor nuevo y parcheado detrás del mismo balanceador, y el comprometido queda aislado para análisis.

> [!NOTE]
> Desenchufar el cable o apagar el equipo también contiene, pero destruye la RAM: procesos, conexiones activas y claves de cifrado se pierden. Aislar por red (EDR o VLAN) consigue lo mismo sin perder la evidencia volátil.

```
Contención corta → frenar ya: aislar, bloquear C2, deshabilitar cuentas.
Copia del sistema → memoria e imagen de disco antes de seguir.
Contención larga → arreglos temporales para seguir operando.
Aislar por red → contiene y conserva la RAM; apagar → contiene y la destruye.
```

### Eradication

**Eradication** es la fase en la que se elimina por completo la presencia del atacante y la causa que le permitió entrar. Existe porque contener no es limpiar: el malware sigue en el disco, las cuentas que creó siguen existiendo y la vulnerabilidad sigue abierta.

Qué se hace:

- Encontrar la causa raíz (root cause): cómo entró. Un correo de phishing, una VPN sin MFA, un servidor sin parchear, una contraseña reutilizada.
- Eliminar todos los mecanismos de persistencia: servicios y tareas programadas creadas por el atacante, claves `Run` del registro, cuentas nuevas, claves SSH añadidas a `authorized_keys`, web shells, reglas de reenvío de correo.
- Reconstruir en vez de limpiar cuando hay duda: reinstalar desde una imagen conocida y limpia (gold image) es más fiable que borrar archivos uno por uno, porque un solo mecanismo de persistencia olvidado basta para que el atacante vuelva.
- Cerrar la puerta: aplicar el parche, activar MFA, rotar todas las credenciales que pudieron verse expuestas. Si el atacante llegó al controlador de dominio, eso incluye resetear dos veces la cuenta `krbtgt` (la que firma los tickets Kerberos), con un intervalo de al menos 10 horas (la vida por defecto de un ticket) entre resets.

Analogía: sacar las termitas de una casa. Matar las que se ven (contención) no basta; hay que encontrar el nido, tratar toda la madera y sellar la grieta por la que entraron.

Ejemplo: tras un phishing, se encuentran en 4 equipos una tarea programada `OneDriveUpdate` que ejecuta un script cada 30 minutos y una cuenta local `support01` creada a las 03:10. Se reimaginan los 4 equipos, se elimina la cuenta en todos los demás donde aparezca, se resetea la contraseña de los 4 usuarios y se bloquea el dominio del correo original.

```
Causa raíz → por dónde entró el atacante.
Persistencia → lo que el atacante dejó para volver (tareas, servicios, cuentas, claves).
Reimaginar → reinstalar desde imagen limpia; más fiable que limpiar a mano.
Contención → frena; erradicación → elimina.
```

### Recovery

**Recovery** es la fase en la que los sistemas limpios vuelven a producción de forma controlada y se vigila que el atacante no regrese. El problema que resuelve: volver a operar sin reintroducir el compromiso, porque restaurar un backup infectado o reconectar un equipo con una puerta trasera olvidada reinicia el incidente.

Cómo se hace:

- Restaurar desde backups verificados como limpios: el backup tiene que ser anterior a la primera actividad del atacante (por eso importa la línea de tiempo). Si el atacante entró hace 40 días y los backups se guardan 30, todos están contaminados.
- Volver por etapas: primero los sistemas críticos, con pruebas funcionales y de seguridad antes de abrirlos a usuarios.
- Monitorización reforzada: reglas de detección específicas con los IoC del incidente durante un periodo definido (30 a 90 días es habitual), porque los atacantes con frecuencia intentan volver.
- Criterio de cierre: la dirección del negocio, no solo TI, aprueba cuándo un sistema vuelve a producción.

Analogía: un paciente que sale del hospital. Recibe el alta, pero tiene controles a la semana y al mes, porque las recaídas aparecen justo después.

Ejemplo: un ransomware cifró 20 servidores. La línea de tiempo dice que el atacante entró el 3 de marzo. Se descartan los backups posteriores a esa fecha, se restaura el del 1 de marzo en servidores reinstalados, se aplican los parches y se repiten los datos de los días perdidos desde otras fuentes (correo, facturas en papel). Durante 60 días, una regla del SIEM alerta de cualquier conexión a las IP del C2.

```
Backup limpio → anterior a la primera actividad del atacante.
Vuelta por etapas → críticos primero, con pruebas antes de abrir.
Monitorización reforzada → reglas con los IoC del incidente, 30-90 días.
Erradicación → elimina al atacante; recovery → devuelve el servicio sin él.
```

### Lessons Learned

**Lessons Learned** (post-incident activity) es la fase final en la que el equipo revisa qué pasó, qué funcionó y qué no, y convierte esas conclusiones en cambios concretos. Existe porque un incidente que se cierra sin revisión se repite: la misma vulnerabilidad, el mismo retraso al avisar, la misma herramienta que faltaba.

Cómo se hace:

- Reunión post-incidente (post-mortem) dentro de las 2 semanas siguientes, mientras la memoria está fresca, con todos los que participaron.
- Formato sin culpables (blameless): se pregunta qué falló en el proceso, no quién se equivocó. Si la gente teme castigo, oculta información y la próxima vez nadie reporta a tiempo.
- Informe final: línea de tiempo completa, causa raíz, alcance, impacto (equipos, datos, horas de caída, coste), qué se hizo en cada fase y recomendaciones con responsable y fecha.
- Métricas: tiempo medio de detección (MTTD, desde que el atacante entra hasta que se detecta) y tiempo medio de respuesta o contención (MTTR). Bajar el MTTD de 30 días a 2 es una mejora que se mide.
- Cerrar el ciclo: cada recomendación vuelve a Preparation (nueva regla del SIEM, nuevo paso en un playbook, nuevo control).

Analogía: la revisión que hace un equipo de fútbol viendo el video del partido perdido. No se trata de señalar al portero, sino de ver por qué la defensa dejó ese hueco.

Ejemplo de preguntas de la reunión: ¿cuánto pasó entre la primera alerta y la declaración del incidente? (respuesta: 9 horas, porque la alerta cayó en una cola que nadie revisaba de noche). Acción: la cola de severidad alta pasa a notificar al teléfono de guardia. Responsable: jefe del SOC. Fecha: 30 días.

```
Post-mortem → reunión de revisión, dentro de 2 semanas, sin culpables.
Informe final → timeline, causa raíz, impacto, recomendaciones con dueño y fecha.
MTTD → tiempo hasta detectar; MTTR → tiempo hasta responder/contener.
Lessons Learned → alimenta Preparation: el ciclo se cierra.
```

### SANS y NIST cuentan el mismo ciclo con distinto corte

> [!IMPORTANT]
> PICERL (SANS) y el ciclo de NIST SP 800-61 describen las mismas actividades: lo único que cambia es dónde se dibujan las fronteras entre fases. Saber traducir de uno a otro es lo que pregunta el examen.

La correspondencia entre modelos es una tabla de traducción que muestra que los seis pasos de SANS caben en las cuatro fases del NIST SP 800-61 Rev. 2 (2012), el documento clásico de NIST sobre manejo de incidentes.

```
                 SANS (PICERL, 6)          NIST 800-61 Rev. 2 (4)
Antes            Preparation               Preparation
Detectar         Identification            Detection & Analysis
Frenar           Containment        \
Limpiar          Eradication         >     Containment, Eradication & Recovery
Volver           Recovery           /
Después          Lessons Learned           Post-Incident Activity
```

La fila "Frenar / Limpiar / Volver" es la diferencia entera: SANS separa en tres pasos lo que NIST agrupa en uno, porque en la práctica contener, erradicar y recuperar se entrelazan (se contiene un equipo mientras se erradica otro). NIST además dibuja una flecha de vuelta desde esa fase a Detection & Analysis, que es la misma idea del bucle de SANS.

En abril de 2025 NIST publicó la SP 800-61 Rev. 3, que reemplaza a la Rev. 2 y abandona el ciclo de cuatro fases: organiza la respuesta a incidentes según las seis funciones del NIST Cybersecurity Framework 2.0 (Govern, Identify, Protect, Detect, Respond, Recover), integrándola en la gestión de riesgos general. Las actividades son las mismas; lo que cambia es que la preparación se reparte entre Govern, Identify y Protect, y las lecciones aprendidas se vuelven mejora continua en todas las funciones. Los exámenes como Security+ siguen usando el ciclo clásico, así que conviene saber los tres.

Analogía: dividir un viaje en "ida, estancia, vuelta" o en "maleta, aeropuerto, vuelo, hotel, playa, regreso". Es el mismo viaje contado con más o menos cortes.

Ningún modelo es "más correcto". Si la organización ya usa el CSF de NIST, la Rev. 3 encaja mejor; si el equipo quiere checklists por fase, PICERL es más práctico.

```
PICERL (SANS) → 6 fases: P, I, C, E, R, L.
NIST 800-61 Rev. 2 → 4 fases; junta C, E y R en una.
NIST 800-61 Rev. 3 (abril 2025) → reemplaza el ciclo por las 6 funciones del CSF 2.0.
Identification (SANS) = Detection & Analysis (NIST).
Lessons Learned (SANS) = Post-Incident Activity (NIST).
```

## Understand Basics of Forensics

La **forense digital** es la disciplina que identifica, recoge, preserva y analiza evidencia digital de forma que el resultado sea reproducible y aceptable ante un tribunal o una auditoría. En respuesta a incidentes responde las preguntas que la contención no responde: cómo entró el atacante, qué hizo, qué datos tocó y desde cuándo. NIST SP 800-86 resume el proceso en cuatro pasos: recolección, examen, análisis y reporte.

Analogía: la policía científica en la escena de un crimen. Antes de mover el cuerpo, fotografía todo, usa guantes, mete cada objeto en una bolsa sellada con etiqueta y anota quién la llevó al laboratorio. Si se salta un paso, el abogado defensor pide que la prueba no valga.

### Order of volatility

El **orden de volatilidad** es una regla de recolección que dice que la evidencia se recoge empezando por la que desaparece antes y terminando por la que dura más. Existe porque cada acción en un sistema vivo destruye algo: ejecutar una herramienta sobrescribe RAM, esperar deja que expiren conexiones, apagar borra la memoria entera.

El RFC 3227 (Guidelines for Evidence Collection and Archiving, 2002) fija este orden, de más a menos volátil:

```
1. Registros de CPU y caché                  nanosegundos
2. Tabla de rutas, caché ARP, tabla de       segundos a minutos
   procesos, estadísticas del kernel, RAM
3. Sistemas de archivos temporales           hasta reiniciar
4. Disco                                     años
5. Logs y monitorización remotos             según retención
6. Configuración física, topología de red    meses
7. Medios de archivo (cintas, backups)       años
```

En la práctica, el analista no puede capturar registros de CPU con herramientas normales, así que el primer paso real es la RAM y la información de red (conexiones activas, caché ARP y DNS), luego el disco, luego los logs remotos.

Analogía: recoger huellas en la playa. Primero se fotografían las pisadas cerca del agua (la marea las borra en minutos), después las de la arena seca y al final las del muelle de cemento, que siguen ahí mañana.

Ejemplo: en un servidor Windows comprometido, la secuencia sería: (1) volcar la RAM con WinPmem o FTK Imager a un disco externo (2 a 5 minutos para 16 GB); (2) guardar `ipconfig /displaydns`, `arp -a` y `netstat -ano`; (3) apagar e imaginar el disco con un write blocker; (4) pedir al equipo de red los logs del firewall de los últimos 30 días.

Cada herramienta que se ejecuta en el equipo vivo ocupa RAM y puede sobrescribir evidencia. Se usan herramientas pequeñas desde un USB externo, se escribe la salida en un disco externo (nunca en el disco de la víctima) y se anota cada comando con su hora.

```
Order of volatility → recoger de lo más efímero a lo más duradero.
RFC 3227 → registros/caché, RAM y red, temporales, disco, logs remotos, topología, archivo.
Primer paso real → RAM e información de red, luego disco.
```

### Apagar el equipo destruye la evidencia más valiosa

> [!IMPORTANT]
> El reflejo de "apagar el equipo infectado" borra la RAM, y en la RAM están el malware que solo vive en memoria, las conexiones al C2, los comandos ejecutados y las claves de cifrado. Primero se aísla por red y se captura la memoria; después se decide si se apaga.

La captura en vivo (live response) es la práctica de recoger la evidencia volátil con el equipo encendido antes de cualquier apagado. Cobró importancia porque el malware moderno es cada vez más fileless: se inyecta en procesos legítimos y nunca escribe un ejecutable en disco. Una imagen de disco de ese equipo no muestra nada; un volcado de RAM lo muestra todo.

```
                      Apagar ya         Aislar + capturar RAM + apagar
Malware solo en RAM   perdido           capturado
Conexiones al C2      perdidas          capturadas
Claves de BitLocker   perdidas          capturadas (si el disco estaba abierto)
Disco                 intacto           intacto
```

La última columna gana en todas las filas y empata en la del disco: no hay ninguna fila donde apagar de golpe sea mejor.

Ejemplo: un portátil con disco cifrado con BitLocker. Si se apaga, la imagen del disco es un bloque cifrado ilegible sin la clave de recuperación. Si antes se volcó la RAM, Volatility puede extraer la clave maestra del volumen que estaba en memoria.

Límite: hay casos en que desconectar ya es correcto, por ejemplo un ransomware cifrando activamente un servidor de archivos compartido, donde cada minuto son miles de archivos perdidos. Ahí la disponibilidad pesa más que la evidencia, y la decisión queda documentada.

```
Live response → recoger evidencia volátil con el equipo encendido.
Malware fileless → vive solo en RAM; invisible en la imagen de disco.
Apagar → contiene y destruye la RAM; aislar + capturar → contiene y conserva.
```

### Chain of custody

La **cadena de custodia** es un registro documental que acompaña a cada pieza de evidencia y anota quién la tuvo, cuándo, dónde y para qué, desde que se recoge hasta que se presenta. Existe porque una prueba digital se puede alterar sin dejar rastro visible; si no se puede demostrar que nadie la manipuló, un juez o un auditor puede descartarla.

Qué registra el formulario, por cada pieza:

- Identificador único de la evidencia (por ejemplo `CASO-2026-014-E03`), descripción (marca, modelo, número de serie del disco) y lugar de recogida.
- Fecha y hora de recogida, con zona horaria, y quién la recogió.
- Hash de la evidencia (o de su imagen) en el momento de la recogida.
- Cada transferencia: entregado por, recibido por, fecha, hora, motivo ("traslado al laboratorio", "análisis"), y firma de ambos.
- Dónde se almacena: armario o caja fuerte con acceso controlado, bolsa antiestática sellada con precinto numerado.

Analogía: la cadena de custodia de una muestra de sangre en un control antidoping. Si en algún tramo la muestra estuvo en una nevera sin firmar, el deportista puede alegar que se la cambiaron, y la prueba cae.

Ejemplo de registro:

```
Evidencia: CASO-2026-014-E03  Disco SSD 512 GB, S/N S4EVNX0R123456
Recogida:  2026-03-04 10:42 UTC-6, oficina 3B, por A. Rojas
SHA-256:   9f2c...e71a
Traspasos:
  2026-03-04 11:15  A. Rojas -> L. Mora     "traslado a laboratorio"   firmas
  2026-03-05 08:30  L. Mora  -> armario E2  "almacenamiento"           firmas
  2026-03-06 09:00  armario E2 -> J. Vega   "creación de copia de trabajo" firmas
```

El análisis nunca se hace sobre el original ni sobre la imagen maestra: se hace sobre una copia de trabajo cuyo hash coincide con el de la maestra. Si algo sale mal, se vuelve a copiar desde la maestra.

```
Chain of custody → registro de quién tuvo la evidencia, cuándo y para qué.
Precinto numerado → demuestra que el contenedor no se abrió.
Imagen maestra → copia que no se toca; copia de trabajo → donde se analiza.
```

### Forensic imaging

Una **imagen forense** es una copia bit a bit de un medio de almacenamiento (disco, USB, tarjeta) que incluye todo: archivos, espacio no asignado, archivos borrados, slack space y metadatos del sistema de archivos. Existe porque copiar archivos con el explorador solo copia lo visible: deja fuera lo borrado, cambia fechas de acceso y no prueba nada sobre el original.

Conceptos que vienen con ella:

- Espacio no asignado (unallocated): sectores que el sistema de archivos marca como libres. Al borrar un archivo, normalmente solo se borra la entrada en el índice; los datos siguen ahí hasta que algo los sobrescribe.
- Slack space: el resto sin usar del último bloque de un archivo. Si un bloque es de 4096 bytes y el archivo ocupa 5000, el segundo bloque tiene 3192 bytes de slack que pueden contener restos de un archivo anterior.
- Write blocker: se conecta entre el disco de evidencia y el equipo del analista; deja pasar lecturas y bloquea escrituras. Sin él, el simple hecho de montar el disco en Windows puede escribir metadatos.
- Formatos de imagen: raw o dd (`.img`, `.dd`, `.001`): copia pura, sin compresión ni metadatos, la entiende cualquier herramienta. E01 (Expert Witness Format, de EnCase): comprimida, guarda metadatos del caso (examinador, número de caso, notas), incluye sumas de verificación por bloque y el hash total, y se puede partir en trozos (`.E01`, `.E02`...). AFF4: formato abierto alternativo.
- Imagen física (todo el disco, incluida la tabla de particiones) frente a imagen lógica (solo una partición o solo ciertos archivos, cuando no hay tiempo o el disco es enorme).

Analogía: fotocopiar un cuaderno página por página, incluidas las páginas arrancadas cuyo relieve quedó marcado en la siguiente, frente a pasar a limpio solo lo que se lee.

Ejemplo: un disco de 500 GB con 120 GB ocupados. Una copia de archivos copia 120 GB. Una imagen raw ocupa 500 GB. Una E01 comprimida puede ocupar unos 130 GB, porque el espacio vacío lleno de ceros se comprime casi entero, y aun así conserva cada sector.

```
Imagen forense → copia bit a bit, incluye lo borrado y lo no asignado.
Unallocated → espacio marcado como libre que aún guarda datos viejos.
Slack space → resto sin usar del último bloque de un archivo.
Raw/dd → copia pura, universal, sin compresión ni metadatos.
E01 → comprimida, con metadatos del caso, CRC por bloque y hash.
Write blocker → permite leer, bloquea escribir.
```

### Evidence hashing

El **hashing de evidencia** es la práctica de calcular un hash criptográfico del original y de cada copia para demostrar que son idénticos bit a bit y que nadie los modificó después. Existe porque una imagen de 500 GB no se puede comparar a ojo; un hash de 64 caracteres sí.

Cómo funciona:

1. Se calcula el hash del disco original (a través del write blocker) antes o durante la adquisición.
2. Se calcula el hash de la imagen resultante. Si coinciden, la imagen es fiel.
3. El hash se anota en la cadena de custodia.
4. Antes de cada análisis y antes de presentar resultados, se recalcula el hash de la copia de trabajo. Si sigue igual, nadie la alteró.

Algoritmos: MD5 (128 bits) y SHA-1 (160 bits) están rotos frente a colisiones (se pueden fabricar dos archivos distintos con el mismo hash), pero siguen usándose en forense por compatibilidad. Lo recomendable hoy es SHA-256 (256 bits), o calcular dos algoritmos a la vez (MD5 y SHA-256): fabricar un archivo que choque en ambos a la vez no es factible.

Analogía: el precinto numerado de un contenedor en un puerto. Si al llegar el número coincide con el anotado en origen, nadie lo abrió por el camino.

Ejemplo:

```
$ sudo sha256sum /dev/sdb
4a1d7c...b93e  /dev/sdb
$ sha256sum evidencia.img
4a1d7c...b93e  evidencia.img        # idénticos: la imagen es fiel
```

Contraejemplo: si el analista montó el disco sin write blocker y Windows escribió un solo byte en el registro del journal, el hash del original cambia, ya no coincide con el anotado al recogerlo, y la defensa puede alegar manipulación.

El hash prueba integridad, no autenticidad ni autoría: demuestra que la copia es igual al original, no quién escribió los archivos que contiene.

```
Hash de evidencia → prueba que copia y original son idénticos y no cambiaron.
MD5 / SHA-1 → débiles ante colisiones; aún aceptados por compatibilidad.
SHA-256 → recomendado; o MD5 + SHA-256 a la vez.
Recalcular → antes de cada análisis y antes de presentar resultados.
```

## Tools for Incident Response and Discovery

Herramientas que el equipo de respuesta usa para descubrir qué pasó, recoger evidencia y analizarla. Las de red se explican a fondo en [`07-herramientas-de-red`](../07-herramientas-de-red/); aquí solo va su uso concreto durante un incidente. Las de forense y de logs se explican completas.

```
Red (ver 07)         Recolección            Análisis              Logs
dig, nslookup        dd                     autopsy               cat, head
nmap, ping           FTK Imager             winhex                tail, grep
arp, tracert         memdump (LiME,         Volatility
ipconfig, curl         AVML, WinPmem)         (sobre el volcado)
wireshark, hping
```

### dig

`dig` es una herramienta de consulta DNS que pregunta registros a un servidor y muestra la respuesta completa. En IR sirve para resolver los dominios sospechosos que aparecen en logs (`dig +short update-check.example`), ver a qué IP apuntan hoy, revisar sus registros TXT y MX, y comprobar que el sinkhole o bloqueo DNS de la contención funciona (el dominio malo debe resolver a la IP del sinkhole). Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### nmap

`nmap` es un escáner de red que descubre hosts y puertos abiertos. En IR sirve para encontrar puertos en escucha que no deberían existir en un equipo comprometido (una puerta trasera en el puerto 4444, un servicio RDP abierto), para medir el alcance buscando el mismo servicio raro en toda la subred y para verificar que la contención realmente cerró el acceso. Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### ping

`ping` es una herramienta que envía paquetes ICMP Echo para comprobar si un host responde. En IR sirve para verificar que un equipo aislado ya no es alcanzable desde la red y que un sistema restaurado volvió a estar en línea; que no responda no prueba que esté apagado, porque muchos firewalls descartan ICMP. Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### arp

`arp` es una herramienta que muestra y modifica la caché ARP, la tabla que asocia IP con direcciones MAC en la red local. En IR, `arp -a` es evidencia volátil que se guarda temprano y delata ARP spoofing: si la IP del gateway aparece con la MAC de un equipo de usuario, o dos IP distintas comparten la misma MAC, alguien se está interponiendo en el tráfico. Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### cat

`cat` es un comando de Unix que lee uno o varios archivos y los escribe seguidos en la salida estándar (su nombre viene de concatenate). Existe para volcar un archivo entero o unir varios en un solo flujo que luego se filtra con otras herramientas.

En triaje de logs su uso real es juntar logs rotados para buscar en todos a la vez. Los logs de Linux rotan: `auth.log` es el actual, `auth.log.1` el de la semana pasada y `auth.log.2.gz` el anterior, comprimido. `cat` une los de texto; su hermano `zcat` descomprime los `.gz` al vuelo.

Analogía: grapar varios fajos de facturas en uno solo antes de buscar una en concreto.

Ejemplo:

```
$ cat /var/log/auth.log.1 /var/log/auth.log | wc -l
48210
$ zcat /var/log/auth.log.*.gz | cat - /var/log/auth.log.1 /var/log/auth.log | grep -c "Failed password"
3117
```

En Windows, el equivalente es `Get-Content` en PowerShell (alias `type` y `cat`), aunque los logs de eventos de Windows (`.evtx`) son binarios y se leen con `Get-WinEvent` o el Visor de eventos.

`cat` de un log de 5 GB en la terminal inunda la pantalla y no enseña nada. Se usa como primer eslabón de una tubería (`cat ... | grep ...`), no para leer a ojo.

```
cat → vuelca y concatena archivos.
zcat → igual, pero descomprime .gz al vuelo.
Get-Content → equivalente en PowerShell.
```

### dd

`dd` es una utilidad de Unix que copia datos bloque a bloque desde una entrada a una salida, sin interpretar sistemas de archivos, y por eso sirve para crear imágenes forenses raw de discos completos. Existe en cualquier Linux, no necesita instalar nada y copia todo lo que el disco tenga, incluidos sectores no asignados.

Cómo funciona: lee bloques de tamaño `bs` desde `if` (input file) y los escribe en `of` (output file). En Linux un disco entero es un archivo especial (`/dev/sdb`), así que `dd` lo copia sector por sector.

Opciones que importan en forense:

- `if=/dev/sdb`: el disco de evidencia, conectado a través de un write blocker.
- `of=/mnt/externo/caso014.img`: la imagen, siempre en otro disco.
- `bs=4M`: tamaño de bloque; bloques grandes aceleran la copia (con el valor por defecto de 512 bytes es mucho más lento).
- `conv=noerror,sync`: si un sector está dañado, no se detiene (`noerror`) y rellena ese bloque con ceros (`sync`) para que los desplazamientos de todo lo que viene después sigan siendo correctos.
- `status=progress`: muestra el avance.

Analogía: un escáner de documentos que pasa cada hoja tal cual, sin leerla ni entenderla, incluidas las hojas en blanco.

Ejemplo completo con verificación:

```
$ sudo blockdev --setro /dev/sdb                    # bloqueo de escritura por software
$ sudo sha256sum /dev/sdb > caso014.original.sha256
$ sudo dd if=/dev/sdb of=/mnt/externo/caso014.img bs=4M conv=noerror,sync status=progress
128035676160 bytes (128 GB, 119 GiB) copied, 1062 s, 121 MB/s
30526+1 records in
30527+0 records out
$ sha256sum /mnt/externo/caso014.img
4a1d7c...b93e  /mnt/externo/caso014.img            # igual al del original
```

Variantes pensadas para forense: `dcfldd` y `dc3dd` son versiones de `dd` que calculan el hash mientras copian, registran errores y pueden partir la imagen en trozos. Si el disco tiene muchos sectores dañados, `ddrescue` hace varias pasadas y deja los sectores difíciles para el final.

> [!WARNING]
> `dd` no pregunta nada. Invertir `if` y `of` (escribir la imagen o ceros sobre el disco de evidencia) destruye el original sin aviso. Por eso se usa un write blocker físico y se revisa el comando dos veces antes de pulsar Enter.

Con `conv=noerror,sync`, si hubo sectores ilegibles, el hash de la imagen no coincidirá con el del original (los ceros de relleno no son los datos reales); se documenta el número de errores, y es esperable.

```
dd → copia bloque a bloque; imagen raw de un disco entero.
if / of → entrada / salida; invertirlos destruye la evidencia.
conv=noerror,sync → sigue ante errores y rellena con ceros para conservar offsets.
dcfldd / dc3dd → dd forense: hash durante la copia y log.
ddrescue → para discos dañados, varias pasadas.
```

### tail

`tail` es un comando de Unix que muestra las últimas líneas de un archivo, y con `-f` sigue mostrando las líneas nuevas a medida que se escriben. Existe porque en un log lo más reciente está al final, y durante un incidente lo que importa es lo que está pasando ahora.

Usos en IR:

- `tail -n 200 /var/log/nginx/access.log`: ver las últimas 200 peticiones al servidor web comprometido.
- `tail -f /var/log/auth.log`: observar en vivo si el atacante sigue intentando entrar mientras se aplica la contención.
- `tail -F`: como `-f`, pero sobrevive a la rotación del log (vuelve a abrir el archivo cuando se renombra).

Analogía: mirar solo las últimas páginas del libro de visitas de un hotel para ver quién llegó anoche.

Ejemplo:

```
$ tail -f /var/log/auth.log | grep --line-buffered "Accepted"
Mar  4 10:51:02 web01 sshd[2211]: Accepted publickey for deploy from 203.0.113.50 port 51122
```

Si tras rotar las claves SSH sigue apareciendo `Accepted` desde `203.0.113.50`, la erradicación no está completa. En PowerShell el equivalente es `Get-Content log.txt -Tail 50 -Wait`; en sistemas con systemd, `journalctl -f -u ssh`.

```
tail -n N → últimas N líneas.
tail -f → sigue el archivo en vivo.
tail -F → sigue incluso tras la rotación del log.
```

### hping

`hping` (en su versión actual, `hping3`) es un generador de paquetes TCP/IP por línea de comandos que permite fabricar paquetes con las cabeceras y flags que uno elija y ver cómo responde el destino. Existe porque `ping` solo envía ICMP Echo y muchos firewalls lo bloquean; `hping3` puede enviar un SYN al puerto 443, un paquete UDP o un ICMP con tamaño concreto, y así probar reglas de firewall con precisión.

Cómo se lee la respuesta a un SYN (el primer paquete del handshake TCP):

```
hping3 -S -p 443  --->  [ destino ]
                  <---  SYN/ACK (flags=SA)  puerto abierto
                  <---  RST/ACK (flags=RA)  puerto cerrado
                  (sin respuesta)           filtrado por un firewall
```

Opciones frecuentes: `-S` (flag SYN), `-A` (ACK), `-F` (FIN), `-p` (puerto destino), `-c` (número de paquetes), `-1` (modo ICMP), `-2` (modo UDP), `--traceroute` (traceroute con el tipo de paquete elegido, útil cuando ICMP está bloqueado).

Usos en IR:

- Verificar la contención: tras bloquear el puerto 3389 en el firewall, comprobar desde fuera que ya no responde.
- Distinguir "cerrado" de "filtrado" cuando `nmap` da un resultado ambiguo.
- En laboratorio, reproducir el patrón de tráfico de un ataque (por ejemplo, un escaneo con FIN) para comprobar que una regla nueva del IDS lo detecta.

Analogía: tocar a una puerta con distintos golpes (suave, fuerte, dos toques) para ver cuál hace que alguien conteste y cuál nadie escucha.

Ejemplo:

```
$ sudo hping3 -S -p 443 -c 2 198.51.100.10
HPING 198.51.100.10 (eth0 198.51.100.10): S set, 40 headers + 0 data bytes
len=46 ip=198.51.100.10 ttl=56 DF id=0 sport=443 flags=SA seq=0 win=64240 rtt=12.1 ms
len=46 ip=198.51.100.10 ttl=56 DF id=0 sport=443 flags=SA seq=1 win=64240 rtt=11.8 ms
--- 198.51.100.10 hping statistic ---
2 packets transmitted, 2 packets received, 0% packet loss
```

`flags=SA` en los dos paquetes: el 443 está abierto y alcanzable.

> [!WARNING]
> `hping3` también sirve para inundar un destino (`--flood`) y falsificar la IP de origen; es una herramienta de DoS en las manos equivocadas. Solo se usa contra sistemas propios o con autorización por escrito.

```
hping3 → fabrica paquetes TCP/UDP/ICMP con flags a medida.
SA → abierto; RA → cerrado; sin respuesta → filtrado.
ping → solo ICMP Echo; hping3 → cualquier paquete, cualquier puerto.
```

### head

`head` es un comando de Unix que muestra las primeras líneas (o bytes) de un archivo. Existe para echar un vistazo rápido a algo grande sin abrirlo entero.

Usos en IR:

- Ver el formato de un log antes de escribir filtros: `head -n 5 access.log` enseña qué campo es la IP, cuál la fecha y cuál la URL.
- Ver cuándo empieza un log (la primera línea dice desde qué fecha hay datos, clave para saber si cubre el momento del ataque).
- Mirar la cabecera de un archivo sospechoso: `head -c 16 archivo | xxd` muestra los primeros bytes (magic bytes), que revelan el tipo real aunque la extensión mienta. `4D 5A` (`MZ`) es un ejecutable de Windows aunque se llame `factura.pdf`.
- Cortar resultados: `... | sort | uniq -c | sort -rn | head -n 10` deja los 10 primeros.

Analogía: leer la portada y el índice de un libro antes de decidir qué capítulo buscar.

Ejemplo:

```
$ head -c 16 factura.pdf | xxd
00000000: 4d5a 9000 0300 0000 0400 0000 ffff 0000  MZ..............
```

Un PDF real empezaría por `25 50 44 46` (`%PDF`); este es un ejecutable disfrazado.

```
head -n N → primeras N líneas.
head -c N → primeros N bytes; con xxd revela los magic bytes.
head → inicio del archivo; tail → final del archivo.
```

### grep

`grep` es un comando de Unix que busca líneas que coinciden con un patrón (texto o expresión regular) dentro de archivos o de una tubería. Es la herramienta central del triaje de logs: convierte millones de líneas en las veinte que importan.

Opciones que más se usan en IR:

- `-i`: ignora mayúsculas y minúsculas.
- `-v`: invierte, muestra lo que no coincide (para quitar ruido conocido).
- `-c`: cuenta coincidencias en vez de mostrarlas.
- `-E`: expresiones regulares extendidas (`-E "union|select|sleep\("`).
- `-r`: busca recursivamente en un directorio.
- `-A 3 -B 3`: muestra 3 líneas después y antes de cada coincidencia, para ver el contexto.
- `-o`: muestra solo la parte que coincide (útil para extraer IP).
- `-F`: busca texto literal, sin interpretar regex (más rápido para listas de IoC con `-f lista.txt`).

Analogía: el buscador de "Ctrl+F" de un documento, pero que funciona sobre cientos de archivos a la vez y entiende patrones.

Ejemplo de triaje: un servidor SSH con sospecha de fuerza bruta.

```
$ grep "Failed password" /var/log/auth.log | grep -oE "from [0-9.]+" | sort | uniq -c | sort -rn | head -n 5
   2840 from 203.0.113.50
    212 from 198.51.100.77
      9 from 10.0.0.15
      3 from 10.0.0.22
      1 from 10.0.0.31
$ grep "Accepted" /var/log/auth.log | grep "203.0.113.50"
Mar  4 03:12:44 web01 sshd[1987]: Accepted password for backup from 203.0.113.50 port 40112 ssh2
```

Lectura: 2840 fallos desde una IP y luego un acceso aceptado con la cuenta `backup` a las 03:12. La fuerza bruta funcionó; esa hora abre la línea de tiempo.

Otro caso típico: buscar intentos de inyección SQL en un log web:

```
$ grep -iE "union.+select|sleep\(|or%201=1" /var/log/nginx/access.log | head -n 3
```

En Windows el equivalente es `Select-String` en PowerShell (`Select-String -Path *.log -Pattern "Failed"`), y para los `.evtx`, `Get-WinEvent` filtrando por ID de evento (4625 = inicio de sesión fallido, 4624 = correcto).

> [!TIP]
> El patrón de triaje que más se repite es `grep ... | sort | uniq -c | sort -rn | head`: filtra, agrupa iguales, cuenta, ordena de mayor a menor y deja el top. Responde en segundos "qué IP, qué usuario o qué URL aparece más".

```
grep → filtra líneas por patrón.
-i / -v / -c / -E / -o → sin mayúsculas / invertir / contar / regex / solo la coincidencia.
-A / -B → contexto después / antes.
grep | sort | uniq -c | sort -rn | head → top N de lo que más aparece.
Select-String / Get-WinEvent → equivalentes en Windows.
```

### nslookup

`nslookup` es una herramienta de consulta DNS disponible en Windows y Unix. En IR cumple el mismo papel que `dig` en equipos Windows donde no hay otra cosa: resolver dominios sospechosos (`nslookup update-check.example`) y comprobar qué servidor DNS usa realmente el equipo; si responde un servidor que no es el corporativo, el atacante pudo cambiar la configuración DNS. Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### tracert

`tracert` (en Unix, `traceroute`) es una herramienta que muestra los saltos (routers) que atraviesa un paquete hasta su destino. En IR sirve para ver por dónde sale el tráfico hacia un C2 (qué proveedor, qué país aproximado) y para comprobar que un bloqueo o una ruta a un sinkhole está aplicado en el punto correcto de la red. Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### winhex

**WinHex** es un editor hexadecimal y de disco para Windows, de X-Ways Software Technology, que muestra y edita el contenido en bytes de archivos, discos, particiones y memoria RAM. Existe porque hay evidencia que ninguna herramienta de alto nivel enseña: un archivo borrado cuyos datos siguen en sectores libres, una cabecera manipulada, datos ocultos en el slack space.

Qué permite hacer:

- Abrir un disco físico o una imagen y navegar por sectores, viendo cada byte en hexadecimal a la izquierda y su texto ASCII a la derecha.
- Interpretar estructuras del sistema de archivos (tabla de particiones, entradas de la MFT de NTFS) con plantillas.
- File carving: recuperar archivos borrados buscando sus firmas en el espacio no asignado, sin depender del índice del sistema de archivos. Firmas típicas: JPEG empieza por `FF D8 FF`, PNG por `89 50 4E 47`, PDF por `25 50 44 46` (`%PDF`), ZIP y DOCX por `50 4B 03 04` (`PK`).
- Clonar discos, crear imágenes, calcular hashes y limpiar medios de forma segura.
- Editar la RAM de un proceso en vivo.

La versión forense completa de la misma casa es X-Ways Forensics, que añade gestión de casos, filtros y reportes; WinHex es la base.

Analogía: una lupa de joyero. Las herramientas normales muestran el anillo; la lupa muestra cada rayita del metal.

Ejemplo: en una imagen de una USB formateada, se busca la firma `FF D8 FF E0` en el espacio no asignado. Aparece en el desplazamiento `0x0012A000`; desde ahí hasta la firma de fin de JPEG (`FF D9`) hay 2,4 MB. Se exporta ese bloque y se obtiene una foto que el sistema de archivos ya no listaba.

```
Offset     00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
0012A000   FF D8 FF E0 00 10 4A 46 49 46 00 01 01 00 00 01   ......JFIF......
```

```
WinHex → editor hexadecimal y de disco (Windows, X-Ways).
File carving → recuperar archivos por su firma, sin el índice del sistema de archivos.
Magic bytes → firma inicial que identifica el tipo real de un archivo.
X-Ways Forensics → suite forense completa construida sobre WinHex.
```

### autopsy

**Autopsy** es una plataforma de análisis forense de discos, gratuita y de código abierto, que ofrece una interfaz gráfica sobre The Sleuth Kit (TSK), un conjunto de herramientas de línea de comandos para analizar sistemas de archivos. La desarrolla Sleuth Kit Labs (Brian Carrier) y la usan policías, SOC y equipos DFIR. Existe para que el analista no tenga que encadenar a mano decenas de comandos de TSK y pueda ver archivos, borrados, historial y línea de tiempo en una sola ventana.

Cómo funciona un caso:

1. Crear un caso (nombre, número, examinador).
2. Añadir una fuente de datos (data source): imagen raw o E01, un disco local, una carpeta de archivos o un disco virtual de VM.
3. Elegir los ingest modules, módulos que recorren la imagen y extraen información automáticamente:
   - Recent Activity: historial de navegadores, descargas, documentos recientes, dispositivos USB conectados (desde el registro de Windows).
   - Hash Lookup: compara cada archivo con bases de hashes conocidos. Los "conocidos buenos" (como la NSRL, la lista de archivos de software legítimo del NIST) se ocultan para reducir ruido; los "conocidos malos" se marcan.
   - File Type Identification y Extension Mismatch: detectan archivos cuya extensión no coincide con su contenido real.
   - Keyword Search: busca palabras o expresiones regulares (correos, tarjetas, IP) en todo el disco, incluido lo no asignado.
   - EXIF: extrae fecha, cámara y GPS de fotos.
   - Embedded File Extractor: abre ZIP, documentos de Office y otros contenedores.
4. Revisar los resultados en el árbol de la izquierda, la línea de tiempo (Timeline) y las vistas de archivos borrados, y etiquetar lo relevante.
5. Generar un reporte HTML o Excel con lo etiquetado.

Por debajo, TSK lee directamente las estructuras del sistema de archivos (NTFS, FAT, ext4, HFS+), por eso ve entradas borradas que el sistema operativo ya no muestra. Los comandos de TSK que Autopsy usa se pueden correr a mano:

```
$ mmls caso014.img                    # tabla de particiones
     Slot      Start        End          Length       Description
002:  000:000   0000002048   0001026047   0001024000   NTFS / exFAT (0x07)
$ fls -o 2048 -r -d caso014.img       # solo archivos borrados (-d), recursivo
r/r * 6421-128-1:   Users/ana/Downloads/factura.pdf.exe
$ icat -o 2048 caso014.img 6421 > recuperado.bin   # extrae el contenido por número de inodo/MFT
```

El asterisco en la salida de `fls` marca un archivo borrado; `6421` es su entrada en la MFT, y `icat` lo recupera si sus clusters no fueron sobrescritos.

Analogía: un laboratorio que recibe un cajón lleno de papeles y devuelve un informe ordenado: qué hay, qué se tiró a la basura, qué fotos tienen GPS y qué pasó cada día.

Ejemplo: en la imagen del portátil de contabilidad, Recent Activity muestra la descarga de `factura.pdf.exe` a las 02:09; Extension Mismatch marca un `.jpg` que en realidad es un ZIP; la línea de tiempo enseña que 3 minutos después se creó la tarea programada de persistencia. Esa secuencia es la causa raíz que pide la fase de erradicación.

```
Autopsy → interfaz gráfica forense de discos, gratis, sobre The Sleuth Kit.
The Sleuth Kit → herramientas CLI: mmls (particiones), fls (archivos), icat (extraer).
Ingest modules → extraen historial, hashes, palabras clave, EXIF, desajustes de extensión.
NSRL → base de hashes de software legítimo, para ocultar lo conocido.
```

### ipconfig

`ipconfig` es la herramienta de Windows que muestra la configuración de red de cada interfaz. En IR se guarda temprano `ipconfig /all` (IP, MAC, gateway y, sobre todo, servidores DNS: uno inesperado indica secuestro de DNS) e `ipconfig /displaydns`, la caché DNS, que es evidencia volátil de qué dominios contactó el equipo hace poco; `ipconfig /flushdns` la borra, así que no se ejecuta antes de guardarla. Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### curl

`curl` es una herramienta de línea de comandos que hace peticiones a URL (HTTP, HTTPS, FTP y otros). En IR sirve para ver las cabeceras de un sitio sospechoso sin abrirlo en un navegador (`curl -I`), descargar una muestra de malware a un entorno aislado para analizarla, comprobar si una web shell sigue respondiendo tras la limpieza y consultar APIs de inteligencia de amenazas (por ejemplo, reputación de un hash o una IP). Se ejecuta desde una máquina de análisis aislada, nunca desde un equipo de producción. Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### wireshark

Wireshark es un analizador de protocolos con interfaz gráfica que captura tráfico y decodifica cada paquete capa por capa. En IR se usa sobre las capturas (pcap) del incidente para reconstruir conexiones con "Follow TCP Stream", extraer archivos transferidos con "Export Objects > HTTP", ver patrones de beaconing (conexiones al C2 a intervalos regulares, por ejemplo cada 60 segundos) y filtrar por IoC (`ip.addr == 203.0.113.50`, `dns.qry.name contains "update-check"`). Detalle en [`07-herramientas-de-red`](../07-herramientas-de-red/).

### memdump

**memdump** es el nombre genérico (y el de varias herramientas concretas) de un volcador de memoria: un programa que copia el contenido completo de la RAM de un equipo vivo a un archivo para analizarlo después. La herramienta original con ese nombre es parte de The Coroner's Toolkit, de Dan Farmer y Wietse Venema, para sistemas Unix; hoy el mismo trabajo lo hacen herramientas mantenidas para cada sistema. Existe porque la RAM contiene lo que el disco no: procesos en ejecución, malware que solo vive en memoria, conexiones de red abiertas, comandos tecleados, contraseñas y claves de cifrado.

Herramientas actuales de adquisición:

- Linux: LiME (Linux Memory Extractor), un módulo del kernel que se carga con `insmod` y escribe la memoria a un archivo o por red. AVML (Acquire Volatile Memory for Linux), de Microsoft, un binario único que no necesita compilar un módulo para cada kernel.
- Windows: WinPmem (proyecto de Velocidex), DumpIt, y la opción "Capture Memory" de FTK Imager.
- Máquinas virtuales: suspender la VM o tomar un snapshot genera un archivo de memoria (`.vmem` en VMware) sin ejecutar nada dentro del invitado.

Cómo se adquiere bien:

- La salida va a un disco externo o a la red, nunca al disco del equipo investigado.
- Se ejecuta lo antes posible (orden de volatilidad) y se anota la hora exacta.
- Se calcula el hash del volcado al terminarlo.
- El volcado ocupa lo mismo que la RAM: un equipo con 32 GB genera un archivo de 32 GB.

Ejemplo en Linux:

```
$ sudo ./avml /mnt/externo/web01.lime
$ sha256sum /mnt/externo/web01.lime
c81e...0d4f  /mnt/externo/web01.lime

$ sudo insmod lime.ko "path=/mnt/externo/web01.lime format=lime"   # alternativa con LiME
```

Ejemplo en Windows (desde un USB):

```
E:\> winpmem_mini_x64.exe F:\caso014\portatil.raw
```

Analogía: sacar una foto instantánea del escritorio de alguien en pleno trabajo, con todos los papeles abiertos, antes de que los guarde en el cajón.

Análisis con Volatility.

Volatility es un framework de código abierto, escrito en Python, para analizar volcados de memoria. La versión actual, Volatility 3, reconoce el sistema operativo del volcado mediante tablas de símbolos y ofrece plugins con el prefijo del sistema (`windows.`, `linux.`, `mac.`). Recorre las estructuras internas del kernel dentro del volcado igual que lo haría el propio sistema operativo.

Plugins básicos en Windows:

- `windows.info`: versión del sistema y hora del volcado.
- `windows.pslist`: procesos, recorriendo la lista enlazada de procesos activos que mantiene el kernel.
- `windows.psscan`: procesos, buscando sus estructuras en toda la memoria por su firma, sin usar la lista.
- `windows.pstree`: procesos en árbol padre-hijo.
- `windows.cmdline`: línea de comandos con que se lanzó cada proceso.
- `windows.netscan`: conexiones y puertos en escucha.
- `windows.dlllist`: DLL cargadas por cada proceso.
- `windows.malfind`: zonas de memoria ejecutables y escribibles sin archivo detrás, típicas de código inyectado.
- En Linux, `linux.pslist` y `linux.bash` (historial de comandos de bash que seguía en memoria).

Por qué `pslist` y `psscan` dan resultados distintos: el kernel de Windows enlaza los procesos activos en una lista doblemente enlazada. Un rootkit puede desenganchar su proceso de esa lista (técnica llamada DKOM, Direct Kernel Object Manipulation) y el proceso sigue corriendo pero desaparece de `pslist`, del Administrador de tareas y de cualquier herramienta que use la lista. `psscan` no usa la lista: barre toda la memoria buscando la firma de la estructura de proceso, así que lo encuentra igual. Un proceso que aparece en `psscan` y no en `pslist` está oculto o ya terminó; las dos cosas merecen mirarse.

Ejemplo de análisis:

```
$ vol -f portatil.raw windows.pstree
PID    PPID   ImageFileName
4      0      System
...
3120   2200   WINWORD.EXE
* 4410 3120   powershell.exe
** 4522 4410  rundll32.exe

$ vol -f portatil.raw windows.cmdline --pid 4410
4410  powershell.exe  powershell -nop -w hidden -enc JABjAD0A...

$ vol -f portatil.raw windows.netscan | grep 4522
TCPv4  10.0.0.15  49822  203.0.113.50  443  ESTABLISHED  4522  rundll32.exe
```

Lectura: Word lanzó un PowerShell oculto con un comando codificado, que lanzó `rundll32.exe`, que tiene una conexión abierta al 203.0.113.50:443. Word nunca tiene motivo para abrir PowerShell: es el rastro de una macro maliciosa, y la IP es el C2. Nada de esto estaba en el disco como archivo.

> [!TIP]
> Primera pasada sobre cualquier volcado de Windows: `pstree` (relaciones raras padre-hijo, como Word o Excel lanzando una shell), `cmdline` (comandos codificados u ocultos), `netscan` (conexiones a IP externas) y `malfind` (código inyectado). Cuatro plugins cubren la mayoría de los casos.

```
memdump → volcado completo de la RAM a un archivo.
LiME / AVML → adquisición en Linux; WinPmem / DumpIt / FTK Imager → en Windows.
Volatility 3 → framework de análisis de volcados; plugins windows.*, linux.*.
pslist → recorre la lista de procesos; psscan → busca firmas en toda la RAM.
DKOM → rootkit se desengancha de la lista; aparece en psscan, no en pslist.
malfind → memoria ejecutable sin archivo detrás: código inyectado.
```

### FTK Imager

**FTK Imager** es una herramienta gratuita de adquisición y vista previa forense para Windows, de Exterro (antes AccessData), que crea imágenes de discos, captura la memoria RAM y permite examinar evidencia sin modificarla. Existe para que el primer respondedor pueda adquirir evidencia de forma forense con una interfaz sencilla y una herramienta reconocida en tribunales, sin tener que dominar `dd`.

Qué hace:

- Create Disk Image: imagen de un disco físico, una partición lógica, una carpeta o una imagen existente, en formatos raw (dd), E01, SMART o AFF. Pide los datos del caso (número, evidencia, examinador, notas), que quedan grabados en la E01.
- Verificación automática: calcula MD5 y SHA-1 del origen y de la imagen y genera un archivo `.txt` de registro con ambos hashes, la hora de inicio y fin y los sectores con error.
- Capture Memory: vuelca la RAM a un archivo (`memdump.mem` por defecto), con opción de incluir `pagefile.sys` (el archivo de paginación, que guarda trozos de RAM que Windows movió a disco) y de generar un archivo AD1.
- Preview: añade un disco o imagen como evidence item y explora su sistema de archivos, incluidos archivos borrados, en modo solo lectura, sin crear la imagen completa.
- Image Mounting: monta una imagen como unidad de solo lectura para revisarla con otras herramientas.
- Export Files y Obtain Protected Files: exporta archivos sueltos, y extrae archivos que Windows mantiene bloqueados en uso, como el registro (`SAM`, `SYSTEM`) en un sistema vivo.

Se suele llevar en un USB (versión portátil, FTK Imager Lite en versiones antiguas) para ejecutarlo en el equipo vivo sin instalarlo, escribiendo la salida en un disco externo.

Analogía: una cámara de fotos de policía con sello de tiempo y número de serie: hace la foto, la firma y anota quién la tomó, todo en un solo paso.

Ejemplo de registro generado tras una imagen E01:

```
Case Information:
 Case Number: 2026-014
 Evidence Number: E03
 Examiner: L. Mora
Information for F:\caso014\E03:
 Physical Evaluated Sector Count: 1000215216
 [Computed Hashes]
  MD5 checksum:    7e1b...a20c
  SHA1 checksum:   3d44...91fe
Image Information:
 Acquisition started:  Wed Mar 04 11:30:12 2026
 Acquisition finished: Wed Mar 04 13:02:47 2026
 Segment list:
  F:\caso014\E03.E01
  F:\caso014\E03.E02
Image Verification Results:
 Verification started: Wed Mar 04 13:02:48 2026
 MD5 checksum:    7e1b...a20c : verified
 SHA1 checksum:   3d44...91fe : verified
```

Ejecutar FTK Imager en el equipo vivo para capturar la RAM también deja huella: carga el programa en memoria y crea entradas de ejecución. Es aceptable y esperado, pero se documenta (qué se ejecutó, desde dónde y a qué hora) para que no se confunda con actividad del atacante.

```
FTK Imager → adquisición y vista previa forense gratuita (Exterro, Windows).
Create Disk Image → raw, E01, SMART, AFF; hashes MD5 y SHA-1 verificados.
Capture Memory → volcado de RAM (+ pagefile.sys opcional).
Obtain Protected Files → extrae SAM, SYSTEM y otros archivos bloqueados.
dd → misma imagen raw, en línea de comandos, sin metadatos de caso.
```

## Understand Audience

Entender la audiencia es la práctica de adaptar qué se comunica durante un incidente, cuándo y con qué nivel de detalle, según quién lo recibe. Existe porque el mismo hecho ("el atacante tuvo acceso al servidor de RR. HH. durante 12 días") significa cosas distintas para cada grupo: para el técnico es un alcance, para legal es una posible obligación de notificar, para dirección es un riesgo de negocio, para RR. HH. es un posible problema con un empleado. Comunicar mal causa daños reales: un correo técnico reenviado puede filtrarse a la prensa, un aviso tardío a legal puede incumplir un plazo legal.

Reglas que valen para todas las audiencias:

- Necesidad de saber (need to know): cada grupo recibe lo que necesita para actuar, no más. Menos gente informada significa menos filtraciones y menos riesgo de avisar al atacante, que puede ser alguien de dentro.
- Hechos, no especulación: "se confirmó acceso a 3 servidores" en vez de "parece que se llevaron todo".
- Canal fuera de banda: si el correo corporativo puede estar comprometido, los mensajes del incidente van por otro canal.
- Un solo portavoz hacia fuera: nadie del equipo técnico habla con prensa ni clientes por su cuenta.
- Registro de qué se comunicó, a quién y cuándo, porque los plazos legales se miden desde momentos concretos.

```
                     +----------------------+
                     |  Líder del incidente |
                     +----------+-----------+
          +-----------+---------+---------+------------+
          v           v         v         v            v
    Management      Legal   Compliance    HR      Stakeholders
   (decisiones,  (privilegio, (plazos de (empleados (clientes,
    recursos,     obligaciones regulador, implicados, socios,
    riesgo)       legales)     auditoría) personas)  proveedores)
```

Analogía: un hospital con un paciente grave. El cirujano habla en términos clínicos con el equipo, el director recibe "estable, sin riesgo para el hospital", el abogado sabe si hay negligencia posible, y a la familia se le explica en palabras claras qué pasa y qué viene.

### Stakeholders

Los **stakeholders** (partes interesadas) son todas las personas u organizaciones afectadas por el incidente o con interés en su resultado, dentro y fuera de la empresa: dueños de los sistemas afectados, clientes, socios, proveedores, aseguradora, accionistas y, si aplica, prensa y fuerzas del orden. Se identifican en la fase de preparación, con una lista de contactos y de quién avisa a cada uno.

Qué se les comunica:

- Dueños de sistemas internos: qué sistemas se van a aislar o apagar y por cuánto tiempo, para que planifiquen la operación.
- Clientes y socios: si sus datos o servicios se ven afectados, qué deben hacer (cambiar contraseñas, vigilar cargos) y cuándo habrá novedades. El mensaje lo aprueban legal y dirección.
- Aseguradora de ciberriesgo: aviso temprano, porque muchas pólizas exigen notificar en plazos cortos y usar proveedores de respuesta aprobados.
- Proveedores implicados: el proveedor de nube o de software si el incidente pasa por su producto.
- Fuerzas del orden: denuncia cuando hay delito, decidida con legal.

Analogía: los vecinos de un edificio con una fuga de gas. Cada vecino necesita saber si debe salir, cuándo puede volver y qué no debe hacer (encender interruptores), no el detalle técnico de la válvula rota.

Ejemplo: una tienda online sufre un robo de datos de 10 000 clientes. A los clientes: correo con qué datos se vieron afectados (nombre, correo, dirección; no tarjetas), qué hacer (cuidado con correos de phishing que usen esos datos) y un contacto. Al procesador de pagos: confirmación de que los datos de tarjeta no estaban en el sistema afectado. A la aseguradora: aviso el primer día.

```
Stakeholders → todos los afectados o interesados, internos y externos.
Clientes → qué pasó con sus datos y qué deben hacer; mensaje aprobado por legal.
Aseguradora → aviso temprano; plazos de la póliza.
```

### HR

**HR** (Human Resources, Recursos Humanos) es el área que gestiona la relación laboral con los empleados y entra en un incidente cuando hay personas de la organización implicadas como causa, como víctimas o como afectadas. Existe en el proceso porque cualquier acción sobre un empleado (investigarlo, suspenderle el acceso, despedirlo) tiene reglas laborales y de privacidad que el equipo técnico no conoce.

Cuándo y qué se le comunica:

- Amenaza interna (insider): un empleado que exfiltra datos o sabotea. RR. HH. coordina con legal cómo y cuándo actuar, y la entrevista se hace con RR. HH. presente. El equipo técnico recoge evidencia en silencio hasta que se decide.
- Error humano: un empleado que cayó en un phishing. RR. HH. ayuda a que la respuesta sea formativa y no punitiva, para que la gente siga reportando.
- Datos de empleados afectados: si se filtraron nóminas o expedientes, RR. HH. organiza la comunicación interna a la plantilla.
- Bajas y altas de acceso: cuando un empleado se va (o es despedido por el incidente), RR. HH. avisa para revocar todos sus accesos a la vez.

Analogía: el árbitro de una disputa entre compañeros de trabajo: no investiga el hecho técnico, pero asegura que el trato a la persona sea justo y legal.

Ejemplo: el DLP detecta que un empleado que presentó su renuncia hace 3 días subió 4 GB de diseños a su nube personal. El equipo técnico preserva logs y una imagen del portátil; RR. HH. y legal deciden revocar el acceso y entrevistarlo al día siguiente con un testigo, en vez de que un analista lo confronte en el pasillo.

El equipo de seguridad nunca confronta a un empleado sospechoso por su cuenta. Además del riesgo laboral y legal, el aviso le da tiempo a borrar evidencia.

```
HR → coordina todo lo que afecta a personas de la organización.
Insider → evidencia en silencio; acción decidida con RR. HH. y legal.
Error humano → respuesta formativa, no punitiva.
```

### Legal

**Legal** (el área jurídica interna o el bufete externo) es la audiencia que determina las obligaciones legales del incidente, protege a la organización en posibles litigios y aprueba toda comunicación externa. Existe porque casi cada decisión del incidente tiene consecuencias legales: notificar o no, qué decir, qué evidencia guardar y cómo.

Qué hace y qué necesita saber:

- Obligaciones de notificación: si hay datos personales afectados, muchas leyes obligan a notificar a una autoridad y a los afectados en plazos concretos. Ejemplos: el RGPD europeo exige notificar a la autoridad de control en un máximo de 72 horas desde que se tiene constancia de la brecha (art. 33); la SEC de EE. UU. exige a las empresas cotizadas reportar un incidente material en un formulario 8-K dentro de 4 días hábiles desde que se determina que es material. Legal decide qué aplica según el país y el sector. Detalle en [`17-estandares-y-cumplimiento`](../17-estandares-y-cumplimiento/).
- Privilegio abogado-cliente: en algunos sistemas legales (sobre todo EE. UU.), si la investigación forense la encarga y dirige el abogado, los informes pueden quedar protegidos frente a demandas. Por eso a veces el proveedor forense se contrata a través del bufete.
- Legal hold: orden de conservar toda la evidencia y los documentos relacionados; suspende las políticas de borrado automático de logs y correos.
- Relación con fuerzas del orden y con los propios atacantes (por ejemplo, en una extorsión de ransomware, si pagar es legal o está sancionado).
- Revisión de todo texto externo: comunicados, correos a clientes, respuestas a la prensa.

Analogía: el abogado que acompaña a alguien después de un accidente de tráfico: le dice qué declarar, qué papeles guardar y en qué plazo presentar el parte al seguro.

Ejemplo: un analista quiere escribir en el ticket "nuestra culpa, el servidor llevaba 2 años sin parchear". Legal pide que el ticket solo registre hechos verificables ("el servidor tenía la versión X, afectada por la vulnerabilidad Y"), porque ese texto puede aparecer en una demanda.

Ante la duda de si algo hay que contárselo a legal, la respuesta es sí y pronto: los plazos legales cuentan desde que la organización "tiene conocimiento", no desde que termina la investigación.

```
Legal → obligaciones de notificación, privilegio, legal hold, aprobación de mensajes.
RGPD art. 33 → 72 h para notificar a la autoridad de control.
SEC 8-K → 4 días hábiles tras determinar que el incidente es material.
Legal hold → conservar evidencia; suspende el borrado automático.
```

### Compliance

**Compliance** (cumplimiento normativo) es la función que asegura que la organización cumple las normas, regulaciones y contratos que le aplican, y en un incidente traduce lo ocurrido a las obligaciones concretas de cada marco. Se diferencia de legal en que legal interpreta la ley y el riesgo jurídico, mientras compliance gestiona el cumplimiento operativo de marcos como PCI DSS, HIPAA, ISO/IEC 27001 o SOC 2, y la relación con auditores y reguladores sectoriales.

Qué necesita del equipo de respuesta:

- Qué datos regulados se vieron afectados: datos de tarjeta (PCI DSS), salud (HIPAA en EE. UU.), datos personales en general.
- Evidencia de que los controles exigidos existían y funcionaron (o no): logs, registros de acceso, la propia documentación del incidente.
- Plazos de notificación sectoriales y contractuales: la marca de tarjetas o el banco adquirente en un incidente PCI, el regulador financiero, clientes con contratos que exigen aviso en X horas.
- Hallazgos para el próximo ciclo de auditoría: el informe de lecciones aprendidas se convierte en acciones correctivas que el auditor revisará.

Analogía: el inspector de seguridad de una fábrica tras un accidente: no investiga la máquina, comprueba qué normas se aplican, si se cumplían y a qué organismo hay que informar.

Ejemplo: en un comercio que procesa tarjetas, se confirma malware en un terminal de punto de venta. Compliance activa el procedimiento PCI: avisar al banco adquirente y a las marcas de tarjeta según sus plazos y contratar un investigador forense acreditado por el PCI SSC (PFI, PCI Forensic Investigator).

```
Compliance → cumplimiento de marcos y contratos (PCI DSS, HIPAA, ISO 27001, SOC 2).
Legal → interpreta la ley y el riesgo jurídico; compliance → ejecuta las obligaciones de cada marco.
PFI → investigador forense acreditado por PCI para incidentes con tarjetas.
```

### Management

**Management** (la dirección: comité ejecutivo, CISO, CIO, CEO y, en incidentes graves, el consejo) es la audiencia que toma las decisiones de negocio del incidente y aporta los recursos. Existe en el proceso porque hay decisiones que no son técnicas: apagar la tienda online un viernes, pagar o no un rescate, contratar una firma forense externa, hacer pública la brecha.

Qué se le comunica y cómo:

- Impacto en términos de negocio, no técnicos: "la facturación está detenida, perdemos unos 50 000 dólares por día" en vez de "el dominio está comprometido vía Kerberoasting".
- Opciones con su coste y riesgo: "A) aislar todo ya: 2 días sin servicio, riesgo bajo; B) aislar por fases: 6 horas sin servicio, riesgo medio de que el atacante reaccione".
- Estado en informes periódicos y breves (por ejemplo, cada 4 horas en un incidente grave), siempre con el mismo formato: qué sabemos, qué hemos hecho, qué necesitamos, próxima actualización.
- Al cierre, el resumen ejecutivo del informe de lecciones aprendidas: causa, impacto, coste y qué inversión evitaría que se repita.

Analogía: el capitán de un barco durante una tormenta. No necesita saber qué tornillo se rompió en la sala de máquinas; necesita saber cuánta velocidad tiene, cuánto aguanta y qué opciones hay.

Ejemplo de actualización a dirección:

```
Incidente 2026-014 - Actualización 3 - 04/03 18:00
Estado: contenido. Severidad: alta.
Sabemos: acceso no autorizado a 4 portátiles y 1 servidor de archivos
         desde el 02/03. Sin evidencia de acceso a datos de clientes.
Hecho:   equipos aislados, cuentas afectadas reseteadas, C2 bloqueado.
Necesitamos: aprobación para reimaginar el servidor de archivos
         (4 h sin servicio esta noche).
Próxima actualización: 05/03 08:00.
```

```
Management → decide y aporta recursos; recibe impacto de negocio, no detalle técnico.
Opciones → cada una con coste, tiempo y riesgo.
Actualización → sabemos / hecho / necesitamos / próxima actualización.
```

## Recursos para aprender y practicar

### Videos

- [Incident Response - CompTIA Security+ SY0-701 - 4.8](https://www.youtube.com/watch?v=X2UiMLxRdhE) — Professor Messer; las fases del proceso de respuesta. Sirve para todos los nodos de "Understand the Incident Response Process".
- [Incident Planning - CompTIA Security+ SY0-701 - 4.8](https://www.youtube.com/watch?v=CYFe16lCRMk) — Professor Messer; ejercicios de mesa, simulaciones y planificación. Nodo Preparation.
- [Digital Forensics - CompTIA Security+ SY0-701 - 4.8](https://www.youtube.com/watch?v=UtDWApdO8Zk) — Professor Messer; legal hold, cadena de custodia, adquisición, preservación. Nodo Understand Basics of Forensics.
- [Log Data - CompTIA Security+ SY0-701 - 4.9](https://www.youtube.com/watch?v=EDru1LTYDJw) — Professor Messer; qué logs existen y qué aporta cada uno en una investigación. Nodos cat, head, tail, grep.
- [Episode 6: Order of Volatility](https://www.youtube.com/watch?v=VBbUQCqKxm4) — SANS Digital Forensics and Incident Response; por qué se recoge primero lo volátil. Nodo Order of volatility.
- [Introduction to Memory Forensics](https://www.youtube.com/watch?v=1PAGcPJFwbE) — 13Cubed; qué hay en la RAM y cómo se analiza. Nodo memdump.
- [Rapid Windows Memory Analysis with Volatility 3](https://www.youtube.com/watch?v=EqGoGwVCVwM) — John Hammond; análisis práctico de un volcado con Volatility 3. Nodo memdump (análisis).
- [Introduction to Memory Forensics with Volatility 3](https://www.youtube.com/watch?v=Uk3DEgY5Ue8) — DFIRScience; instalación y primeros plugins de Volatility 3. Nodo memdump.
- [Disk Analysis with Autopsy | HackerSploit Blue Team Training](https://www.youtube.com/watch?v=o6boK9dG-Lc) — HackerSploit (publicado en el canal Akamai Developers); casos, fuentes de datos e ingest modules. Nodo autopsy.
- [Digital Forensics with FTK Imager (TryHackMe Advent of Cyber Day 8)](https://www.youtube.com/watch?v=7wB0HNf1qh4) — John Hammond; uso práctico de FTK Imager sobre una evidencia. Nodo FTK Imager.

### Lectura y documentación

- [NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final) — recomendaciones de respuesta a incidentes alineadas con el CSF 2.0 (abril 2025).
- [NIST SP 800-61 Rev. 2](https://csrc.nist.gov/pubs/sp/800/61/r2/final) — Computer Security Incident Handling Guide (2012), el ciclo de cuatro fases que siguen usando los exámenes.
- [SANS Incident Handler's Handbook](https://www.sans.org/white-papers/33901/) — Patrick Kral (2012); las seis fases PICERL con checklist.
- [NIST SP 800-86](https://csrc.nist.gov/pubs/sp/800/86/final) — Guide to Integrating Forensic Techniques into Incident Response.
- [RFC 3227](https://www.rfc-editor.org/rfc/rfc3227) — Guidelines for Evidence Collection and Archiving; orden de volatilidad y principios de recolección.
- [CISA: Incident and Vulnerability Response Playbooks](https://www.cisa.gov/resources-tools/resources/federal-government-cybersecurity-incident-and-vulnerability-response-playbooks) — playbooks públicos del gobierno de EE. UU.
- [Volatility 3 documentation](https://volatility3.readthedocs.io/en/latest/) y [repositorio](https://github.com/volatilityfoundation/volatility3) — instalación, tablas de símbolos y plugins.
- [Autopsy](https://www.autopsy.com/) y su [manual de usuario](https://sleuthkit.org/autopsy/docs/user-docs/latest/) — casos, data sources e ingest modules.
- [FTK Imager](https://www.exterro.com/digital-forensics-software/ftk-imager) — página oficial de descarga y funciones.
- [WinHex](https://www.x-ways.net/winhex/) — página oficial de X-Ways.
- [hping](https://github.com/antirez/hping) — código fuente y documentación de hping3.
- [LiME](https://github.com/504ensicsLabs/LiME), [AVML](https://github.com/microsoft/avml) y [WinPmem](https://github.com/Velocidex/WinPmem) — adquisición de memoria en Linux y Windows.
- Man pages: [dd(1)](https://man7.org/linux/man-pages/man1/dd.1.html), [grep(1)](https://man7.org/linux/man-pages/man1/grep.1.html), [tail(1)](https://man7.org/linux/man-pages/man1/tail.1.html).

### Práctica

- [TryHackMe: Incident Response Process](https://tryhackme.com/room/incidentresponseprocess) y [Preparation](https://tryhackme.com/room/preparation) — gratis; fases del proceso y cómo prepararse. Nodos de "Understand the Incident Response Process".
- [TryHackMe: Linux Incident Surface](https://tryhackme.com/room/linuxincidentsurface) y [Linux File System Analysis](https://tryhackme.com/room/linuxfilesystemanalysis) — gratis; artefactos de Linux, logs y persistencia. Nodos cat, head, tail, grep y Eradication.
- [TryHackMe: Windows Forensics 1](https://tryhackme.com/room/windowsforensics1) — registro de Windows y su análisis. Nodos autopsy y FTK Imager.
- [TryHackMe: Memory Forensics](https://tryhackme.com/room/memoryforensics) — análisis de volcados con Volatility. Nodo memdump.
- [TryHackMe: SOC Level 1](https://tryhackme.com/path/outline/soclevel1) — ruta completa con módulos de DFIR, Volatility, Autopsy y Wireshark. Todos los nodos de herramientas.
- [HTB Academy: Incident Handling Process](https://academy.hackthebox.com/course/preview/incident-handling-process) — el ciclo NIST aplicado a un caso. Nodos de proceso.
- [HTB Academy: Introduction to Digital Forensics](https://academy.hackthebox.com/course/preview/introduction-to-digital-forensics) — adquisición, memoria, disco. Nodos de forense, memdump y FTK Imager.
- [CyberDefenders](https://cyberdefenders.org/blueteam-ctf-challenges/) — laboratorios gratuitos de DFIR con pcaps, volcados e imágenes reales. Nodos Wireshark, memdump, autopsy.
- [Blue Team Labs Online](https://blueteamlabs.online/) — retos defensivos de investigación y forense. Nodos de forense y logs.
- Ejercicio en casa (imagen y hash): conecta una USB vieja, calcula `sha256sum /dev/sdX`, crea la imagen con `dd ... conv=noerror,sync status=progress`, recalcula el hash de la imagen y ábrela en Autopsy para recuperar algo que hayas borrado antes. Nodos dd, Evidence hashing, autopsy.
- Ejercicio en casa (memoria): en una VM Linux propia, abre una conexión con `nc` a otra VM y deja un comando en bash; vuelca la RAM con AVML y encuentra la conexión y el comando con `linux.pslist` y `linux.bash` de Volatility 3. Nodo memdump.
- Ejercicio en casa (triaje): expón una VM con SSH a tu red local, lanza 100 intentos fallidos desde otra VM con un script y encuentra la IP atacante en `auth.log` con la tubería `grep | sort | uniq -c | sort -rn | head`. Nodos grep, tail, head.

## Cuadro resumen

Todo lo visto, en una línea por término.

Understand the Incident Response Process

```
PICERL (SANS) → Preparation, Identification, Containment, Eradication, Recovery, Lessons Learned.
Preparation → plan, equipo, playbooks, visibilidad, jump bag, canal fuera de banda, tabletop.
Identification → triaje, declarar el incidente, severidad y alcance (scoping); documentar desde el minuto uno.
Containment → corta (aislar, bloquear C2), copia del sistema, larga (arreglos temporales).
Aislar por red → contiene y conserva la RAM; apagar → contiene y la destruye.
Eradication → causa raíz, quitar toda persistencia, reimaginar, cerrar la puerta.
Recovery → backup anterior a la intrusión, vuelta por etapas, monitorización reforzada 30-90 días.
Lessons Learned → post-mortem sin culpables en 2 semanas; MTTD, MTTR; vuelve a Preparation.
NIST 800-61 Rev. 2 → 4 fases; junta Containment, Eradication & Recovery en una.
NIST 800-61 Rev. 3 (abril 2025) → organiza la respuesta según las 6 funciones del CSF 2.0.
```

Understand Basics of Forensics

```
Forense digital → identificar, recoger, preservar y analizar evidencia de forma reproducible.
Order of volatility (RFC 3227) → registros/caché, RAM y red, temporales, disco, logs remotos, topología, archivo.
Live response → capturar RAM y red con el equipo encendido antes de apagar.
Chain of custody → quién tuvo la evidencia, cuándo y para qué; firmas y precintos.
Imagen maestra → no se toca; copia de trabajo → donde se analiza.
Imagen forense → copia bit a bit, incluye no asignado y slack space.
Raw/dd → copia pura universal; E01 → comprimida, metadatos, CRC y hash.
Write blocker → permite leer, bloquea escribir.
Hash de evidencia → prueba que copia y original son idénticos; SHA-256 o MD5 + SHA-256.
```

Tools for Incident Response and Discovery

```
dig / nslookup → resolver dominios sospechosos; comprobar sinkhole y servidor DNS usado.
nmap → puertos en escucha inesperados; alcance; verificar contención.
ping → comprobar aislamiento y vuelta al servicio; sin respuesta no prueba apagado.
arp -a → evidencia volátil; MAC del gateway cambiada = ARP spoofing.
tracert → ruta hacia el C2; verificar bloqueos.
ipconfig /all, /displaydns → DNS configurado y caché DNS; /flushdns la destruye.
curl → cabeceras y muestras desde máquina aislada; APIs de threat intel.
wireshark → pcap del incidente: Follow TCP Stream, Export Objects, beaconing.
cat / zcat → concatenar logs rotados, primer eslabón de la tubería.
head → formato e inicio del log; head -c + xxd → magic bytes.
tail -f / -F → seguir el log en vivo, incluso tras rotación.
grep → filtrar; grep | sort | uniq -c | sort -rn | head → top N.
dd → imagen raw; conv=noerror,sync; invertir if/of destruye la evidencia.
dcfldd / dc3dd → dd forense con hash y log.
hping3 → paquetes a medida; SA abierto, RA cerrado, nada filtrado.
WinHex → editor hexadecimal y de disco; file carving por firmas.
Autopsy → GUI forense sobre The Sleuth Kit; ingest modules, timeline, borrados.
mmls / fls / icat → particiones / listar archivos (y borrados) / extraer.
memdump → volcado de RAM; LiME, AVML (Linux), WinPmem, DumpIt, FTK Imager (Windows).
Volatility 3 → pstree, cmdline, netscan, malfind como primera pasada.
pslist vs psscan → lista del kernel vs firmas en toda la RAM; DKOM oculta de pslist.
FTK Imager → imagen raw/E01 con MD5 y SHA-1 verificados, Capture Memory, archivos protegidos.
```

Understand Audience

```
Audiencia → misma información, distinto detalle según quién la recibe; need to know.
Stakeholders → afectados internos y externos: dueños de sistemas, clientes, aseguradora, proveedores.
HR → todo lo que afecta a personas: insiders, errores humanos, accesos de bajas.
Legal → notificación (RGPD 72 h, SEC 8-K 4 días hábiles), privilegio, legal hold, mensajes externos.
Compliance → obligaciones de cada marco (PCI DSS, HIPAA, ISO 27001, SOC 2); auditores.
Management → decisiones de negocio y recursos; impacto en dinero y tiempo, opciones con riesgo.
```
