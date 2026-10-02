# Ejercicios: Programación y hacking práctico

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se ejecuta sobre equipos y servidores tuyos. Los scripts de esta nota son
defensivos: miden la integridad de archivos, los puertos que escuchan en tu propia
máquina y los intentos de login contra tus servicios.

## Ejercicio 1: Corre los cinco scripts de la nota y prográmalos

Nodo: [Programming Skills](README.md#programming-skills) (Python, Go, JavaScript, Bash,
PowerShell).

Objetivo: poner en marcha los cinco scripts del README (integridad por hashes, puertos
abiertos, 404 en el log web, fallos de SSH y eventos 4625 de Windows) contra tus propios
equipos, y dejar cada uno ejecutándose solo con cron (Linux) o el Programador de tareas
de Windows, avisándote cuando haya algo anómalo.

Necesitas: tu Linux de siempre o una VM Linux tuya con Python 3, Go y Node.js; para el
último script, un Windows tuyo. Para el aviso por correo, un cliente de correo de línea
de comandos (`msmtp` o `mailx`). Tiempo: 2-3 h. Instalación de los intérpretes:
- Debian/Ubuntu: `sudo apt install python3 golang nodejs msmtp bsd-mailx`.
- Arch: `sudo pacman -S python go nodejs msmtp s-nail`.

Crea una carpeta para los scripts y sus salidas:
```bash
mkdir -p ~/segscripts/logs && cd ~/segscripts
```

### Pasos

1. Script de integridad (Python). Copia `check_hashes.py` del README a
   `~/segscripts/check_hashes.py`. Crea una línea base de los archivos que quieras
   vigilar y pruébalo:
   ```bash
   sha256sum /etc/passwd /etc/hosts > base.txt
   python3 check_hashes.py base.txt ; echo "codigo de salida: $?"
   ```
   Recién hecha la base, todo debe salir `OK` y el código de salida `0`. Cambia a drede
   un archivo de prueba y vuelve a correrlo: debe marcar `CHANGED` y salir con `1`.
2. Script de puertos (Go). Copia `portcheck` del README a `~/segscripts/portcheck.go`,
   compílalo y apúntalo a tu propia máquina:
   ```bash
   go build -o portcheck portcheck.go
   ./portcheck 127.0.0.1
   ```
   Debe listar los puertos tuyos que están escuchando. Contrástalo con la foto real:
   ```bash
   ss -tlnp
   ```
3. Script de 404 (JavaScript/Node). Copia `flag404.js` a `~/segscripts/flag404.js`. Si
   tienes un servidor web propio, apúntalo a su log; si no, crea un log de prueba con el
   formato combinado para ver la lógica:
   ```bash
   node flag404.js /var/log/nginx/access.log
   ```
   Lista las IP con muchos 404, la huella de una enumeración web. Ajusta el umbral
   `THRESHOLD` a tu tráfico real.
4. Script de fallos SSH (Bash). Copia `top_ssh_failures.sh`, dale permiso y córrelo en
   tu servidor con SSH:
   ```bash
   chmod +x top_ssh_failures.sh
   ./top_ssh_failures.sh
   ```
   Debe mostrar las cinco IP con más `Failed password` del último día. Si tu servicio
   no se llama `ssh`, ajusta `-u ssh` por `-u sshd`.
5. Script de eventos 4625 (PowerShell, en tu Windows). Copia `failed_logons.ps1` y
   ejecútalo en una consola de PowerShell como administrador:
   ```powershell
   powershell -ExecutionPolicy Bypass -File .\failed_logons.ps1
   ```
   Lista las cuentas con más inicios de sesión fallidos de las últimas 24 h.
6. Programa los scripts de Linux con cron. Edita tu crontab:
   ```bash
   crontab -e
   ```
   Añade estas líneas (ajusta rutas y tu correo). `MAILTO` hace que cron te envíe por
   correo cualquier salida que el script imprima, y los scripts solo imprimen cuando
   hay algo que mirar:
   ```cron
   MAILTO="tu-correo@ejemplo.com"
   # integridad de archivos cada hora; cron avisa solo si el codigo de salida no es 0
   0 * * * * cd /home/USUARIO/segscripts && python3 check_hashes.py base.txt
   # puertos abiertos cada dia a las 7:00, guardando historico
   0 7 * * * /home/USUARIO/segscripts/portcheck 127.0.0.1 >> /home/USUARIO/segscripts/logs/puertos.log 2>&1
   # top de fallos SSH cada dia a las 8:00
   0 8 * * * /home/USUARIO/segscripts/top_ssh_failures.sh
   # IPs con muchos 404 cada dia a las 8:05
   5 8 * * * node /home/USUARIO/segscripts/flag404.js /var/log/nginx/access.log
   ```
   Para que `MAILTO` funcione, configura `msmtp` como `sendmail` (archivo
   `~/.msmtprc` con tu servidor SMTP) o, como alternativa, cambia cada línea para que
   el aviso lo mande el propio script, por ejemplo:
   `... | mail -s "Fallos SSH" tu-correo@ejemplo.com`.
7. Programa el script de Windows con el Programador de tareas. En una PowerShell de
   administrador, registra una tarea diaria:
   ```powershell
   $accion = New-ScheduledTaskAction -Execute "powershell.exe" `
     -Argument "-ExecutionPolicy Bypass -File C:\segscripts\failed_logons.ps1"
   $disparador = New-ScheduledTaskTrigger -Daily -At 8am
   Register-ScheduledTask -TaskName "Fallos4625" -Action $accion -Trigger $disparador `
     -Description "Resumen diario de logins fallidos (4625)"
   ```
   Para el aviso por correo desde Windows, haz que el `.ps1` guarde su salida en un
   archivo y la envíe con tu propio flujo SMTP (nota: `Send-MailMessage` está obsoleto;
   usa un cliente SMTP o un script con `System.Net.Mail` apuntando a tu servidor).

### Resultado esperado

Los cinco scripts corriendo a mano con salida correcta, y cuatro entradas de cron más
una tarea del Programador de tareas que los ejecutan solos y te avisan cuando hay algo
anómalo.

### Comprueba que lo lograste

- ¿`check_hashes.py` devuelve código de salida `0` cuando nada cambió y `1` cuando
  modificas un archivo? Confírmalo con `echo $?`.
- ¿Las tareas quedaron registradas? En Linux: `crontab -l` las muestra. En Windows:
  `Get-ScheduledTask -TaskName "Fallos4625"` debe aparecer con estado `Ready`.
- ¿Recibes un correo (o una entrada en el log) la primera vez que hay un cambio de hash
  o fallos de SSH? Fuerza un fallo de SSH con un login erróneo a tu propio servidor y
  espera la siguiente ejecución.

### Limpieza

```bash
crontab -e      # borra las lineas que anadiste
```
En Windows: `Unregister-ScheduledTask -TaskName "Fallos4625" -Confirm:$false`. Borra la
carpeta `~/segscripts` y sus logs si no quieres conservarlos.
