# Programación y hacking práctico

## Conceptos previos

- Script: programa corto, normalmente interpretado (no se compila antes), que automatiza una tarea repetitiva.
- Lenguaje compilado vs interpretado: el compilado se traduce entero a código máquina antes de correr (C++, Go); el interpretado lo lee y ejecuta un intérprete línea por línea (Python, Bash, PowerShell, JavaScript).
- Binario o ejecutable: archivo con instrucciones de máquina listo para que el sistema operativo lo cargue (`.exe` en Windows, ELF en Linux).
- Proceso padre e hijo: cuando un programa lanza otro, el primero es el padre; la cadena `explorer.exe → cmd.exe → certutil.exe` se llama árbol de procesos y es una de las pistas favoritas del defensor.
- Hash: huella de tamaño fijo que sale de pasar un dato por una función de un solo sentido; si cambia un bit del dato, cambia el hash entero (ver [`10-criptografia`](../10-criptografia/)).
- Wordlist (diccionario): archivo de texto con una palabra candidata por línea; `rockyou.txt`, la más usada en labs, trae unos 14,3 millones de contraseñas reales filtradas en 2009.
- Vulnerabilidad, exploit y payload: la vulnerabilidad es la falla; el exploit es el código que la dispara; el payload es lo que se ejecuta después de dispararla (ver [`11-ataques-y-amenazas`](../11-ataques-y-amenazas/)).
- Shell y reverse shell: una shell es un intérprete de comandos; en una reverse shell es la víctima la que abre la conexión hacia el atacante, lo que atraviesa firewalls que solo bloquean conexiones entrantes.
- Escalada de privilegios: pasar de un usuario con pocos permisos a uno con más (de `www-data` a `root`, de usuario a administrador).
- SUID y sudo: SUID es un permiso de Linux que hace que un programa corra con los privilegios de su dueño (a menudo `root`); `sudo` permite a un usuario correr comandos concretos como otro (ver [`02-linux`](../02-linux/)).
- Active Directory (AD): servicio de directorio de Windows que centraliza usuarios, equipos y permisos de una empresa; se autentica con Kerberos y NTLM (ver [`08-autenticacion`](../08-autenticacion/)).
- Evento de Windows (Event ID): registro numerado que Windows guarda en el visor de eventos; `4624` es un login correcto, `4625` uno fallido, `4688` la creación de un proceso.
- Sysmon: herramienta gratuita de Microsoft (Sysinternals) que registra con mucho más detalle lo que pasa en un equipo Windows: `Event ID 1` creación de proceso con su línea de comandos, `Event ID 3` conexión de red.
- SIEM y EDR: el SIEM junta y correlaciona logs de toda la red; el EDR es un agente en cada equipo que vigila procesos y puede bloquearlos (ver [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/)).
- Laboratorio propio: máquinas virtuales tuyas o plataformas legales (TryHackMe, Hack The Box, DVWA, Juice Shop) donde sí tienes permiso de atacar. Todo lo ofensivo de esta nota se piensa para ese contexto (ver [`06-virtualizacion`](../06-virtualizacion/)).

## Programming Skills

Saber programar en seguridad no significa escribir aplicaciones grandes: significa poder leer el código que encuentras (un script malicioso, una aplicación web, un exploit público) y escribir tus propias herramientas pequeñas cuando la que existe no hace exactamente lo que necesitas. Cada lenguaje del roadmap domina un terreno distinto.

```
Terreno                         Lenguaje que manda
------------------------------  -------------------------------
Automatizar casi cualquier cosa Python
Herramientas rápidas y portables Go
Aplicaciones web y navegador    JavaScript
Memoria, malware, exploits      C++ (y C)
Terminal y servidores Linux     Bash
Equipos y dominios Windows      PowerShell
```

### Python

**Python** es un lenguaje interpretado de propósito general que, por su sintaxis legible y su enorme biblioteca estándar, se volvió el idioma por defecto para automatizar tareas de seguridad.

Existe en este roadmap porque casi toda tarea de seguridad es "leer algo, filtrarlo y actuar": parsear miles de líneas de log, comparar hashes, hablar con una API, armar un paquete de red. Python trae de fábrica módulos para cada cosa (`hashlib` para hashes, `socket` para red, `ipaddress` para subredes, `json` y `csv` para datos) y una comunidad que publica librerías especializadas: `scapy` para fabricar y leer paquetes, `requests` para HTTP, `pwntools` para ejercicios de explotación en CTF, `impacket` para protocolos de Windows. Muchas herramientas de pentest famosas están escritas en Python, así que leerlo es tan importante como escribirlo.

Por dentro, Python compila tu archivo a un bytecode intermedio (`.pyc`) que una máquina virtual ejecuta instrucción por instrucción. Eso lo hace más lento que C++ o Go, pero en seguridad casi nunca importa: el cuello de botella suele ser la red o el disco, no la CPU.

Analogía: Python es la navaja suiza. No es la mejor sierra ni el mejor destornillador, pero la llevas siempre y resuelve el 80 % de los problemas en dos minutos.

