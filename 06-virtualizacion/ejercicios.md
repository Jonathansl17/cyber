# Ejercicios: Virtualización

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

## Ejercicio 1: Rompe y restaura una VM con un snapshot

Nodo: [Snapshots para labs](README.md#snapshots-para-labs), con los modos de red de [VM](README.md#vm).

Objetivo: dejar dos VMs que solo se ven entre sí en una red interna, tomar un snapshot `limpio` de la víctima, dejarla inservible borrando `/etc/passwd` y volver al estado sano midiendo cuántos segundos tarda la restauración y cuántos hasta que la víctima vuelve a responder en la red.

Necesitas: VirtualBox 7 en el host (Debian/Ubuntu: `sudo apt install virtualbox`; Arch: `sudo pacman -S virtualbox virtualbox-host-modules-arch`; Windows: instalador de virtualbox.org) y dos VMs Linux ya instaladas, llamadas `atacante` y `victima` (por ejemplo Kali y Debian 12; sirve cualquier distribución). En la víctima, un usuario con `sudo`. La red será "internal", sin salida a internet ni a tu casa. Tiempo: 40 minutos más la instalación de las VMs si no las tienes.

### Pasos

1. Apaga las dos VMs y comprueba que existen con esos nombres.
   ```bash
   VBoxManage list vms
   VBoxManage list runningvms
   ```
   - `VBoxManage` → herramienta de línea de comandos de VirtualBox; todo lo que hace la interfaz gráfica se puede hacer con ella.
   - `list vms` → lista las VMs registradas, con su nombre entre comillas y su UUID.
   - `list runningvms` → lista solo las VMs que están encendidas en este momento.

   La primera lista debe incluir `"atacante"` y `"victima"`; la segunda debe salir vacía (si no, apágalas desde dentro con `sudo poweroff`).
2. Conecta la primera tarjeta de red de ambas a una red interna llamada `labnet`.
   ```bash
   VBoxManage modifyvm atacante --nic1=intnet --intnet1=labnet
   VBoxManage modifyvm victima --nic1=intnet --intnet1=labnet
   ```
   - `modifyvm` → cambia la configuración de una VM apagada.
   - `atacante` / `victima` → nombre de la VM que se modifica (también vale su UUID).
   - `--nic1=intnet` → pone la tarjeta de red número 1 en modo red interna (`intnet`), que solo conecta VMs entre sí.
   - `--intnet1=labnet` → nombre de la red interna a la que se conecta la tarjeta 1; las VMs con el mismo nombre de red se ven entre sí.

3. Una red interna no tiene DHCP. Crea un servidor DHCP de VirtualBox para `labnet`, así las VMs reciben IP sin configurar nada dentro.
   ```bash
   VBoxManage dhcpserver add --network=labnet --server-ip=10.10.10.1 --netmask=255.255.255.0 \
     --lower-ip=10.10.10.100 --upper-ip=10.10.10.200 --enable
   ```
   - `dhcpserver add` → crea un servidor DHCP integrado en VirtualBox.
   - `--network=labnet` → red interna a la que da servicio; es el mismo nombre que pusiste con `--intnet1`.
   - `--server-ip=10.10.10.1` → IP que usa el propio servidor DHCP dentro de esa red.
   - `--netmask=255.255.255.0` → máscara de red que entrega a los clientes (una /24).
   - `--lower-ip=10.10.10.100` → primera IP del rango que reparte.
   - `--upper-ip=10.10.10.200` → última IP del rango que reparte.
   - `--enable` → deja el servidor activado.
   - `\` → continúa el comando en la línea siguiente.

   Si responde que ya existe, ya estaba creado de otra práctica; sigue.
4. Arranca las dos VMs (con ventana, para poder iniciar sesión en ellas).
   ```bash
   VBoxManage startvm atacante
   VBoxManage startvm victima
   ```
   - `startvm` → arranca la VM indicada; sin `--type` usa el modo por defecto, con ventana gráfica (`gui`).
   - `atacante` / `victima` → nombre de la VM que se arranca.

5. Dentro de la víctima, inicia sesión y anota su IP.
   ```bash
   ip -4 -br addr
   ```
   - `ip` → herramienta de configuración de red de Linux.
   - `-4` → solo muestra direcciones IPv4.
   - `-br` → formato breve: una línea por interfaz con nombre, estado y direcciones.
   - `addr` → objeto a consultar: las direcciones de las interfaces (`ip address show`).

   Verás algo como `enp0s3  UP  10.10.10.101/24`. A partir de aquí la llamamos `IP_VICTIMA`.
6. Dentro del atacante, comprueba que la red interna funciona y que está aislada.
   ```bash
   ping -c 3 10.10.10.101
   ping -c 3 -W 2 1.1.1.1
   ```
   - `ping` → manda paquetes ICMP echo request y espera los echo reply.
   - `-c 3` → manda 3 paquetes y termina.
   - `10.10.10.101` → IP de la víctima (cámbiala por la tuya).
   - `-W 2` → espera como máximo 2 segundos cada respuesta antes de darla por perdida.
   - `1.1.1.1` → una IP de internet, para comprobar que no hay salida.

   El primero debe responder; el segundo debe fallar (`100% packet loss` o `Network is unreachable`): no hay salida a internet.
7. Con la víctima encendida y sana, toma el snapshot desde el host.
   ```bash
   VBoxManage snapshot victima take limpio --description="sana, antes de romper /etc/passwd"
   VBoxManage snapshot victima list
   ```
   - `snapshot victima` → opera sobre los snapshots de la VM `victima`.
   - `take limpio` → crea un snapshot nuevo llamado `limpio`. Como no lleva `--live`, VirtualBox pausa la VM un momento y guarda también la RAM, así que al restaurar volverá encendida y con la sesión abierta.
   - `--description="..."` → texto descriptivo que se guarda con el snapshot y se ve en la interfaz gráfica.
   - `list` → lista los snapshots de la VM.

   La lista debe mostrar `Name: limpio (UUID: ...) *`; el asterisco marca el snapshot actual.
8. En el atacante, deja un ping continuo hacia la víctima para medir luego el corte. Añade la hora a cada línea:
   ```bash
   ping -D 10.10.10.101
   ```
   - `-D` → antepone la marca de tiempo Unix `[1759400000.123456]` (segundos desde 1970) a cada respuesta.
   - `10.10.10.101` → IP de la víctima. Sin `-c`, el ping no termina hasta que lo detienes.

   No lo detengas.
9. Rompe la víctima. Dentro de ella:
   ```bash
   sudo cp /etc/passwd /tmp/passwd.antes
   sudo rm /etc/passwd
   whoami
   ls -l /home
   sudo -v
   ```
   - `sudo` → ejecuta el comando como root.
   - `cp /etc/passwd /tmp/passwd.antes` → copia el archivo de usuarios a `/tmp` con otro nombre.
   - `rm /etc/passwd` → borra el archivo que asocia nombres de usuario con sus UID, shells y directorios personales.
   - `whoami` → imprime el nombre del usuario actual; lo busca en `/etc/passwd` a partir de tu UID.
   - `ls -l /home` → lista `/home`; `-l` usa el formato largo, con dueño y grupo de cada entrada.
   - `sudo -v` → no ejecuta nada: solo valida (pide tu contraseña y renueva el tiempo de gracia de sudo). Sirve para comprobar si sudo te reconoce.

   `whoami` falla con `cannot find name for user ID 1000`, `ls -l` muestra números en lugar de nombres de usuario y `sudo` responde `you do not exist in the passwd database`. Ya no puedes administrar la máquina ni iniciar sesión de nuevo: está rota. Abre otra consola de texto de la VM (tecla Host + F3; la tecla Host es por defecto el Ctrl derecho) e intenta entrar con tu usuario para confirmarlo.
10. Desde el host, apaga la VM, restaura el snapshot y vuelve a arrancarla, midiendo el tiempo total. VirtualBox exige que la VM esté apagada para restaurar.
    ```bash
    time ( VBoxManage controlvm victima poweroff && \
           VBoxManage snapshot victima restore limpio && \
           VBoxManage startvm victima )
    ```
    - `time ( ... )` → ejecuta todo lo de dentro del paréntesis (una subshell) y al terminar muestra cuánto tardó: `real` es el tiempo de reloj.
    - `controlvm victima poweroff` → apaga la VM en seco, como quitarle la corriente, sin avisar al sistema invitado.
    - `&&` → ejecuta el siguiente comando solo si el anterior terminó bien.
    - `snapshot victima restore limpio` → devuelve la VM al estado del snapshot `limpio` y descarta todo lo posterior.
    - `startvm victima` → (ver paso 4).
    - `\` → continúa el comando en la línea siguiente.

    Al final verás `real 0m4,812s` o similar: ese es el tiempo de la operación de restauración.
11. Mira el ping del atacante. Habrá un hueco en la secuencia (`icmp_seq` salta) entre la última respuesta antes del apagado y la primera después de restaurar. Resta las dos marcas de tiempo `-D`: es el tiempo real sin servicio. Detén el ping con `Ctrl+C`; el resumen dirá cuántos paquetes se perdieron (cada paquete perdido es un segundo).
12. En la ventana de la víctima, que vuelve con la sesión que tenías al tomar el snapshot, comprueba que está sana.
    ```bash
    whoami
    ls -l /etc/passwd /tmp/passwd.antes
    ```
    - `whoami` → (ver paso 9).
    - `ls -l` → (ver paso 9); con varios archivos como argumento, muestra cada uno o un error si no existe.
    - `/etc/passwd /tmp/passwd.antes` → los dos archivos cuya existencia se comprueba.

    `whoami` vuelve a dar tu usuario, `/etc/passwd` existe, y `/tmp/passwd.antes` tampoco existe: todo lo que hiciste después del snapshot desapareció, incluida la copia.
13. Repite los pasos 9 a 11 una segunda vez y anota los dos tiempos. Restaurar no depende de cuánto dañes la VM: solo se descarta el archivo diferencial.

> [!NOTE]
> Con KVM/libvirt (virt-manager) el flujo es el mismo con `virsh`: red interna = una red de libvirt sin `<forward>` (aislada); snapshot con `virsh snapshot-create-as victima limpio`; restauración con `virsh snapshot-revert victima limpio --running`. Requiere discos `qcow2`.

### Resultado esperado

Dos tiempos anotados por intento: la duración de `poweroff + restore + startvm` (segundos) y el hueco del ping (segundos sin respuesta). Debería quedar claro que volver a un estado sano cuesta segundos, frente a reinstalar o reparar a mano un sistema sin `/etc/passwd`.

### Comprueba que lo lograste

- ¿Por qué la copia `/tmp/passwd.antes` tampoco existe tras restaurar? Respuesta: se creó después del snapshot, así que vivía en el delta que la restauración descartó.
- ¿Desde el host puedes hacer ping a la víctima? Respuesta: no; la red "internal" solo une VMs entre sí, ni siquiera el host está en ella.
- ¿Por qué el hueco del ping es más largo que el `real` de `time`? Respuesta: `time` mide hasta que VirtualBox arranca la VM; el ping además espera a que la red de la VM responda.
- Si el disco base `victima.vdi` se borrara, ¿te salvaría el snapshot `limpio`? Respuesta: no; el snapshot depende del disco base. Por eso un snapshot no es un backup.

### Limpieza

Si no vas a reutilizar el snapshot: `VBoxManage snapshot victima delete limpio`. Para quitar el DHCP de la red interna: `VBoxManage dhcpserver remove --network=labnet`.

## Ejercicio 2: Compara el kernel de un contenedor y de una VM

Nodo: [Understand Concept of Isolation](README.md#understand-concept-of-isolation), en concreto [Contenedores](README.md#contenedores) y [El aislamiento es tan fuerte como la frontera que comparten](README.md#el-aislamiento-es-tan-fuerte-como-la-frontera-que-comparten).

Objetivo: demostrar con la salida de `uname -r` que un contenedor usa el kernel del host mientras una VM arranca el suyo propio, y que el contenedor solo cambia el sistema de archivos y la vista de procesos.

Necesitas: un host Linux con Docker (Debian/Ubuntu: `sudo apt install docker.io`; Arch: `sudo pacman -S docker`; después `sudo systemctl start docker`; si tu usuario no está en el grupo `docker`, antepón `sudo` a los comandos `docker`). Una VM Linux cualquiera, por ejemplo la `victima` del ejercicio 1. Con Docker Desktop en Windows o macOS el resultado es distinto (lo explica el último punto de la comprobación). Tiempo: 15 minutos.

### Pasos

1. En el host, anota tu kernel y tu distribución.
   ```bash
   uname -r
   grep PRETTY_NAME /etc/os-release
   ```
   - `uname` → muestra información del kernel en ejecución.
   - `-r` → solo la versión (release) del kernel.
   - `grep PRETTY_NAME` → imprime las líneas que contienen el texto `PRETTY_NAME`.
   - `/etc/os-release` → archivo donde cada distribución describe su nombre y versión; `PRETTY_NAME` es el nombre legible.

   Ejemplo en un Arch: `7.2.7-arch1-1` y `PRETTY_NAME="Arch Linux"`.
2. Ejecuta lo mismo dentro de un contenedor Alpine.
   ```bash
   docker run --rm alpine uname -r
   docker run --rm alpine grep PRETTY_NAME /etc/os-release
   ```
   - `docker run` → crea un contenedor a partir de una imagen y ejecuta un comando dentro.
   - `--rm` → borra el contenedor automáticamente cuando termina.
   - `alpine` → imagen a usar (Alpine Linux, muy pequeña); si no la tienes, Docker la descarga.
   - `uname -r` y `grep PRETTY_NAME /etc/os-release` → el comando que se ejecuta dentro del contenedor (ver paso 1).

   La primera línea es idéntica a la del host (`7.2.7-arch1-1`); la segunda dice `PRETTY_NAME="Alpine Linux v3.xx"`. Los archivos son de Alpine, el kernel es el tuyo.
3. Entra en una shell del contenedor y mira la vista recortada de procesos y de módulos.
   ```bash
   docker run --rm -it alpine sh
   ```
   - `docker run --rm alpine` → (ver paso 2).
   - `-i` → mantiene abierta la entrada estándar para que puedas escribir.
   - `-t` → asigna una pseudo-terminal, para que la shell se comporte como una terminal normal (`-it` junta las dos).
   - `sh` → comando que se ejecuta: la shell de Alpine.

   Dentro:
   ```bash
   ps
   ls /lib/modules
   cat /proc/version
   exit
   ```
   - `ps` → lista los procesos; sin opciones, en Alpine (BusyBox) muestra todos los que ve el contenedor.
   - `ls /lib/modules` → lista el directorio donde una distribución guarda los módulos de su kernel.
   - `cat /proc/version` → muestra la versión del kernel en ejecución, con el compilador y la fecha de compilación.
   - `exit` → sale de la shell; como era el proceso principal, el contenedor termina y `--rm` lo borra.

   `ps` muestra `sh` como PID 1 y casi nada más (namespace de PIDs propio); `/lib/modules` no existe o está vacío porque el contenedor no tiene kernel que cargue módulos; `/proc/version` describe otra vez el kernel del host, con el compilador y la fecha con que se compiló.
4. Desde el host, comprueba que ese contenedor es un proceso normal del host. En una terminal:
   ```bash
   docker run --rm --name prueba alpine sleep 300
   ```
   - `docker run --rm alpine` → (ver paso 2).
   - `--name prueba` → da al contenedor el nombre `prueba` para poder referirte a él después.
   - `sleep 300` → comando del contenedor: esperar 300 segundos sin hacer nada.

   En otra:
   ```bash
   ps -ef | grep 'sleep 300' | grep -v grep
   docker stop prueba
   ```
   - `ps -ef` → lista todos los procesos del sistema (`-e`) en formato completo (`-f`), con usuario, PID, PID del padre y línea de comandos.
   - `grep 'sleep 300'` → se queda con las líneas que contienen `sleep 300`.
   - `grep -v grep` → `-v` invierte la búsqueda: quita las líneas que contienen `grep`, para no ver el propio `grep` en el resultado.
   - `docker stop prueba` → detiene el contenedor llamado `prueba` (le manda SIGTERM y, si no termina en 10 segundos, SIGKILL).

   El `sleep 300` aparece en la lista de procesos del host con un PID normal (por ejemplo 48213), aunque dentro del contenedor sería un PID bajo.
5. Ahora en la VM. Inicia sesión y ejecuta:
   ```bash
   uname -r
   grep PRETTY_NAME /etc/os-release
   ```
   - `uname -r` y `grep PRETTY_NAME /etc/os-release` → (ver paso 1).

   Ejemplo en Debian 12: `6.1.0-26-amd64`. Es distinto del kernel del host: la VM arrancó su propio kernel sobre hardware virtual.
6. En el host, confirma que ese kernel de la VM no aparece por ningún lado como proceso: la VM entera es un único proceso del hypervisor.
   ```bash
   ps -ef | grep -i -E 'VBoxHeadless|VirtualBoxVM|qemu' | grep -v grep
   ```
   - `ps -ef` y `grep -v grep` → (ver paso 4).
   - `grep -i` → ignora mayúsculas y minúsculas.
   - `-E` → usa expresiones regulares extendidas, en las que `|` significa "o".
   - `'VBoxHeadless|VirtualBoxVM|qemu'` → nombres de los procesos que ejecutan una VM en VirtualBox (sin ventana o con ella) y en KVM/QEMU.

   Verás un proceso por VM encendida; los procesos de dentro de la VM no se ven desde el host.
7. Escribe en tus notas una tabla en texto plano con tres columnas (host, contenedor, VM) y dos filas (`uname -r`, `PRETTY_NAME`), y una frase que explique la diferencia.

### Resultado esperado

Tres pares de valores: el host y el contenedor comparten `uname -r` y difieren en `PRETTY_NAME`; la VM difiere en ambos. La explicación: el contenedor es un proceso del host aislado con namespaces y cgroups, sin kernel propio, mientras que la VM ejecuta su propio kernel y la frontera es el hypervisor.

### Comprueba que lo lograste

- ¿Qué significa que `uname -r` coincida entre host y contenedor? Respuesta: que el contenedor usa el kernel del host; un fallo de ese kernel afecta al host y a todos los contenedores.
- ¿Puedes ejecutar en un host Linux un contenedor que necesite un kernel Windows? Respuesta: no; el contenedor no trae kernel, usa el que hay.
- ¿Por qué un malware se analiza en una VM y no en un contenedor? Respuesta: en la VM la frontera es el hypervisor, mucho más pequeña que las más de 300 llamadas al sistema del kernel compartido.
- Con Docker Desktop en Windows o macOS, ¿qué kernel muestra `uname -r` en el contenedor? Respuesta: el de la pequeña VM Linux que Docker Desktop arranca por debajo (por ejemplo `...-linuxkit` o `...-microsoft-standard-WSL2`), no el del Windows o macOS; los contenedores Linux siempre necesitan un kernel Linux.

### Limpieza

Borra la imagen descargada si no la vas a usar: `docker rmi alpine`.
