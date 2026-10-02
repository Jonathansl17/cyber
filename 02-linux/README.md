# Linux

## Conceptos previos

- Kernel: el núcleo del sistema operativo; administra CPU, memoria, discos y dispositivos, y decide qué puede hacer cada programa.
- Shell: el programa que lee los comandos que escribes y los ejecuta (Bash, Zsh); la terminal es solo la ventana donde corre.
- CLI (Command-Line Interface): forma de usar el equipo escribiendo comandos de texto.
- GUI (Graphical User Interface): forma de usar el equipo con ventanas, íconos y ratón.
- Usuario root: el administrador de Linux, con UID 0; el kernel no le aplica restricciones de permisos.
- UID / GID: número que identifica a un usuario (User ID) y a un grupo (Group ID); el kernel trabaja con estos números, no con los nombres.
- Proceso y PID: programa en ejecución y su número identificador.
- Servicio (daemon): programa que corre en segundo plano sin usuario delante, como un servidor web o SSH.
- Ruta (path): la dirección de un archivo dentro del árbol de carpetas, como `/home/ana/notas.txt`.
- Código abierto: software cuyo código fuente se publica y se puede leer, modificar y redistribuir.
- Escalada de privilegios: pasar de un usuario con pocos permisos a uno con más, normalmente root.

## Linux

### Linux

**Linux** es un kernel de sistema operativo de código abierto, tipo Unix, creado por Linus Torvalds en 1991, que junto con herramientas de usuario (en gran parte del proyecto GNU) forma sistemas operativos completos llamados distribuciones. En el habla diaria "Linux" nombra al sistema entero, aunque técnicamente es solo el núcleo.

Por qué existe: en 1991 los Unix comerciales eran caros y cerrados. El proyecto GNU ya tenía casi todo un sistema libre (compilador, shell, utilidades) pero le faltaba el kernel; Linux llenó ese hueco y el resultado se pudo copiar, estudiar y modificar sin pagar licencias.

Analogía: el kernel es el motor de un auto; el userland (todo lo que no es kernel: shell, comandos, bibliotecas, escritorio) es la carrocería, el tablero y los asientos. Una distribución es un modelo concreto de auto armado alrededor del mismo motor: cambia la carrocería y el equipamiento, pero el motor es Linux.

Cómo está organizado por dentro:

```
+---------------------------------------------------------+
|  Aplicaciones: navegador, nmap, Wireshark, servidor web |   userland
|  Shell (bash) y comandos (ls, grep, ps)                 |   (espacio de usuario)
|  Bibliotecas (glibc) y gestor de servicios (systemd)    |
+------------------- llamadas al sistema -----------------+
|  Kernel Linux: procesos, memoria, archivos, red,        |   espacio de kernel
|  permisos, drivers                                      |
+---------------------------------------------------------+
|  Hardware: CPU, RAM, disco, tarjeta de red              |
+---------------------------------------------------------+
```

El userland no toca el hardware directamente: pide cada cosa al kernel mediante una llamada al sistema (syscall), por ejemplo `open()` para abrir un archivo. En ese punto el kernel revisa los permisos. Por eso un bug en el kernel es mucho más grave que uno en un programa: el kernel es el que vigila.

Distribuciones: una **distro** es un sistema operativo completo que empaqueta el kernel Linux con un userland, un gestor de paquetes y una configuración por defecto. Las familias que más se ven en seguridad:

- Debian y derivadas (Ubuntu, Kali Linux, Parrot): paquetes `.deb`, gestor `apt`. Kali y Parrot traen cientos de herramientas de pentesting preinstaladas.
- Red Hat y derivadas (RHEL, Fedora, Rocky Linux, AlmaLinux): paquetes `.rpm`, gestor `dnf`. Muy comunes en servidores corporativos.
- Arch Linux: actualización continua (rolling release), gestor `pacman`; documentación de referencia en la Arch Wiki.
- Alpine: mínima (una imagen base de contenedor ocupa unos pocos MB), muy usada en Docker.

Por qué domina en seguridad:

1. Está en los objetivos: la gran mayoría de los servidores web, la nube, los contenedores, los routers y los dispositivos IoT corren Linux, y desde noviembre de 2017 el 100 % de las 500 supercomputadoras del TOP500 también. Atacar o defender infraestructura es, casi siempre, trabajar sobre Linux.
2. Está en las herramientas: nmap, Wireshark, Metasploit, Burp, tcpdump y casi todo el arsenal nace o funciona mejor en Linux.
3. Es transparente: el código es público y casi toda la configuración son archivos de texto, así que se puede auditar qué hace el sistema y por qué.
4. Se controla por completo desde la CLI, lo que permite automatizar con scripts y trabajar por SSH en servidores sin pantalla.

Ejemplo: comprobar qué kernel y qué distro tiene una máquina.

```
$ uname -r
6.8.0-45-generic
$ cat /etc/os-release | head -3
PRETTY_NAME="Ubuntu 24.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
```

Saber la versión exacta del kernel es lo primero que se mira al evaluar si una máquina es vulnerable a un fallo conocido del kernel.

