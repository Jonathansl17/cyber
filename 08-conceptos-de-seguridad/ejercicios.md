# Ejercicios: Conceptos de seguridad

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se hace sobre tu propio equipo: tus cuentas, tus discos y servicios que tú mismo
levantas. Cuando un ejercicio necesita varias zonas de red o un "sitio remoto", se simulan en
el mismo equipo (interfaces dummy, loopback, `127.0.0.1`), nunca contra equipos o servicios de
terceros. Las herramientas ofensivas (nmap, etc.) se usan solo contra `127.0.0.1` y servicios
que tú mismo levantaste en este equipo.

## Ejercicio 1: Aplica la regla 3-2-1 y mide tu RPO y RTO

Nodo: Backups and Resiliency ([README](README.md#understand-backups-and-resiliency)).

Objetivo: tener los mismos datos en 3 copias, 2 soportes y 1 fuera del sitio con restic,
borrar el original, restaurarlo y obtener dos números tuyos: cuánto perderías (RPO, fijado
por la hora del último snapshot) y cuánto tardas en volver (RTO, el tiempo real de restauración).

Necesitas: Linux. Un pendrive USB. El servidor SSH local (`sshd`) escuchando en `127.0.0.1`
para simular el sitio remoto: la copia "fuera del sitio" será una segunda carpeta local a la
que restic entra por SFTP contra `127.0.0.1` (sin segundo equipo ni VM). Paquetes `restic` y
el servidor OpenSSH. Unos 30 minutos.

Instalación:
- Debian/Ubuntu: `sudo apt update && sudo apt install restic openssh-server`
- Arch: `sudo pacman -S restic openssh`

En Arch arranca el servicio con `sudo systemctl start sshd` (Debian suele arrancarlo solo al
instalar `openssh-server`). Comprueba que escucha con `ss -tlnp | grep ':22'`.

### Pasos

1. Crea una carpeta de prueba con datos reconocibles y mídela.
   ```bash
   mkdir -p ~/datos-prueba
   for i in $(seq 1 200); do head -c 1M /dev/urandom > ~/datos-prueba/archivo-$i.bin; done
   du -sh ~/datos-prueba
   ```
   - `mkdir` → crea directorios.
   - `-p` → crea también los directorios padre que falten y no da error si la carpeta ya existe.
   - `~/datos-prueba` → carpeta a crear dentro de tu directorio personal (`~` es tu `$HOME`).
   - `for i in $(seq 1 200); do ...; done` → bucle de bash que repite el cuerpo una vez por cada valor de `i`.
   - `seq 1 200` → imprime los números del 1 al 200, uno por línea; `$(...)` sustituye el comando por su salida.
   - `head` → muestra el principio de un archivo o flujo.
   - `-c 1M` → toma solo los primeros bytes indicados; `1M` es 1 MiB (1048576 bytes).
   - `/dev/urandom` → dispositivo del kernel que entrega bytes aleatorios sin fin; `head` corta en 1 MiB.
   - `> ~/datos-prueba/archivo-$i.bin` → redirige la salida a un archivo nuevo (o lo sobrescribe); `$i` pone el número del bucle en el nombre.
   - `du` → calcula el espacio en disco que ocupan archivos y carpetas.
   - `-s` → da solo el total de la carpeta, sin listar cada subcarpeta.
   - `-h` → muestra el tamaño en unidades legibles (K, M, G).

   Deberías ver unos `200M`. Son tus datos originales: la copia número 1 de las 3.

2. Copia 2, soporte 1 (disco local): crea un repositorio restic en el disco del equipo.
   restic pide una contraseña de cifrado del repositorio; apúntala, sin ella no se restaura.
   ```bash
   restic -r ~/restic-local init
   restic -r ~/restic-local backup ~/datos-prueba
   ```
   - `restic` → programa de copias de seguridad cifradas y deduplicadas.
   - `-r ~/restic-local` → ruta del repositorio sobre el que actúa el comando (equivale a `--repo`; si no se pone, restic usa la variable `RESTIC_REPOSITORY`).
   - `init` → crea un repositorio vacío en esa ruta y pide la contraseña con la que se cifra.
   - `backup` → guarda un snapshot nuevo con el contenido de las rutas que le pasas.
   - `~/datos-prueba` → carpeta que se copia en el snapshot.

   La última línea imprime `snapshot XXXXXXXX saved`. Apunta la hora: ese instante es tu
   punto de recuperación.

3. Copia 3, soporte 2 (USB): monta el pendrive y repite el backup en él. Ajusta la ruta de
   montaje a la de tu sistema (mira `lsblk` para ver el dispositivo).
   ```bash
   lsblk
   sudo mkdir -p /mnt/usb                                             # crea el punto de montaje
   sudo mount -o uid=$(id -u),gid=$(id -g) /dev/sdX1 /mnt/usb         # sustituye sdX1 por tu partición
   restic -r /mnt/usb/restic-usb init
   restic -r /mnt/usb/restic-usb backup ~/datos-prueba
   ```
   - `lsblk` → lista los discos y particiones (dispositivos de bloque) con su tamaño y punto de montaje; sin opciones muestra el árbol completo.
   - `sudo` → ejecuta el comando siguiente como root; crear la carpeta bajo `/mnt` y montar un sistema de archivos lo exigen.
   - `mkdir -p /mnt/usb` → crea el punto de montaje antes de montar; sin él `mount` falla con "mount point does not exist".
   - `mount` → engancha un sistema de archivos en una carpeta del árbol.
   - `-o uid=$(id -u),gid=$(id -g)` → opciones de montaje. En un pendrive FAT/exFAT (sin permisos Unix) todo lo montado con `sudo` queda de root, y entonces `restic init` como tu usuario falla con "permission denied"; con `uid`/`gid` los archivos aparecen como tuyos y restic puede escribir. En un pendrive ext4 esta opción se ignora: ahí monta con `sudo` y crea el repo con `sudo restic ...`, de forma consistente.
   - `/dev/sdX1` → la partición del pendrive (primer argumento: qué montar).
   - `/mnt/usb` → carpeta donde aparecerá su contenido (segundo argumento: dónde montarlo).
   - `# sustituye ...` → comentario de bash, no se ejecuta.
   - `restic -r /mnt/usb/restic-usb` → mismo `-r` del paso 2, ahora con el repositorio dentro del USB.
   - `init` → (ver paso 2).
   - `backup ~/datos-prueba` → (ver paso 2).

   Dos soportes distintos: si muere el disco del equipo, el USB sobrevive, y al revés.

4. La copia "1 fuera del sitio": manda un backup por SFTP a una segunda carpeta local, pero
   entrando por SSH contra `127.0.0.1` (el mismo equipo hace de "sitio remoto"). restic usa su
   backend SFTP igual que contra una máquina real; lo único que cambia es que el host es tu
   propio `sshd` en loopback. Sustituye `TU_USUARIO` por tu usuario (lo ves con `whoami`).
   ```bash
   mkdir -p ~/restic-offsite
   restic -r sftp:TU_USUARIO@127.0.0.1:/home/TU_USUARIO/restic-offsite init
   restic -r sftp:TU_USUARIO@127.0.0.1:/home/TU_USUARIO/restic-offsite backup ~/datos-prueba
   ```
   - `mkdir -p ~/restic-offsite` → crea de antemano la carpeta que hará de repositorio "remoto"; el backend SFTP de restic no crea la ruta base si no existe.
   - `-r sftp:TU_USUARIO@127.0.0.1:/home/TU_USUARIO/restic-offsite` → repositorio por SFTP: `sftp:` es el tipo de backend, `TU_USUARIO@127.0.0.1` la cuenta y el host al que restic entra por SSH (aquí tu propio equipo por loopback), y lo que va tras los dos puntos es la ruta del repositorio en ese host. En un despliegue real `127.0.0.1` sería la dirección de otra máquina en otro sitio; el backend SFTP y el flujo de datos que ejercitas son idénticos.
   - `init` → (ver paso 2).
   - `backup ~/datos-prueba` → (ver paso 2).

   La primera conexión SSH a `127.0.0.1` te pedirá aceptar la huella del host (escribe `yes`) y
   la contraseña de tu usuario, salvo que ya tengas una clave SSH cargada.

   Con esto recorres el esquema 3-2-1 de punta a punta. Cuenta las copias de los datos: el
   original y el repo local están los dos en el disco del equipo, así que ese disco aporta una
   sola copia útil frente a un fallo (si el disco muere, caen las dos); el USB aporta otra sobre
   un medio distinto. El repositorio por SFTP hace de tercera copia, la "fuera del sitio". Ojo:
   en este laboratorio esa copia vive en el mismo disco (la alcanzas por `127.0.0.1`), así que
   aquí no te da resiliencia geográfica de verdad; su papel es ejercitar el backend SFTP y la
   restauración remota. En producción ese `127.0.0.1` sería otra máquina en otra ubicación, y
   entonces sí sería la copia que sobrevive a que se destruya tu sitio entero. El repo local, por
   estar en el mismo disco que el original, tampoco cuenta como copia extra para el 3-2-1: su
   valor es la restauración rápida (RTO bajo), no la resiliencia.

5. Simula el desastre: borra el original.
   ```bash
   rm -rf ~/datos-prueba
   ls ~/datos-prueba        # debe dar "No such file or directory"
   ```
   - `rm` → borra archivos.
   - `-r` → borra de forma recursiva, entrando en la carpeta y en todo lo que contiene.
   - `-f` → fuerza el borrado sin preguntar y sin error si algo no existe.
   - `ls ~/datos-prueba` → lista el contenido de la carpeta; aquí sirve para confirmar que ya no existe.

6. Mide el RTO: restaura desde el repositorio local y cronométralo.
   ```bash
   time restic -r ~/restic-local restore latest --target ~/datos-restaurados
   ```
   - `time` → palabra clave de bash que ejecuta el comando y al terminar imprime cuánto tardó: `real` (tiempo de reloj), `user` y `sys` (CPU).
   - `-r ~/restic-local` → (ver paso 2).
   - `restore` → extrae los archivos de un snapshot.
   - `latest` → identificador especial que elige el snapshot más reciente; en su lugar puede ir el ID de uno concreto.
   - `--target ~/datos-restaurados` → carpeta donde se escriben los archivos restaurados (forma corta `-t`); restic recrea dentro la ruta original completa.
   Anota el valor de `real` (por ejemplo `real 0m8.4s`): ese es tu RTO medido para este volumen.

7. Comprueba que la restauración es íntegra comparando con lo que esperabas y verificando el repositorio.
   ```bash
   du -sh ~/datos-restaurados/home/*/datos-prueba
   restic -r ~/restic-local check
   ```
   - `du -sh` → (ver paso 1).
   - `~/datos-restaurados/home/*/datos-prueba` → ruta restaurada; `*` es un comodín de bash que encaja con tu nombre de usuario, porque restic recreó la ruta original `/home/<usuario>/datos-prueba` dentro del destino.
   - `-r ~/restic-local` → (ver paso 2).
   - `check` → comprueba la estructura y coherencia del repositorio; debe terminar en `no errors were found`.

### Resultado esperado

Tres repositorios restic con el mismo snapshot, la carpeta restaurada con sus 200 MB
intactos, y dos números escritos: RPO = tiempo entre el último snapshot y el momento del
desastre (si solo haces backup una vez al día, tu RPO es de hasta 24 horas); RTO = el valor
`real` del paso 6.

### Comprueba que lo lograste

- ¿Cuál es tu RPO con este esquema? Respuesta esperada: el intervalo entre backups; con un
  solo backup diario, hasta 24 horas de datos en riesgo.
- ¿Tu RTO medido baja del objetivo que te pondrías (por ejemplo 1 hora)? Para 200 MB debería
  ser segundos; con 500 GB sería mucho mayor, y ahí se ve por qué el RTO depende del volumen.
- ¿Sobrevive el esquema a que se moje el equipo entero? El USB sobrevive si no estaba
  conectado. La copia por SFTP solo salvaría tus datos si estuviera en otra máquina u otra
  ubicación; en este laboratorio está en el mismo disco (la alcanzas por `127.0.0.1`), así que
  aquí se ve justo por qué la copia "fuera del sitio" tiene que estar físicamente lejos para
  contar como tal.

### Limpieza

```bash
rm -rf ~/restic-local ~/datos-restaurados ~/restic-offsite
restic -r /mnt/usb/restic-usb forget --prune latest 2>/dev/null; sudo umount /mnt/usb
```
- `rm -rf` → (ver paso 5): borra el repositorio local, la carpeta restaurada y el repositorio "fuera del sitio" (es una carpeta local, basta con borrarla).
- `restic -r /mnt/usb/restic-usb` → (ver paso 3): elige el repositorio del USB.
- `forget` → elimina snapshots de la lista del repositorio.
- `latest` → el snapshot que se olvida: el más reciente (ver paso 6).
- `--prune` → tras olvidar, ejecuta `prune`, que borra los datos que ya no usa ningún snapshot y libera espacio.
- `2>/dev/null` → manda los mensajes de error (descriptor 2) a `/dev/null`, donde se descartan.
- `;` → separa dos comandos: el segundo se ejecuta aunque el primero falle.
- `sudo umount /mnt/usb` → `sudo` lo ejecuta como root y `umount` desmonta el sistema de archivos montado en `/mnt/usb`, para poder retirar el pendrive sin perder datos.

## Ejercicio 2: Calcula SLE ALE y el valor de un control

Nodo: Definition of Risk ([README](README.md#understand-the-definition-of-risk)).

Objetivo: poner en dinero el riesgo de tres activos tuyos con las fórmulas de la nota
(SLE = AV × EF, ALE = SLE × ARO, Valor del control = ALE antes − ALE después − coste del
control) y decidir con el número qué respuesta al riesgo aplicas a cada uno: mitigar,
transferir, evitar o aceptar.

Necesitas: Python 3 (ya viene en Linux). 20 minutos. No hace falta red ni permisos especiales.

### Pasos

1. Elige tres activos tuyos y ponles un valor (AV) honesto: por ejemplo portátil 1200,
   móvil 800, disco de fotos 300 (en la moneda que uses). Anótalos.

2. Para cada activo inventa un escenario de pérdida con su EF (porcentaje que se pierde en
   un incidente) y su ARO (veces esperadas al año; 0,5 = una vez cada dos años). Guarda esto
   en un script para no equivocarte con la aritmética.
   ```bash
   cat > ~/riesgo.py <<'EOF'
   # activo: (AV, EF, ARO, ALE_despues_con_control, coste_control_anual)
   activos = {
       "portatil":    (1200, 0.80, 0.5,  60,  40),   # robo o rotura; control: seguro + backup
       "movil":       ( 800, 1.00, 0.3,  24,  15),   # perdida total; control: localizar y borrar
       "disco fotos": ( 300, 1.00, 0.2,   0,  20),   # fallo de disco; control: backup 3-2-1
   }
   for nombre, (av, ef, aro, ale_despues, coste) in activos.items():
       sle = av * ef
       ale = sle * aro
       valor_control = ale - ale_despues - coste
       decision = "comprar control (mitigar)" if valor_control > 0 else "otro control o aceptar"
       print(f"{nombre:12} SLE={sle:8.0f}  ALE={ale:8.0f}  "
             f"valor_control={valor_control:8.0f}  -> {decision}")
   EOF
   python3 ~/riesgo.py
   ```
   - `cat > ~/riesgo.py` → `cat` copia su entrada a la salida, y `>` guarda esa salida en el archivo `~/riesgo.py` (lo crea o lo sobrescribe).
   - `<<'EOF' ... EOF` → here-document: las líneas hasta `EOF` son la entrada de `cat`; las comillas en `'EOF'` impiden que bash expanda `$` o comillas dentro del texto.
   - `activos = {...}` → diccionario de Python: clave el nombre del activo, valor una tupla `(AV, EF, ARO, ALE_despues, coste)`; lo que va tras `#` son comentarios.
   - `for nombre, (av, ef, aro, ale_despues, coste) in activos.items():` → recorre cada activo y desempaqueta sus cinco valores.
   - `sle = av * ef` → SLE = AV × EF.
   - `ale = sle * aro` → ALE = SLE × ARO.
   - `valor_control = ale - ale_despues - coste` → valor del control = ALE antes − ALE después − coste anual.
   - `decision = ... if valor_control > 0 else ...` → expresión condicional: mitigar si el valor es positivo, si no otro control o aceptar.
   - `print(f"...")` → imprime una fila con f-strings; `{nombre:12}` reserva 12 caracteres y `{sle:8.0f}` muestra el número en 8 caracteres sin decimales.
   - `python3 ~/riesgo.py` → ejecuta el script con el intérprete de Python 3.

   Cada fila te da SLE, ALE y si el control se paga solo.

3. Para cada activo, elige la respuesta al riesgo de la nota y justifícala con el número:
   mitigar si el valor del control es positivo; aceptar si el ALE ya es pequeño y ningún
   control barato lo mejora; transferir (un seguro) si el impacto es alto y raro; evitar si
   puedes dejar de exponer el activo.

### Resultado esperado

Una tabla con SLE, ALE y valor del control de tus tres activos, y una decisión por activo
escrita con su razón numérica. Por ejemplo: el disco de fotos tiene ALE 60 y un control de
backup que lo lleva a 0 por 20 al año, valor del control +40, así que se mitiga.

### Comprueba que lo lograste

- ¿Qué pasa con el valor del control si subes el coste del control por encima del ALE antes?
  Respuesta: se vuelve negativo; proteger costaría más que el daño que evita y la decisión
  racional es otro control o aceptar.
- ¿Un activo con EF alto pero ARO muy bajo da un ALE alto o bajo? Bajo, porque ALE multiplica
  por la frecuencia anual: un desastre carísimo pero rarísimo puede salir barato por año.

## Ejercicio 3: Segmenta tres zonas en un host con interfaces dummy y nftables

Nodo: Perimiter vs DMZ vs Segmentation ([README](README.md#perimiter-vs-dmz-vs-segmentation)).

Objetivo: montar tres zonas (DMZ, usuarios, servidores) en tu propio equipo, sin ninguna VM,
usando tres interfaces de red *dummy*; cargar la política entre zonas con nftables y demostrar
con `nc` y `ping` desde cada zona que solo alcanzas los puertos que la política permite y que
todo lo demás sale bloqueado.

La idea es "varias zonas, un solo equipo": cada zona es una interfaz dummy con su propia IP, y
el mismo kernel enruta y filtra el tráfico entre ellas. Donde antes había un firewall pfSense y
tres máquinas, ahora hay tres IP locales y una tabla de nftables. En una red real cada zona
estaría en una máquina distinta detrás de un firewall; el tráfico que observas y la política que
aplicas son los mismos.

Necesitas: Linux con `nftables`. Las interfaces dummy vienen de serie en el kernel (el módulo
`dummy` se carga solo al crear la interfaz), así que no se instala nada para ellas. `nc`
(netcat) y `nmap` para probar. Permisos de root (sudo). Unos 40 minutos.

Las tres zonas y sus IP:
- DMZ → `10.0.1.1/24` (interfaz `zonadmz`)
- usuarios → `10.0.10.1/24` (interfaz `zonausuarios`)
- servidores → `10.0.20.1/24` (interfaz `zonaservidores`)

Instalación:
- Debian/Ubuntu: `sudo apt install nftables nmap netcat-openbsd`
- Arch: `sudo pacman -S nftables nmap openbsd-netcat`

### Pasos

1. Crea las tres interfaces dummy, una por zona, y asígnales su IP. Hazlo en una terminal; el
   resto de pasos usará otras.
   ```bash
   for z in dmz:10.0.1.1 usuarios:10.0.10.1 servidores:10.0.20.1; do
     nombre="zona${z%%:*}"; ip="${z##*:}"
     sudo ip link add "$nombre" type dummy
     sudo ip addr add "$ip/24" dev "$nombre"
     sudo ip link set "$nombre" up
   done
   ip -brief addr show | grep '^zona'
   ```
   - `for z in dmz:10.0.1.1 ...; do ...; done` → bucle de bash que recorre tres pares `zona:ip`.
   - `nombre="zona${z%%:*}"` → expansión de bash: `${z%%:*}` quita desde el primer `:` hasta el final y deja el nombre de la zona; así `dmz` se vuelve `zonadmz`.
   - `ip="${z##*:}"` → `${z##*:}` quita todo hasta el último `:` y deja la IP.
   - `ip link add "$nombre" type dummy` → crea una interfaz virtual de tipo *dummy*: una tarjeta de red falsa que acepta una IP y existe solo en este kernel; el módulo `dummy` se carga solo al crearla, no hay que instalar nada.
   - `ip addr add "$ip/24" dev "$nombre"` → asigna la IP y la máscara `/24` a esa interfaz.
   - `ip link set "$nombre" up` → levanta la interfaz (sin esto queda `DOWN` y no enruta).
   - `ip -brief addr show` → lista las interfaces en formato compacto; `-brief` da una línea por interfaz con estado e IP.
   - `grep '^zona'` → filtra solo las líneas que empiezan por `zona`, es decir, las tres recién creadas.

   Cada IP es una zona. Como las tres son locales a este equipo, el tráfico entre ellas lo
   entrega el propio kernel (pasa por `lo`), y ahí es donde nftables lo va a filtrar.

2. Escribe la política entre zonas en un archivo de nftables y cárgala. Por defecto se bloquea
   el tráfico entre zonas; solo se permite lo que la nota autoriza.
   ```bash
   cat > ~/zonas.nft <<'EOF'
   table inet zonas {
       chain interzona {
           type filter hook input priority 0; policy accept;

           # Respuestas de conexiones ya permitidas: dejarlas pasar.
           ct state established,related accept

           # Tráfico que NO es entre las subredes del laboratorio: no lo tocamos.
           ip saddr != { 10.0.1.0/24, 10.0.10.0/24, 10.0.20.0/24 } accept
           ip daddr != { 10.0.1.0/24, 10.0.10.0/24, 10.0.20.0/24 } accept

           # Política permitida (solo llegan aquí los paquetes nuevos entre zonas):
           ip saddr 10.0.10.0/24 ip daddr 10.0.1.0/24  tcp dport 443  accept   # usuarios -> DMZ (web)
           ip saddr 10.0.1.0/24  ip daddr 10.0.20.0/24 tcp dport 5432 accept   # DMZ -> servidores (BD)

           # Cualquier otro tráfico entre las tres zonas: bloquear.
           ip saddr { 10.0.1.0/24, 10.0.10.0/24, 10.0.20.0/24 } ip daddr { 10.0.1.0/24, 10.0.10.0/24, 10.0.20.0/24 } drop
       }
   }
   EOF
   sudo nft -f ~/zonas.nft
   sudo nft list table inet zonas
   ```
   - `cat > ~/zonas.nft <<'EOF' ... EOF` → here-document: guarda en `~/zonas.nft` todo lo que va hasta `EOF`; las comillas en `'EOF'` impiden que bash toque los `$` del texto.
   - `table inet zonas` → una tabla de nftables de familia `inet` (vale para IPv4 e IPv6) llamada `zonas`.
   - `chain interzona { type filter hook input priority 0; policy accept; }` → una cadena enganchada al *hook* `input`, por donde pasa todo paquete que el kernel entrega a una IP local (y el tráfico entre dos IP locales se entrega localmente, así que pasa por aquí). `policy accept` deja pasar lo que ninguna regla bloquee; la segmentación la hace la regla `drop` del final, no la política de la cadena, para no cortar el tráfico normal del equipo.
   - `ct state established,related accept` → acepta los paquetes de vuelta de una conexión ya permitida (sin esto, la respuesta en un puerto efímero caería en el `drop` del final y cortaría la conexión).
   - `ip saddr != { ... } accept` / `ip daddr != { ... } accept` → si el origen o el destino no es una de las tres subredes del laboratorio, se acepta y no se mira más; así solo filtramos el tráfico entre zonas y no el normal del equipo (internet, loopback, SSH...).
   - `ip saddr 10.0.10.0/24 ip daddr 10.0.1.0/24 tcp dport 443 accept` → la única regla de permitir para usuarios: TCP 443 hacia la DMZ (el web).
   - `ip saddr 10.0.1.0/24 ip daddr 10.0.20.0/24 tcp dport 5432 accept` → la única regla de permitir para la DMZ: TCP 5432 hacia servidores (la base de datos).
   - `ip saddr { ... } ip daddr { ... } drop` → todo lo demás entre las tres zonas se descarta; equivale al "default deny" del firewall. Descartar (no rechazar) es lo que hará que un escaneo vea `filtered`, no `closed`.
   - `sudo nft -f ~/zonas.nft` → carga el archivo de reglas; queda activo al instante, sin reiniciar nada.
   - `sudo nft list table inet zonas` → muestra la tabla cargada para confirmar que está.

   Fíjate en la lógica: usuarios solo abre el 443 de la DMZ; la DMZ solo abre el 5432 de
   servidores; servidores no inicia nada. Por eso, si comprometen el web de la DMZ (la zona
   expuesta a internet), desde ahí solo se alcanza el puerto 5432 de servidores, no todo el
   parque: la DMZ **no** puede llegar al resto de servidores ni a los usuarios, y servidores no
   puede "volver" hacia la DMZ por su cuenta. Eso es frenar el movimiento lateral.

3. Levanta un servicio reconocible en cada zona de destino. Usa una terminal por servicio y
   déjalas abiertas.

   **Terminal 1 (web de la DMZ):**
   ```bash
   sudo python3 -m http.server 443 --bind 10.0.1.1
   ```
   **Terminal 2 (base de datos de servidores):**
   ```bash
   python3 -m http.server 5432 --bind 10.0.20.1
   ```
   - `python3 -m http.server` → servidor web mínimo de la biblioteca estándar; aquí solo nos
     importa que deje un puerto TCP escuchando, no lo que sirva.
   - `443` / `5432` → puerto TCP en el que escucha. El 443 está por debajo de 1024, así que su
     terminal necesita `sudo`; el 5432 no.
   - `--bind 10.0.1.1` / `--bind 10.0.20.1` → ata el servidor a la IP de esa zona, no a todas las
     interfaces; así el destino del tráfico es la IP de la zona y nftables puede clasificarlo.

4. Desde una tercera terminal, prueba la política con `nc`, forzando el origen de cada zona.
   `nc` solo comprueba si el TCP conecta; es la forma limpia de "ser" cada zona en un solo equipo.

   **Terminal 3 (cliente):**
   ```bash
   # usuarios -> DMZ:443  => PERMITIDO
   nc -z -v -w2 -s 10.0.10.1 10.0.1.1 443
   # DMZ -> servidores:5432  => PERMITIDO
   nc -z -v -w2 -s 10.0.1.1 10.0.20.1 5432
   # usuarios -> servidores:5432  => BLOQUEADO
   nc -z -v -w2 -s 10.0.10.1 10.0.20.1 5432
   # DMZ -> DMZ:443 no aplica; prueba DMZ -> usuarios => BLOQUEADO
   nc -z -v -w2 -s 10.0.1.1 10.0.10.1 443
   ```
   - `nc` → netcat, navaja suiza de conexiones TCP/UDP.
   - `-z` → modo "zero-I/O": solo intenta conectar y dice si lo logró, sin enviar datos (sirve para sondear un puerto).
   - `-v` → verboso: imprime `Connection to ... succeeded!` si conecta, o el fallo si no.
   - `-w2` → espera como máximo 2 segundos; sin esto, un puerto bloqueado (paquete descartado) dejaría a `nc` colgado.
   - `-s 10.0.10.1` → fuerza la IP de ORIGEN del paquete; así el paquete sale "desde la zona de usuarios" aunque todo corra en el mismo equipo. Es la pieza clave del ejercicio: cambiando `-s` te conviertes en una zona u otra.
   - `10.0.1.1 443` → destino y puerto.

   Lo permitido imprime `succeeded!`; lo bloqueado da `Connection timed out` tras los 2 segundos
   (el paquete se descartó, no hubo respuesta: igual que un `filtered`).

5. Confirma el bloqueo también con `ping`: la política no permite ICMP entre zonas, así que
   cualquier ping entre zonas debe fallar.
   ```bash
   ping -c1 -W1 -I 10.0.10.1 10.0.1.1
   ping -c1 -W1 -I 10.0.10.1 10.0.20.1
   ```
   - `ping` → envía paquetes ICMP de eco y espera la respuesta.
   - `-c1` → envía un solo paquete y termina.
   - `-W1` → espera como máximo 1 segundo la respuesta.
   - `-I 10.0.10.1` → usa esa IP como origen; igual que el `-s` de `nc`, te sitúa en la zona de usuarios.
   - `10.0.1.1` / `10.0.20.1` → destino: la DMZ y servidores.
   Esperado: `100% packet loss` en ambos, porque ICMP no está en las reglas de permitir y cae en
   el `drop`.

6. (Opcional) Escanea una zona con nmap para ver la foto de puertos. Necesita root y la interfaz
   de origen de la zona.
   ```bash
   sudo nmap -Pn -p 443,5432,22 -e zonausuarios -S 10.0.10.1 10.0.1.1
   ```
   - `nmap` → escáner de puertos.
   - `-Pn` → no hace descubrimiento previo por ping y trata el host como encendido; necesario porque el ICMP entre zonas está bloqueado.
   - `-p 443,5432,22` → la lista de puertos a mirar.
   - `-e zonausuarios` → envía por esa interfaz (la zona de usuarios).
   - `-S 10.0.10.1` → usa esa IP de origen, para "ser" la zona de usuarios.
   - `10.0.1.1` → objetivo: la DMZ.
   La política predice `443/tcp open` (permitido) y `5432`/`22` como `filtered` (descartados). Si
   nmap entrega el escaneo por loopback y el resultado no cuadra, quédate con la comprobación de
   `nc`/`ping` de los pasos 4 y 5, que es la fiable en un solo equipo.

### Resultado esperado

Las pruebas con `nc` y `ping` que demuestran la política: desde usuarios solo conecta el 443
del web de la DMZ y nada de servidores; desde la DMZ solo conecta el 5432 de la base de datos;
los `ping` entre zonas dan 100% de pérdida. La misma IP se comporta distinto según desde qué
zona la toques: eso es la segmentación frenando el movimiento lateral, montada entera en un
solo equipo.

### Comprueba que lo lograste

- ¿Por qué lo bloqueado se ve `timed out` / `filtered` y no un rechazo inmediato? Porque la
  regla es `drop` (descarta el paquete sin contestar), igual que un firewall; un `closed` sería
  el host respondiendo con un RST, sin filtro de por medio.
- Si comprometieran el web de la DMZ, ¿podría alcanzar toda la zona de servidores? No: solo el
  5432 de `10.0.20.1`, que es justo lo que la regla permite. El resto de servidores y los
  usuarios quedan fuera de su alcance.
- ¿Por qué la DMZ no puede llegar a servidores salvo en el 5432, ni servidores iniciar nada?
  Porque la DMZ es la zona expuesta a internet y es la que más probablemente caiga; se le da el
  mínimo (un solo puerto de un solo destino), y a servidores no se le deja abrir conexiones
  hacia fuera para que un servidor comprometido no "salte" a otra zona.

### Limpieza

```bash
sudo nft delete table inet zonas
sudo ip link del zonadmz
sudo ip link del zonausuarios
sudo ip link del zonaservidores
rm -f ~/zonas.nft
```
- `sudo nft delete table inet zonas` → borra la tabla de reglas que cargaste; como no se guardó en `/etc/nftables.conf`, con esto el filtrado desaparece (y al reiniciar tampoco volvería).
- `sudo ip link del zonaX` → elimina cada interfaz dummy; al ser virtuales, desaparecen sin dejar rastro y su IP se va con ellas.
- `rm -f ~/zonas.nft` → borra el archivo de reglas que escribiste.

Cierra con Ctrl+C las terminales 1 y 2 de los servidores. Al estar todo montado con interfaces
dummy y reglas en memoria, no queda nada tocando tu red real.

## Ejercicio 4: Escribe un runbook de restauración y pruébalo

Nodo: Runbooks ([README](README.md#understand-concept-of-runbooks)).

Objetivo: convertir "restaurar mi carpeta de proyectos desde backup" en un runbook de una
página con disparador, requisitos, pasos atómicos y verificación por paso, y validarlo
haciendo que otra persona lo siga sin tu ayuda; cada duda que tenga es un paso que falta.

Necesitas: un backup real que puedas restaurar (vale el repositorio restic del ejercicio 1).
Un editor de texto. Otra persona para la prueba. 30 minutos.

### Pasos

1. Escribe el runbook siguiendo la estructura de la nota (título, disparador, requisitos,
   pasos atómicos, verificación, escalado, rollback). Guárdalo como `~/runbook-restore.md`.
   ```
   Título:       Restaurar la carpeta de proyectos desde backup
   Disparador:   la carpeta ~/proyectos se borró, se corrompió o se cifró
   Requisitos:   acceso al equipo, contraseña del repositorio restic, repo accesible
   Pasos:
     1. Verificar que el repo responde:  restic -r ~/restic-local snapshots
        Verificación: aparece al menos un snapshot con su fecha.
     2. Elegir el snapshot a restaurar (normalmente el último, "latest").
        Verificación: anotar el ID y la hora; esa hora es lo que se recupera.
     3. Restaurar a una carpeta nueva, no encima del original:
        restic -r ~/restic-local restore latest --target ~/proyectos-restore
        Verificación: el comando termina sin error.
     4. Comparar tamaño y contar archivos:
        du -sh ~/proyectos-restore  y  ls ~/proyectos-restore | wc -l
        Verificación: coinciden con lo esperado.
     5. Mover la carpeta restaurada a su sitio solo tras comprobar el paso 4.
   Escalado:     si "restic check" da errores de repositorio -> avisar al responsable del backup.
   Rollback:     no se sobreescribe el original hasta validar; si algo falla, se descarta
                 ~/proyectos-restore y se repite.
   ```
   - `restic -r ~/restic-local` → (ver ejercicio 1, paso 2).
   - `snapshots` → lista los snapshots del repositorio con su ID, fecha, host y rutas.
   - `restore latest --target ~/proyectos-restore` → (ver ejercicio 1, paso 6).
   - `du -sh` → (ver ejercicio 1, paso 1).
   - `ls ~/proyectos-restore | wc -l` → `ls` lista los nombres del primer nivel de la carpeta, uno por línea al ir a una tubería; `|` pasa esa lista a `wc`, y `-l` cuenta las líneas, es decir, cuántas entradas hay (no cuenta los archivos de subcarpetas ni los ocultos).
   - `restic check` → (ver ejercicio 1, paso 7).

2. Pide a otra persona que siga el runbook en tu equipo (o en una VM) sin que tú hables.
   Tú solo observas y apuntas cada vez que duda, pregunta o se detiene.

3. Cada duda es un defecto del runbook: una ruta que no estaba, una contraseña que no supo
   de dónde sacar, un comando ambiguo. Corrige el documento para cerrar esa duda y vuelve a
   probar con otra persona si puedes.

### Resultado esperado

Un `runbook-restore.md` de una página que una persona cualificada pero sin contexto ejecuta
de principio a fin sin preguntarte nada, y la carpeta de proyectos restaurada por esa persona.

### Comprueba que lo lograste

- ¿La persona llegó al final sin pedirte ayuda? Si preguntó algo, ese paso aún no era atómico
  o faltaba un requisito; el runbook no está terminado hasta que no hay preguntas.
- ¿Cada paso tiene su verificación? Un paso sin criterio de "salió bien" no sirve en un
  incidente a las 3:00.

### Limpieza

```bash
rm -rf ~/proyectos-restore
```
- `rm -rf` → (ver ejercicio 1, paso 5): borra la carpeta restaurada y todo su contenido.
