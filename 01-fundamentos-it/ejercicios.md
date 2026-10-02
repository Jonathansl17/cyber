# Ejercicios: Fundamentos de IT

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Debajo de cada bloque de comandos hay una lista que explica el comando y todas sus
opciones y argumentos. Cuando un comando y una opción ya se explicaron antes en este
mismo archivo, se indica con "(ver ejercicio N)".

## Ejercicio 1: Diagnostica un fallo de DNS con el método por capas

Nodo: [OS-Independent Troubleshooting](README.md#os-independent-troubleshooting) y la
idea de una variable por prueba.

Objetivo: provocar a propósito un fallo de resolución de nombres en una VM y demostrar,
con tres pruebas de `ping` encadenadas, que el problema está en el DNS y no en el cable,
el router ni el proveedor. Al terminar sabrás decir qué descartó cada prueba.

Necesitas: una máquina virtual Linux propia (Debian, Ubuntu o similar) con salida a
Internet; acceso a una terminal con permisos para editar `/etc/resolv.conf` (o parar el
servicio de red). `ping` ya viene instalado. Tiempo estimado: 20 minutos. Todo se hace
en tu propia VM; no sales a equipos ajenos.

### Pasos

1. Confirma que la red funciona antes de romper nada. Anota tu puerta de enlace.
   ```bash
   ip route | grep default
   ```
   - `ip` → herramienta estándar de Linux para ver y configurar red (interfaces, rutas, vecinos).
   - `route` → subcomando de `ip` que muestra la tabla de rutas.
   - `|` → tubería: envía la salida de `ip route` como entrada de `grep`.
   - `grep` → filtra líneas de texto por un patrón.
   - `default` → patrón que busca `grep`: deja solo la línea de la ruta por defecto.

   Verás algo como `default via 192.168.1.1 dev eth0`. Guarda esa IP del gateway: la
   usarás en el paso 4.

2. Comprueba que ahora la resolución de nombres sí funciona, para tener una base de
   comparación.
   ```bash
   ping -c 2 google.com
   ```
   - `ping` → envía paquetes ICMP de eco a un destino y mide si responde y cuánto tarda.
   - `-c 2` → count: envía solo 2 paquetes y termina (sin esto, ping seguiría indefinidamente).
   - `google.com` → el destino; al ser un nombre, obliga al sistema a resolverlo por DNS.

   Debe responder con líneas `64 bytes from ...`. Si ya falla aquí, arregla la red
   antes de seguir.

3. Rompe el DNS a propósito: pon un servidor DNS que no existe. En una VM con
   `resolv.conf` gestionado directamente, haz una copia y reemplázalo.
   ```bash
   sudo cp /etc/resolv.conf /etc/resolv.conf.bak
   echo "nameserver 192.0.2.53" | sudo tee /etc/resolv.conf
   ```
   - `sudo` → ejecuta el comando como root; hace falta para escribir en `/etc`.
   - `cp` → copia un archivo.
   - `/etc/resolv.conf` → primer argumento de `cp`: el archivo de origen (la config DNS actual).
   - `/etc/resolv.conf.bak` → segundo argumento de `cp`: el destino, una copia de respaldo.
   - `echo "nameserver 192.0.2.53"` → imprime ese texto (una línea de config DNS válida).
   - `|` → tubería (ver ejercicio 1).
   - `tee` → escribe su entrada en un archivo y además la muestra; con `sudo` puede escribir en archivos de root.
   - `/etc/resolv.conf` → argumento de `tee`: el archivo que se sobrescribe con la línea nueva.

   `192.0.2.53` es una dirección del bloque de documentación (RFC 5737): no responde
   nunca, así que simula un DNS caído. En VMs con `systemd-resolved` o NetworkManager
   que sobrescriben el archivo, usa en su lugar su configuración (por ejemplo
   `resolvectl dns eth0 192.0.2.53`) o detén el servicio DNS; la idea es la misma.

4. Primera prueba: ¿llego a la puerta de enlace? Usa la IP del paso 1.
   ```bash
   ping -c 2 192.168.1.1
   ```
   - `ping -c 2` → ver ejercicio 1 (paso 2).
   - `192.168.1.1` → el destino: tu gateway, dado como IP, así que esta prueba no usa DNS.

   Si responde, la capa física y la red local están bien: el cable, la tarjeta y el
   router de tu red no son el problema. Una variable descartada.

5. Segunda prueba: ¿llego a Internet por IP, sin usar nombres?
   ```bash
   ping -c 2 8.8.8.8
   ```
   - `ping -c 2` → ver ejercicio 1 (paso 2).
   - `8.8.8.8` → el destino: el DNS público de Google dado como IP; prueba Internet sin resolver nombres.

   Si responde, el enrutamiento y el proveedor funcionan: el paquete sale de tu red y
   vuelve. Otra variable descartada. Fíjate en que esta prueba no toca el DNS, porque
   `8.8.8.8` ya es una IP.

6. Tercera prueba: ¿funciona la resolución de nombres?
   ```bash
   ping -c 2 google.com
   ```
   - `ping -c 2 google.com` → ver ejercicio 1 (paso 2); aquí se repite a propósito para ver que ahora falla.

   Ahora debe fallar con `Temporary failure in name resolution` o
   `Name or service not known`. Como las dos pruebas anteriores salieron bien, la única
   causa que queda es el DNS. Eso es divide y vencerás: la prueba del paso 5 ya demostró
   que todo lo de abajo funciona.

7. Documenta y revierte. Deja por escrito qué descartó cada prueba y restaura el DNS.
   ```bash
   sudo mv /etc/resolv.conf.bak /etc/resolv.conf
   ping -c 2 google.com
   ```
   - `sudo` → ver ejercicio 1 (paso 3).
   - `mv` → mueve o renombra un archivo; aquí devuelve el respaldo a su sitio.
   - `/etc/resolv.conf.bak` → primer argumento de `mv`: el origen (el respaldo).
   - `/etc/resolv.conf` → segundo argumento de `mv`: el destino (se restaura la config buena).
   - `ping -c 2 google.com` → ver ejercicio 1 (paso 2).

   El último `ping` debe volver a responder.

### Resultado esperado

Una secuencia de tres pruebas donde el ping al gateway y a `8.8.8.8` responden y el ping
a `google.com` falla, más una nota tuya que diga: "cable y router OK (ping al gateway),
Internet y proveedor OK (ping a 8.8.8.8), falla la resolución de nombres (ping al
nombre): la causa es el DNS".

### Comprueba que lo lograste

- ¿Por qué el ping a `8.8.8.8` no prueba nada sobre el DNS? Porque `8.8.8.8` ya es una
  dirección IP; el sistema no necesita traducir ningún nombre para enviarle el paquete.
- Si el ping al gateway también hubiera fallado, ¿qué habrías sospechado primero? Un
  problema más abajo: cable, interfaz de red o configuración de IP local, no el DNS.
- ¿Cambiaste más de una cosa a la vez en algún momento? Si la respuesta es no, aplicaste
  bien la regla de una variable por prueba.

### Limpieza

Si al revertir el paso 7 el nombre sigue sin resolver, reinicia el servicio de red de tu
VM (`sudo systemctl restart systemd-resolved` o `sudo systemctl restart
NetworkManager`, según la que uses) o reinicia la VM. El archivo `/etc/resolv.conf.bak`
puede borrarse una vez confirmado que todo funciona.

## Ejercicio 2: Extrae la macro de un documento con olevba

Nodo: [Understand Basics of Popular Suites](README.md#understand-basics-of-popular-suites)
y la idea de que un documento con macros es un programa.

Objetivo: crear en LibreOffice un documento con una macro inofensiva que muestre un
mensaje, guardarlo en formato de Microsoft con macros y confirmar con `olevba` que la
herramienta ve la macro y marca su ejecución automática, comparándolo con un documento
sin macros. Al terminar entenderás qué mira un analista antes de abrir un adjunto.

Necesitas: una VM Linux propia (aislada, sin datos sensibles); LibreOffice instalado;
Python 3 con `pipx` para instalar oletools. Tiempo estimado: 30 minutos.

Instalación de oletools (la forma recomendada, aislada del sistema):
```bash
pipx install oletools
```
- `pipx` → instala aplicaciones de Python en entornos aislados, sin tocar el Python del sistema.
- `install` → subcomando de `pipx`: instala el paquete indicado.
- `oletools` → el paquete a instalar; incluye `olevba` y otras utilidades de análisis de documentos.

En Debian/Ubuntu, si no tienes `pipx`: `sudo apt install pipx`. En Arch:
`sudo pacman -S python-pipx`. Alternativa directa: `python3 -m pip install --user
oletools`. Comprueba que quedó con `olevba --version` (la opción `--version` solo imprime
la versión y sale).

### Pasos

1. Abre LibreOffice Writer y entra al editor de macros Basic por el menú:
   `Herramientas` → `Macros` → `Editar macros`. Se abre el IDE de LibreOffice Basic.

2. En el panel de la izquierda, dentro de tu documento (`Untitled 1` →
   `Standard` → `Module1`), escribe una macro inofensiva con nombre de autoarranque.
   Pega exactamente este código:
   ```basic
   Sub AutoOpen()
       MsgBox "Documento de laboratorio: macro de prueba inofensiva"
   End Sub
   ```
   - `Sub AutoOpen()` → declara una subrutina (macro) llamada `AutoOpen`; ese nombre es el que Word ejecuta solo al abrir el documento.
   - `MsgBox "..."` → instrucción que muestra un cuadro de mensaje con ese texto; no toca el sistema.
   - `End Sub` → cierra la definición de la subrutina.

3. Cierra el IDE y vuelve al documento. Escribe cualquier texto en la página (por
   ejemplo "factura de prueba") para que no quede vacío.

4. Guarda el documento en formato de Word con macros. Menú `Archivo` → `Guardar como`;
   en "Tipo" elige `Word 2007-365 con macros (.docm)` y ponle de nombre
   `prueba.docm`. Cuando pregunte, mantén el formato de Word (botón "¡Usar Word
   2007-365!").

   > [!NOTE]
   > Si tu versión de LibreOffice no escribe la macro Basic dentro del `.docm` al
   > guardar (es una limitación conocida: el Basic creado en LibreOffice no siempre se
   > exporta al formato de Microsoft), usa un `.docm` de ejemplo público con macro ya
   > incrustada del repositorio de oletools. Descárgalo con
   > `curl -L -o prueba.docm https://github.com/decalage2/oletools/raw/master/tests/test-data/oleform/oleform-PR314.docm`,
   > donde `curl` descarga una URL, `-L` sigue las redirecciones del servidor y
   > `-o prueba.docm` guarda el resultado en ese archivo. El objetivo del ejercicio es
   > leer la salida de `olevba`, y ese archivo sirve igual.

5. Crea también un documento equivalente sin macros para comparar. En Writer, archivo
   nuevo, escribe texto y guárdalo como `limpio.docx` (tipo `Word 2007-365 (.docx)`).
   Las extensiones en `x` no admiten macros.

6. Pasa el documento con macros por `olevba`.
   ```bash
   olevba prueba.docm
   ```
   - `olevba` → extrae y analiza las macros VBA de un documento de oficina sin ejecutarlas.
   - `prueba.docm` → el archivo a analizar.

   Busca en la salida tres cosas: el bloque `VBA MACRO` con el código, la columna
   `Type` con una fila `AutoExec` y, en Keyword, `AutoOpen`. Verás algo parecido a:
   ```
   +----------+--------------------+-----------------------------------------+
   |Type      |Keyword             |Description                              |
   +----------+--------------------+-----------------------------------------+
   |AutoExec  |AutoOpen            |Runs when the Word document is opened    |
   +----------+--------------------+-----------------------------------------+
   ```
   `AutoExec` significa que la macro corre sola al abrir el documento. Si el código
   tuviera `Shell` o `powershell`, aparecerían como `Suspicious`.

7. Pasa ahora el documento sin macros por la misma herramienta.
   ```bash
   olevba limpio.docx
   ```
   - `olevba` → ver ejercicio 2 (paso 6).
   - `limpio.docx` → el archivo a analizar; al no tener macros, la salida lo dirá.

   La salida debe terminar en `No VBA or XLM macros found.`: no hay nada que ejecutar.

### Resultado esperado

Dos salidas de `olevba`: una que muestra la macro, su tipo `AutoExec` y el keyword
`AutoOpen`, y otra que dice que no hay macros. Entiendes que el primer archivo es, en la
práctica, un programa que se ejecuta al abrirlo, y el segundo son solo datos.

### Comprueba que lo lograste

- ¿Qué fila de la tabla de `olevba` indica que la macro se ejecuta sin que el usuario la
  llame? La fila `AutoExec` con el keyword `AutoOpen` (o `Document_Open`).
- ¿Por qué `limpio.docx` no puede tener macro aunque le pegues código? Porque la
  extensión en `x` (`.docx`) no admite proyecto VBA; haría falta un `.docm`.
- ¿Ejecutaste la macro en algún momento para analizarla? No hace falta: `olevba` extrae
  el código sin abrir el documento en Word, que es justo lo que lo hace seguro.

### Limpieza

Borra los archivos de prueba cuando termines: `rm prueba.docm limpio.docx` (`rm` borra
los archivos que recibe como argumentos). Como todo se hizo con una macro que solo
muestra un mensaje, no queda nada instalado ni en ejecución.

## Ejercicio 3: Rastrea la ruta a tres destinos con traceroute

Nodo: [Basics of Computer Networking](README.md#basics-of-computer-networking).

Objetivo: ejecutar `traceroute` hacia tres destinos distintos e identificar en cada
salida tu puerta de enlace, el primer router de tu proveedor y cuántos saltos hay hasta
el destino. Al terminar sabrás leer el camino que recorre un paquete salto a salto.

Necesitas: tu propio equipo o una VM con salida a Internet. En Linux, `traceroute`
(instalar con `sudo apt install traceroute` en Debian/Ubuntu o `sudo pacman -S
traceroute` en Arch); en Windows ya viene `tracert`. Tiempo estimado: 15 minutos.

### Pasos

1. Averigua tu puerta de enlace para reconocerla luego en la primera línea.
   ```bash
   ip route | grep default
   ```
   - `ip route | grep default` → ver ejercicio 1 (paso 1).

   Anota la IP que aparece tras `via` (por ejemplo `192.168.1.1`).

2. Rastrea el primer destino, un DNS público. La opción `-n` muestra las IP sin
   resolver nombres, que es más rápido y más claro para leer el camino.
   ```bash
   traceroute -n 1.1.1.1
   ```
   - `traceroute` → muestra cada router (salto) por el que pasa un paquete hasta el destino.
   - `-n` → numeric: muestra las IP tal cual, sin convertirlas a nombres por DNS.
   - `1.1.1.1` → el destino a rastrear (el DNS público de Cloudflare).

   En Windows sería `tracert -d 1.1.1.1` (`-d` es el equivalente de `-n`: no resuelve
   nombres). Cada línea es un salto: número, IP del router y tres tiempos de ida y vuelta
   en milisegundos.

3. Rastrea un segundo destino distinto.
   ```bash
   traceroute -n 8.8.8.8
   ```
   - `traceroute -n` → ver ejercicio 3 (paso 2); solo cambia el destino.
   - `8.8.8.8` → el destino (DNS público de Google).

4. Rastrea un tercero, por ejemplo otro DNS público conocido por su IP, para no depender
   de resolver un nombre.
   ```bash
   traceroute -n 9.9.9.9
   ```
   - `traceroute -n` → ver ejercicio 3 (paso 2); solo cambia el destino.
   - `9.9.9.9` → el destino (DNS público de Quad9).

5. En las tres salidas, identifica las tres cosas. El salto 1 es siempre tu puerta de
   enlace (la IP del paso 1). El salto 2 o 3, cuando deja de ser una dirección privada
   (`192.168.x.x`, `10.x.x.x`) y pasa a una pública, es el primer router de tu
   proveedor. El número de la última línea que llega al destino es la cantidad de
   saltos. Ejemplo de lectura:
   ```
    1  192.168.1.1    1.2 ms   1.0 ms   1.1 ms    # tu gateway
    2  10.20.0.1      8.4 ms   8.1 ms   8.9 ms    # primer router del proveedor
    3  200.10.5.33   12.7 ms  12.3 ms  12.9 ms
    4  1.1.1.1       14.0 ms  13.8 ms  14.2 ms    # destino: 4 saltos
   ```

6. Compara los tres caminos. Lo normal es que el salto 1 (y a veces el 2) sea el mismo
   en los tres, porque todos salen por tu misma red y tu mismo proveedor, y que diverjan
   más adelante. Un `* * *` en una línea significa que ese router no respondió al
   rastreo, lo cual es normal y no implica fallo.

### Resultado esperado

Tres salidas de `traceroute`, y una nota tuya por destino que diga: cuál es el gateway
(coincide con el paso 1), cuál es el primer router del proveedor (el primero con IP
pública) y cuántos saltos hubo hasta llegar.

### Comprueba que lo lograste

- ¿La primera línea es la misma en los tres rastreos? Debe serlo: es tu puerta de
  enlace, por la que sale todo tu tráfico.
- ¿Cómo distingues el router de tu proveedor de los de tu propia red? Por la dirección:
  los de tu red son privadas (RFC 1918); el primero con IP pública ya es del proveedor
  o de más allá.
- Si una línea muestra `* * *`, ¿significa que la red está rota? No: ese salto existe
  pero el router no contesta al rastreo; el paquete sigue y suele llegar igual al
  destino.

### Limpieza

No aplica: `traceroute` solo envía sondas y no cambia nada en tu equipo ni en la red.