```
Linux → el kernel; en el habla diaria, el sistema completo.
Kernel → núcleo que controla hardware y permisos.
Userland → todo lo que no es kernel: shell, comandos, bibliotecas, apps.
GNU → proyecto que aportó gran parte del userland.
Distro → kernel + userland + gestor de paquetes + configuración.
Syscall → la puerta por la que un programa le pide algo al kernel.
```

## Learn following for each

### Navigating using GUI and CLI

Navegar en Linux es la habilidad de moverse por el árbol de archivos, ubicar dónde vive cada cosa y operar sobre ella, ya sea con un gestor de archivos gráfico (GUI) o con comandos en la shell (CLI).

Por qué importa: en un escritorio se puede usar la GUI (Nautilus en GNOME, Dolphin en KDE, Thunar en XFCE), pero un servidor normalmente no tiene entorno gráfico y se administra por SSH. En seguridad casi todo pasa en la CLI: una shell obtenida en un laboratorio, un servidor que hay que revisar tras un incidente, un contenedor.

Analogía: la GUI es recorrer una biblioteca caminando y mirando estantes; la CLI es pedirle al bibliotecario "tráeme el libro de la estantería 3, fila 2". La segunda es más rápida en cuanto sabes la dirección, y funciona aunque la biblioteca esté a oscuras.

El árbol de archivos: Linux tiene una sola raíz `/` y todo cuelga de ella, incluidos otros discos (que se "montan" en una carpeta). No hay `C:` ni `D:`. La estructura está estandarizada por el FHS (Filesystem Hierarchy Standard):

```
/
├── bin, sbin, usr/bin   programas (en distros modernas apuntan a /usr)
├── etc                  configuración del sistema (texto plano)
├── home                 carpetas personales: /home/ana, /home/luis
├── root                 carpeta personal de root
├── var                  datos que cambian: var/log (logs), var/www (web)
├── tmp                  temporales; cualquiera puede escribir
├── dev                  dispositivos como archivos: /dev/sda (disco)
├── proc, sys            información del kernel y procesos (virtual)
├── boot                 kernel e initramfs para arrancar
└── opt                  software instalado aparte
```

Rutas: una ruta absoluta empieza en `/` y funciona desde cualquier lugar (`/etc/passwd`); una ruta relativa parte de la carpeta actual (`documentos/notas.txt`). Atajos: `.` es la carpeta actual, `..` la carpeta de arriba, `~` tu carpeta personal, `-` la carpeta donde estabas antes. Los archivos que empiezan con punto (`.bashrc`, `.ssh`) son ocultos: no aparecen en `ls` sin `-a` ni en la GUI sin activar "mostrar ocultos".

Ejemplo de recorrido:

```
$ pwd
/home/ana
$ ls
documentos  descargas  script.sh
$ cd /var/log
$ ls -la | head -4
total 2048
drwxrwxr-x  12 root syslog   4096 oct  2 09:00 .
drwxr-xr-x  14 root root     4096 ago 10 12:00 ..
-rw-r-----   1 syslog adm  120000 oct  2 09:15 auth.log
$ cd ~
$ pwd
/home/ana
```

Atajos que ahorran horas en la CLI: `Tab` completa nombres de archivos y comandos (dos `Tab` muestran las opciones); flecha arriba recupera el comando anterior; `Ctrl+R` busca en el historial; `Ctrl+C` corta un comando; `man <comando>` abre su manual y `<comando> --help` da un resumen.

```
GUI → gestor de archivos gráfico; cómodo en escritorio.
CLI → comandos en la shell; único medio en servidores y shells remotas.
/ → raíz única de todo el árbol.
Ruta absoluta → empieza en /; funciona desde cualquier lugar.
Ruta relativa → parte de la carpeta actual.
. / .. / ~ → aquí / arriba / mi carpeta personal.
.archivo → oculto; se ve con ls -a.
FHS → estándar que fija qué va en /etc, /var, /home, etc.
```

### En Linux todo es un archivo

> [!IMPORTANT]
> Linux representa casi todo como un archivo dentro del árbol: documentos, dispositivos, procesos y configuración. Por eso las mismas herramientas (leer, buscar, permisos) sirven para todo.

"Todo es un archivo" es el principio de diseño de Unix que da a cada recurso una ruta en el árbol y lo hace accesible con las operaciones normales de leer y escribir.

```
                    Documento          Disco              Proceso 1234          Configuración
Ruta                /home/ana/a.txt    /dev/sda           /proc/1234/           /etc/ssh/sshd_config
Se lee con          cat                (con cuidado) dd   cat /proc/1234/status cat
Lo protegen         permisos rwx       permisos rwx       permisos rwx          permisos rwx
```

La última fila es idéntica en las cuatro columnas: ahí está la idea entera. Quien controla los permisos de los archivos controla el sistema.

Ejemplo: ver la línea de comandos con la que se lanzó un proceso, algo útil al investigar un proceso sospechoso, es leer un archivo:

```
$ cat /proc/1234/cmdline | tr '\0' ' '
/usr/bin/python3 /tmp/.x/miner.py --pool 203.0.113.5
```

Límite: las conexiones de red no aparecen como archivos con ruta normal (se consultan con `ss`), y `/proc` y `/sys` no están en el disco, los genera el kernel al vuelo.

```
Todo es un archivo → cada recurso tiene ruta y se opera con leer/escribir.
/dev → dispositivos; /proc → procesos; /etc → configuración.
```

