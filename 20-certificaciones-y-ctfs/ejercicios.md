# Ejercicios: Certificaciones y CTFs

Ejercicios guiados para hacer en tu propio laboratorio. Cada uno dice qué vas a lograr,
qué necesitas, los pasos exactos y cómo comprobar que salió bien. La teoría está en
[README.md](README.md).

Estas máquinas son deliberadamente vulnerables: van siempre en una red aislada
(host-only o interna), nunca en modo bridged, para que no queden expuestas a tu red de
casa (ver [`06-virtualizacion`](../06-virtualizacion/)). Atacas solo máquinas tuyas en
tu laboratorio.

## Ejercicio 1: Monta el laboratorio local y descubre la máquina objetivo

Nodo: [VulnHub](README.md#vulnhub).

Objetivo: dejar funcionando un laboratorio de dos máquinas (una de ataque y Kioptrix
Level 1 como objetivo) en una red host-only aislada, y descubrir la IP y los servicios
del objetivo desde la máquina de ataque.

Necesitas: VirtualBox (o VMware); la imagen de
[Kioptrix Level 1](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/) (unos 200 MB);
una VM de ataque con Nmap (Kali, Parrot o cualquier Linux tuyo con Nmap). Tiempo: 1 h.
Todo en red host-only, sin salida a internet para la máquina objetivo.

### Pasos

1. Crea una red host-only en VirtualBox: `Herramientas > Red > Host-only Networks >
   Crear`. Anota su rango (por defecto `192.168.56.0/24`) y deja el servidor DHCP
   activado.
2. Descarga Kioptrix Level 1 desde VulnHub y verifícala si el autor publica un hash:
   ```bash
   sha1sum Kioptrix_Level_1.rar      # compara con el hash de la pagina de VulnHub
   ```
   Descomprime el archivo para obtener el `.ova` o los discos de la VM.
3. Importa la máquina (`Archivo > Importar servicio virtualizado`, o crea una VM y
   adjunta el disco). En su configuración de red, pon el adaptador en `Red solo-anfitrión
   (host-only)` apuntando a la red del paso 1. Haz lo mismo con tu VM de ataque.
4. Comprueba desde la VM de ataque en qué rango estás:
   ```bash
   ip -brief address
   ```
   Debe darte una IP dentro del rango host-only (p. ej. `192.168.56.1` o `.101`).
5. Descubre la IP de la máquina objetivo escaneando tu propio rango (host discovery):
   ```bash
   sudo nmap -sn 192.168.56.0/24
   ```
   Aparecerán tu VM de ataque y la objetivo; la que no reconozcas es Kioptrix. Anótala.
6. Enumera puertos, versiones y servicios del objetivo:
   ```bash
   sudo nmap -sV -sC -p- 192.168.56.NNN -oN kioptrix_nmap.txt
   ```
   `-p-` escanea los 65535 puertos, `-sV` detecta versiones y `-sC` corre los scripts
   por defecto. Guarda la salida: es la base de tu informe.

### Resultado esperado

Dos VMs en la misma red host-only que se ven entre sí, y el archivo `kioptrix_nmap.txt`
con la lista de puertos y versiones de servicios del objetivo (verás servicios antiguos
como Apache, Samba y OpenSSH con versiones vulnerables).

### Comprueba que lo lograste

- ¿La máquina objetivo responde al ping/host discovery pero NO tiene salida a internet?
  Confírmalo: desde la objetivo, un `ping 8.8.8.8` debe fallar. Esa es la prueba de que
  el aislamiento está bien hecho.
- ¿`kioptrix_nmap.txt` muestra al menos un servicio con una versión claramente antigua?
  Ese es tu punto de entrada para el ejercicio 2.

### Limpieza

Apaga las dos VMs cuando no practiques. La red host-only puede quedarse: no da acceso a
tu red real. Si terminas con la máquina, elimínala desde VirtualBox.

## Ejercicio 2: Compromete Kioptrix Level 1 y redacta el informe

Nodo: [VulnHub](README.md#vulnhub) y
[OSCP](README.md#oscp) (práctica del formato de informe que exige el examen).

Objetivo: completar el ciclo de un pentest contra tu Kioptrix Level 1 (reconocimiento,
explotación, escalada a root y lectura de la bandera) y documentarlo en un informe con
la plantilla de abajo, como se espera en el OSCP.

Necesitas: el laboratorio del ejercicio 1 ya montado y el archivo `kioptrix_nmap.txt`.
Tiempo: 3-5 h según experiencia. Enfoque de aprendizaje: si te atascas, usa un writeup
solo después de intentarlo, y documenta con tus palabras.

### Pasos

1. Reconocimiento: a partir de `kioptrix_nmap.txt`, elige el servicio con la versión
   más antigua y busca sus vulnerabilidades conocidas por versión:
   ```bash
   searchsploit apache 1.3
   searchsploit samba 2.2
   ```
   Apunta los CVE o exploits candidatos. Enumera también el servicio web:
   ```bash
   whatweb http://192.168.56.NNN
   gobuster dir -u http://192.168.56.NNN -w /usr/share/wordlists/dirb/common.txt
   ```
2. Análisis de vulnerabilidades: para cada candidato, anota qué fallo es, qué requisito
   tiene y qué impacto daría. No lances nada todavía: primero decide cuál es el camino
   más probable de acceso inicial.
3. Explotación: consigue una shell en la máquina objetivo con el exploit elegido. Puedes
   hacerlo a mano (compilar un exploit de searchsploit) o con Metasploit en tu
   laboratorio. Documenta el comando exacto y la salida que confirma el acceso (por
   ejemplo, el prompt de una shell y `id`).
4. Post-explotación y escalada: comprueba con qué usuario entraste y escala a root si no
   lo eres ya:
   ```bash
   id
   uname -a          # version de kernel, candidata a exploit local
   ```
   Busca un exploit de escalada acorde a esa versión y ejecútalo en tu laboratorio.
5. Bandera: lee la prueba de compromiso total (en Kioptrix, normalmente un archivo en
   el directorio de root):
   ```bash
   cat /root/*.txt 2>/dev/null ; ls -la /root
   ```
   Guarda una captura o el texto como evidencia.
6. Informe: rellena la plantilla de abajo. Para cada hallazgo, incluye evidencia (el
   comando y su salida) y una recomendación de corrección.

#### Plantilla de informe

```
INFORME DE PENTEST - Kioptrix Level 1 (laboratorio propio)

1. Resumen ejecutivo
   - Objetivo del ejercicio y resultado en 3-4 frases, sin tecnicismos.
   - Nivel de riesgo global: alto / medio / bajo.

2. Alcance y reglas
   - Maquina: Kioptrix Level 1, IP 192.168.56.NNN, red host-only aislada.
   - Fechas y duracion. Autorizacion: laboratorio propio.

3. Metodologia
   - Fases seguidas (reconocimiento, analisis, explotacion, post-explotacion).
   - Herramientas usadas.

4. Hallazgos (uno por vulnerabilidad)
   Por cada hallazgo:
   - Titulo y severidad (con vector CVSS si aplica).
   - Descripcion: que es y por que es un problema.
   - Evidencia: comando exacto y salida (captura o texto).
   - Pasos de reproduccion numerados.
   - Impacto: que consiguio el atacante.
   - Recomendacion: como se corrige (parche, configuracion, version).

5. Narrativa del ataque
   - Relato en orden de como se paso de cero a root, enlazando los hallazgos.

6. Conclusiones y remediacion priorizada
   - Lista de acciones ordenadas por urgencia.

7. Anexos
   - Salida completa de nmap y de los exploits.
```

### Resultado esperado

La bandera de root leída en tu máquina y un informe completo que sigue la plantilla, con
al menos el hallazgo de acceso inicial y el de escalada de privilegios documentados con
evidencia y recomendación.

### Comprueba que lo lograste

- ¿`id` devuelve `uid=0(root)` en algún momento de tu evidencia? Esa es la prueba
  objetiva del compromiso total.
- ¿Tu informe permite que otra persona reproduzca el ataque solo leyéndolo, sin pedirte
  nada más? Dáselo a leer a alguien o déjalo un día y vuelve: si puedes repetir los
  pasos desde tu propio texto, está completo.
- ¿Cada hallazgo tiene su recomendación de corrección? Un informe sin remediación no
  sirve en un pentest real.

### Limpieza

Apaga o elimina la VM objetivo. Guarda tu informe en tu carpeta de writeups (es
material valioso para preparar certificaciones como el OSCP). No publiques soluciones
completas de máquinas que sigan activas en plataformas de competición.
