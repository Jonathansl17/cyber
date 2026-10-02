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
   (La cuenta `ana` puede no existir o puedes usar una tuya; lo que importa es que falle y quede
   registrado.)

2. Simula ahora un patrón de spray: un intento fallido contra varias cuentas distintas.
   ```bash
   for u in ana bruno carla diego elena; do ssh "$u@localhost" \
     -o PreferredAuthentications=password -o PubkeyAuthentication=no \
     -o NumberOfPasswordPrompts=1 true; done
   ```

3. Saca de los logs los pares "usuario origen" de cada fallo. El nombre de la unidad es `ssh`
   en Debian/Ubuntu y `sshd` en Arch; usa el que tengas.
   ```bash
   journalctl -u sshd --no-pager | \
     sed -nE 's/.*Failed password for (invalid user )?([^ ]+) from ([0-9.]+).*/\2 \3/p' > ~/fallos.txt
   wc -l ~/fallos.txt
   ```

4. Cuenta los fallos por cuenta. Si una cuenta acumula muchos fallos, huele a fuerza bruta.
   ```bash
   awk '{print $1}' ~/fallos.txt | sort | uniq -c | sort -rn
   ```
   Esperado del paso 1: la cuenta atacada (p. ej. `ana`) con 5 fallos destacando sobre el resto.

5. Cuenta cuántas cuentas distintas falla cada IP de origen. Si una IP toca muchas cuentas con
   uno o pocos fallos cada una, huele a spray.
   ```bash
   awk '{print $2, $1}' ~/fallos.txt | sort -u | awk '{print $1}' | sort | uniq -c | sort -rn
   ```
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