### Understand Permissions

Los **permisos de Linux** son un mecanismo de control de acceso que define, para cada archivo y carpeta, qué puede hacer su dueño, su grupo y el resto de los usuarios: leer, escribir o ejecutar.

Por qué existen: Linux nació multiusuario. En un mismo servidor conviven personas y servicios (el servidor web, la base de datos), y cada uno debe poder tocar solo lo suyo. Si el servidor web es comprometido, los permisos deciden si el atacante solo ve la web o puede leer las contraseñas de todo el sistema.

Analogía: un edificio de apartamentos. El dueño del apartamento tiene todas las llaves; los de su familia (el grupo) entran a ciertas habitaciones; los vecinos (otros) solo ven el buzón. root es el conserje con llave maestra.

Usuarios y grupos: cada usuario tiene un UID y un grupo principal, y puede pertenecer a grupos extra. Se definen en `/etc/passwd` (legible por todos) y `/etc/group`; los hashes de contraseña están en `/etc/shadow`, que solo root puede leer.

```
$ id
uid=1000(ana) gid=1000(ana) groups=1000(ana),27(sudo),998(docker)
$ grep ana /etc/passwd
ana:x:1000:1000:Ana:/home/ana:/bin/bash
```

Los campos de `/etc/passwd` son: usuario, `x` (la contraseña está en shadow), UID, GID, comentario, carpeta personal y shell.

Cómo se leen los permisos en `ls -l`:

```
-rwxr-x---  1 ana  dev  4096 oct  2 10:00 deploy.sh
│└┬┘└┬┘└┬┘     │    │
│ │  │  │      │    └─ grupo del archivo
│ │  │  │      └────── dueño
│ │  │  └─ otros: ---  (nada)
│ │  └──── grupo: r-x  (leer y ejecutar)
│ └─────── dueño: rwx  (todo)
└───────── tipo: - archivo, d carpeta, l enlace simbólico
```

El kernel evalúa en orden y se queda con la primera coincidencia: si eres el dueño, aplica la terna del dueño; si no, y estás en el grupo, la del grupo; si no, la de otros. Por eso un dueño con `---` no puede leer su propio archivo aunque "otros" tenga `r`.

Qué significa cada letra depende de si es archivo o carpeta, y la diferencia se pregunta mucho:

```
r en archivo → leer el contenido.        r en carpeta → listar los nombres (ls).
w en archivo → modificar el contenido.   w en carpeta → crear, borrar y renombrar archivos dentro.
x en archivo → ejecutarlo como programa. x en carpeta → entrar (cd) y acceder a lo de dentro.
```

Notación octal: cada letra vale un número, `r=4`, `w=2`, `x=1`, y se suman por terna. Tres dígitos describen dueño, grupo y otros.

```
rwx = 4+2+1 = 7      r-x = 4+0+1 = 5      r-- = 4      --- = 0
755 = rwxr-xr-x  → programas y carpetas públicas.
644 = rw-r--r--  → archivos normales.
600 = rw-------  → secretos: claves SSH privadas.
700 = rwx------  → carpetas privadas como ~/.ssh.
777 = rwxrwxrwx  → todos pueden todo; casi nunca es correcto.
```

Cambiar permisos y dueños:

```
$ chmod 600 ~/.ssh/id_ed25519        # octal: solo el dueño lee y escribe
$ chmod u+x script.sh                # simbólico: añade x al dueño (u=user, g=group, o=others, a=all)
$ chmod go-w informe.txt             # quita escritura a grupo y otros
$ chmod -R 750 /srv/app              # recursivo sobre toda la carpeta
$ sudo chown www-data:www-data /var/www/html/index.html   # cambia dueño y grupo
$ sudo chgrp dev proyecto/           # solo el grupo
```

Solo root puede cambiar el dueño de un archivo (`chown`); el dueño puede cambiar los permisos (`chmod`) y el grupo a uno de los suyos.

umask: la **umask** es una máscara que resta permisos a los archivos y carpetas recién creados. Los archivos nacen como máximo con 666 y las carpetas con 777; la umask apaga bits. Con la umask típica 022: archivos 644, carpetas 755. Con 077: archivos 600, carpetas 700, nada visible para otros.

```
$ umask
0022
$ touch nuevo.txt && mkdir nueva
$ ls -ld nuevo.txt nueva
-rw-r--r-- 1 ana ana    0 oct  2 10:10 nuevo.txt
drwxr-xr-x 2 ana ana 4096 oct  2 10:10 nueva
```

Permisos especiales, un cuarto dígito delante de los tres normales:

- SUID (4000, se ve como `s` en la x del dueño): el programa corre con los permisos de su dueño, no de quien lo ejecuta. `passwd` es SUID root porque un usuario normal necesita escribir en `/etc/shadow` para cambiar su contraseña.
- SGID (2000, `s` en la x del grupo): en un programa, corre con el grupo del archivo; en una carpeta, todo lo creado dentro hereda el grupo de la carpeta (útil para carpetas de equipo).
- Sticky bit (1000, `t` en la x de otros): en una carpeta, solo el dueño de cada archivo puede borrarlo o renombrarlo aunque todos puedan escribir. Es lo que protege `/tmp`.
- Si aparece en mayúscula (`S` o `T`), el bit especial está puesto pero falta la `x` debajo: suele ser un error de configuración.

