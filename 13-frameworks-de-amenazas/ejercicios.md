# Ejercicios: Frameworks de amenazas

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

## Ejercicio 1: Por qué bloquear un hash no basta

Nodo: [Pirámide del dolor](README.md#pirámide-del-dolor) e [IOCs vs IOAs](README.md#iocs-vs-ioas).

Objetivo: tomar un hash de malware publicado hoy en MalwareBazaar, contar cuántas muestras distintas de la misma familia existen con otro hash, y justificar con ese número por qué un bloqueo por hash dura tan poco.

Necesitas: Linux (o WSL) con `curl` y `python3`; una cuenta gratuita en [abuse.ch](https://auth.abuse.ch/) para obtener tu Auth-Key; un navegador para VirusTotal (no hace falta cuenta para buscar por hash). Nunca descargas ni ejecutas ninguna muestra: solo consultas metadatos. Tiempo estimado: 30 minutos.

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
4. Pide todas las muestras recientes de esa familia (hasta 1000) y cuenta cuántos hashes distintos hay. Sustituye `AgentTesla` por la familia que elegiste:
   ```bash
   FAMILIA=AgentTesla
   curl -s -H "Auth-Key: $ABUSE_KEY" \
     --data "query=get_siginfo&signature=$FAMILIA&limit=1000" \
     https://mb-api.abuse.ch/api/v1/ > familia.json
   python3 -c '
   import json
   from collections import Counter
   d = json.load(open("familia.json"))
   hashes = {s["sha256_hash"] for s in d["data"]}
   por_dia = Counter(s["first_seen"][:10] for s in d["data"])
   print("hashes distintos:", len(hashes))
   for dia, n in sorted(por_dia.items())[-7:]:
       print(dia, n)
   '
   ```
   - `FAMILIA=AgentTesla` → crea una variable de shell con el nombre de la familia; sin espacios alrededor de `=`.
   - `curl -s -H "Auth-Key: $ABUSE_KEY"` → (ver paso 3).
   - `--data "query=get_siginfo&signature=$FAMILIA&limit=1000"` → petición POST; `get_siginfo` pide muestras por firma (familia), `signature=$FAMILIA` indica cuál y `limit=1000` pide hasta 1000 resultados, el máximo de la API.
   - `> familia.json` → guarda la respuesta en ese archivo.
   - `python3 -c '...'`, `import json`, `json.load(open(...))` → (ver paso 3).
   - `from collections import Counter` → importa `Counter`, un diccionario que cuenta apariciones.
   - `hashes = {s["sha256_hash"] for s in d["data"]}` → conjunto con los hashes; un conjunto no admite repetidos, así que su tamaño es el número de hashes distintos.
   - `Counter(s["first_seen"][:10] for s in d["data"])` → cuenta muestras por día; `[:10]` toma los 10 primeros caracteres de la fecha (`AAAA-MM-DD`).
   - `len(hashes)` → número de elementos del conjunto.
   - `sorted(por_dia.items())[-7:]` → ordena los pares (día, cantidad) por fecha y se queda con los 7 últimos.
   - `print(dia, n)` → imprime cada día con su número de muestras.

   Salida de ejemplo:
   ```
   hashes distintos: 1000
   2026-09-26 41
   2026-09-27 38
   ...
   ```
   El número de hashes distintos (y cuántos aparecen cada día) es tu dato. Si la API responde `illegal_signature` o `signature_not_found`, copia el nombre exacto tal como aparece en el paso 3.
5. Abre en el navegador `https://www.virustotal.com/gui/file/<hash>` con el hash elegido. En la pestaña Detection anota la proporción de motores que lo detectan y el "Popular threat label"; en la pestaña Details anota "First Submission Date" y la lista "Names". No subas ningún archivo: solo buscas por hash.
6. Busca en VirusTotal dos hashes más de la misma familia sacados de `familia.json` y compara: misma etiqueta de familia, hashes y nombres de archivo distintos.
7. Escribe en `conclusion.txt` una línea que use tu número, por ejemplo:
   ```bash
   echo "Bloquear este hash detiene 1 de 1000 muestras de $FAMILIA vistas en pocos días; el atacante obtiene otro hash recompilando, así que hay que subir en la pirámide (artefactos, herramienta, TTP)." > conclusion.txt
   ```
   - `echo "..."` → imprime el texto; las comillas dobles permiten que la shell sustituya `$FAMILIA` por el nombre de la familia.
   - `> conclusion.txt` → escribe esa línea en el archivo, sobrescribiéndolo si ya existía.

### Resultado esperado

Un `familia.json` con las muestras de una familia, el recuento de hashes distintos y por día, las anotaciones de VirusTotal de tres muestras de la misma familia y `conclusion.txt` con la justificación en una línea.

### Comprueba que lo lograste

- ¿Cuántos hashes distintos tiene la familia en tu consulta? Debe ser un número grande (decenas a cientos), muy superior a 1.
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

## Ejercicio 2: Huecos de cobertura de Sysmon frente a APT28

Nodo: [ATT&CK](README.md#attck) y [Los tres frameworks responden preguntas distintas](README.md#los-tres-frameworks-responden-preguntas-distintas).

Objetivo: superponer en el ATT&CK Navigator una capa con las técnicas de APT28 (G0007) y otra con las técnicas que cubre una configuración de Sysmon, obtener la lista exacta de técnicas de APT28 sin cobertura y explicar tres de esos huecos.

Necesitas: navegador; Linux (o WSL) con `curl` y `python3` para verificar el resultado; opcional, la VM Windows del ejercicio 4 con Sysmon instalado. La configuración usada es la "Balanced" de [sysmon-modular](https://github.com/olafhartong/sysmon-modular), que etiqueta cada regla con su técnica de ATT&CK y publica la capa del Navigator derivada de ella. Tiempo estimado: 45 minutos.

### Pasos

1. Descarga las dos capas en formato JSON del Navigator:
   ```bash
   mkdir -p ~/lab-navigator && cd ~/lab-navigator
   curl -sL -o apt28.json https://attack.mitre.org/groups/G0007/G0007-enterprise-layer.json
   curl -sL -o sysmon.json https://github.com/olafhartong/sysmon-modular/releases/latest/download/attack-matrix-15.21.json
   curl -sL -o sysmonconfig.xml https://github.com/olafhartong/sysmon-modular/releases/latest/download/sysmonconfig.xml
   python3 -c 'import json; [print(f, json.load(open(f))["versions"]) for f in ("apt28.json","sysmon.json")]'
   ```
   - `mkdir -p`, `&&`, `cd` → (ver ejercicio 1).
   - `curl -s` → (ver ejercicio 1).
   - `-L` → sigue las redirecciones HTTP; hace falta porque `releases/latest/download/...` de GitHub redirige al archivo real de la última versión.
   - `-o apt28.json` → guarda la respuesta en ese archivo en vez de mostrarla en pantalla (igual con `sysmon.json` y `sysmonconfig.xml`).
   - `https://attack.mitre.org/groups/G0007/G0007-enterprise-layer.json` → capa del Navigator que publica MITRE con las técnicas de APT28 (G0007) en la matriz Enterprise.
   - `.../attack-matrix-15.21.json` → capa del Navigator generada por sysmon-modular a partir de su configuración.
   - `.../sysmonconfig.xml` → la configuración "Balanced" de Sysmon de la que sale esa capa.
   - `python3 -c '...'` → (ver ejercicio 1).
   - `for f in ("apt28.json","sysmon.json")` → recorre los dos archivos; los corchetes `[...]` forman una lista por comprensión que solo se usa para ejecutar el `print` de cada uno.
   - `json.load(open(f))["versions"]` → lee el JSON y saca su campo `versions`, que dice la versión de ATT&CK, del Navigator y de la capa.
   Las dos deben decir `'attack': '19'` (o la misma versión entre sí). En `apt28.json` las técnicas usadas por APT28 tienen `score: 1`; en `sysmon.json` el `score` es el número de reglas de Sysmon que apuntan a esa técnica.
2. Comprueba de dónde sale la capa de Sysmon: cuenta las técnicas etiquetadas en la configuración.
   ```bash
   grep -o 'technique_id=T[0-9.]*' sysmonconfig.xml | sort -u | wc -l
   grep -o 'technique_id=T1053[0-9.]*' sysmonconfig.xml | sort | uniq -c
   ```
   - `grep` → busca un patrón (expresión regular) en el archivo `sysmonconfig.xml`.
   - `-o` → imprime solo el trozo que coincide, uno por línea, en vez de la línea entera.
   - `'technique_id=T[0-9.]*'` → patrón: el texto `technique_id=T` seguido de cualquier cantidad de dígitos y puntos (un ID como `T1053.005`).
   - `'technique_id=T1053[0-9.]*'` → igual, pero solo las etiquetas de T1053 y sus subtécnicas.
   - `|` → tubería: pasa la salida del comando de la izquierda como entrada del de la derecha.
   - `sort -u` → ordena las líneas; `-u` deja una sola copia de cada línea repetida.
   - `sort` → ordena las líneas, necesario para que `uniq` vea juntas las repetidas.
   - `uniq -c` → junta líneas repetidas consecutivas; `-c` antepone cuántas veces aparece cada una (aquí, cuántas reglas apuntan a cada técnica).
   - `wc -l` → cuenta líneas, es decir, el número de técnicas distintas.
   Unas 130 técnicas, y varias reglas para T1053 y T1053.005 (tareas programadas). Si ya tienes Sysmon con otra configuración en tu máquina, usa tu XML en vez de este y en PowerShell saca sus técnicas con:
   ```powershell
   Select-String -Path .\sysmonconfig.xml -Pattern 'technique_id=(T[\d.]+)' -AllMatches |
     ForEach-Object { $_.Matches } | ForEach-Object { $_.Groups[1].Value } | Sort-Object -Unique
   ```
   - `Select-String` → cmdlet que busca texto con expresiones regulares, parecido a `grep`.
   - `-Path .\sysmonconfig.xml` → archivo en el que busca, en la carpeta actual (`.\`).
   - `-Pattern 'technique_id=(T[\d.]+)'` → expresión regular; `\d` es un dígito, `[\d.]+` uno o más dígitos o puntos, y los paréntesis forman el grupo 1, que captura solo el ID de la técnica.
   - `-AllMatches` → recoge todas las coincidencias de cada línea, no solo la primera.
   - `|` → tubería de PowerShell: pasa objetos al siguiente cmdlet.
   - `ForEach-Object { $_.Matches }` → para cada línea encontrada (`$_` es el objeto actual), saca la lista de coincidencias.
   - `ForEach-Object { $_.Groups[1].Value }` → de cada coincidencia toma el texto del grupo 1, el ID de la técnica.
   - `Sort-Object -Unique` → ordena los IDs; `-Unique` elimina los repetidos.
3. Abre [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/). En la pestaña nueva elige "Open Existing Layer" y luego "Upload from local", y sube `apt28.json`. Abre otra pestaña (botón `+`) y sube igual `sysmon.json`.
4. En la pestaña de Sysmon, pulsa el botón "Color Setup" y en "Scoring Gradient" pon "low value" en 0 y "high value" en 1, con colores de blanco a verde. Así toda técnica con al menos una regla queda verde.
5. Abre una tercera pestaña, despliega "Create Layer from other layers", elige el dominio Enterprise y fíjate en la letra que el Navigator asigna a cada capa abierta (aparece en la pestaña: `a` para APT28 y `b` para Sysmon si las abriste en ese orden). En "Score Expression" escribe:
   ```
   (a > 0) and not (b > 0)
   ```
   - `a`, `b` → la puntuación de cada técnica en la capa con esa letra (APT28 y Sysmon).
   - `a > 0` → la técnica la usa APT28.
   - `b > 0` → la técnica tiene al menos una regla de Sysmon.
   - `and not` → las dos condiciones a la vez, negando la segunda: técnicas de APT28 sin regla de Sysmon. El resultado verdadero vale 1 y el falso 0.
   Pulsa "Create Layer". La capa nueva tiene puntuación 1 en las técnicas de APT28 sin regla de Sysmon. Pon su gradiente de 0 (blanco) a 1 (rojo) con "Color Setup".
6. Verifica la lista con un script, porque el Navigator distingue "sin puntuar" de "puntuación 0" y conviene confirmar el recuento. Guarda esto como `huecos.py`:
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
   print(f"Cubiertas por Sysmon: {len(grupo & cubiertas)}")
   print(f"Huecos: {len(grupo - cubiertas)}")
   for tid in sorted(grupo - cubiertas):
       print(tid)
   ```
   - `import json`, `import sys` → módulos para leer JSON y para acceder a los argumentos de la línea de comandos.
   - `def tecnicas_puntuadas(ruta):` → función que devuelve las técnicas con puntuación de una capa.
   - `open(ruta, encoding="utf-8")` → abre el archivo como texto UTF-8; `with` lo cierra solo al terminar.
   - `capa["techniques"]` → lista de técnicas de la capa del Navigator.
   - `t.get("score", 0) > 0` → se queda con las técnicas cuyo `score` es mayor que 0; si una no tiene `score`, cuenta como 0.
   - `{t["techniqueID"] ...}` → conjunto con los IDs de esas técnicas.
   - `sys.argv[1]`, `sys.argv[2]` → primer y segundo argumento del script: la capa del grupo y la de Sysmon.
   - `grupo & cubiertas` → intersección: técnicas del grupo que Sysmon cubre.
   - `grupo - cubiertas` → diferencia: técnicas del grupo sin cobertura, los huecos.
   - `f"..."` → cadena con formato; lo que va entre `{}` se evalúa e inserta.
   - `sorted(...)` → ordena los IDs antes de imprimirlos.
   ```bash
   python3 huecos.py apt28.json sysmon.json
   ```
   - `python3 huecos.py` → ejecuta el script con Python 3.
   - `apt28.json` → primer argumento (`sys.argv[1]`), la capa del grupo.
   - `sysmon.json` → segundo argumento (`sys.argv[2]`), la capa de cobertura.
   Salida de ejemplo (los números cambian con cada versión de ATT&CK y de la configuración):
   ```
   Técnicas del grupo: 101
   Cubiertas por Sysmon: 28
   Huecos: 73
   T1001.001
   T1006
   ...
   ```
   Cuenta las casillas rojas de la capa del paso 5 y compáralas con la cifra "Huecos".
7. Elige 3 huecos y abre la página de cada uno (`https://attack.mitre.org/techniques/T1110/003/` para T1110.003, por ejemplo). Para cada uno escribe en `huecos.txt`: el ID, qué hace la técnica, por qué Sysmon no la ve y qué otra fuente la vería. Buenos candidatos:
   ```
   T1110.003 Password Spraying: ocurre en la autenticación, no crea procesos; la ve el log Security (4625, 4771) o el del proveedor de identidad.
   T1071.001 Web Protocols: Sysmon ve la conexión (evento 3) pero no el contenido HTTP del C2; lo ve el proxy o un NIDS.
   T1078 Valid Accounts: un inicio de sesión con credenciales válidas no tiene nada anómalo en el endpoint; lo ve el análisis de inicios de sesión (hora, origen, 4624).
   ```
8. En el Navigator, en la capa de huecos, pulsa el botón de guardar ("save layer") y elige "download single layer as json" para conservar la capa.

### Resultado esperado

Tres capas abiertas en el Navigator (APT28, Sysmon en verde y huecos en rojo), la salida de `huecos.py` con el recuento exacto, la capa de huecos descargada en JSON y `huecos.txt` con tres huecos explicados.

### Comprueba que lo lograste

- ¿Coincide el número de casillas rojas con la cifra "Huecos" del script? Si no, revisa que la expresión use la letra correcta de cada capa.
- ¿Qué porcentaje de las técnicas de APT28 cubre la configuración? Cubiertas dividido entre técnicas del grupo (en el ejemplo, 28/101, un 28 %).
- Una casilla verde, ¿significa que estás protegido contra esa técnica? No: significa que hay al menos una regla que registra algo de ella; puede no cubrir todos sus procedimientos.
- ¿Por qué varios huecos se cubren con otras fuentes y no con más reglas de Sysmon? Porque Sysmon ve procesos, archivos, registro y conexiones del endpoint; la autenticación, el contenido de red o la nube necesitan otra telemetría.

### Limpieza

```bash
rm -rf ~/lab-navigator
```

- `rm -rf` → (ver ejercicio 1); aquí borra `~/lab-navigator`.

## Ejercicio 3: Superficie expuesta de tu dominio con OSINT

Nodo: [OSINT para defensores](README.md#osint-para-defensores) y la fase Reconnaissance de la [Cyber Kill Chain](README.md#cyber-kill-chain).

Objetivo: obtener de Certificate Transparency la lista de subdominios de un dominio propio, ver qué puertos y servicios de esos subdominios aparecen en el índice de Shodan, y producir una lista razonada de lo que no debería estar público.

Necesitas: un dominio tuyo o de una organización que te haya autorizado por escrito; Linux (o WSL) con `curl` y `python3`; navegador. Todo es OSINT pasivo: consultas a bases públicas, sin escanear nada. Tiempo estimado: 40 minutos.

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
   - `DOMINIO=tudominio.com` → guarda tu dominio en una variable de shell para usarlo como `$DOMINIO`; cambia `tudominio.com` por el tuyo.
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

## Ejercicio 4: Caza por hipótesis de una tarea programada sospechosa

Nodo: [Hunting por hipótesis](README.md#hunting-por-hipótesis), [Ciclo de threat hunting](README.md#ciclo-de-threat-hunting) y [Threat Hunting](README.md#threat-hunting).

Objetivo: simular en una VM propia la persistencia T1053.005 (tarea programada que ejecuta un binario desde `C:\Users\Public`), escribir la hipótesis de caza y la consulta que la encuentra cruzando el evento 1 de Sysmon con el evento 4698 de Security, y comprobar que la consulta devuelve exactamente la actividad simulada.

Necesitas: VirtualBox o similar con una VM Windows 10/11 Enterprise de evaluación (gratuita 90 días en el Microsoft Evaluation Center), con una cuenta administradora y una cuenta estándar; red NAT solo para descargar Sysmon y su configuración, luego red solo-anfitrión o desconectada. El "binario sospechoso" es una copia de `whoami.exe`, inofensiva. Tiempo estimado: 1 hora.

### Pasos

1. Haz una instantánea de la VM limpia (VirtualBox: Máquina > Tomar instantánea) para poder volver atrás al final.
2. En la VM, abre PowerShell como administrador e instala Sysmon con la configuración "Balanced" de sysmon-modular:
   ```powershell
   New-Item -ItemType Directory -Force C:\Tools\Sysmon | Out-Null
   Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile C:\Tools\Sysmon.zip
   Expand-Archive C:\Tools\Sysmon.zip -DestinationPath C:\Tools\Sysmon -Force
   Invoke-WebRequest -Uri https://github.com/olafhartong/sysmon-modular/releases/latest/download/sysmonconfig.xml -OutFile C:\Tools\Sysmon\sysmonconfig.xml
   C:\Tools\Sysmon\Sysmon64.exe -accepteula -i C:\Tools\Sysmon\sysmonconfig.xml
   Get-Service Sysmon64
   ```
   - `New-Item` → cmdlet que crea un elemento nuevo (archivo, carpeta, clave de registro).
   - `-ItemType Directory` → el elemento es una carpeta.
   - `-Force` → no da error si la carpeta ya existe y crea las carpetas padre que falten.
   - `C:\Tools\Sysmon` → ruta de la carpeta (es el parámetro `-Path` puesto por posición).
   - `| Out-Null` → descarta la salida para no llenar la pantalla con el objeto creado.
   - `Invoke-WebRequest` → cmdlet que descarga contenido por HTTP, parecido a `curl`.
   - `-Uri` → dirección que se descarga: el ZIP oficial de Sysmon en Sysinternals, y en la cuarta línea la configuración "Balanced" de sysmon-modular.
   - `-OutFile` → archivo local donde se guarda la descarga.
   - `Expand-Archive C:\Tools\Sysmon.zip` → descomprime ese ZIP (el primer argumento es `-Path`).
   - `-DestinationPath C:\Tools\Sysmon` → carpeta donde se extrae.
   - `-Force` → sobrescribe los archivos si ya existían.
   - `C:\Tools\Sysmon\Sysmon64.exe` → ejecutable de Sysmon de 64 bits.
   - `-accepteula` → acepta la licencia automáticamente, sin la ventana interactiva.
   - `-i C:\Tools\Sysmon\sysmonconfig.xml` → instala el servicio y el controlador de Sysmon usando ese archivo de configuración.
   - `-?` → muestra la ayuda, incluida la versión de Sysmon y del esquema; úsalo si Sysmon rechaza el XML por la versión de esquema, y descarga de la misma release el `sysmonconfig-<versión>.xml` que coincida.
   - `Get-Service Sysmon64` → muestra el estado del servicio con ese nombre (parámetro `-Name` por posición); debe aparecer `Running`.

   Ahora cambia la red de la VM a solo-anfitrión.
3. Activa la auditoría que genera el evento 4698 (subcategoría "Other Object Access Events"; en un Windows en español usa el GUID, que no depende del idioma):
   ```powershell
   auditpol /set /subcategory:"{0CCE9227-69AE-11D9-BED3-505054503030}" /success:enable /failure:enable
   auditpol /get /subcategory:"{0CCE9227-69AE-11D9-BED3-505054503030}"
   ```
   - `auditpol` → herramienta de Windows que consulta y cambia la política de auditoría del sistema.
   - `/set` → modifica la política.
   - `/subcategory:"{0CCE9227-69AE-11D9-BED3-505054503030}"` → subcategoría a la que se aplica, indicada por GUID en vez de por nombre; este GUID es "Other Object Access Events" y sirve igual en Windows en inglés y en español.
   - `/success:enable` → registra los intentos que tienen éxito (la creación de la tarea genera el 4698).
   - `/failure:enable` → registra también los intentos fallidos.
   - `/get` → muestra la configuración actual de la subcategoría indicada; debe mostrar `Success and Failure` (o `Aciertos y errores`).
4. Prepara la cuenta estándar y el binario. Todavía como administrador:
   ```powershell
   net user ana LabAna2026! /add
   Copy-Item C:\Windows\System32\whoami.exe C:\Users\Public\svc.exe
   ```
   - `net user` → comando de Windows para gestionar cuentas locales.
   - `ana` → nombre de la cuenta.
   - `LabAna2026!` → contraseña que se le asigna.
   - `/add` → crea la cuenta; por defecto entra solo en el grupo Usuarios, sin privilegios de administrador.
   - `Copy-Item` → cmdlet que copia un archivo; el primer argumento es el origen (`-Path`) y el segundo el destino (`-Destination`).
   - `C:\Windows\System32\whoami.exe` → programa inofensivo de Windows que imprime el usuario actual.
   - `C:\Users\Public\svc.exe` → copia con nombre cambiado en una carpeta donde cualquier usuario puede escribir, como haría un atacante.
   `ana` no pertenece al grupo Administradores, como el atacante sin privilegios de la hipótesis.
5. Antes de simular nada, escribe la hipótesis en `C:\Tools\hipotesis.txt` con el formato de la nota (técnica, dónde se vería, qué datos y qué ventana):
   ```
   Un atacante podría estar usando tareas programadas (T1053.005) para persistir
   en este equipo; si es así, veremos schtasks.exe ejecutado por una cuenta que no
   es de administración creando una tarea que apunta a C:\Users\Public, y un 4698
   cuyo TaskContent contiene esa ruta, en las últimas 24 horas.
   Datos: Sysmon evento 1 (creación de proceso) y Security evento 4698 (tarea creada).
   ```
6. Simula la persistencia: cierra sesión, entra como `ana` y en un `cmd` normal (sin elevar) ejecuta:
   ```cmd
   schtasks /create /tn "LabUpdater" /tr "C:\Users\Public\svc.exe" /sc minute /mo 15 /f
   schtasks /run /tn "LabUpdater"
   schtasks /query /tn "LabUpdater" /v /fo list
   ```
   - `schtasks` → herramienta de línea de comandos del Programador de tareas de Windows.
   - `/create` → crea una tarea programada.
   - `/tn "LabUpdater"` → nombre de la tarea (task name); en `/run` y `/query` indica a qué tarea se aplica.
   - `/tr "C:\Users\Public\svc.exe"` → programa que ejecuta la tarea (task run), con su ruta completa.
   - `/sc minute` → tipo de programación: por minutos.
   - `/mo 15` → modificador de la programación: con `minute`, cada 15 minutos (admite de 1 a 1439).
   - `/f` → crea la tarea sin avisos aunque ya exista una con ese nombre, sustituyéndola.
   - `/run` → ejecuta la tarea ahora, sin esperar a su horario.
   - `/query` → muestra información de tareas.
   - `/v` → salida detallada, con las propiedades avanzadas (`Task To Run`, repetición, usuario).
   - `/fo list` → formato de salida en lista, un campo por línea (también existen `TABLE` y `CSV`).
   La primera línea responde `SUCCESS: The scheduled task "LabUpdater" has successfully been created.` y en la tercera el campo `Task To Run` (`Tarea que se ejecutará` en español) muestra `C:\Users\Public\svc.exe` y la repetición indica cada 15 minutos.
7. Cierra la sesión de `ana`, vuelve a entrar como administrador y abre PowerShell elevado. Define una función que lee cualquier campo de un evento por su nombre:
   ```powershell
   function Get-EventField {
       param($Event, [string]$Name)
       (([xml]$Event.ToXml()).Event.EventData.Data | Where-Object Name -eq $Name).'#text'
   }
   ```
   - `function Get-EventField { ... }` → define una función de PowerShell con ese nombre, válida en esta sesión.
   - `param($Event, [string]$Name)` → sus dos parámetros: el evento a leer y el nombre del campo, forzado a texto con `[string]`.
   - `$Event.ToXml()` → devuelve el evento completo como texto XML.
   - `[xml]` → convierte ese texto en un documento XML navegable con puntos.
   - `.Event.EventData.Data` → lista de campos `<Data Name="...">` del evento.
   - `Where-Object Name -eq $Name` → se queda con el campo cuyo atributo `Name` es igual (`-eq`) al pedido.
   - `.'#text'` → devuelve el texto que contiene ese campo, es decir, su valor.
8. Consulta 1, la creación de la tarea vista por Sysmon (evento 1, `schtasks.exe` con `/create` y la ruta pública):
   ```powershell
   $desde = (Get-Date).AddHours(-24)
   Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1; StartTime=$desde} |
     Where-Object { (Get-EventField $_ 'Image') -like '*\schtasks.exe' -and
                    (Get-EventField $_ 'CommandLine') -match '/create' -and
                    (Get-EventField $_ 'CommandLine') -match 'Users\\Public' } |
     ForEach-Object { [pscustomobject]@{
         Hora        = $_.TimeCreated
         Usuario     = Get-EventField $_ 'User'
         Padre       = Get-EventField $_ 'ParentImage'
         CommandLine = Get-EventField $_ 'CommandLine' } } | Format-List
   ```
   - `$desde = (Get-Date).AddHours(-24)` → guarda en `$desde` la fecha y hora actual menos 24 horas, la ventana de la hipótesis.
   - `Get-WinEvent` → cmdlet que lee eventos de los registros de Windows.
   - `-FilterHashtable @{...}` → filtro que se aplica al leer el registro, más rápido que filtrar después; `@{...}` es una tabla de pares clave = valor separados por `;`.
   - `LogName='Microsoft-Windows-Sysmon/Operational'` → registro de Sysmon.
   - `Id=1` → solo eventos con ID 1, creación de proceso.
   - `StartTime=$desde` → solo eventos posteriores a esa fecha.
   - `Where-Object { ... }` → deja pasar solo los eventos para los que el bloque devuelve verdadero; `$_` es el evento actual.
   - `Get-EventField $_ 'Image'` → usa la función del paso 7 para leer la ruta del ejecutable.
   - `-like '*\schtasks.exe'` → comparación con comodines; `*` equivale a cualquier texto, así que acepta cualquier ruta que acabe en `\schtasks.exe`.
   - `-and` → las condiciones deben cumplirse todas.
   - `-match '/create'` → comparación con expresión regular: la línea de comandos contiene `/create`.
   - `-match 'Users\\Public'` → contiene `Users\Public`; en expresiones regulares la barra invertida se escribe doble (`\\`).
   - `ForEach-Object { ... }` → ejecuta el bloque para cada evento que pasó el filtro.
   - `[pscustomobject]@{...}` → crea un objeto con las propiedades indicadas, para mostrar solo lo que interesa.
   - `Hora = $_.TimeCreated` → fecha y hora del evento.
   - `Usuario = Get-EventField $_ 'User'` → cuenta que lanzó el proceso.
   - `Padre = Get-EventField $_ 'ParentImage'` → ejecutable del proceso padre.
   - `CommandLine = Get-EventField $_ 'CommandLine'` → línea de comandos completa.
   - `| Format-List` → muestra cada propiedad en su propia línea.
   Debe salir un resultado con `Usuario` igual a `<EQUIPO>\ana`, `Padre` igual a `C:\Windows\System32\cmd.exe` y la línea de comandos del paso 6.
9. Consulta 2, la misma tarea vista por Windows (evento 4698), filtrando por la ruta dentro de la definición XML de la tarea:
   ```powershell
   Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4698; StartTime=$desde} |
     Where-Object { (Get-EventField $_ 'TaskContent') -match 'Users\\Public' } |
     ForEach-Object { [pscustomobject]@{
         Hora    = $_.TimeCreated
         Usuario = Get-EventField $_ 'SubjectUserName'
         Tarea   = Get-EventField $_ 'TaskName'
         Comando = ([xml](Get-EventField $_ 'TaskContent')).Task.Actions.Exec.Command } } | Format-List
   ```
   - `Get-WinEvent -FilterHashtable`, `StartTime=$desde`, `Where-Object`, `-match`, `ForEach-Object`, `[pscustomobject]`, `Format-List` → (ver paso 8).
   - `LogName='Security'` → registro de seguridad de Windows.
   - `Id=4698` → solo eventos "se creó una tarea programada".
   - `Get-EventField $_ 'TaskContent'` → definición XML completa de la tarea creada.
   - `Usuario = Get-EventField $_ 'SubjectUserName'` → cuenta que creó la tarea.
   - `Tarea = Get-EventField $_ 'TaskName'` → nombre de la tarea con su carpeta (`\LabUpdater`).
   - `([xml](...)).Task.Actions.Exec.Command` → convierte la definición en XML y saca el programa que la tarea ejecuta.
   Debe salir `Usuario : ana`, `Tarea : \LabUpdater` y `Comando : C:\Users\Public\svc.exe`. Que las dos consultas coincidan en hora (segundos de diferencia) y usuario es la correlación que confirma la hipótesis.
10. Consulta 3, la ejecución de la tarea: el binario de la ruta pública lanzado por el servicio de tareas.
    ```powershell
    Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1; StartTime=$desde} |
      Where-Object { (Get-EventField $_ 'Image') -like 'C:\Users\Public\*' } |
      ForEach-Object { [pscustomobject]@{
          Hora             = $_.TimeCreated
          Image            = Get-EventField $_ 'Image'
          OriginalFileName = Get-EventField $_ 'OriginalFileName'
          ParentCommand    = Get-EventField $_ 'ParentCommandLine' } } | Format-List
    ```
    - `Get-WinEvent -FilterHashtable`, `LogName`, `Id=1`, `StartTime`, `Where-Object`, `ForEach-Object`, `[pscustomobject]`, `Format-List` → (ver paso 8).
    - `-like 'C:\Users\Public\*'` → cualquier ejecutable dentro de `C:\Users\Public`.
    - `Image = Get-EventField $_ 'Image'` → ruta del ejecutable lanzado.
    - `OriginalFileName = Get-EventField $_ 'OriginalFileName'` → nombre original que el binario lleva grabado en sus metadatos, aunque se haya renombrado.
    - `ParentCommand = Get-EventField $_ 'ParentCommandLine'` → línea de comandos del proceso padre.
    `ParentCommand` contiene `svchost.exe -k netsvcs -p -s Schedule` (el servicio Programador de tareas), y `OriginalFileName` dice `whoami.exe` aunque el archivo se llame `svc.exe`: un nombre cambiado que delata el enmascaramiento (T1036), una pista extra para la siguiente hipótesis.
11. Cierra el ciclo de caza (paso 4, informar y enriquecer): añade a `hipotesis.txt` el resultado (confirmada, con hora, usuario y tarea), las tres consultas y una frase con la regla que propondrías al SIEM, por ejemplo "alertar ante un 4698 cuyo TaskContent apunte a C:\Users\Public, AppData o Temp".

### Resultado esperado

`hipotesis.txt` con la hipótesis escrita antes de mirar los datos, las tres consultas de PowerShell que la confirman, la correlación entre el evento 1 de Sysmon y el 4698 de Security (misma hora y usuario) y una propuesta de regla de detección.

### Comprueba que lo lograste

- ¿La consulta del paso 9 devuelve algo si no activaste la auditoría del paso 3? No: el 4698 no se registra por defecto; por eso la hipótesis debe nombrar datos que de verdad existen.
- Si borras la tarea y la vuelves a crear apuntando a `C:\Windows\Temp\svc.exe`, ¿la encuentran tus consultas? No, porque filtran por `Users\Public`; amplía el filtro a `Users\\Public|AppData|\\Temp\\` y repite.
- ¿Por qué esto es threat hunting y no monitoreo? Porque partió de una hipótesis escrita por ti, no de una alerta; la regla del paso 11 es lo que lo convierte en monitoreo para la próxima vez.
- ¿Qué técnica de ATT&CK y qué fase de la Kill Chain representa la tarea? T1053.005 (Persistence) y la fase 5, Installation.

### Limpieza

En PowerShell como administrador (o restaura la instantánea del paso 1):

```powershell
schtasks /delete /tn "LabUpdater" /f
Remove-Item C:\Users\Public\svc.exe
net user ana /delete
auditpol /set /subcategory:"{0CCE9227-69AE-11D9-BED3-505054503030}" /success:disable /failure:disable
C:\Tools\Sysmon\Sysmon64.exe -u
```

- `schtasks /delete` → borra una tarea programada.
- `/tn "LabUpdater"` → (ver paso 6).
- `/f` → borra sin pedir confirmación.
- `Remove-Item C:\Users\Public\svc.exe` → cmdlet que borra el archivo indicado.
- `net user ana /delete` → borra la cuenta local `ana`.
- `auditpol /set /subcategory:"{...}"` → (ver paso 3).
- `/success:disable` → deja de registrar los intentos con éxito de esa subcategoría.
- `/failure:disable` → deja de registrar los intentos fallidos.
- `Sysmon64.exe -u` → desinstala el servicio y el controlador de Sysmon.
