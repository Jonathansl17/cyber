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
   La primera lista debe incluir `"atacante"` y `"victima"`; la segunda debe salir vacía (si no, apágalas desde dentro con `sudo poweroff`).
2. Conecta la primera tarjeta de red de ambas a una red interna llamada `labnet`.
   ```bash
   VBoxManage modifyvm atacante --nic1=intnet --intnet1=labnet
   VBoxManage modifyvm victima --nic1=intnet --intnet1=labnet
   ```
3. Una red interna no tiene DHCP. Crea un servidor DHCP de VirtualBox para `labnet`, así las VMs reciben IP sin configurar nada dentro.
   ```bash
   VBoxManage dhcpserver add --network=labnet --server-ip=10.10.10.1 --netmask=255.255.255.0 \
     --lower-ip=10.10.10.100 --upper-ip=10.10.10.200 --enable
   ```
   Si responde que ya existe, ya estaba creado de otra práctica; sigue.
4. Arranca las dos VMs (con ventana, para poder iniciar sesión en ellas).
   ```bash
   VBoxManage startvm atacante
   VBoxManage startvm victima
   ```
5. Dentro de la víctima, inicia sesión y anota su IP.
   ```bash
   ip -4 -br addr
   ```
   Verás algo como `enp0s3  UP  10.10.10.101/24`. A partir de aquí la llamamos `IP_VICTIMA`.
6. Dentro del atacante, comprueba que la red interna funciona y que está aislada.
   ```bash
   ping -c 3 10.10.10.101
   ping -c 3 -W 2 1.1.1.1
   ```
   El primero debe responder; el segundo debe fallar (`100% packet loss` o `Network is unreachable`): no hay salida a internet.
7. Con la víctima encendida y sana, toma el snapshot desde el host. Sin `--live`, VirtualBox pausa la VM un momento y guarda también la RAM, así que al restaurar volverá encendida y con la sesión abierta.
   ```bash
   VBoxManage snapshot victima take limpio --description="sana, antes de romper /etc/passwd"
   VBoxManage snapshot victima list
   ```
   La lista debe mostrar `Name: limpio (UUID: ...) *`; el asterisco marca el snapshot actual.
8. En el atacante, deja un ping continuo hacia la víctima para medir luego el corte. Añade la hora a cada línea:
   ```bash
   ping -D 10.10.10.101
   ```
   `-D` antepone la marca de tiempo Unix `[1759400000.123456]` a cada respuesta. No lo detengas.
9. Rompe la víctima. Dentro de ella:
   ```bash
   sudo cp /etc/passwd /tmp/passwd.antes
   sudo rm /etc/passwd
   whoami
   ls -l /home
   sudo -v
   ```
   `whoami` falla con `cannot find name for user ID 1000`, `ls -l` muestra números en lugar de nombres de usuario y `sudo` responde `you do not exist in the passwd database`. Ya no puedes administrar la máquina ni iniciar sesión de nuevo: está rota. Abre otra consola de texto de la VM (tecla Host + F3; la tecla Host es por defecto el Ctrl derecho) e intenta entrar con tu usuario para confirmarlo.
10. Desde el host, apaga la VM, restaura el snapshot y vuelve a arrancarla, midiendo el tiempo total. VirtualBox exige que la VM esté apagada para restaurar.
    ```bash
    time ( VBoxManage controlvm victima poweroff && \
           VBoxManage snapshot victima restore limpio && \
           VBoxManage startvm victima )
    ```
    Al final verás `real 0m4,812s` o similar: ese es el tiempo de la operación de restauración.
11. Mira el ping del atacante. Habrá un hueco en la secuencia (`icmp_seq` salta) entre la última respuesta antes del apagado y la primera después de restaurar. Resta las dos marcas de tiempo `-D`: es el tiempo real sin servicio. Detén el ping con `Ctrl+C`; el resumen dirá cuántos paquetes se perdieron (cada paquete perdido es un segundo).
12. En la ventana de la víctima, que vuelve con la sesión que tenías al tomar el snapshot, comprueba que está sana.
    ```bash
    whoami
    ls -l /etc/passwd /tmp/passwd.antes
    ```
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
   Ejemplo en un Arch: `7.2.7-arch1-1` y `PRETTY_NAME="Arch Linux"`.
2. Ejecuta lo mismo dentro de un contenedor Alpine, que se borra al salir (`--rm`).
   ```bash
   docker run --rm alpine uname -r
   docker run --rm alpine grep PRETTY_NAME /etc/os-release
   ```
   La primera línea es idéntica a la del host (`7.2.7-arch1-1`); la segunda dice `PRETTY_NAME="Alpine Linux v3.xx"`. Los archivos son de Alpine, el kernel es el tuyo.
3. Entra en una shell del contenedor y mira la vista recortada de procesos y de módulos.
   ```bash
   docker run --rm -it alpine sh
   ```
   Dentro:
   ```bash
   ps
   ls /lib/modules
   cat /proc/version
   exit
   ```
   `ps` muestra `sh` como PID 1 y casi nada más (namespace de PIDs propio); `/lib/modules` no existe o está vacío porque el contenedor no tiene kernel que cargue módulos; `/proc/version` describe otra vez el kernel del host, con el compilador y la fecha con que se compiló.
4. Desde el host, comprueba que ese contenedor es un proceso normal del host. En una terminal:
   ```bash
   docker run --rm --name prueba alpine sleep 300
   ```
   En otra:
   ```bash
   ps -ef | grep 'sleep 300' | grep -v grep
   docker stop prueba
   ```
   El `sleep 300` aparece en la lista de procesos del host con un PID normal (por ejemplo 48213), aunque dentro del contenedor sería un PID bajo.
5. Ahora en la VM. Inicia sesión y ejecuta:
   ```bash
   uname -r
   grep PRETTY_NAME /etc/os-release
   ```
   Ejemplo en Debian 12: `6.1.0-26-amd64`. Es distinto del kernel del host: la VM arrancó su propio kernel sobre hardware virtual.
6. En el host, confirma que ese kernel de la VM no aparece por ningún lado como proceso: la VM entera es un único proceso del hypervisor.
   ```bash
   ps -ef | grep -i -E 'VBoxHeadless|VirtualBoxVM|qemu' | grep -v grep
   ```
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