```
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 64152 may 30 12:00 /usr/bin/passwd      # 4755, SUID root
$ ls -ld /tmp
drwxrwxrwt 18 root root 4096 oct  2 10:00 /tmp                 # 1777, sticky
$ chmod 2775 /srv/equipo                                       # SGID en carpeta compartida
```

sudo: **sudo** es un programa que permite a usuarios autorizados ejecutar comandos como root (u otro usuario) usando su propia contraseña, y deja registro de cada uso. Las reglas viven en `/etc/sudoers` y `/etc/sudoers.d/`, y se editan siempre con `visudo`, que valida la sintaxis antes de guardar (un error de sintaxis puede dejar a todos sin sudo).

```
$ sudo -l
User ana may run the following commands on srv01:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/systemctl restart nginx
```

La primera regla da root completo; la segunda deja reiniciar nginx sin contraseña. sudo es mejor que compartir la contraseña de root porque cada persona usa la suya, se puede limitar a comandos concretos y todo queda en el log (`/var/log/auth.log` en Debian/Ubuntu, `journalctl` en general).

> [!WARNING]
> Un permiso de más es una escalada de privilegios esperando. Un binario SUID root que permite abrir una shell o escribir archivos, o una regla sudo sobre un editor como `vim`, le da root a cualquiera que lo ejecute.

```
rwx → leer, escribir, ejecutar.
Dueño / grupo / otros → las tres ternas; se aplica la primera que coincide.
x en carpeta → poder entrar; w en carpeta → crear y borrar dentro.
Octal → r=4, w=2, x=1; 755, 644, 600, 700.
chmod → cambia permisos; chown → cambia dueño (solo root); chgrp → cambia grupo.
umask → resta permisos al crear; 022 da 644/755, 077 da 600/700.
SUID 4000 → corre como el dueño del archivo.
SGID 2000 → corre con el grupo; en carpeta, hereda el grupo.
Sticky 1000 → en carpeta, solo el dueño borra lo suyo (/tmp).
sudo → ejecutar como root con tu contraseña, con reglas y registro.
visudo → única forma segura de editar sudoers.
```

### Los permisos son el mapa de la escalada de privilegios

> [!IMPORTANT]
> Tras entrar a una máquina Linux como usuario normal, el atacante busca permisos mal puestos; el defensor tiene que buscarlos antes. SUID, sudo y archivos escribibles son la misma pregunta: ¿qué puedo hacer con permisos de otro?

La escalada de privilegios en Linux se apoya casi siempre en un permiso que concede más de lo necesario, no en un fallo exótico. Las tres búsquedas clásicas, que un auditor hace igual que un atacante:

```
$ find / -perm -4000 -type f 2>/dev/null      # binarios SUID
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/find                                  # <- anómalo: find no debería ser SUID
$ sudo -l                                      # qué deja sudo
$ find /etc -writable -type f 2>/dev/null      # configuración que puedo modificar
```

`2>/dev/null` descarta los errores de "permiso denegado" para que solo se vean los resultados. Si `find` es SUID root, cualquier usuario puede usar su opción `-exec` para ejecutar comandos como root; GTFOBins es el catálogo público de qué binarios permiten esto y es igual de útil para el defensor que audita.

```
                       SUID mal puesto        Regla sudo amplia       /etc/passwd escribible
Qué concede            correr como dueño      correr como root        crear un usuario UID 0
Cómo se detecta        find -perm -4000       sudo -l                 find -writable / ls -l
Resultado si se abusa  root                   root                    root
```

La última fila es idéntica: cualquiera de los tres termina en root. Defensa: principio de mínimo privilegio (dar solo lo necesario), revisar periódicamente los SUID contra una lista conocida y limitar las reglas sudo a comandos exactos. Esto se practica legalmente en las salas de TryHackMe y en OverTheWire Bandit, nunca en equipos ajenos. El hardening completo está en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

```
Escalada de privilegios → pasar de usuario normal a root.
find -perm -4000 → lista binarios SUID.
sudo -l → lista lo que sudo permite.
GTFOBins → catálogo de binarios abusables con SUID o sudo.
Mínimo privilegio → dar solo el permiso necesario.
```

### Troubleshooting

El troubleshooting en Linux es la aplicación del método general de diagnóstico (ver [`01-fundamentos-it`](../01-fundamentos-it/)) con las herramientas propias de Linux: logs, gestor de servicios, procesos, disco, memoria y red.

Por qué importa en seguridad: los mismos comandos que explican por qué un servicio no arranca muestran un proceso que no debería existir, un puerto que nadie abrió o un login a las 3 de la madrugada. Diagnosticar y detectar usan las mismas herramientas.

Analogía: es el tablero y la caja negra de un avión. El tablero (top, df, ss) dice cómo está todo ahora; la caja negra (los logs) dice qué pasó antes.

Logs: la mayoría de las distros actuales usan systemd, que guarda los logs en un diario binario (journal) consultado con `journalctl`. Muchas mantienen además archivos de texto en `/var/log`:

