# Ejercicios: Estándares y cumplimiento

Ejercicios guiados para hacer en tu propio equipo o laboratorio. Cada uno dice qué vas
a lograr, qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría
está en [README.md](README.md).

Todo el tráfico de escaneo va contra máquinas virtuales tuyas en una red aislada
(host-only o interna), nunca contra sistemas de terceros.

## Ejercicio 1: Recalcula el vector CVSS de tres CVE

Nodo: [CVSS](README.md#cvss).

Objetivo: dado el texto de tres CVE reales, reconstruir el vector CVSS 3.1 (y uno en
4.0) métrica a métrica y obtener una nota que caiga en la misma franja de severidad que
la oficial de NVD (None, Low, Medium, High o Critical), explicando de dónde sale cada
letra.

Necesitas: solo un navegador. Las calculadoras oficiales de FIRST
([3.1](https://www.first.org/cvss/calculator/3.1), [4.0](https://www.first.org/cvss/calculator/4.0))
y la base [NVD](https://nvd.nist.gov/vuln/search). Tiempo: 45-60 min. No instalas nada.

### Pasos

1. Elige tres CVE de perfiles distintos para que el ejercicio no sea repetitivo. Una
   terna recomendada: `CVE-2021-44228` (Log4Shell, ejecución remota por red),
   `CVE-2014-0160` (Heartbleed, fuga de memoria sin autenticación) y `CVE-2016-5195`
   (Dirty COW, escalada local). Abre cada uno en NVD:
   `https://nvd.nist.gov/vuln/detail/CVE-2021-44228`.
2. Lee solo la descripción del CVE, sin mirar todavía el apartado "CVSS Severity".
   Tápalo o desplázate sin leerlo.
3. Para cada CVE, responde en papel las ocho métricas Base de 3.1 a partir de la
   descripción:
   - AV (Attack Vector): ¿el atacante está en la red (N), en la red adyacente (A), es
     local (L) o físico (P)?
   - AC (Attack Complexity): ¿sin condiciones especiales (L) o con ellas (H)?
   - PR (Privileges Required): ¿ninguno (N), bajos (L) o altos (H)?
   - UI (User Interaction): ¿ninguna (N) o requiere que la víctima actúe (R)?
   - S (Scope): ¿el fallo afecta solo a su componente (U) o salta a otro (C)?
   - C, I, A: impacto en confidencialidad, integridad y disponibilidad: H, L o N.
4. Entra en la calculadora 3.1, marca esas opciones y anota la nota y el vector que te
   genera.
   ```
   CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H  ->  10.0 Critical
   ```
   El vector resume tus elecciones; la nota es el número de 0,0 a 10,0.
5. Ahora sí, vuelve a NVD y compara tu vector con el oficial ("CVSS 3.x Severity and
   Metrics"). Anota en qué métricas difieres y por qué (relee la descripción).
6. Repite el CVE que más te costó en la calculadora 4.0 y observa que las métricas
   cambian de nombre (por ejemplo, el impacto se separa en Vulnerable System y
   Subsequent System) aunque las franjas de severidad son las mismas.

### Resultado esperado

Una tabla tuya con tres filas (CVE, tu vector, tu nota, nota oficial, franja) y una
nota corta por métrica donde no coincidiste.

### Comprueba que lo lograste

- ¿La franja de severidad (Low/Medium/High/Critical) de tus tres notas coincide con la
  de NVD? Para Log4Shell debe darte Critical (9,0-10,0); para Dirty COW, por ser local
  (AV:L), debe quedar por debajo de la de un fallo remoto equivalente.
- ¿Sabes explicar por qué Dirty COW no llega a 10,0 aunque da control total? Respuesta:
  AV:L baja la nota frente a un AV:N.

## Ejercicio 2: Escaneo con y sin credenciales usando Greenbone

Nodo: [Escáneres de vulnerabilidades](README.md#escáneres-de-vulnerabilidades) y
[Ciclo de gestión de vulnerabilidades](README.md#ciclo-de-gestión-de-vulnerabilidades).

Objetivo: escanear una VM vulnerable propia dos veces, una sin credenciales y otra con
credenciales, comprobar que el escaneo autenticado encuentra más hallazgos, y priorizar
los cinco primeros cruzándolos con CISA KEV y EPSS.

Necesitas: VirtualBox o similar con una red host-only; una VM objetivo vulnerable
(Metasploitable 2 o una VM Linux vieja tuya); una segunda máquina Linux con Docker para
Greenbone Community Edition. Tiempo: 2-3 h (la primera sincronización de feeds de
Greenbone tarda). Todo en red host-only, aislado de tu red doméstica.

### Pasos

1. En VirtualBox crea una red host-only (Herramientas > Red > Host-only Networks >
   Crear) y pon las dos VMs (objetivo y la que correrá Greenbone) en esa red. Verifica
   que se ven entre sí:
   ```bash
   ip -brief address        # anota la IP host-only de cada VM, p. ej. 192.168.56.0/24
   ping -c1 192.168.56.10   # desde la VM de Greenbone hacia la objetivo
   ```
2. Levanta Greenbone Community Edition con los contenedores oficiales (lo dicta la
   [documentación de Greenbone](https://greenbone.github.io/docs/latest/22.4/container/)):
   ```bash
   mkdir ~/greenbone && cd ~/greenbone
   curl -f -L https://greenbone.github.io/docs/latest/_static/docker-compose-22.4.yml \
     -o docker-compose.yml
   docker compose -f docker-compose.yml -p greenbone-community-edition pull
   docker compose -f docker-compose.yml -p greenbone-community-edition up -d
   ```
   Espera a que los feeds terminen de sincronizar (`docker compose ... logs -f` deja de
   mostrar descargas); puede tardar 30-60 min la primera vez.
3. Entra a la interfaz web en `https://127.0.0.1:9392` (usuario `admin`; la contraseña
   se fija según la doc al crear el usuario). En el menú `Configuration > Credentials`
   crea unas credenciales SSH válidas de la VM objetivo (un usuario propio de esa VM).
4. Crea un objetivo (`Configuration > Targets`) con la IP host-only de la VM objetivo
   y, de momento, sin credenciales. Lanza una tarea (`Scans > Tasks > New Task`) con la
   configuración "Full and fast". Anota el número total de resultados al terminar.
5. Duplica el objetivo, esta vez asignándole las credenciales SSH del paso 3, y vuelve
   a escanear. Anota de nuevo el total.
6. Para los cinco hallazgos de mayor CVSS del escaneo autenticado, abre cada CVE y
   cruza su explotabilidad real:
   - CISA KEV: busca el CVE en
     `https://www.cisa.gov/known-exploited-vulnerabilities-catalog`. Si aparece, sube
     a lo más alto de tu lista.
   - EPSS: consulta su probabilidad en `https://www.first.org/epss/` (o la API:
     `curl -s "https://api.first.org/data/v1/epss?cve=CVE-2021-44228"`).
   Ordena los cinco por "está en KEV" primero, luego por EPSS, no solo por CVSS.

### Resultado esperado

Dos números de hallazgos (sin y con credenciales) y una lista priorizada de cinco
vulnerabilidades con su CVSS, si está en KEV y su EPSS, con el orden de parcheo
justificado.

### Comprueba que lo lograste

- ¿El escaneo autenticado dio más hallazgos que el anónimo? Debe darlos, porque lee los
  paquetes instalados en vez de solo adivinar por banners.
- ¿Tu primer puesto de la lista es el que está en KEV aunque no sea el de mayor CVSS?
  Esa es la lección: gravedad no es lo mismo que riesgo.

### Limpieza

```bash
cd ~/greenbone
docker compose -f docker-compose.yml -p greenbone-community-edition down -v
```
Apaga o elimina las VMs del laboratorio. La red host-only puede quedarse; no da acceso
a internet.

## Ejercicio 3: Decide corregir, mitigar o aceptar con Nmap vulners

Nodo: [Ciclo de gestión de vulnerabilidades](README.md#ciclo-de-gestión-de-vulnerabilidades)
y [Escáneres de vulnerabilidades](README.md#escáneres-de-vulnerabilidades).

Objetivo: reproducir la salida de la nota con el script `vulners` de Nmap contra tu VM
vulnerable y, para cada servicio con hallazgos, escribir una decisión razonada de
tratamiento: corregir, mitigar o aceptar.

Necesitas: la misma VM objetivo del ejercicio 2 en red host-only y una VM de ataque con
Nmap. Tiempo: 45-60 min. El script `vulners` consulta su base por internet desde la VM
de ataque; la VM objetivo sigue aislada.

### Pasos

1. Comprueba que tienes el script `vulners` (viene con Nmap reciente):
   ```bash
   nmap --script-help vulners | head
   ```
   Si no aparece, instálalo con: `sudo nmap --script-updatedb` tras copiar
   `vulners.nse` desde su [repositorio](https://github.com/vulnersCom/nmap-vulners),
   o actualiza Nmap. En Debian/Ubuntu: `sudo apt install nmap`; en Arch:
   `sudo pacman -S nmap`.
2. Escanea versión y vulnerabilidades de los puertos abiertos de tu VM objetivo:
   ```bash
   nmap -sV --script vulners -p- 192.168.56.10
   ```
   `-sV` detecta la versión de cada servicio y `--script vulners` lista los CVE
   asociados a esa versión con su CVSS, igual que el ejemplo del README.
3. Copia la salida a un archivo de trabajo:
   ```bash
   nmap -sV --script vulners -oN vulners.txt -p- 192.168.56.10
   ```
4. Para cada servicio con hallazgos, decide el tratamiento y escríbelo en una tabla de
   texto (servicio, CVE principal, decisión, motivo):
   - Corregir: hay actualización disponible y el servicio importa. Ej.: subir la versión
     de Apache.
   - Mitigar: no puedes parchear ya, pero reduces la exposición (cerrar el puerto en el
     firewall, poner una regla, aislar la VM).
   - Aceptar: el riesgo es bajo y lo registras con fecha de revisión (servicio de
     laboratorio sin datos).

### Resultado esperado

El archivo `vulners.txt` y tu tabla de decisiones con una fila por servicio vulnerable.

### Comprueba que lo lograste

- ¿Cada hallazgo tiene exactamente una de las tres decisiones y un motivo de una línea?
- ¿Marcaste "mitigar" en al menos un caso en el que no hay parche inmediato pero sí
  puedes reducir la exposición? Esa distinción es el objetivo del ejercicio.

### Limpieza

Borra `vulners.txt` si contiene detalles de tu laboratorio. Apaga la VM objetivo.

## Ejercicio 4: Audita tu Linux con Lynis y contrástalo con el CIS Benchmark

Nodo: [CIS](README.md#cis).

Objetivo: ejecutar Lynis en tu propia máquina Linux, leer el "hardening index" y
comparar cinco de sus sugerencias con las recomendaciones del CIS Benchmark de tu
distribución, decidiendo cuáles aplicarías.

Necesitas: tu Linux de siempre (o una VM Linux tuya); Lynis instalado; acceso al CIS
Benchmark de tu distribución (descarga gratuita previo registro en
[CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)). Tiempo: 1 h. No modifica
nada del sistema: Lynis solo audita.

### Pasos

1. Instala Lynis. En Debian/Ubuntu: `sudo apt install lynis`; en Arch:
   `sudo pacman -S lynis`. Comprueba la versión:
   ```bash
   lynis show version
   ```
2. Ejecuta la auditoría del sistema (Lynis recomienda correr como root para ver todo):
   ```bash
   sudo lynis audit system
   ```
3. Al terminar, localiza el índice de endurecimiento en la sección final:
   ```bash
   sudo grep -i "hardening index" /var/log/lynis.log
   ```
   Es un número de 0 a 100; anótalo como tu línea base.
4. Extrae las sugerencias (líneas `Suggestion[]`) a un archivo:
   ```bash
   sudo grep "Suggestion\[\]" /var/log/lynis-report.dat > sugerencias.txt
   ```
5. Elige cinco sugerencias (por ejemplo, sobre SSH, módulos del kernel, parámetros de
   `sysctl`, permisos de archivos o banners). Para cada una, busca en el PDF del CIS
   Benchmark de tu distribución la recomendación equivalente y anota: número del
   control CIS, si es Level 1 o Level 2, y si coincide con lo que dice Lynis.
6. Verifica una a mano para entender la diferencia entre "sugerencia" y "estado real".
   Ejemplo del README, el acceso SSH como root:
   ```bash
   sudo sshd -T | grep -i permitrootlogin
   ```
   Si devuelve `permitrootlogin yes`, no cumple el benchmark; `no` cumple.

### Resultado esperado

Tu hardening index inicial y una tabla de cinco filas que enlaza cada sugerencia de
Lynis con su control CIS (número, nivel, coincide sí/no).

### Comprueba que lo lograste

- ¿Encontraste al menos una sugerencia de Lynis que mapea a un control CIS de Level 1
  (ajuste básico) y otra de Level 2 (más estricto)?
- ¿Entiendes por qué Lynis es "aproximado" frente a una herramienta oficial como
  CIS-CAT? Respuesta: Lynis es un auditor genérico, no comprueba el benchmark exacto
  línea por línea.

### Limpieza

Borra los archivos de trabajo (`sugerencias.txt`). No apliques cambios de hardening en
tu equipo principal sin entenderlos; practícalos en una VM.

## Ejercicio 5: Perfil CSF 2.0 actual y objetivo de tu casa

Nodo: [CSF](README.md#csf).

Objetivo: construir un perfil actual y un perfil objetivo del CSF 2.0 para tus propios
dispositivos domésticos (portátil, móvil, router) con una línea por función, y deducir
de la diferencia tu plan de trabajo personal.

Necesitas: solo papel o un archivo de texto y el documento
[CSF 2.0 (CSWP 29)](https://nvlpubs.nist.gov/nistpubs/CSWP/NIST.CSWP.29.pdf) como
referencia de las seis funciones. Tiempo: 45 min. No toca ningún sistema.

### Pasos

1. Escribe las seis funciones del CSF 2.0 como encabezados de una tabla de texto:
   Govern (GV), Identify (ID), Protect (PR), Detect (DE), Respond (RS), Recover (RC).
2. Perfil actual: para cada función, escribe en una línea qué haces hoy de verdad en tu
   casa. Sé honesto. Ejemplos:
   - ID: no tengo inventario de mis dispositivos ni sé qué versión de firmware tiene el
     router.
   - PR: uso gestor de contraseñas y MFA en el correo, pero el router tiene la clave de
     fábrica.
   - RC: no tengo copias de seguridad probadas del portátil.
3. Perfil objetivo: para cada función, escribe el resultado concreto y verificable que
   quieres alcanzar en, digamos, tres meses. Ejemplos:
   - ID: lista de mis dispositivos con su IP y versión de firmware.
   - PR: contraseña propia y fuerte en el router, firmware actualizado, MFA en todas las
     cuentas importantes.
   - RC: copia 3-2-1 del portátil con una restauración de prueba hecha.
4. Resta: para cada función, la diferencia entre actual y objetivo es una tarea. Ordena
   las tareas por esfuerzo y riesgo y ponles fecha.

### Resultado esperado

Una tabla con tres columnas (función, perfil actual, perfil objetivo) y una lista de
tareas con fecha que sale de las diferencias.

### Comprueba que lo lograste

- ¿Cada una de las seis funciones tiene una línea en actual y otra en objetivo, sin
  dejar ninguna en blanco (incluida Govern, que suele olvidarse: aquí sería "decido qué
  protejo y me hago responsable")?
- ¿Tu plan de trabajo tiene al menos una tarea por cada hueco encontrado, con una fecha?
