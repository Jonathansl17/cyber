# Ejercicios: Conceptos de seguridad

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se hace sobre equipos, cuentas, discos y máquinas virtuales tuyos. Cuando hay varias
zonas de red, se montan en una red aislada del laboratorio (host-only o interna), nunca
contra equipos o servicios de terceros.

## Ejercicio 1: Aplica la regla 3-2-1 y mide tu RPO y RTO

Nodo: Backups and Resiliency ([README](README.md#understand-backups-and-resiliency)).

Objetivo: tener los mismos datos en 3 copias, 2 soportes y 1 fuera del sitio con restic,
borrar el original, restaurarlo y obtener dos números tuyos: cuánto perderías (RPO, fijado
por la hora del último snapshot) y cuánto tardas en volver (RTO, el tiempo real de restauración).

Necesitas: Linux (puede ser tu equipo o una VM). Un pendrive USB. Un segundo equipo o VM
accesible por SSH para la copia fuera del sitio (vale otra VM en red interna). Paquete
`restic`. Unos 30 minutos.

- Debian/Ubuntu: `sudo apt update && sudo apt install restic`
- Arch: `sudo pacman -S restic`

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
   sudo mount /dev/sdX1 /mnt/usb          # sustituye sdX1 por tu partición
   restic -r /mnt/usb/restic-usb init
   restic -r /mnt/usb/restic-usb backup ~/datos-prueba
   ```
   - `lsblk` → lista los discos y particiones (dispositivos de bloque) con su tamaño y punto de montaje; sin opciones muestra el árbol completo.
   - `sudo` → ejecuta el comando siguiente como root; montar un sistema de archivos lo exige.
   - `mount` → engancha un sistema de archivos en una carpeta del árbol.
   - `/dev/sdX1` → la partición del pendrive (primer argumento: qué montar).
   - `/mnt/usb` → carpeta donde aparecerá su contenido (segundo argumento: dónde montarlo); debe existir antes.
   - `# sustituye ...` → comentario de bash, no se ejecuta.
   - `restic -r /mnt/usb/restic-usb` → mismo `-r` del paso 2, ahora con el repositorio dentro del USB.
   - `init` → (ver paso 2).
   - `backup ~/datos-prueba` → (ver paso 2).

   Dos soportes distintos: si muere el disco del equipo, el USB sobrevive, y al revés.

4. La copia "1 fuera del sitio": manda un backup por SFTP al segundo equipo o VM. Cambia
   `usuario` y `host-lab` por los de tu otra máquina.
   ```bash
   restic -r sftp:usuario@host-lab:/home/usuario/restic-offsite init
   restic -r sftp:usuario@host-lab:/home/usuario/restic-offsite backup ~/datos-prueba
   ```
   - `-r sftp:usuario@host-lab:/home/usuario/restic-offsite` → repositorio remoto por SFTP: `sftp:` es el tipo de backend, `usuario@host-lab` la cuenta y máquina a la que restic entra por SSH, y lo que va tras los dos puntos es la ruta del repositorio en esa máquina.
   - `init` → (ver paso 2).
   - `backup ~/datos-prueba` → (ver paso 2).

   Ya cumples 3-2-1: 3 copias (original + 2 repos), 2 soportes (disco y USB) y 1 fuera del sitio.

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
- ¿Sobrevive el esquema a que se moje el equipo entero? Sí: queda la copia del USB y la de
  fuera del sitio.

### Limpieza

```bash
rm -rf ~/restic-local ~/datos-restaurados
restic -r /mnt/usb/restic-usb forget --prune latest 2>/dev/null; sudo umount /mnt/usb
restic -r sftp:usuario@host-lab:/home/usuario/restic-offsite forget --prune latest
```
- `rm -rf` → (ver paso 5): borra el repositorio local y la carpeta restaurada.
- `restic -r ...` → (ver pasos 2, 3 y 4): elige el repositorio del USB o el remoto.
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

## Ejercicio 3: Segmenta una red de laboratorio y compruébala con nmap

Nodo: Perimiter vs DMZ vs Segmentation ([README](README.md#perimiter-vs-dmz-vs-segmentation)).

Objetivo: montar tres zonas (DMZ, usuarios, servidores) separadas por un firewall, cargar
las reglas de la nota y demostrar con nmap desde cada zona que solo ves los puertos que la
política permite y que todo lo demás sale filtrado.

Necesitas: un hipervisor (VirtualBox o Proxmox). Un firewall pfSense o OPNsense como VM con
tres interfaces (una por zona), todas en redes internas o host-only, sin salida real a
internet. Tres VMs ligeras (una por zona) con `nmap` instalado. Unas dos horas la primera vez.

- Debian/Ubuntu (en las VMs cliente): `sudo apt install nmap`
- Arch: `sudo pacman -S nmap`

### Pasos

1. En el hipervisor crea tres redes internas aisladas y nómbralas `dmz`, `usuarios`,
   `servidores`. En VirtualBox, para cada VM: Configuración, Red, Conectado a "Red interna",
   y escribe el nombre de la red. Ninguna VM debe tener NAT ni puente a tu red real.

2. Da al firewall (pfSense/OPNsense) tres tarjetas de red, una en cada red interna, y asigna
   direcciones de ejemplo: DMZ `10.0.1.1/24`, usuarios `10.0.10.1/24`, servidores
   `10.0.20.1/24`. Conecta cada VM cliente a su red y dale una IP de esa subred (por ejemplo
   el servidor web de la DMZ `10.0.1.10`, un PC de usuarios `10.0.10.10`, la base de datos de
   servidores `10.0.20.10`).

3. Levanta servicios reconocibles para escanear: en la VM DMZ abre un web en el 443, en la VM
   servidores abre un postgres simulado en el 5432.
   ```bash
   # VM DMZ (10.0.1.10)
   python3 -m http.server 443
   # VM servidores (10.0.20.10)
   python3 -m http.server 5432
   ```
   - `# VM DMZ ...` → comentarios que indican en qué VM va cada comando; no se ejecutan.
   - `python3` → intérprete de Python 3.
   - `-m http.server` → ejecuta el módulo `http.server` de la biblioteca estándar como programa: un servidor web mínimo que sirve los archivos de la carpeta actual por HTTP sin cifrar (no habla TLS aunque escuche en el 443).
   - `443` / `5432` → puerto TCP en el que escucha (por defecto 8000) en todas las interfaces. Los puertos por debajo de 1024, como el 443, solo se pueden abrir como root, así que en la VM DMZ ejecútalo con `sudo`.

4. En el firewall, crea las reglas de la nota, denegando por defecto. En pfSense: Firewall,
   Rules, una pestaña por interfaz. Traduce así la tabla del README:
   ```
   WAN/Internet -> DMZ   : permitir TCP 443 a 10.0.1.10; denegar el resto
   WAN/Internet -> LAN   : denegar todo
   DMZ          -> SERV  : permitir TCP 5432 a 10.0.20.10; denegar el resto
   USUARIOS     -> GESTION/SERV: denegar lo que la nota prohíbe
   ```
   - `WAN/Internet -> DMZ` → regla en la interfaz WAN: solo deja pasar TCP al 443 del servidor web `10.0.1.10`; cualquier otro tráfico hacia la DMZ se bloquea.
   - `WAN/Internet -> LAN` → desde fuera no se permite ninguna conexión iniciada hacia la red interna.
   - `DMZ -> SERV` → regla en la interfaz DMZ: el servidor web solo puede abrir TCP 5432 (la base de datos) hacia `10.0.20.10`; nada más hacia servidores.
   - `USUARIOS -> GESTION/SERV` → regla en la interfaz de usuarios: bloquea los accesos a gestión y servidores que la tabla del README marca como prohibidos.

   Deja activa la regla implícita de denegar al final de cada interfaz.

5. Escanea desde la zona de usuarios hacia las otras dos. Debes ver filtrado casi todo.
   ```bash
   # desde la VM usuarios (10.0.10.10)
   nmap -Pn -p 443,5432,22,80 10.0.1.10 10.0.20.10
   ```
   - `# desde la VM usuarios ...` → comentario: indica dónde ejecutarlo.
   - `nmap` → escáner de puertos.
   - `-Pn` → no hace el descubrimiento previo (ping) y trata los hosts como encendidos; sin esto, un firewall que bloquea el ping haría que nmap diera el host por caído y no escaneara.
   - `-p 443,5432,22,80` → escanea solo esa lista de puertos.
   - `10.0.1.10 10.0.20.10` → los objetivos: el web de la DMZ y la base de datos de servidores.
   - Sin opción de tipo de escaneo, nmap hace un SYN scan si eres root y un connect scan (`-sT`) si no.
   Lo esperado: los puertos no permitidos salen `filtered` (el firewall los descarta), no
   `closed` (eso sería el host rechazando sin firewall).

6. Escanea desde la DMZ hacia servidores: solo debe verse el 5432 permitido.
   ```bash
   # desde la VM DMZ (10.0.1.10)
   nmap -Pn -p 443,5432,22 10.0.20.10
   ```
   - `-Pn` → (ver paso 5).
   - `-p 443,5432,22` → (ver paso 5); ahora con tres puertos.
   - `10.0.20.10` → objetivo: solo la base de datos de servidores.
   Esperado: `5432/tcp open`, el resto `filtered`.

### Resultado esperado

Tres capturas o salidas de nmap que demuestran la política: desde usuarios no llegas a los
puertos de servidores salvo lo permitido, desde la DMZ solo alcanzas el 5432 de la base de
datos, y nada de internet simulado entra a la red interna. El mismo host se ve distinto según
desde qué zona lo mires: eso es la segmentación frenando el movimiento lateral.

### Comprueba que lo lograste

- ¿Un puerto bloqueado por el firewall sale `filtered` o `closed` en nmap? `filtered`: el
  firewall descarta el paquete y nmap no recibe respuesta. `closed` sería el propio host
  contestando con un RST, sin firewall de por medio.
- Si comprometieran la VM de la DMZ, ¿podría escanear toda la red de servidores? No: solo
  alcanzaría el 5432 de `10.0.20.10`, que es justo lo que la regla permite.

### Limpieza

Apaga y borra las tres VMs cliente y el firewall, y elimina las tres redes internas del
hipervisor. Al estar todo en redes aisladas, no queda nada tocando tu red real.

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