```
/var/log/syslog o /var/log/messages → log general (Debian / Red Hat).
/var/log/auth.log o /var/log/secure → logins, sudo, SSH (Debian / Red Hat).
/var/log/kern.log                   → mensajes del kernel.
/var/log/nginx/, /var/log/apache2/   → logs del servidor web.
```

```
$ journalctl -u ssh --since "1 hour ago"     # logs de un servicio en la última hora
$ journalctl -p err -b                       # solo errores (prioridad 3 o peor) desde el último arranque
$ journalctl -f                              # seguir en vivo, como tail -f
$ journalctl -k                              # solo kernel (equivale a dmesg)
```

Las prioridades van de 0 (emerg) a 7 (debug); `-p err` muestra 0 a 3.

Servicios con systemctl:

```
$ systemctl status nginx
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
     Active: failed (Result: exit-code) since Thu 2026-10-02 10:20:01 UTC
    Process: 812 ExecStart=/usr/sbin/nginx (code=exited, status=1/FAILURE)
oct 02 10:20:01 srv01 nginx[812]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
$ sudo systemctl restart nginx       # start, stop, restart, reload
$ sudo systemctl enable nginx        # arrancar al iniciar el sistema
$ systemctl --failed                 # todos los servicios caídos
```

`enabled` significa que arranca con el sistema; `active` que está corriendo ahora. Son cosas distintas. En el ejemplo, el log dice que el puerto 80 ya está ocupado: el siguiente paso es ver quién lo tiene.

Procesos:

```
$ ps aux --sort=-%cpu | head -3
USER   PID %CPU %MEM COMMAND
www    4321 98.5  2.0 /tmp/.x/kworkerd
root    812  0.1  0.5 /usr/sbin/sshd -D
$ kill 4321          # pide terminar (señal 15, SIGTERM)
$ kill -9 4321       # fuerza (señal 9, SIGKILL); el proceso no puede negarse
```

`top` o `htop` muestran lo mismo en vivo. Un proceso al 98 % de CPU corriendo desde `/tmp` con nombre de proceso del kernel es la firma típica de un minero de criptomonedas: antes de matarlo, en un incidente real se documenta (ruta, PID, conexiones) para no perder evidencia.

Disco y memoria:

```
$ df -h /
Filesystem  Size  Used Avail Use% Mounted on
/dev/sda1    50G   49G  1.0G  98% /
$ du -sh /var/log/* | sort -h | tail -2
1.2G  /var/log/journal
18G   /var/log/nginx
$ df -i /            # inodos: puede quedarse sin ellos con millones de archivos pequeños aunque haya espacio
$ free -h
       total  used  free  available
Mem:    8.0G  6.1G  300M       1.6G
```

Un disco por encima del 90 % suele romper servicios (no pueden escribir logs ni temporales). En `free`, la columna que importa es `available`, no `free`: Linux usa la RAM libre como caché y la devuelve cuando hace falta.

Red:

```
$ ip -br addr                # interfaces y direcciones
eth0   UP   192.168.1.20/24
$ ip route                   # tabla de rutas; "default via" es la puerta de enlace
default via 192.168.1.1 dev eth0
$ ss -tulpn                  # puertos escuchando y qué proceso los tiene
Netid State  Local Address:Port  Process
tcp   LISTEN 0.0.0.0:22          users:(("sshd",pid=812))
tcp   LISTEN 0.0.0.0:80          users:(("apache2",pid=990))
$ cat /etc/resolv.conf       # servidores DNS configurados
$ dig +short ejemplo.com     # probar la resolución
```

`ss -tulpn` resuelve el caso de nginx de arriba: apache2 tiene el puerto 80. Y sirve en seguridad: un puerto en escucha que nadie reconoce es motivo de investigación.

> [!TIP]
> Orden práctico ante un servicio que falla: `systemctl status` del servicio, luego `journalctl -u` con el rango de tiempo, luego recursos (`df -h`, `free -h`) y por último red (`ss -tulpn`). El mensaje de error del log casi siempre dice exactamente qué pasó.

```
journalctl → consulta el diario de systemd; -u servicio, -p prioridad, -b arranque, -f en vivo.
/var/log → logs en texto; auth.log/secure guarda logins y sudo.
systemctl status → estado y últimas líneas de log de un servicio.
enabled → arranca con el sistema; active → corre ahora.
ps / top → procesos y consumo.
kill (15) → pide terminar; kill -9 → fuerza.
df -h → espacio por disco; du -sh → peso por carpeta; df -i → inodos.
free -h → memoria; mirar available.
ip addr / ip route → direcciones y rutas.
ss -tulpn → puertos en escucha y su proceso.
```

### Common Commands

Los comandos comunes de Linux son el conjunto de utilidades de la shell que se usan a diario para manejar archivos, leer y filtrar texto, buscar, administrar procesos, usuarios, paquetes y red, y que se combinan entre sí con tuberías y redirecciones.

Por qué importan: casi ningún problema real se resuelve con un comando suelto. La filosofía Unix es "programas pequeños que hacen una cosa bien y se encadenan", y la potencia está en la cadena.

Analogía: son piezas de Lego. Cada una es simple (`sort` solo ordena, `uniq` solo quita repetidos), pero conectadas construyen un análisis de logs completo en una línea.

Tuberías y redirecciones, lo que une todo:

