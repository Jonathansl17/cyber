# Ejercicios: Linux

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

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
   `-G equipo` añade cada usuario al grupo compartido sin cambiar su grupo principal.

2. Crea la carpeta compartida y asígnala al grupo.
   ```bash
   sudo mkdir /srv/compartido
   sudo chgrp equipo /srv/compartido
   ```

3. Pon los permisos especiales: SGID para que los archivos nuevos hereden el grupo, y
   sticky para que cada quien solo borre lo suyo. El `2` delante es SGID, el `1` del
   final (en `rwt`) es sticky; `770` da todo al dueño y al grupo, nada a otros.
   ```bash
   sudo chmod 3770 /srv/compartido
   ls -ld /srv/compartido
   ```
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
   Fíjate en el grupo del archivo: debe ser `equipo`, no `ana`. Eso es el SGID
   funcionando: el archivo heredó el grupo de la carpeta, no el grupo principal de quien
   lo creó.

5. Entra como el segundo usuario, crea su archivo y comprueba que también puede escribir.
   ```bash
   su - luis
   echo "notas de luis" > /srv/compartido/luis.txt
   ls -l /srv/compartido
   ```
   Los dos archivos deben aparecer, ambos con grupo `equipo`.

6. Todavía como `luis`, intenta borrar el archivo del otro usuario.
   ```bash
   rm /srv/compartido/ana.txt
   ```
   Debe fallar con `Operation not permitted`. Eso es el sticky bit: aunque `luis` tiene
   permiso de escritura en la carpeta por el grupo, no puede borrar lo que no es suyo.
   Sal con `exit`.

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
   `-perm -4000` busca el bit SUID; `-type f` limita a archivos. Verás rutas como
   `/usr/bin/passwd`, `/usr/bin/sudo`, `/usr/bin/mount`.

2. Cuenta cuántos hay, para tener una idea del tamaño de la superficie.
   ```bash
   wc -l ~/suid.txt
   ```

3. Para cada binario de la lista, mira solo el nombre (la última parte de la ruta), que
   es lo que buscarás en GTFOBins.
   ```bash
   awk -F/ '{print $NF}' ~/suid.txt | sort -u
   ```

4. Abre [GTFOBins](https://gtfobins.github.io/) en el navegador y busca cada nombre.
   Para cada uno, fíjate si tiene la etiqueta `SUID`: si la tiene, ese binario puede
   abusarse para ejecutar comandos como su dueño (normalmente root) cuando está marcado
   SUID. Por ejemplo, busca `find`: su página muestra que con SUID se puede ejecutar
   una shell con los privilegios del dueño mediante su opción `-exec`.

5. Clasifica tu lista en dos grupos, en el mismo archivo o en uno nuevo. Marca con una
   nota los que aparecen en GTFOBins con etiqueta SUID (peligrosos si se les pone SUID) y
   los que no. Los que vienen SUID de fábrica y son normales (`passwd`, `sudo`, `su`,
   `mount`, `ping`) necesitan ese permiso para su función; lo anómalo sería ver ahí
   `find`, `vim`, `bash`, `cp` o `nmap`.

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

Borra la lista cuando termines: `rm ~/suid.txt`. El ejercicio no cambia ningún permiso
del sistema, solo lee; no hay nada más que deshacer.

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
   Debe mostrar `Active: active (running)`. Si ya está corriendo, párialo para empezar
   limpio: `sudo systemctl stop nginx`.

2. Ocupa el puerto 80 con otro programa, un servidor web de prueba de Python. Déjalo
   corriendo en segundo plano.
   ```bash
   sudo python3 -m http.server 80 &
   ```
   El `&` lo manda al fondo. Ahora el puerto 80 está tomado por Python.

3. Intenta arrancar nginx. Fallará.
   ```bash
   sudo systemctl start nginx
   ```
   El comando devuelve un error de `Job ... failed`. No adivines la causa: investígala
   en orden.

4. Primer paso del diagnóstico: el estado del servicio.
   ```bash
   systemctl status nginx
   ```
   Verás `Active: failed` y, en las últimas líneas, un mensaje que menciona `Address
   already in use`.

5. Segundo paso: el log del servicio, que suele decir exactamente qué pasó.
   ```bash
   journalctl -u nginx --since "5 min ago"
   ```
   Busca la línea `bind() to 0.0.0.0:80 failed (98: Address already in use)`. El log ya
   te dice que el problema es el puerto 80, no nginx en sí.

6. Tercer paso: ¿quién tiene el puerto 80? Aquí se cierra el diagnóstico.
   ```bash
   sudo ss -tulpn | grep ':80'
   ```
   La columna `Process` mostrará `python3` con su PID. Esa es la causa raíz: Python está
   ocupando el puerto que nginx quiere.

7. Arregla la causa, no el síntoma: libera el puerto parando Python y vuelve a arrancar
   nginx.
   ```bash
   sudo kill %1
   sudo systemctl start nginx
   systemctl status nginx
   ```
   `%1` es el trabajo en segundo plano del paso 2 (si lanzaste otros, usa el PID que
   viste en `ss`). Ahora nginx debe quedar `active (running)`.

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
sudo apt purge nginx -y     # o: sudo pacman -Rns nginx
```
Asegúrate de que no quedó ningún `python3 -m http.server` en segundo plano con
`jobs` y, si lo hay, mátalo con `kill`.

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
   sudo systemctl enable --now ssh      # en Arch el servicio se llama sshd
   ip -br addr
   ```

2. Desde la otra máquina (o desde la misma VM apuntando a su propia IP), genera varios
   intentos fallidos usando un usuario que no existe. Repite el comando varias veces y
   escribe cualquier contraseña incorrecta cuando la pida.
   ```bash
   ssh usuariofalso@192.168.56.10
   ```
   Hazlo unas cinco o seis veces para tener datos que contar. Cada intento fallido deja
   una línea en el log del servidor.

3. De vuelta en el servidor, localiza las líneas de contraseña fallida. En Debian/Ubuntu
   están en `/var/log/auth.log`; en distros con journal puro, en `journalctl`.
   ```bash
   sudo grep "Failed password" /var/log/auth.log | tail
   ```
   Si tu distro no tiene `auth.log`, usa:
   `sudo journalctl -u ssh --since "15 min ago" | grep "Failed password"`.

4. Construye la cadena paso a paso. Primero extrae el campo de la IP: en estas líneas la
   IP es el cuarto campo empezando por el final (`$(NF-3)`).
   ```bash
   sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}'
   ```
   Debe salir una IP por línea, una por intento.

5. Añade el conteo y el orden: ordena, cuenta repetidos y vuelve a ordenar de mayor a
   menor.
   ```bash
   sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
   ```
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

Detén el servicio SSH si lo activaste solo para la prueba: `sudo systemctl disable --now
ssh`. Los intentos fallidos quedan en el log como evidencia; si quieres limpiarlos, rota
el log con `sudo logrotate -f /etc/logrotate.conf` en lugar de borrarlo a mano. Desinstala
el servidor SSH si no lo necesitas: `sudo apt purge openssh-server -y`.
