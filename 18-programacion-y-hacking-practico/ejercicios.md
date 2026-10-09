# Ejercicios: Programación y hacking práctico

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se ejecuta sobre equipos y servidores tuyos. Los scripts de esta nota son
defensivos: miden la integridad de archivos, los puertos que escuchan en tu propia
máquina y los intentos de login contra tus servicios.

## Ejercicio 1: Corre los cuatro scripts de la nota y prográmalos

Nodo: [Programming Skills](README.md#programming-skills) (Python, Go, JavaScript, Bash).

Objetivo: poner en marcha los cuatro scripts del README (integridad por hashes, puertos
abiertos, 404 en el log web y fallos de SSH) contra tus propios equipos, y dejar cada
uno ejecutándose solo con cron, avisándote cuando haya algo anómalo.

Necesitas: tu propio equipo Linux con Python 3, Go y Node.js. Para el
aviso por correo, un cliente de correo de línea
de comandos (`msmtp` o `mailx`). Tiempo: 2-3 h. Instalación de los intérpretes:
- Debian/Ubuntu: `sudo apt install python3 golang nodejs msmtp bsd-mailx`.
- Arch: `sudo pacman -S python go nodejs msmtp s-nail`.

Crea una carpeta para los scripts y sus salidas:
```bash
mkdir -p ~/segscripts/logs && cd ~/segscripts
```

- `mkdir -p ~/segscripts/logs` → crea la carpeta y su subcarpeta `logs`; `-p` crea los directorios padre que falten y no falla si ya existen.
- `&& cd ~/segscripts` → si lo anterior tuvo éxito, entra en la carpeta de trabajo.

### Pasos

1. Script de integridad (Python). Copia `check_hashes.py` del README a
   `~/segscripts/check_hashes.py`. Crea una línea base de los archivos que quieras
   vigilar y pruébalo:
   ```bash
   sha256sum /etc/passwd /etc/hosts > base.txt
   python3 check_hashes.py base.txt ; echo "codigo de salida: $?"
   ```
   - `sha256sum /etc/passwd /etc/hosts` → calcula el hash SHA-256 de cada uno de esos archivos.
   - `> base.txt` → guarda esos hashes en el archivo de línea base.
   - `python3 check_hashes.py base.txt` → ejecuta el script con Python 3, pasándole la línea base como argumento.
   - `;` → separa comandos: ejecuta el siguiente ocurra lo que ocurra con el anterior.
   - `echo "codigo de salida: $?"` → imprime el código de salida del último comando; `$?` contiene ese valor (0 = todo bien).
   Recién hecha la base, no debe imprimir nada (solo informa de lo que no coincide) y
   el código de salida debe ser `0`. Cambia a drede
   un archivo de prueba y vuelve a correrlo: debe marcar `CHANGED` y salir con `1`.
2. Script de puertos (Go). Copia `portcheck` del README a `~/segscripts/portcheck.go`,
   compílalo y apúntalo a tu propia máquina:
   ```bash
   go build -o portcheck portcheck.go
   ./portcheck 127.0.0.1
   ```
   - `go build` → compila el programa en Go.
   - `-o portcheck` → nombra `portcheck` al binario de salida.
   - `portcheck.go` → el archivo fuente a compilar.
   - `./portcheck 127.0.0.1` → ejecuta el binario pasándole la IP a revisar como argumento.

   Debe listar los puertos tuyos que están escuchando. Contrástalo con la foto real:
   ```bash
   ss -tlnp
   ```
   - `ss` → muestra sockets de red.
   - `-t` → solo sockets TCP.
   - `-l` → solo los que están en escucha (listening).
   - `-n` → no resuelve nombres de servicio; muestra el número de puerto.
   - `-p` → muestra el proceso dueño de cada socket.

3. Script de 404 (JavaScript/Node). Copia `flag404.js` a `~/segscripts/flag404.js`. Si
   tienes un servidor web propio, apúntalo a su log; si no, crea un log de prueba con el
   formato combinado para ver la lógica:
   ```bash
   node flag404.js /var/log/nginx/access.log
   ```
   - `node` → ejecuta un programa JavaScript con Node.js.
   - `flag404.js` → el script a ejecutar.
   - `/var/log/nginx/access.log` → argumento: el log de accesos que analiza.

   Lista las IP con muchos 404, la huella de una enumeración web. Ajusta el umbral
   `THRESHOLD` a tu tráfico real.
4. Script de fallos SSH (Bash). Copia `top_ssh_failures.sh`, dale permiso y córrelo en
   tu servidor con SSH:
   ```bash
   chmod +x top_ssh_failures.sh
   ./top_ssh_failures.sh
   ```
   - `chmod +x top_ssh_failures.sh` → da permiso de ejecución al script.
   - `./top_ssh_failures.sh` → ejecuta el script desde el directorio actual.

   Debe mostrar las cinco IP con más `Failed password` del último día. Si tu servicio
   no se llama `ssh`, ajusta `-u ssh` por `-u sshd`. Leer el journal del sistema exige
   ser root o pertenecer al grupo `systemd-journal` (`sudo usermod -aG systemd-journal
   USUARIO` y vuelve a iniciar sesión); sin eso, el script no verá los fallos de SSH,
   ni a mano ni desde tu crontab.
5. Programa los scripts con cron. Edita tu crontab:
   ```bash
   crontab -e
   ```
   - `crontab` → gestiona la tabla de tareas programadas del usuario.
   - `-e` → abre esa tabla en el editor para modificarla.

   Añade estas líneas (ajusta rutas y tu correo). `MAILTO` hace que cron te envíe por
   correo cualquier salida que la tarea imprima (no mira el código de salida), así que
   cada línea debe imprimir solo cuando hay algo que mirar o mandar su salida a un log:
   ```cron
   MAILTO="tu-correo@ejemplo.com"
   # integridad de archivos cada hora; el script solo imprime (y cron solo avisa) si algo cambio
   0 * * * * cd /home/USUARIO/segscripts && python3 check_hashes.py base.txt
   # puertos abiertos cada dia a las 7:00, guardando historico
   0 7 * * * /home/USUARIO/segscripts/portcheck 127.0.0.1 >> /home/USUARIO/segscripts/logs/puertos.log 2>&1
   # top de fallos SSH cada dia a las 8:00
   0 8 * * * /home/USUARIO/segscripts/top_ssh_failures.sh
   # IPs con muchos 404 cada dia a las 8:05
   5 8 * * * node /home/USUARIO/segscripts/flag404.js /var/log/nginx/access.log
   ```
   - `MAILTO="tu-correo@ejemplo.com"` → variable de cron: a este correo se envía cualquier salida que impriman las tareas.
   - Los cinco campos antes de cada comando son, en orden: minuto, hora, día del mes, mes y día de la semana (`*` significa "cualquiera").
   - `0 * * * *` → minuto 0 de cada hora (una vez por hora); ejecuta `cd .../segscripts && python3 check_hashes.py base.txt`.
   - `&&` (en esa línea) → corre el `python3` solo si el `cd` tuvo éxito. Como
     `check_hashes.py` no imprime nada cuando todo coincide, solo recibes correo si un
     archivo cambió o falta.
   - `0 7 * * *` → a las 07:00 todos los días; corre `portcheck 127.0.0.1`.
   - `>> /home/USUARIO/segscripts/logs/puertos.log` → añade la salida al final de ese archivo (histórico).
   - `2>&1` → manda también los errores al mismo archivo.
   - `0 8 * * *` → a las 08:00 todos los días; corre `top_ssh_failures.sh`.
   - `5 8 * * *` → a las 08:05 todos los días; corre `flag404.js` sobre el log de nginx.

   Para que `MAILTO` funcione, configura `msmtp` como `sendmail` (archivo
   `~/.msmtprc` con tu servidor SMTP) o, como alternativa, cambia cada línea para que
   el aviso lo mande el propio script, por ejemplo:
   `... | mail -s "Fallos SSH" tu-correo@ejemplo.com`.

### Resultado esperado

Los cuatro scripts corriendo a mano con salida correcta, y cuatro entradas de cron que
los ejecutan solos y te avisan cuando hay algo anómalo.

### Comprueba que lo lograste

- ¿`check_hashes.py` devuelve código de salida `0` cuando nada cambió y `1` cuando
  modificas un archivo? Confírmalo con `echo $?`.
- ¿Las tareas quedaron registradas? `crontab -l` las muestra.
- ¿Recibes un correo (o una entrada en el log) la primera vez que hay un cambio de hash
  o fallos de SSH? Fuerza un fallo de SSH con un login erróneo a tu propio servidor y
  espera la siguiente ejecución.

### Limpieza

```bash
crontab -e      # borra las lineas que anadiste
```

- `crontab -e` → abre la tabla de cron en el editor para borrar a mano las líneas que añadiste (ver paso 5).
Borra la carpeta `~/segscripts` y sus logs si no quieres conservarlos.