Ejemplo: verificar que nadie cambió los archivos de una carpeta. Guardas un manifiesto con el SHA-256 de cada archivo y el script lo compara con el estado actual. Es la idea detrás de las herramientas de integridad de archivos (FIM) como AIDE o Tripwire.

```python
#!/usr/bin/env python3
"""Compare files against a manifest of SHA-256 hashes ("<hash>  <path>" per line)."""
import hashlib
import sys
from pathlib import Path

CHUNK_SIZE = 64 * 1024  # read 64 KiB at a time so large files do not fill memory


def sha256_of(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for chunk in iter(lambda: handle.read(CHUNK_SIZE), b""):
            digest.update(chunk)
    return digest.hexdigest()


def main(manifest: str) -> int:
    changed = 0
    for line in Path(manifest).read_text().splitlines():
        expected, name = line.split(maxsplit=1)
        path = Path(name)
        status = "MISSING" if not path.exists() else (
            "OK" if sha256_of(path) == expected else "CHANGED")
        changed += status != "OK"
        print(f"{status:8} {name}")
    return 1 if changed else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1]))
```

```
$ sha256sum /etc/passwd /etc/hosts > base.txt     # snapshot today
$ python3 check_hashes.py base.txt                # run again tomorrow
OK       /etc/passwd
CHANGED  /etc/hosts
```

El código de salida `1` cuando algo cambió permite encadenarlo con `cron` o con una alerta.

> [!TIP]
> Si solo vas a aprender un lenguaje para seguridad, que sea Python: es el que más herramientas, cursos y respuestas en foros tiene, y el que más vas a encontrar leyendo exploits públicos.

```
Python → automatización general, parseo de logs, red, herramientas de pentest.
hashlib → módulo estándar de Python para calcular hashes.
scapy → librería para fabricar y analizar paquetes.
```

### Go

**Go** (o Golang) es un lenguaje compilado creado por Google en 2009 que produce un único ejecutable estático, sin dependencias, y que maneja la concurrencia de forma nativa.

Existe porque Python obliga a tener el intérprete y las librerías instaladas en la máquina donde corre el script, y en un pentest esa máquina rara vez es tuya. Go compila todo dentro de un solo binario y además hace compilación cruzada con dos variables (`GOOS=windows GOARCH=amd64 go build`) para sacar un `.exe` desde Linux. Por eso muchas herramientas modernas están escritas en Go: Gobuster, ffuf, Nuclei, Amass, subfinder, httpx y el framework Sliver. Por la misma razón, también hay cada vez más malware en Go, y un analista lo reconoce por binarios grandes (de 2 a 10 MB para algo trivial) llenos de nombres de funciones del runtime de Go.

Por dentro, su ventaja es la goroutine: una tarea concurrente que cuesta unos pocos KB de memoria, frente a 1 MB o más de un hilo del sistema. Lanzar 1000 goroutines que prueban 1000 puertos a la vez es normal; los canales (`chan`) permiten que se pasen resultados sin pisarse.

Analogía: Python es una receta que necesita tu cocina; Go es una comida precocida en su envase: la llevas a cualquier casa y solo hay que calentarla.

Ejemplo: comprobar qué puertos de **tu propio** equipo o de un servidor tuyo están escuchando, en paralelo y con tiempo límite.

```go
// portcheck: report which TCP ports answer on a host you own.
package main

import (
	"fmt"
	"net"
	"os"
	"sync"
	"time"
)

const dialTimeout = 500 * time.Millisecond

var ports = []int{22, 80, 443, 3306, 5432, 8080}

func main() {
	host := os.Args[1]
	var wg sync.WaitGroup
	for _, port := range ports {
		wg.Add(1)
		go func(p int) {
			defer wg.Done()
			conn, err := net.DialTimeout("tcp", fmt.Sprintf("%s:%d", host, p), dialTimeout)
			if err != nil {
				return
			}
			conn.Close()
			fmt.Printf("%d/tcp open\n", p)
		}(port)
	}
	wg.Wait()
}
```

```
$ go build -o portcheck . && ./portcheck 127.0.0.1
22/tcp open
5432/tcp open
```

