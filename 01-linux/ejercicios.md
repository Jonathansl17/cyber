# Ejercicios: Linux

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Debajo de cada bloque de comandos hay una lista que explica el comando y todas sus
opciones y argumentos. Cuando un comando y una opción ya se explicaron antes en este
mismo archivo, se indica con "(ver ejercicio N)".

## Ejercicio 1: Carpeta de equipo con SGID y sticky bit

Nodo: [Understand Permissions](README.md#understand-permissions), en concreto SGID y el
sticky bit.

Objetivo: crear dos usuarios y un grupo compartido, armar una carpeta donde ambos puedan
escribir pero ninguno pueda borrar los archivos del otro, y comprobar con `ls -l` que
los archivos nuevos heredan el grupo de la carpeta. Al terminar sabrás por qué `/tmp`
protege lo que cada usuario deja dentro.

Necesitas: una VM Linux propia con acceso a `sudo` (Debian, Ubuntu o similar). Todo con
herramientas del sistema base. Tiempo estimado: 20 minutos.

### Pasos

1. Crea el grupo compartido y dos usuarios de prueba.
   ```bash
   sudo groupadd equipo
   sudo useradd -m -s /bin/bash -G equipo ana
   sudo useradd -m -s /bin/bash -G equipo luis
   sudo passwd ana
   sudo passwd luis
   ```
   - `sudo` → ejecuta el comando como root; crear usuarios y grupos exige privilegios.
   - `groupadd` → crea un grupo nuevo.
   - `equipo` → argumento de `groupadd`: el nombre del grupo que se crea.
   - `useradd` → crea una cuenta de usuario.
   - `-m` → crea el directorio personal del usuario (`/home/ana`, `/home/luis`).
   - `-s /bin/bash` → fija la shell de inicio de sesión del usuario en Bash.
   - `-G equipo` → añade al usuario a grupos secundarios (aquí `equipo`) sin cambiar su grupo principal.
   - `ana` / `luis` → último argumento de `useradd`: el nombre del usuario que se crea.
   - `passwd` → establece o cambia la contraseña de un usuario.
   - `ana` / `luis` → argumento de `passwd`: a qué usuario se le pone la contraseña.

2. Crea la carpeta compartida y asígnala al grupo.
   ```bash
   sudo mkdir /srv/compartido
   sudo chgrp equipo /srv/compartido
   ```
   - `sudo` → ver ejercicio 1 (paso 1).
   - `mkdir` → crea un directorio.
   - `/srv/compartido` → argumento de `mkdir`: la ruta del directorio a crear.
   - `chgrp` → cambia el grupo propietario de un archivo o carpeta.
   - `equipo` → primer argumento de `chgrp`: el grupo que pasa a ser dueño.
   - `/srv/compartido` → segundo argumento de `chgrp`: la carpeta cuyo grupo se cambia.

3. Pon los permisos especiales: SGID para que los archivos nuevos hereden el grupo, y
   sticky para que cada quien solo borre lo suyo. El `3` inicial combina SGID (2) y
   sticky (1); `770` da todo al dueño y al grupo, nada a otros.
   ```bash
   sudo chmod 3770 /srv/compartido
   ls -ld /srv/compartido
   ```
   - `sudo` → ver ejercicio 1 (paso 1).
   - `chmod` → cambia los permisos de un archivo o carpeta.
   - `3770` → permisos en octal: el `3` activa SGID+sticky, `7` (dueño) = rwx, `7` (grupo) = rwx, `0` (otros) = nada.
   - `/srv/compartido` → argumento de `chmod`: la carpeta a la que se le aplican.
   - `ls` → lista archivos y carpetas.
   - `-l` → formato largo: muestra permisos, dueño, grupo, tamaño y fecha.
   - `-d` → trata la carpeta como una entrada: muestra la carpeta en sí, no su contenido.
   - `/srv/compartido` → argumento de `ls`: la carpeta a mostrar.

   La salida debe mostrar los bits especiales: `drwxrws--T`. La `s` en la terna del
   grupo es SGID; la `T` al final es el sticky (en mayúscula porque "otros" no tiene
   `x`, lo cual aquí es correcto porque no queremos que otros entren).

4. Crea un archivo dentro actuando como el primer usuario. Se usa `sudo -u` para
   ejecutar cada comando con la identidad de `ana` sin abrir una shell nueva, así el
   bloque se puede pegar entero.
   ```bash
   echo "notas de ana" | sudo -u ana tee /srv/compartido/ana.txt
   ls -l /srv/compartido/ana.txt
   ```
   - `echo "notas de ana"` → imprime ese texto.
   - `|` → tubería: pasa el texto a `tee`.
   - `sudo` → ver ejercicio 1 (paso 1).
   - `-u ana` → opción de `sudo`: ejecuta el comando como el usuario `ana` en lugar de root (con sus grupos, incluido `equipo`).
   - `tee` → escribe lo que recibe en un archivo y también lo muestra en pantalla. Se usa en lugar de `>` porque la redirección `>` la haría tu shell con tu usuario, no `ana`.
   - `/srv/compartido/ana.txt` → argumento de `tee`: el archivo que se crea (con dueño `ana`).
   - `ls -l` → ver ejercicio 1 (paso 3).
   - `/srv/compartido/ana.txt` → argumento de `ls`: el archivo a mostrar.

   Alternativa interactiva: `su - ana` (`su` cambia de usuario, `-` carga su entorno
   como en un login) abre una shell como `ana`; en ella escribes los comandos uno a uno
   (`echo "notas de ana" > /srv/compartido/ana.txt`) y sales con `exit`. No pegues un
   bloque con `su` dentro: `su` abre una shell nueva y las líneas siguientes solo se
   ejecutan cuando sales de ella, ya como tu usuario.

   Fíjate en el grupo del archivo: debe ser `equipo`, no `ana`. Eso es el SGID
   funcionando: el archivo heredó el grupo de la carpeta, no el grupo principal de quien
   lo creó.

5. Como el segundo usuario, crea su archivo y comprueba que también puede escribir.
   ```bash
   echo "notas de luis" | sudo -u luis tee /srv/compartido/luis.txt
   ls -l /srv/compartido
   ```
   - `echo "notas de luis" | sudo -u luis tee /srv/compartido/luis.txt` → ver ejercicio 1 (paso 4); crea el archivo con dueño `luis`.
   - `ls -l` → ver ejercicio 1 (paso 3).
   - `/srv/compartido` → argumento de `ls`: la carpeta cuyo contenido se lista.

   Los dos archivos deben aparecer, ambos con grupo `equipo`.

6. Como `luis`, intenta borrar el archivo del otro usuario.
   ```bash
   sudo -u luis rm /srv/compartido/ana.txt
   ```
   - `sudo -u luis` → ver ejercicio 1 (paso 4); ejecuta `rm` con la identidad de `luis`.
   - `rm` → borra archivos.
   - `/srv/compartido/ana.txt` → argumento de `rm`: el archivo que se intenta borrar (de `ana`).

   Debe fallar con `Operation not permitted`. Eso es el sticky bit: aunque `luis` tiene
   permiso de escritura en la carpeta por el grupo, no puede borrar lo que no es suyo.
   Si usaste la alternativa con `su`, sal con `exit`.

### Resultado esperado

Una carpeta `/srv/compartido` con permisos `drwxrws--T`, dos archivos dentro creados por
usuarios distintos y ambos con grupo `equipo`, y la confirmación de que un usuario no
puede borrar el archivo del otro aunque pueda crear los suyos.

### Comprueba que lo lograste

- ¿Qué grupo tiene `ana.txt` y por qué no es `ana`? Tiene `equipo` por el SGID: los
  archivos creados dentro heredan el grupo de la carpeta.
- ¿Por qué `luis` no pudo borrar `ana.txt` si tiene escritura en la carpeta? Por el
  sticky bit: en una carpeta con sticky, solo el dueño de cada archivo (o root) puede
  borrarlo o renombrarlo.
- ¿Por qué aparece `T` en mayúscula y no `t`? Porque "otros" no tiene permiso de
  ejecución (`x`); el sticky está puesto igual, pero la mayúscula avisa de que falta la
  `x` debajo. Aquí es intencional: los ajenos al grupo no deben ni entrar.

### Limpieza

```bash
sudo rm -rf /srv/compartido
sudo userdel -r ana
sudo userdel -r luis
sudo groupdel equipo
```
- `sudo` → ver ejercicio 1 (paso 1).
- `rm` → ver ejercicio 1 (paso 6).
- `-r` → recursivo: borra la carpeta y todo su contenido.
- `-f` → force: no pide confirmación ni se queja si algo no existe.
- `/srv/compartido` → argumento de `rm`: la carpeta a borrar.
- `userdel` → elimina una cuenta de usuario.
- `-r` → borra también su directorio personal y su correo.
- `ana` / `luis` → argumento de `userdel`: el usuario a eliminar.
- `groupdel` → elimina un grupo.
- `equipo` → argumento de `groupdel`: el grupo a eliminar.

## Ejercicio 2: Inventario de binarios SUID y cotejo con GTFOBins

Nodo: [Los permisos son el mapa de la escalada de privilegios](README.md#los-permisos-son-el-mapa-de-la-escalada-de-privilegios).

Objetivo: listar todos los binarios SUID de tu VM, guardar la lista y revisar cada uno
en GTFOBins para saber cuáles serían peligrosos si estuvieran mal puestos. Al terminar
harás la misma búsqueda que hace un auditor (o un atacante) al entrar a una máquina.

Necesitas: una VM Linux propia. Solo herramientas del sistema base y un navegador para
consultar GTFOBins. Tiempo estimado: 25 minutos. Se practica sobre tu propia VM, nunca
sobre equipos ajenos.

### Pasos

1. Lista todos los archivos con el bit SUID y guárdalos en un archivo. El `2>/dev/null`
   descarta los errores de "permiso denegado" para que solo veas resultados.
   ```bash
   find / -perm -4000 -type f 2>/dev/null | tee ~/suid.txt
   ```
   - `find` → busca archivos y carpetas según criterios.
   - `/` → punto de partida de la búsqueda: la raíz, o sea todo el sistema.
   - `-perm -4000` → filtra por permisos: `-4000` significa "que tenga puesto el bit SUID" (los demás bits dan igual).
   - `-type f` → filtra por tipo: solo archivos regulares, no carpetas ni enlaces.
   - `2>/dev/null` → redirige la salida de errores (descriptor 2) a la nada, descartando los "permiso denegado".
   - `|` → tubería: pasa la lista a `tee`.
   - `tee` → muestra la entrada en pantalla y a la vez la escribe en un archivo.
   - `~/suid.txt` → argumento de `tee`: el archivo donde se guarda la lista (`~` es tu carpeta personal).

   Verás rutas como `/usr/bin/passwd`, `/usr/bin/sudo`, `/usr/bin/mount`.

2. Cuenta cuántos hay, para tener una idea del tamaño de la superficie.
   ```bash
   wc -l ~/suid.txt
   ```
   - `wc` → cuenta líneas, palabras o bytes de un archivo.
   - `-l` → cuenta solo líneas (aquí, un binario SUID por línea).
   - `~/suid.txt` → argumento de `wc`: el archivo a contar.

3. Para cada binario de la lista, mira solo el nombre (la última parte de la ruta), que
   es lo que buscarás en GTFOBins.
   ```bash
   awk -F/ '{print $NF}' ~/suid.txt | sort -u
   ```
   - `awk` → procesa texto por campos y líneas.
   - `-F/` → fija la barra `/` como separador de campos, para partir cada ruta por sus carpetas.
   - `'{print $NF}'` → programa de awk: imprime el último campo (`NF` = número de campos), es decir el nombre del binario.
   - `~/suid.txt` → argumento de `awk`: el archivo a procesar.
   - `|` → tubería (ver ejercicio 2, paso 1).
   - `sort` → ordena líneas.
   - `-u` → unique: elimina duplicados tras ordenar.

4. Abre [GTFOBins](https://gtfobins.github.io/) en el navegador y busca cada nombre.
   Para cada uno, fíjate si tiene la etiqueta `SUID`: si la tiene, ese binario puede
   abusarse para ejecutar comandos como su dueño (normalmente root) cuando está marcado
   SUID. Por ejemplo, busca `find`: su página muestra que con SUID se puede ejecutar
   una shell con los privilegios del dueño mediante su opción `-exec`.

5. Clasifica tu lista en dos grupos, en el mismo archivo o en uno nuevo. Marca los que
   aparecen en GTFOBins con etiqueta SUID (peligrosos si se les pone SUID) y los que no.
   Los que vienen SUID de fábrica y son normales (`passwd`, `sudo`, `su`, `mount`,
   `ping`) necesitan ese permiso para su función; lo anómalo sería ver ahí `find`,
   `vim`, `bash`, `cp` o `nmap`.

6. Para entender por qué importa, compara tu lista con lo que debería ser normal: un
   binario SUID que no reconoces, o uno de uso general como un editor o un intérprete,
   es justo lo que un auditor marca para investigar.

### Resultado esperado

Un archivo `~/suid.txt` con todos los SUID de tu VM y, al lado de cada uno, una nota de
si GTFOBins lo lista como abusable con SUID. Sabrás distinguir los SUID legítimos del
sistema de los que serían una puerta a root si estuvieran mal puestos.

### Comprueba que lo lograste

- ¿Qué concede exactamente el bit SUID a un binario? Que se ejecute con los permisos de
  su dueño (no de quien lo lanza); si el dueño es root, corre como root.
- Si encontraras `/usr/bin/find` con SUID en una máquina, ¿por qué sería grave? Porque
  `find -exec` ejecutaría comandos arbitrarios como root, según GTFOBins.
- ¿Por qué `passwd` sí debe ser SUID root? Porque un usuario normal necesita escribir en
  `/etc/shadow` (que solo root puede tocar) para cambiar su propia contraseña.

### Limpieza

Borra la lista cuando termines: `rm ~/suid.txt` (`rm` borra el archivo que recibe; ver
ejercicio 1, paso 6). El ejercicio no cambia ningún permiso del sistema, solo lee; no hay
nada más que deshacer.

## Ejercicio 3: Diagnostica un puerto ocupado con systemctl, journalctl y ss

Nodo: [Troubleshooting](README.md#troubleshooting).

Objetivo: provocar que nginx no arranque porque el puerto 80 ya está ocupado y
diagnosticar la causa usando solo `systemctl status`, `journalctl -u` y `ss -tulpn`,
sin tocar nada a ciegas. Al terminar seguirás el orden práctico que recomienda la nota.

Necesitas: una VM Linux propia con `sudo` (Debian/Ubuntu). Instalación de nginx: `sudo
apt install nginx` (Debian/Ubuntu) o `sudo pacman -S nginx` (Arch). `python3`, `ss` y
`journalctl` ya vienen. Tiempo estimado: 25 minutos.

### Pasos

1. Instala nginx y comprueba que arranca bien en condiciones normales.
   ```bash
   sudo apt install nginx
   sudo systemctl status nginx
   ```
   - `sudo` → ver ejercicio 1 (paso 1).
   - `apt` → gestor de paquetes de Debian/Ubuntu.
   - `install` → subcomando de `apt`: instala el paquete indicado.
   - `nginx` → el paquete a instalar (el servidor web).
   - `systemctl` → controla los servicios gestionados por systemd.
   - `status` → subcomando de `systemctl`: muestra el estado y las últimas líneas de log de un servicio.
   - `nginx` → argumento de `systemctl`: el servicio a consultar.

   Debe mostrar `Active: active (running)`. Si ya está corriendo, páralo para empezar
   limpio: `sudo systemctl stop nginx` (`stop` detiene el servicio).

2. Ocupa el puerto 80 con otro programa, un servidor web de prueba de Python. Déjalo
   corriendo en segundo plano.
   ```bash
   sudo -v
   sudo python3 -m http.server 80 &
   ```
   - `sudo -v` → validate: pide tu contraseña ahora y deja la credencial en caché unos minutos. Hace falta porque un `sudo` mandado a segundo plano con `&` no puede leer el teclado: se quedaría colgado esperando una contraseña que nunca le llega.
   - `sudo` → ver ejercicio 1 (paso 1); hace falta porque el puerto 80 es privilegiado (< 1024).
   - `python3` → el intérprete de Python 3.
   - `-m http.server` → ejecuta el módulo `http.server` como programa: un servidor web mínimo.
   - `80` → argumento del módulo: el puerto en el que escucha.
   - `&` → manda el proceso a segundo plano para recuperar la terminal.

   Ahora el puerto 80 está tomado por Python.

3. Intenta arrancar nginx. Fallará.
   ```bash
   sudo systemctl start nginx
   ```
   - `sudo` → ver ejercicio 1 (paso 1).
   - `systemctl` → ver ejercicio 3 (paso 1).
   - `start` → subcomando de `systemctl`: intenta arrancar el servicio.
   - `nginx` → el servicio a arrancar.

   El comando devuelve un error de `Job ... failed`. No adivines la causa: investígala
   en orden.

4. Primer paso del diagnóstico: el estado del servicio.
   ```bash
   systemctl status nginx
   ```
   - `systemctl status nginx` → ver ejercicio 3 (paso 1); aquí mostrará `Active: failed`.

   Verás `Active: failed` y, en las últimas líneas, un mensaje que menciona `Address
   already in use`.

5. Segundo paso: el log del servicio, que suele decir exactamente qué pasó.
   ```bash
   journalctl -u nginx --since "5 min ago"
   ```
   - `journalctl` → consulta el diario de logs de systemd.
   - `-u nginx` → unit: filtra solo los mensajes del servicio `nginx`.
   - `--since "5 min ago"` → limita a los mensajes de los últimos 5 minutos.

   Busca la línea `bind() to 0.0.0.0:80 failed (98: Address already in use)`. El log ya
   te dice que el problema es el puerto 80, no nginx en sí.

6. Tercer paso: ¿quién tiene el puerto 80? Aquí se cierra el diagnóstico.
   ```bash
   sudo ss -tulpn | grep ':80'
   ```
   - `sudo` → ver ejercicio 1 (paso 1); sin él no se ve a qué proceso pertenece cada socket.
   - `ss` → muestra sockets (conexiones y puertos en escucha).
   - `-t` → sockets TCP.
   - `-u` → sockets UDP.
   - `-l` → solo los que están en escucha (listening).
   - `-p` → muestra el proceso (nombre y PID) dueño de cada socket.
   - `-n` → numeric: muestra puertos como número, sin traducir a nombre de servicio.
   - `|` → tubería (ver ejercicio 2, paso 1).
   - `grep` → filtra líneas por un patrón.
   - `':80'` → patrón que busca `grep`: deja solo las líneas del puerto 80.

   La columna `Process` mostrará `python3` con su PID. Esa es la causa raíz: Python está
   ocupando el puerto que nginx quiere.

7. Arregla la causa, no el síntoma: libera el puerto parando Python y vuelve a arrancar
   nginx.
   ```bash
   sudo pkill -f '[h]ttp.server 80'
   sudo systemctl start nginx
   systemctl status nginx
   ```
   - `sudo` → ver ejercicio 1 (paso 1); el Python corre como root, así que solo root puede pararlo.
   - `pkill` → envía una señal (por defecto SIGTERM, que le pide terminar) a los procesos cuyo nombre coincide con un patrón.
   - `-f` → full: compara el patrón con la línea de comandos completa, no solo con el nombre `python3`, para no matar otros Python.
   - `'[h]ttp.server 80'` → el patrón (una expresión regular): solo el servidor de prueba del paso 2. Los corchetes `[h]` equivalen a `h`, pero hacen que el patrón no coincida con su propio texto; así `pkill` no se lleva por delante al proceso `sudo` que lo lanzó, cuya línea de comandos contiene literalmente `[h]ttp.server`.
   - Alternativa equivalente: `sudo kill <PID>` con el PID que viste en la columna `Process` de `ss`. No sirve `sudo kill %1`: `%1` es un número de trabajo que solo entiende tu shell, y `sudo` lanza `kill` como un programa aparte que no lo conoce.
   - `sudo systemctl start nginx` → ver ejercicio 3 (paso 3); ahora sí debe arrancar.
   - `systemctl status nginx` → ver ejercicio 3 (paso 1); confirma `active (running)`.

   Ahora nginx debe quedar `active (running)`.

### Resultado esperado

Una secuencia de diagnóstico que va del estado (`failed`), al log (`Address already in
use`), a quién tiene el puerto (`python3` en `ss`), y termina liberando el puerto para
que nginx arranque. No reiniciaste nada "por si acaso": cada comando confirmó una cosa.

### Comprueba que lo lograste

- ¿Qué comando te dijo la causa exacta sin tener que adivinar? `journalctl -u nginx`, con
  el mensaje `bind() to 0.0.0.0:80 failed`.
- ¿Qué columna de `ss -tulpn` identifica al culpable? La columna `Process`, que muestra
  el nombre y el PID del programa que escucha en el puerto.
- ¿Por qué `ss -tulpn` necesita `sudo` para ver el proceso? Porque sin privilegios no
  puede leer a qué proceso pertenece cada socket de otros usuarios.

### Limpieza

```bash
sudo systemctl stop nginx
sudo apt purge nginx -y
```
- `sudo systemctl stop nginx` → ver ejercicio 3 (paso 1); detiene el servicio.
- `apt` → ver ejercicio 3 (paso 1).
- `purge` → subcomando de `apt`: desinstala el paquete y borra también su configuración.
- `nginx` → el paquete a eliminar.
- `-y` → responde "sí" automáticamente a la confirmación.

En Arch sería `sudo pacman -Rns nginx` (`-R` elimina, `n` borra la configuración, `s`
quita dependencias que queden huérfanas). Asegúrate de que no quedó ningún `python3 -m
http.server` en segundo plano con `jobs` (lista los trabajos de la shell) o con `pgrep
-af '[h]ttp.server'` (`pgrep` busca procesos por patrón, `-a` muestra su línea de comandos y
`-f` compara con ella) y, si lo hay, mátalo con `sudo pkill -f '[h]ttp.server 80'` (ver
paso 7).

## Ejercicio 4: Cuenta intentos de login fallidos por IP en los logs

Nodo: [Common Commands](README.md#common-commands), en concreto la cadena
`grep | awk | sort | uniq -c | sort -rn`.

Objetivo: generar varios intentos de login SSH fallidos contra tu propia VM y reconstruir
la cadena de comandos que cuenta cuántos intentos vinieron de cada IP, ordenados de mayor
a menor. Al terminar sabrás leer un ataque de fuerza bruta en los logs.

Necesitas: una VM Linux propia con el servicio SSH instalado y corriendo, y otra máquina
(o la misma VM) desde donde lanzar los intentos. Instalación del servidor SSH: `sudo apt
install openssh-server` (Debian/Ubuntu) o `sudo pacman -S openssh` (Arch). Red aislada
host-only recomendada, ya que vas a generar tráfico de login fallido; todo contra tu
propia VM. Tiempo estimado: 25 minutos.

### Pasos

1. En la VM que hace de servidor, asegúrate de que SSH está activo y anota su IP.
   ```bash
   sudo systemctl enable --now ssh
   ip -br addr
   ```
   - `sudo` → ver ejercicio 1 (paso 1).
   - `systemctl` → ver ejercicio 3 (paso 1).
   - `enable` → subcomando de `systemctl`: marca el servicio para arrancar en cada inicio del sistema.
   - `--now` → además de habilitarlo, lo arranca ya mismo.
   - `ssh` → el servicio (en Arch se llama `sshd`).
   - `ip` → ver ejercicio 1 del archivo 01; herramienta de red.
   - `-br` → brief: salida resumida, una línea por interfaz.
   - `addr` → subcomando de `ip`: muestra las direcciones de las interfaces.

2. Desde la otra máquina (o desde la misma VM apuntando a su propia IP), genera varios
   intentos fallidos usando un usuario que no existe. Repite el comando varias veces y
   escribe cualquier contraseña incorrecta cuando la pida.
   ```bash
   ssh usuariofalso@192.168.56.10
   ```
   - `ssh` → cliente de shell remota cifrada.
   - `usuariofalso@192.168.56.10` → argumento de `ssh`: el usuario (`usuariofalso`, que no existe) y la IP del servidor, separados por `@`.

   Hazlo unas cinco o seis veces para tener datos que contar. Cada intento fallido deja
   una línea en el log del servidor.

3. De vuelta en el servidor, localiza las líneas de contraseña fallida. En Debian/Ubuntu
   están en `/var/log/auth.log`; en distros con journal puro, en `journalctl`.
   ```bash
   sudo grep "Failed password" /var/log/auth.log | tail
   ```
   - `sudo` → ver ejercicio 1 (paso 1); el log de autenticación suele ser legible solo por root.
   - `grep` → ver ejercicio 3 (paso 6); filtra líneas por patrón.
   - `"Failed password"` → patrón que busca `grep`: las líneas de contraseña fallida.
   - `/var/log/auth.log` → argumento de `grep`: el archivo donde busca.
   - `|` → tubería (ver ejercicio 2, paso 1).
   - `tail` → muestra solo las últimas líneas (por defecto 10).

   Si tu distro no tiene `auth.log`, usa:
   `sudo journalctl -u ssh --since "15 min ago" | grep "Failed password"`
   (`journalctl -u ssh` ver ejercicio 3 paso 5; `--since` limita el tiempo).

4. Construye la cadena paso a paso. Primero extrae el campo de la IP: en estas líneas la
   IP es el cuarto campo empezando por el final (`$(NF-3)`).
   ```bash
   sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}'
   ```
   - `sudo grep "Failed password" /var/log/auth.log` → ver ejercicio 4 (paso 3).
   - `|` → tubería (ver ejercicio 2, paso 1).
   - `awk` → ver ejercicio 2 (paso 3); procesa texto por campos.
   - `'{print $(NF-3)}'` → programa de awk: imprime el cuarto campo contando desde el final (`NF` es el total de campos), que es la IP de origen.

   Debe salir una IP por línea, una por intento.

5. Añade el conteo y el orden: ordena, cuenta repetidos y vuelve a ordenar de mayor a
   menor.
   ```bash
   sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
   ```
   - `sudo grep ... | awk '{print $(NF-3)}'` → ver ejercicio 4 (paso 4).
   - `sort` → ver ejercicio 2 (paso 3); ordena las IP para que las iguales queden juntas.
   - `uniq` → colapsa líneas repetidas contiguas.
   - `-c` → count: antepone a cada línea cuántas veces se repitió.
   - `sort` (segundo) → ordena de nuevo, ahora por el conteo.
   - `-r` → reverse: orden descendente.
   - `-n` → numeric: ordena por valor numérico, no alfabético.
   - `head` → muestra solo las primeras líneas (por defecto 10): las IP con más intentos.

   La salida es el número de intentos y la IP, lo más alto primero:
   ```
        6 192.168.56.1
        1 192.168.56.20
   ```

6. Interpreta el resultado. Cada eslabón de la cadena hace una cosa: `grep` filtra las
   líneas de fallo, `awk` saca la IP, `sort` las junta, `uniq -c` cuenta repetidas (por
   eso hace falta ordenar antes), `sort -rn` ordena por número descendente y `head` deja
   las primeras. Muchos intentos desde una sola IP en poco tiempo es la firma de fuerza
   bruta.

### Resultado esperado

Una tabla con el número de intentos fallidos por cada IP, ordenada de mayor a menor,
reconstruida eslabón a eslabón, y la comprensión de qué hace cada comando de la cadena.

### Comprueba que lo lograste

- ¿Por qué hay que hacer `sort` antes de `uniq -c`? Porque `uniq` solo agrupa líneas
  iguales si están contiguas; sin ordenar antes, contaría mal.
- ¿Qué hace `$(NF-3)` en `awk` y por qué no un número fijo como `$11`? `NF` es el número
  de campos de la línea; contar desde el final es más robusto porque el inicio de la
  línea (fecha, host) puede variar de longitud.
- Si una sola IP tuviera cientos de intentos en un minuto, ¿qué concluyes? Que es un
  ataque de fuerza bruta automatizado, no un usuario que se equivocó de contraseña.

### Limpieza

Detén el servicio SSH si lo activaste solo para la prueba:
```bash
sudo systemctl disable --now ssh
```
- `sudo systemctl` → ver ejercicio 3 (paso 1).
- `disable` → subcomando de `systemctl`: quita el arranque automático del servicio.
- `--now` → además lo detiene ya mismo.
- `ssh` → el servicio.

Los intentos fallidos quedan en el log como evidencia; si quieres limpiarlos, rota el log
con `sudo logrotate -f /etc/logrotate.conf` (`logrotate` rota logs, `-f` fuerza la
rotación aunque no toque todavía, y el argumento es su archivo de configuración) en lugar
de borrarlo a mano. Desinstala el servidor SSH si no lo necesitas con `sudo apt purge
openssh-server -y` (ver ejercicio 3, limpieza).

## Ejercicio 5: Audita tareas programadas, PATH y capabilities

Nodo: [Tareas programadas, PATH y capabilities](README.md#tareas-programadas-path-y-capabilities).

Objetivo: crear un timer de systemd de prueba para ver cómo se programa una tarea, y
luego auditar tu propia máquina como lo haría un defensor: qué tareas programadas corren
como root, si el `PATH` tiene carpetas peligrosas y qué binarios tienen capabilities o
SUID. Al terminar sabrás revisar los sitios donde un permiso de más termina en root.

Necesitas: una VM Linux propia con `sudo` (Debian, Ubuntu o similar; en Arch funciona
igual salvo cron, que no viene instalado). `getcap` viene en el paquete `libcap2-bin`
(Debian/Ubuntu) o `libcap` (Arch), normalmente ya instalado. Tiempo estimado: 25
minutos. Solo se audita tu propia máquina; el único cambio es el timer de prueba, que se
borra al final.

### Pasos

1. Crea un timer de usuario que escriba la fecha en un archivo cada minuto. No hace falta
   `sudo`: es un timer de tu usuario.
   ```bash
   systemd-run --user --unit=prueba-timer --on-calendar='*:*:00' /bin/sh -c 'date >> /tmp/prueba-timer.log'
   ```
   - `systemd-run` → crea y arranca una unidad de systemd temporal (transitoria) sin escribir archivos de unidad a mano.
   - `--user` → la crea en el systemd de tu usuario, no en el del sistema (no necesita root).
   - `--unit=prueba-timer` → nombre de la unidad; se crean `prueba-timer.timer` y `prueba-timer.service`.
   - `--on-calendar='*:*:00'` → cuándo se dispara, en formato `OnCalendar`: cualquier hora, cualquier minuto, en el segundo 00, o sea cada minuto.
   - `/bin/sh` → el programa que ejecuta el servicio: una shell.
   - `-c 'date >> /tmp/prueba-timer.log'` → opción de `sh`: ejecuta ese texto como comando; `date` imprime la fecha y `>>` la añade al final del archivo.

   Debe responder `Running timer as unit: prueba-timer.timer` y `Will run service as
   unit: prueba-timer.service`.

2. Mira el timer y su definición.
   ```bash
   systemctl --user list-timers
   systemctl --user cat prueba-timer.timer
   ```
   - `systemctl` → controla las unidades de systemd (ver ejercicio 3, paso 1).
   - `--user` → actúa sobre el systemd de tu usuario.
   - `list-timers` → subcomando: lista los timers con su próxima ejecución (`NEXT`), cuánto falta (`LEFT`) y la última (`LAST`).
   - `cat` → subcomando: muestra el archivo de la unidad.
   - `prueba-timer.timer` → la unidad a mostrar.

   En `list-timers` verás `prueba-timer.timer` con `LEFT` de menos de un minuto. En
   `cat` verás la sección `[Timer]` con `OnCalendar=*:*:00`. Espera un par de minutos y
   comprueba que funciona con `cat /tmp/prueba-timer.log` (`cat` muestra el archivo):
   debe haber una línea de fecha por minuto.

3. Ahora audita: lista los timers del sistema (los que corren como root) y las tareas
   cron.
   ```bash
   systemctl list-timers --all
   crontab -l
   ls -l /etc/crontab /etc/cron.d /etc/cron.daily /etc/cron.hourly
   ```
   - `systemctl list-timers` → ver paso 2; sin `--user` muestra los timers del sistema.
   - `--all` → incluye también los timers inactivos.
   - `crontab` → gestiona la tabla cron de tu usuario.
   - `-l` → list: la muestra. Si no tienes ninguna, responde `no crontab for <usuario>`.
   - `ls -l` → ver ejercicio 1 (paso 3).
   - `/etc/crontab /etc/cron.d /etc/cron.daily /etc/cron.hourly` → argumentos de `ls`: la tabla del sistema y las carpetas con tareas del sistema.

   Verás timers normales como `logrotate.timer`, `apt-daily.timer` o `fstrim.timer`, y
   en `/etc/cron.daily` scripts como `logrotate` o `dpkg`. Todos deben ser de `root` y
   no escribibles por otros (`-rwxr-xr-x`). Si `cron` no está instalado (Arch), `crontab`
   y `/etc/cron*` no existen: es normal, ahí todo va por timers.

4. Para un timer de root, averigua qué ejecuta y comprueba quién puede modificarlo. Usa
   uno de la lista del paso 3, por ejemplo `logrotate`.
   ```bash
   systemctl cat logrotate.service | grep ExecStart
   ls -l /usr/sbin/logrotate
   ```
   - `systemctl cat` → ver paso 2.
   - `logrotate.service` → el servicio que activa el timer (columna `ACTIVATES`).
   - `|` → tubería (ver ejercicio 2, paso 1).
   - `grep ExecStart` → deja solo la línea que dice qué programa ejecuta el servicio.
   - `ls -l` → ver ejercicio 1 (paso 3).
   - `/usr/sbin/logrotate` → el programa que aparece en `ExecStart`.

   Debe salir algo como `ExecStart=/usr/sbin/logrotate /etc/logrotate.conf` y el binario
   con dueño `root` y permisos `-rwxr-xr-x`. Lo grave sería una `w` en el grupo o en
   otros: quien pudiera escribirlo ejecutaría código como root en la próxima vuelta.

5. Revisa tu `PATH`: una entrada por línea, en el orden en que se buscan.
   ```bash
   echo "$PATH" | tr ':' '\n'
   ```
   - `echo "$PATH"` → imprime el valor de la variable `PATH`; las comillas evitan que la shell la parta.
   - `|` → tubería (ver ejercicio 2, paso 1).
   - `tr` → traduce caracteres.
   - `':' '\n'` → argumentos de `tr`: cambia cada `:` por un salto de línea.

   Busca dos cosas que no deberían estar: una línea `.` o una línea vacía (las dos
   significan "la carpeta actual"), y carpetas de otros usuarios o de `/tmp`.

6. Comprueba si alguna carpeta del `PATH` es escribible por cualquier usuario, que es la
   condición del PATH hijacking.
   ```bash
   echo "$PATH" | tr ':' '\n' | xargs -I{} find {} -maxdepth 0 -perm -o+w 2>/dev/null
   sudo grep secure_path /etc/sudoers
   ```
   - `echo "$PATH" | tr ':' '\n'` → ver paso 5.
   - `xargs` → ejecuta un comando por cada línea que recibe.
   - `-I{}` → opción de `xargs`: sustituye `{}` por la línea (aquí, cada carpeta del PATH).
   - `find {}` → busca empezando en esa carpeta (ver ejercicio 2, paso 1).
   - `-maxdepth 0` → no entra en la carpeta: evalúa solo la carpeta en sí.
   - `-perm -o+w` → que tenga puesto el permiso de escritura para "otros".
   - `2>/dev/null` → descarta los errores de carpetas que no existen (ver ejercicio 2, paso 1).
   - `sudo grep` → busca texto con privilegios, porque `/etc/sudoers` solo lo lee root.
   - `secure_path` → el patrón: la opción que fija el `PATH` con el que `sudo` ejecuta los comandos.
   - `/etc/sudoers` → el archivo de reglas de sudo.

   La primera línea no debería imprimir nada; cada carpeta que salga es escribible por
   cualquiera y se debe corregir con `chmod o-w <carpeta>`. La segunda debe mostrar algo
   como `Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:..."`:
   aunque tu `PATH` tenga carpetas tuyas, `sudo` no las usa.

7. Lista los binarios con capabilities de todo el sistema.
   ```bash
   getcap -r / 2>/dev/null
   ```
   - `getcap` → muestra las capabilities de archivos.
   - `-r` → recursivo: recorre todo lo que hay bajo la ruta dada.
   - `/` → la ruta de partida: todo el sistema.
   - `2>/dev/null` → descarta los "permiso denegado" (ver ejercicio 2, paso 1).

   Verás pocas líneas, por ejemplo `/usr/bin/ping cap_net_raw=ep` o
   `/usr/bin/mtr-packet cap_net_raw=ep`. Son normales: solo pueden fabricar paquetes de
   red. Lo anómalo sería `cap_setuid`, `cap_dac_override` o `cap_sys_admin` en un
   intérprete, un editor o un programa que no reconoces.

8. Completa el cuadro con los SUID y SGID (el ejercicio 2 cubre solo SUID).
   ```bash
   find / -perm /6000 -type f 2>/dev/null
   ```
   - `find /` → ver ejercicio 2 (paso 1).
   - `-perm /6000` → filtra por permisos: con `/` basta con que tenga cualquiera de los bits indicados, aquí SUID (4000) o SGID (2000).
   - `-type f` → solo archivos regulares (ver ejercicio 2, paso 1).
   - `2>/dev/null` → ver ejercicio 2 (paso 1).

   Además de la lista del ejercicio 2 aparecerán binarios SGID como `/usr/bin/crontab`
   (grupo `crontab`) o `/usr/bin/chage` (grupo `shadow`) en Debian/Ubuntu, que lo
   necesitan para su función.

### Resultado esperado

Un timer de prueba funcionando que escribe una fecha por minuto, y una auditoría de tu
máquina con cuatro respuestas: qué tareas programadas corren como root y si sus programas
son solo de root; si el `PATH` tiene `.`, entradas vacías o carpetas escribibles por
otros; qué binarios tienen capabilities y si alguna equivale a root; y qué binarios son
SUID o SGID.

### Comprueba que lo lograste

- ¿Qué dos unidades necesita un timer de systemd y qué dice cada una? El `.timer` dice
  cuándo (`OnCalendar`) y el `.service` dice qué se ejecuta (`ExecStart`).
- ¿Por qué es peligrosa una tarea de root que ejecuta un script escribible por otros?
  Porque quien edite el script ejecuta su código como root en la siguiente ejecución.
- ¿Qué condición hace falta para un PATH hijacking? Que un programa privilegiado llame a
  un comando sin ruta absoluta y que en su `PATH`, antes de la carpeta legítima, haya una
  carpeta (o `.`) donde el atacante pueda escribir.
- ¿Por qué `cap_net_raw` en `ping` es aceptable y `cap_setuid` en `python3` no? La
  primera solo permite fabricar paquetes; la segunda permite cambiar el UID a 0, o sea
  volverse root.

### Limpieza

```bash
systemctl --user stop prueba-timer.timer
rm /tmp/prueba-timer.log
```
- `systemctl --user` → ver paso 2.
- `stop` → subcomando: detiene la unidad. Al ser transitoria, desaparece al pararla (y en cualquier caso al reiniciar).
- `prueba-timer.timer` → el timer a detener.
- `rm` → ver ejercicio 1 (paso 6).
- `/tmp/prueba-timer.log` → el archivo que generó el timer.

Si el servicio quedó en estado `failed`, bórralo de la lista con `systemctl --user
reset-failed prueba-timer.service` (`reset-failed` limpia el estado de error de la
unidad). La auditoría solo leyó; no hay nada más que deshacer.