```
cmd1 | cmd2    → la salida de cmd1 entra a cmd2 (pipe).
cmd > f        → salida a archivo, sobrescribe.
cmd >> f       → salida a archivo, añade al final.
cmd 2> f       → errores a archivo (2 es la salida de errores); 2>/dev/null los descarta.
cmd < f        → el archivo entra como entrada.
cmd1 && cmd2   → cmd2 solo si cmd1 salió bien.
```

Archivos y carpetas:

```
$ ls -lah                  # lista con detalles, ocultos y tamaños legibles
$ mkdir -p a/b/c           # crea carpetas anidadas
$ cp -r origen/ destino/   # copia (recursivo para carpetas)
$ mv viejo.txt nuevo.txt   # mueve o renombra
$ rm archivo.txt           # borra; rm -r carpeta borra recursivo, sin papelera
$ touch vacio.txt          # crea vacío o actualiza la fecha
$ ln -s /etc/hosts atajo   # enlace simbólico
$ file misterio.bin
misterio.bin: ELF 64-bit LSB executable, x86-64
```

`file` mira el contenido, no la extensión: en Linux la extensión no decide nada, y un `.jpg` puede ser un ejecutable.

Leer texto:

```
$ cat archivo              # todo de una vez
$ less archivo             # página a página; q para salir, / para buscar
$ head -n 5 archivo        # primeras 5 líneas
$ tail -n 20 -f /var/log/auth.log   # últimas 20 y sigue en vivo
$ wc -l archivo            # contar líneas
```

Buscar y filtrar:

```
$ find /home -name "*.kdbx" 2>/dev/null       # buscar archivos por nombre
$ find / -mtime -1 -type f 2>/dev/null        # modificados en las últimas 24 h
$ grep -i "failed password" /var/log/auth.log # buscar texto (-i ignora mayúsculas)
$ grep -rn "password" /etc/ 2>/dev/null       # recursivo con número de línea
$ which nmap                                  # dónde está un programa
/usr/bin/nmap
```

Procesar texto, la cadena que más se usa en análisis de logs. Ejemplo: qué IP intentó más contraseñas fallidas por SSH.

```
$ grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -3
    412 203.0.113.50
     37 198.51.100.7
      3 192.168.1.15
```

`grep` filtra las líneas, `awk` extrae el campo de la IP (cuarto empezando por el final), `sort` ordena, `uniq -c` cuenta repetidos, `sort -rn` ordena por número de mayor a menor y `head` deja los 3 primeros. 412 intentos desde una sola IP es fuerza bruta. Otros útiles: `cut -d: -f1 /etc/passwd` (primer campo separado por `:`), `sed 's/viejo/nuevo/g'` (reemplazar), `tr` (cambiar caracteres), `diff a b` (diferencias).

Usuarios e identidad:

```
$ whoami          # usuario actual
$ id              # UID, GID y grupos
$ who / w         # quién está conectado
$ last -n 3       # últimos inicios de sesión
$ sudo useradd -m -s /bin/bash luis && sudo passwd luis
```

Paquetes (varía por familia):

```
$ sudo apt update && sudo apt install nmap     # Debian/Ubuntu/Kali
$ sudo dnf install nmap                        # Fedora/RHEL
$ sudo pacman -S nmap                          # Arch
```

Red y transferencia:

```
$ ping -c 4 1.1.1.1
$ curl -I https://ejemplo.com          # solo cabeceras HTTP
$ wget https://ejemplo.com/archivo.zip
$ ssh ana@192.168.1.20                 # shell remota cifrada, puerto 22
$ scp informe.txt ana@192.168.1.20:/tmp/
```

Comprimir, codificar y verificar:

```
$ tar -czf respaldo.tar.gz carpeta/    # crear (c), gzip (z), archivo (f)
$ tar -xzf respaldo.tar.gz             # extraer (x)
$ base64 -d <<< "aG9sYQ=="
hola
$ sha256sum descarga.iso
3f2a...e91c  descarga.iso
```

`sha256sum` sirve para verificar que un archivo descargado no fue alterado, comparando con el hash que publica el sitio oficial (detalle en [`10-criptografia`](../10-criptografia/)).

> [!NOTE]
> Ante cualquier duda, `man <comando>` es la documentación oficial instalada en la propia máquina y funciona sin Internet; `apropos <palabra>` busca qué comando hace algo.

```
| → encadena la salida de un comando a la entrada del siguiente.
> / >> / 2> → a archivo sobrescribiendo / añadiendo / solo errores.
ls, cd, pwd, mkdir, cp, mv, rm, touch, ln → manejo de archivos.
file → tipo real por contenido, no por extensión.
cat, less, head, tail -f, wc → leer texto.
find → buscar archivos; grep → buscar texto dentro.
awk, cut, sort, uniq -c, sed, tr → procesar texto.
whoami, id, w, last → identidad y sesiones.
apt / dnf / pacman → gestores de paquetes por familia.
ssh, scp, curl, wget → acceso remoto y transferencia.
tar -czf / -xzf → comprimir / extraer.
sha256sum → verificar integridad con hash.
man → manual oficial del comando.
```

## Recursos para aprender y practicar

### Videos