Las seis comprobaciones corren a la vez, así que el script tarda como mucho 0,5 s en lugar de 3 s. Escanear redes que no son tuyas puede ser delito aunque solo "mires": este tipo de prueba se hace en tu red local o dentro del alcance firmado (ver [Penetration Testing Rules of Engagement](#penetration-testing-rules-of-engagement)).

```
Go → binario único y portable, concurrencia barata; herramientas modernas y C2.
Goroutine → tarea concurrente ligera de Go.
Compilación cruzada → compilar para otro sistema operativo desde el tuyo.
```

### JavaScript

**JavaScript** es el lenguaje que ejecutan todos los navegadores web y, con Node.js, también los servidores, por lo que es el idioma en el que viven la mayoría de las vulnerabilidades del lado del cliente.

Existe en el roadmap porque para auditar una aplicación web hay que leer su JavaScript: ahí aparecen rutas de API ocultas, claves olvidadas, validaciones que solo ocurren en el navegador (y por tanto se saltan con un proxy) y los puntos donde un dato del usuario termina dentro de `innerHTML` o `eval`, que es donde nace el XSS (ver [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/)). La consola del navegador (F12) es un intérprete de JavaScript con acceso a la página, a sus cookies no protegidas y a su `localStorage`. Del lado del servidor, Node.js y su gestor npm traen otro riesgo: paquetes de terceros maliciosos o secuestrados, un ataque de cadena de suministro.

Por dentro, el navegador ejecuta el JavaScript de cada sitio en una caja aislada por origen (same-origin policy): el código de `tienda.com` no puede leer las respuestas de `banco.com`. Casi todo ataque web del lado del cliente consiste en lograr que el navegador ejecute tu código dentro del origen equivocado.

Analogía: el JavaScript de una página es el folleto de instrucciones que el restaurante le da al cliente. Si el restaurante confía en que el cliente siga el folleto al pie de la letra (validación solo en el navegador), cualquiera puede escribir su propio pedido.

Ejemplo: un script de Node.js que revisa el log de acceso de **tu** servidor web y señala las IP con demasiadas respuestas 404, la huella típica de una herramienta de enumeración como Gobuster.

```javascript
// flag404.js: list IPs with too many 404 responses in an access log (combined format).
const fs = require("node:fs");

const THRESHOLD = 100;              // 404s per IP that look like automated enumeration
const LINE = /^(\S+) .*" (\d{3}) /; // client IP ... "request" status

const counts = new Map();
for (const line of fs.readFileSync(process.argv[2], "utf8").split("\n")) {
  const m = LINE.exec(line);
  if (m && m[2] === "404") counts.set(m[1], (counts.get(m[1]) ?? 0) + 1);
}
for (const [ip, n] of [...counts].sort((a, b) => b[1] - a[1])) {
  if (n >= THRESHOLD) console.log(`${ip}\t${n} x 404`);
}
```

```
$ node flag404.js /var/log/nginx/access.log
203.0.113.50	4812 x 404
198.51.100.7	133 x 404
```

Una IP con casi 5.000 respuestas 404 en un día no es una persona perdida: es una herramienta probando miles de rutas de una lista. Ese es el rastro que deja la enumeración web en tus logs.

```
JavaScript → lenguaje del navegador y, con Node.js, del servidor; donde nace el XSS.
Same-origin policy → el código de un sitio no puede leer los datos de otro.
npm → gestor de paquetes de Node; riesgo de cadena de suministro.
```

### C++

**C++** es un lenguaje compilado de bajo nivel que da control directo sobre la memoria, y por eso es el lenguaje en el que están escritos los sistemas operativos, los navegadores, muchos programas maliciosos y la mayoría de los fallos de corrupción de memoria.

Existe en el roadmap por dos motivos. El primero: para entender un buffer overflow, un use-after-free o una fuga de memoria (ver [`12-ataques-web-y-de-red`](../12-ataques-web-y-de-red/)) hay que ver cómo un programa en C o C++ reserva y libera memoria a mano. El segundo: el análisis de malware y la ingeniería inversa trabajan sobre binarios compilados desde C/C++, y reconocer sus patrones en el ensamblador (ver [Basics of Reverse Engineering](#basics-of-reverse-engineering)) exige haberlos escrito alguna vez.

Por dentro, el compilador traduce el código directamente a instrucciones de la CPU; no hay máquina virtual ni recolector de basura. El programador decide cuántos bytes reserva y cuándo los libera, y nada impide escribir más allá del final de un arreglo: el programa simplemente pisa lo que haya al lado.

Analogía: C++ es conducir un coche de carreras sin control de tracción. Llegas más rápido que nadie, pero si te equivocas en una curva no hay nada que te salve.

Ejemplo: la diferencia entre la copia insegura y la segura de un texto a un búfer de 16 bytes.

```cpp
#include <cstring>
#include <string>

void unsafe_copy(const char* input) {
    char name[16];
    strcpy(name, input);   // no length check: input longer than 15 chars overwrites adjacent memory
}

void safe_copy(const char* input) {
    std::string name(input);   // std::string grows as needed; no fixed buffer to overflow
}
```

Si `input` trae 200 caracteres, `unsafe_copy` escribe 184 bytes fuera del arreglo y corrompe la pila del programa; `safe_copy` simplemente reserva lo que haga falta. Los compiladores modernos añaden defensas (canarios de pila, ASLR, DEP/NX) que dificultan aprovechar ese fallo, pero el remedio de fondo es no usar funciones sin control de longitud.

```
C++ → control directo de memoria; base de SO, navegadores, malware y exploits de memoria.
strcpy → copia sin límite; origen clásico de buffer overflows.
std::string → tipo que gestiona su propia memoria; evita el desbordamiento.
```

### Bash

**Bash** es el intérprete de comandos por defecto en la mayoría de sistemas Linux y el lenguaje con el que se automatizan tareas en la terminal, encadenando programas existentes.

Existe en el roadmap porque los servidores, los contenedores y casi todas las herramientas de seguridad viven en Linux, y Bash es la forma de pegarlas entre sí. Su poder no está en el lenguaje en sí, que es limitado, sino en las tuberías: la salida de un programa entra como entrada del siguiente (ver [`02-linux`](../02-linux/)). Un analista que domina `grep`, `awk`, `sort`, `uniq` y `cut` responde en una línea preguntas que en otro lenguaje llevarían un script entero.

Analogía: Bash es el director de orquesta. No toca ningún instrumento, pero hace que cada músico (cada comando) entre en el momento justo.

Ejemplo: contar los intentos fallidos de SSH por IP en tu servidor y mostrar los cinco orígenes más insistentes.

```bash
#!/usr/bin/env bash
# top_ssh_failures.sh: top 5 source IPs of failed SSH logins in the last day.
set -euo pipefail

journalctl -u ssh --since "24 hours ago" --no-pager \
  | grep "Failed password" \
  | grep -oE "from [0-9.]+" \
  | awk '{print $2}' \
  | sort | uniq -c | sort -rn | head -n 5
```

```
$ ./top_ssh_failures.sh
   2140 203.0.113.50
    310 198.51.100.7
     12 192.0.2.33
```

`set -euo pipefail` hace que el script se detenga ante el primer error en vez de seguir con datos incompletos; es la primera línea de cualquier script de Bash serio. ShellCheck revisa los errores típicos (variables sin comillas, por ejemplo).

```
Bash → pegamento de la terminal Linux; tuberías entre comandos.
Tubería (|) → la salida de un comando entra al siguiente.
set -euo pipefail → detener el script ante errores.
```

### Power Shell

**PowerShell** es el intérprete de comandos y lenguaje de scripting de Microsoft que trabaja con objetos en lugar de texto, y es la herramienta central para administrar equipos Windows, Active Directory y Azure.

Existe en el roadmap porque es un arma de doble filo. Para el defensor es la forma de consultar a escala cientos de equipos (servicios, usuarios, eventos, configuración). Para el atacante es una herramienta que ya viene instalada y firmada por Microsoft, así que muchos ataques "sin archivos" la usan para no dejar ejecutables (ver [Using tools for Unintended Purposes](#using-tools-for-unintended-purposes)). Entenderla es requisito para detectar su abuso.

Por dentro, cada comando (cmdlet, con forma Verbo-Sustantivo como `Get-Process`) devuelve objetos .NET con propiedades, no líneas de texto. Por eso se filtra por campo (`Where-Object Id -eq 4625`) en lugar de recortar texto con expresiones regulares.

Analogía: si Bash pasa hojas de papel de un comando a otro, PowerShell pasa fichas de un archivador, con campos que se pueden consultar por nombre.

Ejemplo: listar los inicios de sesión fallidos (evento 4625) de las últimas 24 horas en tu equipo, agrupados por cuenta.

```powershell
# failed_logons.ps1: failed logons (event 4625) in the last 24 h, grouped by account.
$since = (Get-Date).AddHours(-24)
Get-WinEvent -FilterHashtable @{ LogName = 'Security'; Id = 4625; StartTime = $since } |
    ForEach-Object { $_.Properties[5].Value } |   # TargetUserName field
    Group-Object | Sort-Object Count -Descending |
    Select-Object Count, Name -First 5
```

```
Count Name
----- ----
  812 administrador
   44 ana
    3 bruno
```

Defensas frente al abuso de PowerShell: activar el registro de bloques de script (Script Block Logging, evento 4104) y de módulos, que guarda el código realmente ejecutado aunque llegue ofuscado; usar el modo de lenguaje restringido y JEA (Just Enough Administration) para limitar qué puede hacer cada administrador; y alertar cuando un programa de oficina lanza PowerShell.

```
PowerShell → shell de Windows basado en objetos; administración y también abuso.
Cmdlet → comando con forma Verbo-Sustantivo que devuelve objetos.
Script Block Logging → registra el código PowerShell ejecutado (evento 4104).
```

## Understand Common Hacking Tools

Las **herramientas de hacking** son programas que automatizan las tareas repetitivas de una prueba de seguridad (descubrir, probar, validar) y que usan por igual los pentesters autorizados y los atacantes reales. Conocerlas sirve para dos cosas: usarlas en un laboratorio o en una prueba autorizada, y reconocer su rastro cuando alguien las usa contra ti.

Analogía: son las herramientas de un cerrajero. El mismo juego de ganzúas sirve para abrir la puerta del cliente que perdió sus llaves y para robar; lo que cambia es el permiso.

Se agrupan por la tarea que resuelven:

- Proxies de interceptación web (Burp Suite, OWASP ZAP): se ponen entre el navegador y la aplicación para ver y modificar cada petición. Muestran que la validación hecha solo en el navegador no protege nada.
- Escáneres de red (nmap, ver [`07-herramientas-de-red`](../07-herramientas-de-red/)): descubren equipos, puertos y servicios.
- Enumeración de contenido web (Gobuster, ffuf): prueban miles de rutas de una lista para encontrar páginas no enlazadas.
- Crackeo de hashes offline (John the Ripper, hashcat): prueban contraseñas candidatas contra hashes ya obtenidos; la defensa es usar hashes lentos y con sal (ver [`10-criptografia`](../10-criptografia/)).
- Ataques de contraseña en línea (Hydra): prueban credenciales contra un servicio activo; la defensa es MFA, bloqueo y detección (ver [`11-ataques-y-amenazas`](../11-ataques-y-amenazas/)).

Cómo se ven desde el lado defensivo, que es lo que más importa en un SOC:

```
Herramienta          Rastro típico en tus logs
-------------------  -----------------------------------------------------------
Escáner de puertos   Conexiones a muchos puertos desde una IP en segundos
Enumeración web      Miles de 404 desde una IP, agente de usuario de la herramienta
Fuerza bruta online  Cientos de fallos de login (4625, "Failed password") seguidos
Proxy web            Peticiones con parámetros que el formulario nunca enviaría
Crackeo offline      Nada en tus logs: ocurre en el equipo del atacante
```

La última fila es la importante: el crackeo offline no se ve, así que la única defensa es que el hash robado sea muy caro de romper.

Ejemplo: en tu propio laboratorio con DVWA levantas Burp Suite, interceptas el envío del formulario de login y ves la contraseña viajando en el cuerpo de la petición. Si el sitio no usa HTTPS, cualquiera en tu red vería lo mismo.

```
Burp/ZAP     → interceptar y modificar tráfico web.
Gobuster     → descubrir rutas web ocultas por fuerza bruta de nombres.
John/hashcat → crackear hashes offline; invisible para el defensor.
Hydra        → probar credenciales contra un servicio en línea; ruidoso.
```

## Understand Common Exploit Frameworks

Un **exploit framework** es una plataforma que organiza exploits, payloads y herramientas de post-explotación en módulos intercambiables, para que una prueba de penetración no tenga que escribir cada pieza desde cero.

Analogía: es una caja de herramientas modular con piezas estándar: el mango (el framework) es siempre el mismo y le acoplas la punta que necesita cada tornillo.

Los conceptos que comparten todos:

- Vulnerabilidad: el fallo en el software.
- Exploit: el código que aprovecha ese fallo concreto para conseguir algo (normalmente ejecutar código).
- Payload: lo que se ejecuta una vez que el exploit funcionó, por ejemplo una sesión remota.
- Módulo: cada pieza del framework (exploit, payload, escáner auxiliar, post-explotación).
- C2 (command and control): el servidor desde el que el operador controla los equipos comprometidos.

```
Vulnerabilidad  →  Exploit (abre la puerta)  →  Payload (lo que entra)  →  C2 (control remoto)
```

Los frameworks del roadmap:

- Metasploit: el framework abierto de referencia para aprender; miles de módulos de exploits conocidos y una base de datos de resultados. Es el que se usa en los laboratorios de TryHackMe y en las máquinas de VulnHub.
- Cobalt Strike: framework comercial de "simulación de adversario" para equipos rojos profesionales. Sus versiones piratas son muy usadas por grupos criminales, por eso los defensores conocen bien su rastro.
- Sliver: alternativa abierta y moderna (escrita en Go) centrada en el C2.

Desde el lado defensivo: los exploits de estos frameworks aprovechan vulnerabilidades ya públicas, así que parchear a tiempo los deja sin munición; sus payloads y su tráfico de C2 tienen firmas que los EDR e IDS conocen; y el beaconing (conexiones periódicas al C2, por ejemplo cada 60 segundos) se ve en el análisis de tráfico (ver [`15-deteccion-y-monitoreo`](../15-deteccion-y-monitoreo/)).

Ejemplo: en el laboratorio de TryHackMe de Metasploit buscas un módulo por el CVE de un servicio vulnerable de la máquina del laboratorio, configuras el objetivo, eliges un payload y obtienes una sesión. Lo instructivo es lo de después: mirar en los logs de esa máquina qué rastro quedó.

> [!IMPORTANT]
> Un exploit framework no crea vulnerabilidades: solo automatiza el aprovechamiento de las que ya están publicadas. Un sistema parcheado deja inútiles la mayoría de sus módulos.

```
Exploit  → código que aprovecha un fallo concreto.
Payload  → lo que se ejecuta tras el exploit.
C2       → servidor desde el que se controla lo comprometido.
Metasploit → framework abierto de referencia para aprender.
Cobalt Strike → framework comercial de red team, muy abusado por criminales.
```

## Using tools for Unintended Purposes

Usar herramientas para fines no previstos, conocido como **living off the land** (LotL), es la técnica de abusar de programas legítimos que ya vienen instalados en el sistema para tareas maliciosas, en lugar de traer malware propio.

Analogía: el ladrón que no trae herramientas y usa la escalera y el martillo que encontró en el garaje de la casa; nadie sospecha de objetos que siempre estuvieron ahí.

Por qué funciona: los programas del sistema están firmados por el fabricante, los antivirus confían en ellos y los administradores los usan a diario, así que su uso no levanta alarmas por sí solo. A estos programas abusables se les llama LOLBins (Living Off the Land Binaries).

Los tres catálogos del roadmap documentan qué puede hacerse con cada binario legítimo, y los defensores los usan para saber qué vigilar:

- LOLBAS: binarios, scripts y librerías de Windows abusables (por ejemplo, utilidades del sistema que pueden descargar archivos o ejecutar código de forma inesperada).
- GTFOBins: binarios de Linux que permiten saltarse restricciones, por ejemplo escalar privilegios si tienen el bit SUID o se permiten con `sudo` (ver [`02-linux`](../02-linux/)).
- WADComs: hoja de referencia interactiva de herramientas y comandos para entornos Windows y Active Directory.

Ejemplo defensivo: auditas tu servidor Linux con `sudo -l` y descubres que un usuario puede ejecutar un editor de texto como root sin contraseña. Buscas ese editor en GTFOBins y ves que desde él se puede abrir una shell: ese permiso equivale a dar root completo. La corrección es quitar la regla de `sudoers`.

Cómo se detecta el abuso: no por el programa (es legítimo) sino por el contexto: quién lo lanza, con qué argumentos y hacia dónde se conecta. Un procesador de texto que lanza una consola, o una utilidad de certificados descargando un archivo de internet, son anomalías aunque ambos programas sean de Microsoft. Sysmon y el EDR registran esas cadenas de procesos.

```
LotL     → abusar de programas legítimos ya instalados.
LOLBAS   → catálogo de binarios abusables de Windows.
GTFOBins → catálogo de binarios abusables de Linux (sudo, SUID).
WADComs  → referencia de comandos para Windows y Active Directory.
```

## Basics of Reverse Engineering

La **ingeniería inversa** (reverse engineering) es el proceso de analizar un programa ya compilado para entender qué hace y cómo, sin tener su código fuente.

Analogía: desmontar un reloj para entender su mecanismo cuando nadie te dio los planos.

Para qué se usa en seguridad: analizar malware (qué hace, a dónde se conecta, cómo se elimina), encontrar vulnerabilidades en software cerrado, verificar que un programa hace lo que dice, y resolver retos "crackme" de CTF.

Los dos enfoques, que se combinan:

- Análisis estático: estudiar el binario sin ejecutarlo. Primero lo superficial (cadenas de texto, funciones importadas, hash para buscarlo en VirusTotal) y luego el desensamblado y la decompilación con herramientas como Ghidra.
- Análisis dinámico: ejecutarlo en un entorno aislado y observar qué hace (procesos, archivos, registro, red) o recorrerlo paso a paso con un depurador. Siempre en una VM desechable sin acceso a tu red (ver [`06-virtualizacion`](../06-virtualizacion/)).

Lo mínimo de ensamblador x86-64 para empezar: la CPU tiene registros (pequeñas variables internas como `rax`, `rbx`, `rsp`), y cada instrucción hace una cosa simple:

```asm
mov  eax, 5        ; eax = 5
add  eax, 3        ; eax = eax + 3  -> 8
cmp  eax, 8        ; compare eax with 8 (sets flags)
jne  wrong         ; jump to "wrong" if not equal
call print_ok      ; otherwise call the success routine
```

Ese patrón `cmp` seguido de un salto condicional es exactamente lo que se busca en un crackme: el punto donde el programa compara tu contraseña con la correcta y decide qué mensaje mostrar. Ghidra lo traduce a pseudocódigo como `if (x == 8) print_ok(); else wrong();`, que se lee mucho más fácil.

Ejemplo: descargas un crackme de nivel 1 de crackmes.one, lo abres en Ghidra, buscas la cadena "Wrong password", ves qué función la usa y en el pseudocódigo aparece la comparación con la contraseña esperada.

```
Estático  → analizar sin ejecutar: cadenas, imports, desensamblado.
Dinámico  → ejecutar en aislamiento y observar el comportamiento.
Ghidra    → desensamblador y decompilador gratuito de la NSA.
cmp + jne → comparar y saltar: donde el programa toma decisiones.
```

## Penetration Testing Rules of Engagement

Las **reglas de enfrentamiento** (rules of engagement, RoE) son el acuerdo escrito y firmado entre el cliente y el equipo de pentest que define qué se puede probar, cómo, cuándo y qué hacer si algo sale mal. Sin ese documento, una prueba de penetración es un ataque, y puede ser delito.

Analogía: es el contrato del cirujano: consentimiento firmado, qué se va a operar, qué no, y qué hacer si aparece una complicación. Sin él, abrir a alguien es una agresión.

Lo que fija como mínimo:

- Autorización por escrito de quien realmente tiene autoridad sobre los sistemas.
- Alcance (scope): qué IP, dominios, aplicaciones y sedes entran y, tan importante, cuáles no (por ejemplo, sistemas de terceros alojados en la misma nube).
- Ventanas de tiempo: fechas y horas permitidas, para no tumbar producción en hora punta.
- Técnicas permitidas y prohibidas: ¿se permite ingeniería social, denegación de servicio, pruebas físicas?
- Contactos de emergencia y qué hacer si se encuentra un fallo crítico o rastros de un atacante real.
- Manejo de los datos encontrados y confidencialidad del informe.

Tipos de prueba según lo que sabe el equipo de antemano: caja negra (nada, como un atacante externo), caja gris (algo, como un usuario normal) y caja blanca (todo: código, diagramas, credenciales).

Las fases de un pentest según PTES (Penetration Testing Execution Standard), en orden:

1. Pre-engagement: acordar las reglas anteriores.
2. Intelligence gathering: reconocimiento del objetivo.
3. Threat modeling: decidir qué activos importan y qué ataques son realistas.
4. Vulnerability analysis: buscar fallos.
5. Exploitation: confirmar que los fallos son aprovechables.
6. Post exploitation: medir el impacto real (qué datos se alcanzan) sin causar daño.
7. Reporting: informe con hallazgos, evidencias, riesgo y cómo corregir.

Ejemplo: una empresa contrata un pentest de su web `tienda.ejemplo` del 3 al 7 de noviembre, de 20:00 a 6:00, sin denegación de servicio ni phishing a empleados. El tercer día el equipo descubre que la base de datos es de un proveedor externo: aunque sea vulnerable, queda fuera de alcance y solo se reporta, no se prueba.

> [!WARNING]
> Practicar contra sistemas sin autorización por escrito es ilegal aunque "solo mires". Para aprender, usa laboratorios hechos para eso: TryHackMe, VulnHub, DVWA, Juice Shop o tus propias VMs.

```
RoE        → acuerdo firmado de qué, cómo y cuándo se prueba.
Scope      → lo que entra y lo que no entra en la prueba.
Caja negra / gris / blanca → cuánto sabe el equipo de antemano.
PTES       → 7 fases: pre-engagement, intelligence, threat modeling, vulnerability analysis, exploitation, post exploitation, reporting.
```

## Recursos para aprender y practicar

### Videos

- [you need to learn Python RIGHT NOW!!](https://www.youtube.com/watch?v=mRMmlo_Uqcs) — NetworkChuck; inicio de su serie de Python (Programming Skills: Python).
- [you need to learn BASH Scripting RIGHT NOW!!](https://www.youtube.com/watch?v=SPwyp2NG-bE) — NetworkChuck; scripting en Bash desde cero (Bash).
- [PowerShell for Hackers](https://www.youtube.com/watch?v=s2kquCwKNs8) — NahamSec; PowerShell desde la perspectiva de seguridad (Power Shell).
- [Assembly Language in 100 Seconds](https://www.youtube.com/watch?v=4gwYkEK0gOk) — Fireship; qué es el ensamblador (Basics of Reverse Engineering).
- [Ghidra quickstart & tutorial: Solving a simple crackme](https://www.youtube.com/watch?v=fTGTnrgjuGA) — stacksmashing; primer uso de Ghidra con un crackme (Basics of Reverse Engineering).
- [Burpsuite Basics (FREE Community Edition)](https://www.youtube.com/watch?v=G3hpAeoZ4ek) — John Hammond; proxy de interceptación (Common Hacking Tools).
- [Password Cracking](https://www.youtube.com/watch?v=7U-RbOKanYs) — Computerphile; cómo funciona el crackeo de hashes (Common Hacking Tools).
- [Metasploit For Beginners - The Basics](https://www.youtube.com/watch?v=8lR27r8Y_ik) — HackerSploit; módulos, exploits y payloads (Common Exploit Frameworks).
- [How to Proxy Command Execution: "Living Off The Land" Hacks](https://www.youtube.com/watch?v=QBvM-MzQ570) — John Hammond; LOLBins en Windows (Using tools for Unintended Purposes).
- [Penetration Tests](https://www.youtube.com/watch?v=wEMzVfwBiWY) — Professor Messer; tipos de pentest y reglas de enfrentamiento (Rules of Engagement).

### Lectura y documentación

- [Python docs](https://docs.python.org/3/), [Go docs](https://go.dev/doc/), [Node.js docs](https://nodejs.org/en/docs), [isocpp: Get Started](https://isocpp.org/get-started), [PowerShell docs](https://learn.microsoft.com/en-us/powershell/) — documentación oficial de cada lenguaje.
- [ShellCheck](https://www.shellcheck.net/) — revisa errores típicos en scripts de Bash.
- [JEA: Just Enough Administration](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/jea/overview) — limitar lo que puede hacer cada administrador en PowerShell.
- [MITRE ATT&CK T1059 Command and Scripting Interpreter](https://attack.mitre.org/techniques/T1059/) — cómo se abusan PowerShell, Bash, Python y JavaScript, y cómo detectarlo.
- [Burp Suite documentation](https://portswigger.net/burp/documentation), [John the Ripper](https://www.openwall.com/john/), [hashcat wiki](https://hashcat.net/wiki/), [THC Hydra](https://github.com/vanhauser-thc/thc-hydra), [Gobuster](https://github.com/OJ/gobuster) — documentación de las herramientas.
- [Metasploit docs](https://docs.metasploit.com/) y [Sliver](https://sliver.sh/) — documentación de los frameworks.
- [MITRE ATT&CK TA0011 Command and Control](https://attack.mitre.org/tactics/TA0011/) — cómo funciona y se detecta el C2.
- [LOLBAS](https://lolbas-project.github.io/), [GTFOBins](https://gtfobins.github.io/), [WADComs](https://wadcoms.github.io/) — los tres catálogos.
- [MITRE ATT&CK T1218 System Binary Proxy Execution](https://attack.mitre.org/techniques/T1218/) — abuso de binarios firmados de Windows y su detección.
- [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) — registrar cadenas de procesos para detectar LotL.
- [Ghidra](https://ghidra-sre.org/) y [Intel x86 instruction reference](https://www.felixcloutier.com/x86/) — herramienta y referencia de instrucciones.
- [PTES Pre-engagement](http://www.pentest-standard.org/index.php/Pre-engagement) — qué acordar antes de un pentest.
- [NIST SP 800-115](https://csrc.nist.gov/pubs/sp/800/115/final) — guía técnica de pruebas y evaluaciones de seguridad.

### Práctica

- [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) — terminal Linux y Bash por niveles.
- [OverTheWire Narnia](https://overthewire.org/wargames/narnia/) — introducción a fallos de memoria en C, para después de Bandit.
- [crackmes.one](https://crackmes.one/) — retos de ingeniería inversa por dificultad; empieza por nivel 1.
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — labs gratuitos para practicar con Burp.
- [TryHackMe: Python Basics](https://tryhackme.com/room/pythonbasics), [Custom Tooling Using Python](https://tryhackme.com/room/customtoolingpython) y [Bash Scripting](https://tryhackme.com/room/bashscripting) — gratis; los lenguajes en contexto de seguridad. Para PowerShell, [Windows Command Line](https://tryhackme.com/room/windowscommandline) (gratis).
- [TryHackMe: Burp Suite: Repeater](https://tryhackme.com/room/burpsuiterepeater), [Hydra](https://tryhackme.com/room/hydra), [Crack the hash](https://tryhackme.com/room/crackthehash) y [Content Discovery](https://tryhackme.com/room/contentdiscoveryx) — gratis; una sala por tipo de herramienta.
- [TryHackMe: Metasploit Introduction](https://tryhackme.com/room/metasploitintro) — el framework en un lab guiado.
- [TryHackMe: Linux Privilege Escalation](https://tryhackme.com/room/linprivesc) y [Linux PrivEsc](https://tryhackme.com/room/linuxprivesc) — gratis; GTFOBins en práctica. Para LOLBins de Windows, [Windows PrivEsc](https://tryhackme.com/room/windows10privesc) (gratis).
- [TryHackMe: Basic Malware RE](https://tryhackme.com/room/basicmalwarere) — primeros pasos de ingeniería inversa.
- [TryHackMe: Pentesting Fundamentals](https://tryhackme.com/room/pentestingfundamentals) — metodologías y reglas de enfrentamiento.
- [Ejercicio 1: Corre los cinco scripts de la nota y prográmalos](ejercicios.md#ejercicio-1-corre-los-cinco-scripts-de-la-nota-y-prográmalos) — ejecuta los scripts de hashes, puertos, 404, SSH y 4625 sobre tus equipos y déjalos corriendo solos con cron o el Programador de tareas, avisándote por correo.

## Cuadro resumen

Todo lo visto, en una línea por término.

Programming Skills

```
Python     → automatización general, parseo de logs, red, herramientas de pentest.
Go         → binario único y portable, concurrencia barata; herramientas modernas y C2.
JavaScript → lenguaje del navegador y de Node.js; donde nace el XSS.
C++        → control directo de memoria; base de SO, malware y exploits de memoria.
Bash       → pegamento de la terminal Linux; tuberías entre comandos.
PowerShell → shell de Windows basado en objetos; administración y también abuso.
```

Understand Common Hacking Tools

```
Burp/ZAP     → interceptar y modificar tráfico web.
Gobuster     → descubrir rutas web ocultas por fuerza bruta de nombres.
John/hashcat → crackear hashes offline; invisible para el defensor.
Hydra        → probar credenciales contra un servicio en línea; ruidoso.
```

Understand Common Exploit Frameworks

```
Exploit       → código que aprovecha un fallo concreto.
Payload       → lo que se ejecuta tras el exploit.
C2            → servidor desde el que se controla lo comprometido.
Metasploit    → framework abierto de referencia para aprender.
Cobalt Strike → framework comercial de red team, muy abusado por criminales.
```

Using tools for Unintended Purposes

```
LotL     → abusar de programas legítimos ya instalados.
LOLBAS   → catálogo de binarios abusables de Windows.
GTFOBins → catálogo de binarios abusables de Linux (sudo, SUID).
WADComs  → referencia de comandos para Windows y Active Directory.
```

Basics of Reverse Engineering

```
Estático  → analizar sin ejecutar: cadenas, imports, desensamblado.
Dinámico  → ejecutar en aislamiento y observar el comportamiento.
Ghidra    → desensamblador y decompilador gratuito de la NSA.
cmp + jne → comparar y saltar: donde el programa toma decisiones.
```

Penetration Testing Rules of Engagement

```
RoE   → acuerdo firmado de qué, cómo y cuándo se prueba.
Scope → lo que entra y lo que no entra en la prueba.
PTES  → 7 fases: pre-engagement, intelligence, threat modeling, vulnerability analysis, exploitation, post exploitation, reporting.
```
