# Ejercicios: Ataques y amenazas

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo se hace sobre equipos, cuentas y logs tuyos. Los intentos fallidos se generan contra tu
propia máquina o VM, nunca contra equipos de terceros.

## Ejercicio 1: Distingue fuerza bruta de password spray en tus logs

Nodo: Brute Force vs Password Spray ([README](README.md#brute-force-vs-password-spray)).

Objetivo: a partir de los logs de autenticación de tu propio equipo o VM, contar fallos por
cuenta y cuentas distintas por IP de origen, y decidir con esos dos números si un patrón se
parece más a fuerza bruta (muchos fallos en una cuenta) o a password spray (un fallo en cada
una de muchas cuentas desde el mismo origen).

Necesitas: una VM o equipo Linux con `sshd` corriendo (o, en Windows, el visor de eventos con
el ID 4625). Acceso a `journalctl`. 30 minutos. En Windows harías lo mismo filtrando el evento
4625 por campo "Cuenta" y "Dirección de red de origen".

### Pasos

1. Genera unos pocos fallos de autenticación contra tu propia máquina, para tener datos
   reproducibles. Simula primero un patrón de fuerza bruta: varios intentos fallidos contra una
   sola cuenta. Responde con una contraseña incorrecta cada vez.
   ```bash
   for i in 1 2 3 4 5; do ssh ana@localhost -o PreferredAuthentications=password \
     -o PubkeyAuthentication=no -o NumberOfPasswordPrompts=1 true; done
   ```
   - `for i in 1 2 3 4 5; do ...; done` → bucle de la shell que repite el comando cinco veces; `i` solo cuenta las vueltas.
   - `\` → al final de la línea, continúa el comando en la línea siguiente.
   - `ssh` → cliente SSH que abre una sesión en otra máquina (aquí, la tuya).
   - `ana@localhost` → usuario `ana` en el equipo `localhost` (127.0.0.1, tu propia máquina).
   - `-o` → pasa una opción de configuración del cliente (las mismas de `ssh_config`) solo para esta conexión.
   - `PreferredAuthentications=password` → orden de métodos de autenticación a probar; aquí solo contraseña.
   - `PubkeyAuthentication=no` → no intenta autenticarse con clave pública, para que el fallo sea de contraseña.
   - `NumberOfPasswordPrompts=1` → pide la contraseña una sola vez antes de rendirse (por defecto son 3), así cada vuelta deja un fallo.
   - `true` → comando remoto que se ejecutaría si el login funcionara; no hace nada y termina.

   (La cuenta `ana` puede no existir o puedes usar una tuya; lo que importa es que falle y quede
   registrado.)

2. Simula ahora un patrón de spray: un intento fallido contra varias cuentas distintas.
   ```bash
   for u in ana bruno carla diego elena; do ssh "$u@localhost" \
     -o PreferredAuthentications=password -o PubkeyAuthentication=no \
     -o NumberOfPasswordPrompts=1 true; done
   ```
   - `for u in ana bruno carla diego elena; do ...; done` → bucle que recorre los cinco nombres y guarda cada uno en la variable `u`.
   - `"$u@localhost"` → usuario de la vuelta actual en tu propia máquina; las comillas evitan que la shell parta el valor.
   - `ssh`, `\`, `-o PreferredAuthentications=password`, `-o PubkeyAuthentication=no`, `-o NumberOfPasswordPrompts=1`, `true` → (ver paso 1).

3. Saca de los logs los pares "usuario origen" de cada fallo.
   ```bash
   journalctl -u sshd --no-pager | \
     sed -nE 's/.*Failed password for (invalid user )?([^ ]+) from ([0-9.]+).*/\2 \3/p' > ~/fallos.txt
   wc -l ~/fallos.txt
   ```
   - `journalctl` → lee el journal de systemd, donde quedan los logs del sistema.
   - `-u sshd` → muestra solo los mensajes de esa unidad. El nombre es `ssh` en Debian/Ubuntu y `sshd` en Arch; usa el que tengas.
   - `--no-pager` → vuelca la salida entera sin abrir el paginador (`less`), necesario para pasarla a otro comando.
   - `|` → tubería: la salida del comando de la izquierda entra como entrada del de la derecha.
   - `sed` → editor de flujo que transforma el texto línea a línea.
   - `-n` → no imprime nada salvo lo que se pida explícitamente con `p`.
   - `-E` → usa expresiones regulares extendidas (paréntesis y `?` sin escapar).
   - `s/patrón/reemplazo/p` → sustituye lo que casa con el patrón por el reemplazo; la `p` final imprime la línea solo si hubo sustitución, así quedan solo las líneas de fallo.
   - `.*Failed password for` → casa cualquier texto hasta el mensaje de contraseña fallida de `sshd`.
   - `(invalid user )?` → grupo 1, opcional: aparece cuando el usuario no existe en el sistema.
   - `([^ ]+)` → grupo 2: el nombre de usuario (caracteres hasta el siguiente espacio).
   - `from ([0-9.]+)` → grupo 3: la IP de origen (dígitos y puntos, IPv4).
   - `.*` → el resto de la línea (puerto, protocolo), que se descarta.
   - `\2 \3` → reemplazo: deja solo "usuario IP" separados por un espacio.
   - `> ~/fallos.txt` → redirige la salida al archivo `fallos.txt` de tu carpeta personal, sobrescribiéndolo.
   - `wc` → cuenta líneas, palabras o bytes de un archivo.
   - `-l` → cuenta solo líneas: el número de fallos extraídos.
   - `~/fallos.txt` → archivo que se cuenta.

4. Cuenta los fallos por cuenta. Si una cuenta acumula muchos fallos, huele a fuerza bruta.
   ```bash
   awk '{print $1}' ~/fallos.txt | sort | uniq -c | sort -rn
   ```
   - `awk '{print $1}' ~/fallos.txt` → `awk` procesa el archivo por campos separados por espacios; `{print $1}` imprime solo el primer campo de cada línea, el usuario.
   - `|` → (ver paso 3).
   - `sort` → ordena las líneas alfabéticamente, para que los usuarios repetidos queden juntos.
   - `uniq` → colapsa líneas consecutivas iguales en una sola.
   - `-c` → antepone a cada línea cuántas veces se repetía: fallos por cuenta.
   - `sort -rn` → ordena de nuevo la salida: `-n` compara como número (el recuento) y `-r` invierte el orden, de mayor a menor.
   Esperado del paso 1: la cuenta atacada (p. ej. `ana`) con 5 fallos destacando sobre el resto.

5. Cuenta cuántas cuentas distintas falla cada IP de origen. Si una IP toca muchas cuentas con
   uno o pocos fallos cada una, huele a spray.
   ```bash
   awk '{print $2, $1}' ~/fallos.txt | sort -u | awk '{print $1}' | sort | uniq -c | sort -rn
   ```
   - `awk '{print $2, $1}' ~/fallos.txt` → invierte los campos y escribe "IP usuario"; la coma de `print` los separa con un espacio.
   - `sort -u` → ordena y `-u` deja una sola copia de cada línea repetida: cada par IP-usuario cuenta una vez, aunque fallara varias veces.
   - `awk '{print $1}'` → (ver paso 4); aquí el primer campo es la IP.
   - `sort | uniq -c | sort -rn` → (ver paso 4); ahora el recuento es de cuentas distintas por IP.
   Esperado del paso 2: `127.0.0.1` (o tu IP) con 5 cuentas distintas.

### Resultado esperado

Dos recuentos sacados de tus propios logs: uno que muestra una cuenta con muchos fallos (fuerza
bruta) y otro que muestra una IP tocando muchas cuentas distintas (spray). Con eso puedes
etiquetar cada patrón como dice la nota.

### Comprueba que lo lograste

- ¿Cuál es la señal de la fuerza bruta y cuál la del spray? Fuerza bruta: número alto de fallos
  por cuenta. Spray: número alto de cuentas distintas falladas desde un mismo origen, aunque
  cada cuenta tenga un solo fallo.
- ¿Por qué una alerta que solo cuenta fallos por cuenta no ve nunca un spray? Porque en el spray
  cada cuenta falla una sola vez; hay que contar por origen y por número de cuentas distintas.
- ¿Qué defensa deja inútil la contraseña adivinada en ambos casos? MFA; además, bloqueo por
  cuenta contra la fuerza bruta y detección por origen contra el spray.

### Limpieza

```bash
rm -f ~/fallos.txt
```
- `rm` → borra archivos.
- `-f` → no pide confirmación y no da error si el archivo ya no existe.
- `~/fallos.txt` → el archivo de trabajo creado en el paso 3.