- [Linux for Hackers // EP 1 (FREE Linux course for beginners)](https://www.youtube.com/watch?v=VbEx7B_PTOE) — NetworkChuck; por qué Linux en seguridad y primeros pasos. Nodo Linux.
- [apt, dpkg, git, Python PiP (Linux Package Management) // Linux for Hackers // EP 5](https://www.youtube.com/watch?v=vX3krP6JmOY) — NetworkChuck; gestión de paquetes. Nodos Linux y Common Commands.
- [KILL Linux processes!! (also manage them) // Linux for Hackers // EP 7](https://www.youtube.com/watch?v=LfC6pv8VISk) — NetworkChuck; ps, top, kill y señales. Nodo Troubleshooting.
- [Introduction to Linux – Full Course for Beginners](https://www.youtube.com/watch?v=sWbUDq4S6Y8) — freeCodeCamp.org; curso de unas 6 h que cubre GUI, CLI, sistema de archivos y administración. Nodos Linux y Navigating using GUI and CLI.
- [The 50 Most Popular Linux & Terminal Commands - Full Course for Beginners](https://www.youtube.com/watch?v=ZtqBQ68cfJc) — freeCodeCamp.org (Colt Steele); comandos uno por uno con ejemplos. Nodo Common Commands.
- [Linux Crash Course - Understanding File & Directory Permissions](https://www.youtube.com/watch?v=4e669hSjaX8) — Learn Linux TV; rwx, octal, chmod y chown. Nodo Understand Permissions.
- [Linux Crash Course - sudo](https://www.youtube.com/watch?v=07JOqKOBRnU) — Learn Linux TV; sudo, sudoers y visudo. Nodo Understand Permissions.
- [journalctl Basics: How to Easily Check Your Linux Logs](https://www.youtube.com/watch?v=0dG3vUYt7Uk) — Learn Linux TV; filtros de journalctl. Nodo Troubleshooting.

### Lectura y documentación

- [Linux and GNU](https://www.gnu.org/gnu/linux-and-gnu.html) — GNU Project; la relación entre el kernel Linux y el userland GNU. Nodo Linux.
- [The Linux Kernel Archives](https://www.kernel.org/) — sitio oficial del kernel y sus versiones. Nodo Linux.
- [Filesystem Hierarchy Standard 3.0](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) — Linux Foundation; qué va en cada carpeta. Nodo Navigating.
- [hier(7)](https://man7.org/linux/man-pages/man7/hier.7.html) — man page de la jerarquía de carpetas. Nodo Navigating.
- [chmod(1)](https://man7.org/linux/man-pages/man1/chmod.1.html), [chown(1)](https://man7.org/linux/man-pages/man1/chown.1.html) y [umask(2)](https://man7.org/linux/man-pages/man2/umask.2.html) — man pages oficiales. Nodo Understand Permissions.
- [credentials(7)](https://man7.org/linux/man-pages/man7/credentials.7.html) — UID, GID y UID efectivo, la base de SUID. Nodo Understand Permissions.
- [sudoers manual](https://www.sudo.ws/docs/man/sudoers.man/) — sintaxis oficial de las reglas sudo. Nodo Understand Permissions.
- [Arch Wiki: File permissions and attributes](https://wiki.archlinux.org/title/File_permissions_and_attributes) y [Arch Wiki: Umask](https://wiki.archlinux.org/title/Umask) — explicación detallada con ejemplos. Nodo Understand Permissions.
- [GTFOBins](https://gtfobins.github.io/) — catálogo de binarios abusables con SUID o sudo, para auditar. Nodo Understand Permissions.
- [journalctl(1)](https://man7.org/linux/man-pages/man1/journalctl.1.html) y [systemctl(1)](https://man7.org/linux/man-pages/man1/systemctl.1.html) — man pages oficiales. Nodo Troubleshooting.
- [Arch Wiki: Systemd/Journal](https://wiki.archlinux.org/title/Systemd/Journal) — filtros, persistencia y tamaño del journal. Nodo Troubleshooting.
- [The Linux Command Line (William Shotts)](https://linuxcommand.org/tlcl.php) — libro libre y gratuito, de cero a scripts. Nodos Navigating y Common Commands.

### Práctica

- [OverTheWire: Bandit](https://overthewire.org/wargames/bandit/) — wargame por SSH, cada nivel exige un comando o permiso nuevo (find, grep, sort, uniq, base64, SUID, ssh). Nodos Common Commands, Navigating y Understand Permissions.
- [TryHackMe: Linux Fundamentals Part 1](https://tryhackme.com/room/linuxfundamentalspart1) — primeros comandos, navegación, find y grep en una máquina en el navegador. Nodos Navigating y Common Commands.
- [TryHackMe: módulo Linux Fundamentals](https://tryhackme.com/module/linux-fundamentals) — agrupa las partes 1, 2 y 3: la 2 añade SSH, flags, permisos y carpetas clave; la 3 añade editores, procesos, cron, paquetes y logs. Nodos Understand Permissions, Troubleshooting y Common Commands.
- [TryHackMe: Linux Strength Training](https://tryhackme.com/room/linuxstrengthtraining) — retos de búsqueda de archivos, permisos y procesado de texto. Nodo Common Commands.
- [TryHackMe: Linux Privilege Escalation](https://tryhackme.com/room/linprivesc) — SUID, sudo, cron y capabilities en un laboratorio legal. Nodo Understand Permissions.
- [TryHackMe: Linux Logging for SOC](https://tryhackme.com/room/linuxloggingforsoc) — gratis; /var/log, auth.log, journal y auditd desde el punto de vista del analista. Nodo Troubleshooting.
- [HTB Academy: Linux Fundamentals](https://academy.hackthebox.com/course/preview/linux-fundamentals) — módulo completo de sistema, usuarios, permisos, servicios y red. Todos los nodos.
- Ejercicio en casa (Permissions): en una VM crea dos usuarios y un grupo compartido; arma una carpeta con SGID (2770) y sticky para que ambos escriban pero ninguno borre lo del otro, y comprueba con `ls -l` el grupo heredado.
- Ejercicio en casa (Permissions): ejecuta `find / -perm -4000 -type f 2>/dev/null` en tu VM, guarda la lista y busca cada binario en GTFOBins para saber cuáles serían peligrosos si se les pusiera SUID.
- Ejercicio en casa (Troubleshooting): instala nginx, ocupa el puerto 80 con `python3 -m http.server 80` y diagnostica el fallo usando solo `systemctl status`, `journalctl -u nginx` y `ss -tulpn`.
- Ejercicio en casa (Common Commands): intenta varios logins SSH fallidos contra tu propia VM y reproduce la cadena `grep | awk | sort | uniq -c | sort -rn` para contar intentos por IP en `auth.log` o en `journalctl -u ssh`.

## Cuadro resumen

Todo lo visto, en una línea por término.

Linux

```
Linux → el kernel; en el habla diaria, el sistema completo.
Kernel → núcleo que controla hardware y permisos.
Userland → todo lo que no es kernel: shell, comandos, bibliotecas, apps.
GNU → proyecto que aportó gran parte del userland.
Distro → kernel + userland + gestor de paquetes + configuración.
Syscall → la puerta por la que un programa le pide algo al kernel.
Por qué en seguridad → está en los objetivos, en las herramientas, es auditable y se automatiza por CLI.
```

Learn following for each

```
GUI → gestor de archivos gráfico; cómodo en escritorio.
CLI → comandos en la shell; único medio en servidores y shells remotas.
/ → raíz única de todo el árbol.
Ruta absoluta → empieza en /; funciona desde cualquier lugar.
Ruta relativa → parte de la carpeta actual.
. / .. / ~ → aquí / arriba / mi carpeta personal.
.archivo → oculto; se ve con ls -a.
FHS → estándar que fija qué va en /etc, /var, /home, etc.
Todo es un archivo → cada recurso tiene ruta y se opera con leer/escribir.
/dev → dispositivos; /proc → procesos; /etc → configuración.
```

```
rwx → leer, escribir, ejecutar.
Dueño / grupo / otros → las tres ternas; se aplica la primera que coincide.
x en carpeta → poder entrar; w en carpeta → crear y borrar dentro.
Octal → r=4, w=2, x=1; 755, 644, 600, 700.
chmod → cambia permisos; chown → cambia dueño (solo root); chgrp → cambia grupo.
umask → resta permisos al crear; 022 da 644/755, 077 da 600/700.
SUID 4000 → corre como el dueño del archivo.
SGID 2000 → corre con el grupo; en carpeta, hereda el grupo.
Sticky 1000 → en carpeta, solo el dueño borra lo suyo (/tmp).
sudo → ejecutar como root con tu contraseña, con reglas y registro.
visudo → única forma segura de editar sudoers.
Escalada de privilegios → pasar de usuario normal a root.
find -perm -4000 → lista binarios SUID.
sudo -l → lista lo que sudo permite.
GTFOBins → catálogo de binarios abusables con SUID o sudo.
Mínimo privilegio → dar solo el permiso necesario.
```

```
journalctl → consulta el diario de systemd; -u servicio, -p prioridad, -b arranque, -f en vivo.
/var/log → logs en texto; auth.log/secure guarda logins y sudo.
systemctl status → estado y últimas líneas de log de un servicio.
enabled → arranca con el sistema; active → corre ahora.
ps / top → procesos y consumo.
kill (15) → pide terminar; kill -9 → fuerza.
df -h → espacio por disco; du -sh → peso por carpeta; df -i → inodos.
free -h → memoria; mirar available.
ip addr / ip route → direcciones y rutas.
ss -tulpn → puertos en escucha y su proceso.
```

```
| → encadena la salida de un comando a la entrada del siguiente.
> / >> / 2> → a archivo sobrescribiendo / añadiendo / solo errores.
ls, cd, pwd, mkdir, cp, mv, rm, touch, ln → manejo de archivos.
file → tipo real por contenido, no por extensión.
cat, less, head, tail -f, wc → leer texto.
find → buscar archivos; grep → buscar texto dentro.
awk, cut, sort, uniq -c, sed, tr → procesar texto.
whoami, id, w, last → identidad y sesiones.
apt / dnf / pacman → gestores de paquetes por familia.
ssh, scp, curl, wget → acceso remoto y transferencia.
tar -czf / -xzf → comprimir / extraer.
sha256sum → verificar integridad con hash.
man → manual oficial del comando.
```
