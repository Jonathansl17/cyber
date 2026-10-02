# Ejercicios: Respuesta a incidentes y forense

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se hace sobre discos, VMs y cuentas tuyas, en una red de laboratorio aislada
(host-only o interna de VirtualBox) cuando hay tráfico entre máquinas. El objetivo es
preservar, analizar y detectar: nada se hace contra equipos de terceros.

## Ejercicio 1: Imagen forense de un disco y recuperación de un archivo borrado

Nodo: [dd](README.md#dd), [Evidence hashing](README.md#evidence-hashing),
[Forensic imaging](README.md#forensic-imaging), [autopsy](README.md#autopsy).

Objetivo: crear una imagen raw bit a bit de un medio propio, demostrar con SHA-256 que
la copia es idéntica al original, y recuperar desde esa imagen un archivo que borraste
antes, sin tocar el medio original.

Necesitas: Linux (cualquier distribución). Lo ideal es una USB vieja que puedas dedicar
al ejercicio; si no tienes ninguna, el ejercicio incluye una variante con un archivo-disco
creado a mano, que no requiere hardware. Paquetes: `dd`, `sha256sum` y `blockdev` vienen
en coreutils y util-linux (ya instalados). Para formatear FAT: `dosfstools`. Para abrir la
imagen: Autopsy (GUI) o The Sleuth Kit (CLI: `mmls`, `fls`, `icat`).

```
# Debian/Ubuntu
sudo apt install dosfstools sleuthkit autopsy
# Arch
sudo pacman -S dosfstools sleuthkit
#   (Autopsy está en AUR: yay -S autopsy)
```

Tiempo estimado: 45-60 minutos.

### Pasos

1. Elige el medio. Con una USB física, conéctala y averigua su nombre de dispositivo;
   anótalo con cuidado, porque equivocarse de disco en los pasos siguientes destruye datos.
   ```bash
   lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
   ```
   La USB aparece como `sdb`, `sdc`, etc. (nunca `sda`, que suele ser tu disco de sistema).
   En este ejercicio la llamamos `/dev/sdX`: sustituye `X` por la letra real en todos los
   comandos.

2. Variante sin hardware (sáltala si usas una USB real). Crea un archivo de 64 MB, dale
   formato FAT y úsalo como si fuera el disco de evidencia. Trabajas sobre un archivo normal
   en tu `$HOME`, sin riesgo para ningún disco.
   ```bash
   cd ~/lab-forense
   dd if=/dev/zero of=evidencia-origen.img bs=1M count=64
   mkfs.vfat evidencia-origen.img
   ```
   La salida de `mkfs.vfat` termina con el nombre del volumen y el tamaño del FAT; significa
   que el archivo ya contiene un sistema de archivos FAT vacío.

3. Pon algo que borrar y luego bórralo, para tener qué recuperar. Monta el medio, copia un
   archivo con contenido reconocible, desmóntalo, vuélvelo a montar, bórralo y desmonta.
   Con la USB real usa `/dev/sdX1` (la partición) en vez del `.img`.
   ```bash
   mkdir -p /tmp/mnt
   sudo mount -o loop evidencia-origen.img /tmp/mnt      # USB real: sudo mount /dev/sdX1 /tmp/mnt
   echo "secreto-de-prueba-$(date +%s)" | sudo tee /tmp/mnt/pista.txt
   sync
   sudo umount /tmp/mnt
   sudo mount -o loop evidencia-origen.img /tmp/mnt
   sudo rm /tmp/mnt/pista.txt
   sync
   sudo umount /tmp/mnt
   ```
   Borrar con `rm` solo quita la entrada del directorio FAT; los bytes de `pista.txt` siguen
   en el área de datos hasta que algo los sobrescriba. Eso es lo que recuperarás.

4. Bloquea la escritura sobre el medio antes de copiarlo. Con una USB real, el bloqueo por
   software marca el dispositivo como solo lectura; lo ideal en un caso real es un write
   blocker de hardware, pero esto evita escrituras accidentales durante el ejercicio.
   ```bash
   sudo blockdev --setro /dev/sdX
   sudo blockdev --getro /dev/sdX       # debe imprimir: 1
   ```
   `1` confirma que el kernel rechazará cualquier escritura a `/dev/sdX`. (Con la variante de
   archivo no hay dispositivo de bloques, así que este paso se omite; el write blocker es
   precisamente el control que la variante no puede practicar.)

5. Calcula el hash del original antes de copiar nada. Es el valor que irá a la cadena de
   custodia.
   ```bash
   sudo sha256sum /dev/sdX > caso.original.sha256     # variante: sha256sum evidencia-origen.img > caso.original.sha256
   cat caso.original.sha256
   ```

6. Crea la imagen raw con `dd`. Los flags son los de la nota: bloque grande para ir rápido,
   `noerror,sync` para no detenerse ante sectores dañados y conservar los desplazamientos, y
   `status=progress` para ver el avance.
   ```bash
   sudo dd if=/dev/sdX of=caso.img bs=4M conv=noerror,sync status=progress
   #  variante: dd if=evidencia-origen.img of=caso.img bs=4M conv=noerror,sync status=progress
   ```
   Al terminar imprime cuántos bytes copió y los `records in`/`records out`. Si no hubo
   sectores ilegibles, el conteo de entrada y salida coincide.

7. Verifica que la imagen es fiel. Calcula el hash de `caso.img` y compáralo con el del
   original. Deben ser idénticos (salvo que el paso 6 reportara errores de lectura, en cuyo
   caso los ceros de relleno cambian el hash y hay que documentarlo).
   ```bash
   sha256sum caso.img
   cat caso.original.sha256
   ```

8. Recupera el archivo borrado desde la imagen, nunca desde el original. Opción CLI con The
   Sleuth Kit: localiza la partición, lista las entradas borradas y extrae la que te interesa.
   ```bash
   mmls caso.img                         # ubica la partición y su sector de inicio (offset)
   fls -o OFFSET -rd caso.img            # -d: solo borrados; apunta el número de inodo/entrada
   icat -o OFFSET caso.img NUMERO > recuperado.txt
   cat recuperado.txt
   ```
   Si el medio no tiene tabla de particiones (el caso de la variante con `mkfs.vfat` sobre el
   archivo entero), usa offset 0 y omite `mmls`: `fls -rd caso.img` e `icat caso.img NUMERO`.

9. Opción GUI con Autopsy (equivalente al paso 8, recomendada si prefieres interfaz). Abre
   Autopsy, crea un caso (nombre, número, examinador), añade una fuente de datos de tipo
   "Disk Image or VM File" apuntando a `caso.img`, deja correr los ingest modules y busca el
   archivo en la vista de "Deleted Files"; clic derecho > Extract File para recuperarlo.

### Resultado esperado

Tienes `caso.img` (imagen raw del medio), un hash SHA-256 que coincide entre original e
imagen, y `recuperado.txt` con la línea "secreto-de-prueba-..." que habías borrado en el
paso 3. El medio original nunca se escribió.

### Comprueba que lo lograste

- ¿Coincide el SHA-256 del original con el de `caso.img`? Si sí, la imagen es válida como
  copia forense. Si no, revisa si `dd` reportó errores de lectura (entonces es esperado y se
  documenta) o si montaste el medio sin bloqueo de escritura por error.
- ¿El contenido de `recuperado.txt` es exactamente el texto que escribiste antes de borrar?
- `sudo blockdev --getro /dev/sdX` seguía devolviendo `1` al final: no se escribió nada en el
  original durante todo el proceso.

### Limpieza

Quita el bloqueo de escritura de la USB para volver a usarla y borra los archivos del
ejercicio.

```bash
sudo blockdev --setro /dev/sdX --setrw /dev/sdX   # o reconecta la USB
rm -f caso.img recuperado.txt caso.original.sha256 evidencia-origen.img
rmdir /tmp/mnt 2>/dev/null
```

## Ejercicio 2: Volcado de RAM con AVML y análisis con Volatility 3

Nodo: [memdump](README.md#memdump).

Objetivo: capturar la memoria de una VM Linux propia mientras tiene una conexión de red y un
historial de bash activos, y luego, desde el volcado, encontrar ese proceso, esa conexión y
los comandos tecleados con Volatility 3.

Necesitas: dos VMs Linux en una red host-only o interna de VirtualBox (una "víctima" donde
capturas, otra que hace de extremo de la conexión). En la víctima: AVML (binario estático de
Microsoft, no instala nada) y, para el análisis, Volatility 3 (en tu equipo de análisis, no
hace falta que sea la víctima). El punto delicado de la forense de memoria en Linux es que
Volatility necesita una tabla de símbolos (ISF) que corresponda exactamente al kernel del
volcado; la generas con dwarf2json a partir de los símbolos de depuración de ese kernel.

```
# Volatility 3 en el equipo de análisis (entorno virtual de Python)
python3 -m venv ~/vol3 && ~/vol3/bin/pip install volatility3

# AVML: descarga el binario de la release oficial en la VM víctima
curl -L -o avml https://github.com/microsoft/avml/releases/download/v0.20.0/avml
chmod +x avml

# dwarf2json para crear la tabla de símbolos del kernel
curl -L -o dwarf2json https://github.com/volatilityfoundation/dwarf2json/releases/download/v0.9.0/dwarf2json-linux-amd64
chmod +x dwarf2json
```

Tiempo estimado: 60-90 minutos (la tabla de símbolos es lo que más cuesta la primera vez).

### Pasos

1. En la VM víctima, deja actividad que luego buscarás en memoria. Abre una conexión a la
   otra VM con `nc` y teclea un par de comandos reconocibles en bash. Mantén esa terminal
   abierta (la conexión debe seguir viva cuando captures).
   ```bash
   # en la segunda VM (el otro extremo), escuchando:
   nc -l -p 9000
   # en la víctima, conectando y tecleando algo memorable:
   nc 10.0.0.20 9000
   echo marcador-volatility-2026
   ```
   La idea es que en el momento del volcado existan un proceso `nc`, una conexión TCP
   establecida al puerto 9000 y una línea de historial de bash con "marcador-volatility-2026".

2. Captura la RAM con AVML a un archivo. El binario elige solo la fuente de memoria
   (`/proc/kcore`, `/dev/crash` o `/dev/mem`); necesita privilegios para leerla.
   ```bash
   sudo ./avml memoria.lime
   sha256sum memoria.lime > memoria.sha256
   ```
   El archivo sale en formato LiME y ocupa aproximadamente lo mismo que la RAM de la VM.
   Calcular el hash justo al terminar es parte del procedimiento forense.

3. Identifica el kernel exacto de la víctima. La tabla de símbolos debe coincidir con esta
   versión; apúntala.
   ```bash
   uname -r        # ej.: 6.1.0-18-amd64
   ```

4. Consigue los símbolos de depuración de ese kernel y genera la tabla ISF con dwarf2json.
   En Debian/Ubuntu los símbolos vienen en un paquete `linux-image-$(uname -r)-dbg` o en el
   repositorio ddeb de depuración; dan un `vmlinux` con información DWARF.
   ```bash
   # Debian/Ubuntu: instala los símbolos de depuración del kernel en uso
   sudo apt install linux-image-$(uname -r)-dbg
   # el vmlinux con símbolos suele quedar en /usr/lib/debug/boot/vmlinux-$(uname -r)
   ./dwarf2json linux --elf /usr/lib/debug/boot/vmlinux-$(uname -r) > linux-$(uname -r).json
   ```
   Si no hay paquete de símbolos para tu kernel, también puedes partir del `System.map`
   (`--system-map /boot/System.map-$(uname -r)`), aunque da menos información de tipos.

5. Coloca la tabla donde Volatility la encuentra. Volatility busca los símbolos de Linux en
   un subdirectorio `linux` dentro de sus carpetas de símbolos; puedes añadir la tuya con
   `-s` al invocarlo.
   ```bash
   mkdir -p ~/vol3-symbols/linux
   cp linux-$(uname -r).json ~/vol3-symbols/linux/
   ```

6. Lista los procesos del volcado y localiza `nc`. El plugin `linux.pslist` recorre la lista
   de tareas del kernel dentro del volcado.
   ```bash
   ~/vol3/bin/vol -s ~/vol3-symbols -f memoria.lime linux.pslist | grep -i nc
   ```
   Debe aparecer una fila con `nc` y su PID. Anota el PID.

7. Encuentra la conexión de red. El plugin `linux.sockstat` lista los sockets abiertos con
   proceso, direcciones y puertos.
   ```bash
   ~/vol3/bin/vol -s ~/vol3-symbols -f memoria.lime linux.sockstat | grep 9000
   ```
   Verás el socket TCP hacia el puerto 9000 asociado al proceso `nc`: es la conexión que
   dejaste viva.

8. Recupera los comandos tecleados. El plugin `linux.bash` extrae el historial de bash que
   seguía en memoria.
   ```bash
   ~/vol3/bin/vol -s ~/vol3-symbols -f memoria.lime linux.bash
   ```
   Entre las líneas debe estar `echo marcador-volatility-2026`, con la marca de tiempo de
   cuando lo escribiste.

### Resultado esperado

Tienes `memoria.lime` con su hash, una tabla de símbolos propia del kernel de la VM, y la
salida de tres plugins que, juntos, reconstruyen lo que estaba pasando: el proceso `nc`, su
conexión al puerto 9000 y el comando `echo marcador-volatility-2026` que tecleaste. Nada de
esto queda en el disco de la víctima: solo vivía en RAM.

### Comprueba que lo lograste

- ¿Aparece `nc` en `linux.pslist` con un PID, y ese mismo proceso en el socket que muestra
  `linux.sockstat`? Eso liga proceso y conexión.
- ¿`linux.bash` muestra la línea con "marcador-volatility-2026"?
- Si algún plugin falla con un error de símbolos ("unsatisfied requirement ... symbol table"),
  la tabla ISF no corresponde al kernel del volcado: revisa que la versión de `uname -r` del
  paso 3 sea la misma con la que generaste el JSON en el paso 4.

### Limpieza

```bash
rm -f memoria.lime memoria.sha256 avml dwarf2json linux-*.json
rm -rf ~/vol3-symbols
# opcional: desinstalar los símbolos de depuración y Volatility
sudo apt remove linux-image-$(uname -r)-dbg
rm -rf ~/vol3
```

## Ejercicio 3: Triaje de un log de autenticación con la tubería grep sort uniq

Nodo: [grep](README.md#grep), [tail](README.md#tail), [head](README.md#head),
[cat](README.md#cat).

Objetivo: generar en tu laboratorio una ráfaga de inicios de sesión SSH fallidos contra una
VM propia y, desde el log de autenticación, identificar en segundos qué IP origina más
intentos usando la tubería de triaje `grep | sort | uniq -c | sort -rn | head`.

Necesitas: dos VMs Linux en red host-only o interna. Una "defensora" con el servidor SSH
(`openssh-server`) y otra desde la que se originan los intentos. Ambas son tuyas y están
aisladas; aquí no se ataca a nadie, se provoca ruido de login fallido en tu propio servidor
para practicar el análisis del log. Herramientas: `ssh` (cliente), `grep`, `sort`, `uniq`,
`head` (todas de base). Tiempo estimado: 30 minutos.

```
# en la VM defensora, si no tiene servidor SSH
# Debian/Ubuntu
sudo apt install openssh-server
sudo systemctl enable --now ssh
# Arch
sudo pacman -S openssh
sudo systemctl enable --now sshd
```

### Pasos

1. En la VM defensora, averigua su IP de la red de laboratorio y confirma que el servidor
   SSH escucha.
   ```bash
   ip -4 addr show
   ss -tlnp | grep :22
   ```
   Apunta la IP (por ejemplo `10.0.0.10`). `ss` debe mostrar un proceso `sshd` en el puerto 22.

2. Desde la VM de origen, genera inicios de sesión fallidos contra un usuario que no existe.
   Un bucle que intenta conectar varias veces basta para llenar el log de entradas "Failed".
   No necesitas acertar ninguna credencial; el objetivo es producir los registros, no entrar.
   ```bash
   for i in $(seq 1 30); do
     ssh -o BatchMode=yes -o ConnectTimeout=3 -o StrictHostKeyChecking=no \
         usuario-inexistente@10.0.0.10 true 2>/dev/null
   done
   ```
   `BatchMode=yes` hace que el cliente no se quede esperando una contraseña: cada intento
   falla solo y deja su rastro en el log del servidor. Las 30 vueltas simulan la ráfaga.

3. Vuelve a la VM defensora y mira en vivo cómo entran los registros mientras el bucle corre
   (puedes lanzar el paso 2 y este a la vez en terminales distintas).
   ```bash
   sudo tail -f /var/log/auth.log         # systemd sin auth.log: sudo journalctl -u ssh -f
   ```
   Verás líneas del tipo `Failed password for invalid user usuario-inexistente from 10.0.0.20`.
   Corta con Ctrl-C cuando dejen de llegar.

4. Aplica la tubería de triaje al log completo: filtra los fallos, extrae solo la IP de
   origen, agrupa iguales, cuéntalos, ordénalos de mayor a menor y quédate con los primeros.
   ```bash
   sudo grep "Failed password" /var/log/auth.log \
     | grep -oE "from [0-9.]+" \
     | sort | uniq -c | sort -rn | head
   ```
   La salida es una tabla "conteo IP" con la IP más ruidosa arriba:
   ```
   30 from 10.0.0.20
    2 from 10.0.0.31
   ```

5. Variante para sistemas con systemd sin `auth.log` (Arch, Fedora y otros). El log de SSH
   está en el journal; `cat`/`grep` se sustituyen por `journalctl`, pero la tubería es la misma.
   ```bash
   sudo journalctl -u sshd --no-pager \
     | grep "Failed password" \
     | grep -oE "from [0-9.]+" \
     | sort | uniq -c | sort -rn | head
   ```

6. Confirma el hallazgo mirando el contexto de esa IP: cuántos fallos y si hubo algún acceso
   aceptado después (en este laboratorio no debería haberlo, porque el usuario no existe).
   ```bash
   sudo grep "10.0.0.20" /var/log/auth.log | grep -E "Failed|Accepted" | head
   ```

### Resultado esperado

En una sola línea de comandos obtienes qué IP de tu red de laboratorio concentra los inicios
de sesión fallidos y cuántos, que es exactamente el primer paso del triaje de una sospecha de
fuerza bruta. La IP de la VM de origen aparece en lo alto con el número de intentos del bucle.

### Comprueba que lo lograste

- ¿La IP de la VM de origen encabeza la lista del paso 4 con un conteo cercano a las vueltas
  del bucle (30)?
- ¿Entiendes cada eslabón de la tubería? `grep "Failed password"` deja solo los fallos,
  `grep -oE "from [0-9.]+"` extrae la IP, `sort` agrupa, `uniq -c` cuenta, `sort -rn` ordena de
  mayor a menor y `head` deja el top. Quita el último `| head` y verás el listado completo.
- ¿`grep "Accepted"` para esa IP no devuelve nada? Confirma que ningún intento tuvo éxito, que
  es lo esperable contra un usuario inexistente.

### Limpieza

Para el servidor SSH si solo lo levantaste para el ejercicio y, si quieres, limpia las
entradas del log de laboratorio.

```bash
sudo systemctl disable --now ssh      # o sshd, según la distribución
```
