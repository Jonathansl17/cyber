# Ejercicios: Frameworks de amenazas

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

## Ejercicio 1: Por qué bloquear un hash no basta

Nodo: [Pirámide del dolor](README.md#pirámide-del-dolor) e [IOCs vs IOAs](README.md#iocs-vs-ioas).

Objetivo: tomar un hash de malware publicado hoy en MalwareBazaar, agrupar las muestras recientes de su familia por imphash (la huella de las funciones que importa el ejecutable, un escalón más arriba en la pirámide que el hash del archivo) y medir con esos números cuántas muestras detiene bloquear un hash frente a detectar un artefacto compartido.

Necesitas: Linux con `curl` y `python3`; una cuenta gratuita en [abuse.ch](https://auth.abuse.ch/) para obtener tu Auth-Key; un navegador para VirusTotal (no hace falta cuenta para buscar por hash). Nunca descargas ni ejecutas ninguna muestra: solo consultas metadatos. Tiempo estimado: 30 minutos.

```bash
sudo apt install curl python3     # Debian/Ubuntu
sudo pacman -S curl python        # Arch
```

- `sudo` → ejecuta el comando que sigue como root, necesario para instalar paquetes del sistema.
- `apt install` → gestor de paquetes de Debian/Ubuntu; la acción `install` descarga e instala los paquetes indicados con sus dependencias.
- `pacman -S` → gestor de paquetes de Arch; `-S` (`--sync`) instala los paquetes desde los repositorios.
- `curl` → paquete del cliente HTTP de línea de comandos que usarás para hablar con las API.
- `python3` / `python` → paquete del intérprete de Python 3 (en Arch se llama `python`).
- `# Debian/Ubuntu`, `# Arch` → comentarios de la shell: todo lo que va tras `#` se ignora; ejecuta solo la línea de tu distribución.

### Pasos

1. Comprueba primero en tu máquina, con un archivo inofensivo, lo que dice la base de la pirámide: un byte distinto da un hash completamente distinto.
   ```bash
   mkdir -p ~/lab-piramide && cd ~/lab-piramide
   printf 'factura 0423\n' > original.txt
   cp original.txt modificado.txt
   printf ' ' >> modificado.txt
   sha256sum original.txt modificado.txt
   ```
   - `mkdir -p ~/lab-piramide` → crea la carpeta de trabajo en tu directorio personal (`~`); `-p` crea también los directorios padre que falten y no da error si ya existe.
   - `&&` → ejecuta el comando de la derecha solo si el de la izquierda terminó bien.
   - `cd ~/lab-piramide` → entra en esa carpeta.
   - `printf 'factura 0423\n'` → escribe el texto tal cual; `\n` es un salto de línea.
   - `> original.txt` → redirige la salida a ese archivo, creándolo o sobrescribiéndolo.
   - `cp original.txt modificado.txt` → copia el primer archivo (origen) en el segundo (destino).
   - `printf ' '` → escribe un solo espacio, es decir, un byte.
   - `>> modificado.txt` → añade la salida al final del archivo sin borrar lo que ya tenía.
   - `sha256sum original.txt modificado.txt` → calcula e imprime el hash SHA-256 de cada archivo, uno por línea, seguido de su nombre.

   Verás dos hashes de 64 caracteres hexadecimales sin ningún parecido entre sí. Eso es lo que hace un malware polimórfico en cada infección.
2. Entra a [auth.abuse.ch](https://auth.abuse.ch/), inicia sesión y copia tu Auth-Key. Guárdala solo en la variable de entorno de esta terminal (no en un archivo ni en el historial compartido):
   ```bash
   read -rs ABUSE_KEY && export ABUSE_KEY
   ```
   - `read` → orden interna de bash que lee una línea del teclado y la guarda en una variable.
   - `-r` → no trata las barras invertidas `\` como caracteres de escape, así la clave se guarda literal.
   - `-s` → modo silencioso: no muestra en pantalla lo que escribes o pegas. Pega la clave y pulsa Enter.
   - `ABUSE_KEY` → nombre de la variable donde queda la clave.
   - `&&` → ejecuta lo siguiente solo si `read` terminó bien.
   - `export ABUSE_KEY` → marca la variable para que la hereden los programas que lances desde esta terminal; desaparece al cerrarla.
3. Pide a MalwareBazaar las 100 muestras más recientes y quédate con hash, familia (`signature`) y fecha:
   ```bash
   curl -s -H "Auth-Key: $ABUSE_KEY" \
     --data "query=get_recent&selector=100" \
     https://mb-api.abuse.ch/api/v1/ > recientes.json
   python3 -c '
   import json
   d = json.load(open("recientes.json"))
   print("estado:", d["query_status"])
   for s in d["data"][:15]:
       print(s["first_seen"], s["signature"], s["sha256_hash"])
   '
   ```
   - `curl` → cliente HTTP que hace la petición a la API.
   - `-s` → modo silencioso: no muestra la barra de progreso ni mensajes de error.
   - `-H "Auth-Key: $ABUSE_KEY"` → añade esa cabecera HTTP a la petición; la shell sustituye `$ABUSE_KEY` por tu clave, y las comillas dobles lo permiten.
   - `--data "query=get_recent&selector=100"` → envía esos datos en el cuerpo de una petición POST (activar `--data` convierte la petición en POST); `query=get_recent` pide las muestras más recientes y `selector=100` limita a las últimas 100.
   - `\` al final de línea → continúa el mismo comando en la línea siguiente.
   - `https://mb-api.abuse.ch/api/v1/` → URL de la API de MalwareBazaar.
   - `> recientes.json` → guarda la respuesta JSON en ese archivo.
   - `python3 -c '...'` → ejecuta el programa de Python escrito entre comillas simples en vez de leerlo de un archivo.
   - `import json` → carga el módulo para leer JSON.
   - `d = json.load(open("recientes.json"))` → abre el archivo y lo convierte en un diccionario de Python.
   - `d["query_status"]` → campo de la respuesta que indica si la consulta fue bien (`ok`).
   - `for s in d["data"][:15]:` → recorre las primeras 15 muestras de la lista `data`.
   - `print(s["first_seen"], s["signature"], s["sha256_hash"])` → imprime fecha de primera aparición, familia y hash SHA-256 de cada muestra.

   `estado: ok` y una lista de líneas con fecha de hoy. Elige una muestra cuya columna `signature` no sea `None` (por ejemplo `AgentTesla`, `Formbook`, `RemcosRAT`) y anota su hash y su familia.
4. Pide todas las muestras recientes de esa familia (hasta 1000) y agrúpalas por imphash. MalwareBazaar guarda cada archivo una sola vez, así que cada fila es ya un SHA-256 distinto: contar hashes distintos solo repetiría cuántas filas devolvió la API. Lo que sí informa es cuántos imphash distintos hay detrás de esos hashes y cuántas muestras comparte cada uno. Sustituye `AgentTesla` por la familia que elegiste:
   ```bash
   FAMILIA=AgentTesla
   curl -s -H "Auth-Key: $ABUSE_KEY" \
     --data "query=get_siginfo&signature=$FAMILIA&limit=1000" \
     https://mb-api.abuse.ch/api/v1/ > familia.json
   python3 -c '
   import json
   from collections import Counter
   d = json.load(open("familia.json"))
   muestras = d["data"]
   tipos = Counter(s["file_type"] for s in muestras)
   imp = Counter(s["imphash"] for s in muestras if s.get("imphash"))
   print("muestras (= SHA-256 distintos):", len(muestras))
   print("tipos de archivo:", tipos.most_common(5))
   print("muestras PE con imphash:", sum(imp.values()))
   print("imphash distintos:", len(imp))
   for h, n in imp.most_common(5):
       print(h, n)
   '
   ```
   - `FAMILIA=AgentTesla` → crea una variable de shell con el nombre de la familia; sin espacios alrededor de `=`.
   - `curl -s -H "Auth-Key: $ABUSE_KEY"` → (ver paso 3).
   - `--data "query=get_siginfo&signature=$FAMILIA&limit=1000"` → petición POST; `get_siginfo` pide muestras por firma (familia), `signature=$FAMILIA` indica cuál y `limit=1000` pide hasta 1000 resultados, el máximo de la API (por defecto devuelve 100).
   - `> familia.json` → guarda la respuesta en ese archivo.
   - `python3 -c '...'`, `import json`, `json.load(open(...))` → (ver paso 3).
   - `from collections import Counter` → importa `Counter`, un diccionario que cuenta apariciones.
   - `muestras = d["data"]` → lista de muestras devuelta; cada una es un archivo único en MalwareBazaar.
   - `Counter(s["file_type"] for s in muestras)` → cuenta las muestras por tipo de archivo (`exe`, `dll`, `zip`, `js`...); `file_type` es un campo de la respuesta.
   - `s.get("imphash")` → lee el campo `imphash` sin dar error si falta; la API solo lo rellena en ejecutables PE de Windows y en el resto viene vacío o `null`, por eso el `if` descarta esas muestras.
   - `Counter(s["imphash"] for s in muestras if ...)` → cuenta cuántas muestras comparten cada imphash.
   - `len(muestras)` → número de muestras, que es también el número de SHA-256 distintos.
   - `tipos.most_common(5)` → los 5 tipos de archivo más frecuentes con su recuento.
   - `sum(imp.values())` → total de muestras PE que tienen imphash.
   - `len(imp)` → número de imphash distintos.
   - `imp.most_common(5)` → los 5 imphash compartidos por más muestras; `print(h, n)` imprime cada uno con su recuento.

   Salida de ejemplo (tus números serán otros):
   ```
   muestras (= SHA-256 distintos): 1000
   tipos de archivo: [('exe', 702), ('zip', 131), ('rar', 64), ('js', 41), ('gz', 22)]
   muestras PE con imphash: 702
   imphash distintos: 9
   f34d5f2d4577ed6d9ceec516c1f5a744 655
   ...
   ```
   Lo que importa es la proporción: cientos de SHA-256 distintos frente a un puñado de imphash. Anota el imphash más repetido y cuántas muestras lo comparten (aquí, 655): es tu dato. Que muchas muestras compartan imphash suele indicar el mismo empaquetador o la misma base de código recompilada; un imphash muy genérico (por ejemplo, el de los ejecutables .NET, que casi solo importan `_CorExeMain`) también lo comparten programas legítimos, y eso mismo es una lección sobre falsos positivos. El imphash del ejemplo es justo ese genérico de .NET (AgentTesla está escrito en .NET): agrupa la familia, pero como regla de bloqueo sola daría falsos positivos; con una familia compilada en código nativo, como `RemcosRAT`, los grupos suelen ser más específicos. Si la API responde `illegal_signature` o `signature_not_found`, copia el nombre exacto tal como aparece en el paso 3; si la familia elegida apenas tiene ejecutables PE (`muestras PE con imphash` cercano a 0), elige otra en el paso 3.
5. Abre en el navegador `https://www.virustotal.com/gui/file/<hash>` con el hash elegido. En la pestaña Detection anota la proporción de motores que lo detectan y el "Popular threat label"; en la pestaña Details anota "First Submission Date" y la lista "Names". No subas ningún archivo: solo buscas por hash.
6. Saca de `familia.json` dos hashes que compartan el imphash más repetido y búscalos en VirusTotal igual que en el paso 5:
   ```bash
   IMPHASH=f34d5f2d4577ed6d9ceec516c1f5a744
   python3 -c '
   import json, sys
   d = json.load(open("familia.json"))
   for s in [s for s in d["data"] if s.get("imphash") == sys.argv[1]][:2]:
       print(s["sha256_hash"], s["file_name"])
   ' "$IMPHASH"
   ```
   - `IMPHASH=...` → variable con el imphash que anotaste en el paso 4; sustituye el valor de ejemplo por el tuyo.
   - `python3 -c '...'`, `import json`, `json.load(open(...))` → (ver paso 3).
   - `import ..., sys` → módulo que da acceso a los argumentos de la línea de comandos en `sys.argv`.
   - `"$IMPHASH"` tras el programa → se pasa como primer argumento; dentro de Python es `sys.argv[1]`.
   - `[s for s in d["data"] if s.get("imphash") == sys.argv[1]][:2]` → filtra las muestras con ese imphash y toma las dos primeras.
   - `print(s["sha256_hash"], s["file_name"])` → imprime el hash SHA-256 y el nombre de archivo con que se subió cada una.

   Compara en VirusTotal: misma etiqueta de familia, SHA-256 y nombres de archivo distintos, y en la pestaña Details el mismo valor en "Imphash".
7. Escribe en `conclusion.txt` una línea que use tu número, por ejemplo:
   ```bash
   echo "Bloquear un hash detiene 1 de 1000 muestras de $FAMILIA; una detección por el imphash $IMPHASH cubre 655 (validándola antes contra software legítimo), y aun así el atacante la rompe cambiando de empaquetador, así que hay que seguir subiendo en la pirámide (herramienta, TTP)." > conclusion.txt
   ```
   - `echo "..."` → imprime el texto; las comillas dobles permiten que la shell sustituya `$FAMILIA` y `$IMPHASH` por sus valores. Cambia 1000 y 655 por tus números del paso 4.
   - `> conclusion.txt` → escribe esa línea en el archivo, sobrescribiéndolo si ya existía.

### Resultado esperado

Un `familia.json` con las muestras de una familia, el recuento de muestras frente a imphash distintos con el imphash más compartido, las anotaciones de VirusTotal de tres muestras de la misma familia (dos con el mismo imphash) y `conclusion.txt` con la justificación en una línea.

### Comprueba que lo lograste

- ¿Cuántas muestras (SHA-256 distintos) cubre tu imphash más compartido y cuántos imphash distintos hay en total? El primero debe ser muy superior a 1 y el segundo mucho menor que el número de muestras: muchos hashes, pocas huellas de compilación.
- ¿Por qué no sirve contar "hashes distintos" en MalwareBazaar? Porque la base guarda cada archivo una sola vez: ese recuento siempre es igual al número de filas devueltas.
- ¿Qué escalón de la pirámide ocupa el hash y cuánto le cuesta al atacante cambiarlo? Hash values, trivial: segundos, como mostró el paso 1.
- ¿Para qué sí sirve el hash aunque no baste para bloquear? Para buscar hacia atrás si alguna máquina ya lo tuvo y para bloqueo inmediato de esa muestra concreta.
- Nombra un indicador más arriba de la pirámide que puedas sacar del informe de VirusTotal (un dominio C2, un mutex, un comportamiento en la pestaña Behavior).

### Limpieza

```bash
unset ABUSE_KEY
rm -rf ~/lab-piramide
```

- `unset ABUSE_KEY` → borra la variable de la sesión para que la clave no quede en memoria de la terminal.
- `rm` → borra archivos.
- `-r` → recursivo: borra la carpeta y todo su contenido.
- `-f` → forzado: no pide confirmación ni da error si algo no existe.
- `~/lab-piramide` → carpeta del ejercicio que se elimina.

## Ejercicio 2: Huecos de cobertura de las reglas Sigma de Linux frente a TeamTNT

Nodo: [ATT&CK](README.md#attck) y [Los tres frameworks responden preguntas distintas](README.md#los-tres-frameworks-responden-preguntas-distintas).

Objetivo: superponer en el ATT&CK Navigator una capa con las técnicas de TeamTNT (G0139), un grupo que ataca sobre todo servidores Linux y contenedores, y otra con las técnicas que cubren las reglas de Linux de [SigmaHQ](https://github.com/SigmaHQ/sigma), obtener la lista exacta de técnicas de TeamTNT sin cobertura y explicar tres de esos huecos.

Necesitas: navegador; Linux con `curl`, `tar` y `python3`. Sigma es el formato abierto de reglas de detección para logs; cada regla de SigmaHQ lleva etiquetas `attack.tNNNN` con su técnica de ATT&CK, y las de `rules/linux/` se escriben para auditd, para el journal y para la telemetría de procesos del EDR. Tiempo estimado: 45 minutos.

### Pasos

1. Descarga la capa del grupo y las reglas Sigma de Linux:
   ```bash
   mkdir -p ~/lab-navigator && cd ~/lab-navigator
   curl -sL -o teamtnt.json https://attack.mitre.org/groups/G0139/G0139-enterprise-layer.json
   curl -sL https://github.com/SigmaHQ/sigma/archive/refs/heads/master.tar.gz | tar -xz --wildcards 'sigma-master/rules/linux/*'
   python3 -c 'import json; print(json.load(open("teamtnt.json"))["versions"])'
   find sigma-master/rules/linux -name '*.yml' | wc -l
   ```
   - `mkdir -p`, `&&`, `cd` → (ver ejercicio 1).
   - `curl -s` → (ver ejercicio 1).
   - `-L` → sigue las redirecciones HTTP; hace falta porque GitHub redirige la descarga del archivo comprimido a su servidor de descargas.
   - `-o teamtnt.json` → guarda la respuesta en ese archivo en vez de mostrarla en pantalla.
   - `https://attack.mitre.org/groups/G0139/G0139-enterprise-layer.json` → capa del Navigator que publica MITRE con las técnicas de TeamTNT (G0139) en la matriz Enterprise.
   - `https://github.com/SigmaHQ/sigma/archive/refs/heads/master.tar.gz` → el repositorio de SigmaHQ completo, en su versión actual, como archivo `.tar.gz`; sin `-o`, `curl` lo escribe en su salida.
   - `|` → tubería: pasa la salida del comando de la izquierda como entrada del de la derecha.
   - `tar -xz` → `-x` extrae archivos y `-z` descomprime gzip; sin `-f`, lee el archivo de la entrada estándar, es decir, de la tubería.
   - `--wildcards 'sigma-master/rules/linux/*'` → extrae solo las rutas que encajan con ese patrón (`*` es cualquier texto): las reglas de Linux, no el repositorio entero.
   - `python3 -c '...'` → (ver ejercicio 1).
   - `json.load(open("teamtnt.json"))["versions"]` → lee el JSON y saca su campo `versions`, que dice la versión de ATT&CK, del Navigator y de la capa.
   - `find sigma-master/rules/linux -name '*.yml'` → lista los archivos de esa carpeta y sus subcarpetas cuyo nombre acaba en `.yml`: una regla por archivo.
   - `wc -l` → cuenta líneas, es decir, el número de reglas.
   La primera orden debe mostrar `'attack': '19'` y la segunda unas 200 reglas. En `teamtnt.json` las técnicas usadas por el grupo tienen `score: 1`.
2. Cuenta las técnicas etiquetadas en las reglas y mira cuántas reglas apuntan a cron (T1053.003):
   ```bash
   grep -rhoE 'attack\.t[0-9]{4}(\.[0-9]{3})?' sigma-master/rules/linux | sort -u | wc -l
   grep -rhoE 'attack\.t1053(\.[0-9]{3})?' sigma-master/rules/linux | sort | uniq -c
   ```
   - `grep` → busca un patrón (expresión regular) en archivos.
   - `-r` → recursivo: busca en todos los archivos de la carpeta y sus subcarpetas.
   - `-h` → no antepone el nombre del archivo a cada coincidencia.
   - `-o` → imprime solo el trozo que coincide, uno por línea, en vez de la línea entera.
   - `-E` → expresión regular extendida, que permite `{4}`, `?` y paréntesis sin escaparlos.
   - `'attack\.t[0-9]{4}(\.[0-9]{3})?'` → patrón: `attack.t` (`\.` es un punto literal) seguido de 4 dígitos y, opcionalmente (`?`), un punto y 3 dígitos más; es decir, una etiqueta como `attack.t1053.003`.
   - `'attack\.t1053(\.[0-9]{3})?'` → igual, pero solo T1053 y sus subtécnicas.
   - `sort -u` → ordena las líneas; `-u` deja una sola copia de cada línea repetida.
   - `sort` → ordena las líneas, necesario para que `uniq` vea juntas las repetidas.
   - `uniq -c` → junta líneas repetidas consecutivas; `-c` antepone cuántas veces aparece cada una (aquí, cuántas reglas apuntan a cada técnica).
   - `wc -l` → (ver paso 1); aquí, el número de técnicas distintas.
   Unas 100 técnicas, y varias reglas para `attack.t1053.003` (cron). Fíjate en que no aparece `attack.t1053.006` (timers de systemd): es un hueco que retomarás al final del ejercicio 4.
3. Convierte las etiquetas en una capa del Navigator. Guarda esto como `capa_sigma.py`:
   ```python
   import json
   import re
   import sys
   from collections import Counter
   from pathlib import Path

   ETIQUETA = re.compile(r"^\s*-\s*attack\.(t\d{4}(?:\.\d{3})?)\s*$", re.MULTILINE)

   reglas = Counter()
   for archivo in Path(sys.argv[1]).rglob("*.yml"):
       texto = archivo.read_text(encoding="utf-8")
       for tid in set(ETIQUETA.findall(texto)):
           reglas[tid.upper()] += 1

   capa = {
       "name": "Sigma Linux",
       "domain": "enterprise-attack",
       "versions": {"attack": "19", "navigator": "5.3.2", "layer": "4.5"},
       "techniques": [{"techniqueID": t, "score": n} for t, n in sorted(reglas.items())],
   }
   with open(sys.argv[2], "w", encoding="utf-8") as f:
       json.dump(capa, f, indent=2)
   print(f"Técnicas con al menos una regla: {len(reglas)}")
   ```
   - `import json`, `re`, `sys` → módulos para escribir JSON, usar expresiones regulares y leer los argumentos de la línea de comandos.
   - `from collections import Counter` → (ver ejercicio 1, paso 4).
   - `from pathlib import Path` → clase para manejar rutas y recorrer carpetas.
   - `re.compile(r"...", re.MULTILINE)` → prepara la expresión regular; `r"..."` es una cadena en bruto, donde `\` no se interpreta; `re.MULTILINE` hace que `^` y `$` marquen el inicio y el fin de cada línea, no del archivo entero.
   - `^\s*-\s*attack\.` → una línea de la lista `tags:` del YAML: espacios opcionales (`\s*`), un guion y `attack.`.
   - `(t\d{4}(?:\.\d{3})?)` → captura el ID de la técnica (`\d` es un dígito); `(?:...)` agrupa sin capturar. Así no se cuentan etiquetas de táctica como `attack.persistence`.
   - `Path(sys.argv[1]).rglob("*.yml")` → recorre todos los `.yml` de la carpeta que recibe como primer argumento, incluidas las subcarpetas.
   - `archivo.read_text(encoding="utf-8")` → lee el archivo entero como texto.
   - `set(ETIQUETA.findall(texto))` → todas las técnicas de esa regla, sin repetir.
   - `reglas[tid.upper()] += 1` → suma una regla a esa técnica; `.upper()` pasa `t1053.003` a `T1053.003`, el formato del Navigator.
   - `capa = {...}` → la capa mínima que entiende el Navigator: nombre, dominio, versiones y la lista de técnicas.
   - `"versions": {"attack": "19", ...}` → debe coincidir con la versión de ATT&CK de `teamtnt.json` (paso 1); cámbiala si allí sale otra.
   - `{"techniqueID": t, "score": n}` → cada técnica con su puntuación: el número de reglas que la etiquetan.
   - `sorted(reglas.items())` → pares (técnica, número de reglas) ordenados por ID.
   - `open(sys.argv[2], "w", encoding="utf-8")` → abre para escribir el archivo que recibe como segundo argumento; `with` lo cierra al terminar.
   - `json.dump(capa, f, indent=2)` → escribe la capa en JSON con sangría de 2 espacios.
   - `f"..."` → cadena con formato; lo que va entre `{}` se evalúa e inserta.
   ```bash
   python3 capa_sigma.py sigma-master/rules/linux sigma-linux.json
   ```
   - `python3 capa_sigma.py` → ejecuta el script con Python 3.
   - `sigma-master/rules/linux` → primer argumento (`sys.argv[1]`), la carpeta de reglas.
   - `sigma-linux.json` → segundo argumento (`sys.argv[2]`), la capa que se genera.
   Debe imprimir el mismo número de técnicas que el paso 2.
4. Abre [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/). En la pestaña nueva elige "Open Existing Layer" y luego "Upload from local", y sube `teamtnt.json`. Abre otra pestaña (botón `+`) y sube igual `sigma-linux.json`.
5. En la pestaña de Sigma, pulsa el botón "Color Setup" y en "Scoring Gradient" pon "low value" en 0 y "high value" en 1, con colores de blanco a verde. Así toda técnica con al menos una regla queda verde.
6. Abre una tercera pestaña, despliega "Create Layer from other layers", elige el dominio Enterprise y fíjate en la letra que el Navigator asigna a cada capa abierta (aparece en la pestaña: `a` para TeamTNT y `b` para Sigma si las abriste en ese orden). En "Score Expression" escribe:
   ```
   (a > 0) and not (b > 0)
   ```
   - `a`, `b` → la puntuación de cada técnica en la capa con esa letra (TeamTNT y Sigma).
   - `a > 0` → la técnica la usa TeamTNT.
   - `b > 0` → la técnica tiene al menos una regla Sigma de Linux.
   - `and not` → las dos condiciones a la vez, negando la segunda: técnicas de TeamTNT sin regla. El resultado verdadero vale 1 y el falso 0.
   Pulsa "Create Layer". La capa nueva tiene puntuación 1 en las técnicas de TeamTNT sin regla. Pon su gradiente de 0 (blanco) a 1 (rojo) con "Color Setup".
7. Verifica la lista con un script, porque el Navigator distingue "sin puntuar" de "puntuación 0" y conviene confirmar el recuento. Guarda esto como `huecos.py`:
   ```python
   import json
   import sys


   def tecnicas_puntuadas(ruta):
       with open(ruta, encoding="utf-8") as f:
           capa = json.load(f)
       return {t["techniqueID"] for t in capa["techniques"] if t.get("score", 0) > 0}


   grupo = tecnicas_puntuadas(sys.argv[1])
   cubiertas = tecnicas_puntuadas(sys.argv[2])
   print(f"Técnicas del grupo: {len(grupo)}")
   print(f"Cubiertas por Sigma: {len(grupo & cubiertas)}")
   print(f"Huecos: {len(grupo - cubiertas)}")
   for tid in sorted(grupo - cubiertas):
       print(tid)
   ```
   - `import json`, `import sys`, `open(..., encoding="utf-8")`, `with`, `f"..."` → (ver paso 3).
   - `def tecnicas_puntuadas(ruta):` → función que devuelve las técnicas con puntuación de una capa.
   - `capa["techniques"]` → lista de técnicas de la capa del Navigator.
   - `t.get("score", 0) > 0` → se queda con las técnicas cuyo `score` es mayor que 0; si una no tiene `score`, cuenta como 0.
   - `{t["techniqueID"] ...}` → conjunto con los IDs de esas técnicas.
   - `sys.argv[1]`, `sys.argv[2]` → primer y segundo argumento del script: la capa del grupo y la de Sigma.
   - `grupo & cubiertas` → intersección: técnicas del grupo que las reglas cubren.
   - `grupo - cubiertas` → diferencia: técnicas del grupo sin cobertura, los huecos.
   - `sorted(...)` → ordena los IDs antes de imprimirlos.
   ```bash
   python3 huecos.py teamtnt.json sigma-linux.json
   ```
   - `python3 huecos.py` → ejecuta el script con Python 3.
   - `teamtnt.json` → primer argumento (`sys.argv[1]`), la capa del grupo.
   - `sigma-linux.json` → segundo argumento (`sys.argv[2]`), la capa de cobertura.
   Salida de ejemplo (los números cambian con cada versión de ATT&CK y de las reglas):
   ```
   Técnicas del grupo: 56
   Cubiertas por Sigma: 26
   Huecos: 30
   T1021.004
   T1027.002
   ...
   ```
   Cuenta las casillas rojas de la capa del paso 6 y compáralas con la cifra "Huecos". Algunos huecos son técnicas que TeamTNT usó fuera de Linux: abre su página y mira el campo "Platforms"; si no incluye Linux ni Containers, sácala de tu lista de huecos reales.
8. Elige 3 huecos y abre la página de cada uno (`https://attack.mitre.org/techniques/T1098/004/` para T1098.004, por ejemplo). Para cada uno escribe en `huecos.txt`: el ID, qué hace la técnica, por qué las reglas no la ven y qué otra fuente la vería. Buenos candidatos:
   ```
   T1098.004 SSH Authorized Keys: el atacante añade su clave pública a ~/.ssh/authorized_keys y vuelve cuando quiere; ninguna regla de Linux la etiqueta; la ve una vigilancia de auditd sobre esos archivos (-w /root/.ssh/authorized_keys -p wa) o un monitor de integridad de archivos.
   T1021.004 SSH: un inicio de sesión SSH con una clave o contraseña robada no ejecuta nada anómalo; lo ve el análisis del log de autenticación (journalctl -u ssh, "Accepted publickey for ... from ...") por hora y origen.
   T1071 Application Layer Protocol: el C2 por HTTP no crea procesos nuevos en cada orden; lo ve el proxy o un NIDS (Suricata, Zeek) por el destino y el beaconing.
   ```
9. En el Navigator, en la capa de huecos, pulsa el botón de guardar ("save layer") y elige "download single layer as json" para conservar la capa.

### Resultado esperado

Tres capas abiertas en el Navigator (TeamTNT, Sigma en verde y huecos en rojo), la salida de `huecos.py` con el recuento exacto, la capa de huecos descargada en JSON y `huecos.txt` con tres huecos explicados.

### Comprueba que lo lograste

- ¿Coincide el número de casillas rojas con la cifra "Huecos" del script? Si no, revisa que la expresión use la letra correcta de cada capa.
- ¿Qué porcentaje de las técnicas de TeamTNT cubren las reglas? Cubiertas dividido entre técnicas del grupo (en el ejemplo, 26/56, un 46 %).
- Una casilla verde, ¿significa que estás protegido contra esa técnica? No: significa que existe al menos una regla que la detecta en algún caso, y además solo funciona si tú recoges el log que la regla necesita (su `logsource`: auditd, journal o procesos del EDR).
- ¿Por qué varios huecos se cubren con otras fuentes y no con más reglas de procesos? Porque las reglas de proceso ven lo que se ejecuta en el equipo; la autenticación, el contenido de red o los contenedores necesitan otra telemetría.

### Limpieza

```bash
rm -rf ~/lab-navigator
```

- `rm -rf` → (ver ejercicio 1); aquí borra `~/lab-navigator`.

## Ejercicio 3: Superficie expuesta de tu dominio con OSINT

Nodo: [OSINT para defensores](README.md#osint-para-defensores) y la fase Reconnaissance de la [Cyber Kill Chain](README.md#cyber-kill-chain).

Objetivo: obtener de Certificate Transparency la lista de subdominios de un dominio propio, ver qué puertos y servicios de esos subdominios aparecen en el índice de Shodan, y producir una lista razonada de lo que no debería estar público.

Necesitas: un dominio tuyo o de una organización que te haya autorizado por escrito; Linux con `curl` y `python3`; navegador. Todo es OSINT pasivo: consultas a bases públicas, sin escanear nada. Tiempo estimado: 40 minutos.

Si no tienes dominio propio ni autorización: usa un dominio que esté dentro del alcance (scope) de un programa público de bug bounty, por ejemplo `hackerone.com`, que figura en el alcance del programa público de HackerOne en [hackerone.com/security](https://hackerone.com/security). Antes de empezar, lee la política del programa y confirma que el dominio sigue en el alcance. El alcance enumera hosts concretos, no todo `*.hackerone.com`: los subdominios que encuentres fuera de esa lista solo los anotas. Con un dominio ajeno trabajas en modo solo lectura: haces únicamente las consultas pasivas de los pasos 2 a 5 (bases públicas y resolución DNS normal), no te conectas a ningún host de la lista, no pruebas credenciales ni escaneas nada, y `revision.txt` queda como ejercicio privado: no publicas ni compartes lo que encuentres.

```bash
sudo apt install curl python3     # Debian/Ubuntu
sudo pacman -S curl python        # Arch
```

- `sudo`, `apt install`, `pacman -S`, `curl`, `python3`/`python`, `#` → (ver ejercicio 1).

### Pasos

1. Fija el dominio y crea la carpeta de trabajo:
   ```bash
   DOMINIO=tudominio.com
   mkdir -p ~/lab-osint && cd ~/lab-osint
   ```
   - `DOMINIO=tudominio.com` → guarda tu dominio en una variable de shell para usarlo como `$DOMINIO`; cambia `tudominio.com` por el tuyo, o por el dominio de bug bounty elegido si no tienes uno (ver "Necesitas").
   - `mkdir -p`, `&&`, `cd` → (ver ejercicio 1).
2. Abre en el navegador `https://crt.sh/?q=%25.tudominio.com` (el `%25` es un `%`, el comodín de crt.sh). Verás cada certificado emitido con su columna "Matching Identities": cada nombre ahí es un subdominio que alguien certificó, y que por tanto es público aunque nadie lo enlace. crt.sh a menudo responde `502 Bad Gateway` con dominios grandes; si pasa, reintenta o usa el paso 3.
3. Saca la lista de subdominios por la API de Cert Spotter, que lee los mismos registros de Certificate Transparency:
   ```bash
   curl -s "https://api.certspotter.com/v1/issuances?domain=$DOMINIO&include_subdomains=true&expand=dns_names" |
     python3 -c 'import json,sys; print("\n".join(sorted({n.removeprefix("*.").lower() for c in json.load(sys.stdin) for n in c["dns_names"]})))' \
     > subdominios.txt
   wc -l subdominios.txt
   cat subdominios.txt
   ```
   - `curl -s` → (ver ejercicio 1); la URL va entre comillas dobles para que `&` no lo interprete la shell y `$DOMINIO` sí se sustituya.
   - `https://api.certspotter.com/v1/issuances` → endpoint de Cert Spotter que lista los certificados emitidos para un dominio; sin clave admite un número limitado de consultas por hora.
   - `domain=$DOMINIO` → dominio cuyos certificados se buscan.
   - `include_subdomains=true` → incluye los subdominios de cualquier profundidad (por defecto es `false`).
   - `expand=dns_names` → añade a cada certificado el campo `dns_names`, la lista de nombres para los que es válido.
   - `|` → pasa el JSON de la respuesta a Python por la entrada estándar.
   - `python3 -c '...'` → (ver ejercicio 1).
   - `import json,sys` → módulos para JSON y para leer la entrada estándar (`sys.stdin`).
   - `json.load(sys.stdin)` → convierte la respuesta en una lista de certificados.
   - `for c in ... for n in c["dns_names"]` → recorre cada nombre de cada certificado.
   - `n.removeprefix("*.")` → quita el `*.` de los certificados comodín (requiere Python 3.9 o superior).
   - `.lower()` → pasa el nombre a minúsculas para no contar duplicados por mayúsculas.
   - `{...}` → conjunto: elimina los nombres repetidos.
   - `sorted(...)` y `"\n".join(...)` → ordena los nombres y los une con un salto de línea, uno por línea.
   - `\` al final de línea → continúa el comando en la línea siguiente.
   - `> subdominios.txt` → guarda la lista en ese archivo.
   - `wc -l subdominios.txt` → cuenta las líneas, es decir, cuántos subdominios hay.
   - `cat subdominios.txt` → muestra el contenido del archivo.
   Una línea por nombre. La API sin cuenta devuelve los certificados vigentes; los antiguos están en crt.sh.
4. Resuelve cada subdominio y consulta su IP en InternetDB, la base pública y sin clave de Shodan, que devuelve los puertos y vulnerabilidades que Shodan vio en esa IP:
   ```bash
   while read -r host; do
     ip=$(getent ahostsv4 "$host" | awk 'NR==1{print $1}')
     if [ -z "$ip" ]; then echo "$host -> no resuelve"; continue; fi
     printf '%s %s ' "$host" "$ip"
     curl -s "https://internetdb.shodan.io/$ip"; echo
   done < subdominios.txt | tee exposicion.txt
   ```
   - `while read -r host; do ... done` → bucle que lee una línea por vuelta y la guarda en la variable `host`; `-r` (ver ejercicio 1, paso 2) evita que `\` se interprete.
   - `< subdominios.txt` → el bucle lee sus líneas de ese archivo.
   - `ip=$(...)` → ejecuta el comando entre paréntesis y guarda su salida en la variable `ip`.
   - `getent ahostsv4 "$host"` → resuelve el nombre con el mismo mecanismo que usa el sistema (DNS, `/etc/hosts`) y devuelve solo direcciones IPv4, una línea por dirección y tipo de socket.
   - `awk 'NR==1{print $1}'` → de la salida anterior toma solo la primera línea (`NR==1`, número de registro 1) e imprime su primer campo (`$1`), la IP.
   - `if [ -z "$ip" ]; then ...; fi` → condición; `-z` es verdadero si la cadena está vacía, es decir, si el nombre no resolvió.
   - `echo "$host -> no resuelve"` → anota el subdominio que no tiene IP.
   - `continue` → salta a la siguiente vuelta del bucle sin consultar nada.
   - `printf '%s %s ' "$host" "$ip"` → imprime nombre e IP separados por espacio, sin salto de línea; cada `%s` se sustituye por un argumento.
   - `curl -s "https://internetdb.shodan.io/$ip"` → consulta InternetDB de Shodan para esa IP; devuelve en JSON puertos, CPE, etiquetas y CVE vistos.
   - `;` → separa dos comandos en la misma línea.
   - `echo` → sin argumentos, imprime un salto de línea para cerrar la línea de ese host.
   - `| tee exposicion.txt` → `tee` muestra la salida de todo el bucle en pantalla y a la vez la guarda en `exposicion.txt`.
   Salida de ejemplo:
   ```
   www.tudominio.com 203.0.113.10 {"cpes":["cpe:/a:f5:nginx"],"hostnames":[...],"ip":"203.0.113.10","ports":[80,443],"tags":["cdn"],"vulns":[]}
   staging.tudominio.com 198.51.100.7 {"cpes":["cpe:/a:openbsd:openssh:9.2p1"],"ip":"198.51.100.7","ports":[22,80,3306],"tags":[],"vulns":["CVE-..."]}
   viejo.tudominio.com -> no resuelve
   ```
   `{"detail":"No information available"}` significa que Shodan no tiene datos de esa IP.
5. Opcional, con cuenta gratuita de Shodan: busca en [shodan.io](https://www.shodan.io/) `hostname:tudominio.com` y abre cada resultado para ver el banner completo (versión del servidor, título de la página, certificado).
6. Clasifica cada subdominio en `revision.txt` con tres criterios y escribe la razón:
   ```
   - Nombre que delata un entorno interno o de pruebas (dev, staging, test, admin, vpn, jenkins, grafana).
   - Puerto de administración o de base de datos abierto (22, 3389, 3306, 5432, 6379, 9200, 27017).
   - Campo "vulns" no vacío (versión con CVE conocidos) o subdominio que ya no resuelve pero conserva certificado (posible activo olvidado).
   ```
   Formato sugerido de cada línea: `staging.tudominio.com | 3306 abierto y nombre de pruebas | no debería ser público: restringir a VPN`.

### Resultado esperado

`subdominios.txt` con los nombres sacados de Certificate Transparency, `exposicion.txt` con IP, puertos y CVE vistos por Shodan para cada uno, y `revision.txt` con los activos que no deberían estar expuestos y la acción propuesta.

### Comprueba que lo lograste

- ¿Por qué un subdominio aparece en Certificate Transparency aunque no esté enlazado en ninguna web? Porque las autoridades de certificación publican todos los certificados que emiten en registros públicos, y el nombre va dentro del certificado.
- ¿Escaneaste algún puerto? No: los puertos vienen del índice de Shodan, que es una fuente pública; por eso es OSINT pasivo.
- ¿En qué fase de la Kill Chain hace esto mismo un atacante y qué corte aplicas tú? Reconnaissance; reduces lo que se expone (cerrar o restringir lo listado en `revision.txt`).
- Para cada línea de `revision.txt`, ¿hay una acción concreta (cerrar puerto, mover tras VPN, borrar registro DNS, actualizar versión)? Si alguna dice solo "revisar", no está terminada.

### Limpieza

```bash
rm -rf ~/lab-osint
```

- `rm -rf` → (ver ejercicio 1); aquí borra `~/lab-osint`.

## Ejercicio 4: Caza por hipótesis de un cron sospechoso

Nodo: [Hunting por hipótesis](README.md#hunting-por-hipótesis), [Ciclo de threat hunting](README.md#ciclo-de-threat-hunting) y [Threat Hunting](README.md#threat-hunting).

Objetivo: simular en una VM propia la persistencia T1053.003 (un cron de un usuario sin privilegios que ejecuta un binario desde `/tmp`), escribir la hipótesis de caza y las consultas que la encuentran cruzando auditd con el journal, y comprobar que devuelven exactamente la actividad simulada.

Necesitas: VirtualBox o similar con una VM Ubuntu Server 24.04 (o Debian 12) con tu usuario administrador (con `sudo`); red NAT solo para instalar paquetes, luego red solo-anfitrión o desconectada. El "binario sospechoso" es una copia de `/usr/bin/id`, inofensiva. Tiempo estimado: 1 hora.

### Pasos

1. Haz una instantánea de la VM limpia (VirtualBox: Máquina > Tomar instantánea) para poder volver atrás al final.
2. En la VM, con tu usuario administrador, instala auditd y cron (en Ubuntu cron ya viene instalado) y comprueba que los dos servicios corren:
   ```bash
   sudo apt update && sudo apt install -y auditd cron
   systemctl is-active auditd cron
   ```
   - `sudo` → ejecuta el comando como root, necesario para instalar paquetes.
   - `apt update` → descarga la lista actualizada de paquetes de los repositorios.
   - `&&` → (ver ejercicio 1).
   - `apt install -y auditd cron` → instala el demonio de auditoría del kernel (`auditd`, que trae `auditctl`, `augenrules` y `ausearch`) y el planificador `cron`; `-y` responde "sí" a la confirmación.
   - `systemctl is-active auditd cron` → imprime el estado de cada servicio, uno por línea.
   Debe responder `active` dos veces. Ahora cambia la red de la VM a solo-anfitrión.
3. Crea las reglas de auditoría que generan los datos de la hipótesis. Crea `/etc/audit/rules.d/caza.rules` (por ejemplo con `sudo nano /etc/audit/rules.d/caza.rules`) con este contenido completo:
   ```
   -w /var/spool/cron/crontabs/ -p wa -k cron_persist
   -a always,exit -F arch=b64 -S execve -F auid>=1000 -F auid!=unset -k exec_usuario
   ```
   - `-w /var/spool/cron/crontabs/` → vigila esa carpeta, donde Debian y Ubuntu guardan el crontab de cada usuario (un archivo por usuario, con su nombre).
   - `-p wa` → registra las escrituras (`w`) y los cambios de atributos (`a`), como el renombrado y el cambio de permisos con que `crontab` instala el archivo.
   - `-k cron_persist` → etiqueta (key) que se pone a cada evento de la regla, para buscarlos después con `ausearch -k`.
   - `-a always,exit` → regla de llamada al sistema: añadir (`-a`) a la lista `exit`, que se evalúa al terminar la llamada, con la acción `always` (registrar siempre).
   - `-F arch=b64` → solo llamadas de programas de 64 bits; en x86_64 los números de llamada dependen de la arquitectura.
   - `-S execve` → la llamada `execve`, la que ejecuta un programa: es la creación de proceso de Linux.
   - `-F auid>=1000` → solo si el auid (audit UID, el usuario que inició la sesión, que no cambia con `sudo` ni con `su`) es de una persona; las cuentas de sistema están por debajo de 1000.
   - `-F auid!=unset` → descarta los procesos sin sesión de usuario (servicios arrancados por systemd), cuyo auid no está asignado.
   - `-k exec_usuario` → etiqueta de esta regla.

   Carga las reglas y compruébalas:
   ```bash
   sudo augenrules --load
   sudo auditctl -l
   ```
   - `augenrules --load` → une todos los archivos `.rules` de `/etc/audit/rules.d/` en `/etc/audit/audit.rules` y los carga en el kernel.
   - `auditctl -l` → lista las reglas cargadas en el kernel.
   Deben aparecer tus dos reglas, con `-F key=cron_persist` y `-F key=exec_usuario`.
4. Prepara la cuenta estándar, todavía como administrador:
   ```bash
   sudo useradd -m -s /bin/bash ana
   sudo passwd ana
   ```
   - `useradd` → crea una cuenta local.
   - `-m` → crea su directorio personal, `/home/ana`.
   - `-s /bin/bash` → shell de inicio de sesión de la cuenta.
   - `ana` → nombre de la cuenta; no entra en el grupo `sudo`, así que no tiene privilegios de administrador.
   - `passwd ana` → pide por teclado la contraseña nueva de `ana` (dos veces), sin escribirla en la línea de comandos.
   `ana` no puede usar `sudo`, como el atacante sin privilegios de la hipótesis.
5. Antes de simular nada, escribe la hipótesis en `~/hipotesis.txt` con el formato de la nota (técnica, dónde se vería, qué datos y qué ventana):
   ```
   Un atacante podría estar usando cron (T1053.003) para persistir en este
   servidor; si es así, veremos un crontab instalado por una cuenta que no es
   de administración cuyo comando apunta a /tmp, /var/tmp o /dev/shm, y ese
   binario ejecutándose cada pocos minutos lanzado por cron, en las últimas 24 horas.
   Datos: auditd (claves cron_persist y exec_usuario) y journal (crontab y CRON).
   ```
6. Simula la persistencia: cierra tu sesión (`exit`) y entra en la consola de la VM como `ana` (un `su` o `sudo -iu ana` no sirve, porque conservaría tu auid). Como `ana`, ejecuta:
   ```bash
   cp /usr/bin/id /tmp/.svc
   (crontab -l 2>/dev/null; echo '* * * * * /tmp/.svc') | crontab -
   crontab -l
   ```
   - `cp /usr/bin/id /tmp/.svc` → copia el programa `id` (inofensivo, imprime el usuario actual) a `/tmp`, una carpeta donde cualquier usuario puede escribir, con un nombre que empieza por punto para que `ls` no lo muestre, como haría un atacante; `cp` conserva el permiso de ejecución.
   - `( ... ; ... )` → ejecuta los dos comandos en una subshell y junta sus salidas en una sola.
   - `crontab -l` → imprime el crontab actual del usuario.
   - `2>/dev/null` → descarta los errores; si `ana` no tiene crontab, `crontab -l` se queja de `no crontab for ana`.
   - `echo '* * * * * /tmp/.svc'` → imprime la línea nueva: los cinco `*` (minuto, hora, día del mes, mes, día de la semana) significan "cada minuto", y luego va el comando.
   - `| crontab -` → instala como crontab de `ana` lo que llega por la tubería (`-` es la entrada estándar), sustituyendo al anterior; por eso se reimprime primero el actual.
   La última orden muestra `* * * * * /tmp/.svc`. Espera dos minutos para que cron lo ejecute al menos una vez y cierra la sesión de `ana`.
7. Vuelve a entrar con tu usuario administrador. Consulta 1, la instalación del crontab vista por auditd:
   ```bash
   sudo ausearch -k cron_persist --start today --format text
   ```
   - `ausearch` → busca en el log de auditd, `/var/log/audit/audit.log`, que solo puede leer root.
   - `-k cron_persist` → solo eventos con esa etiqueta (ver paso 3).
   - `--start today` → solo eventos desde las 00:00 de hoy; admite también `recent` (10 minutos), `yesterday` o una fecha y hora.
   - `--format text` → resume cada evento en una frase: quién, qué hizo, sobre qué y con qué programa.
   Verás varias líneas del mismo segundo (crea un temporal, le cambia el dueño, lo renombra) como esta:
   ```
   At 10:15:20 10/08/2026 ana successfully renamed /var/spool/cron/crontabs/ana using /usr/bin/crontab
   ```
   El sujeto es `ana`, no root, y el programa `/usr/bin/crontab`. Si `ana` hubiera hecho `sudo`, verías `<tu usuario>, acting as root`: el auid delata siempre quién abrió la sesión.
8. Consulta 2, la misma instalación vista por el journal, y el contenido de todos los crontabs con rutas temporales:
   ```bash
   sudo journalctl -t crontab --since "24 hours ago" --no-pager
   sudo grep -rE '/(tmp|var/tmp|dev/shm)/' /var/spool/cron/crontabs/
   ```
   - `journalctl` → consulta el journal de systemd; con `sudo` ve los mensajes de todos los usuarios.
   - `-t crontab` → solo mensajes con ese identificador de syslog, el que usa el programa `crontab`.
   - `--since "24 hours ago"` → la ventana de la hipótesis.
   - `--no-pager` → imprime sin abrir el paginador `less`.
   - `grep -rE` → (ver ejercicio 2, paso 2): búsqueda recursiva con expresión regular extendida.
   - `'/(tmp|var/tmp|dev/shm)/'` → cualquiera de las tres carpetas donde cualquier usuario puede escribir; `|` dentro del paréntesis significa "o".
   - `/var/spool/cron/crontabs/` → la carpeta de crontabs; solo root puede leerla, de ahí el `sudo`.
   ```
   oct 08 10:15:20 lab crontab[2250]: (ana) REPLACE (ana)
   /var/spool/cron/crontabs/ana:* * * * * /tmp/.svc
   ```
   Que el journal y auditd coincidan en hora (mismo segundo) y usuario es la primera correlación.
9. Consulta 3, la ejecución, vista por el journal (cron registra cada comando que lanza) y por auditd:
   ```bash
   sudo journalctl -t CRON --since "24 hours ago" --no-pager | grep -E 'CMD \(/(tmp|var/tmp|dev/shm)/'
   sudo ausearch -k exec_usuario --start today --format text | grep -E 'executed /(tmp|var/tmp|dev/shm)/'
   ```
   - `journalctl -t CRON --since ... --no-pager` → (ver paso 8); `CRON` es el identificador del demonio cron al ejecutar trabajos.
   - `grep -E 'CMD \(/(tmp|var/tmp|dev/shm)/'` → deja las líneas `CMD (` cuyo comando empieza en una carpeta temporal; `\(` es un paréntesis literal.
   - `ausearch -k exec_usuario --start today --format text` → (ver paso 7); eventos de la regla `execve`.
   - `grep -E 'executed /(tmp|var/tmp|dev/shm)/'` → deja los programas ejecutados desde esas carpetas.
   ```
   oct 08 10:16:01 lab CRON[2301]: (ana) CMD (/tmp/.svc)
   oct 08 10:17:01 lab CRON[2315]: (ana) CMD (/tmp/.svc)
   At 10:16:01 10/08/2026 ana successfully executed /tmp/.svc
   At 10:17:01 10/08/2026 ana successfully executed /tmp/.svc
   ```
   Una ejecución por minuto, siempre en el segundo 01: la regularidad es la firma de una tarea programada. El auid es `ana` porque cron abre una sesión PAM para cada trabajo con su usuario. Si además ves `(CRON) info (No MTA installed, discarding output)`, es cron descartando la salida de `id` porque no hay servidor de correo.
10. Comprueba qué es en realidad `/tmp/.svc`:
    ```bash
    sudo sha256sum /tmp/.svc /usr/bin/id
    ```
    - `sha256sum` → (ver ejercicio 1); calcula el hash de los dos archivos.
    Los dos hashes son iguales: el binario oculto es una copia renombrada de una herramienta del sistema. Un nombre cambiado para pasar desapercibido es enmascaramiento (T1036), una pista extra para la siguiente hipótesis.
11. Cierra el ciclo de caza (paso 4, informar y enriquecer): añade a `hipotesis.txt` el resultado (confirmada, con hora, usuario y comando), las consultas de los pasos 7 a 9 y una frase con la regla que propondrías al SIEM, por ejemplo "alertar ante un evento de auditd con clave cron_persist seguido de un CRON CMD cuyo comando esté en /tmp, /var/tmp o /dev/shm".

### Resultado esperado

`hipotesis.txt` con la hipótesis escrita antes de mirar los datos, las consultas que la confirman, la correlación entre auditd y el journal (mismo segundo y mismo usuario para la instalación, y una ejecución por minuto en ambos) y una propuesta de regla de detección.

### Comprueba que lo lograste

- ¿La consulta del paso 7 devuelve algo si no cargaste las reglas del paso 3? No: auditd sin reglas no vigila los crontabs; por eso la hipótesis debe nombrar datos que de verdad existen. El journal del paso 8 sí estaría, porque `crontab` lo escribe siempre.
- Si `ana` cambia el comando a `/home/ana/.cache/svc`, ¿lo encuentran tus consultas? No, porque filtran por carpetas temporales; amplía el patrón (por ejemplo, cualquier ruta oculta: `/\.`) y repite.
- Si en vez de cron usa un timer de usuario de systemd (T1053.006, `~/.config/systemd/user/`), ¿lo ven tus reglas? No: no cambia ningún crontab ni aparece `CRON` en el journal. Haría falta vigilar esas carpetas (`-w`) y buscar los procesos lanzados por `systemd --user`. Es el mismo hueco que viste en las reglas Sigma del ejercicio 2.
- ¿Por qué esto es threat hunting y no monitoreo? Porque partió de una hipótesis escrita por ti, no de una alerta; la regla del paso 11 es lo que lo convierte en monitoreo para la próxima vez.
- ¿Qué técnica de ATT&CK y qué fase de la Kill Chain representa el cron? T1053.003 (Persistence) y la fase 5, Installation.

### Limpieza

Con tu usuario administrador (o restaura la instantánea del paso 1):

```bash
sudo crontab -r -u ana
sudo rm -f /tmp/.svc
sudo userdel -r ana
sudo rm /etc/audit/rules.d/caza.rules
sudo augenrules --load
```

- `crontab -r` → borra el crontab.
- `-u ana` → del usuario `ana`, en vez del propio; solo root puede usarlo.
- `rm -f /tmp/.svc` → borra el binario copiado; `-f` no da error si ya no existe.
- `userdel -r ana` → borra la cuenta `ana`; `-r` borra también su directorio personal y su buzón.
- `rm /etc/audit/rules.d/caza.rules` → borra el archivo de reglas del paso 3.
- `augenrules --load` → (ver paso 3); vuelve a cargar las reglas, ya sin las tuyas.

Para desinstalar auditd: `sudo apt purge -y auditd`.
