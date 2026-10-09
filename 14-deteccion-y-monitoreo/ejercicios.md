# Ejercicios: Detección y monitoreo

Ejercicios guiados para hacer en tu propio equipo. Cada uno dice qué vas a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría está en [README.md](README.md).

## Ejercicio 1: Suricata como IDS y luego como IPS

Nodo: [Basics of IDS and IPS](README.md#basics-of-ids-and-ips), [NIPS](README.md#nips) y [Un IPS solo puede bloquear lo que lo atraviesa](README.md#un-ips-solo-puede-bloquear-lo-que-lo-atraviesa).

Objetivo: cargar en Suricata las dos reglas de laboratorio de la nota (ping y cadena `/hola-ids` en el URI), ver sus alertas en `fast.log` y `eve.json`, y después convertirlas en `drop` con Suricata inline por NFQUEUE para comprobar que el ping y la petición dejan de llegar y que `eve.json` registra `"action":"blocked"`.

Necesitas:

- Un solo equipo Linux. Todo el ejercicio corre en él, en varias terminales, contra la interfaz de loopback (`127.0.0.1` / `lo`): **Terminal 1 (servidor víctima)** levanta un servidor web, **Terminal 2 (Suricata)** lleva el motor y **Terminal 3 (cliente)** genera el tráfico con `ping` y `curl`. No hacen falta máquinas virtuales ni una segunda tarjeta de red.
- Suricata, `jq` y Python 3. Instalación:
  - Debian/Ubuntu: `sudo apt update && sudo apt install -y suricata jq python3`
  - Arch: `sudo pacman -S suricata jq python`
  En Arch el binario de Python es `python` (hay symlink `python3` en instalaciones recientes). El paquete `suricata` de Arch no arranca el servicio solo; si lo quisieras como demonio se activa con `sudo systemctl start suricata` (Debian suele dejarlo arrancado tras instalar). En este ejercicio, de hecho, lo pararemos para lanzarlo a mano.
- nftables (para el modo inline). Instalación:
  - Debian/Ubuntu: `sudo apt install -y nftables`
  - Arch: `sudo pacman -S nftables`
  Arch ya trae nftables por defecto; si usas `iptables`, en Arch el camino moderno es `iptables-nft` (un `iptables` que escribe reglas de nftables por debajo).
- `curl` e `iputils`/`ping` para generar el tráfico (vienen de serie; en Arch el ping lo trae `iputils`).
- Tiempo estimado: 60 minutos.

> [!WARNING]
> Las reglas y el modo inline de este ejercicio actúan solo sobre tráfico de loopback que tú mismo generas contra `127.0.0.1`. No apuntes estas técnicas contra equipos ajenos ni servicios que no hayas levantado tú en este equipo.

### Pasos

1. Comprueba que tienes loopback en marcha (donde hablarán el servidor y el cliente) y abre tres terminales en el mismo equipo:
   ```bash
   ip -br addr show lo
   ping -c 1 127.0.0.1
   ```
   - `ip` → herramienta de iproute2 para ver y configurar interfaces, direcciones y rutas.
   - `-br` → salida breve (`-brief`): una línea por interfaz con nombre, estado y dirección.
   - `addr show lo` → muestra las direcciones de la interfaz de loopback `lo`; debe aparecer `lo  UNKNOWN  127.0.0.1/8`.
   - `ping -c 1 127.0.0.1` → envía un paquete ICMP Echo Request a loopback y espera respuesta (`-c 1` envía uno solo y termina).

   En una red real el "sensor" (Suricata) y el "cliente" estarían en máquinas distintas; aquí los dos papeles viven en el mismo equipo y el tráfico que observarás por `lo` es idéntico al que viajaría entre dos hosts.
2. En la **Terminal 2 (Suricata)**, detén el servicio de Suricata que el paquete arranca solo (si estaba activo), para lanzarlo a mano con tus reglas:
   ```bash
   sudo systemctl stop suricata
   suricata --build-info | grep -E 'Version|NFQueue'
   ```
   - `sudo` → ejecuta el comando que sigue como root.
   - `systemctl stop suricata` → detiene ahora el servicio `suricata` de systemd (no lo deshabilita para el próximo arranque). En Arch, si nunca lo arrancaste, este comando no hace nada y no pasa nada.
   - `suricata --build-info` → muestra con qué opciones se compiló este binario de Suricata.
   - `|` → tubería: la salida del comando de la izquierda pasa como entrada al de la derecha.
   - `grep` → filtra y muestra solo las líneas que coinciden con un patrón.
   - `-E` → interpreta el patrón como expresión regular extendida, donde `|` significa "o".
   - `'Version|NFQueue'` → patrón: líneas que contengan `Version` o `NFQueue`. `NFQueue support: yes` confirma que este binario puede funcionar inline; si dice `no`, el modo IPS del paso 8 no funcionará con ese paquete.
3. Crea `/etc/suricata/rules/local.rules` con las dos reglas de la nota. Este es el archivo completo; la segunda usa el buffer `http.uri;` de Suricata en lugar del `http_uri;` de Snort:
   ```
   alert icmp any any -> $HOME_NET any (msg:"LAB ICMP Echo Request hacia la red interna"; itype:8; sid:1000001; rev:1;)
   alert http any any -> $HOME_NET any (msg:"LAB cadena de prueba en el URI"; http.uri; content:"/hola-ids"; nocase; sid:1000002; rev:1;)
   ```
   - `alert` → acción: si la regla coincide, genera una alerta y deja pasar el paquete.
   - `icmp` / `http` → protocolo que inspecciona la regla; `http` hace que Suricata detecte el protocolo de aplicación sea cual sea el puerto.
   - `any any` (el primero) → cualquier IP de origen y cualquier puerto de origen.
   - `->` → dirección del tráfico: de origen a destino.
   - `$HOME_NET any` → destino: las redes de la variable `HOME_NET` (paso 4), cualquier puerto.
   - `msg:"..."` → texto que aparece en la alerta.
   - `itype:8` → tipo ICMP 8, Echo Request (el ping de ida).
   - `http.uri;` → buffer adhesivo: el `content` que sigue se busca solo en el URI normalizado de la petición HTTP (equivale al modificador `http_uri;` de Snort).
   - `content:"/hola-ids"` → cadena de bytes que debe aparecer en ese buffer.
   - `nocase` → la comparación del `content` anterior no distingue mayúsculas de minúsculas.
   - `sid:1000001` / `sid:1000002` → identificador único de la regla; de 1000000 en adelante es el rango reservado para reglas locales.
   - `rev:1` → revisión de la regla; se incrementa cada vez que se edita.
   - `;` → separa cada opción dentro de los paréntesis; la última también lo lleva.
4. La parte de `/etc/suricata/suricata.yaml` que importa es la variable `HOME_NET`. Como ahora todo pasa por loopback, la fijaremos a la red de loopback al lanzar Suricata, sin tocar el yaml:
   ```yaml
   vars:
     address-groups:
       HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"
   ```
   - `vars:` → sección de variables que las reglas pueden usar.
   - `address-groups:` → grupo de variables de direcciones IP (las de puertos van en `port-groups`).
   - `HOME_NET: "[...]"` → la red que se protege; aquí, por defecto, los tres rangos privados de RFC 1918. Las reglas la usan como `$HOME_NET`.

   No la edites: se sobrescribe al lanzar Suricata con `--set`. Valida la configuración y las reglas antes de arrancar, apuntando `HOME_NET` a loopback:
   ```bash
   sudo suricata -T -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/local.rules \
     --set "vars.address-groups.HOME_NET=[127.0.0.0/8]" -v
   ```
   - `sudo` → (ver paso 2).
   - `suricata` → el motor IDS/IPS.
   - `-T` → modo prueba: carga configuración y reglas, informa de errores y termina sin inspeccionar tráfico.
   - `-c /etc/suricata/suricata.yaml` → ruta del archivo de configuración principal (en Arch está en la misma ruta).
   - `-S /etc/suricata/rules/local.rules` → carga exclusivamente este archivo de reglas e ignora las reglas que liste el yaml.
   - `\` → al final de la línea, continúa el mismo comando en la línea siguiente.
   - `--set "vars.address-groups.HOME_NET=[127.0.0.0/8]"` → sobrescribe un valor del yaml solo para esta ejecución; `127.0.0.0/8` es toda la red de loopback, que incluye `127.0.0.1`. La ruta con puntos sigue la jerarquía `vars` → `address-groups` → `HOME_NET`.
   - `-v` → aumenta el nivel de detalle de los mensajes de Suricata.

   Al final debe aparecer `Configuration provided was successfully loaded. Exiting.` y una línea que dice `2 rules successfully loaded`.
5. En la **Terminal 1 (servidor víctima)**, arranca un servidor web mínimo en loopback para que el `curl` tenga a quién pedir, y déjalo corriendo:
   ```bash
   sudo python3 -m http.server 80 --bind 127.0.0.1
   ```
   - `sudo` → (ver paso 2); hace falta porque los puertos por debajo de 1024 requieren root.
   - `python3` → el intérprete de Python 3 (en Arch, `python`).
   - `-m http.server` → ejecuta el módulo `http.server` de la biblioteca estándar como programa: un servidor web que sirve el directorio actual.
   - `80` → puerto en el que escucha el servidor (por defecto usaría el 8000).
   - `--bind 127.0.0.1` → escucha solo en la interfaz de loopback, no en todas las interfaces.
6. En la **Terminal 2 (Suricata)**, lanza Suricata como IDS, escuchando una copia del tráfico de loopback:
   ```bash
   sudo rm -f /var/log/suricata/fast.log /var/log/suricata/eve.json
   sudo suricata -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/local.rules \
     --set "vars.address-groups.HOME_NET=[127.0.0.0/8]" -k none -i lo
   ```
   - `rm` → borra archivos.
   - `-f` → no da error si el archivo no existe y no pide confirmación.
   - `/var/log/suricata/fast.log` → log de alertas de Suricata en una línea por alerta.
   - `/var/log/suricata/eve.json` → log de eventos de Suricata en JSON, un objeto por línea.
   - `-c`, `-S`, `--set`, `\` → (ver paso 4).
   - `-k none` → desactiva todas las comprobaciones de checksum; en loopback el offloading del kernel hace que los checksums no estén calculados y, sin esto, Suricata descartaría los paquetes.
   - `-i lo` → captura una copia de los paquetes de la interfaz de loopback `lo` con el mejor método disponible; Suricata solo observa, no está en el camino.
   Espera a ver `Engine started.` (en versiones antiguas, `all ... packet processing threads ... started`).
7. En la **Terminal 3 (cliente)**, genera el tráfico y luego mira las alertas:
   ```bash
   ping -c 1 127.0.0.1
   curl http://127.0.0.1/hola-ids
   ```
   - `ping -c 1 127.0.0.1` → (ver paso 1); dispara la regla 1000001.
   - `curl` → cliente HTTP de línea de comandos; sin opciones hace un GET y muestra el cuerpo de la respuesta.
   - `http://127.0.0.1/hola-ids` → URL pedida al servidor local; su ruta contiene la cadena de la regla 1000002.

   El `curl` devuelve un 404 del servidor Python: es normal, la regla mira la petición, no la respuesta. En otra terminal (o en la de Suricata, cuando lo detengas):
   ```bash
   sudo cat /var/log/suricata/fast.log
   sudo jq -c 'select(.event_type=="alert") | {src_ip, dest_ip, sid: .alert.signature_id, action: .alert.action}' /var/log/suricata/eve.json
   ```
   - `sudo` → (ver paso 2); los logs de Suricata solo los lee root.
   - `cat` → muestra el contenido del archivo en pantalla.
   - `jq` → procesador de JSON en línea de comandos.
   - `-c` → salida compacta: cada resultado en una sola línea.
   - `select(.event_type=="alert")` → deja pasar solo los objetos cuyo campo `event_type` vale `alert` (en `eve.json` también hay `flow`, `http`, `stats`, etc.).
   - `|` (dentro de las comillas) → tubería de jq: pasa cada objeto filtrado a la expresión siguiente.
   - `{src_ip, dest_ip, sid: .alert.signature_id, action: .alert.action}` → construye un objeto nuevo con la IP de origen, la de destino, el sid de la regla y la acción tomada (`allowed` o `blocked`).
   - `/var/log/suricata/eve.json` → archivo de entrada de jq.
   Salida esperada (las horas cambian; origen y destino son `127.0.0.1` porque el tráfico no sale del equipo):
   ```
   10/02/2026-14:03:11.120344  [**] [1:1000001:1] LAB ICMP Echo Request hacia la red interna [**] [Classification: (null)] [Priority: 3] {ICMP} 127.0.0.1:8 -> 127.0.0.1:0
   10/02/2026-14:03:15.402981  [**] [1:1000002:1] LAB cadena de prueba en el URI [**] [Classification: (null)] [Priority: 3] {TCP} 127.0.0.1:51544 -> 127.0.0.1:80
   {"src_ip":"127.0.0.1","dest_ip":"127.0.0.1","sid":1000001,"action":"allowed"}
   {"src_ip":"127.0.0.1","dest_ip":"127.0.0.1","sid":1000002,"action":"allowed"}
   ```
   `"action":"allowed"`: Suricata vio y avisó, pero el ping tuvo respuesta y el servidor recibió la petición. Eso es un IDS. Detén Suricata con `Ctrl+C` en la Terminal 2.
8. Pasa a IPS. Cambia `alert` por `drop` en las dos reglas y sube la revisión, como se hace al editar una regla:
   ```bash
   sudo sed -i -e 's/^alert /drop /' -e 's/rev:1;/rev:2;/' /etc/suricata/rules/local.rules
   cat /etc/suricata/rules/local.rules
   ```
   - `sudo` → (ver paso 2).
   - `sed` → editor de flujo: aplica sustituciones a un texto línea por línea.
   - `-i` → edita el archivo en el sitio en lugar de imprimir el resultado.
   - `-e 's/^alert /drop /'` → primera expresión: sustituye `alert ` por `drop ` solo al principio de la línea (`^`).
   - `-e 's/rev:1;/rev:2;/'` → segunda expresión: cambia la revisión de 1 a 2.
   - `/etc/suricata/rules/local.rules` → archivo que se modifica.
   - `cat` → (ver paso 7); muestra el archivo para comprobar el cambio.

   Ahora el archivo completo queda así:
   ```
   drop icmp any any -> $HOME_NET any (msg:"LAB ICMP Echo Request hacia la red interna"; itype:8; sid:1000001; rev:2;)
   drop http any any -> $HOME_NET any (msg:"LAB cadena de prueba en el URI"; http.uri; content:"/hola-ids"; nocase; sid:1000002; rev:2;)
   ```
   - `drop` → acción: si la regla coincide, descarta el paquete y genera alerta; solo descarta de verdad si Suricata está inline (paso 10).
   - `rev:2` → nueva revisión tras la edición.
   - Resto de opciones → (ver paso 3).
9. Pon a Suricata en el camino del tráfico. Con `-i` solo recibe una copia; para poder descartar, el kernel debe entregarle cada paquete y esperar su veredicto. Crea `/etc/suricata/lab-nfqueue.nft` con este contenido completo, que manda a la cola 0 solo el tráfico del laboratorio por loopback (ping y HTTP al puerto 80); el resto del sistema no se toca:
   ```
   table inet suricata_lab {
       chain input {
           type filter hook input priority 0; policy accept;
           iif "lo" icmp type echo-request queue num 0 bypass
           iif "lo" tcp dport 80 queue num 0 bypass
           iif "lo" tcp sport 80 queue num 0 bypass
       }
   }
   ```
   - `table inet suricata_lab` → crea una tabla llamada `suricata_lab` de la familia `inet`, que cubre a la vez IPv4 e IPv6.
   - `chain input` → cadena (lista de reglas) enganchada al tráfico que llega a esta máquina.
   - `type filter` → tipo de cadena de filtrado de paquetes.
   - `hook input` → se engancha a los paquetes que llegan dirigidos a esta máquina; un paquete que generas hacia `127.0.0.1` sale por el camino de salida y se recibe por este hook de entrada, así que basta con enganchar aquí para inspeccionarlo una sola vez.
   - `priority 0` → orden de evaluación respecto a otras cadenas en el mismo hook; 0 es la prioridad estándar de filtrado.
   - `policy accept` → lo que no coincida con ninguna regla de la cadena se acepta.
   - `iif "lo"` → solo paquetes que entran por la interfaz de loopback `lo`; así solo se encola el tráfico del laboratorio y no el resto del sistema.
   - `icmp type echo-request` → los ping de ida (equivale al `itype:8` de la regla).
   - `tcp dport 80` → la petición HTTP hacia el servidor (lleva el URI que inspecciona la regla).
   - `tcp sport 80` → la respuesta del servidor; se encola también para que Suricata reconstruya la sesión TCP completa.
   - `queue num 0` → en lugar de aceptar o descartar, entrega el paquete a la cola NFQUEUE 0, donde un programa de usuario (Suricata) da el veredicto.
   - `bypass` → si nadie escucha la cola, el kernel acepta el paquete: es el fail-open de la nota. Sin `bypass` sería fail-closed y, con Suricata caído, el ping y el `curl` por loopback no funcionarían.

   Carga las reglas:
   ```bash
   sudo nft -f /etc/suricata/lab-nfqueue.nft
   sudo nft list table inet suricata_lab
   ```
   - `sudo` → (ver paso 2).
   - `nft` → herramienta de administración de nftables.
   - `-f /etc/suricata/lab-nfqueue.nft` → lee y aplica las órdenes desde ese archivo, de forma atómica.
   - `list table inet suricata_lab` → muestra el contenido de la tabla `suricata_lab` de la familia `inet`, para comprobar que se cargó.

   Si tu sistema usa `iptables` en vez de `nft`, el equivalente es:
   ```bash
   sudo iptables -I INPUT -i lo -p icmp --icmp-type echo-request -j NFQUEUE --queue-num 0 --queue-bypass
   sudo iptables -I INPUT -i lo -p tcp --dport 80 -j NFQUEUE --queue-num 0 --queue-bypass
   sudo iptables -I INPUT -i lo -p tcp --sport 80 -j NFQUEUE --queue-num 0 --queue-bypass
   ```
   - `iptables` → herramienta clásica de filtrado IPv4 del kernel (en Arch, `iptables-nft`).
   - `-I INPUT` → inserta la regla al principio de la cadena `INPUT` (tráfico entrante).
   - `-i lo` → solo paquetes que entran por la interfaz de loopback.
   - `-p icmp --icmp-type echo-request` → solo los ping de ida.
   - `-p tcp --dport 80` / `-p tcp --sport 80` → la petición al puerto 80 y la respuesta desde él.
   - `-j NFQUEUE` → salta al objetivo NFQUEUE: entrega el paquete a una cola de espacio de usuario.
   - `--queue-num 0` → número de la cola (0 a 65535).
   - `--queue-bypass` → si no hay ningún programa en la cola, deja pasar el paquete (igual que `bypass` en nft).
10. En la **Terminal 2 (Suricata)**, arranca Suricata inline sobre la cola 0:
    ```bash
    sudo rm -f /var/log/suricata/fast.log /var/log/suricata/eve.json
    sudo suricata -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/local.rules \
      --set "vars.address-groups.HOME_NET=[127.0.0.0/8]" -k none -q 0
    ```
    - `rm -f` y archivos → (ver paso 6).
    - `-c`, `-S`, `--set`, `-k none` → (ver pasos 4 y 6).
    - `-q 0` → en lugar de `-i`, funciona inline sobre la cola NFQUEUE 0: recibe cada paquete, lo inspecciona y le devuelve al kernel el veredicto de aceptar o descartar.
11. En la **Terminal 3 (cliente)**, repite el tráfico:
    ```bash
    ping -c 3 127.0.0.1
    curl -m 5 http://127.0.0.1/hola-ids
    curl -m 5 http://127.0.0.1/
    ```
    - `ping -c 3 127.0.0.1` → (ver paso 1); envía tres paquetes en lugar de uno.
    - `curl` → (ver paso 7).
    - `-m 5` → tiempo máximo total de la operación, en segundos; corta el `curl` a los 5 segundos porque la petición bloqueada nunca obtendrá respuesta.
    - `http://127.0.0.1/` → la raíz del servidor, sin la cadena de la regla: debe pasar.
    Esperado: `3 packets transmitted, 0 received, 100% packet loss`; el primer `curl` termina con `curl: (28) Operation timed out`; el segundo, sin la cadena, recibe el listado de directorio del servidor Python. En la Terminal 1 (servidor web) solo aparece la petición a `/`, nunca la de `/hola-ids`.
12. Confirma el bloqueo en `eve.json`:
    ```bash
    sudo jq -c 'select(.event_type=="alert") | {sid: .alert.signature_id, action: .alert.action}' /var/log/suricata/eve.json
    ```
    - `sudo jq -c 'select(...) | {...}'` → (ver paso 7); esta vez el objeto de salida solo lleva el sid y la acción.
    ```
    {"sid":1000001,"action":"blocked"}
    {"sid":1000001,"action":"blocked"}
    {"sid":1000001,"action":"blocked"}
    {"sid":1000002,"action":"blocked"}
    ```
    Puede haber más de una alerta del sid 1000002 si `curl` reenvía el segmento TCP descartado.
13. Comprueba el fail-open: deja las reglas `nft` cargadas, detén Suricata con `Ctrl+C` en la Terminal 2 y repite `ping -c 3 127.0.0.1` desde la Terminal 3. Ahora responde, porque con `bypass` y nadie escuchando la cola el kernel acepta. Ningún paquete se inspecciona.

### Resultado esperado

Las mismas dos reglas producen `"action":"allowed"` cuando Suricata escucha por `-i` y `"action":"blocked"` cuando recibe el tráfico por NFQUEUE: la capacidad de bloquear salió de la posición, no de la regla. También viste que, con `bypass`, un IPS caído deja pasar todo.

### Comprueba que lo lograste

- ¿Qué significa `[1:1000002:1]` en `fast.log`? Generador 1, sid 1000002, revisión 1.
- En modo IDS, ¿recibió el servidor la petición a `/hola-ids`? Sí: aparece en el log de `python3 -m http.server` con un 404.
- ¿Qué cambió entre el paso 7 y el 11 además de `alert` por `drop`? Que Suricata pasó de leer una copia (`-i lo`) a estar inline (`-q 0` con la regla `queue` de nftables). Con `drop` y `-i`, Suricata solo habría alertado.
- Si quitas `bypass` y Suricata se cae, ¿qué le pasa al tráfico de loopback del laboratorio? Pierde la conectividad encolada (fail-closed), aunque solo afecta al ping y al puerto 80 por `lo`, no al resto del sistema.

### Limpieza

```bash
sudo nft delete table inet suricata_lab
sudo rm /etc/suricata/lab-nfqueue.nft /etc/suricata/rules/local.rules
sudo systemctl start suricata
```
- `sudo` → (ver paso 2).
- `nft delete table inet suricata_lab` → borra la tabla y todas sus cadenas y reglas.
- `rm` → borra los dos archivos; sin `-f`, avisa si alguno no existe.
- `systemctl start suricata` → vuelve a arrancar el servicio (en Arch, solo si lo quieres como demonio; el paquete no lo arranca solo).

Si usaste `iptables`, borra sus tres reglas repitiendo los comandos del paso 9 con `-D` (borra la regla que coincida exactamente) en lugar de `-I`. Detén el servidor web con `Ctrl+C` en la Terminal 1. Para desinstalar: `sudo apt purge -y suricata` (Debian/Ubuntu) o `sudo pacman -Rns suricata` (Arch).

## Ejercicio 2: Rastrea un ataque simulado en el journal y en auditd

Nodo: [Event Logs](README.md#event-logs).

Objetivo: activar en Linux la auditoría que la nota da por necesaria (creación de procesos con línea de comandos y cambios en cuentas y grupos), reproducir la cadena "5 logins SSH fallidos, cuenta nueva, cuenta añadida al grupo `sudo`" y encontrarla con `journalctl` (sshd, `useradd`, `usermod`, `sudo`) y con `ausearch` (los procesos que hicieron cada cambio y quién los lanzó).

Necesitas:

- Tu propio equipo Linux con un usuario administrador (con `sudo`). Todo ocurre en el mismo equipo contra `127.0.0.1`: no hace falta ninguna máquina virtual ni una segunda máquina.
- auditd, el servidor SSH y `sshpass` (para automatizar los intentos fallidos). Instalación:
  - Debian/Ubuntu: `sudo apt update && sudo apt install -y auditd openssh-server sshpass`
  - Arch: `sudo pacman -S audit openssh sshpass`
  En Arch el paquete de auditoría es `audit` (trae `auditd`, `auditctl`, `augenrules` y `ausearch`) y el de SSH es `openssh` (el demonio es `sshd`). Ninguno arranca solo tras instalar: actívalos con `sudo systemctl start auditd` y `sudo systemctl start sshd` (en Debian/Ubuntu el servicio SSH se llama `ssh` y suele quedar arrancado al instalar).
- Tiempo estimado: 45 minutos.

### Pasos

1. Comprueba que el servidor SSH acepta contraseñas (ya lo instalaste en Necesitas):
   ```bash
   grep -ri '^passwordauthentication' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
   ```
   - `grep` → busca un patrón en archivos.
   - `-r` → recursivo: busca también en los archivos de la carpeta `sshd_config.d/`.
   - `-i` → ignora mayúsculas y minúsculas.
   - `'^passwordauthentication'` → líneas que empiezan (`^`) por esa opción; las comentadas empiezan por `#` y no cuentan.
   - `/etc/ssh/sshd_config /etc/ssh/sshd_config.d/` → el archivo de configuración del servidor SSH y la carpeta de añadidos que incluye.
   Si no sale nada, vale el valor por defecto, `yes`. Si sale `PasswordAuthentication no` (habitual en imágenes de nube), cámbialo a `yes` en ese archivo y ejecuta `sudo systemctl restart ssh` (en Arch, `sudo systemctl restart sshd`); si no, los intentos del paso 4 fallarán sin pedir contraseña y no dejarán `Failed password`.
2. Crea las reglas de auditd. Crea `/etc/audit/rules.d/lab.rules` (por ejemplo con `sudo nano /etc/audit/rules.d/lab.rules`) con este contenido completo:
   ```
   -a always,exit -F arch=b64 -S execve -F auid>=1000 -F auid!=unset -k procesos
   -w /etc/passwd -p wa -k identidad
   -w /etc/shadow -p wa -k identidad
   -w /etc/group -p wa -k identidad
   -w /etc/gshadow -p wa -k identidad
   ```
   - `-a always,exit` → regla de llamada al sistema: se añade (`-a`) a la lista `exit`, que se evalúa al terminar la llamada, con la acción `always` (registrar siempre).
   - `-F arch=b64` → solo programas de 64 bits; los números de llamada al sistema dependen de la arquitectura.
   - `-S execve` → la llamada que ejecuta un programa: la creación de proceso de Linux. Su registro `EXECVE` guarda todos los argumentos, es decir, la línea de comandos.
   - `-F auid>=1000` → solo si el auid (audit UID, el usuario que inició la sesión y que no cambia con `sudo`) es de una persona; las cuentas de sistema están por debajo de 1000.
   - `-F auid!=unset` → descarta los procesos de servicios sin sesión de usuario.
   - `-k procesos` → etiqueta (key) de los eventos de esta regla, para buscarlos con `ausearch -k`.
   - `-w /etc/passwd` → vigila ese archivo: la lista de cuentas. `/etc/shadow` guarda sus contraseñas cifradas, `/etc/group` y `/etc/gshadow` los grupos y sus miembros.
   - `-p wa` → registra escrituras (`w`) y cambios de atributos (`a`).
   - `-k identidad` → etiqueta de las cuatro vigilancias.

   Carga las reglas y compruébalas:
   ```bash
   sudo augenrules --load
   sudo auditctl -l
   ```
   - `augenrules --load` → une los archivos `.rules` de `/etc/audit/rules.d/` en `/etc/audit/audit.rules` y los carga en el kernel.
   - `auditctl -l` → lista las reglas cargadas.
   Deben aparecer tus cinco reglas, con `-F key=procesos` y `-F key=identidad`. En Arch, si `ausearch` no encuentra eventos luego, asegúrate de que el demonio está activo con `sudo systemctl start auditd`.
3. Anota la hora de inicio para acotar después las búsquedas:
   ```bash
   date '+%H:%M:%S'
   ```
   - `date` → muestra la fecha y la hora.
   - `'+%H:%M:%S'` → formato de salida: horas, minutos y segundos.
4. Simula la fuerza bruta: cinco intentos SSH fallidos contra el propio equipo con un usuario que no existe (así no bloqueas ninguna cuenta):
   ```bash
   for i in 1 2 3 4 5; do
     sshpass -p "ClaveMala$i" ssh -o StrictHostKeyChecking=no -o PubkeyAuthentication=no -o NumberOfPasswordPrompts=1 noexiste@127.0.0.1 true
   done
   ```
   - `for i in 1 2 3 4 5; do ... done` → repite el bloque cinco veces; en cada vuelta la variable `i` vale 1, 2, 3, 4 y 5.
   - `sshpass -p "ClaveMala$i"` → entrega a `ssh` esa contraseña cuando la pide; las comillas dobles expanden `$i`, así que cada intento usa `ClaveMala1`, `ClaveMala2`, etc.
   - `ssh` → cliente SSH.
   - `-o StrictHostKeyChecking=no` → acepta la clave del servidor sin preguntar la primera vez.
   - `-o PubkeyAuthentication=no` → no prueba claves públicas, solo contraseña.
   - `-o NumberOfPasswordPrompts=1` → un solo intento de contraseña por conexión, así cada vuelta deja exactamente un fallo.
   - `noexiste@127.0.0.1` → usuario y servidor: una cuenta que no existe en el propio equipo.
   - `true` → comando remoto que se ejecutaría si el login tuviera éxito; nunca llega a correr.
   Cada intento termina con `Permission denied` o sin mensaje: `sshpass` sale con el código 5, contraseña incorrecta.
5. Crea la cuenta y añádela al grupo `sudo` (el de administradores en Debian y Ubuntu; en Arch el grupo equivalente suele ser `wheel`):
   ```bash
   sudo useradd -m -s /bin/bash labuser
   sudo passwd labuser
   sudo usermod -aG sudo labuser
   ```
   - `useradd` → crea una cuenta local.
   - `-m` → crea su directorio personal, `/home/labuser`.
   - `-s /bin/bash` → su shell de inicio de sesión.
   - `passwd labuser` → pide por teclado la contraseña de la cuenta, sin escribirla en la línea de comandos (y por tanto en el registro `EXECVE`).
   - `usermod` → modifica una cuenta existente.
   - `-aG sudo` → `-G` indica grupos suplementarios y `-a` los añade sin quitar los que ya tenía; `sudo` es el grupo que puede usar `sudo` (en Arch, cambia `sudo` por `wheel` si ese es el que tienes habilitado).
6. Busca los logins fallidos en el journal del servicio SSH y resúmelos por IP y usuario:
   ```bash
   sudo journalctl -u ssh --since "30 minutes ago" --no-pager -g "Failed password"
   sudo journalctl -u ssh --since "30 minutes ago" --no-pager -g "Failed password" -o cat | awk '{print $(NF-3), $(NF-5)}' | sort | uniq -c
   ```
   - `journalctl` → consulta el journal de systemd.
   - `-u ssh` → solo los mensajes de la unidad `ssh.service`, el servidor SSH en Debian y Ubuntu (en Arch y Fedora se llama `sshd`, así que allí usa `-u sshd`).
   - `--since "30 minutes ago"` → solo entradas de los últimos 30 minutos.
   - `--no-pager` → imprime sin abrir el paginador `less`.
   - `-g "Failed password"` → (`--grep`) solo mensajes que contienen ese texto.
   - `-o cat` → imprime solo el texto del mensaje, sin fecha, equipo ni proceso.
   - `|` → tubería: pasa la salida al comando siguiente.
   - `awk '{print $(NF-3), $(NF-5)}'` → `awk` separa cada línea en campos por espacios y `NF` es el número de campos; en `... for invalid user noexiste from 127.0.0.1 port 51234 ssh2`, el tercero desde el final es la IP y el quinto el usuario.
   - `sort | uniq -c` → ordena y cuenta las líneas repetidas: intentos por cada pareja IP y usuario.
   Esperado: cinco líneas como esta, y después el resumen:
   ```
   oct 08 10:41:07 lab sshd[2311]: Failed password for invalid user noexiste from 127.0.0.1 port 51234 ssh2
         5 127.0.0.1 noexiste
   ```
   `invalid user` significa que la cuenta no existe; contra una cuenta real con contraseña errónea la línea sería `Failed password for ana from ...`. En Ubuntu el mismo texto está también en `/var/log/auth.log` (`sudo grep "Failed password" /var/log/auth.log`); Debian 12 y Arch no instalan `rsyslog` por defecto, así que allí solo está en el journal.
7. Busca la cuenta creada, su contraseña y su alta en `sudo`, y quién lo hizo:
   ```bash
   sudo journalctl -t useradd -t passwd -t usermod -t sudo --since "30 minutes ago" --no-pager
   ```
   - `journalctl --since ... --no-pager` → (ver paso 6).
   - `-t useradd -t passwd -t usermod -t sudo` → mensajes con cualquiera de esos identificadores de syslog: cada programa firma sus mensajes con su nombre; repetir `-t` suma identificadores.
   Esperado (resumido):
   ```
   sudo[2402]:     jony : TTY=pts/0 ; PWD=/home/jony ; USER=root ; COMMAND=/usr/sbin/useradd -m -s /bin/bash labuser
   useradd[2404]: new group: name=labuser, GID=1002
   useradd[2404]: new user: name=labuser, UID=1002, GID=1002, home=/home/labuser, shell=/bin/bash, from=/dev/pts/0
   passwd[2410]: pam_unix(passwd:chauthtok): password changed for labuser
   sudo[2415]:     jony : TTY=pts/0 ; PWD=/home/jony ; USER=root ; COMMAND=/usr/sbin/usermod -aG sudo labuser
   usermod[2417]: add 'labuser' to group 'sudo'
   usermod[2417]: add 'labuser' to shadow group 'sudo'
   ```
   La línea de `sudo` es el autor (tu usuario) con la orden exacta; las de `useradd` y `usermod` son el cambio. El grupo de administradores sale por su nombre, `sudo`.
8. Busca los procesos que hicieron esos cambios en auditd:
   ```bash
   sudo ausearch -k procesos --start today --format text | grep -E 'useradd|usermod|passwd'
   sudo ausearch -k procesos --start today -i -x /usr/sbin/useradd | grep -E '^type=(SYSCALL|EXECVE)'
   ```
   - `ausearch` → busca en el log de auditd, `/var/log/audit/audit.log`, que solo puede leer root.
   - `-k procesos` → eventos de la regla `execve` del paso 2.
   - `--start today` → desde las 00:00 de hoy; también admite `recent` (10 minutos) o una hora como la que anotaste en el paso 3 (`--start 10:40:00`).
   - `--format text` → resume cada evento en una frase: quién, en nombre de quién, qué hizo y sobre qué.
   - `grep -E 'useradd|usermod|passwd'` → deja las ejecuciones de esos tres programas; `|` significa "o".
   - `-i` → interpreta los números: muestra nombres de usuario en vez de UID y la llamada al sistema por su nombre.
   - `-x /usr/sbin/useradd` → solo eventos cuyo ejecutable es ese.
   - `grep -E '^type=(SYSCALL|EXECVE)'` → deja el registro de la llamada (quién y con qué programa) y el de los argumentos.
   Esperado:
   ```
   At 10:42:13 10/08/2026 jony, acting as root, successfully executed /usr/sbin/useradd
   At 10:42:20 10/08/2026 jony, acting as root, successfully executed /usr/bin/passwd
   At 10:42:31 10/08/2026 jony, acting as root, successfully executed /usr/sbin/usermod
   type=EXECVE msg=audit(10/08/2026 10:42:13.512:871) : argc=5 a0=useradd a1=-m a2=-s a3=/bin/bash a4=labuser
   type=SYSCALL msg=audit(10/08/2026 10:42:13.512:871) : arch=x86_64 syscall=execve success=yes ... ppid=2403 pid=2404 auid=jony uid=root ... comm=useradd exe=/usr/sbin/useradd ... key=procesos
   ```
   `auid=jony uid=root` es lo que el journal no siempre da: el programa corrió como root, pero la sesión la abrió `jony`. Aunque el atacante se haga root con `sudo -i`, cada orden posterior sigue llevando su auid.
9. Comprueba qué archivos de identidad cambiaron:
   ```bash
   sudo ausearch -k identidad --start today --format text
   ```
   - `ausearch -k identidad --start today --format text` → (ver paso 8); eventos de las vigilancias sobre `/etc/passwd`, `/etc/shadow`, `/etc/group` y `/etc/gshadow`.
   Verás varias líneas por cambio, todas de `jony, acting as root`, sobre esos cuatro archivos y `using /usr/sbin/useradd`, `/usr/bin/passwd` o `/usr/sbin/usermod`: `useradd` toca los cuatro, `passwd` solo `/etc/shadow` y `usermod -aG` solo `/etc/group` y `/etc/gshadow`.
10. Escribe la línea de tiempo en un archivo de texto, con el formato de la nota: hora, fuente, qué significa. Por ejemplo `10:41  sshd "Failed password" x5  noexiste desde 127.0.0.1  fuerza bruta`.

### Resultado esperado

Una línea de tiempo de cuatro filas (5 `Failed password` de sshd, `useradd` de `labuser`, cambio de contraseña, `usermod` al grupo `sudo`) que cuenta el ataque simulado en orden, sacada solo de los logs, con las consultas que la producen y, para cada cambio, el proceso que lo hizo y el usuario que abrió la sesión (auid).

### Comprueba que lo lograste

- ¿Qué diferencia `Failed password for invalid user noexiste` de `Failed password for ana`? La primera es contra una cuenta que no existe; la segunda, contra una cuenta real con contraseña incorrecta, que es más grave porque el atacante ya acertó un nombre válido.
- ¿Qué habrías perdido sin la regla `execve` del paso 2? Las órdenes que no pasan por `sudo`: si el atacante ya tiene una shell de root, `sudo` no registra nada, pero auditd sí, con la línea de comandos completa y el auid de quien abrió la sesión.
- ¿Qué habría quedado escrito con `sudo useradd -m -p "$(openssl passwd -6 Clave123)" labuser`? La contraseña en claro en el registro `EXECVE` de `openssl` (sus argumentos) y su hash en la línea de `sudo`, en logs que se envían al SIEM. Por eso el paso 5 usa `passwd`, que la pide por teclado.
- ¿Por qué `ausearch` necesita `sudo` y dónde está su log? Porque lee `/var/log/audit/audit.log`, que solo puede leer root. Es una fuente distinta del journal: lo que vigilan tus reglas está ahí, no en `journalctl`.

### Limpieza

```bash
sudo userdel -r labuser
sudo rm /etc/audit/rules.d/lab.rules
sudo augenrules --load
```
- `userdel -r labuser` → borra la cuenta `labuser`, su grupo personal y, con `-r`, su directorio personal; deja de estar en el grupo `sudo`.
- `rm /etc/audit/rules.d/lab.rules` → borra el archivo de reglas del paso 2.
- `augenrules --load` → (ver paso 2); recarga las reglas, ya sin las tuyas.

Para desinstalar lo añadido: `sudo apt purge -y auditd sshpass` (Debian/Ubuntu) o `sudo pacman -Rns audit sshpass` (Arch); deja el servidor SSH si lo usas para administrar el equipo.

## Ejercicio 3: Escribe un mensaje syslog y calcula su PRI

Nodo: [syslogs](README.md#syslogs).

Objetivo: escribir un mensaje con facility `auth` y severidad `warning`, encontrarlo con `journalctl -p warning`, calcular a mano su PRI (debe dar 36) y comprobar ese número viendo el mensaje tal como viaja por la red en formato RFC 5424.

Necesitas: cualquier Linux con systemd (Ubuntu, Debian, Arch) y `logger` (paquete `bsdutils` en Debian/Ubuntu, `util-linux` en Arch; vienen instalados). Para leer el journal del sistema sin `sudo` tu usuario debe estar en el grupo `adm`, `systemd-journal` o `wheel`; si no, antepón `sudo` a `journalctl`. Python 3 para el receptor UDP (en Arch, `python`). Tiempo estimado: 15 minutos.

### Pasos

1. Escribe el mensaje en el log local:
   ```bash
   logger -p auth.warning "prueba de syslog"
   ```
   - `logger` → escribe un mensaje en el log del sistema (syslog o journal).
   - `-p auth.warning` → prioridad del mensaje en formato `facility.severidad`: facility `auth` (4) y severidad `warning` (4).
   - `"prueba de syslog"` → texto del mensaje.
2. Encuéntralo filtrando por severidad:
   ```bash
   journalctl -p warning --since "5 minutes ago" --no-pager | grep "prueba de syslog"
   ```
   - `journalctl` → consulta el journal de systemd.
   - `-p warning` → muestra la severidad 4 y todas las más graves (0 a 4).
   - `--since "5 minutes ago"` → solo entradas de los últimos 5 minutos; acepta fechas absolutas o relativas.
   - `--no-pager` → imprime directamente sin abrir el paginador `less`, necesario para encadenar con `|`.
   - `|` → (ver ejercicio 1, paso 2).
   - `grep "prueba de syslog"` → deja solo las líneas que contienen ese texto.

   ```
   oct 02 12:34:46 equipo jony[72901]: prueba de syslog
   ```
   Repite con `journalctl -p err --since "5 minutes ago" | grep "prueba de syslog"`: no sale nada, porque `err` (3) es más grave que tu mensaje.
3. Mira los campos que guardó el journal:
   ```bash
   journalctl --since "5 minutes ago" -o verbose --no-pager -g "prueba de syslog" | grep -E 'PRIORITY|SYSLOG_FACILITY|MESSAGE='
   ```
   - `journalctl --since "5 minutes ago" --no-pager` → (ver paso 2).
   - `-o verbose` → formato de salida que muestra todos los campos guardados de cada entrada, uno por línea.
   - `-g "prueba de syslog"` → (`--grep`) solo entradas cuyo campo `MESSAGE` coincide con ese patrón.
   - `grep -E` → (ver ejercicio 1, paso 2).
   - `'PRIORITY|SYSLOG_FACILITY|MESSAGE='` → deja las líneas de severidad, facility y texto del mensaje.
   ```
       SYSLOG_FACILITY=4
       PRIORITY=4
       MESSAGE=prueba de syslog
   ```
4. Calcula el PRI a mano con la fórmula de la nota, `PRI = facility × 8 + severity`: facility `auth` = 4, severidad `warning` = 4, así que 4 × 8 + 4 = 36. Anótalo antes de seguir.
5. Comprueba el número en el cable. Abre un receptor UDP en el puerto 5514 de tu propio equipo (no hace falta root por encima del 1024) y deja esa terminal esperando:
   ```bash
   python3 -c "import socket; s=socket.socket(socket.AF_INET, socket.SOCK_DGRAM); s.bind(('127.0.0.1', 5514)); print(s.recv(2048).decode())"
   ```
   - `python3 -c "..."` → ejecuta el código Python que va entre comillas sin crear un archivo (en Arch, `python -c`).
   - `import socket` → carga el módulo de sockets de red.
   - `socket.socket(socket.AF_INET, socket.SOCK_DGRAM)` → crea un socket IPv4 (`AF_INET`) de datagramas, es decir, UDP (`SOCK_DGRAM`).
   - `s.bind(('127.0.0.1', 5514))` → escucha en la dirección local 127.0.0.1, puerto 5514.
   - `s.recv(2048)` → espera un datagrama y lee hasta 2048 bytes.
   - `.decode()` → convierte los bytes en texto.
   - `print(...)` → lo muestra; después el programa termina.

   En otra terminal, envía el mismo mensaje a ese receptor en formato RFC 5424 por UDP:
   ```bash
   logger -p auth.warning -n 127.0.0.1 -P 5514 -d --rfc5424 "prueba de syslog"
   ```
   - `logger -p auth.warning` → (ver paso 1).
   - `-n 127.0.0.1` → envía el mensaje a ese servidor syslog remoto en lugar de al log local.
   - `-P 5514` → puerto de destino (por defecto 514).
   - `-d` → usa solo UDP.
   - `--rfc5424` → formato del protocolo syslog de RFC 5424 (el predeterminado al enviar a remoto).
   - `"prueba de syslog"` → texto del mensaje.
   El receptor imprime algo así y termina:
   ```
   <36>1 2026-10-02T12:34:46.740976-06:00 equipo jony - - [timeQuality tzKnown="1" isSynced="1" syncAccuracy="1497500"] prueba de syslog
   ```
   `<36>` es el PRI que calculaste, `1` la versión del formato, luego marca de tiempo con zona horaria, equipo, aplicación, PID y MSGID (`-` = ninguno) y los datos estructurados que `logger` añade sobre la sincronización del reloj.
6. Practica la fórmula al revés con dos casos más y compruébalos repitiendo el paso 5 con otra `-p`:
   - `logger -p local0.err ...` → 16 × 8 + 3 = 131.
   - `logger -p cron.info ...` → 9 × 8 + 6 = 78.
7. Descompón un PRI recibido: si un firewall envía `<134>`, 134 ÷ 8 = 16 con resto 6, es decir facility 16 (`local0`) y severidad 6 (`Informational`).

### Resultado esperado

El mensaje localizado en el journal con `journalctl -p warning`, el PRI 36 calculado a mano y el mismo `<36>` visto en el mensaje RFC 5424 que salió por UDP.

### Comprueba que lo lograste

- ¿Sale tu mensaje con `journalctl -p notice`? Sí: `notice` (5) incluye todo lo que va de 0 a 5, y `warning` es 4.
- ¿Qué PRI tiene `authpriv.crit`? 10 × 8 + 2 = 82.
- ¿Qué se pierde si el receptor no estaba escuchando cuando enviaste por `-d`? El mensaje: UDP no confirma la entrega y `logger` no da error. Con `-T` (TCP) sí fallaría la conexión.

## Ejercicio 4: Analiza DNS y SNI de tu propia navegación

Nodo: [Packet Captures](README.md#packet-captures).

Objetivo: capturar 5 minutos de tu propia navegación con `tcpdump` y, usando solo filtros de visualización de Wireshark (y su versión en `tshark`), responder dos preguntas con números: cuántos dominios distintos consultaste por DNS y a qué SNI fue cada conexión TLS.

Necesitas:

- Tu propio equipo Linux y tu propia red. Captura solo tu tráfico; capturar el de otros sin permiso no es un laboratorio.
- `tcpdump` y Wireshark (que trae `tshark`). Instalación:
  - Debian/Ubuntu: `sudo apt install -y tcpdump wireshark tshark`
  - Arch: `sudo pacman -S tcpdump wireshark-qt` (la GUI; `wireshark-qt` ya incluye `tshark`). Si solo quieres la línea de comandos, `sudo pacman -S tcpdump wireshark-cli`.
  Para capturar sin ser root debes estar en el grupo `wireshark`: `sudo usermod -aG wireshark $USER` y vuelve a iniciar sesión. En Debian/Ubuntu, además, `sudo dpkg-reconfigure wireshark-common` y elige "sí" para permitir la captura a los usuarios del grupo `wireshark`.
- Tiempo estimado: 30 minutos.

### Pasos

1. Desactiva el DNS cifrado del navegador para esta prueba, o casi no verás consultas DNS: en Firefox, Ajustes → Privacidad y seguridad → DNS sobre HTTPS → Desactivado; en Chrome o Brave, Configuración → Privacidad y seguridad → Seguridad → "Usar DNS seguro" desactivado. Comprueba también si tu sistema usa DNS sobre TLS:
   ```bash
   resolvectl status | grep -i -E 'DNSOverTLS|Current DNS'
   ```
   - `resolvectl status` → muestra la configuración de resolución DNS de systemd-resolved, global y por interfaz.
   - `|` → (ver ejercicio 1, paso 2).
   - `grep` → filtra líneas por patrón.
   - `-i` → no distingue mayúsculas de minúsculas.
   - `-E` → expresión regular extendida (`|` significa "o").
   - `'DNSOverTLS|Current DNS'` → líneas del modo DNS sobre TLS y del servidor DNS en uso.
   Si dice `+DNSOverTLS`, el DNS del sistema irá cifrado al puerto 853 y tampoco lo verás; anótalo como hallazgo.
2. Averigua la interfaz por la que sales a Internet:
   ```bash
   ip route get 1.1.1.1
   ```
   - `ip route get` → pregunta al kernel qué ruta usaría para llegar a una dirección, sin enviar nada.
   - `1.1.1.1` → una IP cualquiera de Internet; la palabra tras `dev` en la respuesta es la interfaz (por ejemplo `wlan0` o `enp3s0`).
3. Captura exactamente 5 minutos:
   ```bash
   sudo timeout 300 tcpdump -i wlan0 -w navegacion.pcap 'port 53 or port 443'
   ```
   - `sudo` → (ver ejercicio 1, paso 2); capturar paquetes requiere root.
   - `timeout 300` → ejecuta el comando que sigue y lo termina a los 300 segundos (5 minutos).
   - `tcpdump` → capturador de paquetes de línea de comandos.
   - `-i wlan0` → interfaz donde capturar; pon la del paso 2.
   - `-w navegacion.pcap` → escribe los paquetes en bruto en ese archivo en lugar de mostrarlos.
   - `'port 53 or port 443'` → filtro de captura BPF: guarda solo DNS (53) y el tráfico al 443 (TCP para HTTPS y UDP para QUIC), en origen o destino.

   Mientras corre, visita al menos cinco sitios distintos. Al terminar, `tcpdump` dice cuántos paquetes capturó. Haz tuyo el archivo:
   ```bash
   sudo chown $USER: navegacion.pcap
   ```
   - `chown` → cambia el dueño de un archivo.
   - `$USER:` → tu usuario como dueño; los dos puntos sin grupo detrás asignan además el grupo principal de ese usuario.
   - `navegacion.pcap` → archivo creado por root en el paso anterior.
4. Ábrelo en Wireshark (`wireshark navegacion.pcap`) y aplica en la barra de filtro de visualización:
   ```
   dns.flags.response == 0
   ```
   - `dns.flags.response` → campo del bit QR de la cabecera DNS: 0 es pregunta, 1 es respuesta.
   - `== 0` → deja solo las preguntas, sin las respuestas.

   Para contar dominios distintos, ve a Statistics → DNS, o cuenta con `tshark`:
   ```bash
   tshark -r navegacion.pcap -Y 'dns.flags.response == 0' -T fields -e dns.qry.name | sort -u | wc -l
   tshark -r navegacion.pcap -Y 'dns.flags.response == 0' -T fields -e dns.qry.name | sort | uniq -c | sort -rn | head -20
   ```
   - `tshark` → versión de terminal de Wireshark.
   - `-r navegacion.pcap` → lee los paquetes de ese archivo en lugar de capturar.
   - `-Y 'dns.flags.response == 0'` → aplica ese filtro de visualización de Wireshark.
   - `-T fields` → en lugar del resumen, imprime solo los campos pedidos con `-e`.
   - `-e dns.qry.name` → campo a imprimir: el nombre consultado.
   - `sort` → ordena las líneas alfabéticamente.
   - `-u` → elimina las líneas repetidas tras ordenar.
   - `wc -l` → cuenta las líneas: el número de dominios distintos.
   - `uniq -c` → junta las líneas iguales consecutivas y antepone cuántas veces aparecen.
   - `sort -rn` → ordena por número (`-n`) de mayor a menor (`-r`).
   - `head -20` → muestra solo las 20 primeras líneas.
   La primera cifra es la respuesta a la primera pregunta. La segunda lista suele sorprender: CDNs, telemetría, anuncios y dominios que nunca escribiste.
5. Para las conexiones TLS, filtra los Client Hello (el primer mensaje del handshake, donde va el SNI en claro):
   ```
   tls.handshake.type == 1
   ```
   - `tls.handshake.type` → tipo de mensaje del handshake TLS.
   - `== 1` → ClientHello, el primer mensaje del cliente.

   Clic derecho sobre el campo Server Name de cualquier paquete (Transport Layer Security → Handshake Protocol → Extension: server_name → Server Name) → Apply as Column. La versión en terminal:
   ```bash
   tshark -r navegacion.pcap -Y 'tls.handshake.type == 1' -T fields -e ip.dst -e ipv6.dst -e tls.handshake.extensions_server_name | sort | uniq -c | sort -rn
   ```
   - `tshark -r`, `-Y`, `-T fields`, `sort`, `uniq -c`, `sort -rn` → (ver paso 4).
   - `'tls.handshake.type == 1'` → solo los ClientHello.
   - `-e ip.dst` → campo IP de destino IPv4.
   - `-e ipv6.dst` → campo IP de destino IPv6; en cada línea uno de los dos sale vacío.
   - `-e tls.handshake.extensions_server_name` → el SNI: nombre del servidor que pide el cliente.
   Cada línea es número de conexiones, IP de destino y SNI. Las conexiones QUIC (UDP 443) también aparecen: Wireshark descifra el paquete Initial de QUIC y muestra su Client Hello.
6. Cruza las dos respuestas: los SNI deberían coincidir en su mayoría con nombres que consultaste por DNS. Busca los que no:
   ```bash
   tshark -r navegacion.pcap -Y 'dns.flags.response == 0' -T fields -e dns.qry.name | sort -u > dns.txt
   tshark -r navegacion.pcap -Y 'tls.handshake.type == 1' -T fields -e tls.handshake.extensions_server_name | sort -u > sni.txt
   comm -13 dns.txt sni.txt
   ```
   - `tshark -r`, `-Y`, `-T fields`, `-e`, `sort -u` → (ver pasos 4 y 5).
   - `> dns.txt` / `> sni.txt` → redirige la salida a ese archivo, creándolo o vaciándolo antes.
   - `comm` → compara dos archivos ordenados y saca tres columnas: solo en el primero, solo en el segundo y en ambos.
   - `-13` → suprime la columna 1 (solo en `dns.txt`) y la 3 (en ambos): quedan los nombres que están solo en `sni.txt`.
   - `dns.txt sni.txt` → los dos archivos, ya ordenados con `sort -u` como exige `comm`.
   Lo que sale son SNI sin consulta DNS visible: nombres que estaban en caché antes de empezar a capturar o resueltos por DNS cifrado. Si ves un SNI genérico como `cloudflare-ech.com`, esa conexión usó ECH (Encrypted Client Hello) y el nombre real viajó cifrado.
7. Escribe las dos respuestas en un texto: número de dominios distintos consultados y la lista SNI → IP de destino.

### Resultado esperado

Dos datos sacados de tu pcap con filtros reproducibles: el número de dominios distintos de `dns.qry.name` y la tabla SNI → IP de cada Client Hello, más la lista de SNI sin consulta DNS visible y su explicación.

### Comprueba que lo lograste

- ¿Qué diferencia hay entre `'port 53 or port 443'` en `tcpdump` y `dns.flags.response == 0` en Wireshark? El primero es un filtro de captura BPF y decide qué se guarda; el segundo es de visualización y solo esconde paquetes del archivo.
- ¿Puedes ver la URL completa de una página HTTPS en el pcap? No: solo el SNI (el nombre del servidor), la IP y el certificado; la ruta va cifrada.
- Si un equipo de tu red hiciese 500 consultas a un mismo dominio raro en 5 minutos, ¿con qué comando lo verías? Con el `uniq -c | sort -rn` del paso 4.

### Limpieza

El pcap contiene tu navegación: bórralo cuando termines.

```bash
shred -u navegacion.pcap dns.txt sni.txt
```
- `shred` → sobrescribe el contenido de los archivos varias veces para dificultar su recuperación (en SSD y sistemas de archivos con copia en escritura no lo garantiza).
- `-u` → después de sobrescribir, borra el archivo.
- `navegacion.pcap dns.txt sni.txt` → archivos a destruir.

Vuelve a activar el DNS cifrado del navegador si lo tenías.

## Ejercicio 5: Enriquece un hash y tres dominios sin subir nada

Nodo: [VirusTotal](README.md#virustotal), [WHOIS](README.md#whois) y [Subir una muestra a un servicio público es publicarla](README.md#subir-una-muestra-a-un-servicio-público-es-publicarla).

Objetivo: calcular el SHA-256 del archivo de prueba EICAR en tu equipo, consultar su informe en VirusTotal buscando por hash (sin subir el archivo) e interpretar el ratio, la primera fecha de envío y los nombres; después obtener la fecha de creación de tres dominios con RDAP, en lookup.icann.org y por línea de comandos.

Necesitas: Linux con navegador, `sha256sum`, `curl` y Python 3 (en Arch, `python`). EICAR no es malware: es una cadena de 68 bytes que los antivirus acuerdan detectar para probar que funcionan. Aun así, tu antivirus puede borrarla al crearla; en ese caso calcula el hash directamente desde la cadena como en el paso 1. Tiempo estimado: 20 minutos.

### Pasos

1. Calcula el SHA-256 de la cadena EICAR sin escribir el archivo en disco:
   ```bash
   printf '%s' 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' | sha256sum
   ```
   - `printf` → imprime texto según un formato.
   - `'%s'` → formato: imprime el argumento tal cual, sin interpretar `%` ni `\` dentro de él y sin añadir salto de línea; por eso se usa `printf '%s'` y no `echo`.
   - `'X5O!P%@AP...'` → la cadena EICAR; las comillas simples evitan que la shell interprete `$` y `!`.
   - `|` → (ver ejercicio 1, paso 2).
   - `sha256sum` → calcula el SHA-256 de lo que recibe; sin archivo lee la entrada estándar, que en la salida aparece como `-`.

   ```
   275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f  -
   ```
   Si te sale otro hash, sobra o falta un byte (por ejemplo un salto de línea final).
2. Busca ese hash en VirusTotal. Abre [virustotal.com](https://www.virustotal.com/), pestaña SEARCH (no FILE), pega el hash y pulsa Enter, o ve directo a:
   ```
   https://www.virustotal.com/gui/file/275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f
   ```
   - `https://www.virustotal.com/gui/` → interfaz web de VirusTotal.
   - `file/` → tipo de objeto consultado: un archivo, identificado por su hash.
   - `275a02...fd0f` → el SHA-256 del paso 1; la consulta solo envía este hash, no el contenido.
   No uses la pestaña FILE: eso sube el archivo.
3. Lee el informe y anota en un texto:
   - Detection: el ratio (será alto, del orden de 60 de unos 70 motores). Un ratio alto aquí no significa peligro, significa que los motores cumplen el acuerdo EICAR: el veredicto se interpreta con contexto.
   - Details → History → First Submission: la fecha en que alguien lo subió por primera vez. Un archivo conocido desde hace muchos años es lo contrario de una muestra dirigida.
   - Details → Names: con cuántos nombres distintos circula el mismo contenido (`eicar.com`, `eicar.txt` y cientos más). Mismo hash, nombres distintos: el nombre de un archivo no es un IOC fiable, el hash sí.
4. Consulta la fecha de creación de tres dominios en [lookup.icann.org](https://lookup.icann.org/): escribe `google.com` y pulsa Lookup; en el resultado busca Dates → Created. Repite con `wikipedia.org` y `github.com`. lookup.icann.org consulta por RDAP, el sucesor de WHOIS.
5. Haz la misma consulta por línea de comandos contra el servidor RDAP del registro de cada TLD (los de `.com` y `.org` salen de la lista oficial de IANA en `https://data.iana.org/rdap/dns.json`):
   ```bash
   for url in https://rdap.verisign.com/com/v1/domain/google.com \
              https://rdap.publicinterestregistry.org/rdap/domain/wikipedia.org \
              https://rdap.verisign.com/com/v1/domain/github.com; do
     curl -s "$url" | python3 -c "import json, sys; d = json.load(sys.stdin); print(d['ldhName'], [e['eventDate'] for e in d['events'] if e['eventAction'] == 'registration'])"
   done
   ```
   - `for url in ... ; do ... done` → bucle de la shell: repite el cuerpo una vez por cada URL, guardándola en la variable `url`.
   - `\` → continúa la lista en la línea siguiente.
   - `https://rdap.verisign.com/com/v1/domain/...` → servidor RDAP del registro de `.com`, consulta de dominio.
   - `https://rdap.publicinterestregistry.org/rdap/domain/...` → servidor RDAP del registro de `.org`.
   - `curl` → (ver ejercicio 1, paso 7).
   - `-s` → modo silencioso: sin barra de progreso ni mensajes de error.
   - `"$url"` → la URL del turno; entre comillas para que la shell no la parta.
   - `|` → pasa el JSON de RDAP a Python.
   - `python3 -c "..."` → (ver ejercicio 3, paso 5; en Arch, `python -c`).
   - `import json, sys` → módulos para leer JSON y la entrada estándar.
   - `d = json.load(sys.stdin)` → convierte la respuesta en un diccionario.
   - `d['ldhName']` → nombre del dominio tal como lo devuelve el registro.
   - `[e['eventDate'] for e in d['events'] if e['eventAction'] == 'registration']` → de la lista de eventos, la fecha del evento de registro (creación).

   ```
   GOOGLE.COM ['1997-09-15T04:00:00Z']
   wikipedia.org ['2001-01-13T00:12:14.754Z']
   GITHUB.COM ['2007-10-09T18:20:50Z']
   ```
6. Si tienes `whois` instalado (`sudo apt install -y whois` en Debian/Ubuntu o `sudo pacman -S whois` en Arch), compáralo con el formato antiguo:
   ```bash
   whois github.com | grep -i -E 'creation date|registrar:|registrant organization'
   ```
   - `whois github.com` → consulta por el protocolo WHOIS clásico (puerto 43) los datos de registro del dominio.
   - `grep -i -E` → (ver ejercicio 4, paso 1).
   - `'creation date|registrar:|registrant organization'` → líneas de fecha de creación, registrador y organización titular.
   Fíjate en si el titular aparece o sale `REDACTED FOR PRIVACY`.
7. Cierra con una conclusión por escrito de dos líneas: qué te dijo cada servicio y qué enviaste a cada uno (un hash y tres nombres de dominio, ninguno de ellos dato interno).

### Resultado esperado

Un texto con el SHA-256 de EICAR, su ratio de detección, su fecha de primer envío y su número de nombres, las fechas de creación de tres dominios obtenidas por dos vías que coinciden, y la constancia de que no subiste ningún archivo.

### Comprueba que lo lograste

- ¿Qué habría pasado si hubieses usado la pestaña FILE con un documento interno? Que el archivo quedaría disponible para los clientes de pago de VirusTotal.
- ¿Un `0/72` en otro hash significaría que el archivo es seguro? No: un malware nuevo o dirigido puede tener cero detecciones.
- Un correo de "tu banco" llega desde un dominio cuyo `registration` es de hace dos días. ¿Qué concluyes aunque urlvoid diga 0 detecciones? Que es sospechoso igual: los bancos no estrenan dominio para escribir a sus clientes.

## Ejercicio 6: SIEM con Wazuh

Nodo: [SIEM](README.md#siem), [Event Logs](README.md#event-logs) y [syslogs](README.md#syslogs).

Objetivo: montar un SIEM real, Wazuh 4.14 (la versión vigente; la 4.14.8 al escribir esto), con el servidor y un agente en tu propio equipo; generar los mismos eventos del ejercicio 2 (logins SSH fallidos, `sudo`, cuenta nueva, alta en el grupo `sudo`) más un cambio de archivo vigilado por FIM; encontrarlos en el dashboard con consultas por regla y por grupo, ver su mapeo a MITRE ATT&CK, cubrir con una regla propia el evento que Wazuh no alerta de fábrica y redactar el triaje de una alerta como lo haría un analista de SOC.

Necesitas:

- Tu propio equipo Linux. El servidor de Wazuh (manager + indexador + dashboard) y el agente se instalan en el mismo equipo; el agente se conecta al manager por `127.0.0.1`. Usarás varias terminales y el navegador local. No hacen falta máquinas virtuales ni una segunda máquina. En una red real el manager y los agentes estarían en máquinas distintas; aquí el SIEM completo corre en tu equipo y el agente reporta por loopback.
- Recursos: Wazuh recomienda 4 vCPU, 8 GB de RAM y 50 GB de disco para hasta 25 agentes; el instalador todo-en-uno exige un mínimo de ~4 GB de RAM y 2 núcleos (con menos de unos 3,7 GB se niega a seguir). Con 4 GB el dashboard va lento. Si tu equipo no llega a ese mínimo, o si estás en Arch, usa la variante con Docker.
- El instalador todo-en-uno oficial (`wazuh-install.sh`) solo soporta Debian/Ubuntu y RHEL/CentOS. En Arch no hay paquete nativo oficial: el camino realista es el despliegue con Docker (contenedores, independiente de la distribución) que se describe en el paso 2. Docker en Arch:
  - `sudo pacman -S docker docker-compose`
  - `sudo systemctl start docker` (Arch no arranca el servicio solo; Debian/Ubuntu sí tras instalar `docker.io`).
- `openssh-server` y `sshpass` para generar los eventos. Instalación:
  - Debian/Ubuntu: `sudo apt install -y openssh-server sshpass`
  - Arch: `sudo pacman -S openssh sshpass` (demonio `sshd`, arráncalo con `sudo systemctl start sshd`).
- Haber hecho el ejercicio 2 ayuda: aquí se repiten sus eventos, pero los ve el SIEM en vez de `journalctl`.
- Tiempo estimado: 90 minutos (unos 20 son la instalación del servidor).

### Pasos

1. Comprueba que el equipo responde en loopback, por donde hablarán el agente y el manager:
   ```bash
   ping -c 1 127.0.0.1
   ```
   - `ping -c 1 127.0.0.1` → envía un paquete ICMP a la interfaz de loopback y espera respuesta (ver ejercicio 2, paso 4); el agente y el manager viven en este mismo equipo y se comunican por `127.0.0.1`.
2. Descarga y ejecuta el asistente de instalación, que instala y configura en la misma máquina el servidor (manager), el indexador y el dashboard (**camino Debian/Ubuntu**; en Arch, salta a la alternativa con Docker más abajo):
   ```bash
   curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
   ```
   - `curl` → cliente HTTP de línea de comandos.
   - `-s` → modo silencioso: sin barra de progreso.
   - `-O` → guarda la descarga en un archivo con el mismo nombre que tiene en el servidor, `wazuh-install.sh`.
   - `https://packages.wazuh.com/4.14/wazuh-install.sh` → el asistente oficial de la rama 4.14; la URL lleva la versión menor (la variante con `4.x` en la ruta responde 403).
   - `&&` → ejecuta lo siguiente solo si la descarga terminó bien.
   - `sudo` → ejecuta el comando como root, necesario para instalar paquetes y servicios.
   - `bash ./wazuh-install.sh` → ejecuta el script descargado con bash.
   - `-a` → (`--all-in-one`) instala los tres componentes centrales en este mismo equipo.

   Al final (unos 10-20 minutos) imprime un resumen como este; copia la contraseña, solo se muestra aquí:
   ```
   INFO: --- Summary ---
   INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP_ADDRESS>
       User: admin
       Password: <ADMIN_PASSWORD>
   INFO: Installation finished.
   ```
   Como el dashboard corre en tu propio equipo, lo abrirás en `https://127.0.0.1` (ver paso 4). Si dice `Your system does not meet the recommended minimum hardware requirements of 4Gb of RAM and 2 CPU cores`, no uses `-i` (ignorar la comprobación), porque el indexador se caerá por falta de memoria: usa la variante con Docker. Si pierdes la contraseña, está en el archivo `wazuh-install-files.tar` que el asistente deja en la carpeta actual: `sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt` (`-x` extrae, `-v` muestra el nombre, `-f` indica el archivo y `-O` vuelca el contenido a la pantalla en vez de escribirlo en disco).

   Alternativa con Docker (**el camino en Arch**, y también si no llegas al mínimo de RAM; necesita Docker y Docker Compose, al menos 8 GB de RAM y 50 GB de disco según la documentación de Wazuh):
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.8
   cd wazuh-docker/single-node/
   sudo docker compose -f generate-indexer-certs.yml run --rm generator
   sudo docker compose up -d
   ```
   - `sysctl -w vm.max_map_count=262144` → sube el número máximo de áreas de memoria mapeada por proceso; el indexador (basado en OpenSearch) falla con el valor por defecto de Linux, 65530. `-w` escribe el valor ahora; no sobrevive a un reinicio.
   - `git clone ... -b v4.14.8` → descarga el repositorio oficial de despliegue en Docker en la etiqueta de esa versión (`-b` elige la rama o etiqueta).
   - `cd wazuh-docker/single-node/` → la variante de un solo nodo, con los tres componentes en contenedores de la misma máquina.
   - `docker compose -f generate-indexer-certs.yml run --rm generator` → ejecuta una vez el contenedor que genera los certificados TLS entre componentes; `-f` elige ese archivo de Compose y `--rm` borra el contenedor al terminar.
   - `docker compose up -d` → crea y arranca los contenedores definidos en `docker-compose.yml`; `-d` los deja en segundo plano.

   Con Docker el usuario es `admin` y la contraseña inicial `SecretPassword` (cámbiala si el equipo no es solo de laboratorio). El resto del ejercicio es igual, salvo que los comandos que se ejecutan sobre el manager en `/var/ossec` se lanzan dentro del contenedor del manager con `sudo docker exec -it single-node-wazuh.manager-1 <comando>` (comprueba el nombre exacto con `sudo docker ps`). En Arch, arranca Docker antes con `sudo systemctl start docker`.
3. (Solo camino Debian/Ubuntu.) Desactiva el repositorio de Wazuh para que un `apt upgrade` no actualice los componentes por separado y los desincronice:
   ```bash
   sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
   sudo apt update
   sudo /var/ossec/bin/wazuh-control info
   ```
   - `sed -i` → edita el archivo en el sitio (`-i`) aplicando la expresión.
   - `"s/^deb /#deb /"` → sustituye (`s`) `deb ` al principio de línea (`^`) por `#deb `: comenta la línea del repositorio.
   - `/etc/apt/sources.list.d/wazuh.list` → archivo con el repositorio que añadió el asistente.
   - `apt update` → relee las listas de paquetes, ya sin la de Wazuh.
   - `/var/ossec/bin/wazuh-control info` → muestra la versión y el tipo de instalación de Wazuh.

   Debe salir `WAZUH_VERSION="v4.14.8"` (o la 4.14.x que se instaló) y `WAZUH_TYPE="server"`. Anota la versión: el agente no puede ser más nuevo que el servidor.
4. Desde el navegador de tu propio equipo abre `https://127.0.0.1` (o `https://localhost`). El aviso de certificado es esperado (es autofirmado): acéptalo como excepción. Entra con `admin` y la contraseña del paso 2. Verás el panel de inicio con 0 agentes.
5. Instala el agente en este mismo equipo. **Camino Debian/Ubuntu**: añade el repositorio oficial de Wazuh:
   ```bash
   sudo apt-get install -y gnupg apt-transport-https curl
   curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg
   echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
   sudo apt-get update
   ```
   - `apt-get install -y gnupg apt-transport-https curl` → instala sin preguntar (`-y`) `gnupg` (para importar la clave), `apt-transport-https` (en Ubuntu 24.04 ya está integrado en apt; se deja por compatibilidad) y `curl`.
   - `curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH` → descarga la clave pública con la que Wazuh firma sus paquetes.
   - `|` → pasa la clave al comando siguiente.
   - `gpg --no-default-keyring` → no usa el llavero personal del usuario.
   - `--keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg` → guarda la clave en ese llavero aparte, en formato de GnuPG, que es donde apt la buscará.
   - `--import` → importa la clave leída de la entrada estándar.
   - `chmod 644` → deja el llavero legible por todos (apt lo lee sin privilegios) y escribible solo por root.
   - `echo "deb [signed-by=...] https://packages.wazuh.com/4.x/apt/ stable main"` → línea de repositorio: `deb` (paquetes binarios), `signed-by` (solo se aceptan paquetes firmados con esa clave), URL, distribución `stable` y componente `main`.
   - `| sudo tee /etc/apt/sources.list.d/wazuh.list` → `tee` escribe la línea en ese archivo como root (una redirección `>` tras `sudo` la haría tu usuario y fallaría por permisos).
   - `apt-get update` → descarga la lista de paquetes, ahora con la de Wazuh.

   En Arch, con el manager en Docker, enrola el agente desde el paquete AUR `wazuh-agent` (`yay -S wazuh-agent`, necesita un ayudante de AUR) o levanta un contenedor de agente de la imagen oficial `wazuh/wazuh-agent`; en ambos casos apúntalo al manager con la variable `WAZUH_MANAGER="127.0.0.1"` del paso 6.
6. Instala el agente de la misma versión que el servidor, indicándole a qué manager conectarse (el propio equipo, `127.0.0.1`), y arráncalo:
   ```bash
   sudo WAZUH_MANAGER="127.0.0.1" WAZUH_AGENT_NAME="agente" apt-get install -y wazuh-agent=4.14.8-1
   sudo systemctl daemon-reload
   sudo systemctl enable --now wazuh-agent
   echo "wazuh-agent hold" | sudo dpkg --set-selections
   ```
   - `WAZUH_MANAGER="127.0.0.1"` → variable de entorno que lee el instalador del paquete para escribir la IP del manager en `/var/ossec/etc/ossec.conf` y registrar (enrolar) el agente con él al arrancar; como el manager está en el mismo equipo, es loopback.
   - `WAZUH_AGENT_NAME="agente"` → nombre con el que aparecerá en el dashboard; si no se da, usa el nombre del equipo.
   - `sudo VAR=valor comando` → pasa esas variables al comando ejecutado como root (con `sudo VAR=valor` y no `VAR=valor sudo`, porque `sudo` limpia el entorno).
   - `apt-get install -y wazuh-agent=4.14.8-1` → instala el paquete del agente en esa versión exacta; cambia `4.14.8` por la que anotaste en el paso 3. El repositorio `4.x` ofrece siempre la última, que podría ser más nueva que tu servidor.
   - `systemctl daemon-reload` → hace que systemd relea las unidades de servicio, incluida la nueva `wazuh-agent.service`.
   - `systemctl enable --now wazuh-agent` → activa el servicio en el arranque (`enable`) y lo inicia ya (`--now`).
   - `echo "wazuh-agent hold" | sudo dpkg --set-selections` → marca el paquete como retenido (`hold`): `apt upgrade` no lo actualizará por su cuenta.

   Comprueba la conexión (en terminales distintas del mismo equipo):
   ```bash
   sudo grep -E "Connected to the server|ERROR" /var/ossec/logs/ossec.log | tail -3     # log del agente
   sudo /var/ossec/bin/agent_control -l                                                # consulta al manager
   ```
   - `grep -E "Connected to the server|ERROR"` → busca en el log del agente la conexión al manager o cualquier error; `-E` permite el `|` como "o".
   - `/var/ossec/logs/ossec.log` → log propio de Wazuh (el del agente y el del manager comparten ruta; en el camino Docker, el del manager está dentro del contenedor).
   - `| tail -3` → muestra solo las 3 últimas líneas que coinciden.
   - `agent_control -l` → en el manager, lista los agentes registrados y su estado.
   - `# log del agente`, `# consulta al manager` → comentarios: indican qué mira cada línea.

   El log del agente debe mostrar una línea con `Connected to the server`, la IP `127.0.0.1` y el puerto `1514/tcp`; `agent_control -l` debe listar el agente `001` con nombre `agente` y estado `Active`. Si no conecta, revisa que nada filtre los puertos 1514/tcp (eventos) y 1515/tcp (registro) del manager en loopback. En el dashboard, el contador de agentes pasa a 1.
7. Comprueba de dónde lee logs el agente:
   ```bash
   sudo grep -A2 "<localfile>" /var/ossec/etc/ossec.conf
   ```
   - `grep -A2 "<localfile>"` → muestra cada bloque `<localfile>` y las 2 líneas siguientes (`-A2`, after); cada bloque es una fuente de logs del agente.
   - `/var/ossec/etc/ossec.conf` → configuración del agente.

   Verás un bloque con `<location>journald</location>` (el journal de systemd) o con `<location>/var/log/auth.log</location>`, según lo que detectó el paquete; con cualquiera de los dos llegan al manager los mensajes de `sshd`, `sudo`, `useradd` y `usermod` del ejercicio 2.
8. Activa en el agente la vigilancia de integridad de archivos (FIM, el módulo `syscheck`) en tiempo real sobre una carpeta de prueba. Crea primero la carpeta, porque el módulo ignora las que no existen al arrancar:
   ```bash
   sudo mkdir -p /srv/lab-fim
   sudo nano /var/ossec/etc/ossec.conf
   ```
   - `mkdir -p /srv/lab-fim` → crea la carpeta vigilada; `-p` no da error si ya existe.
   - `nano /var/ossec/etc/ossec.conf` → abre la configuración del agente en el editor `nano`.

   Dentro del bloque `<syscheck>` que ya existe, justo debajo de la línea `<syscheck>`, añade esta línea:
   ```xml
   <directories check_all="yes" report_changes="yes" realtime="yes">/srv/lab-fim</directories>
   ```
   - `<directories>...</directories>` → carpeta que vigila FIM, con todo su contenido.
   - `check_all="yes"` → registra todos los atributos: hashes MD5, SHA-1 y SHA-256, tamaño, dueño, grupo, permisos y fecha de modificación.
   - `report_changes="yes"` → en los archivos de texto guarda una copia y adjunta a la alerta el `diff` de lo que cambió.
   - `realtime="yes"` → vigilancia continua con inotify del kernel: avisa en segundos en vez de esperar al escaneo periódico (cada 12 horas por defecto). Solo vale para directorios.

   Guarda (`Ctrl+O`, Enter, `Ctrl+X`) y reinicia el agente:
   ```bash
   sudo systemctl restart wazuh-agent
   sudo grep -E "\(6009\)|\(6012\)" /var/ossec/logs/ossec.log | tail -2
   ```
   - `systemctl restart wazuh-agent` → reinicia el servicio para que lea la nueva configuración.
   - `grep -E "\(6009\)|\(6012\)"` → busca los mensajes con código 6009 y 6012; `\(` y `\)` son paréntesis literales.
   - `tail -2` → (ver paso 6).

   Espera hasta ver `(6009): File integrity monitoring scan ended.` (terminó el escaneo inicial, la línea base) y `(6012): Real-time file integrity monitoring started.`. Lo que pase antes de la línea base no genera alerta.
9. Anota la hora y genera, en este mismo equipo, la actividad sospechosa:
   ```bash
   date '+%H:%M:%S'
   for i in $(seq 1 10); do
     sshpass -p "ClaveMala$i" ssh -o StrictHostKeyChecking=no -o PubkeyAuthentication=no -o NumberOfPasswordPrompts=1 noexiste@127.0.0.1 true
   done
   sshpass -p "ClaveMala" ssh -o StrictHostKeyChecking=no -o PubkeyAuthentication=no -o NumberOfPasswordPrompts=1 "$USER"@127.0.0.1 true
   sudo -k
   sudo ls /root
   sudo useradd -m -s /bin/bash labwazuh
   sudo usermod -aG sudo labwazuh
   echo "puerto=8080" | sudo tee /srv/lab-fim/config.txt
   echo "puerto=4444" | sudo tee -a /srv/lab-fim/config.txt
   sudo rm /srv/lab-fim/config.txt
   ```
   - `date '+%H:%M:%S'` → muestra la hora actual (ver ejercicio 2, paso 3).
   - `for i in $(seq 1 10); do sshpass ... done` → diez intentos SSH fallidos contra una cuenta que no existe; `$(seq 1 10)` se sustituye por los números del 1 al 10, y cada opción de `sshpass` y `ssh` está explicada en el ejercicio 2, paso 4. Cada intento deja al menos la línea `Invalid user noexiste`, así que se superan los 8 eventos en 2 minutos que cuenta la regla de fuerza bruta.
   - `"$USER"@127.0.0.1` → un intento más con contraseña errónea contra tu propia cuenta, que sí existe; `$USER` es tu nombre de usuario.
   - `sudo -k` → invalida la contraseña de `sudo` en caché, para que el siguiente `sudo` la pida y quede registrado como uso nuevo.
   - `sudo ls /root` → un uso de `sudo` que ejecuta un comando como root: lista la carpeta personal de root.
   - `useradd -m -s /bin/bash labwazuh` → crea la cuenta `labwazuh` con su carpeta personal (`-m`) y bash como shell (`-s`); ver ejercicio 2, paso 5.
   - `usermod -aG sudo labwazuh` → la añade (`-a`) al grupo suplementario (`-G`) `sudo`: le da permisos de administrador.
   - `echo "puerto=8080" | sudo tee /srv/lab-fim/config.txt` → crea el archivo vigilado con ese contenido (FIM: archivo añadido).
   - `sudo tee -a` → añade (`-a`) una línea al final sin borrar lo que había (FIM: archivo modificado).
   - `rm /srv/lab-fim/config.txt` → borra el archivo (FIM: archivo eliminado).
10. En el dashboard abre el menú (☰) → Threat intelligence → Threat Hunting, pestaña Events, rango de tiempo "Last 1 hour". En la barra de búsqueda escribe una consulta cada vez (lenguaje DQL: `campo:valor`, con `AND`, `OR` y paréntesis):
    ```
    agent.name:agente AND rule.groups:authentication_failed
    rule.id:(5710 OR 5760 OR 5712)
    rule.id:(5402 OR 5403)
    rule.id:(5901 OR 5902)
    rule.id:(550 OR 553 OR 554)
    ```
    - `agent.name:agente` → solo alertas del agente con ese nombre.
    - `rule.groups:authentication_failed` → alertas cuya regla pertenece al grupo de autenticación fallida, sea cual sea el servicio (sshd, PAM, etc.).
    - `rule.id:(5710 OR 5760 OR 5712)` → reglas de sshd: 5710 "Attempt to login using a non-existent user" (nivel 5), 5760 "authentication failed" con usuario real (nivel 5) y 5712 "brute force trying to get access to the system. Non existent user." (nivel 10, salta con 8 eventos de la 5710 en 120 segundos).
    - `rule.id:(5402 OR 5403)` → 5402 "Successful sudo to ROOT executed" (nivel 3) y 5403 "First time user executed sudo" (nivel 4, solo la primera vez).
    - `rule.id:(5901 OR 5902)` → 5901 "New group added to the system" y 5902 "New user added to the system" (nivel 8), del `useradd`.
    - `rule.id:(550 OR 553 OR 554)` → FIM: 554 "File added to the system", 550 "Integrity checksum changed" y 553 "File deleted".

    Abre la alerta 550 (flecha a la izquierda de la fila) y busca los campos `syscheck.path` (`/srv/lab-fim/config.txt`), `syscheck.sha256_before` y `syscheck.sha256_after` (hashes antes y después) y `syscheck.diff` (la línea `puerto=4444` añadida). Lo mismo se ve en el menú → Endpoint security → File Integrity Monitoring.

    Ahora busca el alta en el grupo `sudo`:
    ```
    agent.name:agente AND full_log:*usermod*
    ```
    - `full_log:*usermod*` → alertas cuyo log original contenga `usermod`; `*` es comodín.

    No sale nada: Wazuh recibió la línea `usermod[...]: add 'labwazuh' to group 'sudo'`, pero ninguna regla de fábrica la convierte en alerta (sí la hay para la misma acción hecha con `gpasswd -a`, la 2961, de nivel 5). Que un usuario pase a administrador sin alerta es un hueco de detección; lo cierras en el paso 12.
11. Mira el mapeo a MITRE ATT&CK. En la alerta 5712 los campos `rule.mitre.id`, `rule.mitre.tactic` y `rule.mitre.technique` dicen `T1110`, `Credential Access` y `Brute Force`; en la 5710, `T1110.001` (Password Guessing) y `T1021.004` (SSH); en la 5402, `T1548.003` (Sudo and Sudo Caching); en la 5902, `T1136` (Create Account). Después abre el menú → Threat intelligence → MITRE ATT&CK: el panel agrupa las alertas del agente por táctica y técnica, y es la misma matriz de la nota del tema 12.
12. Escribe la regla que falta. En el manager, abre el archivo de reglas locales (el único que no se sobrescribe al actualizar):
    ```bash
    sudo nano /var/ossec/etc/rules/local_rules.xml
    ```
    - `/var/ossec/etc/rules/local_rules.xml` → archivo de reglas propias del manager; trae una regla de ejemplo (id 100001) que puedes dejar. En el camino Docker, edítalo dentro del contenedor del manager (`sudo docker exec -it single-node-wazuh.manager-1 vi /var/ossec/etc/rules/local_rules.xml`) o monta el volumen correspondiente.

    Añade al final del archivo este bloque completo:
    ```xml
    <group name="local,syslog,usermod,">
      <rule id="100100" level="10">
        <program_name>^usermod$</program_name>
        <match>to group 'sudo'</match>
        <description>Usuario añadido al grupo sudo con usermod</description>
        <mitre>
          <id>T1098</id>
        </mitre>
        <group>account_changed,</group>
      </rule>
    </group>
    ```
    - `<group name="local,syslog,usermod,">` → grupos que heredan las reglas de dentro (aparecerán en `rule.groups`); la lista termina en coma.
    - `<rule id="100100" level="10">` → regla nueva; los ids de 100000 a 120000 están reservados para reglas propias. El nivel va de 0 a 15; 10 es alto, a la altura de la 5712.
    - `<program_name>^usermod$</program_name>` → solo si el programa que escribió el log es exactamente `usermod` (`^` inicio, `$` final).
    - `<match>to group 'sudo'</match>` → y si el mensaje contiene ese texto. No coincide con la segunda línea que escribe `usermod`, `add 'labwazuh' to shadow group 'sudo'`, así que sale una alerta por alta y no dos.
    - `<description>` → texto que verá el analista en la alerta.
    - `<mitre><id>T1098</id></mitre>` → técnica ATT&CK asociada: T1098, Account Manipulation; Wazuh rellena táctica y nombre.
    - `<group>account_changed,</group>` → grupo adicional solo de esta regla, para buscarla con `rule.groups:account_changed`.

    Pruébala sin reiniciar nada con `wazuh-logtest`, que carga las reglas del disco en una sesión aparte:
    ```bash
    sudo /var/ossec/bin/wazuh-logtest
    ```
    - `/var/ossec/bin/wazuh-logtest` → herramienta del manager que pasa una línea de log por los decodificadores y las reglas y dice qué alerta generaría. También está en el dashboard: menú → Server management → Ruleset Test.

    Cuando diga `Type one log per line`, pega esta línea y pulsa Enter (sal con `Ctrl+C`):
    ```
    Oct  8 10:42:31 agente usermod[2417]: add 'labwazuh' to group 'sudo'
    ```
    Esperado (resumido):
    ```
    **Phase 1: Completed pre-decoding.
            full event: 'Oct  8 10:42:31 agente usermod[2417]: add 'labwazuh' to group 'sudo''
            timestamp: 'Oct  8 10:42:31'
            hostname: 'agente'
            program_name: 'usermod'

    **Phase 2: Completed decoding.
            No decoder matched.

    **Phase 3: Completed filtering (rules).
            id: '100100'
            level: '10'
            description: 'Usuario añadido al grupo sudo con usermod'
            groups: '["local","syslog","usermod","account_changed"]'
            mitre.id: '["T1098"]'
            mitre.technique: '["Account Manipulation"]'
    **Alert to be generated.
    ```
    La fase 1 separa la cabecera syslog (hora, equipo, programa); la fase 2 no encuentra decodificador para `usermod` (`No decoder matched.`), y no importa, porque la regla solo usa el `program_name` de la fase 1 y el texto; la fase 3 dice qué regla salta. Si sale un error de XML, revisa que cada etiqueta esté cerrada. Para que el motor real la use, reinicia el manager:
    ```bash
    sudo systemctl restart wazuh-manager
    ```
    - `systemctl restart wazuh-manager` → reinicia el servidor de Wazuh, que relee las reglas; el agente reconecta solo. En el camino Docker, `sudo docker restart single-node-wazuh.manager-1`.
13. Repite el alta y búscala:
    ```bash
    sudo gpasswd -d labwazuh sudo
    sudo usermod -aG sudo labwazuh
    ```
    - `gpasswd -d labwazuh sudo` → quita (`-d`, delete) a `labwazuh` del grupo `sudo`, para poder repetir el alta.
    - `usermod -aG sudo labwazuh` → (ver paso 9).

    En Threat Hunting busca `rule.id:100100`: debe aparecer una alerta de nivel 10 con `rule.description` "Usuario añadido al grupo sudo con usermod" y, en el panel MITRE ATT&CK, la técnica T1098 en las tácticas Persistence y Privilege Escalation.
14. Haz el triaje de una alerta como en un SOC. Elige la 5712 (fuerza bruta) y escribe en tu equipo `triaje-5712.txt` con este formato, rellenando cada campo con los datos de la alerta:
    ```
    Alerta:      5712 - sshd: brute force trying to get access to the system. Non existent user. (nivel 10)
    Agente:      agente (127.0.0.1)       Hora: <timestamp de la alerta>
    Evidencia:   <n> eventos 5710 en <segundos> s, usuario "noexiste", origen 127.0.0.1 (data.srcip)
    Contexto:    la IP de origen es el propio equipo; la cuenta no existe; hubo 1 fallo (5760) contra una cuenta real y ningún 5715 (login correcto) después
    Veredicto:   verdadero positivo benigno: la actividad ocurrió tal como la describe la regla, pero es la prueba autorizada del paso 9
    Acción:      ninguna en el laboratorio; en producción, bloquear el origen, comprobar que no hubo login correcto desde él y revisar si PasswordAuthentication debe seguir activo
    ```
    Las cuatro categorías que se usan al cerrar una alerta: verdadero positivo (pasó y es malicioso), verdadero positivo benigno (pasó, pero es autorizado o esperado), falso positivo (la regla se disparó sin que pasara lo que describe, por ejemplo por un patrón demasiado amplio) y falso negativo (pasó y no hubo alerta, como el `usermod` del paso 10 antes de tu regla). Para comprobar el "ningún login correcto después", busca `agent.name:agente AND rule.id:5715` en la misma ventana de tiempo.

### Resultado esperado

Un Wazuh 4.14 con un agente `Active`; en Threat Hunting, las alertas de autenticación fallida (5710, 5760, 5712), `sudo` (5402), cuenta nueva (5901, 5902) y FIM (554, 550 con su `diff`, 553), cada una con su técnica ATT&CK; una regla local 100100 que cierra el hueco del `usermod`, probada con `wazuh-logtest` y vista disparar en el dashboard; y `triaje-5712.txt` con el veredicto razonado.

### Comprueba que lo lograste

- ¿Qué hace el agente y qué hace el manager? El agente lee los logs y vigila los archivos del equipo y lo envía cifrado por el puerto 1514; el manager decodifica, aplica las reglas y decide qué es alerta; el indexador la guarda y el dashboard la muestra. Aquí los dos corren en el mismo equipo y hablan por loopback.
- ¿Por qué la 5712 salta y la 5710 no tiene nivel 10? La 5710 es un evento suelto; la 5712 es una regla de correlación (`frequency="8"`, `timeframe="120"`) que solo salta cuando se acumulan 8 eventos 5710 en 2 minutos. Es la correlación de la nota de SIEM.
- ¿Por qué el `usermod` no generaba alerta y el `useradd` sí? Porque hay reglas de fábrica para `useradd` (5902) y para `gpasswd` (2961), pero ninguna para el mensaje de `usermod`. Un SIEM solo alerta de lo que alguna regla describe.
- ¿Por qué FIM con `realtime="yes"` y no con el escaneo por defecto? Porque el escaneo periódico es cada 12 horas: un archivo creado y borrado entre dos escaneos no dejaría rastro.
- ¿Por qué se fija la versión del agente? Porque Wazuh solo garantiza compatibilidad si el manager es igual o más nuevo que el agente.

### Limpieza

Deshaz lo del agente:
```bash
sudo userdel -r labwazuh
sudo rm -rf /srv/lab-fim
sudo apt-get purge -y wazuh-agent
sudo rm /etc/apt/sources.list.d/wazuh.list /usr/share/keyrings/wazuh.gpg
sudo apt-get update
```
- `userdel -r labwazuh` → borra la cuenta, su grupo y su carpeta personal (`-r`); deja de estar en `sudo`.
- `rm -rf /srv/lab-fim` → borra la carpeta vigilada con todo su contenido (`-r`, recursivo) sin pedir confirmación ni fallar si no existe (`-f`).
- `apt-get purge -y wazuh-agent` → desinstala el agente y su configuración.
- `rm ...wazuh.list ...wazuh.gpg` → quita el repositorio y la clave del paso 5.
- `apt-get update` → relee las listas de paquetes sin el repositorio de Wazuh.

Deshaz el servidor:
```bash
sudo /var/ossec/bin/manage_agents -r 001
sudo bash ./wazuh-install.sh -u
```
- `manage_agents -r 001` → elimina del manager el registro del agente con id `001`.
- `bash ./wazuh-install.sh -u` → (`--uninstall`) desinstala el manager, el indexador y el dashboard; ejecútalo en la carpeta donde descargaste el asistente.

Con Docker, en `wazuh-docker/single-node/`: `sudo docker compose down -v` (`down` para y borra los contenedores; `-v` borra también los volúmenes con los datos). En Arch, para Docker al terminar con `sudo systemctl stop docker` si lo arrancaste solo para esto.
