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

4. Entra como el primer usuario y crea un archivo dentro.
   ```bash
   su - ana
   echo "notas de ana" > /srv/compartido/ana.txt
   ls -l /srv/compartido/ana.txt
   exit
   ```
   - `su` → cambia a otro usuario abriendo una shell con su identidad.
   - `-` → login shell: carga el entorno del usuario destino como si iniciara sesión.
   - `ana` → argumento de `su`: el usuario al que se cambia.
   - `echo "notas de ana"` → imprime ese texto.
   - `>` → redirección: escribe la salida en un archivo, sobrescribiéndolo.
   - `/srv/compartido/ana.txt` → el archivo donde se guarda el texto.
   - `ls -l` → ver ejercicio 1 (paso 3).
   - `/srv/compartido/ana.txt` → argumento de `ls`: el archivo a mostrar.
   - `exit` → cierra la shell de `ana` y vuelve a tu usuario.

   Fíjate en el grupo del archivo: debe ser `equipo`, no `ana`. Eso es el SGID
   funcionando: el archivo heredó el grupo de la carpeta, no el grupo principal de quien
   lo creó.

5. Entra como el segundo usuario, crea su archivo y comprueba que también puede escribir.
   ```bash
   su - luis
   echo "notas de luis" > /srv/compartido/luis.txt
   ls -l /srv/compartido
   ```
   - `su - luis` → ver ejercicio 1 (paso 4); cambia al usuario `luis`.
   - `echo "notas de luis" > /srv/compartido/luis.txt` → ver ejercicio 1 (paso 4); crea el archivo de `luis`.
   - `ls -l` → ver ejercicio 1 (paso 3).
   - `/srv/compartido` → argumento de `ls`: la carpeta cuyo contenido se lista.

   Los dos archivos deben aparecer, ambos con grupo `equipo`.

6. Todavía como `luis`, intenta borrar el archivo del otro usuario.
   ```bash
   rm /srv/compartido/ana.txt
   ```
   - `rm` → borra archivos.
   - `/srv/compartido/ana.txt` → argumento de `rm`: el archivo que se intenta borrar (de `ana`).

   Debe fallar con `Operation not permitted`. Eso es el sticky bit: aunque `luis` tiene
   permiso de escritura en la carpeta por el grupo, no puede borrar lo que no es suyo.
   Sal con `exit` (ver ejercicio 1, paso 4).

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
  `x` debajo.

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

   Debe mostrar `Active: active (running)`. Si ya está corriendo, párialo para empezar
   limpio: `sudo systemctl stop nginx` (`stop` detiene el servicio).

2. Ocupa el puerto 80 con otro programa, un servidor web de prueba de Python. Déjalo
   corriendo en segundo plano.
   ```bash
   sudo python3 -m http.server 80 &
   ```
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
   sudo kill %1
   sudo systemctl start nginx
   systemctl status nginx
   ```
   - `sudo` → ver ejercicio 1 (paso 1).
   - `kill` → envía una señal a un proceso (por defecto SIGTERM, que le pide terminar).
   - `%1` → identifica el trabajo número 1 en segundo plano de esta shell (el Python del paso 2); si lanzaste otros, usa el PID que viste en `ss`.
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
http.server` en segundo plano con `jobs` (lista los trabajos de la shell) y, si lo hay,
mátalo con `kill` (ver paso 7).

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
