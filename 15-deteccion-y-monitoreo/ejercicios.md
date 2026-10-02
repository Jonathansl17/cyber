# Ejercicios: Detección y monitoreo

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría está en [README.md](README.md).

## Ejercicio 1: Suricata como IDS y luego como IPS

Nodo: [Basics of IDS and IPS](README.md#basics-of-ids-and-ips), [NIPS](README.md#nips) y [Un IPS solo puede bloquear lo que lo atraviesa](README.md#un-ips-solo-puede-bloquear-lo-que-lo-atraviesa).

Objetivo: cargar en Suricata las dos reglas de laboratorio de la nota (ping y cadena `/hola-ids` en el URI), ver sus alertas en `fast.log` y `eve.json`, y después convertirlas en `drop` con Suricata inline por NFQUEUE para comprobar que el ping y la petición dejan de llegar y que `eve.json` registra `"action":"blocked"`.

Necesitas:

- VirtualBox con dos VMs Ubuntu Server 24.04 (o Debian 12), 2 GB de RAM cada una. Las dos con un adaptador "Red solo-anfitrión" (`vboxnet0`, red `192.168.56.0/24`). El adaptador NAT solo hace falta para instalar paquetes; desactívalo (Configuración → Red → Adaptador 1 → desmarcar "Habilitar adaptador de red") antes de generar tráfico.
- VM `sensor` con IP `192.168.56.20`: Suricata, `jq` y Python 3. Instalación: `sudo apt update && sudo apt install -y suricata jq python3`.
- VM `cliente` con IP `192.168.56.10`: `curl` e `iputils-ping` (vienen de serie).
- Tiempo estimado: 60 minutos.

### Pasos

1. Fija las IPs del laboratorio. En cada VM comprueba el nombre de la interfaz solo-anfitrión (normalmente `enp0s8`):
   ```bash
   ip -br link
   ```
   En `sensor`, crea `/etc/netplan/60-lab.yaml` con este contenido completo (en `cliente` cambia la dirección por `192.168.56.10/24`):
   ```yaml
   network:
     version: 2
     ethernets:
       enp0s8:
         addresses: [192.168.56.20/24]
   ```
   Aplícalo y verifica:
   ```bash
   sudo chmod 600 /etc/netplan/60-lab.yaml
   sudo netplan apply
   ip -br addr show enp0s8
   ```
   Debe verse `enp0s8  UP  192.168.56.20/24`. Desde `cliente`, `ping -c 1 192.168.56.20` tiene que responder.
2. En `sensor`, detén el servicio de Suricata que el paquete arranca solo, para lanzarlo a mano con tus reglas:
   ```bash
   sudo systemctl stop suricata
   suricata --build-info | grep -E 'Version|NFQueue'
   ```
   `NFQueue support: yes` confirma que este binario puede funcionar inline. Si dice `no`, el modo IPS del paso 8 no funcionará con ese paquete.
3. Crea `/etc/suricata/rules/local.rules` con las dos reglas de la nota. Este es el archivo completo; la segunda usa el buffer `http.uri;` de Suricata en lugar del `http_uri;` de Snort:
   ```
   alert icmp any any -> $HOME_NET any (msg:"LAB ICMP Echo Request hacia la red interna"; itype:8; sid:1000001; rev:1;)
   alert http any any -> $HOME_NET any (msg:"LAB cadena de prueba en el URI"; http.uri; content:"/hola-ids"; nocase; sid:1000002; rev:1;)
   ```
4. La parte de `/etc/suricata/suricata.yaml` que importa es la variable `HOME_NET`, que por defecto incluye todas las redes privadas:
   ```yaml
   vars:
     address-groups:
       HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"
   ```
   No la edites: se sobrescribe al lanzar Suricata con `--set`. Valida la configuración y las reglas antes de arrancar:
   ```bash
   sudo suricata -T -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/local.rules \
     --set "vars.address-groups.HOME_NET=[192.168.56.0/24]" -v
   ```
   Al final debe aparecer `Configuration provided was successfully loaded. Exiting.` y una línea que dice `2 rules successfully loaded`. `-S` carga exclusivamente ese archivo e ignora las reglas del yaml.
5. Arranca un servidor web mínimo en `sensor` (en una segunda terminal o sesión SSH) para que el `curl` tenga a quién pedir:
   ```bash
   sudo python3 -m http.server 80
   ```
6. Lanza Suricata como IDS, escuchando una copia del tráfico de la interfaz. `-k none` desactiva la comprobación de checksums, que en VirtualBox suele fallar por el offloading de la tarjeta virtual:
   ```bash
   sudo rm -f /var/log/suricata/fast.log /var/log/suricata/eve.json
   sudo suricata -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/local.rules \
     --set "vars.address-groups.HOME_NET=[192.168.56.0/24]" -k none -i enp0s8
   ```
   Espera a ver `Engine started.` (en versiones antiguas, `all ... packet processing threads ... started`).
7. Desde `cliente`, genera el tráfico y luego mira las alertas en `sensor`:
   ```bash
   ping -c 1 192.168.56.20
   curl http://192.168.56.20/hola-ids
   ```
   El `curl` devuelve un 404 del servidor Python: es normal, la regla mira la petición, no la respuesta. En `sensor`:
   ```bash
   sudo cat /var/log/suricata/fast.log
   sudo jq -c 'select(.event_type=="alert") | {src_ip, dest_ip, sid: .alert.signature_id, action: .alert.action}' /var/log/suricata/eve.json
   ```
   Salida esperada (las horas cambian):
   ```
   10/02/2026-14:03:11.120344  [**] [1:1000001:1] LAB ICMP Echo Request hacia la red interna [**] [Classification: (null)] [Priority: 3] {ICMP} 192.168.56.10:8 -> 192.168.56.20:0
   10/02/2026-14:03:15.402981  [**] [1:1000002:1] LAB cadena de prueba en el URI [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.56.10:51544 -> 192.168.56.20:80
   {"src_ip":"192.168.56.10","dest_ip":"192.168.56.20","sid":1000001,"action":"allowed"}
   {"src_ip":"192.168.56.10","dest_ip":"192.168.56.20","sid":1000002,"action":"allowed"}
   ```
   `"action":"allowed"`: Suricata vio y avisó, pero el ping tuvo respuesta y el servidor recibió la petición. Eso es un IDS. Detén Suricata con `Ctrl+C`.
8. Pasa a IPS. Cambia `alert` por `drop` en las dos reglas y sube la revisión, como se hace al editar una regla:
   ```bash
   sudo sed -i -e 's/^alert /drop /' -e 's/rev:1;/rev:2;/' /etc/suricata/rules/local.rules
   cat /etc/suricata/rules/local.rules
   ```
   Ahora el archivo completo queda así:
   ```
   drop icmp any any -> $HOME_NET any (msg:"LAB ICMP Echo Request hacia la red interna"; itype:8; sid:1000001; rev:2;)
   drop http any any -> $HOME_NET any (msg:"LAB cadena de prueba en el URI"; http.uri; content:"/hola-ids"; nocase; sid:1000002; rev:2;)
   ```
9. Pon a Suricata en el camino del tráfico. Con `-i` solo recibe una copia; para poder descartar, el kernel debe entregarle cada paquete y esperar su veredicto. Crea `/etc/suricata/lab-nfqueue.nft` con este contenido completo, que manda a la cola 0 solo el tráfico con `cliente` (tu SSH desde el anfitrión no pasa por Suricata):
   ```
   table inet suricata_lab {
       chain input {
           type filter hook input priority 0; policy accept;
           ip saddr 192.168.56.10 queue num 0 bypass
       }
       chain output {
           type filter hook output priority 0; policy accept;
           ip daddr 192.168.56.10 queue num 0 bypass
       }
   }
   ```
   Se encolan las dos direcciones para que Suricata reconstruya la sesión TCP completa. `bypass` hace que, si Suricata no está escuchando la cola, el kernel acepte el paquete: es el fail-open de la nota. Sin `bypass` sería fail-closed y, con Suricata caído, `cliente` no podría hablar con `sensor`. Carga las reglas:
   ```bash
   sudo nft -f /etc/suricata/lab-nfqueue.nft
   sudo nft list table inet suricata_lab
   ```
   Si tu sistema usa `iptables` en vez de `nft`, el equivalente es:
   ```bash
   sudo iptables -I INPUT -s 192.168.56.10 -j NFQUEUE --queue-num 0 --queue-bypass
   sudo iptables -I OUTPUT -d 192.168.56.10 -j NFQUEUE --queue-num 0 --queue-bypass
   ```
10. Arranca Suricata inline sobre la cola 0 (`-q 0` en lugar de `-i`):
    ```bash
    sudo rm -f /var/log/suricata/fast.log /var/log/suricata/eve.json
    sudo suricata -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/local.rules \
      --set "vars.address-groups.HOME_NET=[192.168.56.0/24]" -k none -q 0
    ```
11. Repite el tráfico desde `cliente`. `-m 5` corta el `curl` a los 5 segundos, porque la petición nunca obtendrá respuesta:
    ```bash
    ping -c 3 192.168.56.20
    curl -m 5 http://192.168.56.20/hola-ids
    curl -m 5 http://192.168.56.20/
    ```
    Esperado: `3 packets transmitted, 0 received, 100% packet loss`; el primer `curl` termina con `curl: (28) Operation timed out`; el segundo, sin la cadena, recibe el listado de directorio del servidor Python. En la terminal del servidor web solo aparece la petición a `/`, nunca la de `/hola-ids`.
12. Confirma el bloqueo en `eve.json`:
    ```bash
    sudo jq -c 'select(.event_type=="alert") | {sid: .alert.signature_id, action: .alert.action}' /var/log/suricata/eve.json
    ```
    ```
    {"sid":1000001,"action":"blocked"}
    {"sid":1000001,"action":"blocked"}
    {"sid":1000001,"action":"blocked"}
    {"sid":1000002,"action":"blocked"}
    ```
    Puede haber más de una alerta del sid 1000002 si `curl` reenvía el segmento TCP descartado.
13. Comprueba el fail-open: deja las reglas `nft` cargadas, detén Suricata con `Ctrl+C` y repite `ping -c 3 192.168.56.20` desde `cliente`. Ahora responde, porque con `bypass` y nadie escuchando la cola el kernel acepta. Ningún paquete se inspecciona.

### Resultado esperado

Las mismas dos reglas producen `"action":"allowed"` cuando Suricata escucha por `-i` y `"action":"blocked"` cuando recibe el tráfico por NFQUEUE: la capacidad de bloquear salió de la posición, no de la regla. También viste que, con `bypass`, un IPS caído deja pasar todo.

### Comprueba que lo lograste

- ¿Qué significa `[1:1000002:1]` en `fast.log`? Generador 1, sid 1000002, revisión 1.
- En modo IDS, ¿recibió el servidor la petición a `/hola-ids`? Sí: aparece en el log de `python3 -m http.server` con un 404.
- ¿Qué cambió entre el paso 7 y el 11 además de `alert` por `drop`? Que Suricata pasó de leer una copia (`-i`) a estar inline (`-q 0` con la regla `queue` de nftables). Con `drop` y `-i`, Suricata solo habría alertado.
- Si quitas `bypass` y Suricata se cae, ¿qué le pasa a `cliente`? Pierde toda la conectividad con `sensor` (fail-closed).

### Limpieza

```bash
sudo nft delete table inet suricata_lab
sudo rm /etc/suricata/lab-nfqueue.nft /etc/suricata/rules/local.rules
sudo systemctl start suricata
```

Si usaste `iptables`, borra sus dos reglas repitiendo los comandos del paso 9 con `-D` en lugar de `-I`. Detén el servidor web con `Ctrl+C`. Para desinstalar: `sudo apt purge -y suricata`.

## Ejercicio 2: Rastrea un ataque simulado en los Event Logs

Nodo: [Event Logs](README.md#event-logs).

Objetivo: activar en Windows la auditoría que la nota da por necesaria (inicio de sesión, creación de procesos con línea de comandos, gestión de cuentas y grupos), instalar Sysmon, reproducir la cadena "5 logins fallidos, cuenta nueva, cuenta añadida a Administradores" y encontrar con `Get-WinEvent` los eventos 4625, 4720, 4732 y 4688, más el evento 1 de Sysmon del mismo comando.

Necesitas:

- Una VM con Windows 11 Enterprise o Windows Server 2022 de evaluación (gratis 90 días en el [Evaluation Center de Microsoft](https://www.microsoft.com/en-us/evalcenter)), sin cuenta Microsoft, solo con cuentas locales. Red NAT para descargar Sysmon; no hace falta otra máquina.
- PowerShell abierto como administrador (clic derecho en Inicio → Terminal (Administrador)).
- Tiempo estimado: 45 minutos.

### Pasos

1. Guarda la política de auditoría actual para poder restaurarla al final:
   ```powershell
   auditpol /backup /file:C:\audit-antes.csv
   ```
2. Activa las cuatro subcategorías. Se usan los GUID en lugar de los nombres porque los nombres cambian con el idioma de Windows (`Process Creation` en inglés, `Creación del proceso` en español):
   ```powershell
   auditpol /set /subcategory:"{0CCE9215-69AE-11D9-BED3-505054503030}" /success:enable /failure:enable
   auditpol /set /subcategory:"{0CCE922B-69AE-11D9-BED3-505054503030}" /success:enable
   auditpol /set /subcategory:"{0CCE9235-69AE-11D9-BED3-505054503030}" /success:enable
   auditpol /set /subcategory:"{0CCE9237-69AE-11D9-BED3-505054503030}" /success:enable
   auditpol /list /subcategory:* /v | Select-String "0CCE9215|0CCE922B|0CCE9235|0CCE9237"
   ```
   La última línea confirma a qué subcategoría corresponde cada GUID: Logon, Process Creation, User Account Management y Security Group Management. Cada `/set` debe responder `The command was successfully executed.`
3. Activa la línea de comandos en el evento 4688 (es la directiva "Include command line in process creation events"):
   ```powershell
   reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
   auditpol /get /subcategory:"{0CCE922B-69AE-11D9-BED3-505054503030}"
   ```
   La segunda orden debe mostrar `Process Creation    Success`.
4. Descarga e instala Sysmon con su configuración por defecto, que ya registra la creación de procesos (evento 1):
   ```powershell
   Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile $env:TEMP\Sysmon.zip
   Expand-Archive -Path $env:TEMP\Sysmon.zip -DestinationPath C:\Tools\Sysmon -Force
   C:\Tools\Sysmon\Sysmon64.exe -accepteula -i
   Get-Service Sysmon64
   ```
   Debe terminar con `Sysmon64 started.` y el servicio en estado `Running`.
5. Simula la fuerza bruta: cinco autenticaciones de red fallidas contra el propio equipo con un usuario que no existe (así no bloqueas ninguna cuenta):
   ```powershell
   1..5 | ForEach-Object { net use \\127.0.0.1\IPC$ /user:noexiste "ClaveMala$_" 2>$null }
   ```
   Cada intento responde con `System error 1326` (usuario o contraseña incorrectos). Si más adelante no encuentras los cinco 4625, hazlo a mano: `Win+L`, elige "Otro usuario" y escribe cinco veces una contraseña errónea.
6. Crea la cuenta y añádela a Administradores. El `*` hace que `net user` pida la contraseña por teclado en lugar de escribirla en la línea de comandos (y por tanto en el 4688). El grupo se indica por su SID, `S-1-5-32-544`, que es el mismo en cualquier idioma:
   ```powershell
   net user labuser * /add
   Add-LocalGroupMember -SID 'S-1-5-32-544' -Member labuser
   ```
7. Define una función que lee cualquier campo del evento por su nombre, sin depender de su posición en `Properties`:
   ```powershell
   function Get-Campo($Evento, $Nombre) {
       (([xml]$Evento.ToXml()).Event.EventData.Data | Where-Object Name -eq $Nombre).'#text'
   }
   ```
8. Busca los logins fallidos de los últimos 30 minutos:
   ```powershell
   Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddMinutes(-30)} |
       Select-Object TimeCreated,
           @{n='Usuario';  e={Get-Campo $_ 'TargetUserName'}},
           @{n='Tipo';     e={Get-Campo $_ 'LogonType'}},
           @{n='SubStatus';e={Get-Campo $_ 'SubStatus'}},
           @{n='IP';       e={Get-Campo $_ 'IpAddress'}}
   ```
   Esperado: cinco filas con `noexiste`, tipo `3` (red, como una carpeta compartida SMB), `0xc0000064` (el usuario no existe) e IP `127.0.0.1`. Si usaste `Win+L`, el tipo será `2` y el SubStatus `0xc000006a` (contraseña incorrecta).
9. Busca la cuenta creada y su alta en Administradores:
   ```powershell
   Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4720,4732; StartTime=(Get-Date).AddMinutes(-30)} |
       Select-Object TimeCreated, Id,
           @{n='Cuenta'; e={ if ($_.Id -eq 4720) { Get-Campo $_ 'TargetUserName' } else { Get-Campo $_ 'MemberSid' } }},
           @{n='Grupo';  e={ if ($_.Id -eq 4732) { Get-Campo $_ 'TargetUserName' } }},
           @{n='Autor';  e={Get-Campo $_ 'SubjectUserName'}}
   ```
   Esperado: un 4720 con `labuser` y, segundos después, un 4732 con el SID de `labuser` y el grupo `Administrators` (o `Administradores`). El campo `Autor` es tu usuario: el que lo hizo. En equipos sin dominio el 4732 trae el SID del miembro, no el nombre; compruébalo con `(Get-LocalUser labuser).SID.Value`.
10. Busca los procesos que hicieron esos cambios:
    ```powershell
    Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688; StartTime=(Get-Date).AddMinutes(-30)} |
        Where-Object { (Get-Campo $_ 'NewProcessName') -match '\\net1?\.exe$' } |
        Select-Object TimeCreated,
            @{n='Proceso'; e={Get-Campo $_ 'NewProcessName'}},
            @{n='Padre';   e={Get-Campo $_ 'ParentProcessName'}},
            @{n='Comando'; e={Get-Campo $_ 'CommandLine'}}
    ```
    Esperado: `net.exe` con padre `powershell.exe` (o `pwsh.exe`) y comando `net user labuser * /add`, seguido de `net1.exe` con padre `net.exe`; y las cinco ejecuciones de `net use`. Si `Comando` sale vacío, el paso 3 no se aplicó antes de crear el proceso.
11. Compara con el evento 1 de Sysmon del mismo comando:
    ```powershell
    Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1; StartTime=(Get-Date).AddMinutes(-30)} |
        Where-Object { (Get-Campo $_ 'CommandLine') -match 'labuser' } |
        Select-Object TimeCreated,
            @{n='Imagen';  e={Get-Campo $_ 'Image'}},
            @{n='Comando'; e={Get-Campo $_ 'CommandLine'}},
            @{n='Hashes';  e={Get-Campo $_ 'Hashes'}} | Format-List
    ```
    Sysmon añade lo que el 4688 no trae: el hash del ejecutable (`SHA256=...`), la línea de comandos del padre y un `ProcessGuid` único para seguir el proceso.
12. Escribe la línea de tiempo en un archivo de texto, con el formato de la nota: hora, ID, qué significa. Por ejemplo `10:41  4625 x5  tipo 3  noexiste desde 127.0.0.1  fuerza bruta`.

### Resultado esperado

Una línea de tiempo de cuatro filas (4625 ×5, 4688 de `net.exe`, 4720, 4732) que cuenta el ataque simulado en orden, sacada solo de los logs y con las consultas que la producen. El alta en el grupo no deja 4688 propio porque `Add-LocalGroupMember` corre dentro de PowerShell sin crear un proceso nuevo: solo el 4732 la delata.

### Comprueba que lo lograste

- ¿Por qué el 4625 de `net use` es de tipo 3 y no de tipo 2? Porque la autenticación llegó por la red (SMB), aunque fuese contra el propio equipo.
- ¿Qué habrías perdido sin el paso 3? La línea de comandos del 4688: sabrías que se ejecutó `net.exe`, pero no con qué argumentos.
- ¿Qué habría quedado escrito en el 4688 con `net user labuser Clave123 /add`? La contraseña en claro, en un log que se envía al SIEM. Por eso el paso 6 usa `*`.
- ¿En qué canal está el evento 1 de Sysmon? `Microsoft-Windows-Sysmon/Operational`, no en Security.

### Limpieza

```powershell
net user labuser /delete
C:\Tools\Sysmon\Sysmon64.exe -u
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /f
auditpol /restore /file:C:\audit-antes.csv
Remove-Item C:\audit-antes.csv, C:\Tools\Sysmon -Recurse
```

Si la VM era solo para esto, basta con volver a una instantánea anterior o borrarla.

## Ejercicio 3: Escribe un mensaje syslog y calcula su PRI

Nodo: [syslogs](README.md#syslogs).

Objetivo: escribir un mensaje con facility `auth` y severidad `warning`, encontrarlo con `journalctl -p warning`, calcular a mano su PRI (debe dar 36) y comprobar ese número viendo el mensaje tal como viaja por la red en formato RFC 5424.

Necesitas: cualquier Linux con systemd (Ubuntu, Debian, Arch) y `logger` (paquete `bsdutils` en Debian/Ubuntu, `util-linux` en Arch; vienen instalados). Para leer el journal del sistema sin `sudo` tu usuario debe estar en el grupo `adm`, `systemd-journal` o `wheel`; si no, antepón `sudo` a `journalctl`. Python 3 para el receptor UDP. Tiempo estimado: 15 minutos.

### Pasos

1. Escribe el mensaje en el log local:
   ```bash
   logger -p auth.warning "prueba de syslog"
   ```
2. Encuéntralo filtrando por severidad:
   ```bash
   journalctl -p warning --since "5 minutes ago" --no-pager | grep "prueba de syslog"
   ```
   ```
   oct 02 12:34:46 equipo jony[72901]: prueba de syslog
   ```
   `-p warning` muestra la severidad 4 y todas las más graves (0 a 4). Repite con `journalctl -p err --since "5 minutes ago" | grep "prueba de syslog"`: no sale nada, porque `err` (3) es más grave que tu mensaje.
3. Mira los campos que guardó el journal:
   ```bash
   journalctl --since "5 minutes ago" -o verbose --no-pager -g "prueba de syslog" | grep -E 'PRIORITY|SYSLOG_FACILITY|MESSAGE='
   ```
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
   En otra terminal, envía el mismo mensaje a ese receptor en formato RFC 5424 por UDP:
   ```bash
   logger -p auth.warning -n 127.0.0.1 -P 5514 -d --rfc5424 "prueba de syslog"
   ```
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
- `tcpdump` y Wireshark (que trae `tshark`). Debian/Ubuntu: `sudo apt install -y tcpdump wireshark tshark`. Arch: `sudo pacman -S tcpdump wireshark-qt`. Para usar Wireshark sin root, añade tu usuario al grupo `wireshark` (`sudo usermod -aG wireshark $USER`) y vuelve a iniciar sesión.
- Tiempo estimado: 30 minutos.

### Pasos

1. Desactiva el DNS cifrado del navegador para esta prueba, o casi no verás consultas DNS: en Firefox, Ajustes → Privacidad y seguridad → DNS sobre HTTPS → Desactivado; en Chrome o Brave, Configuración → Privacidad y seguridad → Seguridad → "Usar DNS seguro" desactivado. Comprueba también si tu sistema usa DNS sobre TLS:
   ```bash
   resolvectl status | grep -i -E 'DNSOverTLS|Current DNS'
   ```
   Si dice `+DNSOverTLS`, el DNS del sistema irá cifrado al puerto 853 y tampoco lo verás; anótalo como hallazgo.
2. Averigua la interfaz por la que sales a Internet:
   ```bash
   ip route get 1.1.1.1
   ```
   La palabra tras `dev` es la interfaz (por ejemplo `wlan0` o `enp3s0`).
3. Captura exactamente 5 minutos. El filtro BPF guarda solo DNS y el tráfico al 443 (TCP para HTTPS y UDP para QUIC), que es lo que hace falta:
   ```bash
   sudo timeout 300 tcpdump -i wlan0 -w navegacion.pcap 'port 53 or port 443'
   ```
   Mientras corre, visita al menos cinco sitios distintos. Al terminar, `tcpdump` dice cuántos paquetes capturó. Haz tuyo el archivo:
   ```bash
   sudo chown $USER: navegacion.pcap
   ```
4. Ábrelo en Wireshark (`wireshark navegacion.pcap`) y aplica en la barra de filtro de visualización:
   ```
   dns.flags.response == 0
   ```
   Así quedan solo las preguntas, sin las respuestas. Para contar dominios distintos, ve a Statistics → DNS, o cuenta con `tshark`:
   ```bash
   tshark -r navegacion.pcap -Y 'dns.flags.response == 0' -T fields -e dns.qry.name | sort -u | wc -l
   tshark -r navegacion.pcap -Y 'dns.flags.response == 0' -T fields -e dns.qry.name | sort | uniq -c | sort -rn | head -20
   ```
   La primera cifra es la respuesta a la primera pregunta. La segunda lista suele sorprender: CDNs, telemetría, anuncios y dominios que nunca escribiste.
5. Para las conexiones TLS, filtra los Client Hello (el primer mensaje del handshake, donde va el SNI en claro):
   ```
   tls.handshake.type == 1
   ```
   Clic derecho sobre el campo Server Name de cualquier paquete (Transport Layer Security → Handshake Protocol → Extension: server_name → Server Name) → Apply as Column. La versión en terminal:
   ```bash
   tshark -r navegacion.pcap -Y 'tls.handshake.type == 1' -T fields -e ip.dst -e ipv6.dst -e tls.handshake.extensions_server_name | sort | uniq -c | sort -rn
   ```
   Cada línea es número de conexiones, IP de destino y SNI. Las conexiones QUIC (UDP 443) también aparecen: Wireshark descifra el paquete Initial de QUIC y muestra su Client Hello.
6. Cruza las dos respuestas: los SNI deberían coincidir en su mayoría con nombres que consultaste por DNS. Busca los que no:
   ```bash
   tshark -r navegacion.pcap -Y 'dns.flags.response == 0' -T fields -e dns.qry.name | sort -u > dns.txt
   tshark -r navegacion.pcap -Y 'tls.handshake.type == 1' -T fields -e tls.handshake.extensions_server_name | sort -u > sni.txt
   comm -13 dns.txt sni.txt
   ```
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

Vuelve a activar el DNS cifrado del navegador si lo tenías.

## Ejercicio 5: Enriquece un hash y tres dominios sin subir nada

Nodo: [VirusTotal](README.md#virustotal), [WHOIS](README.md#whois) y [Subir una muestra a un servicio público es publicarla](README.md#subir-una-muestra-a-un-servicio-público-es-publicarla).

Objetivo: calcular el SHA-256 del archivo de prueba EICAR en tu equipo, consultar su informe en VirusTotal buscando por hash (sin subir el archivo) e interpretar el ratio, la primera fecha de envío y los nombres; después obtener la fecha de creación de tres dominios con RDAP, en lookup.icann.org y por línea de comandos.

Necesitas: cualquier equipo con navegador, `sha256sum` (Linux) o `Get-FileHash` (PowerShell), `curl` y Python 3. EICAR no es malware: es una cadena de 68 bytes que los antivirus acuerdan detectar para probar que funcionan. Aun así, tu antivirus puede borrarla al crearla; en ese caso calcula el hash directamente desde la cadena como en el paso 1. Tiempo estimado: 20 minutos.

### Pasos

1. Calcula el SHA-256 de la cadena EICAR sin escribir el archivo en disco (las comillas simples evitan que la shell interprete `$` y `!`):
   ```bash
   printf '%s' 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' | sha256sum
   ```
   ```
   275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f  -
   ```
   Si te sale otro hash, sobra o falta un byte (por ejemplo un salto de línea final); por eso se usa `printf '%s'` y no `echo`.
2. Busca ese hash en VirusTotal. Abre [virustotal.com](https://www.virustotal.com/), pestaña SEARCH (no FILE), pega el hash y pulsa Enter, o ve directo a:
   ```
   https://www.virustotal.com/gui/file/275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f
   ```
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
   ```
   GOOGLE.COM ['1997-09-15T04:00:00Z']
   wikipedia.org ['2001-01-13T00:12:14.754Z']
   GITHUB.COM ['2007-10-09T18:20:50Z']
   ```
6. Si tienes `whois` instalado (`sudo apt install -y whois` o `sudo pacman -S whois`), compáralo con el formato antiguo:
   ```bash
   whois github.com | grep -i -E 'creation date|registrar:|registrant organization'
   ```
   Fíjate en si el titular aparece o sale `REDACTED FOR PRIVACY`.
7. Cierra con una conclusión por escrito de dos líneas: qué te dijo cada servicio y qué enviaste a cada uno (un hash y tres nombres de dominio, ninguno de ellos dato interno).

### Resultado esperado

Un texto con el SHA-256 de EICAR, su ratio de detección, su fecha de primer envío y su número de nombres, las fechas de creación de tres dominios obtenidas por dos vías que coinciden, y la constancia de que no subiste ningún archivo.

### Comprueba que lo lograste

- ¿Qué habría pasado si hubieses usado la pestaña FILE con un documento interno? Que el archivo quedaría disponible para los clientes de pago de VirusTotal.
- ¿Un `0/72` en otro hash significaría que el archivo es seguro? No: un malware nuevo o dirigido puede tener cero detecciones.
- Un correo de "tu banco" llega desde un dominio cuyo `registration` es de hace dos días. ¿Qué concluyes aunque urlvoid diga 0 detecciones? Que es sospechoso igual: los bancos no estrenan dominio para escribir a sus clientes.
